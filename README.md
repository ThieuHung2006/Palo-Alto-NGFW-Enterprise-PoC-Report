# Enterprise Multi-Zone Security Architecture, Dual-ISP BGP/OSPF Dynamic Routing Redundancy, and Perimeter Defense Engineering on Palo Alto Networks NGFW (PAN-OS)

**Author:** Network & Information Security Engineer / Blue Team Specialist  
**Platform:** Palo Alto Networks PA-VM (PAN-OS 10.1.0) on EVE-NG Emulation Platform  
**Target Audience:** Technical Hiring Managers, Senior Network Architects, Lead Security Engineers (SOC / Blue Team)

---

## Executive Summary

This document presents a comprehensive Proof of Concept (PoC) validating an enterprise-grade perimeter network architecture deployed on a **Palo Alto Networks Next-Generation Firewall (PA-VM, PAN-OS 10.1.0)**. The design integrates:
1. **Hierarchical 3-Tier Enterprise Networking** (Core, Distribution/Aggregation, Access) with intra-domain **OSPFv2**.
2. **Dual-Homed WAN Edge Routing** utilizing **eBGP** with dual Internet Service Providers (ISP1 & ISP2), implementing sub-second failover and automatic failback capabilities.
3. **Adaptive Network Address Translation (NAT/PAT)** that automatically tracks routing table convergence.
4. **Zero Trust Inter-Zone Micro-segmentation** leveraging application-level visibility (App-ID) and strict least-privilege security policies.
5. **Hardware-Level Perimeter Security (Zone Protection)** mitigating high-speed reconnaissance (TCP Port Scans) and state-table exhaustion attacks (TCP SYN Flood) at the firewall ingress stage before session allocation.

---

## 1. Enterprise Architecture Overview

The topology simulates a high-availability enterprise environment connecting three isolated security zones (`Untrust`, `DMZ`, and `Trust`) across dynamic routing boundaries.

![Figure 29: End-to-End Enterprise Network Architecture Topology on EVE-NG](./images/29_eve_ng_enterprise_network_topology.png)
*Markdown Embed Code:* `![Figure 29 - Enterprise Network Topology](./images/29_eve_ng_enterprise_network_topology.png)`

### Key Architectural Highlights for Technical Recruiters:
* **Zero Trust Security Zone Segmentation:** The firewall (`PaloAlto1`) acts as the single point of policy enforcement, partitioning traffic into `Untrust` (Dual-ISP WAN edge), `DMZ` (publicly exposed servers in `10.10.100.0/24` on `VLAN 100`), and `Trust` (internal corporate departments).
* **Dual-Homed WAN Edge Redundancy:** Interface `eth1/1` (`198.51.100.0/30`) peers with `ISP-R1` (Autonomous System `AS 65001`), while `eth1/2` (`198.51.200.0/30`) peers with `ISP-R2` (`AS 65002`). The two carrier routers are interconnected over `203.0.113.0/30`. A dedicated `Attacker` node (Kali Linux) is stationed at `e0/2` of `ISP-R1` to conduct realistic red-team penetration tests.
* **Resilient Core & Access Fabric:** Downstream connectivity links the firewall to dual multilayer core switches (`Core-SW1` via `eth1/4` `10.0.0.0/30` and `Core-SW2` via `eth1/5` `10.0.1.0/30`). A redundant cross-connect (`10.0.2.0/30` across `e0/2`–`e0/3`) prevents split-brain scenarios, aggregating downstream traffic across four access switches (`SW-Access1` through `SW-Access4`) serving `VLAN 10 (IT)` (`10.10.10.0/24`), `VLAN 20 (HR)` (`10.10.20.0/24`), and `VLAN 30 (Server)` (`10.10.30.0/24`).
* **Isolated DMZ Perimeter:** Interface `eth1/3` (`10.10.100.254`) bridges to `SW_DMZ`, hosting critical infrastructure including `Web_Server` (`10.10.100.10`) and `DNS_Server` (`10.10.100.20`), ensuring zero unrestricted traffic traversal between external clients and internal hosts.

---

## 2. Interface Provisioning & Object-Oriented Design

### 2.1. Layer 3 Interfaces and Security Zone Binding

![Figure 05: Layer 3 Interface Addressing, Virtual Router Binding, and Security Zone Mapping](./images/05_paloalto_network_interfaces_ip_zones.png)
*Markdown Embed Code:* `![Figure 05 - Layer 3 Interfaces and Zones](./images/05_paloalto_network_interfaces_ip_zones.png)`

