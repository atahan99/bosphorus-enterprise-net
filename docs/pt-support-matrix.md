# Packet Tracer support matrix

What Packet Tracer 9 supports for this design, verified hands-on rather than assumed. Tests are defined in [`phase0/phase0-feature-tests.md`](../phase0/phase0-feature-tests.md).

**Packet Tracer build tested:** 9.0.1.0858
**Legend:** ✅ works as designed · ⚠️ partial or fallback syntax only · ❌ not supported · ⏳ not yet tested

## Phase 0 results

| ID | Feature | Device(s) | Result | Notes (rejected command / fallback used) |
|---|---|---|---|---|
| P0-01 | `securityk9` license | ISR4331, ISR2911 | ⚠️ | Active on both ISR4331s. The ISR2911 accepted the boot command but remained `security disable`; do not make the 2911 responsible for features that need this package. |
| P0-02a | Root guard | 3650 | ✅ | Persisted as `spanning-tree guard root`; the test port entered root-inconsistent when it received superior BPDUs. |
| P0-02b | Loop guard (interface) | 2960, 3650 | ❌ | The interface command did not persist on either switch. |
| P0-02c | Loop guard (global default) | 2960 | ❌ | `show spanning-tree summary` remained `Loopguard Default is disabled`. |
| P0-03 | Err-disable recovery for bpduguard | 2960 | ❌ | Configuration did not persist and `show errdisable recovery` is invalid. |
| P0-04 | DHCP snooping + `no ip dhcp snooping information option` | 2960 | ✅ | PC1 received `10.10.20.100`; its MAC/IP/VLAN/port appeared in the snooping binding table. |
| P0-05a | Dynamic ARP Inspection | 2960 | ❌ | Trust on the uplink persists, but global DAI enable/show commands are absent and an unbound static host was not blocked. |
| P0-05b | DAI ARP ACL filter for static hosts | 2960 | ❌ | `arp access-list` and `ip arp inspection filter` are invalid. |
| P0-06 | Wired 802.1X with RADIUS | 2960 + Server-PT | ❌ | `dot1x`, RADIUS/AAA authentication and interface authentication commands did not persist; both show commands are invalid. |
| P0-07a | HSRP v2 with MD5 | 3650 | ⚠️ | HSRP v2, VIP, priority and preempt persist and `10.10.20.1` answers; the MD5 authentication line is rejected. Use unauthenticated HSRP in the lab. |
| P0-07b | HSRP object tracking (`track` + `decrement`) | 3650 | ❌ | Track object and HSRP decrement did not persist; `show track` is invalid. |
| P0-08a | OSPF MD5 per area | ISR4331 | ✅ | Neighbor reached FULL; a forced wrong key dropped it and restoring the key recovered it. |
| P0-08b | OSPF `area range` summarisation | ISR4331 | ✅ | R1 learned only `O IA 10.20.0.0/16`, not the component /24s. |
| P0-08c | OSPF `auto-cost reference-bandwidth`, point-to-point network type | ISR4331 | ✅ | Both persisted; interface reported POINT-TO-POINT and cost 10. |
| P0-09 | EIGRP MD5 key chain | ISR4331, ISR2911 | ✅ | Neighbor formed and `10.40.20.0/24` was learned; a wrong key dropped the neighbor and restoring it recovered it. |
| P0-10 | Redistribution with route-map tags | ISR4331 | ⚠️ | Route-map syntax is invalid. Plain two-way redistribution works, but routes have tag 0; keep one redistribution point and no tag-based loop prevention. |
| P0-11a | BGP on Router-PT | Router-PT | ✅ | Two eBGP neighbors established and `203.0.113.0/24` was learned. |
| P0-11b | iBGP between loopbacks (`update-source`, `next-hop-self`) | ISR4331 | ❌ | PT reports: `does not support internal BGP ... Only external neighbors are supported`. |
| P0-11c | BGP outbound prefix-list | ISR4331 | ❌ | Both `ip prefix-list` and neighbor `prefix-list` are invalid. |
| P0-11d | AS-path prepend via route-map | ISR4331 | ❌ | `route-map` and `set as-path prepend` are invalid. |
| P0-11e | `default-originate` | Router-PT | ❌ | Neighbor `default-originate` is invalid. |
| P0-11f | BGP neighbor password | ISR4331, Router-PT | ❌ | Neighbor `password` is invalid. |
| P0-11g | Inbound prefix-list rejects partner-edge default over iBGP | ISR4331 | ❌ | Neither iBGP nor prefix-lists are available; the production BGP policy must be simplified for PT. |
| P0-12 | OSPF `default-information originate metric` | ISR4331 | ⚠️ | The `metric` option is invalid. Plain `default-information originate` works and produced an E2 default with metric 1. |
| P0-13 | IP SLA + object tracking on static route | ISR4331 | ❌ | `ip sla`, SLA tracking and a static route with `track` are invalid. |
| P0-14a | IKE `hash sha256` | ISR4331 | ⚠️ | `sha256` is invalid; `hash sha` works. |
| P0-14b | IKE `group 14` | ISR4331 | ⚠️ | Group 14 is invalid; group 5 works. |
| P0-14c | ESP `esp-sha256-hmac` | ISR4331 | ⚠️ | SHA-256 HMAC is invalid; `esp-sha-hmac` works with AES-256. |
| P0-14d | IPsec transport mode for GRE | ISR4331 | ⚠️ | Transform-set submode/`mode transport` is unavailable; default tunnel mode works. GRE ping succeeded and ISAKMP reached `QM_IDLE`. |
| P0-15a | Named ACL for NAT with deny exemption | ISR4331 | ✅ | Named extended ACL and PAT worked; PC2 reached `3.3.3.3` through R1. |
| P0-15b | ZBF (inspect HTTP/HTTPS/DNS/ICMP) | ISR4331 | ✅ | Zone pair and inspect policy persisted; ICMP counter increased by four packets. |
| P0-15c | ZBF + NAT + crypto map + GRE on one router | ISR4331 | ✅ | PC2 Internet ping, GRE/IPsec (5/5) and eBGP all remained operational together. |
| P0-16a | TACACS+ with local fallback | ISR2911 | ✅ | Legacy TACACS server and `group tacacs+ local` authentication persisted. |
| P0-16b | `login block-for` | ISR2911 | ✅ | Persisted with the requested thresholds. |
| P0-16c | Authenticated NTP | ISR2911 | ✅ | Authentication key, trusted key and keyed server all persisted. |
| P0-16d | SNMP community with ACL | ISR2911 | ❌ | ACL 99 persisted, but the SNMP community command did not. |
| P0-17a | MQC marking (`set ip dscp`) | ISR2911 | ✅ | Input policy attached and reported `QoS Set dscp ef`. |
| P0-17b | MQC queuing (LLQ, CBWFQ, fair-queue) | ISR2911 | ✅ | Output policy contains priority, bandwidth and fair-queue classes. |
| P0-17c | MQC queuing | ISR4331 | ✅ | Output policy attached to Gi0/0/1 and reported LLQ/CBWFQ classes. |

