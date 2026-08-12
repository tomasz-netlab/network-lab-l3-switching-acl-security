# acl-design.md — Lab 02: ACL Design

## Overview

This document is the core of Lab 02. It describes how the network's security policy is designed and then translated into concrete ACLs, in three layers:

1. **Security Zones & Policy** — what the zones are and the rules that govern traffic between them.
2. **Traffic Flow Matrix** — every permitted and denied flow between zones, each given a Flow ID (F-XX).
3. **ACL Deployment Map** — which ACL enforces which flows, on which interface and in which direction.
4. **ACL Detail Sheet** — every ACL written out rule by rule, each rule tied back to a Flow ID.

The point of this structure is that the policy is designed *before* any ACL is written. Each ACL rule exists to serve a specific flow in the matrix, and each flow exists to serve a stated policy. Reading top to bottom, you can trace any single ACL line back to the business reason it exists.

Addressing is defined in [addressing-plan.md](addressing-plan.md); the physical/logical architecture and zones are described in [topology.md](topology.md).

---

## Security Zones & Policy

Five zones, four of them internal VLANs routed by SW-DISTRO, plus the simulated internet beyond the edge router.

| Zone | VLAN | Subnet |
| ---------- | ---- | ------------- |
| HR | 10 | 192.168.10.0/27 |
| FINANCE | 20 | 192.168.20.0/27 |
| MANAGEMENT | 101 | 10.0.100.0/27 |
| SERVERS | 123 | 10.0.120.0/28 |
| INTERNET | — | 203.0.113.0/30 |

The policy rests on a few principles:

- **Zero-trust between user zones.** HR and Finance cannot talk to each other directly. There is no permit rule between them — mail and any other exchange goes through Server-1 (store-and-forward).
- **HR is the only zone with internet access, and only HTTPS.** Finance has no internet access at all. Management and Servers do not browse the internet; the server reaches out only to relay mail and resolve DNS.
- **Management is the administrative plane.** It has full service access to Servers and can ping the user zones. SSH to any device is permitted only from the Management subnet (enforced separately by `ACL_MGMT_VTY` on the VTY lines).
- **Servers is the service hub.** It answers DHCP, DNS, mail (SMTP/POP3/IMAP), NTP and Syslog for the internal zones, and relays mail/DNS outward. Its replies back to clients are return traffic, permitted explicitly because ACLs here are stateless.
- **Differentiated privilege between user zones.** Finance and HR do not have identical rights — for example, Finance has sFTP access to the server (file storage) while HR does not. This is deliberate, to model zones with genuinely different roles rather than copies of one template.
- **Everything not explicitly permitted is denied.** Every ACL ends in `deny ip any any`, and the intent is to log that final deny (see PT limitation note).

---

## Traffic Flow Matrix → Security Policy

Each row is one directional flow between two zones, with a Flow ID used throughout the rest of this document. Return traffic is treated as its own flow, because these ACLs are stateless and every direction must be permitted explicitly.