#### Engineering Insights:
* All five operational physical interfaces (`ethernet1/1` through `ethernet1/5`) are configured in **Layer 3** mode under a unified global routing entity (`Virtual Router: default`), bound to the management profile `ALLOW-PING` for diagnostics.
* Dual upstream WAN interfaces (`ethernet1/1` with IP `198.51.100.2/30` and `ethernet1/2` with IP `198.51.200.2/30`) are mapped to the single **`Untrust`** zone. This design pattern simplifies rule evaluation and allows seamless multi-WAN failover without rewriting inter-zone policy headers.
* Interface `ethernet1/3` (`10.10.100.254/24`) serves as the default gateway for the `DMZ` zone, while `ethernet1/4` (`10.0.0.1/30`) and `ethernet1/5` (`10.0.1.1/30`) provide Point-to-Point routed uplinks to internal core switches within the `Trust` zone. `ethernet1/6` is assigned as a `Tap` interface for passive out-of-band monitoring.

---

### 2.2. Address Object Hierarchy and Pre-NAT / Post-NAT Structuring

![Figure 16: Centralized Address Objects Inventory Defining Network Subnets and Hosts](./images/16_paloalto_objects_addresses_inventory.png)
*Markdown Embed Code:* `![Figure 16 - Address Objects Inventory](./images/16_paloalto_objects_addresses_inventory.png)`

#### Engineering Insights:
* Adhering to the principle of **Object-Oriented Configuration**, hardcoded IP addresses are strictly avoided. Network subnets are abstracted into named objects: `NET-DMZ` (`10.10.100.0/24`), `NET-HR` (`10.10.20.0/24`), `NET-IT` (`10.10.10.0/24`), and `NET-SRV` (`10.10.30.0/24`).
* Host definitions for the Web Server strictly decouple the real internal IP (`WEB-DMZ`: `10.10.100.10/32`) from the publicly routed Virtual IP (`WEB-PUBLIC`: `192.0.2.10/32`). This explicit separation is essential for constructing PAN-OS Destination NAT (DNAT) policies, where security enforcement occurs after route evaluation but before packet egress.

---

## 3. Interior Dynamic Routing (OSPFv2) & Route Redistribution

### 3.1. Point-to-Point OSPF Interface Optimization & Metric Engineering

![Figure 01: OSPF Area 0.0.0.0 Interface Properties on Trust Uplinks (P2P Link Type)](./images/01_paloalto_ospf_interfaces_p2p_trust.png)
*Markdown Embed Code:* `![Figure 01 - OSPF Point-to-Point Interfaces](./images/01_paloalto_ospf_interfaces_p2p_trust.png)`

#### Engineering Insights:
* Uplink interfaces `ethernet1/4` and `ethernet1/5` within backbone `Area ID: 0.0.0.0` are configured with **`Link Type: p2p` (Point-to-Point)**. This bypasses the traditional Designated Router/Backup Designated Router (DR/BDR) election process on broadcast media, eliminating 40 seconds of election latency during link convergence.
* Asymmetric path cost engineering is deployed: `ethernet1/4` is assigned `Metric: 10` (Primary path via Core-SW1), whereas `ethernet1/5` is assigned `Metric: 20` (Standby path via Core-SW2). Timers are calibrated for rapid dead-link identification: `Hello Interval: 10`, `Dead Counts: 4`, `Retransmit Intervals: 5`, and `Transit Delay: 1`.

---

### 3.2. Routing Plane Hardening via OSPF Passive Interface on DMZ

![Figure 02: OSPF Passive Interface Configuration on DMZ Link (ethernet1/3)](./images/02_paloalto_ospf_passive_interface_dmz.png)
*Markdown Embed Code:* `![Figure 02 - OSPF Passive Interface DMZ](./images/02_paloalto_ospf_passive_interface_dmz.png)`

#### Engineering Insights:
* Interface `ethernet1/3` (serving the DMZ segment) is flagged as **`Passive`** within the OSPF area.
* **Security Rationale:** Activating passive mode ensures the firewall advertises the DMZ subnet (`10.10.100.0/24`) to downstream enterprise switches without transmitting OSPF Hello multicasts into the DMZ. This eliminates the risk of rogue OSPF neighbor formation or route poisoning attacks originating from a compromised DMZ host.

---

### 3.3. Redistribution Profile Filtering: BGP Default Route Injection

![Figure 03: Redistribution Profile RP-BGP-DEF Matching Inbound BGP Default Route](./images/03_paloalto_redistribution_profile_bgp_default.png)
*Markdown Embed Code:* `![Figure 03 - Redistribution Profile BGP Default Route](./images/03_paloalto_redistribution_profile_bgp_default.png)`

#### Engineering Insights:
* Redistribution profile `RP-BGP-DEF` (`Priority: 1`) is configured to match dynamic routes where `Source Type: bgp` and destination matches strictly `Destination: 0.0.0.0/0`.
* This selective filter prevents dynamic Internet prefixes learned from upstream transit providers from flooding the internal OSPF domain, protecting the finite memory of internal switches while propagating outbound routing reachability.

---

### 3.4. OSPF Export Rule: External Type 2 Propagation