## Checked during the build

Commands the configs rely on that aren't covered by Phase 0. Each config has the fallback next to the line; record what happened here.

| ID | Feature | Device(s) | Result | Notes |
|---|---|---|---|---|
| B-01 | `ip tacacs source-interface Loopback0` | ISR4331, ISR2911, 3650 | ⏳ | Fallback: register the outgoing interface IP as the AAA client |
| B-02 | `redistribute ospf ... match internal external 1 external 2` into EIGRP | ISR4331 | ⏳ | Needed for Toronto's default route |
| B-03 | `eigrp stub connected` | ISR2911 | ⏳ | |
| B-04 | `ip tcp adjust-mss` on tunnels | ISR4331 | ⏳ | |
| B-05 | `aaa authentication ppp default local` | ISR4331, ISR2911 | ⏳ | |
| B-06 | ZBF `pass` with ACL class-map including `established` | ISR4331 | ⏳ | INSIDE ↔ DMZ (ADR 0011) |
| B-07 | ASA twice NAT (`nat (inside,outside) source static ... destination static ...`) | ASA 5506-X | ⏳ | VPN NAT exemption |
| B-08 | ASA `crypto ikev1 policy` with `group 14` | ASA 5506-X | ⏳ | Fallback group 5, then 2, on both ends |
| B-09 | ASA `inspect icmp` in global_policy | ASA 5506-X | ⏳ | |
| B-10 | ASA `management-access inside`, `logging host`, `ntp server` | ASA 5506-X | ⏳ | |
| B-11 | WLC management interface tagged VLAN 10 | WLC-3504 | ⏳ | Fallback: native VLAN 10 on the WLC trunk |
| B-12 | WPA2-Enterprise on Laptop-PT / Smartphone-PT | Wireless clients | ⏳ | |
| B-13 | HSRP track on ISR4331 (`track 2 interface ... line-protocol`) | ISR4331 | ⏳ | |
| B-14 | WPA2-Enterprise (RADIUS) on AccessPoint-PT | Branch APs | ⏳ | Fallback: WPA2-PSK `Lab-Corp-Wifi-1` on branch corporate SSID |
| B-15 | IOS DHCP server on 3650 (split scopes) | Branch switch pairs | ⏳ | Fallback: relay with `ip helper-address 10.10.80.10` on the SVIs and add the branch scopes to LON-SRV-CORE-01 |
| B-16 | `ip summary-address eigrp` on serial | ISR2911 | ⏳ | Fallback: no summary; London sees Toronto's /24s |
| B-17 | EIGRP stub + `passive-interface default` on 3650 | TOR-SW-01/02 | ⏳ | |
| B-18 | ASA second inside interface + floating static route | CHI-FW-01 | ⏳ | Fallback: single inside link to CHI-SW-01 (Chicago loses switch-level WAN redundancy) |
| B-19 | ASA crypto map with two peers | CHI-FW-01 | ⏳ | Fallback: single peer 198.51.100.6 |
| B-20 | PoE from 3650-24PS to IP phones and AccessPoint-PT | Branch switches | ⏳ | Fallback: power adapters |
| B-21 | DHCP snooping and DAI on 3650 | Branch switches | ⏳ | |
| B-22 | Static route with `track` | LON-EDGE-02 | ❌ | Covered by P0-13; use the documented untracked-route fallback. |
| B-23 | MCP device/interface names match the GUI and cabling plan | 3650-24PS, 5506-X, Router-PT, modular routers | ⚠️ | Phase 0 confirmed live 3650 names `Gi1/0/1`–`Gi1/0/24` and Router-PT `Fa0/0`/`Fa1/0`; raw live links worked. The MCP static 3650 catalog is wrong, and ASA/module mappings still require the Phase 2 preflight. |

