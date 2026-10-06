# Prioritized Remediation Plan & Business Impact Assessment
## Case Study No. 30: Pinehollow Data Centers — Capstone Diagnostics

---

### Remediation Priority Ranking Matrix

Remediation must follow a risk-weighted engineering hierarchy: **Security Threats & Active Exploits** must be contained first to prevent data compromise, followed by **Core Routing & Reachability** to restore customer SLAs, and finally **Performance & Optimization** tuning.

| Priority Rank | Symptom | Location | Business Impact & Risk Justification | Blast Radius & SLA Risk | Proposed Remediation Order | Verification Check |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **P1 (CRITICAL)** | **Symptom E** (DNS Cache Poisoning) | Facility B Access Layer | **Active Security Breach / Confidentiality Compromise**: Malicious actors are actively redirecting internal hosts to fraudulent external IPs via DNS cache poisoning. Immediate risk of credential theft, session hijacking, data exfiltration, and compliance violations (SOC2, ISO 27001, HIPAA). | High: All internal users and automated workloads relying on Facility B resolver (`10.20.20.53`). | **1st**: Purge poisoned resolver cache, enforce source-port randomization, enable DNSSEC, apply firewall DNS inspection/filtering. | Run `dig +dnssec [domain] @10.20.20.53` and inspect resolver cache via `rndc dumpdb -cache` to confirm authentic A record mapping. |
| **P2 (HIGH)** | **Symptom B** (OSPF Adjacency Failure) | Facility A–B WAN Link | **Core Backbone Partition**: The primary OSPF transit circuit linking Headquarters (NOC) and Facility B is completely down due to network-type mismatch. Facility B is isolated from the central monitoring plane and relies entirely on asymmetric suboptimal backup paths. | High: NOC loses real-time telemetry and management access; inter-facility failover paths degraded. | **2nd**: Align OSPF network type to `point-to-point` on `R-FAC-B GigabitEthernet0/0`. | Run `show ip ospf neighbor` to verify `FULL/-` state on `Gi0/0` and verify routing table convergence with `show ip route ospf`. |
| **P3 (HIGH)** | **Symptom C** (EIGRP Passive Interface) | Facility B–C Interconnect | **Customer Service Blackhole**: Customer-facing systems at Facility C cannot communicate with colocation services at Facility B. Generates direct breach of availability SLAs and customer ticket escalation. | Medium-High: Direct traffic blackholing between primary customer hall and colocation database clusters. | **3rd**: Remove `passive-interface GigabitEthernet0/1` under `router eigrp 100` on `R-FAC-B`. | Execute `show ip protocols` and `show ip route eigrp` on `R-FAC-C` to confirm receipt of `10.20.20.0/24`. Test bidirectional ping. |
| **P4 (MEDIUM)** | **Symptom D** (BGP Route-Map Direction) | Facility C BGP Edge | **Transit Policy / Routing Degradation**: External BGP routes are not properly advertised or received due to inverted route-map directionality. If misapplied outbound, upstream ISPs do not receive customer prefix advertisements; if misapplied inbound, customer transit route filtering fails. | Medium: Internet-facing customer applications experience suboptimal routing or loss of multihomed redundancy. | **4th**: Correct route-map attachment direction on `R-FAC-C` under `router bgp 65000` with proper prefix-lists. | Execute `clear ip bgp [peer] soft out` and verify prefix propagation via `show ip bgp neighbors [peer] advertised-routes`. |
| **P5 (MEDIUM)** | **Symptom A** (EtherChannel Imbalance) | Facility B Switching Backplane | **Performance Degradation & Link Congestion**: Severe traffic polarization over the 2-link port-channel causes buffer drops and latency spikes for sessions hashing to Link 1, while Link 2 is underutilized. No total outage, but severe jitter and packet loss under peak load. | Low-Medium: Localized to inter-switch traffic in Facility B colocation hall. | **5th**: Change global load-balancing hash algorithm from `src-ip` to `src-dst-ip` on `SW-B1` and `SW-B2`. | Execute `show etherchannel load-balance` and observe interface packet counters on `Fa0/23` and `Fa0/24` showing balanced traffic distribution. |