| Flow | Src Zone | Dst Zone | Service | Dst Port(s) | Action | Justification |
| ---- | -------- | -------- | ------- | ----------- | ------ | ------------- |
| F-01 | INTERNET | SERVERS | SMTP, DNS | TCP 25, UDP 53 | PERMIT | Inbound mail to the server; DNS replies |
| F-02 | INTERNET | HR | HTTPS | TCP 443 | PERMIT | Web replies to HR — established sessions only |
| F-03 | INTERNET | ANY | any | any | DENY | Block all inbound internet except F-01/F-02 |
| F-04 | MGMT | SERVERS | DNS, SMTP, POP3, IMAP, sFTP, NTP, Syslog, ICMP | TCP 22/25/110/143, UDP 53/123/514 | PERMIT | Full admin service access + ICMP echo |
| F-05 | MGMT | HR | ICMP | echo | PERMIT | Admin reachability check to HR |
| F-06 | MGMT | FINANCE | ICMP | echo | PERMIT | Admin reachability check to Finance |
| F-07 | FINANCE | MGMT | ICMP | echo-reply, ttl-exceeded, unreachable | PERMIT | ICMP return traffic to Management |
| F-08 | FINANCE | SERVERS | DNS, sFTP, DHCP, SMTP, POP3, IMAP | TCP 22/25/110/143, UDP 53/67 | PERMIT | DNS, DHCP, mail, and sFTP file storage (Finance only) |
| F-09 | HR | MGMT | ICMP | echo-reply, ttl-exceeded, unreachable | PERMIT | ICMP return traffic to Management |
| F-10 | HR | INTERNET | HTTPS | TCP 443 | PERMIT | Encrypted web access only (mail goes via server) |
| F-11 | HR | SERVERS | DNS, DHCP, SMTP, POP3, IMAP | TCP 25/110/143, UDP 53/67 | PERMIT | Mail and dynamic addressing (no sFTP for HR) |
| F-12 | SERVERS | INTERNET | DNS, SMTP | TCP 25, UDP 53 | PERMIT | Store-and-forward mail relay (HR) + DNS resolution |
| F-13 | SERVERS | FINANCE | DNS, DHCP, SMTP, POP3, IMAP, sFTP, ICMP | TCP 22/25/110/143, UDP 53/68 | PERMIT | Service replies to Finance + ICMP return |
| F-14 | SERVERS | HR | DNS, DHCP, SMTP, POP3, IMAP, ICMP | TCP 25/110/143, UDP 53/68 | PERMIT | Service replies to HR (no sFTP) + ICMP return |
| F-15 | SERVERS | MGMT | DNS, SMTP, POP3, IMAP, NTP, Syslog, ICMP | TCP 25/110/143, UDP 53/123/514 | PERMIT | Service replies to Admin-PC only + ICMP return |
| F-16 | ANY | MGMT | — | — | DENY | Block all to Management except the permitted flows above |
| F-17 | ANY | SERVERS | — | — | DENY | Block all to Servers except F-04 / F-08 / F-11 |
| F-99 | ANY | ANY | any | any | DENY | Implicit deny — the default at the end of every ACL |

A note on the DENY rows: F-03, F-16, F-17 and F-99 are not separate ACEs scattered through the config. They are the *closing* `deny ip any any` on each relevant ACL — the matrix names them so the "block everything else" intent is explicit and traceable, rather than left implicit.

---

## ACL Deployment Map

Where each ACL lives and which flows it enforces. All ACLs are applied **inbound** — an inbound ACL on a zone's SVI filters traffic *originating from* that zone as it enters the routing engine.

| ACL Name | Device | Interface | Direction | Flows enforced |
| ------------------ | ------------------------ | ------------- | --------- | -------------- |
| ACL_INTERNET_IN | R1 | Se0/1/0 | Inbound | F-01, F-02, F-03 |
| ACL_MGMT_IN | SW-DISTRO | SVI VLAN 101 | Inbound | F-04, F-05, F-06, F-16 |
| ACL_HR_IN | SW-DISTRO | SVI VLAN 10 | Inbound | F-09, F-10, F-11 |
| ACL_FINANCE_IN | SW-DISTRO | SVI VLAN 20 | Inbound | F-07, F-08 |
| ACL_SERVERS_IN | SW-DISTRO | SVI VLAN 123 | Inbound | F-12, F-13, F-14, F-15, F-17 |
| ACL_MGMT_VTY | R1, SW-DISTRO, SW1, SW2 | VTY 0 15 | Inbound | SSH to VTY from VLAN 101 only |

`ACL_MGMT_VTY` is a standard ACL applied with `access-class` on the VTY lines of every managed device. It is the single control that restricts remote SSH management to the Management subnet, independent of the zone SVIs.

---

## ACL Detail Sheet