![Figure 04: OSPF Export Rule Injecting BGP Default Route as External Type 2](./images/04_paloalto_ospf_export_rules_redistribute_bgp.png)
*Markdown Embed Code:* `![Figure 04 - OSPF Export Rules Redistribute BGP](./images/04_paloalto_ospf_export_rules_redistribute_bgp.png)`

#### Engineering Insights:
* The Virtual Router (`Router ID: 10.255.255.1`) exports `RP-BGP-DEF` into the internal OSPF topology as **`PATH TYPE: ext-2` (External Type 2)**. This maintains a predictable cost across internal routing hops.
* Inbound protection is enforced via `Reject Default Route` on the OSPF process, ensuring the firewall never accepts a downstream default route that could induce routing loops.

---

### 3.5. OSPF Runtime Adjacency Verification

![Figure 06: Runtime OSPF Neighbor Table Confirming FULL Adjacency with Dual Core Routers](./images/06_paloalto_ospf_neighbors_state_full.png)
*Markdown Embed Code:* `![Figure 06 - OSPF Neighbors Full State](./images/06_paloalto_ospf_neighbors_state_full.png)`

#### Engineering Insights:
* The operational state confirms `STATUS: full` within `AREA ID: 0.0.0.0` across both core distribution switches: `10.0.0.2` (`NEIGHBOR ROUTER ID: 10.255.255.2`, `Remaining Lifetime: 39s`) and `10.0.1.2` (`NEIGHBOR ROUTER ID: 10.255.255.3`, `Remaining Lifetime: 33s`).
* With `MESSAGES PENDING: 0`, the Link State Database (LSDB) is fully synchronized across the control plane.

---

## 4. Exterior Dynamic Routing: Dual-ISP eBGP & Traffic Engineering

### 4.1. Global BGP Autonomous System Parameters

![Figure 11: Global BGP Virtual Router Parameters (AS 65000, Router ID 10.255.255.1)](./images/11_paloalto_bgp_general_settings.png)
*Markdown Embed Code:* `![Figure 11 - BGP General Settings](./images/11_paloalto_bgp_general_settings.png)`

#### Engineering Insights:
* The enterprise firewall operates under private Autonomous System **`AS Number: 65000`**, with `Router ID: 10.255.255.1` and `Install Route` enabled to push selected BGP best paths from the Loc-RIB into the Forwarding Information Base (FIB).
* Crucially, `Reject Default Route` is intentionally **unchecked**, permitting the firewall to accept `0.0.0.0/0` advertisements from upstream providers. `Default Local Preference` is set to `100`, and `Deterministic MED comparison` is enabled to enforce RFC-compliant path selection across peers.

---

### 4.2. Upstream eBGP Peer Groups Definition

![Figure 09: BGP Peer Groups Overview for Carrier Interconnections (ISP1 & ISP2)](./images/09_paloalto_bgp_peer_groups_summary.png)
*Markdown Embed Code:* `![Figure 09 - BGP Peer Groups Summary](./images/09_paloalto_bgp_peer_groups_summary.png)`

#### Engineering Insights:
* To maintain isolated control-plane policies, the firewall establishes two distinct `ebgp` peer groups:
  * `ISP1`: Peering with `ISP1-R1` (`198.51.100.1`) via local interface `198.51.100.2/30`.
  * `ISP2`: Peering with `ISP2-R2` (`198.51.200.1`) via local interface `198.51.200.2/30`.

---

### 4.3. Detailed eBGP Peer Configuration: Primary & Secondary Paths

![Figure 13: eBGP Peer Details for ISP1-R1 (Peer AS 65001, Max Prefixes 5000)](./images/13_paloalto_bgp_peer_details_isp1.png)
*Markdown Embed Code:* `![Figure 13 - BGP Peer Details ISP1](./images/13_paloalto_bgp_peer_details_isp1.png)`

![Figure 14: eBGP Peer Details for ISP2-R2 (Peer AS 65002, Max Prefixes 5000)](./images/14_paloalto_bgp_peer_details_isp2.png)
*Markdown Embed Code:* `![Figure 14 - BGP Peer Details ISP2](./images/14_paloalto_bgp_peer_details_isp2.png)`

#### Engineering Insights:
* `ISP1-R1` belongs to transit provider `AS 65001`, configured as the primary gateway, while `ISP2-R2` belongs to `AS 65002` as the secondary failover path.
* **Control Plane Defense:** Both peers enforce **`MAX PREFIXES: 5000`**. This setting guards the firewall’s control plane against routing table exhaustion caused by upstream BGP misconfigurations or route leaks. Export and Import next-hop resolution is enforced through `Next Hop: Resolve`.

---

### 4.4. Inbound & Outbound Policy Filtering (Preventing AS Transit & Route Leaks)

![Figure 08: BGP Import Rules for ISP1 (ISP1-IN)](./images/08_paloalto_bgp_import_rules_isp1.png)
*Markdown Embed Code:* `![Figure 08 - BGP Import Rules ISP1](./images/08_paloalto_bgp_import_rules_isp1.png)`

