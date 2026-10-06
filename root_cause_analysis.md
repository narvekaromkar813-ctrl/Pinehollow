# Root Cause Analysis & Technical Deep Dive
## Case Study No. 30: Pinehollow Data Centers — Capstone Diagnostics

---

### Executive Summary

Pinehollow Data Centers operates three primary interconnected facilities:
* **Facility A (HQ / Network Operations Center)**: Manages network monitoring, operations, and hosts the central OSPF Area 0 backbone router (`R-HQ-A`).
* **Facility B (Colocation Hall)**: Houses legacy customer colocation racks operating an internal EIGRP Autonomous System 100 (`R-FAC-B`) and a dual-switch access layer bundle (`SW-B1`, `SW-B2`).
* **Facility C (Primary Customer-Facing Data Hall & Transit Edge)**: Houses high-density customer applications (`R-FAC-C`), internal distribution, and multihomed BGP edge uplinks to dual Tier-1 Internet Service Providers (`ISP-1`, `ISP-2`).

Over a 48-hour window, widespread intermittent outages were reported. This document provides a granular diagnosis, root-cause identification, and remediation plan for all five observed symptoms.

---

### Deliverable 1: Root Cause Analysis Matrix

| Symptom | Location | Observed Behavior | Likely Root Cause | Technical Evidence | Proposed Fix | Verification Step |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Symptom A** | Facility B Access Layer | EtherChannel bundle between switches shows severe polarization; one member link near capacity while others idle. | EtherChannel hashing algorithm misconfigured to default source IP (`src-ip`) or source MAC in an environment dominated by high-volume point-to-point traffic (e.g., backup or server-to-server stream between a single IP pair). | `show etherchannel load-balance` confirms `Source IP (src-ip)`. Interface packet counters on Port-channel member links show link 1 carrying >95% of frames while link 2 carries <5%. | Reconfigure global load-balancing hash algorithm to include both Layer 3 and Layer 4 parameters or combined Source/Destination IP: `port-channel load-balance src-dst-ip` (or `src-dst-port` if supported). | Execute `show etherchannel load-balance` to verify active hashing method; run `show interfaces port-channel 1` and `show interfaces [member-ports]` over 60 seconds to confirm balanced load distribution across bundle members. |
| **Symptom B** | Facility A–B OSPF Link | OSPF adjacency between Facility A (`R-HQ-A`) and Facility B (`R-FAC-B`) fails to form; log messages cite network-type mismatch. | OSPF interface network type discrepancy: `R-HQ-A Gi0/0` is configured as `point-to-point` while `R-FAC-B Gi0/0` is configured with default `broadcast`. | Syslog outputs show OSPF Hello interval mismatch (`Hello 10, Dead 40` on Broadcast vs `Hello 30, Dead 120` or Hello timer agreement but dead-lock due to DR/BDR election expectations). `show ip ospf interface` confirms type mismatch. | Align OSPF network types on both sides of the point-to-point transit link: configure `ip ospf network point-to-point` on `R-FAC-B Gi0/0` (or `broadcast` on both ends). | Run `show ip ospf neighbor` to verify state transitions to `FULL/-`. Confirm OSPF routes are populated in RIB via `show ip route ospf`. |
| **Symptom C** | Facility B–C EIGRP Domain | Devices at Facility C cannot reach Facility B subnets despite boundary EIGRP neighbor relationship being up and stable. | The `passive-interface` command was inadvertently applied to the boundary interface facing Facility C (`GigabitEthernet0/1`) on `R-FAC-B`, suppressing EIGRP Hello and Update multicast transmissions on that link while another interface maintained adjacency, OR silencing outbound route advertisements. | `show ip protocols` displays `GigabitEthernet0/1` under "Passive Interface(s)". `show ip eigrp neighbors` reveals adjacency formed over an alternate/transit link (e.g. via Facility A or secondary path) rather than the direct link, or `show ip eigrp topology` indicates routes are not advertised out Gi0/1. | Remove the passive-interface directive on the transit interface: execute `no passive-interface GigabitEthernet0/1` under `router eigrp 100` on `R-FAC-B`. | Execute `show ip protocols` to confirm Gi0/1 is no longer passive. On `R-FAC-C`, execute `show ip route eigrp` to verify Facility B subnets (`10.20.20.0/24`) appear with direct next-hop `10.20.23.2`. |
| **Symptom D** | Facility C Internet Edge (BGP) | Routes learned from ISP-1 are not advertised onward to ISP-2 as expected; an outbound route-map was applied in the wrong neighbor direction. | Directional route-map misapplication: An outbound policy (intended to prevent transit routing or filter advertised prefixes) was applied `in` instead of `out`, or an inbound customer-filter route-map was applied `out` toward the wrong eBGP peer, suppressing expected BGP UPDATE advertisements. | `show running-config | section router bgp` reveals `neighbor 198.51.100.1 route-map <MAP> in` instead of `out`, or vice versa. `show ip bgp neighbors [IP] advertised-routes` returns empty. | Realign route-map directionality: remove `neighbor [peer] route-map [name] in` and apply `neighbor [peer] route-map [name] out` with explicit prefix-lists matching local autonomous system prefixes. | Execute `clear ip bgp [peer] soft out` followed by `show ip bgp neighbors [peer] advertised-routes` to confirm intended route propagation to upstream peers. |
| **Symptom E** | Facility B Access Layer (Wireshark) | Packet capture reveals burst of DNS responses with mismatched Transaction IDs and spoofed source IPs for a single query, followed by DNS cache poisoning. | Kaminsky-style DNS Cache Poisoning / Response Spoofing attack targeting the internal caching resolver at Facility B (`10.20.20.53`). The attacker flooded fraudulent responses guessing the 16-bit Transaction ID and UDP port before authoritative upstream servers responded. | Wireshark capture shows multiple rapid DNS response packets (Type A) arriving per single DNS query, exhibiting sequential/randomized 16-bit IDs, varying source IPs/ports, followed by poisoned `A` record resolution mapping legitimate domain to malicious IP. | Implement DNS Source Port Randomization (SPR), enforce DNSSEC validation on recursive resolvers, configure DNS inspection on perimeter firewalls/routers, and implement Access Control Lists restricting external UDP/53 responses strictly to authorized authoritative nameservers. | Query DNS resolver via `nslookup` / `dig` with `+dnssec` validation enabled. Inspect resolver logs to confirm rejection of unauthenticated responses and verify correct canonical IP mapping. |

