# Firewall Edge Design

This document covers the internet-facing firewall edge added on top of the base 3-tier fabric described in [`3-Tier_Enterprise_Network_Lab_v4.docx`](./3-Tier_Enterprise_Network_Lab_v4.docx). It went through three distinct architectures before landing on the current design — each iteration is documented here because the reasoning behind rejecting the first two is arguably more valuable than the final config alone.

---

## Iteration 1 — ASA Active/Standby Failover Pair (Rejected — topology mismatch)

**Design:** ASAv-1 and ASAv-2 configured as a classic Cisco failover pair — `failover lan unit primary/secondary`, a dedicated failover link, shared `standby` addresses on inside/outside interfaces.

**Initial wiring (wrong):**
```
CORE1 ── ASAv-1 ── ext-conn-0
CORE2 ── ASAv-2 ── ext-conn-1
```
Each firewall had its **own private point-to-point links** — not a shared segment.

**Why it failed:** ASA failover requires both units to sit on the *same* L2 segment on each interface, so the standby can silently monitor the active unit and take over the same IP instantly. Two independent point-to-point links are not a shared segment — `show failover` reported `Interface inside: Failed (Waiting)` because ASAv-2 had nothing to monitor on its own isolated link to CORE2.

---

## Iteration 2 — Two Independent Firewalls (Rejected — asymmetric routing)

**Design:** abandoned failover clustering. ASAv-1 protected CORE1's uplink, ASAv-2 protected CORE2's uplink, each fully independent — separate NAT, separate default routes, no shared state.

**Why it's a valid pattern in general:** legitimate for dual-ISP edges or deliberately separated trust zones, where "path diversity" (not seamless takeover) is the goal.

**Why it broke here:** once both CORE1 and CORE2 originated a default route into OSPF, DIST/ACC saw two equal-cost paths and ECMP-load-balanced across them. Since ASAv-1 and ASAv-2 don't share a connection-state table, a session that exited through one and returned through the other was silently dropped — classic **asymmetric routing** against stateful firewalls.

**Mitigation identified (not required in the end):** policy-based routing to deterministically pin VLANs to a specific firewall, avoiding ECMP-driven asymmetry. Documented as the correct fix for this pattern, but superseded by Iteration 3.

---

## Iteration 3 — Single Firewall, Three Legs (Final / Implemented)

**Design:** one ASAv (`asav-0`) with three interfaces — no HA pair, no shared switches, no PBR needed.

```
CORE1 ── Gi0/0 (inside-core1) ──┐
                                  asav-0 ── Gi0/2 (outside) ── ext-conn-0
CORE2 ── Gi0/1 (inside-core2) ──┘
```

**Why this avoids Iteration 2's problem entirely:** there's only one connection-state table, because it's the same physical device. A session can legitimately enter via CORE1's leg and be tracked correctly regardless of which core it came from — no asymmetric-drop risk, structurally.

**Explicit tradeoff accepted:** this is a genuine **single point of failure (SPOF)** for the internet path. If `asav-0` fails, both cores lose internet access simultaneously — no partial availability (unlike Iteration 2) and no instant failover (unlike Iteration 1). This is a deliberate, stated simplification, common in smaller real-world deployments that accept the risk in exchange for lower cost and complexity.

**ECMP static routing caveat:** ASA does not support two static routes to the same destination at equal administrative distance across different interfaces — attempting this returns `ERROR: Cannot add route entry, conflict with existing routes`. Solved with a **floating static route** (primary AD 1, backup AD 254) so CORE2's path only activates if CORE1's fails.

---

## Final Addressing Plan (Iteration 3)

| Link | Device | Address |
|---|---|---|
| CORE1 Gi6 ↔ asav-0 Gi0/0 | CORE1 | `10.0.12.21/30` |
| | asav-0 (inside-core1) | `10.0.12.22/30` |
| CORE2 Gi6 ↔ asav-0 Gi0/1 | CORE2 | `10.0.12.25/30` |
| | asav-0 (inside-core2) | `10.0.12.26/30` |
| asav-0 Gi0/2 ↔ ext-conn-0 | asav-0 (outside) | `192.168.255.95/24` |
| | Gateway (CML host bridge) | `192.168.255.1` |