Each ACL is written out below. The **Flow** column ties every rule back to the Traffic Flow Matrix. Port numbers are shown numerically for clarity (e.g. `eq 25` for SMTP). ICMP return traffic (echo-reply, ttl-exceeded, unreachable) is permitted explicitly because these ACLs are stateless — there is no automatic return path.

### ACL_INTERNET_IN — R1, Se0/1/0, inbound

Filters traffic arriving from the simulated internet. Only two things are allowed in: inbound mail to the server, and established HTTPS replies back to HR. Everything else is dropped at the edge.

| Seq | Action | Protocol | Source | Destination | Flags | Flow | Comment |
| --- | ------ | -------- | ------ | ----------- | ----- | ---- | ------- |
| 10 | permit | tcp | any | host 10.0.120.2 eq 25 | | F-01 | Inbound SMTP to server |
| 20 | permit | udp | any eq 53 | host 10.0.120.2 | | F-01 | DNS replies to server |
| 30 | permit | tcp | any eq 443 | 192.168.10.0/27 | established | F-02 | HTTPS replies to HR (established only) |
| 200 | deny | ip | any | any | log | F-03 | Block all other inbound internet traffic |

The `established` keyword on seq 30 lets return HTTPS traffic back to HR only if it belongs to a session HR initiated — the internet cannot open new connections inward. Seq 20 permits DNS as return traffic by matching the server's source port (53), since UDP has no `established` equivalent.

### ACL_MGMT_IN — SW-DISTRO, SVI VLAN 101, inbound

Traffic originating from the Management zone. Management can ping the user zones, has full service access to Servers, and nothing else.

| Seq | Action | Protocol | Source | Destination | Flags | Flow | Comment |
| --- | ------ | -------- | ---------------- | ----------------- | ----- | ---- | ------- |
| 10 | permit | icmp | 10.0.100.0/27 | 192.168.20.0/27 echo | | F-06 | Ping test to Finance |
| 20 | permit | icmp | 10.0.100.0/27 | 192.168.10.0/27 echo | | F-05 | Ping test to HR |
| 30 | permit | icmp | 10.0.100.0/27 | 10.0.120.0/28 echo | | F-04 | Ping test to Servers |
| 40 | permit | udp | 10.0.100.0/27 | 10.0.120.0/28 eq 53 | | F-04 | DNS |
| 50 | permit | tcp | 10.0.100.0/27 | 10.0.120.0/28 eq 25 | | F-04 | SMTP |
| 60 | permit | tcp | 10.0.100.0/27 | 10.0.120.0/28 eq 110 | | F-04 | POP3 |
| 70 | permit | tcp | 10.0.100.0/27 | 10.0.120.0/28 eq 143 | | F-04 | IMAP |
| 80 | permit | tcp | 10.0.100.0/27 | 10.0.120.0/28 eq 22 | | F-04 | sFTP / SSH |
| 90 | permit | udp | 10.0.100.0/27 | 10.0.120.0/28 eq 123 | | F-04 | NTP |
| 100 | permit | udp | 10.0.100.0/27 | 10.0.120.0/28 eq 514 | | F-04 | Syslog |
| 200 | deny | ip | any | any | log | F-16 | Block all other traffic from Management |

### ACL_HR_IN — SW-DISTRO, SVI VLAN 10, inbound

Traffic originating from HR. HR reaches the server for mail/DNS/DHCP, sends ICMP return traffic to Management, and is the only zone allowed outbound HTTPS. HR has **no** sFTP access.

