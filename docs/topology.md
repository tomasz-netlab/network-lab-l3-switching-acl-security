# topology.md — Lab 02: L3 Switching, ACL Segmentation & Security

## Overview

This lab builds an enterprise-style segmented network around a Layer 3 distribution switch. Four internal zones (HR, Finance, Management, Servers) are routed and firewalled against each other by extended ACLs, while an edge router connects the network to a simulated internet.

The design goal was a *coherent security scenario* rather than a collection of isolated features: every device, VLAN, and ACL exists to serve a stated policy. This document describes the physical and logical architecture and the reasoning behind the main design decisions. Addressing details live in [addressing-plan.md](addressing-plan.md); the traffic policy and ACL structure live in [acl-design.md](acl-design.md).

See `diagrams/topology.drawio.png` for the full topology diagram.

---

## Device Inventory

| Device | Model (Packet Tracer) | Role |
| ---------- | --------------------- | ---- |
| R2 | Cisco ISR 4331 | ISP / internet simulation — upstream of the edge |
| R1 | Cisco ISR 4331 | Edge router — WAN link, edge ACL, default routing |
| SW-DISTRO | Cisco Catalyst 3650-24PS | Layer 3 distribution switch — inter-VLAN routing via SVIs, root bridge, zone ACLs, DHCP relay |
| SW1 | Cisco Catalyst 2960-24TT | Layer 2 access — HR and Finance hosts, full L2 security stack |
| SW2 | Cisco Catalyst 2960-24TT | Layer 2 access — Admin workstation |
| Server-1 | Server-PT | Internal services — DHCP, DNS, SMTP/POP3, NTP, Syslog |
| Internet-WWW-Server | Server-PT | Public web server behind R2 (HTTPS) |
| PC1-HR | PC-PT | HR user host (DHCP) |
| PC2-FINANCE | PC-PT | Finance user host (DHCP) |
| Admin-PC | PC-PT | Administrative workstation (static, SSH source) |

The specific models matter: the Catalyst 3650 is a multilayer (L3) switch, which is what makes SVI-based inter-VLAN routing possible. The 2960s are pure L2 access switches. Using the real model names keeps the simulation representative of the equipment these configurations would run on.

---

## Security Zones

The network is divided into five zones. The four internal zones map one-to-one onto VLANs and SVIs on SW-DISTRO; the fifth is the simulated internet beyond the edge.

| Zone | VLAN | Role | Internet access |
| ---------- | ---- | ---- | --------------- |
| HR | 10 | End users — the only internal zone allowed outbound HTTPS | Yes (HTTPS only) |
| FINANCE | 20 | End users — fully isolated from the internet | No |
| MANAGEMENT | 101 | Administrative plane — SSH source, full reach into Servers | No |
| SERVERS | 123 | Central services (DHCP/DNS/mail/NTP/Syslog) | Return traffic only |
| INTERNET | — | Simulated public network (R2 + web server) | — |

The zone boundaries are the point of enforcement. Because all four internal VLANs are routed through SW-DISTRO's SVIs, every packet crossing from one zone to another passes through an inbound ACL on the source zone's SVI. Traffic that stays *within* a VLAN is pure L2 switching and never touches an ACL — the ACLs govern inter-zone movement, which is exactly where the policy needs to apply. The full permit/deny policy between these zones is documented in [acl-design.md](acl-design.md).

---

## Architectural Decisions

### Inter-VLAN routing via L3 switch SVIs (not router-on-a-stick)

Lab 01 used router-on-a-stick — a single router interface split into subinterfaces, one per VLAN, with an 802.1Q trunk carrying all VLANs to the router. That works, but the trunk is a single bottleneck and the router does all the routing.

Lab 02 moves inter-VLAN routing into the distribution switch itself. SW-DISTRO runs `ip routing` and has a Switch Virtual Interface (SVI) for each VLAN, acting as the default gateway for that segment. Routing happens in the switch's hardware (ASIC), close to the hosts, and the uplink to R1 is a dedicated **routed port** (`no switchport`) carrying only traffic that actually needs to leave the site — not a VLAN trunk.

This is the natural progression from Lab 01: same goal (inter-VLAN routing), more scalable and more representative of how a real distribution layer is built. It also creates the right place to enforce zone policy — the SVIs — which is where all four zone ACLs are applied inbound.