![Figure 07: BGP Export Rules Enforcing Public Prefix Whitelisting and Explicit Deny-All](./images/07_paloalto_bgp_export_rules_public_prefix.png)
*Markdown Embed Code:* `![Figure 07 - BGP Export Rules Public Prefix](./images/07_paloalto_bgp_export_rules_public_prefix.png)`

#### Engineering Insights:
* **Inbound Traffic Steering:** Rule `ISP1-IN` assigns higher attributes (e.g., elevated Local Preference) to default routes learned from ISP1, designating ISP1 as the preferred egress path under steady-state conditions.
* **Outbound Whitelisting (Anti-Transit AS Architecture):** The export policy applies a strict white-list model: rule `ALLOW-PUBLIC` permits only the enterprise public block `192.0.2.0/24(exact)` to be advertised externally (`ACTION: allow`). Immediately below, `DENY-ALL` and `DENY-ALL2` drop all other route advertisements (`ACTION: deny`). This guarantees internal RFC 1918 prefixes are never leaked and prevents the enterprise from inadvertently operating as a transit network between ISP1 and ISP2.

---

### 4.5. Enterprise Public Service Prefix Redistribution

![Figure 10: BGP Redistribution Rules Summary for Public Service Subnets](./images/10_paloalto_bgp_redistribute_rules_summary.png)
*Markdown Embed Code:* `![Figure 10 - BGP Redistribution Rules Summary](./images/10_paloalto_bgp_redistribute_rules_summary.png)`

![Figure 12: Detailed Redistribution Rule Injecting Web VIP 192.0.2.10/32 into BGP](./images/12_paloalto_bgp_redistribute_rule_web_ip.png)
*Markdown Embed Code:* `![Figure 12 - BGP Redistribute Rule Web IP](./images/12_paloalto_bgp_redistribute_rule_web_ip.png)`

![Figure 17: Redistribution Rule Injecting Enterprise Public IP Block 192.0.2.0/24 into BGP](./images/17_paloalto_bgp_redistribute_rule_public_subnet.png)
*Markdown Embed Code:* `![Figure 17 - BGP Redistribute Rule Public Subnet](./images/17_paloalto_bgp_redistribute_rule_public_subnet.png)`

#### Engineering Insights:
* To ensure public web services hosted in the DMZ remain reachable over both ISP connections, host routes (`192.0.2.10/32`) and the aggregated enterprise public block (`192.0.2.0/24`) are redistributed into the BGP Loc-RIB with `Set Origin: incomplete`. This setup provides resilient ingress path options across the upstream providers.

---

### 4.6. eBGP Adjacency Convergence State

![Figure 15: BGP Peer Status Confirming Dual ESTABLISHED State across ISP1 and ISP2](./images/15_paloalto_bgp_peers_state_established.png)
*Markdown Embed Code:* `![Figure 15 - BGP Peers State Established](./images/15_paloalto_bgp_peers_state_established.png)`

#### Engineering Insights:
* The runtime BGP peer monitor verifies that both external peering sessions (`ISP1-R1` on `198.51.100.1` and `ISP2-R2` on `198.51.200.1`) have completed the TCP port 179 handshake and transitioned into **`STATUS: Established`**, demonstrating a synchronized exterior routing control plane.

---

## 5. Security Architecture: Adaptive NAT & Zero Trust Policies

### 5.1. Route-Adaptive Multi-ISP Source NAT & Inbound Destination NAT

![Figure 18: Complete NAT Policy Configuration (Inbound DNAT and Multi-ISP SNAT)](./images/18_paloalto_nat_policy_dnat_snat_rules.png)
*Markdown Embed Code:* `![Figure 18 - NAT Policy Configuration](./images/18_paloalto_nat_policy_dnat_snat_rules.png)`

![Figure 26: NAT Rule Hit Count Statistics Validating Primary SNAT Path Allocation](./images/26_paloalto_nat_policy_hit_count_isp1_active.png)
*Markdown Embed Code:* `![Figure 26 - NAT Policy Hit Counts](./images/26_paloalto_nat_policy_hit_count_isp1_active.png)`