## Known unsupported, with the substitute used

| Production feature | Status in PT | Used instead |
|---|---|---|
| VRRP / GLBP | Not available | HSRP |
| DMVPN, FlexVPN, IPsec VTI | Not available | GRE over IPsec with crypto maps |
| MPLS L3VPN | Not available | Internet + IPsec |
| VSS / StackWise | Not available | HSRP + STP across two cores |
| Private VLANs | Not available | Inter-VLAN ACLs |
| PoE access switches | 2960-24TT has no PoE | Power adapters on London phones and APs (branch 3650s do supply PoE) |
| FlexConnect APs | Not available | Autonomous AccessPoint-PT at branches (ADR 0013) |
| Carrier Ethernet / MPLS private circuit | Not modelled | Serial link with PPP |
| DNS forwarding on Server-PT | Not available | External records added to the internal DNS server |

## Production would add

Items outside what PT can show, listed so the gaps are explicit: SNMPv3, CoPP, UDLD, IKEv2 with VTI or DMVPN, 802.1X on every access port with MAB for phones and printers, NetFlow to a collector, WLC HA pair, FlexConnect or cloud-managed branch APs, 48-port PoE access switches, SRST at the branches, a multi-chassis/stateful core system with redundant control planes, a second VPN peer for Chicago and Toronto, a second circuit (or LTE backup) for Toronto, and SD-WAN in place of hand-built tunnels. _This list is extended as Phase 0 marks items ❌._
