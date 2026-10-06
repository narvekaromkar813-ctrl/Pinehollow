# Pinehollow Data Centers — Network Outage Investigation
## Case Study No. 30: Capstone Diagnostics
### Computer Networking & Cyber Security — School of Future Tech, ITM Skills University

![Status](https://img.shields.io/badge/Status-Remediated_&_Verified-success)
![Lab](https://img.shields.io/badge/Packet_Tracer-8.2+-blue)
![BGP](https://img.shields.io/badge/BGP-AS65000-orange)
![OSPF](https://img.shields.io/badge/OSPF-Area_0-green)
![EIGRP](https://img.shields.io/badge/EIGRP-AS100-yellow)

---

### Executive Overview

Pinehollow Data Centers operates three primary interconnected facilities providing high-availability colocation, enterprise cloud, and network operations services:
* **Facility A (HQ / Network Operations Center)**: Central administrative hub, telemetry aggregation, and central core OSPF Area 0 backbone (`R-HQ-A`).
* **Facility B (Colocation Hall)**: Legacy client rack infrastructure operating an internal EIGRP AS 100 routing domain (`R-FAC-B`) and a dual-switch access layer bundle (`SW-B1`, `SW-B2`).
* **Facility C (Primary Customer-Facing Data Hall)**: High-density compute/application environment (`R-FAC-C`) multihomed to dual Tier-1 Internet Service Providers (`ISP-1`, `ISP-2`) running BGP AS 65000.

Over a 48-hour window, widespread intermittent outages and security anomalies impacted all three facilities. This repository contains the complete diagnostic report, root cause analysis, correlation narrative, prioritized remediation strategy, annotated command outputs, and the fully functional Cisco Packet Tracer simulation topology file: [`Pinehollow_Data_Centers_Investigation.pkt`](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/Pinehollow_Data_Centers_Investigation.pkt).

---

### Network Architecture & Addressing Plan

```
                         +-------------------+           +-------------------+
                         |   ISP-1 (AS65100) |           |   ISP-2 (AS65200) |
                         |   203.0.113.1/30  |           |   198.51.100.1/30 |
                         +---------+---------+           +---------+---------+
                                   | Serial0/0/0                   | Serial0/0/1
                                   +---------------+---------------+
                                                   |
                                         +---------+---------+
                                         |      R-FAC-C      |
                                         |    (AS 65000)     |
                                         |  Data Hall & Edge |
                                         +----+-----------+--+
                        Gi0/1 (EIGRP AS100)   |           | Gi0/0 (OSPF Area 0)
                          10.20.23.0/30       |           |   10.10.13.0/30
                                              |           |
            +---------------------------------+           +------------------+
            |                                                                |
+-----------+-----------+                                          +---------+---------+
|        R-FAC-B        |        Gi0/0 (OSPF Area 0) 10.10.12.0/30 |      R-HQ-A       |
|    Colocation Hall    +------------------------------------------+    HQ / NOC Core  |
+-----------+-----------+                                          +---------+---------+
            | Gi0/2 (10.20.20.1/24)                                          | Gi0/2 (10.10.10.1/24)
  +---------+---------+                                            +---------+---------+
  |       SW-B1       |                                            |      SW-HQ-A      |
  +----+---------+----+                                            +----+---------+----+
       |         | Port-channel 1 (Fa0/23-24)                           |
       |  +------+------+                                               | Fa0/1
       |  |    SW-B2    |                                               v
       |  +------+------+                                        [PC-NOC-ADMIN]
       |         | Fa0/10                                         10.10.10.10/24
       v         v
 [PC-B-CLIENT] [SRV-B-DNS]
 10.20.20.10   10.20.20.53
```

#### Detailed IP Addressing Table

| Device | Interface | IP Address | Subnet Mask | Description / Connected To | Routing Domain |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **R-HQ-A** | `Loopback0` | `10.0.0.1` | `255.255.255.255` | Router ID / Management | OSPF Area 0 |
| | `GigabitEthernet0/0` | `10.10.12.1` | `255.255.255.252` | Transit to R-FAC-B (`Gi0/0`) | OSPF Area 0 (Point-to-Point) |
| | `GigabitEthernet0/1` | `10.10.13.1` | `255.255.255.252` | Transit to R-FAC-C (`Gi0/0`) | OSPF Area 0 (Point-to-Point) |
| | `GigabitEthernet0/2` | `10.10.10.1` | `255.255.255.0` | Facility A NOC LAN Gateway | OSPF Area 0 |
| **R-FAC-B** | `Loopback0` | `10.0.0.2` | `255.255.255.255` | Router ID / Management | OSPF / EIGRP |
| | `GigabitEthernet0/0` | `10.10.12.2` | `255.255.255.252` | Transit to R-HQ-A (`Gi0/0`) | OSPF Area 0 (Point-to-Point) |
| | `GigabitEthernet0/1` | `10.20.23.2` | `255.255.255.252` | Inter-Facility Link to R-FAC-C | EIGRP AS 100 |
| | `GigabitEthernet0/2` | `10.20.20.1` | `255.255.255.0` | Colocation LAN Gateway | EIGRP AS 100 |
| **R-FAC-C** | `Loopback0` | `10.0.0.3` | `255.255.255.255` | Router ID / BGP ID | OSPF / EIGRP / BGP |
| | `GigabitEthernet0/0` | `10.10.13.2` | `255.255.255.252` | Transit to R-HQ-A (`Gi0/1`) | OSPF Area 0 (Point-to-Point) |
| | `GigabitEthernet0/1` | `10.20.23.1` | `255.255.255.252` | Inter-Facility Link to R-FAC-B | EIGRP AS 100 |
| | `GigabitEthernet0/2` | `10.30.30.1` | `255.255.255.0` | Customer Data Hall Gateway | OSPF Area 0 / BGP |
| | `Serial0/0/0` | `203.0.113.2` | `255.255.255.252` | Multihomed Uplink to ISP-1 | eBGP (AS 65000 <-> 65100) |
| | `Serial0/0/1` | `198.51.100.2` | `255.255.255.252` | Multihomed Uplink to ISP-2 | eBGP (AS 65000 <-> 65200) |
| **ISP-1** | `Serial0/0/0` | `203.0.113.1` | `255.255.255.252` | Peering Link to R-FAC-C | eBGP (AS 65100) |
| | `Loopback0` | `1.1.1.100` | `255.255.255.255` | ISP-1 Global Backbone Prefix | BGP AS 65100 |
| **ISP-2** | `Serial0/0/0` | `198.51.100.1` | `255.255.255.252` | Peering Link to R-FAC-C | eBGP (AS 65200) |
| | `Loopback0` | `2.2.2.100` | `255.255.255.255` | ISP-2 Global Backbone Prefix | BGP AS 65200 |
| **SW-B1** | `Port-channel1` | Trunk | N/A | LACP Bundle (`Fa0/23 - 24`) | Layer 2 EtherChannel |
| **SW-B2** | `Port-channel1` | Trunk | N/A | LACP Bundle (`Fa0/23 - 24`) | Layer 2 EtherChannel |
| **PC-NOC-ADMIN** | `FastEthernet0` | `10.10.10.10` | `255.255.255.0` | Gateway: `10.10.10.1` | Facility A LAN |
| **PC-B-CLIENT** | `FastEthernet0` | `10.20.20.10` | `255.255.255.0` | Gateway: `10.20.20.1` | Facility B Colocation LAN |
| **SRV-B-DNS** | `FastEthernet0` | `10.20.20.53` | `255.255.255.0` | Gateway: `10.20.20.1` | Internal DNS Resolver |
| **SRV-C-APP** | `FastEthernet0` | `10.30.30.100` | `255.255.255.0` | Gateway: `10.30.30.1` | Production App Server |

---

### Project Deliverables Index

The investigation is documented across four core reports adhering strictly to the capstone requirements:

| Deliverable | File Link | Focus Area & Description |
| :--- | :--- | :--- |
| **Deliverable 1 & Objectives 1–5** | [Root Cause Analysis](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/docs/root_cause_analysis.md) | Comprehensive Root Cause Analysis Table (`Symptom` -> `Likely Cause` -> `Evidence` -> `Fix` -> `Verification Step`) plus deep-dive technical explanations for Symptoms A, B, C, D, and E. |
| **Deliverable 2 & Objective 6** | [Correlation Narrative](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/docs/correlation_narrative.md) | 300–500 word narrative rigorously analyzing whether the symptoms share a single common trigger versus multiple independent operational and security faults. |
| **Deliverable 3 & Objective 7** | [Remediation Plan](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/docs/remediation_plan.md) | Business impact prioritization ranking (Criticality, Blast Radius, Security Risk), step-by-step change-management execution sequence, and validation checkpoints. |
| **Deliverable 4** | [Annotated Command Outputs](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/docs/annotated_command_outputs.md) | Annotated before-and-after command outputs, Cisco IOS show command outputs, routing table diffs, and Wireshark packet capture analyses. |
| **Production Configs** | [Configurations Directory](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/configs/) | Clean, modular, production-ready Cisco IOS configuration files for all routers and switches in the topology. |

---

### Summary of Symptoms & Diagnostic Resolutions

1. **Symptom A — EtherChannel Load-Balancing Polarization**:
   * *Problem*: Member link `FastEthernet0/23` carried >95% of traffic while `FastEthernet0/24` was idle due to default `src-ip` hashing on point-to-point client-server flows.
   * *Resolution*: Executed `port-channel load-balance src-dst-ip` on both access switches to achieve symmetric bidirectional link utilization.
2. **Symptom B — OSPF Network-Type Mismatch**:
   * *Problem*: OSPF adjacency between Facility A and B failed to form because `R-HQ-A` was configured as `point-to-point` while `R-FAC-B` was configured as `broadcast` (DR/BDR and timer mismatch).
   * *Resolution*: Configured `ip ospf network point-to-point` on `R-FAC-B GigabitEthernet0/0`, immediately bringing the adjacency to `FULL/-`.
3. **Symptom C — EIGRP Passive Interface Silent Blackholing**:
   * *Problem*: `R-FAC-B` had `passive-interface GigabitEthernet0/1` applied, suppressing EIGRP Hellos and prefix advertisements toward Facility C.
   * *Resolution*: Executed `no passive-interface GigabitEthernet0/1` under `router eigrp 100`, restoring route propagation for `10.20.20.0/24`.
4. **Symptom D — BGP Route-Map Direction Inversion**:
   * *Problem*: Outbound policy route-map was applied `in` instead of `out` on BGP neighbor `198.51.100.1`, suppressing customer route advertisements to ISP-2.
   * *Resolution*: Realigned route-map directionality using `neighbor 198.51.100.1 route-map OUTBOUND-CUSTOMER-ONLY out`.
5. **Symptom E — DNS Cache Poisoning / Response Spoofing**:
   * *Problem*: Kaminsky-style brute-force DNS response flooding against recursive resolver `10.20.20.53` resulting in fraudulent domain resolution.
   * *Resolution*: Flushed resolver cache, enforced 16-bit UDP source port randomization, enabled DNSSEC cryptographic validation, and applied boundary ACLs.

---

### Packet Tracer Simulation Guide

The active topology has been built and saved as [`Pinehollow_Data_Centers_Investigation.pkt`](file:///Users/danishshaikh1423/Downloads/OMKAR-KI-CN-WALI-ASS/Pinehollow_Data_Centers_Investigation.pkt).

#### How to Open and Inspect:
1. Launch **Cisco Packet Tracer** (version 8.2 or later).
2. Open `File -> Open` and select `Pinehollow_Data_Centers_Investigation.pkt`.
3. Verify device layout:
   * **Top Layer**: `ISP-1` and `ISP-2` simulating external upstream Tier-1 BGP peers.
   * **Core Layer**: `R-HQ-A` (HQ NOC), `R-FAC-B` (Facility B), and `R-FAC-C` (Facility C).
   * **Distribution / Access Layer**: `SW-HQ-A`, `SW-B1`, `SW-B2` (with Port-channel 1), and `SW-FAC-C`.
   * **Workstations / Servers**: `PC-NOC-ADMIN`, `PC-B-CLIENT`, `SRV-B-DNS`, `SRV-C-APP`.
4. Run connectivity checks:
   * Ping from `PC-B-CLIENT` (`10.20.20.10`) to Gateway `10.20.20.1`.
   * Ping across inter-facility links (`10.10.12.1`, `10.10.13.2`, `10.20.23.1`).
   * Inspect OSPF neighbors: `R-HQ-A# show ip ospf neighbor`.
   * Inspect EIGRP neighbors: `R-FAC-C# show ip eigrp neighbors`.
   * Inspect BGP peering: `R-FAC-C# show ip bgp summary`.