#### Deep-Dive Technical Insights (PAN-OS Packet Flow):
* **Inbound DNAT Evaluation (`DNAT-Web-80`):** In Rule 1, both the `SOURCE ZONE` and `DESTINATION ZONE` in the Original Packet specification are configured as **`Untrust`**. Under PAN-OS packet processing, when a packet arrives on `ethernet1/1` destined for `WEB-PUBLIC` (`192.0.2.10`), the firewall performs an initial route lookup prior to NAT translation. Since `192.0.2.10` resides on an untrusted edge interface, the initial destination zone is evaluated as `Untrust`. After matching `DNAT-Web-80`, the destination IP is translated to `WEB-DMZ` (`10.10.100.10:80`). Subsequent route evaluation then vectors the translated packet to the `DMZ` zone.
* **Dynamic Egress-Bound SNAT (`SNAT-via-ISP1` & `SNAT-via-ISP2`):** Rules 2 and 3 govern outbound internet access for internal zones (`DMZ`, `Trust`) with `dynamic-ip-and-port` translation. Crucially, they differentiate traffic using **`DESTINATION INTERFACE`** (`ethernet1/1` for ISP1 and `ethernet1/2` for ISP2).
  * **Hit Count Proof:** During primary path operation, `SNAT-via-ISP1` recorded **34 hits**, while `SNAT-via-ISP2` remained at **0 hits**. When a failover event occurs, the FIB automatically updates the egress interface, steering packets to `SNAT-via-ISP2` without requiring any manual policy updates.

---

### 5.2. Layered Security Policy Ruleset (Least-Privilege Enforcement)

![Figure 19: Enterprise Security Policy Ruleset Implementing App-ID and Least Privilege](./images/19_paloalto_security_policy_ruleset.png)
*Markdown Embed Code:* `![Figure 19 - Security Policy Ruleset](./images/19_paloalto_security_policy_ruleset.png)`

#### Engineering Insights:
1. `IT-to-DMZ-Admin`: Restricts administration to `NET-IT` sources within the `Trust` zone, allowing only explicitly verified management applications (`ping`, `ssh`, `ssl`, `web-browsing`) over standard application ports (`application-default`).
2. `HR-to-DMZ-Web` & `HR-to-DMZ-Deny`: The Human Resources department (`NET-HR`) is restricted to standard web services (`dns`, `ssl`, `web-browsing`). All administrative protocols, including ICMP ping, are explicitly rejected by rule 3 (`HR-to-DMZ-Deny`, `Action: Deny`).
3. `Trust-to-Internet`: Permits corporate users outbound internet connectivity across verified business applications.
4. `Internet-to-Web`: Illustrates the foundational PAN-OS security rule principle: **"Pre-NAT Destination Address with Post-NAT Destination Zone"**. Destination Address references the external VIP (`WEB-PUBLIC`), while Destination Zone specifies the post-translation zone (`DMZ`), restricting inbound traffic to `ssl` and `web-browsing`.
5. `DMZ-to-Trust-Deny`: Blocks all inbound connections initiated from the `DMZ` toward internal subnets (`Trust`). If an external web server is compromised, the attacker cannot pivot into the corporate intranet.
6. `Cleanup`: Acts as an explicit deny-all rule to log all unauthorized connection attempts.

---

### 5.3. Layer 7 Threat Profile Integration & Licensing Architecture

![Figure 20: Security Profile Group SPG-Strict Combining L7 Deep Packet Inspection](./images/20_paloalto_security_profile_groups_strict.png)
*Markdown Embed Code:* `![Figure 20 - Security Profile Group SPG-Strict](./images/20_paloalto_security_profile_groups_strict.png)`

![Figure 20a: Vulnerability Protection Profile Strict Ruleset Configuration](./images/image_c96219.png)
*Markdown Embed Code:* `![Figure 20a - Vulnerability Protection Profile Strict Rules](./images/image_c96219.png)`

![Figure 20b: Binding Security Profile Groups and Session Logging on Security Policy Rules](./images/image_c962b2.png)
*Markdown Embed Code:* `![Figure 20b - Security Policy Rule Actions and Profile Attachment](./images/image_c962b2.png)`

#### Engineering Insights:
* The `SPG-Strict` profile group bundles four inspection engines: `Anti-Spyware: strict`, `Vulnerability Protection: strict`, `URL Filtering: default`, and `File Blocking: strict file blocking`.
* Under the `strict` profile, attacks categorized under `critical`, `high`, and `medium` severity for both client and server traffic are configured to **`reset-both`**, terminating connections at both endpoints.
* **Architectural Note on PAN-OS Licensing:** In evaluation environments without an active Threat Prevention license (`vm-license: none`), the dataplane bypasses L7 signature matching to preserve throughput. L3/L4 enforcement, dynamic routing, NAT, App-ID, and Zone Protection function fully at line rate regardless of licensing status.

---

## 6. Security Policy & NAT Functional Verification

### 6.1. IT Department DMZ Administrative Access Verification

![Figure 21: Detailed Traffic Log Confirming Authorized ICMP Access from IT to DMZ](./images/21_paloalto_traffic_log_detailed_it_to_dmz.png)
*Markdown Embed Code:* `![Figure 21 - Traffic Log IT to DMZ Allow](./images/21_paloalto_traffic_log_detailed_it_to_dmz.png)`

#### Verification Results:
* **Session ID:** 62 | **Application:** `ping` | **Action:** `allow` (`Action Source: from-policy`)
* **Source:** `10.10.10.2` (`NET-IT`, `Zone: Trust`, `Interface: ethernet1/4`)
* **Destination:** `10.10.100.10` (`WEB-DMZ`, `Zone: DMZ`, `Interface: ethernet1/3`)
* **Analysis:** Verifies authorized inter-zone access per the `IT-to-DMZ-Admin` security policy.

