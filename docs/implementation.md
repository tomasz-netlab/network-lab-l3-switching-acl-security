# implementation.md — Lab 02: Implementation

## Overview

This document explains *how* the design is implemented on the devices, focusing on the two things that are the actual subject of this lab: the **ACLs** that enforce the zone policy, and the **Layer 2 access security** that protects the edge of the network. Baseline IOS hardening (SSH, banners, password policy, `login block-for`, exec-timeouts) is carried over from Lab 01 and is not repeated here — it is present in the full device configs under `configs/`.

Two things are deliberately new compared to Lab 01, and are called out where they appear:
- **Layer 2 anti-spoofing** — DHCP Snooping and Dynamic ARP Inspection, which Lab 01 did not have.
- **Spanning Tree** — Rapid-PVST+ with an explicitly configured root bridge, where Lab 01 left STP at its defaults.

The complete rule-by-rule ACL specification lives in [acl-design.md](acl-design.md); this document shows the commands that put it into effect and the reasoning behind how they are applied. Full configs are in `configs/`.

---

## Part 1 — Layer 3 Switching (the foundation for the ACLs)

Before any ACL can enforce zone policy, the zones have to be *routed* — and in this lab that routing happens on the distribution switch, not a router. SW-DISTRO runs `ip routing` and has an SVI per VLAN:

```
ip routing
!
interface Vlan10
 ip address 192.168.10.1 255.255.255.224
interface Vlan20
 ip address 192.168.20.1 255.255.255.224
interface Vlan101
 ip address 10.0.100.1 255.255.255.224
interface Vlan123
 ip address 10.0.120.1 255.255.255.240
```

The uplink to R1 is a **routed port**, not a trunk — `no switchport` turns the physical interface into a Layer 3 link:

```
interface GigabitEthernet1/0/1
 no switchport
 ip address 10.0.1.2 255.255.255.252
```