---

### Detailed Technical Analysis & Objective Responses

#### Objective 1: EtherChannel Load-Balancing Hash Algorithm Analysis (Symptom A)

**1. Architectural Mechanics:**
EtherChannel bundles multiple physical links into a single logical channel (Port-Channel). To prevent out-of-order packet delivery within an individual TCP session, frame distribution across member links does not use round-robin packet slicing. Instead, Cisco switches employ a deterministic hardware hashing algorithm executed by the ASIC. The hashing function accepts header fields (MAC addresses, IP addresses, or Layer 4 TCP/UDP ports) and produces a numerical hash value (typically 0–7 for an 8-link bundle) pointing to an egress physical link.

**2. Cause of Imbalance:**
When the default load-balancing hash algorithm is set to `src-ip` (Source IP) or `dst-ip` (Destination IP):
$$\text{Egress Link} = f(\text{Source IP})$$
If high-volume traffic in Facility B is generated primarily between a single source and destination (e.g., automated database replication or large NFS/SAN backup traffic between server `10.20.20.10` and `10.20.20.53`), the source IP remains identical for every packet. Consequently, the hash value remains completely static:
$$f(10.20.20.10) \equiv k \implies \text{All frames egress member link } k$$
Link $k$ reaches saturation (>95% bandwidth utilization), triggering tail drops and interface buffer overflows, while the remaining bundled links sit idle (0–2% utilization).

**3. Diagnostic Commands:**
```cisco
SW-B1# show etherchannel load-balance
EtherChannel Load-Balancing Configuration:
        Global LB Method: src-ip

SW-B1# show interfaces Port-channel 1
Port-channel1 is up, line protocol is up (connected)
  5 minute input rate 98450000 bits/sec, 12200 packets/sec
  5 minute output rate 98200000 bits/sec, 12150 packets/sec

SW-B1# show interfaces FastEthernet0/23 | include 5 minute
  5 minute input rate 98300000 bits/sec, 12100 packets/sec  <-- 99% Saturated
SW-B1# show interfaces FastEthernet0/24 | include 5 minute
  5 minute input rate 150000 bits/sec, 100 packets/sec      <-- Idle

SW-B1# test etherchannel load-balance interface port-channel 1 ip 10.20.20.10 10.20.20.53
Computed hash: 0x0 -> FastEthernet0/23
```

**4. Corrective Action:**
```cisco
SW-B1(config)# port-channel load-balance src-dst-ip
SW-B2(config)# port-channel load-balance src-dst-ip
```
*(On Layer 4-capable platforms: `port-channel load-balance src-dst-mixed-ip-port` or `src-dst-port` offers optimal distribution).*

