# Correlation Narrative & Incident Trigger Analysis
## Case Study No. 30: Pinehollow Data Centers — Capstone Diagnostics

---

### Incident Correlation Assessment: Single Trigger vs. Multiple Unrelated Faults

A rigorous diagnostic analysis of the symptom log reveals that the five observed anomalies across Pinehollow Data Centers do **not** stem from a single overarching root cause or single technical trigger. Instead, the incident log demonstrates a classic multi-event operational cascade comprising **two independent operational categories**: (1) a series of uncoordinated, manual maintenance changes executed without pre-flight validation, and (2) an opportunistic, concurrent external cyberattack targeting architectural vulnerabilities.

#### Technical Evidence Supporting Disjoint Triggers

1. **Protocol & Architectural Isolation**:
   The faults reside within fundamentally independent control planes and operational layers across distinct physical sites:
   * **Symptom A (EtherChannel)** is a local Layer 2 ASIC load-balancing hashing behavior confined exclusively to the switching backplane between `SW-B1` and `SW-B2` in Facility B. Layer 2 hashing algorithms have zero functional interaction with OSPF state machines, EIGRP route advertisements, or BGP policy routing.
   * **Symptom B (OSPF Network-Type)** is an interior gateway protocol (IGP) point-to-point state-machine failure across the Facility A–B inter-site WAN link, caused by a local interface command omission (`ip ospf network point-to-point`).
   * **Symptom C (EIGRP Passive-Interface)** is an interior routing policy suppression on the Facility B–C boundary router (`R-FAC-B`). While also in Facility B, it reflects a manual CLI error inside `router eigrp 100` that silenced updates on `Gi0/1`.
   * **Symptom D (BGP Route-Map)** resides exclusively on the Autonomous System boundary router at Facility C (`R-FAC-C`), involving policy directionality (`in` vs. `out`) on external border gateway peering with Tier-1 ISPs.
   * **Symptom E (DNS Cache Poisoning)** is an active network security incident originating from malicious actors exploiting spoofed UDP packets to hijack name resolution.

2. **Temporal & Human Factor Correlation (The "Maintenance Window" Illusion)**:
   While the timing suggests a common 48-hour incident window, this clustering is characteristic of **uncoordinated multi-site maintenance activity** (e.g., an aggressive migration project attempting to standardize routing protocols across facilities). An engineering team likely attempted to transition Facility B’s legacy EIGRP network toward the OSPF backbone while simultaneously adjusting BGP edge peering at Facility C. During this hurried change window, engineers introduced multiple human configuration errors: misconfiguring OSPF network types, applying an erroneous `passive-interface` statement, applying BGP route-maps in the wrong direction, and leaving default EtherChannel hashing in place.

3. **Security Incident Coincidence**:
   Symptom E is strictly an external cyber exploit. However, its success was compounded by the operational instability: when link saturation (Symptom A) and routing flapped, network monitoring visibility was degraded, allowing the DNS spoofing attempt to proceed unhindered.

#### Conclusion

The five symptoms represent **multiple unrelated configuration faults** coupled with an independent **external security attack**, clustered during an unvetted operational maintenance cycle. Treating this outage as a single monolithic bug would lead to flawed remediation; each symptom requires isolated, protocol-specific corrective action and strict change-control governance.