---

## Final Configuration

### asav-0
```
hostname ASAV-0
enable password ciscolab
!
interface GigabitEthernet0/0
 nameif inside-core1
 security-level 100
 ip address 10.0.12.21 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/1
 nameif inside-core2
 security-level 100
 ip address 10.0.12.25 255.255.255.252
 no shutdown
!
interface GigabitEthernet0/2
 nameif outside
 security-level 0
 ip address 192.168.255.95 255.255.255.0
 no shutdown
!
object network INTERNAL-NET
 subnet 10.0.0.0 255.0.0.0
!
nat (any,outside) source dynamic INTERNAL-NET interface
!
route inside-core1 10.0.0.0 255.0.0.0 10.0.12.22 1
route inside-core2 10.0.0.0 255.0.0.0 10.0.12.26 254
route outside 0.0.0.0 0.0.0.0 192.168.255.1 1
!
policy-map global_policy
 class inspection_default
  inspect icmp
!
write memory
```

### CORE1 (edge-facing interface only)
```
interface GigabitEthernet6
 ip address 10.0.12.22 255.255.255.252
 negotiation auto
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.12.21
!
write memory
```

### CORE2 (edge-facing interface only)
```
interface GigabitEthernet6
 ip address 10.0.12.26 255.255.255.252
 negotiation auto
 no shutdown
!
ip route 0.0.0.0 0.0.0.0 10.0.12.25
!
write memory
```

> `default-information originate` is configured under CORE1's `router ospf 1` process so the default route propagates to DIST/ACC automatically.

---

## Redundancy — What's Covered, What Isn't

| Failure | Protected? | Mechanism |
|---|---|---|
| One core router dies | ✅ Yes | OSPF reconvergence, dual-core mesh to DIST |
| One distribution switch dies | ✅ Yes | HSRP failover, dual uplinks to core |
| One EtherChannel member/link dies | ✅ Yes | LACP redundancy within each Po bundle |
| One CORE-to-firewall link dies | ✅ Yes | Floating static route (AD 1 → AD 254) |
| The firewall itself dies | ❌ No | Single device — SPOF by design |
| The ISP/internet circuit dies | ❌ No | Single `ext-conn-0` — out of scope (would need dual-ISP + BGP) |

This design deliberately covers the two most common internal failure domains (core, distribution) but knowingly accepts the firewall and the internet circuit as single points of failure — a documented tradeoff, not an oversight. Iteration 1 (the HA pair) is the proven fix if firewall redundancy becomes a requirement;

---

## Key Lessons

1. **NAT/DHCP leftovers after inline device insertion** — inserting a firewall between a router and its uplink leaves stale `ip nat outside` / `ip address dhcp` config on the router if not explicitly cleaned up; caught and fixed on both CORE1 and CORE2.
2. **ASA doesn't inspect ICMP by default** — transit ping tests fail through an ASA until `inspect icmp` is added to `policy-map global_policy`; traffic originated *by* the ASA itself works fine regardless, which is a useful diagnostic signal.
3. **Failover requires shared L2 segments, not just failover commands** — the CLI can be perfectly correct and failover will still show `Failed (Waiting)` if the physical topology doesn't provide a shared segment to monitor.
4. **ECMP + stateful firewalls without shared state = asymmetric routing** — a subtle, realistic production failure mode, diagnosed here via intermittent ping loss traced back to a stale DHCP-installed default route acting as a rogue second ASBR.
5. **ASA static routing has no built-in ECMP** — floating static routes (AD-based backup) are the correct mechanism for primary/backup paths to the same destination.

---

## DHCP Server Note

CORE1 additionally runs as the centralized DHCP server for all four internal VLANs (pools `DATA`, `GUESTS`, `MGMT`, `SERVERS`), relayed via `ip helper-address` on the DIST1/DIST2 SVIs. This is unrelated to the firewall edge itself but is part of the same CORE1 device and worth noting here since it was verified and corrected (pool naming) during this phase of the build.