---

#### Objective 2: OSPF Network-Type Mismatch Adjacency Failure (Symptom B)

**1. Root Cause Breakdown:**
Unlike protocols that tolerate minor parameter drift, OSPF (Open Shortest Path First) mandates strict agreement between adjacent neighbors on transit media. When `R-HQ-A` configures `point-to-point` and `R-FAC-B` leaves the default Ethernet interface network type as `broadcast`:
* **Hello & Dead Interval Mismatch**:
  * Default OSPF timers for **Broadcast**: Hello = **10 seconds**, Dead = **40 seconds**.
  * Default OSPF timers for **Point-to-Point**: Hello = **10 seconds** on Ethernet, but on serial/certain non-broadcast types or customized profiles, timers differ (typically 30s/120s or 10s/40s). Even if Hello/Dead timers match, adjacency fails due to network-type state-machine divergence.
* **DR/BDR Election Conflict**:
  * On a `broadcast` network, OSPF requires Designated Router (DR) and Backup Designated Router (BDR) election to minimize link-state advertisements (LSAs). The router in `broadcast` mode listens on multicast `224.0.0.5` (AllSPFRouters) and `224.0.0.6` (AllDRouters) and sends Hellos listing neighbor Router IDs and DR/BDR priorities.
  * On a `point-to-point` network, OSPF suppresses DR/BDR election entirely. Hellos do not contain DR/BDR fields.
  * `R-FAC-B` waits to receive Hellos listing itself as a 2-way neighbor and indicating DR status. Because `R-HQ-A` does not participate in DR election, `R-FAC-B` remains stuck in `INIT` or `2-WAY` state, or repeatedly fails database exchange (`EXSTART`/`EXCHANGE`) because MTU negotiation and master/slave role negotiation fail to resolve.

**2. Why It Prevents Adjacency Completely:**
RFC 2328 mandates that if the network types are fundamentally incompatible in their Link State Database (LSDB) representation, the link state cannot be described in Router-LSA (Type 1). A point-to-point link is described as a connection to another router (Type 1 link), whereas a broadcast link is described as a connection to a transit network (Type 2 LSA generated by the DR). Without a DR, no Type 2 LSA can be generated, making shortest-path first (SPF) calculation mathematically impossible.

**3. Remediation Command:**
```cisco
R-FAC-B(config)# interface GigabitEthernet0/0
R-FAC-B(config-if)# ip ospf network point-to-point
```

---

#### Objective 3: EIGRP Passive-Interface Silent Blackholing (Symptom C)

**1. Functional Behavior of `passive-interface`:**
In Cisco EIGRP (Enhanced Interior Gateway Routing Protocol), configuring an interface as passive disables the transmission and reception of EIGRP multicast packets (sent to `224.0.0.10`). Specifically:
* The router ceases sending EIGRP Hello packets out of that interface.
* Inbound EIGRP Hellos received on that interface are discarded.
* Existing neighbor relationships on that physical link are immediately torn down upon Hold-timer expiration.
* However, the subnet attached to the passive interface is still included in the EIGRP topology table and advertised out of *other* active EIGRP interfaces.

**2. Why Route Advertisement Was Blocked Toward Facility C:**
In this scenario, `R-FAC-B` was configured with:
```cisco
router eigrp 100
 passive-interface GigabitEthernet0/1
```
Because `GigabitEthernet0/1` is the physical link directly connecting `R-FAC-B` to `R-FAC-C`:
1. `R-FAC-B` refused to establish an EIGRP neighbor relationship over `Gi0/1`.
2. As a result, no EIGRP UPDATE packets carrying Facility B's internal colocation prefixes (`10.20.20.0/24`) could be transmitted across the direct B-C interconnect.
3. If an EIGRP neighbor relationship remained "up and stable" elsewhere, it was either:
   * Formed across a secondary/redundant physical circuit, or
   * Formed via an OSPF-to-EIGRP redistribution boundary through Facility A that only advertised a subset of prefixes.
4. Hence, Facility C had no route to Facility B's internal subnets, creating a silent reachability blackhole.

**3. Remediation Command:**
```cisco
R-FAC-B(config)# router eigrp 100
R-FAC-B(config-router)# no passive-interface GigabitEthernet0/1
```

---

#### Objective 4: BGP Route-Map Directionality Misapplication (Symptom D)

