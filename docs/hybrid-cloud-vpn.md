# Hybrid Cloud VPN — On-Prem to AWS (IKEv2/IPsec)

Site-to-site IPsec tunnel extending the on-prem lab fabric into an AWS VPC. The tunnel terminates on `asav-0` — the same firewall already performing NAT and inspection for the internal network — rather than a dedicated VPN appliance.

> Public IPs and pre-shared keys in this document are placeholders. Substitute your own.

---

## 1. Topology

```
PC4 (VLAN 40)
  └─ ACC4 ─ DIST2 (HSRP active) ─ CORE1 (OSPF) ─ asav-0
                                                    │ outside 192.168.1.51/24
                                                    │
                                          [ Home router — NAT ]
                                            public <HOME-PUBLIC-IP>
                                                    │
                                            ~ Internet (IKEv2) ~
                                                    │
                                          EC2 <EC2-PUBLIC-IP>
                                          strongSwan, 10.99.0.12
                                                    │
                                          AWS VPC 10.99.0.0/24
```

**Design note:** the ASA sits behind the home router's NAT, so it must be the tunnel **initiator**. AWS cannot reach inward to start negotiation. This also means IKEv2 NAT-Traversal applies — see §6.

---

## 2. Addressing

| Element | Value |
|---|---|
| On-prem tunnelled subnet | `10.0.40.0/24` (VLAN 40 — SERVERS) |
| ASA outside interface | `192.168.1.51/24`, gw `192.168.1.254` |
| On-prem public IP | `<HOME-PUBLIC-IP>` |
| AWS VPC CIDR | `10.99.0.0/24` |
| AWS public subnet | `10.99.0.0/25` (strongSwan instance) |
| AWS private subnet | `10.99.0.128/28` |
| strongSwan private IP | `10.99.0.12` |
| strongSwan public IP | `<EC2-PUBLIC-IP>` |

Tunnel selectors: `10.0.40.0/24` ↔ `10.99.0.12/32`

---

## 3. Crypto parameters

| Phase | Parameters |
|---|---|
| IKEv2 (P1) | AES-256-CBC, SHA-256, DH group 14, PSK, lifetime 86400s |
| IPsec (P2) | ESP AES-256, SHA-256, lifetime 3600s |
| DPD | 30s delay, action restart |
| Encapsulation | NAT-T (UDP 4500) — auto-negotiated |

Both ends must match exactly. Mismatch presents as `NO_PROPOSAL_CHOSEN`.

---

## 4. ASA configuration (`asav-0`)

```
! Phase 1
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14
 prf sha256
 lifetime seconds 86400
!
crypto ikev2 enable outside

! Phase 2
crypto ipsec ikev2 ipsec-proposal AWS-PROPOSAL
 protocol esp encryption aes-256
 protocol esp integrity sha-256

! Interesting traffic
access-list VPN-TO-AWS extended permit ip 10.0.40.0 255.255.255.0 host 10.99.0.12

! Crypto map
crypto map OUTSIDE-MAP 10 match address VPN-TO-AWS
crypto map OUTSIDE-MAP 10 set peer <EC2-PUBLIC-IP>
crypto map OUTSIDE-MAP 10 set ikev2 ipsec-proposal AWS-PROPOSAL
crypto map OUTSIDE-MAP interface outside

! Peer authentication
tunnel-group <EC2-PUBLIC-IP> type ipsec-l2l
tunnel-group <EC2-PUBLIC-IP> ipsec-attributes
 ikev2 remote-authentication pre-shared-key <PSK>
 ikev2 local-authentication pre-shared-key <PSK>

! Specific route for AWS subnet — see Lesson 1
route outside 10.99.0.0 255.255.255.0 192.168.1.254 1

! NAT exemption — see Lessons 2 and 3
object network VLAN40-SUBNET
 subnet 10.0.40.0 255.255.255.0
!
object network AWS-HOST
 host 10.99.0.12
!
nat (any,outside) 1 source static VLAN40-SUBNET VLAN40-SUBNET destination static AWS-HOST AWS-HOST no-proxy-arp route-lookup
```

---

## 5. strongSwan configuration (EC2)

Ubuntu 24.04+ ships strongSwan 6.x, which **removed the legacy `ipsec.conf` / starter daemon**. Configuration uses `swanctl` only.

