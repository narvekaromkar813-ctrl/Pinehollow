# Annotated Before/After Command Output Verification
## Case Study No. 30: Pinehollow Data Centers — Capstone Diagnostics

---

### Overview

This document presents annotated CLI outputs and packet capture analyses contrasting the defective state ("Before Fix") with the remediated state ("After Fix") across all five symptoms.

---

### Symptom A: EtherChannel Load-Balancing Imbalance (Facility B Access Layer)

#### BEFORE FIX: Defective State (Source-IP Hashing)
```cisco
SW-B1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        Global LB Method: src-ip

! --- Member interface traffic counters show 99% polarization onto Fa0/23 ---
SW-B1# show interfaces FastEthernet0/23 | include 5 minute|packets input
  5 minute input rate 98400000 bits/sec, 12250 packets/sec
  5 minute output rate 98150000 bits/sec, 12200 packets/sec
     14502394 packets input, 1856306432 bytes, 0 no buffer
     18432 packets dropped, 1202 buffer failures

SW-B1# show interfaces FastEthernet0/24 | include 5 minute|packets input
  5 minute input rate 120000 bits/sec, 85 packets/sec
  5 minute output rate 115000 bits/sec, 80 packets/sec
     45102 packets input, 5773056 bytes, 0 no buffer
     0 packets dropped, 0 buffer failures

! --- Hashing test confirms all frames between Client and DNS/Server map to Fa0/23 ---
SW-B1# test etherchannel load-balance interface Port-channel 1 ip 10.20.20.10 10.20.20.53
Computed hash: 0x0
Selected physical interface: FastEthernet0/23
```

#### CONFIGURATION APPLIED
```cisco
SW-B1(config)# port-channel load-balance src-dst-ip
SW-B2(config)# port-channel load-balance src-dst-ip
```

#### AFTER FIX: Remediated State (Source-Destination IP Hashing)
```cisco
SW-B1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        Global LB Method: src-dst-ip

! --- Traffic is now dynamically and symmetrically distributed across bundle members ---
SW-B1# show interfaces FastEthernet0/23 | include 5 minute
  5 minute input rate 49200000 bits/sec, 6125 packets/sec
  5 minute output rate 49100000 bits/sec, 6100 packets/sec

SW-B1# show interfaces FastEthernet0/24 | include 5 minute
  5 minute input rate 49150000 bits/sec, 6120 packets/sec
  5 minute output rate 49050000 bits/sec, 6100 packets/sec

! --- Bidirectional flow hashing maps traffic in opposite directions to complementary links ---
SW-B1# test etherchannel load-balance interface Port-channel 1 ip 10.20.20.10 10.20.20.53
Computed hash: 0x1
Selected physical interface: FastEthernet0/24

SW-B1# test etherchannel load-balance interface Port-channel 1 ip 10.20.20.53 10.20.20.10
Computed hash: 0x0
Selected physical interface: FastEthernet0/23
```

---

### Symptom B: OSPF Network-Type Mismatch (Facility A–B Link)

#### BEFORE FIX: Defective State (Adjacency Failure)
```cisco
! --- R-HQ-A is configured as point-to-point; R-FAC-B is default broadcast ---
R-HQ-A# show ip ospf interface GigabitEthernet0/0
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 10.10.12.1/30, Area 0
  Process ID 1, Router ID 1.1.1.1, Network Type POINT_TO_POINT, Cost: 1
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:04

R-FAC-B# show ip ospf interface GigabitEthernet0/0
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 10.10.12.2/30, Area 0
  Process ID 1, Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
  Designated Router (ID) 2.2.2.2, Interface address 10.10.12.2
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

! --- Neighbor table remains empty or flaps continuously ---
R-FAC-B# show ip ospf neighbor
% OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1 on GigabitEthernet0/0 from INIT to DOWN, Neighbor Down: Dead timer expired
(No active neighbors found)

R-FAC-B# debug ip ospf adj
OSPF adjacency debugging is on
*Oct  5 12:40:12.301: OSPF: Rcv pkt from 10.10.12.1, GigabitEthernet0/0, area 0.0.0.0 : mismatched network type (p2p vs broadcast)
```

#### CONFIGURATION APPLIED
```cisco
R-FAC-B(config)# interface GigabitEthernet0/0
R-FAC-B(config-if)# ip ospf network point-to-point
```

