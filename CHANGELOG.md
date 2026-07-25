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

## v3.0 — Hybrid Cloud VPN

Site-to-site IKEv2/IPsec tunnel from the on-prem fabric to an AWS VPC, terminated on `asav-0` — the same firewall already handling NAT and inspection, rather than a dedicated VPN appliance.

- IKEv2 (AES-256/SHA-256/DH14) + ESP (AES-256/SHA-256), PSK auth
- Tunnel selectors: `10.0.40.0/24` ↔ AWS `10.99.0.12/32`
- ASA is initiator — it sits behind the home router's NAT; NAT-T encapsulates ESP in UDP 4500
- AWS side: VPC `10.99.0.0/24`, strongSwan 6.x on EC2 (`swanctl`, not legacy `ipsec.conf`)
- No NAT Gateway deployed (~$33/mo avoided)

📄 Full design, configs, and troubleshooting: [`docs/hybrid-cloud-vpn.md`](./docs/hybrid-cloud-vpn.md)

**Validated:** end-to-end reachability PC4 → AWS EC2 through the tunnel, ~48ms RTT, symmetric encaps/decaps counters on the ASA, `ESTABLISHED`/`INSTALLED` on strongSwan.