---

### Step-by-Step Remediation Procedure

#### Phase 1: Security Containment (P1 — Immediate Hotfix)
1. **Flush Poisoned Resolver Cache**:
   * On DNS Server `SRV-B-DNS` (`10.20.20.53`):
     ```bash
     rndc flush
     rndc flushname [affected-domain.com]
     ```
2. **Implement Source Port Randomization & ACL Filtering**:
   * Enforce resolver configuration to use 16-bit ephemeral UDP port randomization.
   * On boundary router `R-FAC-B`, apply an ingress/egress ACL restricting DNS traffic to authorized upstream recursive forwarders:
     ```cisco
     R-FAC-B(config)# ip access-list extended SECURE-DNS-ACL
     R-FAC-B(config-ext-nacl)# permit udp host 10.20.20.53 any eq 53
     R-FAC-B(config-ext-nacl)# permit udp any eq 53 host 10.20.20.53 established
     R-FAC-B(config-ext-nacl)# deny udp any any eq 53
     R-FAC-B(config-ext-nacl)# permit ip any any
     ```

#### Phase 2: Core Backbone Restoration (P2 — OSPF Adjacency)
1. Enter interface configuration on `R-FAC-B`:
   ```cisco
   R-FAC-B(config)# interface GigabitEthernet0/0
   R-FAC-B(config-if)# ip ospf network point-to-point
   R-FAC-B(config-if)# end
   ```
2. Verification:
   ```cisco
   R-FAC-B# show ip ospf neighbor
   Neighbor ID     Pri   State           Dead Time   Address         Interface
   1.1.1.1           0   FULL/  -        00:00:36    10.10.12.1      GigabitEthernet0/0
   ```

#### Phase 3: Inter-Facility Reachability Restoration (P3 — EIGRP Fix)
1. Remove passive-interface configuration on `R-FAC-B`:
   ```cisco
   R-FAC-B(config)# router eigrp 100
   R-FAC-B(config-router)# no passive-interface GigabitEthernet0/1
   R-FAC-B(config-router)# end
   ```
2. Verification:
   ```cisco
   R-FAC-C# show ip route eigrp
   D    10.20.20.0/24 [90/30720] via 10.20.23.2, 00:00:14, GigabitEthernet0/1
   R-FAC-C# ping 10.20.20.10 source GigabitEthernet0/2
   Sending 5, 100-byte ICMP Echos to 10.20.20.10, timeout is 2 seconds:
   !!!!!
   Success rate is 100 percent (5/5)
   ```

#### Phase 4: Edge Routing Policy Realignment (P4 — BGP Route-Map Fix)
1. Correct route-map attachment on `R-FAC-C`:
   ```cisco
   R-FAC-C(config)# router bgp 65000
   R-FAC-C(config-router)# no neighbor 198.51.100.1 route-map TRANSIT-POLICY in
   R-FAC-C(config-router)# neighbor 198.51.100.1 route-map OUTBOUND-CUSTOMER-ONLY out
   R-FAC-C(config-router)# end
   R-FAC-C# clear ip bgp 198.51.100.1 soft out
   ```
2. Verification:
   ```cisco
   R-FAC-C# show ip bgp neighbors 198.51.100.1 advertised-routes
   Network          Next Hop            Metric LocPrf Weight Path
   *> 10.30.30.0/24 0.0.0.0                  0         32768 i
   ```

#### Phase 5: Access Layer Performance Optimization (P5 — EtherChannel Hash)
1. Apply symmetric source-destination hashing on both switches:
   ```cisco
   SW-B1(config)# port-channel load-balance src-dst-ip
   SW-B2(config)# port-channel load-balance src-dst-ip
   ```
2. Verification:
   ```cisco
   SW-B1# show etherchannel load-balance
   EtherChannel Load-Balancing Configuration:
           Global LB Method: src-dst-ip
   ```
