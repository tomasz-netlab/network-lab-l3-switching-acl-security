[validation.md](https://github.com/user-attachments/files/31117392/validation.md)
# validation.md — Lab 02: Validation

All tests below were run on the completed lab in Cisco Packet Tracer. Outputs are copied verbatim from the device CLI or host command prompt. Supporting screenshots are in `evidence/`.

The tests are grouped in three parts: configuration verification (is the network built as designed), Layer 2 security verification (are the access-layer controls active), and policy tests (does the ACL policy actually permit and deny the right traffic). The last group is the point of the lab — a policy is only proven when both the permits *and* the denies behave as specified.

---

## Part A — Configuration Verification

### 1. VLAN configuration — SW-DISTRO

**Command:** `show vlan brief`

```
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    
10   HR                               active    
20   FINANCE                          active    
99   Native                           active    
101  Management                       active    
123  Servers                          active    Gig1/0/24
999  UNUSED                           active    Gig1/0/4, Gig1/0/5, Gig1/0/6, Gig1/0/7
                                                Gig1/0/8, Gig1/0/9, Gig1/0/10, Gig1/0/11
                                                Gig1/0/12, Gig1/0/13, Gig1/0/14, Gig1/0/15
                                                Gig1/0/16, Gig1/0/17, Gig1/0/18, Gig1/0/19
                                                Gig1/0/20, Gig1/0/21, Gig1/0/22, Gig1/0/23
                                                Gig1/1/1, Gig1/1/2, Gig1/1/3, Gig1/1/4
```

**Result:** ✅ All six VLANs created and active. Server-1 sits on Gi1/0/24 in VLAN 123. All unused ports are parked in the black-hole VLAN 999.

### 2. Interfaces and SVIs — SW-DISTRO

**Command:** `show ip interface brief`

```
Interface              IP-Address      OK? Method Status                Protocol 
GigabitEthernet1/0/1   10.0.1.2        YES NVRAM  up                    up 
GigabitEthernet1/0/2   unassigned      YES NVRAM  up                    up 
GigabitEthernet1/0/3   unassigned      YES NVRAM  up                    up 
GigabitEthernet1/0/24  unassigned      YES NVRAM  up                    up 
Vlan10                 192.168.10.1    YES manual up                    up 
Vlan20                 192.168.20.1    YES manual up                    up 
Vlan101                10.0.100.1      YES manual up                    up 
Vlan123                10.0.120.1      YES manual up                    up
```
*(unused ports omitted — all administratively down)*

**Result:** ✅ Gi1/0/1 carries an IP address directly, confirming it is a routed port (`no switchport`), not a trunk. All four SVIs are up with the addresses from the addressing plan.

### 3. Trunk configuration — SW-DISTRO

**Command:** `show interfaces trunk`

```
Port        Mode         Encapsulation  Status        Native vlan
Gig1/0/2    on           802.1q         trunking      99
Gig1/0/3    on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gig1/0/2    10,20,101,123
Gig1/0/3    10,20,101,123
```

**Result:** ✅ Mode is `on` rather than `desirable`/`auto`, confirming DTP is disabled (`switchport nonegotiate`). Native VLAN is 99 and — importantly — 99 is **not** in the allowed list, closing the double-tagging path.

### 4. Routing table — SW-DISTRO

**Command:** `show ip route`

```
Gateway of last resort is 10.0.1.1 to network 0.0.0.0

     10.0.0.0/8 is variably subnetted, 6 subnets, 4 masks
C       10.0.1.0/30 is directly connected, GigabitEthernet1/0/1
C       10.0.100.0/27 is directly connected, Vlan101
C       10.0.120.0/28 is directly connected, Vlan123
C       192.168.10.0/27 is directly connected, Vlan10
C       192.168.20.0/27 is directly connected, Vlan20
S*   0.0.0.0/0 [1/0] via 10.0.1.1
```
*(local /32 routes omitted)*

**Result:** ✅ All four zone subnets are directly connected via their SVIs — inter-VLAN routing is handled entirely by the L3 switch. Everything else follows the default route to R1.

### 5. Spanning Tree root bridge — SW-DISTRO

**Command:** `show spanning-tree vlan 10`

```
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     0002.17EC.48CE
             This bridge is the root

  Bridge ID  Priority    24586  (priority 24576 sys-id-ext 10)
             Address     0002.17EC.48CE

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/0/2          Desg FWD 4         128.2    P2p
Gi1/0/3          Desg FWD 4         128.3    P2p
```

**Result:** ✅ SW-DISTRO is the root bridge, running RSTP (Rapid-PVST+). The priority reads 24586 = 24576 (set by `root primary`) + 10 (Extended System ID carrying the VLAN number). Both downlinks are Designated/Forwarding, as expected on a root bridge.

### 6. Spanning Tree triangle — SW1

**Command:** `show spanning-tree vlan 10`

```
VLAN0010
  Spanning tree enabled protocol rstp
  Root ID    Priority    24586
             Address     0002.17EC.48CE
             Cost        4
             Port        25(GigabitEthernet0/1)

  Bridge ID  Priority    32778  (priority 32768 sys-id-ext 10)
             Address     0030.F2AA.43A7

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Fa0/2            Desg FWD 19        128.2    P2p
Gi0/1            Root FWD 4         128.25   P2p
Gi0/2            Altn BLK 4         128.26   P2p
```

**Result:** ✅ This is the proof that the triangle is managed. SW1 sees SW-DISTRO as root (not itself — its own priority is the 32768 default). Gi0/1 is the Root port toward the distribution layer; Gi0/2 (the SW1↔SW2 base of the triangle) is Alternate/Blocking, breaking the loop while remaining available for failover.

Confirmed from the other direction by `show interfaces trunk` on SW1, where Gi0/2 shows `none` under "Vlans in spanning tree forwarding state".

### 7. ACL definitions with hit counters — SW-DISTRO

**Command:** `show access-lists`

```
Extended IP access list ACL_FINANCE_IN
    5 permit icmp 192.168.20.0 0.0.0.31 host 192.168.20.1 echo (6 match(es))
    10 permit icmp 192.168.20.0 0.0.0.31 10.0.100.0 0.0.0.31 echo-reply (6 match(es))
    20 permit icmp 192.168.20.0 0.0.0.31 10.0.100.0 0.0.0.31 ttl-exceeded
    30 permit icmp 192.168.20.0 0.0.0.31 10.0.100.0 0.0.0.31 unreachable
    35 permit udp host 0.0.0.0 host 255.255.255.255 eq bootps (22 match(es))
    40 permit udp 192.168.20.0 0.0.0.31 10.0.120.0 0.0.0.15 eq domain (3 match(es))
    50 permit tcp 192.168.20.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 22
    60 permit udp 192.168.20.0 0.0.0.31 eq bootpc 10.0.120.0 0.0.0.15 eq bootps
    70 permit tcp 192.168.20.0 0.0.0.31 10.0.120.0 0.0.0.15 eq smtp
    80 permit tcp 192.168.20.0 0.0.0.31 10.0.120.0 0.0.0.15 eq pop3
    90 permit tcp 192.168.20.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 143
    100 deny ip any any (114 match(es))
Extended IP access list ACL_HR_IN
    5 permit icmp 192.168.10.0 0.0.0.31 host 192.168.10.1 echo (22 match(es))
    10 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 echo-reply (10 match(es))
    20 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 ttl-exceeded
    30 permit icmp 192.168.10.0 0.0.0.31 10.0.100.0 0.0.0.31 unreachable
    40 permit udp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq domain (18 match(es))
    45 permit udp host 0.0.0.0 host 255.255.255.255 eq bootps (60 match(es))
    50 permit udp 192.168.10.0 0.0.0.31 eq bootpc 10.0.120.0 0.0.0.15 eq bootps
    60 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq smtp
    70 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq pop3
    80 permit tcp 192.168.10.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 143
    90 deny ip 192.168.10.0 0.0.0.31 10.0.0.0 0.255.255.255 (24 match(es))
    100 deny ip 192.168.10.0 0.0.0.31 172.16.0.0 0.15.255.255
    110 deny ip 192.168.10.0 0.0.0.31 192.168.0.0 0.0.255.255 (28 match(es))
    120 permit tcp 192.168.10.0 0.0.0.31 any eq 443 (158 match(es))
    130 deny ip any any (68 match(es))
Standard IP access list ACL_MGMT_VTY
    10 permit 10.0.100.0 0.0.0.31
    20 deny any
Extended IP access list ACL_MGMT_IN
    10 permit icmp 10.0.100.0 0.0.0.31 192.168.20.0 0.0.0.31 echo (14 match(es))
    20 permit icmp 10.0.100.0 0.0.0.31 192.168.10.0 0.0.0.31 echo (18 match(es))
    30 permit icmp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 echo (22 match(es))
    40 permit udp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq domain (2 match(es))
    50 permit tcp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq smtp
    60 permit tcp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq pop3
    70 permit tcp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 143
    80 permit tcp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 22
    90 permit udp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 123 (2964 match(es))
    100 permit udp 10.0.100.0 0.0.0.31 10.0.120.0 0.0.0.15 eq 514 (48 match(es))
    200 deny ip any any (142 match(es))
Extended IP access list ACL_SERVERS_IN
    10 permit icmp 10.0.120.0 0.0.0.15 10.0.100.0 0.0.0.31 (10 match(es))
    20 permit udp 10.0.120.0 0.0.0.15 eq domain 10.0.100.0 0.0.0.31 (2 match(es))
    70 permit udp 10.0.120.0 0.0.0.15 eq 123 10.0.100.0 0.0.0.31 (2933 match(es))
    80 permit udp 10.0.120.0 0.0.0.15 eq domain 192.168.20.0 0.0.0.31 (3 match(es))
    140 permit icmp 10.0.120.0 0.0.0.15 192.168.20.0 0.0.0.31 (20 match(es))
    150 permit udp 10.0.120.0 0.0.0.15 eq domain 192.168.10.0 0.0.0.31 (18 match(es))
    155 permit udp 10.0.120.0 0.0.0.15 eq bootps 192.168.10.0 0.0.0.31 eq bootps (35 match(es))
    156 permit udp 10.0.120.0 0.0.0.15 eq bootps 192.168.20.0 0.0.0.31 eq bootps (16 match(es))
    200 permit icmp 10.0.120.0 0.0.0.15 192.168.10.0 0.0.0.31 (56 match(es))
    391 deny ip 10.0.120.0 0.0.0.15 10.0.0.0 0.255.255.255 (1036 match(es))
    393 deny ip 10.0.120.0 0.0.0.15 192.168.0.0 0.0.255.255 (8 match(es))
    400 deny ip any any (4 match(es))
```
*(ACL_SERVERS_IN abbreviated to rules with hits — full list in `configs/sw-distro-config.txt`)*

**Result:** ✅ All five ACLs are present and, crucially, **matching real traffic**. Hit counters prove enforcement rather than just configuration: 158 hits on the HR HTTPS permit, 114 on the Finance closing deny, and traffic on all three DHCP patterns (initial Discover from `0.0.0.0`, and the relay-to-server leg on 67→67).

### 8. ACL application on SVIs — SW-DISTRO

**Command:** `show ip interface vlan 10 | include access list` (same for VLANs 20, 101, 123)

```
  Outgoing access list is not set 
  Inbound  access list is not set 
```

**Result:** ⚠️ Reported as not set — but the ACLs **are** filtering, as the hit counters in test 7 and the policy tests in Part C prove.

> **Note — Packet Tracer limitation.** `ip access-group` applied to an SVI on a multilayer switch does not appear in `show running-config` or `show ip interface`, and stops filtering after a reload. Re-applying the command restores enforcement immediately. This is a documented Packet Tracer defect on the 3560/3650 platforms, reported on the Cisco Community forum. Compare with test 10 below, where the same command on a router interface displays normally — the defect is specific to SVIs.

### 9. Edge ACL with hit counters — R1

**Command:** `show access-lists`

```
Extended IP access list ACL_INTERNET_IN
    10 permit tcp any host 10.0.120.2 eq smtp
    20 permit udp any host 10.0.120.2 eq domain
    30 permit tcp any eq 443 192.168.10.0 0.0.0.31 established (6 match(es))
    200 deny ip any any (30 match(es))
Standard IP access list ACL_MGMT_VTY
    10 permit 10.0.100.0 0.0.0.31
    20 deny any
```

**Result:** ✅ The edge ACL is enforcing. Seq 30 shows return HTTPS traffic reaching HR — the `established` keyword working as intended, admitting replies to sessions HR initiated. Seq 200 shows 30 packets from the internet dropped at the perimeter.

### 10. ACL application on a router interface — R1

**Command:** `show ip interface Serial0/1/0 | include access list`

```
  Outgoing access list is not set
  Inbound  access list is ACL_INTERNET_IN
```

**Result:** ✅ Displayed correctly. This is the direct contrast to test 8: the identical `ip access-group ... in` command reports properly on a router interface but not on an SVI, confirming the Packet Tracer defect is scoped to SVIs on multilayer switches, not to the configuration itself.

### 11. Routing table — R1

**Command:** `show ip route`

```
Gateway of last resort is 172.16.0.1 to network 0.0.0.0

     10.0.0.0/8 is variably subnetted, 4 subnets, 4 masks
C       10.0.1.0/30 is directly connected, GigabitEthernet0/0/0
S       10.0.100.0/27 [1/0] via 10.0.1.2
S       10.0.120.0/28 [1/0] via 10.0.1.2
     172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C       172.16.0.0/30 is directly connected, Serial0/1/0
     192.168.10.0/27 is subnetted, 1 subnets
S       192.168.10.0/27 [1/0] via 10.0.1.2
     192.168.20.0/27 is subnetted, 1 subnets
S       192.168.20.0/27 [1/0] via 10.0.1.2
S*   0.0.0.0/0 [1/0] via 172.16.0.1
```
*(local /32 routes omitted)*

**Result:** ✅ Four explicit static routes to the internal zones via SW-DISTRO, plus a default route toward R2. Explicit per-subnet routes were chosen over a summary for clarity — see [addressing-plan.md](addressing-plan.md).

### 12. Interfaces — R1 and R2

**Command (R1):** `show ip interface brief`

```
GigabitEthernet0/0/0   10.0.1.1        YES NVRAM  up                    up 
Serial0/1/0            172.16.0.2      YES NVRAM  up                    up 
```

**Command (R2):** `show ip interface brief`

```
GigabitEthernet0/0/0   203.0.113.1     YES NVRAM  up                    up 
Serial0/1/0            172.16.0.1      YES NVRAM  up                    up 
```

**Result:** ✅ Both WAN and LAN links are up on both routers. R2's Gi0/0/0 carries the public-side /30 toward the internet web server.

---

## Part B — Layer 2 Security Verification (SW1)

### 13. DHCP Snooping status

**Command:** `show ip dhcp snooping`

```
Switch DHCP snooping is enabled
DHCP snooping is configured on following VLANs:
10,20
Insertion of option 82 is disabled
Option 82 on untrusted port is not allowed
Verification of hwaddr field is enabled
Interface                  Trusted    Rate limit (pps)
-----------------------    -------    ----------------
FastEthernet0/1            no         unlimited       
FastEthernet0/2            no         unlimited       
GigabitEthernet0/1         yes        unlimited       
GigabitEthernet0/2         yes        unlimited   
```

**Result:** ✅ Snooping is active on the two user VLANs. Trust is set only on the uplinks (Gi0/1, Gi0/2); both access ports remain untrusted, so a rogue DHCP server plugged into a user port would have its offers dropped. Option 82 insertion is disabled — correct for a Layer 2 switch that is not the relay agent. `Verification of hwaddr field is enabled` adds protection against DHCP starvation by comparing the frame source MAC with the CHADDR field.

### 14. Binding table

**Command:** `show ip dhcp snooping binding`

```
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  -----------------
00:07:EC:68:41:E6   192.168.10.2     0           dhcp-snooping  10    FastEthernet0/2
00:90:2B:29:BB:9E   192.168.20.2     0           dhcp-snooping  20    FastEthernet0/1
Total number of bindings: 2
```

**Result:** ✅ Both DHCP clients are bound: PC1-HR on Fa0/2 in VLAN 10, PC2-Finance on Fa0/1 in VLAN 20, each holding the first address in its pool. This table is the source of truth that DAI depends on.

### 15. Binding table persistence across reload

**Command:** `show ip dhcp snooping database` *(taken after a switch reload)*

```
Agent URL : flash:/dhcp-snooping.db
Write delay Timer : 30 seconds
Agent Running : No
Total Attempts       :        3   Startup Failures :        0
Successful Transfers :        0   Failed Transfers :        0
Successful Reads     :        1   Failed Reads     :        0
Successful Writes    :        2   Failed Writes    :        0
Media Failures       :        0
```

**Result:** ✅ `Successful Reads: 1` after boot, with the binding table (test 14) populated without any client renewing its lease — the database agent restored the bindings from flash.

This control was added *because* testing exposed the problem: the binding table lives in RAM, so after the first reload it was empty, leaving DAI with nothing to validate against until every host re-ran DHCP. `Agent Running: No` simply means no transfer is in progress at the moment of the query — the agent wakes only to read at boot or write after the delay timer.

> **Note — Packet Tracer limitation.** `Last Succeded Time` remains `None` even after successful transfers; the field is not populated in the simulator although the counters increment correctly.

### 16. Dynamic ARP Inspection

**Command:** `show ip arp inspection`

```
Source Mac Validation      : Disabled
Destination Mac Validation : Disabled
IP Address Validation      : Disabled
 Vlan     Configuration    Operation   ACL Match          Static ACL
 ----     -------------    ---------   ---------          ----------
   10     Enabled          Active
   20     Enabled          Active
 Vlan      Forwarded        Dropped     DHCP Drops      ACL Drops
 ----      ---------        -------     ----------      ---------
   10              0              0              0              0
   20              0              0              0              0
```

**Result:** ⚠️ Configuration is correct and active on both user VLANs, but the counters remain at zero and enforcement could not be demonstrated.

The three `Disabled` lines at the top are the *optional* extra validations (`ip arp inspection validate src-mac | dst-mac | ip`), which are off by default — DAI still performs its core check of sender IP/MAC against the binding table.

> **Note — Packet Tracer limitation, verified by testing.** A deliberate spoofing test was run: PC1-HR was reassigned a static address (`192.168.10.5`) while keeping its original MAC, creating a mismatch against the binding table entry (`192.168.10.2`). On real hardware DAI would drop the resulting ARP. In Packet Tracer the ping succeeded and no counter incremented. Three configuration hypotheses were eliminated first — trust state on the access ports (only the two uplinks are trusted, confirmed by `show ip arp inspection interfaces`), whether ARP was actually being generated (`arp -d` followed by ping), and the packet path (traced in Simulation Mode) — before concluding this is a simulator limitation. The behaviour matches reports on the Cisco Community forum describing DAI in Packet Tracer as non-enforcing with zeroed statistics despite by-the-book configuration.

### 17. Port Security

**Command:** `show port-security`

```
Secure Port MaxSecureAddr CurrentAddr SecurityViolation Security Action
               (Count)       (Count)        (Count)
--------------------------------------------------------------------
        Fa0/1        1          1                 0         Restrict
        Fa0/2        1          1                 0         Restrict
```

**Command:** `show port-security address`

```
Vlan    Mac Address       Type                          Ports   Remaining Age
----    -----------       ----                          -----   -------------
  20    0090.2B29.BB9E    SecureSticky                  Fa0/1        -
  10    0007.EC68.41E6    SecureSticky                  Fa0/2        -
```

**Result:** ✅ Both access ports allow exactly one MAC, learned via sticky and written into the running config. Violation mode is `restrict` — traffic from any other MAC is dropped and logged without disabling the port. The learned MACs match the binding table entries in test 14.

> **Note — configuration gotcha.** During testing these ports initially reported `Port Security: Disabled` despite all parameters being present in `show run`. The cause was a missing bare `switchport port-security` command — the `maximum`, `sticky` and `violation` lines are only parameters and do nothing without the enabling command. This is the same two-level pattern as DHCP snooping.

### 18. Trunk configuration — SW1

**Command:** `show interfaces trunk`

```
Port        Mode         Encapsulation  Status        Native vlan
Gig0/1      on           802.1q         trunking      99
Gig0/2      on           802.1q         trunking      99

Port        Vlans allowed on trunk
Gig0/1      10,20,101,123
Gig0/2      10,20,101,123

Port        Vlans in spanning tree forwarding state and not pruned
Gig0/1      10,20,101,123
Gig0/2      none
```

**Result:** ✅ Both uplinks are hardened identically to SW-DISTRO (DTP off, native VLAN 99, restricted allowed list). The final block independently confirms the STP result from test 6: Gi0/2 forwards no VLANs because it is the blocked leg of the triangle.

---

## Part C — Policy Tests

This is where the ACL design is actually proven. Each test states what the policy requires and whether the network behaved accordingly.

### 19. HR → Finance — must be DENIED

*Policy: zero-trust between user zones; all exchange goes through Server-1.*

**From PC1-HR:** `ping 192.168.20.2`

```
Pinging 192.168.20.2 with 32 bytes of data:

Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.

Ping statistics for 192.168.20.2:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```

**Result:** ✅ Denied. The "unreachable" comes from `192.168.10.1` — HR's own gateway — meaning the packet reached the SVI and was rejected there by ACL_HR_IN, not lost in transit.

### 20. HR → Management — must be DENIED

*Policy: users have no access to the administrative plane.*

**From PC1-HR:** `ping 10.0.100.10`

```
Pinging 10.0.100.10 with 32 bytes of data:

Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.
Reply from 192.168.10.1: Destination host unreachable.

Ping statistics for 10.0.100.10:
    Packets: Sent = 4, Received = 0, Lost = 4 (100% loss),
```

**Result:** ✅ Denied at the HR SVI. Management remains unreachable from user zones.

### 21. Finance → internet — must be DENIED

*Policy: Finance has no internet access at all.*

**From PC2-Finance:** browser to `https://203.0.113.2` and `https://www.example.com`

Both return **Request Timeout** (screenshots in `evidence/`).

**Result:** ✅ Denied. Confirmed at the rule level in Simulation Mode, which reported the packet matching `deny ip any any` in ACL_FINANCE_IN and being dropped. The 114 hits on that rule in test 7 include these attempts.

### 22. HR → internet over HTTPS — must be PERMITTED

*Policy: HR is the only zone allowed outbound, and only on HTTPS.*

**From PC1-HR:** browser to `https://203.0.113.2` and `https://www.example.com`

Both load the web server page successfully (screenshots in `evidence/`).

**Result:** ✅ Permitted. This exercises the full chain end to end: DNS resolution via Server-1, routing SW-DISTRO → R1 → R2, the outbound permit in ACL_HR_IN (158 hits), and the `established` return permit in ACL_INTERNET_IN on R1 (6 hits). Simulation Mode confirmed the TCP handshake completing with the connection state set to ESTABLISHED.

### 23. HR → internet over HTTP — must be DENIED

*Policy: only encrypted web traffic leaves the network.*

**From PC1-HR:** browser to `http://www.example.com`

Returns **Request Timeout** (screenshot in `evidence/`).

**Result:** ✅ Denied. The same host that successfully reaches the site over HTTPS is blocked on port 80 — the policy permits TCP 443 only, and plain HTTP falls through to the closing deny. This pair of tests (22 and 23, same host, same site, different protocol) is the cleanest demonstration that the filtering is port-specific and not merely a reachability question.

### 24. DNS resolution — must be PERMITTED

*Policy: internal zones resolve names through Server-1, which relays outward.*

**From PC1-HR:** `nslookup www.example.com`

```
Server: [10.0.120.2]
Address:  10.0.120.2

Non-authoritative answer:
Name:   www.example.com
Address:   203.0.113.2
```

**Result:** ✅ Permitted. The client queried Server-1 and received the public address of the web server, confirming both the client→server DNS permit and the server's return path.

### 25. Management → all zones — must be PERMITTED

*Policy: Management is the administrative plane with reach into every zone.*

**From Admin-PC:**

```
C:\>ping -n 2 192.168.10.2
Reply from 192.168.10.2: bytes=32 time<1ms TTL=127
Reply from 192.168.10.2: bytes=32 time<1ms TTL=127
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)

C:\>ping -n 2 192.168.20.2
Reply from 192.168.20.2: bytes=32 time<1ms TTL=127
Reply from 192.168.20.2: bytes=32 time<1ms TTL=127
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)

C:\>ping -n 2 10.0.120.2
Reply from 10.0.120.2: bytes=32 time<1ms TTL=127
Reply from 10.0.120.2: bytes=32 time<1ms TTL=127
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)
```

**Result:** ✅ All three zones reachable from Management. Note the asymmetry against tests 19 and 20: Management reaches the user zones, but the user zones cannot reach Management or each other. This directional difference is exactly what the policy specifies, and it is enforced by pairing an `echo` permit in ACL_MGMT_IN with matching `echo-reply` permits in the user-zone ACLs.

### 26. Users → own default gateway — must be PERMITTED

*Policy: users may verify basic connectivity to their own gateway.*

**From PC1-HR:** `ping -n 2 192.168.10.1`

```
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255
Reply from 192.168.10.1: bytes=32 time<1ms TTL=255
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)
```

**From PC2-Finance:** `ping -n 2 192.168.20.1`

```
Reply from 192.168.20.1: bytes=32 time<1ms TTL=255
Reply from 192.168.20.1: bytes=32 time<1ms TTL=255
    Packets: Sent = 2, Received = 2, Lost = 0 (0% loss)
```

**Result:** ✅ Both permitted. Also confirms both hosts hold the correct DHCP-assigned addressing (`192.168.10.2` and `192.168.20.2`, /27, correct gateways) as shown by `ipconfig` in the same session.

---

## Summary

| # | Test | Device / Host | Expected | Result |
| -- | ---- | ------------- | -------- | ------ |
| 1 | VLAN configuration | SW-DISTRO | 6 VLANs, unused in 999 | ✅ Pass |
| 2 | Interfaces and SVIs | SW-DISTRO | Routed port + 4 SVIs up | ✅ Pass |
| 3 | Trunk hardening | SW-DISTRO | DTP off, native 99, allowed list | ✅ Pass |
| 4 | Routing table | SW-DISTRO | 4 connected zones + default | ✅ Pass |
| 5 | STP root bridge | SW-DISTRO | Root, both ports forwarding | ✅ Pass |
| 6 | STP triangle | SW1 | One leg blocking | ✅ Pass |
| 7 | ACL definitions + counters | SW-DISTRO | 5 ACLs matching traffic | ✅ Pass |
| 8 | ACL application on SVI | SW-DISTRO | Reported "not set" | ⚠️ PT limitation |
| 9 | Edge ACL + counters | R1 | Enforcing, established works | ✅ Pass |
| 10 | ACL application on interface | R1 | Displayed correctly | ✅ Pass |
| 11 | Routing table | R1 | 4 statics + default | ✅ Pass |
| 12 | Interfaces | R1, R2 | WAN and LAN up | ✅ Pass |
| 13 | DHCP Snooping | SW1 | Active, trust on uplinks only | ✅ Pass |
| 14 | Binding table | SW1 | 2 client bindings | ✅ Pass |
| 15 | Binding persistence | SW1 | Restored after reload | ✅ Pass |
| 16 | Dynamic ARP Inspection | SW1 | Configured, not enforcing | ⚠️ PT limitation |
| 17 | Port Security | SW1 | Sticky MAC, restrict | ✅ Pass |
| 18 | Trunk hardening | SW1 | DTP off, blocked leg confirmed | ✅ Pass |
| 19 | HR → Finance | PC1-HR | DENY | ✅ Pass |
| 20 | HR → Management | PC1-HR | DENY | ✅ Pass |
| 21 | Finance → internet | PC2-Finance | DENY | ✅ Pass |
| 22 | HR → internet (HTTPS) | PC1-HR | PERMIT | ✅ Pass |
| 23 | HR → internet (HTTP) | PC1-HR | DENY | ✅ Pass |
| 24 | DNS resolution | PC1-HR | PERMIT | ✅ Pass |
| 25 | Management → all zones | Admin-PC | PERMIT | ✅ Pass |
| 26 | Users → own gateway | PC1-HR, PC2-Finance | PERMIT | ✅ Pass |

Twenty-four of twenty-six tests pass as designed. The two marked ⚠️ are documented Packet Tracer limitations, not configuration faults — in both cases the configuration is correct and would behave as intended on physical hardware. Details and the diagnostic process behind those conclusions are in [lessons-learned.md](lessons-learned.md).
