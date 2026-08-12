[addressing-plan.md](https://github.com/user-attachments/files/30972419/addressing-plan.md)
# addressing-plan.md — Lab 02: L3 Switching, ACL Segmentation & Security

## Overview

This document defines the complete IP addressing scheme for Lab 02, including VLAN subnets, static host assignments, point-to-point links, DHCP scopes, and physical port mapping.

Unlike Lab 01, the addressing here was designed from scratch rather than reused — one of the goals of this lab was to practice deliberate subnet planning. The hardening baseline (SSH, banners, password policy, timeouts) was carried over from Lab 01, but every address in this topology was planned specifically for this scenario.

The core design decision is a clear split between two address spaces: `192.168.x.x` for user zones and `10.0.x.x` for infrastructure. The reasoning behind this split is documented in the Design Rationale section below.

---

## Address Space Split

| Space | Range | Purpose |
| ----- | ----- | ------- |
| `192.168.x.x` | User zones | HR and Finance — end-user segments |
| `10.0.x.x` | Infrastructure | Management, Servers, and point-to-point links |

Reading any address in this topology immediately reveals which category it belongs to. This is a deliberate, self-documenting convention — a packet sourced from `192.168.x.x` is a user; anything in `10.0.x.x` is infrastructure.

---

## VLAN Subnets and Gateways

Each user and infrastructure VLAN is routed through a Switch Virtual Interface (SVI) on SW-DISTRO, which acts as the default gateway for that segment.

| VLAN | Name | Subnet | Mask | SVI / Gateway | Usable Hosts |
| ---- | ---------- | ---------------- | ---- | ------------- | ------------ |
| 10 | HR | 192.168.10.0 | /27 | 192.168.10.1 | .2 – .30 |
| 20 | FINANCE | 192.168.20.0 | /27 | 192.168.20.1 | .2 – .30 |
| 101 | MANAGEMENT | 10.0.100.0 | /27 | 10.0.100.1 | .2 – .30 |
| 123 | SERVERS | 10.0.120.0 | /28 | 10.0.120.1 | .2 – .14 |
| 99 | NATIVE | — | — | none | — |
| 999 | UNUSED | — | — | none | — |

VLAN 99 (native) and VLAN 999 (unused) carry no IP addressing by design. VLAN 99 exists solely to absorb untagged frames on trunks; VLAN 999 is a black-hole VLAN for disabled ports. Neither has a gateway or any host.

---

## Static Host Assignments

| Device | VLAN | IP Address | Mask | Gateway | Role |
| -------- | ---- | ------------ | ---- | ------------ | ---------------------------- |
| SW-DISTRO SVI 10 | 10 | 192.168.10.1 | /27 | — | HR gateway |
| SW-DISTRO SVI 20 | 20 | 192.168.20.1 | /27 | — | Finance gateway |
| SW-DISTRO SVI 101 | 101 | 10.0.100.1 | /27 | — | Management gateway |
| SW-DISTRO SVI 123 | 123 | 10.0.120.1 | /28 | — | Servers gateway |
| Server-1 | 123 | 10.0.120.2 | /28 | 10.0.120.1 | DHCP + DNS + SMTP/POP3 + NTP + Syslog |
| Admin-PC | 101 | 10.0.100.10 | /27 | 10.0.100.1 | Administrative workstation (SSH source) |
| SW1 SVI 101 | 101 | 10.0.100.2 | /27 | 10.0.100.1 | SW1 management |
| SW2 SVI 101 | 101 | 10.0.100.3 | /27 | 10.0.100.1 | SW2 management |

Server-1 consolidates all network services (DHCP, DNS, mail, NTP, Syslog) on a single host. In a production environment these would typically be separated, but for the scope of this lab a single service host keeps the topology focused on the segmentation and ACL objectives rather than service redundancy.

The two access switches (SW1, SW2) are Layer 2 devices. They receive a management IP on an SVI in VLAN 101 and use `ip default-gateway` to reach anything outside their local segment, since they perform no routing of their own.

---

## Point-to-Point Links

| Link | Subnet | Mask | Side A | Side B |
| ----------------------- | ------------ | ---- | -------------------- | ---------------------- |
| R1 ↔ SW-DISTRO (routed) | 10.0.1.0 | /30 | R1 → 10.0.1.1 | SW-DISTRO → 10.0.1.2 |
| R1 ↔ R2 (WAN / serial) | 172.16.0.0 | /30 | R2 → 172.16.0.1 | R1 → 172.16.0.2 |
| R2 ↔ Server_WWW | 203.0.113.0 | /30 | R2 → 203.0.113.1 | Server_WWW → 203.0.113.2 |

All inter-device links use /30 subnets — the standard for point-to-point connections, providing exactly two usable addresses and wasting no address space.

The R1 ↔ SW-DISTRO link is a **routed port** (`no switchport` on the SW-DISTRO side), not a trunk. Since SW-DISTRO handles all inter-VLAN routing internally via its SVIs, this uplink only needs to carry routed traffic between the distribution switch and the edge router — no VLAN tagging is required.

The `203.0.113.0/24` range is IANA-reserved TEST-NET-3, a documentation prefix that will never appear as a real routable prefix on the internet. Using it makes the "internet" side of this lab safe to publish publicly. The same convention was used in Lab 01.

---

## DHCP Scopes

Two DHCP pools are configured on Server-1. Requests from the user VLANs reach the server via `ip helper-address 10.0.120.2` configured on the HR and Finance SVIs on SW-DISTRO (DHCP relay).

| Pool | Subnet | Mask | Start IP | Gateway | DNS |
| -------------- | ------------- | ---- | -------------- | ------------ | ---------- |
| VLAN10_HR | 192.168.10.0 | /27 | 192.168.10.2 | 192.168.10.1 | 10.0.120.2 |
| VLAN20_FINANCE | 192.168.20.0 | /27 | 192.168.20.2 | 192.168.20.1 | 10.0.120.2 |

Both the Management and Servers zones use static addressing only — no DHCP. This is a deliberate choice: infrastructure devices should have predictable, fixed addresses, and it also means DHCP Snooping and Dynamic ARP Inspection are not required in those VLANs (there is no DHCP traffic to protect and no dynamic bindings to build).

---

## Physical Port Mapping

### Inter-device links

| Link | A-side port | B-side port | Type |
| ------------------------ | ----------------- | ---------------- | -------------- |
| R1 ↔ SW-DISTRO | R1 Gi0/0/0 | SW-DISTRO Gi1/0/1 | Routed (L3) |
| R1 ↔ R2 (WAN) | R1 Se0/1/0 | R2 Se0/1/0 | Serial |
| SW-DISTRO ↔ SW1 | SW-DISTRO Gi1/0/2 | SW1 Gi0/1 | 802.1Q trunk |
| SW-DISTRO ↔ SW2 | SW-DISTRO Gi1/0/3 | SW2 Gi0/1 | 802.1Q trunk |
| SW1 ↔ SW2 | SW1 Gi0/2 | SW2 Gi0/2 | 802.1Q trunk |
| SW-DISTRO ↔ Server-1 | SW-DISTRO Gi1/0/24 | Server-1 NIC | Access VLAN 123 |
| R2 ↔ Server_WWW | R2 Gi0/0/0 | Server_WWW NIC | Routed (L3) |

### Host access ports

| Host | Switch port | VLAN |
| ----------- | ----------- | ---- |
| PC1-HR | SW1 Fa0/2 | 10 |
| PC2-FINANCE | SW1 Fa0/1 | 20 |
| Admin-PC | SW2 Fa0/24 | 101 |

The SW-DISTRO ↔ SW1 ↔ SW2 links form a triangle, creating a physical Layer 2 loop that is managed by STP. SW-DISTRO is configured as the root bridge for all VLANs, so STP blocks one leg of the triangle to prevent a broadcast storm while retaining redundancy.

---

## Design Rationale

### Why split user and infrastructure address spaces

The `192.168.x.x` / `10.0.x.x` split is not cosmetic. It gives every ACL a natural, readable structure: a rule that references `192.168.0.0/16` as a destination is talking about user zones, while `10.0.0.0/8` covers all infrastructure. When writing the "block all other internal traffic" rules (the RFC 1918 deny blocks in the zone ACLs), this separation makes the intent of each rule immediately clear to anyone reading the configuration months later.

### Why /27 for user VLANs

HR and Finance each use a /27 (30 usable hosts). This is sized for a small department — comfortably more than the handful of hosts in the lab, but not so large that the broadcast domain or potential blast radius is unnecessarily wide. Sizing subnets to their actual purpose is a discipline carried over from Lab 01.

### Why /28 for the Servers VLAN

The Servers VLAN uses a /28 (14 usable hosts). The lab has a single server, but /28 leaves room for a small number of additional service hosts without being oversized. A /29 was considered but rejected — it would leave almost no headroom, and infrastructure segments benefit from a small margin for growth.

### Why VLAN 101 and 123 rather than 100 and 120

Round-number VLAN IDs (100, 120) are predictable defaults that automated scanners often target first. Using 101 and 123 is a small, low-cost measure against predictability — the same reasoning that led to VLAN 101 for management in Lab 01. The security gain is modest, but avoiding defaults costs nothing.

### Third-octet-matches-VLAN convention

Where practical, the third octet echoes the VLAN number: VLAN 10 → `192.168.10.0`, VLAN 20 → `192.168.20.0`, VLAN 101 → `10.0.100.0`, VLAN 123 → `10.0.120.0`. This makes the addressing self-documenting — the VLAN a host belongs to is visible from its IP alone. (The Management and Servers octets, 100 and 120, are the closest round values to their VLAN IDs 101 and 123 that fit cleanly; the small mismatch between VLAN ID and third octet in those two cases is a minor cost of keeping both the VLAN-avoidance and the readability conventions.)
