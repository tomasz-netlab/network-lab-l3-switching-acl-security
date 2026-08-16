# lessons-learned.md — Lab 02

Notes kept while building this lab, cleaned up at the end. Some of it is technical detail I want to remember; some of it is about how I work when something doesn't behave the way I expect.

The short version: designing the ACL policy on paper before touching the CLI was the single best decision in this project. Almost everything that went wrong afterwards was a gap between the model in my head and how the protocol actually behaves on the wire — and having a written policy made those gaps visible instead of leaving me guessing.

---

## Designing before configuring

In Lab 01 I wrote ACLs the way most tutorials do: think about the rule, type it, test it. That works when you have four rules. It falls apart at forty.

This time I built the policy in three layers before configuring anything — zone definitions and rules in plain language, then a Traffic Flow Matrix with every permitted and denied flow given an ID, then a deployment map saying which ACL sits on which interface, and only then the rule-by-rule detail sheet. It took a couple of evenings and felt slow at the time.

It paid for itself the first time something broke. When DHCP stopped working, I didn't have to ask "is this rule wrong?" — I could ask "which flow is this, and does the rule match the flow?" That's a much smaller question. Every ACE traces back to a flow ID, and every flow traces back to a stated policy. When I later had to add rules mid-project, I knew exactly where they belonged and what they were for.

The other benefit only showed up at the end: the design documents made the ACLs auditable. Anyone can read the matrix and check whether the configuration matches the intent, without reverse-engineering forty lines of CLI.

---

## The mistakes that cost the most time

### DHCP through a relay is three flows, not one

This was the expensive one — most of an evening.

DHCP looks simple: client asks, server answers. But with a relay agent in the path it's three separate conversations, and each needs its own ACE:

1. **Initial Discover** comes from `0.0.0.0`, because the client has no address yet. It does not match a rule scoped to the zone subnet — from the ACL's point of view that host isn't in the subnet at all.
2. **Renewal** comes from the client's real address, source port 68 to destination 67. This is the rule every textbook shows.
3. **Server reply to the relay** runs 67 → 67, not 67 → 68. Port 68 only appears on the last hop, from the relay to the client, inside the VLAN.

I had written only the second one. The Discover fell through to the closing deny, and even after I fixed that, the server's Offer was still being killed by the RFC 1918 deny block further down the list, because my "server to client" rule expected port 68 on the destination.

What actually solved it was Simulation Mode. I could see the packet matching `deny ip 10.0.120.0 0.0.0.15 192.168.0.0 0.0.255.255` and being dropped — at which point the problem stopped being "DHCP doesn't work" and became "why is the server's reply hitting an internal deny rule". Very different question.

The general lesson is worth keeping: **when a protocol goes through a relay or a proxy, the port pairs on each leg are not the same.** An ACL written from the end-to-end mental model will block the middle leg, and it will do it silently.

### `0.0.0.0 0.0.0.0` is not `any`

I wanted HR to reach any HTTPS destination on the internet, so I wrote `0.0.0.0 0.0.0.0` as the destination — the address plus what I thought was a "match everything" mask.

It isn't. As an address/wildcard pair that means *exactly one host*: `0.0.0.0`. IOS even renders it back as `host 0.0.0.0`, which should have tipped me off. "Any address" is `0.0.0.0 255.255.255.255` — wildcard all ones, no bits need to match — or just the `any` keyword.

The rule sat in the config looking perfectly reasonable and matched nothing. HR couldn't reach the web server and I spent a while looking at routing and at the server before going back to the ACL. The tell was in the hit counters: zero matches on a rule that should have been busy.

Wildcard masks are inverted subnet masks. I knew that. Knowing it and applying it under time pressure turn out to be different things.

### Two-level features: the parameters are not the switch

Twice in this lab I configured a feature completely and it did nothing, because the enabling command was missing:

**Port security.** I had `maximum`, `mac-address sticky` and `violation restrict` on both access ports. All three showed up in `show run`. But `show port-security interface fa0/2` said `Port Security: Disabled`. The bare `switchport port-security` — the actual switch — wasn't there. Those three lines are only parameters; without the enabler they're inert configuration.

**DHCP snooping** works the same way: global `ip dhcp snooping` turns the feature on, `ip dhcp snooping vlan 10,20` says where to inspect. Global alone sets ports to untrusted-by-default but inspects nothing. Per-VLAN alone does nothing at all, because the master switch is off.

This pattern is easy to miss precisely because the configuration *looks* complete. Worth remembering as a general check: when a feature is configured but not working, verify the feature is actually enabled before debugging its parameters.

### Check the host before you blame the network

Two separate incidents, same shape.

Admin-PC could ping devices in its own subnet but nothing beyond it. I assumed ACLs — I'd just written a lot of them, and blaming your newest change is a natural instinct. It was a missing default gateway on the PC.

Later the internet web server wouldn't respond at all. The router's interface was up with the right address, routing looked fine. The server had two NICs and the IP address was on the one that wasn't cabled.

The pattern to remember: **"works locally, fails across subnets" points at the host's default gateway, not at the routers.** And when Layer 3 looks correct on the network device, check the other end — the host, and specifically *which interface* the address is on.