| Seq | Action | Protocol | Source | Destination | Flags | Flow | Comment |
| --- | ------ | -------- | ---------------- | ----------------- | ----- | ---- | ------- |
| 10 | permit | icmp | 192.168.10.0/27 | 10.0.100.0/27 echo-reply | | F-09 | ICMP return to Management |
| 20 | permit | icmp | 192.168.10.0/27 | 10.0.100.0/27 ttl-exceeded | | F-09 | ICMP return to Management |
| 30 | permit | icmp | 192.168.10.0/27 | 10.0.100.0/27 unreachable | | F-09 | ICMP return to Management |
| 40 | permit | udp | 192.168.10.0/27 | 10.0.120.0/28 eq 53 | | F-11 | DNS |
| 50 | permit | udp | 192.168.10.0/27 eq 68 | 10.0.120.0/28 eq 67 | | F-11 | DHCP (client → server) |
| 60 | permit | tcp | 192.168.10.0/27 | 10.0.120.0/28 eq 25 | | F-11 | SMTP |
| 70 | permit | tcp | 192.168.10.0/27 | 10.0.120.0/28 eq 110 | | F-11 | POP3 |
| 80 | permit | tcp | 192.168.10.0/27 | 10.0.120.0/28 eq 143 | | F-11 | IMAP |
| 90 | deny | ip | 192.168.10.0/27 | 10.0.0.0/8 | | F-10 | Block internal before internet permit |
| 100 | deny | ip | 192.168.10.0/27 | 172.16.0.0/12 | | F-10 | Block internal before internet permit |
| 110 | deny | ip | 192.168.10.0/27 | 192.168.0.0/16 | | F-10 | Block internal before internet permit |
| 120 | permit | tcp | 192.168.10.0/27 | any eq 443 | | F-10 | HTTPS to internet |
| 200 | deny | ip | any | any | log | F-99 | Implicit deny |

The three RFC 1918 deny rules (seq 90–110) exist specifically to protect the broad internet permit that follows. Seq 120 permits HTTPS to `any` — but "any" would also match internal networks. Placing the RFC 1918 denies immediately before it ensures the permit applies only to genuine internet destinations. This ordering — deny internal, then permit internet — is what makes a broad `any` permit safe to use.

### ACL_FINANCE_IN — SW-DISTRO, SVI VLAN 20, inbound

Traffic originating from Finance. Finance reaches the server (including sFTP, which HR does not have) and sends ICMP return traffic to Management. Finance has **no internet access** — there is no HTTPS permit and no RFC 1918 block, because with no internet permit to protect, the closing `deny ip any any` already covers all outbound traffic.

| Seq | Action | Protocol | Source | Destination | Flags | Flow | Comment |
| --- | ------ | -------- | ---------------- | ----------------- | ----- | ---- | ------- |
| 10 | permit | icmp | 192.168.20.0/27 | 10.0.100.0/27 echo-reply | | F-07 | ICMP return to Management |
| 20 | permit | icmp | 192.168.20.0/27 | 10.0.100.0/27 ttl-exceeded | | F-07 | ICMP return to Management |
| 30 | permit | icmp | 192.168.20.0/27 | 10.0.100.0/27 unreachable | | F-07 | ICMP return to Management |
| 40 | permit | udp | 192.168.20.0/27 | 10.0.120.0/28 eq 53 | | F-08 | DNS |
| 50 | permit | tcp | 192.168.20.0/27 | 10.0.120.0/28 eq 22 | | F-08 | sFTP (Finance only) |
| 60 | permit | udp | 192.168.20.0/27 eq 68 | 10.0.120.0/28 eq 67 | | F-08 | DHCP (client → server) |
| 70 | permit | tcp | 192.168.20.0/27 | 10.0.120.0/28 eq 25 | | F-08 | SMTP |
| 80 | permit | tcp | 192.168.20.0/27 | 10.0.120.0/28 eq 110 | | F-08 | POP3 |
| 90 | permit | tcp | 192.168.20.0/27 | 10.0.120.0/28 eq 143 | | F-08 | IMAP |
| 200 | deny | ip | any | any | log | F-99 | Implicit deny — no internet for Finance |

### ACL_SERVERS_IN — SW-DISTRO, SVI VLAN 123, inbound

Traffic originating from the Servers zone — almost entirely *replies* to clients that initiated a request, plus the server's own outbound mail/DNS to the internet. TCP replies use `established`; UDP and ICMP replies are matched explicitly by source port / ICMP type. This is the largest ACL because the server answers four different zones.