---

### 6.2. Corporate Internet Outbound Access & Dynamic SNAT Verification

![Figure 22: Detailed Traffic Log Confirming Outbound Internet Access and SNAT via ISP1](./images/22_paloalto_traffic_log_detailed_trust_to_internet_snat.png)
*Markdown Embed Code:* `![Figure 22 - Traffic Log Trust to Internet SNAT ISP1](./images/22_paloalto_traffic_log_detailed_trust_to_internet_snat.png)`

#### Verification Results:
* **Session ID:** 140 | **Application:** `ping` | **Action:** `allow` per `Trust-to-Internet`
* **Source:** `10.10.10.2` translated to **`NAT IP: 198.51.100.2`** (Interface `ethernet1/1`)
* **Destination:** `8.8.8.8` (`Zone: Untrust`, `Port: 0`)
* **Analysis:** Confirms that outbound enterprise traffic correctly matches `SNAT-via-ISP1`, translating internal private IPs to the primary public WAN IP.

---

### 6.3. Inter-VLAN Micro-segmentation: HR DMZ Isolation

![Figure 23: Detailed Traffic Log Demonstrating Immediate Denial of HR-to-DMZ Traffic](./images/23_paloalto_traffic_log_detailed_hr_to_dmz_deny.png)
*Markdown Embed Code:* `![Figure 23 - Traffic Log HR to DMZ Deny](./images/23_paloalto_traffic_log_detailed_hr_to_dmz_deny.png)`

#### Verification Results:
* **Session ID:** 0 | **Application:** `ping` | **Action:** `deny` (`TYPE: drop`)
* **Policy Matched:** `HR-to-DMZ-Deny` | **Session End Reason:** `policy-deny`
* **Source:** `10.10.20.2` (`NET-HR`, `Zone: Trust`) $\rightarrow$ **Destination:** `10.10.100.10` (`Zone: DMZ`)
* **Analysis:** Confirms that unauthorized lateral network movement from non-technical workstations is dropped at the ingress stage (`Session ID: 0`, `0 bytes`).

---

### 6.4. Reverse Pivot Prevention: DMZ-to-Trust Isolation

![Figure 24: Detailed Traffic Log Demonstrating Complete Isolation of DMZ-to-Trust Traffic](./images/24_paloalto_traffic_log_detailed_dmz_to_trust_deny.png)
*Markdown Embed Code:* `![Figure 24 - Traffic Log DMZ to Trust Deny](./images/24_paloalto_traffic_log_detailed_dmz_to_trust_deny.png)`

#### Verification Results:
* **Session ID:** 0 | **Application:** `ping` | **Action:** `deny` (`TYPE: drop`)
* **Policy Matched:** `DMZ-to-Trust-Deny` | **Session End Reason:** `policy-deny`
* **Source:** `10.10.100.10` (`Zone: DMZ`) $\rightarrow$ **Destination:** `10.10.20.2` (`Zone: Trust`)
* **Analysis:** Proves the DMZ cannot initiate traffic into the internal network, mitigating pivot risks should an internet-facing host be compromised.

---

### 6.5. Inbound Destination NAT Publication Verification

![Figure 25: Detailed Traffic Log Confirming Inbound DNAT and Web Service Delivery](./images/25_paloalto_traffic_log_detailed_dnat_internet_to_web.png)
*Markdown Embed Code:* `![Figure 25 - Traffic Log Inbound DNAT Internet to Web](./images/25_paloalto_traffic_log_detailed_dnat_internet_to_web.png)`

#### Verification Results:
* **Session ID:** 570 | **Application:** `web-browsing` | **Action:** `allow` via `Internet-to-Web`
* **Original Packet:** Source `198.51.100.1:53233` $\rightarrow$ Destination **`192.0.2.10:80`** (`Untrust`, `ethernet1/1`)
* **Translated Packet:** Destination translated to **`10.10.100.10:80`** (`DMZ`, `ethernet1/3`)
* **Analysis:** Validates that external HTTP clients successfully access the internal DMZ web server through the public virtual IP address.

---

## 7. High Availability & Convergence: BGP Failover & Failback

### 7.1. BGP Dynamic Routing State Machine Convergence Logs

![Figure 27: System Event Log Sequence Demonstrating BGP Carrier Link Withdrawal and Recovery](./images/27_paloalto_system_log_bgp_failover_established.png)
*Markdown Embed Code:* `![Figure 27 - System Log BGP Failover Events](./images/27_paloalto_system_log_bgp_failover_established.png)`