```bash
sudo apt install strongswan strongswan-swanctl charon-systemd -y
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

`/etc/swanctl/swanctl.conf`:

```
connections {
    asa-tunnel {
        version = 2
        local_addrs  = %any
        remote_addrs = <HOME-PUBLIC-IP>
        proposals = aes256-sha256-modp2048
        dpd_delay = 30s

        local  { auth = psk
                 id   = <EC2-PUBLIC-IP> }
        remote { auth = psk
                 id   = %any }

        children {
            asa-child {
                local_ts  = 10.99.0.12/32
                remote_ts = 10.0.40.0/24
                esp_proposals = aes256-sha256
                start_action  = trap
                dpd_action    = restart
            }
        }
    }
}

secrets {
    ike-asa {
        id = %any
        secret = "<PSK>"
    }
}
```

```bash
sudo systemctl enable --now strongswan
sudo swanctl --load-all      # re-run after every config edit
```

**`remote id = %any` is required.** The ASA identifies itself as `192.168.1.51`, but packets arrive from `<HOME-PUBLIC-IP>`. Strict identity matching would reject the peer.

---

## 6. Supporting infrastructure

**Home router**
- Port forward UDP 500 and UDP 4500 → CML host
- ESP (protocol 50) forwarding not required — NAT-T encapsulates ESP in UDP 4500. Consumer routers generally cannot forward protocol 50; this is not a blocker.

**AWS**
| Item | Setting |
|---|---|
| Source/destination check | **Disabled** on the strongSwan instance |
| Security group inbound | UDP 500, UDP 4500, ESP — source `<HOME-PUBLIC-IP>/32`; ICMP from `10.0.40.0/24` |
| NAT Gateway | Not deployed (~$33/mo avoided; private subnet needs no outbound internet) |

---

## 7. Verification

**ASA**
```
show crypto ikev2 sa     → Status: UP-ACTIVE, READY, Role: INITIATOR
show crypto ipsec sa     → #pkts encaps and #pkts decaps both incrementing
```

**strongSwan**
```
sudo swanctl --list-sas  → ESTABLISHED / INSTALLED
```

**End to end**
```
PC4:~$ ping 10.99.0.12
8 packets received, ~48ms avg
```

First packet loss on cold start is expected — IKE negotiates while packet 1 is queued.

---

## 8. Troubleshooting lessons

Three faults stacked, each masking the next. All were diagnosed with `packet-tracer`, which walks a simulated packet through every ASA processing phase and names the exact drop point.

```
packet-tracer input inside-core1 icmp 10.0.40.11 8 0 10.99.0.12 detailed
```

**Lesson 1 — Overly broad static route caused a hairpin drop**
`route inside-core1 10.0.0.0 255.0.0.0 10.0.12.22` matched `10.99.0.12`, so the ASA routed the packet back out its ingress interface. Dropped as `acl-drop` with `input_ifc == output_ifc`.
*Fix:* add a specific `route outside 10.99.0.0 255.255.255.0`. Longest-prefix match wins.
*Symptom vs cause:* presented as an ACL problem; was a routing problem.

**Lesson 2 — Dynamic PAT clobbered the crypto ACL**
The pre-existing `nat (any,outside) source dynamic INTERNAL-NET interface` translated `10.0.40.11 → 192.168.1.51` before crypto evaluation. The translated source no longer matched `VPN-TO-AWS`, so the crypto map never fired and traffic left as ordinary internet traffic.
*Fix:* identity NAT exemption (`source static X X destination static Y Y`).
*Detection:* no `VPN`/`encrypt` phase in the packet-tracer output.

**Lesson 3 — NAT rule order**
The exemption was configured but landed *after* the dynamic rule. Both are manual (twice) NAT in section 1, where first match wins, so the PAT rule still won.
*Fix:* insert at an explicit position — `nat (any,outside) 1 source static ...`.

**Lesson 4 — strongSwan 6.x dropped `ipsec.conf`**
`strongswan-starter` has no installation candidate on current Ubuntu. Most tutorials online still use the legacy format. Use `swanctl.conf` and `swanctl --load-all`.

---

## 9. Known limitations

| Limitation | Notes |
|---|---|
| Single tunnel, no redundancy | Matches the existing single-firewall SPOF; a second peer would require a second public endpoint |
| Tunnel scoped to one host | `10.99.0.12/32` — widen `local_ts` and the ASA ACL to `10.99.0.128/28` to reach the full private subnet |
| PSK authentication | Certificate-based auth is preferred in production |
| Dynamic on-prem public IP | If the ISP-assigned address changes, `remote_addrs` on strongSwan must be updated |
