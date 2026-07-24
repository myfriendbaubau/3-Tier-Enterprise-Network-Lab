# Changelog

## v1.0 — Base 3-Tier Fabric

Core / Distribution / Access network built in Cisco Modeling Labs.

- OSPF area 0 across Core and Distribution
- L3 routed EtherChannel (LACP) — Core ↔ Distribution
- L2 EtherChannel + 802.1Q trunking — Distribution ↔ Access
- HSRP first-hop redundancy, active/standby split per VLAN
- Deterministic STP root placement aligned with active HSRP gateway
- VTP version 2 for VLAN propagation
- L2 security: BPDU Guard, port-security, DHCP snooping
- Named ACL (`MGMT-RESTRICT`) restricting inter-VLAN management access
- Centralized DHCP service from CORE1, relayed via `ip helper-address`

📄 Full spec: [`docs/3-Tier_Enterprise_Network_Lab_v4.docx`](./docs/3-Tier_Enterprise_Network_Lab_v4.docx)

**Validated:**
- OSPF adjacencies FULL on all Core–Distribution links
- HSRP failover tested (DIST1 shutdown → active role moves to DIST2 cleanly)
- EtherChannel member-link removal tested (traffic continues uninterrupted)
- `MGMT-RESTRICT` confirmed blocking VLAN 10/20/40 → VLAN 30 on both distribution switches

---

## v2.0 — Firewall Edge Added

Internet-facing firewall edge added on top of the v1.0 fabric. Evaluated three architectures before settling on the final design:

1. **ASA active/standby failover pair** — rejected: failover requires a shared L2 segment, not independent point-to-point links
2. **Two independent firewalls** (one per core) — rejected: OSPF ECMP across two stateful firewalls with no shared connection state causes asymmetric routing and silent drops
3. **Single firewall, three legs** (adopted) — no shared-state requirement, no ECMP asymmetry risk; accepted the firewall itself as a documented single point of failure in exchange for simplicity

📄 Full design writeup, addressing, and configs: [`docs/firewall-edge-design.md`](./docs/firewall-edge-design.md)

**Validated:**
- End-to-end internet reachability confirmed from all four VLANs through the firewall edge
- Floating static route confirmed providing backup path if one core-to-firewall link fails
- ICMP inspection gap identified and fixed (`inspect icmp` required for transit ping tests through ASA)