---

## Things I understand better now

### Stateless means every direction is a rule

Extended ACLs on routed interfaces don't track sessions. There's no "allow the reply" — the reply is a separate flow with its own rule, or it's dropped.

That's why ACL_SERVERS_IN is so much bigger than the others: the server answers four zones, and every one of those replies is an explicit permit. It's also why the design distinguishes carefully between:

- **TCP replies** → `established`, which matches segments with ACK/RST set, i.e. traffic inside a session the client already opened.
- **UDP replies** → no session state exists, so they're matched by the server's *source* port instead (DNS answering from 53, DHCP from 67).
- **ICMP replies** → matched by type: `echo` outbound, `echo-reply` / `ttl-exceeded` / `unreachable` coming back.

Once that clicked, the whole shape of ACL_SERVERS_IN stopped looking arbitrary.

### Broad permits need a fence in front of them

Any rule that permits to `any` will also match internal networks. HR's HTTPS permit and the server's DNS/SMTP relay both destination-match `any`, so each is preceded by three RFC 1918 deny rules blocking `10.0.0.0/8`, `172.16.0.0/12` and `192.168.0.0/16`.

Deny-internal-then-permit-internet is the safe ordering. Without it, a rule meant to allow web browsing quietly allows internal traffic to anything on port 443. The hit counters confirmed these fences do work — 24 and 28 matches on the HR deny lines during testing.

### PVST+ really does run a separate tree per VLAN

I knew this as a fact. Seeing it was different.

SW-DISTRO is root for VLANs 10, 20, 101 and 123 — I set that explicitly. VLAN 99 isn't in that list, so for VLAN 99 the root is elected by default priority and lands elsewhere. The result: on SW1, Gi0/2 blocks for the production VLANs but *forwards* for VLAN 99, while Gi0/1 blocks. Mirror image, same physical link, different VLAN.

It has no functional impact here, because VLAN 99 carries no traffic. But it's a clean demonstration that "the blocked port" isn't a property of the topology — it's a property of the topology *per VLAN*.

Also worth noting: `spanning-tree vlan X root primary` is a macro. It sets priority to 24576. The reason priorities move in steps of 4096 is the Extended System ID, which reserves the lower 12 bits of the Bridge ID for the VLAN number — which is why `show spanning-tree` reports 24586 for VLAN 10 (24576 + 10) rather than a round number.

### STP needs time, and sometimes a nudge

After a batch of configuration changes I ran `show spanning-tree` on the root bridge and saw both downlinks in BLK. On a root bridge that shouldn't happen — root ports are always designated and forwarding.

It was mid-convergence. But it didn't resolve on its own; I ended up reloading SW1 and SW2 before the tree settled into the correct state. Whether that's simulator behaviour or a genuine convergence stall I can't say with certainty, but the practical lesson stands: **after topology or STP changes, verify the tree has actually converged before drawing conclusions from it.** A snapshot taken mid-convergence looks like a fault.

### Put controls where they protect something

DHCP Snooping and DAI run on SW1 only. Not on SW2, not on SW-DISTRO, not on the routers.

The reasoning: SW1 is the only switch with DHCP clients on access ports. SW2's single access host — the Admin-PC — is statically addressed, so snooping there would build an empty binding table and protect nothing. Worse, DAI on a static host actively breaks it: with no binding entry, the host's ARP gets dropped unless you add an ARP ACL, which Packet Tracer doesn't support.

That's the point I want to keep. Defense in depth doesn't mean every control everywhere — it means the right control where the risk actually is. A control with nothing to protect isn't a layer of defense, it's cost: CPU, ASIC resources, and a real risk of blocking legitimate traffic. SW2's access port is covered by port security, PortFast and BPDUGuard, which is the correct set for a static host.

The same logic drove ICMP handling: granular by type at the trust boundaries, general within the infrastructure, where the server and the internal planes already trust each other for service traffic.

### Binding tables don't survive a reboot

The DHCP snooping binding table lives in RAM. After the first switch reload it was empty — and DAI, which depends on it, had nothing to validate against until every client happened to renew its lease. In the meantime the protection was nominally configured and practically absent.

The fix is the database agent:

```
ip dhcp snooping database flash:dhcp-snooping.db
ip dhcp snooping database write-delay 30
```

After adding it, a reload showed `Successful Reads: 1` and the bindings restored from flash without any client action.

I added this because testing exposed the problem, not because a checklist told me to. That's the version of this lesson I want to keep: **a security control that resets to zero after a reboot is a gap, and reboots happen.**

### The modern version of all this

Snooping, DAI and IP Source Guard grew up separately over about fifteen years, each added in response to a specific attack. That history is why they look like three loosely-coupled features sharing one table.

Cisco's answer to that is **SISF (Switch Integrated Security Features) / device tracking** — one unified binding database fed by DHCPv4, DHCPv6, ARP, ND and static entries, which all the security features read from. On Catalyst 9000 and IOS-XE that's the underlying mechanism, and the classic "snooping binding table" is effectively a compatibility view over it.