#### AFTER FIX: Remediated State (Adjacency FULL)
```cisco
R-FAC-B# show ip ospf interface GigabitEthernet0/0
GigabitEthernet0/0 is up, line protocol is up
  Internet Address 10.10.12.2/30, Area 0
  Process ID 1, Router ID 2.2.2.2, Network Type POINT_TO_POINT, Cost: 1
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5

R-FAC-B# show ip ospf neighbor
Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           0   FULL/  -        00:00:34    10.10.12.1      GigabitEthernet0/0

! --- OSPF routes from Facility A are now learned in the RIB ---
R-FAC-B# show ip route ospf
O        10.0.0.1/32 [110/2] via 10.10.12.1, 00:01:15, GigabitEthernet0/0
O        10.10.10.0/24 [110/2] via 10.10.12.1, 00:01:15, GigabitEthernet0/0
O        10.10.13.0/30 [110/2] via 10.10.12.1, 00:01:15, GigabitEthernet0/0
```

---

### Symptom C: EIGRP Passive-Interface Statement (Facility B–C Boundary)

#### BEFORE FIX: Defective State (Gi0/1 Passive)
```cisco
R-FAC-B# show ip protocols
Routing Protocol is "eigrp 100"
  Outgoing update filter list for all interfaces is not set
  Incoming update filter list for all interfaces is not set
  Default networks flagged in outgoing updates
  Default networks accepted from incoming updates
  EIGRP-IPv4 Protocol for AS(100)
    Metric weight K1=1, K2=0, K3=1, K4=0, K5=0
    NSF-aware capability is enabled
    Router-ID: 10.0.0.2
    Topology : 0 (base)
      Active Timer: 3 min
      Distance: internal 90 external 170
      Maximum Interfaces: 1024
    Passive Interface(s):
      GigabitEthernet0/1       <-- ERRONEOUS PASSIVE STATEMENT SILENCING B-C LINK!
    Routing for Networks:
      10.0.0.2/32
      10.20.20.0/24
      10.20.23.0/30

! --- R-FAC-C has no EIGRP route for Facility B internal subnet ---
R-FAC-C# show ip route eigrp
(Empty - No EIGRP routes installed for 10.20.20.0/24)

R-FAC-C# ping 10.20.20.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.20.20.10, timeout is 2 seconds:
U.U.U
Success rate is 0 percent (0/5)
```

#### CONFIGURATION APPLIED
```cisco
R-FAC-B(config)# router eigrp 100
R-FAC-B(config-router)# no passive-interface GigabitEthernet0/1
```

#### AFTER FIX: Remediated State (EIGRP Adjacency & Reachability Restored)
```cisco
R-FAC-B# show ip protocols | section Passive
    Passive Interface(s):
      (None)

%DUAL-5-NBRCHANGE: EIGRP-IPv4 100: Neighbor 10.20.23.1 (GigabitEthernet0/1) is up: new adjacency

R-FAC-C# show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(100)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q   Seq
                                                   (sec)         (ms)       Cnt  Num
0   10.20.23.2              Gi0/1                    12 00:02:18   12   200  0   14

R-FAC-C# show ip route eigrp
D        10.20.20.0/24 [90/30720] via 10.20.23.2, 00:02:22, GigabitEthernet0/1
D        10.0.0.2/32 [90/28416] via 10.20.23.2, 00:02:22, GigabitEthernet0/1

R-FAC-C# ping 10.20.20.10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.20.20.10, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/2/4 ms
```

---

### Symptom D: BGP Route-Map Inbound/Outbound Inversion (Facility C Edge)

#### BEFORE FIX: Defective State (Wrong Direction Applied)
```cisco
R-FAC-C# show running-config | section router bgp
router bgp 65000
 bgp log-neighbor-changes
 neighbor 198.51.100.1 remote-as 65200
 neighbor 198.51.100.1 route-map FILTER-TRANSIT in   <-- APPLIED INBOUND INSTEAD OF OUTBOUND!
 neighbor 203.0.113.1 remote-as 65100
 network 10.30.30.0 mask 255.255.255.0

! --- ISP-2 receives zero prefix advertisements ---
R-FAC-C# show ip bgp neighbors 198.51.100.1 advertised-routes
Total number of prefixes 0

! --- Inbound routes from ISP-2 are accidentally suppressed by the outbound policy ---
R-FAC-C# show ip bgp neighbors 198.51.100.1 routes
Total number of prefixes 0 (All incoming routes dropped by FILTER-TRANSIT)
```