#### State Machine Analysis:
The PAN-OS `System Logs` (`TYPE: routing`) confirm rapid link failure detection and re-convergence:
1. **Link Severance:** At `03:05:37` and `03:10:52`, administrative downing of the primary `ISP1` link triggered a high-severity alert: `routed-BGP-peer-left-established` (peer `198.51.100.1`). The BGP engine withdrew the primary default route and shifted traffic to `ISP2-R2` in **sub-second time**.
2. **Path Re-establishment:** Upon interface recovery at `03:05:47` and `03:10:57`, the BGP control plane initiated capabilities exchange (`routed-BGP-peer-mp-extension-negotiate` for `AFI/SAFI 1/1`) and transitioned back to `routed-BGP-peer-enter-established` within seconds.

---

### 7.2. Traffic Continuity & Dynamic SNAT Rerouting (Failover vs. Failback)

![Figure 28: Continuous Traffic Stream Demonstrating Uninterrupted Outbound Connectivity During Failover](./images/28_paloalto_traffic_log_continuous_ping_failover_test.png)
*Markdown Embed Code:* `![Figure 28 - Continuous Ping Traffic Log During Failover](./images/28_paloalto_traffic_log_continuous_ping_failover_test.png)`

![Figure 30: Detailed Traffic Log Demonstrating Dynamic SNAT Rerouting via ISP2 During Failover](./images/30_paloalto_traffic_log_failover_active_isp2.png)
*Markdown Embed Code:* `![Figure 30 - Traffic Log Failover Active on ISP2](./images/30_paloalto_traffic_log_failover_active_isp2.png)`

![Figure 31: Detailed Traffic Log Confirming Automatic Failback and SNAT Restoration via ISP1](./images/31_paloalto_traffic_log_failback_restored_isp1.png)
*Markdown Embed Code:* `![Figure 31 - Traffic Log Failback Restored on ISP1](./images/31_paloalto_traffic_log_failback_restored_isp1.png)`

#### End-to-End Validation Summary:
* **Service Continuity:** As shown in Figure 28, continuous outbound ICMP streams from `10.10.10.2` to `8.8.8.8` remained active across failure transitions (`03:06:17`, `03:07:17`, `03:10:07`, `03:10:22`, `03:10:42`).
* **Automated Failover (Figure 30, Session 167):** When the primary link failed, traffic seamlessly rerouted out `Interface: ethernet1/2` and translated to **`NAT IP: 198.51.200.2`** (ISP2), confirming dynamic NAT policy realignment driven by routing table updates.
* **Automated Failback (Figure 31, Session 162):** When ISP1 recovered, its higher Local Preference restored it as the active path. Outbound sessions shifted back to `Interface: ethernet1/1` and translated to **`NAT IP: 198.51.100.2`** (ISP1) without administrative intervention.

---

## 8. Perimeter Defense: L3/L4 Reconnaissance & DoS Attack Mitigation

### 8.1. Attack Simulation via Kali Linux (Red Team Perspective)

![Figure 32a: Kali Linux Route Table Provisioning for Target Network Reachability](./images/image_c88802.png)
*Markdown Embed Code:* `![Figure 32a - Kali Linux Nmap Route Verification](./images/image_c88802.png)`

![Figure 32b: Nmap Reconnaissance Probes Blocked by Firewall Stealth Baseline](./images/image_c88f86.png)
*Markdown Embed Code:* `![Figure 32b - Kali Linux Nmap Ping Probe Blocked](./images/image_c88f86.png)`

![Figure 32c: High-Volume TCP SYN Flood Attack Executed via hping3](./images/image_c8f1f8.png)
*Markdown Embed Code:* `![Figure 32c - Kali Linux hping3 SYN Flood Execution](./images/image_c8f1f8.png)`

#### Attack Execution Walkthrough:
* **Reconnaissance Testing:** The attacker node (`172.16.0.2` behind ISP1) targeted public IP `192.0.2.10`. An initial sweep using `nmap -sS` was thwarted by default probe suppression (`Host seems down... try -Pn`), verifying the firewall drops unsolicited ICMP discovery traffic. Bypassing host discovery with `sudo nmap -Pn -sS -p 21,22,23,25,53,80,443,444,445... 192.0.2.10` launched a concurrent port scan across exposed services.
* **Denial of Service (DoS) Attack:** To test state exhaustion defenses, `hping3` was deployed:
  ```bash
  sudo hping3 -S -p 80 --flood 192.0.2.10
  ```
  The attack flooded the target with **3,594,555 TCP SYN packets** (`40 headers + 0 data bytes`) to exhaust state resources.

---

### 8.2. Reconnaissance Protection: High-Rate TCP Port Scan Mitigation

![Figure 32: Threat Log Demonstrating Hardware-Level Ingress Drop of Nmap Port Scan Probes](./images/32_paloalto_threat_log_zone_protection_tcp_port_scan_drop.png)
*Markdown Embed Code:* `![Figure 32 - Threat Log TCP Port Scan Drop](./images/32_paloalto_threat_log_zone_protection_tcp_port_scan_drop.png)`