This matters for the ACLs because each SVI is the point where traffic *leaves* its zone and enters the routing engine. That is exactly where an inbound ACL can enforce "what may this zone originate" — so the SVIs are both the routing layer and the policy-enforcement layer. (This is the main architectural change from Lab 01's router-on-a-stick; the reasoning is in [topology.md](topology.md).)

---

## Part 2 — ACL Implementation (the core)

This is the heart of the lab. Each ACL is built as a named extended list and then applied inbound on the interface that faces the source zone. The pattern is always the same: define the list, then bind it with `ip access-group <name> in`.

The full rules are specified in [acl-design.md](acl-design.md) — here the focus is on *how* each is implemented and the mechanisms that make it work.

### Applying an ACL to an SVI

For the four zone ACLs, the list is applied inbound on the zone's SVI. Inbound on an SVI = traffic entering the routing engine from that VLAN, i.e. what the zone is sending out:

```
ip access-list extended ACL_HR_IN
 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 echo-reply
 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 ttl-exceeded
 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 unreachable
 permit udp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 53
 permit udp 192.168.10.0 0.0.0.31 eq 68 10.0.120.0 0.0.0.15 eq 67
 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 25
 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 110
 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 143
 deny ip 192.168.10.0 0.0.0.31 10.0.0.0 0.255.255.255
 deny ip 192.168.10.0 0.0.0.31 172.16.0.0 0.15.255.255
 deny ip 192.168.10.0 0.0.0.31 192.168.0.0 0.0.255.255
 permit tcp 192.168.10.0 0.0.0.31 any eq 443
 deny ip any any
!
interface Vlan10
 ip access-group ACL_HR_IN in
```

The same pattern applies to ACL_MGMT_IN (SVI 101), ACL_FINANCE_IN (SVI 20), and ACL_SERVERS_IN (SVI 123).

### ACL_INTERNET_IN — applied on a serial interface, not an SVI

The edge ACL is applied on R1's WAN serial interface, filtering everything arriving from the simulated internet:

```
ip access-list extended ACL_INTERNET_IN
 permit tcp any host 10.0.120.2 eq 25
 permit udp any eq 53 host 10.0.120.2
 permit tcp any eq 443 192.168.10.0 0.0.0.31 established
 deny ip any any
!
interface Serial0/1/0
 ip access-group ACL_INTERNET_IN in
```

This is a useful contrast: the same `ip access-group ... in` command binds an ACL to whatever interface faces the traffic you want to filter — an SVI for a zone, a serial interface for the WAN. The edge ACL only needs to work because the WAN routing is in place (static routes on R1 toward the internal networks and a default route toward R2); without return routes the permitted replies would have nowhere to go.

### Mechanisms that run through every ACL

**`established` for TCP return traffic.** Return traffic for TCP sessions is matched with `established`, which only matches segments with ACK/RST set — replies inside a session the client already opened. The internet cannot open new inbound connections to HR; it can only answer HR's outbound HTTPS:

```
permit tcp any eq 443 192.168.10.0 0.0.0.31 established
```

**Source-port matching for UDP.** UDP has no session state, so `established` doesn't apply. UDP return traffic (DNS replies, DHCP) is matched by the server's *source* port instead:

```
permit udp any eq 53 host 10.0.120.2        ! DNS reply, source port 53
```

**ICMP by type.** Rather than a blanket `permit icmp`, each direction permits the specific type it needs — `echo` outbound from Management, and `echo-reply` / `ttl-exceeded` / `unreachable` as the return. This keeps ping usable for diagnostics without opening ICMP wide at the trust boundaries.

**RFC 1918 deny before a broad internet permit.** Any rule that permits to `any` (HR's HTTPS, the server's DNS/SMTP relay) is preceded by three RFC 1918 deny rules. Without them, `any` would also match the internal networks and the broad permit could leak internal traffic. The order — deny internal, then permit internet — is what makes an `any` permit safe:

```
 deny ip 192.168.10.0 0.0.0.31 10.0.0.0 0.255.255.255
 deny ip 192.168.10.0 0.0.0.31 172.16.0.0 0.15.255.255
 deny ip 192.168.10.0 0.0.0.31 192.168.0.0 0.0.255.255
 permit tcp 192.168.10.0 0.0.0.31 any eq 443
```

### ACL_MGMT_VTY — restricting SSH by source

Separate from the zone SVIs, a standard ACL restricts remote SSH to the Management subnet, applied with `access-class` on the VTY lines of every managed device:

```
ip access-list standard ACL_MGMT_VTY
 permit 10.0.100.0 0.0.0.31
 deny any
!
line vty 0 15
 access-class ACL_MGMT_VTY in
 transport input ssh
```

`access-class` is the VTY-specific equivalent of `ip access-group` — it filters who can open a management session, regardless of which zone the traffic transits. This is the one control applied identically on R1, SW-DISTRO, SW1 and SW2.

---

## Part 3 — Layer 2 Access Security

This is the second pillar of the lab and the biggest step up from Lab 01, which had only basic IOS hardening and no Layer 2 anti-spoofing. Here the access layer gets a full set of L2 controls — and, just as importantly, those controls are placed *only where they protect something*.

### Port Security

On every access port connecting an end host, port-security limits the port to a single learned MAC and reacts to violations:

```
interface FastEthernet0/2
 switchport mode access
 switchport access vlan 10
 switchport port-security
 switchport port-security maximum 1
 switchport port-security mac-address sticky
 switchport port-security violation restrict
```

`sticky` learns the connected host's MAC and writes it into the running config; `violation restrict` drops traffic from any other MAC and logs it, without shutting the port down. `restrict` is chosen for user ports (rather than `shutdown`) so an accidental device swap doesn't hard-disable the port and generate a support ticket — the trade-off between stability and strictness is deliberate, and depends on a working Syslog to be useful.

### DHCP Snooping and Dynamic ARP Inspection — SW1 only

This is the anti-spoofing layer Lab 01 didn't have. DHCP Snooping protects against rogue DHCP servers and builds a binding table (IP↔MAC↔port↔VLAN); Dynamic ARP Inspection reads that table to catch ARP spoofing. They are configured on SW1:

```
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
ip arp inspection vlan 10,20
!
interface range GigabitEthernet0/1 - 2
 ip dhcp snooping trust
 ip arp inspection trust
```

The uplinks are set `trust`; the access ports stay untrusted by default. `no ip dhcp snooping information option` disables Option 82, which a Layer 2 switch shouldn't be inserting — without it, the server can drop the requests.

**Why only SW1, and nowhere else — this is the key decision.** These controls protect *DHCP clients on access ports*. SW1 is the only switch with dynamic hosts: HR and Finance connect there and pull addresses via DHCP. So SW1 is where snooping has clients to protect and where DAI has a binding table to work from.

- **SW2** has one access host — the Admin-PC — and it is *statically* addressed. With no DHCP traffic, snooping would build an empty binding table and protect nothing. Worse, DAI on a static host would drop its ARP (no binding exists) unless a manual ARP ACL were added — which Packet Tracer doesn't support. So SW2 runs neither: applying them would add cost and actively break the static host. Its access port is protected by port-security, PortFast and BPDUGuard instead, which is the correct control set for a static host.
- **SW-DISTRO and the routers** are not access-layer devices — no end hosts pull DHCP on their ports (Server-1 is a static DHCP *server*, not a client), so there is nothing for snooping/DAI to inspect there either.

The principle: a security control belongs where it protects something. Snooping and DAI on SW2 would be cost without benefit — defense-in-depth means the right control in the right place, not every control everywhere.

### PortFast and BPDUGuard

Access ports use PortFast (skip STP listening/learning for faster host connectivity) paired with BPDUGuard (err-disable the port if it ever receives a BPDU — i.e. if someone plugs in a switch where a host should be):

```
interface FastEthernet0/2
 spanning-tree portfast
 spanning-tree bpduguard enable
```

These are applied only on access ports, never on the trunks — PortFast on a switch-to-switch link would defeat STP loop protection.

### Trunk hardening and unused ports

Trunks are locked down against VLAN-hopping: an explicit native VLAN (99, which carries no hosts), an allowed-VLAN list, and DTP disabled:

```
interface range GigabitEthernet0/1 - 2
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20,101,123
 switchport nonegotiate
```

The `allowed vlan` list means the trunk carries only the four production VLANs — a frame in any other VLAN is dropped. The native VLAN is set to an empty VLAN 99 (not the default VLAN 1) and is deliberately kept off the allowed list, closing the double-tagging path. `switchport nonegotiate` turns off DTP so the port can't be negotiated into an unexpected mode. Unused ports are parked in a black-hole VLAN and shut:

```
interface range FastEthernet0/3 - 24
 switchport access vlan 999
 shutdown
```

---

## Context (not the subject of this lab, but part of the build)

**Spanning Tree.** Unlike Lab 01, which left STP at defaults, this lab runs **Rapid-PVST+** and sets SW-DISTRO as the **explicit root bridge** for the production VLANs:

```
spanning-tree mode rapid-pvst
spanning-tree vlan 10,20,101,123 priority 24576
```

The priority is set in steps of 4096 (the Extended System ID reserves the lower bits for the VLAN number), and 24576 is low enough to win the root election. With the root at the distribution layer, STP blocks one leg of the SW1↔SW2 base to break the triangle loop. See [topology.md](topology.md) for the details.

**Server-1** provides the internal services that the ACLs reference as traffic destinations: DHCP (with the HR/Finance pools, reached via `ip helper-address` relay on the SVIs), DNS, mail (SMTP/POP3), NTP and Syslog. It is a static host at 10.0.120.2 — it is the DHCP *server*, not a client.

**Internet-WWW-Server** sits behind R2 and exists so that "HR can reach the internet over HTTPS" can be tested against a real service (an HTTPS server at 203.0.113.2) rather than an empty address — it is the target that makes the F-02 / F-10 flows verifiable.

---

The full, unabridged device configurations are in `configs/`. The tests that confirm each ACL and control behaves as designed are in [validation.md](validation.md).