### Distribution/access split with an STP triangle

SW-DISTRO is the distribution layer; SW1 and SW2 are the access layer. The three switches are cabled in a triangle: SW-DISTRO↔SW1, SW-DISTRO↔SW2, and SW1↔SW2.

The triangle is deliberate — it provides a redundant path. If either uplink to SW-DISTRO fails, the access switches can still reach the distribution layer through the SW1↔SW2 base. The cost of that redundancy is a physical Layer 2 loop, which is exactly what Spanning Tree Protocol exists to manage. All three switches run **Rapid-PVST+** for faster convergence than legacy PVST+.

SW-DISTRO is configured as the **root bridge for the production VLANs** (10, 20, 101, 123) with an explicit priority of 24576. The priority is set in steps of 4096 because the Extended System ID reserves the lower 12 bits of the Bridge ID for the VLAN number — a value of 24576 is what makes SW-DISTRO win the root election for those VLANs. With the root at the distribution layer, both of its downlinks stay forwarding, and STP blocks one end of the SW1↔SW2 base to break the loop. In this build, SW1's Gi0/2 end settles into blocking for the production VLANs while SW2's end stays designated — verified with `show spanning-tree`.

One detail worth noting: VLAN 99 (native) is deliberately *not* included in the root-priority command, so SW-DISTRO is not the root for VLAN 99. Its STP instance therefore builds a different tree, and a different port ends up blocking (on SW1, Gi0/1 blocks and Gi0/2 forwards for VLAN 99 — the mirror image of the production VLANs). This is a direct consequence of PVST+ running an independent spanning tree per VLAN: change the root for some VLANs and not others, and the blocked links differ per VLAN. Since VLAN 99 carries no host traffic, this has no functional impact — but it's a clear illustration of how per-VLAN STP actually behaves.

The link stays physically connected and ready; if an active path drops, STP converges and brings it into forwarding.

### Edge router plus separate internet simulation

R1 is the edge — it owns the WAN link, the edge ACL that filters traffic arriving from outside (`ACL_INTERNET_IN`), and the default route toward the internet. R2 sits beyond R1 and simulates an ISP, with a dedicated public web server (Internet-WWW-Server) hanging off it so that "internet access" can be tested against a real HTTPS service rather than an empty address.

Separating the edge (R1) from the simulated internet (R2 + web server) keeps the trust boundary clean: everything R1-and-inward is the organisation's network under policy; everything beyond R1 is untrusted. The `ACL_INTERNET_IN` on R1's WAN interface is the gate — it permits only the specific return traffic the internal policy expects (established HTTPS back to HR, DNS replies, inbound SMTP to the server) and denies everything else.

### Store-and-forward mail through Server-1

Mail is never sent directly between user zones. HR and Finance both talk only to Server-1, which acts as the mail hub (SMTP/POP3). This store-and-forward model means the ACLs never need to permit direct HR↔Finance traffic — the two user zones stay fully isolated from each other, and all mail flows through a single controlled point. It also mirrors how real mail infrastructure works: clients talk to a server, not to each other.

### Static infrastructure, dynamic users

The Management and Servers zones are statically addressed; only the HR and Finance user zones use DHCP (relayed by SW-DISTRO to Server-1). Keeping infrastructure static gives predictable addresses for ACLs and management, and has a security side effect: with no DHCP in those zones there is nothing for DHCP Snooping or Dynamic ARP Inspection to protect. Those controls are therefore applied only on SW1, where the HR and Finance DHCP clients connect. SW2 — whose only access host is the statically-addressed Admin-PC — runs no snooping or DAI at all: with an empty binding table there would be nothing to inspect, and DAI would in fact break the static host unless a manual ARP ACL were added (which Packet Tracer does not support). SW2's access port is instead protected by port-security, PortFast and BPDUGuard, which is the appropriate control set for a static host.

---

## What This Topology Demonstrates

- Layer 3 switching with SVIs as the inter-VLAN routing and policy-enforcement layer
- Zone-based segmentation enforced by inbound extended ACLs
- Redundant distribution/access design with STP loop management
- A clean trust boundary between the internal network and a simulated internet
- Security controls placed where they are actually needed, not applied uniformly for their own sake

The next document, [acl-design.md](acl-design.md), turns these zones and this policy into the concrete ACL structure that enforces them.