**Replies to Management (F-15):**

| Seq | Action | Protocol | Source | Destination | Flags | Flow |
| --- | ------ | -------- | -------------- | ----------------- | ----------- | ---- |
| 10 | permit | icmp | 10.0.120.0/28 | 10.0.100.0/27 | | F-15 |
| 20 | permit | udp | 10.0.120.0/28 eq 53 | 10.0.100.0/27 | | F-15 |
| 30 | permit | tcp | 10.0.120.0/28 eq 25 | 10.0.100.0/27 | established | F-15 |
| 40 | permit | tcp | 10.0.120.0/28 eq 110 | 10.0.100.0/27 | established | F-15 |
| 50 | permit | tcp | 10.0.120.0/28 eq 143 | 10.0.100.0/27 | established | F-15 |
| 60 | permit | tcp | 10.0.120.0/28 eq 22 | 10.0.100.0/27 | established | F-15 |
| 70 | permit | udp | 10.0.120.0/28 eq 123 | 10.0.100.0/27 | | F-15 |

**Replies to Finance (F-13):**

| Seq | Action | Protocol | Source | Destination | Flags | Flow |
| --- | ------ | -------- | -------------- | ----------------- | ----------- | ---- |
| 80 | permit | udp | 10.0.120.0/28 eq 53 | 192.168.20.0/27 | | F-13 |
| 90 | permit | udp | 10.0.120.0/28 eq 67 | 192.168.20.0/27 eq 68 | | F-13 |
| 100 | permit | tcp | 10.0.120.0/28 eq 25 | 192.168.20.0/27 | established | F-13 |
| 110 | permit | tcp | 10.0.120.0/28 eq 110 | 192.168.20.0/27 | established | F-13 |
| 120 | permit | tcp | 10.0.120.0/28 eq 143 | 192.168.20.0/27 | established | F-13 |
| 130 | permit | tcp | 10.0.120.0/28 eq 22 | 192.168.20.0/27 | established | F-13 |
| 140 | permit | icmp | 10.0.120.0/28 | 192.168.20.0/27 | | F-13 |

**Replies to HR (F-14):**

| Seq | Action | Protocol | Source | Destination | Flags | Flow |
| --- | ------ | -------- | -------------- | ----------------- | ----------- | ---- |
| 150 | permit | udp | 10.0.120.0/28 eq 53 | 192.168.10.0/27 | | F-14 |
| 160 | permit | udp | 10.0.120.0/28 eq 67 | 192.168.10.0/27 eq 68 | | F-14 |
| 170 | permit | tcp | 10.0.120.0/28 eq 25 | 192.168.10.0/27 | established | F-14 |
| 180 | permit | tcp | 10.0.120.0/28 eq 110 | 192.168.10.0/27 | established | F-14 |
| 190 | permit | tcp | 10.0.120.0/28 eq 143 | 192.168.10.0/27 | established | F-14 |
| 200 | permit | icmp | 10.0.120.0/28 | 192.168.10.0/27 | | F-14 |

**Outbound to internet + closing deny (F-12, F-17):**

| Seq | Action | Protocol | Source | Destination | Flags | Flow | Comment |
| --- | ------ | -------- | -------------- | ----------------- | ----- | ---- | ------- |
| 391 | deny | ip | 10.0.120.0/28 | 10.0.0.0/8 | | F-12 | Block internal before internet permit |
| 392 | deny | ip | 10.0.120.0/28 | 172.16.0.0/12 | | F-12 | Block internal before internet permit |
| 393 | deny | ip | 10.0.120.0/28 | 192.168.0.0/16 | | F-12 | Block internal before internet permit |
| 394 | permit | udp | 10.0.120.0/28 | any eq 53 | | F-12 | DNS resolution to internet |
| 395 | permit | tcp | 10.0.120.0/28 | any eq 25 | | F-12 | SMTP relay to internet (store-and-forward) |
| 400 | deny | ip | any | any | log | F-17 | Block all other traffic from Servers |