#### Key Defense Insights for Technical Interviewers:
* **Threat Classification:** At `03:54:38`, the firewall flagged the scan from `Source: 172.16.0.2` on `Interface: ethernet1/1` as **`TYPE: scan`**, assigned **`SEVERITY: medium`**, and applied **`Action: drop`**.
* **Pre-Session Mitigation Architecture:**
  * **`Session ID: 0`:** The probe packets were dropped at the hardware ingress stage before session creation, preventing session table consumption.
  * **`Rule: Blank`:** The packets were dropped by the **Zone Protection Profile** prior to security policy evaluation, conserving CPU cycles during high-rate scans.

---

### 8.3. DoS Flood Mitigation: SYN Cookies Defense Mechanism

![Figure 33: Threat Log Confirming SYN Cookie Proxy Activation Under Heavy Flood Attack](./images/33_paloalto_threat_log_zone_protection_syn_flood_cookie.png)
*Markdown Embed Code:* `![Figure 33 - Threat Log SYN Flood Cookie Mitigation](./images/33_paloalto_threat_log_zone_protection_syn_flood_cookie.png)`

#### Detailed Defense Mechanics:
* **Threat Classification:** When inundated by over 3.59 million SYN packets, the firewall raised a critical alert: **`TYPE: flood`**, **`SEVERITY: critical`** (`Protocol: tcp`, `Zone: Untrust`).
* **SYN Cookie Proxy (`Action: allow`, `Session ID: 0`):**
  * When the SYN rate exceeds the profile's `Activate Rate`, the firewall activates **SYN Cookie mode**.
  * Instead of allocating memory for half-open connections, the firewall responds with a cryptographic challenge encoded in the TCP Sequence Number (`Action: allow` indicates the challenge packet was transmitted; `Session ID: 0` confirms no session was allocated).
  * Only clients that return a valid `ACK` completing the 3-way handshake are allocated a state table entry and forwarded to the DMZ web server. This protects both the firewall and backend servers from connection table exhaustion.

---

## 9. PoC Acceptance & Validation Matrix

| Test ID | Technical Validation Objective | Test Methodology & Execution | Empirical Results & Verified Artifacts |
| :--- | :--- | :--- | :--- |
| **TC-01** | OSPFv2 Interior Routing & DMZ Hardening | Configured OSPF `0.0.0.0` with `Link Type: p2p`, enabled `Passive` on DMZ, verified neighbors. | Reached `STATUS: full` with dual core switches (`10.255.255.2`, `10.255.255.3`). Injected `0.0.0.0/0` via external type `ext-2`. |
| **TC-02** | Dual-ISP eBGP Peering & Route Filtering | Configured eBGP (`AS 65000`) with ISP1 (`AS 65001`) and ISP2 (`AS 65002`); enforced prefix limits and export whitelisting. | Both peers reached `STATUS: Established`. Successfully advertised `192.0.2.0/24(exact)` while blocking internal RFC 1918 routes. |
| **TC-03** | Zero Trust Inter-Zone Policy & NAT/PAT | Tested connectivity across `NET-IT`, `NET-HR`, `DMZ`, and external internet hosts (`8.8.8.8`). | Authorized IT access to DMZ (`Session ID: 62`); blocked HR access (`HR-to-DMZ-Deny`); prevented DMZ reverse traversal (`DMZ-to-Trust-Deny`); verified DNAT to `10.10.100.10:80`. |
| **TC-04** | Sub-Second BGP Failover & Automatic Failback | Severed primary ISP1 link during active traffic (`ping 8.8.8.8`), monitored convergence, and restored the link. | Outbound sessions dynamically failed over to ISP2 (`ethernet1/2`, `NAT IP: 198.51.200.2`), then automatically failed back to ISP1 (`ethernet1/1`, `NAT IP: 198.51.100.2`). |
| **TC-05** | L3/L4 Perimeter Defense & DoS Mitigation | Ran `nmap -Pn -sS` port scans and launched `3,594,555` SYN flood packets using `hping3` from Kali Linux. | Dropped scan probes at ingress (`TYPE: scan`, `Action: drop`, `Session ID: 0`). Mitigated flood attack via SYN Cookies (`TYPE: flood`, `Severity: critical`, `Session ID: 0`). |

---

## 10. Key Engineering Takeaways & Professional Competencies

1. **Production-Ready Routing Architecture:** Combines exterior eBGP multihoming with interior OSPFv2, managing route redistribution and filtering to ensure resilient network reachability.
2. **PAN-OS Flow Logic Mastery:** Successfully deploys Pre-NAT address definitions alongside Post-NAT zone constructs, coordinating dynamic NAT with routing table updates across multiple WAN interfaces.
3. **Defense-in-Depth Engineering:** Implements layered controls that combine Zero Trust inter-zone policies with ingress Zone Protection profiles, dropping malicious traffic early to safeguard system resources.