Worth knowing that the CCNA-level model I'm learning is the historical one, and where it has gone since.

---

## How I work when something breaks

Halfway through this lab I caught myself reaching for "it's a Packet Tracer bug" too quickly. That's a comfortable conclusion — it ends the investigation and puts the blame elsewhere. It's also dangerous, because if the real cause is my configuration, I've just left a hole and convinced myself it's fine.

So I made it a rule for the rest of the project: **exhaust the configuration hypotheses first, then verify against outside sources, and only then conclude it's the tool.**

That order changed the outcome at least twice. With DAI I formed three concrete hypotheses — wrong trust state on the access ports, ARP not actually being generated, packet not reaching the inspection process — and tested each one (`show ip arp inspection interfaces`, `arp -d` followed by ping, Simulation Mode trace). All three were eliminated before I went looking for reports of the same behaviour. When I found them, I knew what I was reading, instead of hoping.

The inverse also happened: I twice stated something confidently that turned out to be wrong when tested properly — including one case where I'd concluded the snooping database agent doesn't restore on boot, and a clean test showed it does. The habit of running the test rather than trusting the first read is what caught it.

For diagnostics specifically, the tools that earned their place here were Simulation Mode (which told me *which rule* dropped a packet — nothing else does that), ACL hit counters (a rule with zero matches on traffic that should be busy is a strong signal), and plain `show` commands cross-checked against each other. `show interfaces trunk` independently confirming which port STP had blocked was a nice example — two unrelated commands agreeing is worth more than either alone.

---

## Packet Tracer limitations found

Documented here so I don't re-diagnose them next time, and so the configuration in `configs/` is read as correct rather than broken.

**ACL on SVIs doesn't persist or display.** `ip access-group` applied to an SVI on a multilayer switch doesn't show in `show running-config` or `show ip interface`, and stops filtering after a reload. Re-applying restores enforcement immediately — confirmed by hit counters — even though the interface still reports "Inbound access list is not set". Known defect on the 3560/3650 platforms, reported on the Cisco Community forum. The contrast is clean: the same command on R1's serial interface displays perfectly. Practical consequence for this lab: **re-apply the four SVI ACLs after every reload before testing.**

**DAI doesn't enforce.** Configuration is accepted and reports `Enabled / Active`, but counters stay at zero and spoofed ARP passes. Tested deliberately: PC1-HR reassigned to a static `192.168.10.5` while keeping its MAC, creating a mismatch against the binding entry for `192.168.10.2`. On real hardware that ARP gets dropped. Here the ping succeeded and nothing incremented. Three configuration hypotheses eliminated first (see above).

**`log` keyword unsupported on ACEs.** The closing `deny ip any any` in each ACL is meant to be logged for security visibility. Not available, so it's documented as intent in the design rather than implemented.

**No IMAP server.** The EMAIL service implements SMTP and POP3 only. IMAP (TCP 143) stays in the ACLs as part of the intended mail policy, but there's nothing to test against.

**No ARP ACLs.** `arp access-list` isn't available, which is why the static-host workaround for DAI couldn't be demonstrated.

**`Last Succeded Time` never populates** in `show ip dhcp snooping database`, even after successful transfers. The counters increment correctly; the timestamp field just isn't filled in.

**Default `serverPool` can't be deleted** from the DHCP service. Left in place with Max Users 0 — inactive and harmless, but it can't be cleaned up.

---

## What I'd do differently

**Test the deny cases earlier.** For a long stretch everything "worked", which felt like progress. It wasn't — the ACLs weren't filtering at all because of the SVI defect, and I only found out when I first tested something that was *supposed* to fail. A policy isn't validated until both the permits and the denies have been checked. In hindsight the very first ACL test should have been a deny.

**Trace the protocol before writing the rule.** The DHCP relay problem existed in the design from day one. Half an hour spent drawing the actual packet flow — who talks to whom, on which ports, on each leg — would have caught it before it cost an evening of debugging.

**Move to real IOS images.** Several hours in this lab went into diagnosing simulator defects rather than learning networking. Packet Tracer is fine for topology and basic configuration, but for security features — DAI, snooping, ACLs on SVIs — the gap between simulated and real behaviour is large enough to actively get in the way. The next lab goes on CML.

**Keep the notes as I go, not at the end.** Reconstructing which test produced which output, days later, took longer than it should have. The Traffic Flow Matrix stayed current throughout and was the most useful document in the project; the running notes didn't, and I paid for it.

---

## What worked

The three-layer ACL design. The addressing plan being written down before configuration rather than after. Simulation Mode as a diagnostic tool. Hit counters as evidence rather than a curiosity. And the discipline — learned partway through, admittedly — of not accepting "it's the tool" until the configuration hypotheses are genuinely exhausted.

Compared to Lab 01 this one added Layer 3 switching with SVIs instead of router-on-a-stick, Layer 2 anti-spoofing (snooping, DAI, port security) which Lab 01 didn't have at all, an explicit STP root bridge on Rapid-PVST+, and — the actual point of the exercise — a full zone-based security policy designed from scratch rather than assembled rule by rule.
