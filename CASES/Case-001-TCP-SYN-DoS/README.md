# SOC Investigation Case Study: TCP SYN Flood Validation
## 1. Overview
 * **Objective:** Validate a suspected Denial-of-Service (DoS) condition on a target web service by correlating cross-layer telemetry rather than relying on isolated alert outputs.
 * **Core Analytical Skill:** Multi-source evidence correlation (IDS Alerts \rightarrow Firewall Flow Statistics \rightarrow Packet-Level Inspection) to arrive at a definitive incident disposition.
 * **Environment:** Segmented, isolated security testing topology involving an external sensor, perimeter firewall, and web application host.
 * **Primary Metrics:** Time to Detect **2.050 seconds**; incident disposition of **True Positive**.
## 2. Investigation Timeline & Telemetry Correlation
```
[07:08:38.399] Attack Simulation Begins
       │
       ├──► [07:08:40.449] Suricata Alert Generated (2.050s)
       │
       ├──► OPNsense Flow Analysis: 99% Interface Traffic Spike from 10.100.100.25
       │
       └──► Wireshark PCAP Validation: 886,917 Unanswered SYNs / Minimal ACKs

```
## 3. Detailed Phase-by-Phase Triage
### Phase 1: Initial Detection & Alert Assessment (Suricata IDS)
 * **Observed Data:** Suricata generated alerts identifying TCP SYN flood patterns directed at 10.100.100.23:80.
 * **Analyst Perspective:** Automated IDS signatures can produce false positives during legitimate traffic spikes or load testing. This alert was logged as an initial hypothesis requiring further correlation across firewall and packet data.
 ![[suricata-alert.png]]
### Phase 2: Traffic Flow & Volume Scoping (OPNsense Firewall)
 * **Observed Data:** NetFlow/traffic statistics on OPNsense showed that ~99% of total interface bandwidth was consumed by traffic directed to the single web service from origin 10.100.100.25.
 * **Analyst Perspective:** The concentration of traffic rules out a general network-wide anomaly or broad subnet scan, confirming a targeted resource exhaustion attempt against a specific host.

![[flow-statistics 2.png]]

### Phase 3: Packet-Level Verification (Wireshark Deep Inspection)
 * **Observed Data:** Deep packet analysis of the 77-second capture interval (1,370,284 total packets) revealed a peak volume of 7.5 × 10^6 SYN packets per second.
 * **Analyst Perspective:** The critical metric was connection state asymmetry: 886,917 outbound SYN packets were transmitted by the source host with negligible completed handshakes (ACKs) returned. This structural imbalance confirms an active half-open connection flood rather than high-volume legitimate web browsing.
 
![[Wireshark-FlowGraph 1.png]]
## 4. Evidence Matrix
| Source | Findings | Analytical Contribution |
|---|---|---|
| **Suricata IDS** | ET DROP MC-SMC IPv4 TCP SYN Flood Rate Exceeded | Initial anomaly indicator & hypothesis generation |
| **OPNsense** | 99% traffic concentration from 10.100.100.25 to 10.100.100.23:80 | Scope isolation & host attribution |
| **Wireshark** | Severe SYN-to-ACK ratio imbalance over 77-second capture | Definitive proof of incomplete handshakes (True Positive) |
## 5. Disposition & Defensive Recommendations
 * **Incident Disposition:** **True Positive** — Confirmed Network Denial of Service (MITRE ATT&CK **T1498**).
 * **Immediate Containment:**
   * Implement temporary access control lists (ACLs) or null-routing on OPNsense for host 10.100.100.25.
   * Enable kernel-level SYN cookies (tcp_syncookies) on the target web host to preserve socket availability.