#### CONFIGURATION APPLIED
```cisco
R-FAC-C(config)# ip prefix-list PINEHOLLOW-PREFIXES permit 10.30.30.0/24
R-FAC-C(config)# route-map OUTBOUND-CUSTOMER-ONLY permit 10
R-FAC-C(config-route-map)# match ip address prefix-list PINEHOLLOW-PREFIXES
R-FAC-C(config)# router bgp 65000
R-FAC-C(config-router)# no neighbor 198.51.100.1 route-map FILTER-TRANSIT in
R-FAC-C(config-router)# neighbor 198.51.100.1 route-map OUTBOUND-CUSTOMER-ONLY out
R-FAC-C(config-router)# end
R-FAC-C# clear ip bgp 198.51.100.1 soft
```

#### AFTER FIX: Remediated State (BGP Route Advertisement Restored)
```cisco
R-FAC-C# show ip bgp neighbors 198.51.100.1 advertised-routes
BGP table version is 4, local router ID is 10.0.0.3
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal
Origin codes: i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*> 10.30.30.0/24    0.0.0.0                  0         32768 i
Total number of prefixes 1

R-FAC-C# show ip bgp summary
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
198.51.100.1    4        65200      48      45        4    0    0 00:32:15        1
203.0.113.1     4        65100      50      47        4    0    0 00:34:02        1
```

---

### Symptom E: DNS Response Spoofing / Poisoning (Wireshark Capture)

#### BEFORE FIX: Defective State (Packet Capture of Active Spoofing Attack)
```text
=== Wireshark Packet Capture Analysis (Facility B Access Layer) ===
No.   Time      Source         Destination     Protocol Length Info
001   0.000000  10.20.20.10    10.20.20.53     DNS      74     Standard query 0x1a2b A portal.pinehollow.com
002   0.000185  198.51.100.99  10.20.20.53     DNS      90     Standard query response 0x4f12 A 203.0.113.66 [MISMATCHED TXID]
003   0.000210  198.51.100.99  10.20.20.53     DNS      90     Standard query response 0x89ab A 203.0.113.66 [MISMATCHED TXID]
004   0.000245  198.51.100.99  10.20.20.53     DNS      90     Standard query response 0x1a2b A 198.51.100.222 [MATCHED TXID - SPOOFED INJECTION!]
005   0.015210  8.8.8.8        10.20.20.53     DNS      90     Standard query response 0x1a2b A 104.21.45.10 [LEGITIMATE RESPONSE - ARRIVED TOO LATE]

! --- Verification on client host showing poisoned resolution ---
PC-B-CLIENT> nslookup portal.pinehollow.com
Server:  SRV-B-DNS.colo.pinehollow.local
Address: 10.20.20.53

Non-authoritative answer:
Name:    portal.pinehollow.com
Address: 198.51.100.222   <-- ATTACKER CONTROLLED MALICIOUS SERVER IP!
```

#### CONFIGURATION APPLIED
1. **Flushed Cache**: Flushed resolver cache on `10.20.20.53`.
2. **Enabled Source Port Randomization**: Upgraded resolver daemon configuration to randomize all 16 bits of ephemeral UDP source ports.
3. **Enabled DNSSEC Validation**: Enforced cryptographic validation of `RRSIG` records.
4. **Boundary Ingress Filtering**: Dropped unsolicited external DNS responses.

#### AFTER FIX: Remediated State (Attack Neutralized & Validated Resolution)
```text
=== Post-Remediation Wireshark Capture ===
No.   Time      Source         Destination     Protocol Length Info
001   0.000000  10.20.20.10    10.20.20.53     DNS      74     Standard query 0x7c91 A portal.pinehollow.com (Src Port: 54192)
002   0.000150  198.51.100.99  10.20.20.53     DNS      90     [DROPPED BY ACL / INVALID PORT 53->53000]
003   0.012450  8.8.8.8        10.20.20.53     DNS      124    Standard query response 0x7c91 A 104.21.45.10 RRSIG [DNSSEC VALIDATED]

! --- Client resolution verified accurate and secure ---
PC-B-CLIENT> nslookup portal.pinehollow.com
Server:  SRV-B-DNS.colo.pinehollow.local
Address: 10.20.20.53

Non-authoritative answer:
Name:    portal.pinehollow.com
Address: 104.21.45.10   <-- AUTHENTIC CANONICAL PRODUCTION IP RESTORED!
```