**1. Inbound vs Outbound Route-Maps in BGP:**
* **Inbound Route-Map (`neighbor [IP] route-map [NAME] in`)**:
  * Evaluated when BGP UPDATE messages arrive from the remote peer.
  * Used to accept/reject incoming routes, modify path attributes (Local Preference, Weight, AS-Path prepending, MED), and influence outbound egress traffic selection.
* **Outbound Route-Map (`neighbor [IP] route-map [NAME] out`)**:
  * Evaluated before BGP UPDATE messages are transmitted to the remote peer.
  * Used to filter which prefixes from the local BGP table (Loc-RIB) are shared with the neighbor and manipulate attributes (AS-Path prepending, MED, communities) to influence inbound ingress traffic from the Internet.

**2. Analysis of the Observed Defect:**
The problem statement notes: *"Routes learned from one ISP are not being advertised onward to the other ISP as expected; investigation shows an outbound route-map was applied to the wrong BGP neighbor direction."*
* **Intended Transit Policy**: In multi-homed customer networks, an enterprise must avoid acting as an open transit Autonomous System between two commercial Tier-1 ISPs unless explicitly configured as a transit provider. If an enterprise advertises routes learned from ISP-1 onward to ISP-2, third-party Internet traffic will traverse Pinehollow’s private infrastructure, quickly saturating WAN circuits.
* **The Glitch**: An engineer intended to filter or allow specific prefixes using a route-map (e.g. `ISP-TRANSIT-FILTER`). However, the command was applied:
  ```cisco
  neighbor 198.51.100.1 route-map TRANSIT-POLICY in
  ```
  instead of `out`. Because the route-map matched on outbound attributes or local IP prefixes, applying it inbound caused all incoming prefixes from ISP-2 to be dropped or misclassified, or applying an inbound filter outbound blocked route advertisements entirely.

**3. Remediation Command:**
```cisco
R-FAC-C(config)# router bgp 65000
R-FAC-C(config-router)# no neighbor 198.51.100.1 route-map TRANSIT-POLICY in
R-FAC-C(config-router)# neighbor 198.51.100.1 route-map OUTBOUND-CUSTOMER-ONLY out
R-FAC-C(config-router)# exit
R-FAC-C# clear ip bgp 198.51.100.1 soft out
```

---

#### Objective 5: DNS Response Spoofing / Cache Poisoning Investigation (Symptom E)

**1. Attack Signature & Capture Pattern:**
The Wireshark capture at Facility B reveals a classic **Kaminsky-style DNS Cache Poisoning attack**:
* A single legitimate DNS query (`IN A domain.com`) is dispatched from recursive resolver `10.20.20.53`.
* Almost instantaneously, a massive flood of forged UDP/53 response packets arrives before the legitimate authoritative nameserver can respond.
* These responses feature:
  1. Identical query names and questions.
  2. Mismatched or brute-forced 16-bit **Transaction IDs (TXIDs)** (cycling rapidly through $0\text{x}0000$ to $0\text{xFFFF}$).
  3. Inconsistent or spoofed source IP addresses and source ports.
* Once one of the attacker’s forged packets successfully guesses the active TXID and destination UDP port of the pending query, the resolver accepts the packet as authentic, commits the spoofed IP mapping into its local cache, and serves fraudulent IP mappings to all internal hosts.

**2. Threat Intent:**
* **Adversary Objective**: Man-in-the-Middle (MitM) interception, credential harvesting, phishing, or malware distribution. By steering internal hosts from authentic corporate/external portals to an attacker-controlled server, the attacker can harvest administrative credentials or session tokens.

**3. Concrete Mitigations:**
1. **Source Port Randomization (SPR)**: Enforce recursive resolvers to randomize ephemeral source UDP ports across the full 16-bit range (1024–65535) for every outgoing recursive query. This expands the entropy space from 16 bits (65,536 combinations) to 32 bits ($65,536 \times 64,511 \approx 4.29 \times 10^9$ combinations), making blind brute-force guessing computationally infeasible before legitimate responses arrive.
2. **DNSSEC (DNS Security Extensions) Validation**: Enable cryptographically signed resource records (`RRSIG`, `DNSKEY`). When the resolver validates cryptographic digital signatures against the chain of trust originating at the DNS root, all spoofed packets lacking valid cryptographic signatures are discarded immediately.
3. **Network-Level Protections**:
   * Deploy Cisco IOS/ASA DNS Inspection with protocol compliance verification.
   * Apply strict egress/ingress Access Control Lists (ACLs) permitting external DNS UDP/53 traffic only with established authoritative server IPs.
   * Implement Response Rate Limiting (RRL) and Unicast Reverse Path Forwarding (uRPF) to block IP-spoofed packets.