Note the same RFC 1918 pattern as HR (seq 391–393 before the internet permits 394–395): the server relays DNS and mail outward to `any`, so the internal networks are blocked first to keep those broad permits internet-only. The store-and-forward model lives here — the server is the only host that sends SMTP to the internet, on behalf of HR.

---

## Design Decisions

### Stateless ACLs and explicit return traffic

These are extended ACLs on routed interfaces — stateless. Unlike a stateful firewall, they do not automatically permit the reply to a permitted request. Every direction is a separate flow with its own rules. This is why the server's replies (F-13/14/15) are a large block of explicit permits, and why ICMP return types (echo-reply, ttl-exceeded, unreachable) are permitted by name.

### `established` for TCP, source-port matching for UDP

TCP return traffic uses the `established` keyword — it matches segments with ACK/RST set, i.e. replies within a session the client already opened. UDP has no such state, so UDP return traffic (DNS replies, DHCP) is matched by the server's *source* port instead (e.g. `udp ... eq 53` as source). This distinction runs through the whole design: TCP replies → `established`; UDP replies → source-port match.

### ICMP handled by type, not blanket permit — with a deliberate exception

At the network edge and between user zones, ICMP is permitted by specific type in the direction it belongs: `echo` outbound from Management (the ping), and `echo-reply`, `ttl-exceeded`, `unreachable` as the return traffic from the pinged zone. This keeps ICMP usable for diagnostics without opening it wide where it matters most.

Inside the infrastructure, the choice is deliberately different. The server's ICMP replies (ACL_SERVERS_IN seq 10, 140, 200) use a *general* `permit icmp` without a type. This is a conscious trade-off: granular ICMP typing is worth the extra rules at the trust boundary (the edge, the user zones), but between the server and the internal zones it adds rule count for little security gain — the server and the internal planes already trust each other for service traffic. So the design is **granular ICMP at the boundary, general ICMP within the infrastructure** — matching the level of scrutiny to where the risk actually is.

### Inbound-on-SVI = "traffic from this zone"

Every zone ACL is applied inbound on that zone's SVI. Inbound on an SVI means traffic *entering the routing engine from that VLAN* — i.e. traffic the zone is sending out. So ACL_HR_IN governs what HR is allowed to send, ACL_SERVERS_IN governs what the server is allowed to send, and so on. This is the natural place to enforce "what may this zone originate."

### Differentiated privilege: Finance has sFTP, HR does not

Finance and HR are not copies of one template. Finance has sFTP (TCP 22) to the server for file storage; HR does not. This models two user zones with genuinely different roles, and demonstrates that the policy is per-zone rather than a single user-zone profile stamped twice.

### Broad permits are always preceded by RFC 1918 denies

Wherever a rule permits traffic to `any` (HR's HTTPS, the server's DNS/SMTP relay), three RFC 1918 deny rules precede it. Without them, "any" would include the internal networks, and the broad permit could accidentally allow internal traffic it was never meant to. Deny-internal-then-permit-internet is the safe ordering for any rule whose destination is `any`.

### Known limitations (Packet Tracer)

- **`log` keyword** — the closing `deny ip any any` is intended to be logged (`deny ip any any log`) for security visibility. Packet Tracer does not support the `log` keyword on ACEs, so it is documented here as intent; the deny itself functions normally.
- **IMAP (TCP 143)** — permitted in the ACLs as part of the mail policy, but Packet Tracer's EMAIL service implements only SMTP and POP3. The IMAP rules reflect the intended production posture; there is no IMAP server to test against in the simulator.
- **sFTP (TCP 22)** — SFTP shares port 22 with SSH. In this lab they are separated by policy and by Layer 3 destination (device management vs file storage on the server), not by port. On real equipment this would warrant tighter separation.

The full mapping of policy → matrix → deployment → rules is what makes this design auditable: any ACL line can be traced back to a Flow ID, and any flow back to a stated policy principle.
