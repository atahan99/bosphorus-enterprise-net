# Packet Tracer support matrix

What Packet Tracer 9 supports for this design, verified hands-on rather than assumed. Tests are defined in [`phase0/phase0-feature-tests.md`](../phase0/phase0-feature-tests.md).

**Packet Tracer build tested:** 9.0.1.0858
**Legend:** ✅ works as designed · ⚠️ partial or fallback syntax only · ❌ not supported · ⏳ not yet tested

## Phase 0 results

| ID | Feature | Device(s) | Result | Notes (rejected command / fallback used) |
|---|---|---|---|---|
| P0-01 | `securityk9` license | ISR4331, ISR2911 | ⏳ | |
| P0-02a | Root guard | 3650 | ⏳ | |
| P0-02b | Loop guard (interface) | 2960, 3650 | ⏳ | |
| P0-02c | Loop guard (global default) | 2960 | ⏳ | |
| P0-03 | Err-disable recovery for bpduguard | 2960 | ⏳ | |
| P0-04 | DHCP snooping + `no ip dhcp snooping information option` | 2960 | ⏳ | |
| P0-05a | Dynamic ARP Inspection | 2960 | ⏳ | |
| P0-05b | DAI ARP ACL filter for static hosts | 2960 | ⏳ | |
| P0-06 | Wired 802.1X with RADIUS | 2960 + Server-PT | ⏳ | |
| P0-07a | HSRP v2 with MD5 | 3650 | ⏳ | |
| P0-07b | HSRP object tracking (`track` + `decrement`) | 3650 | ⏳ | |
| P0-08a | OSPF MD5 per area | ISR4331 | ⏳ | |
| P0-08b | OSPF `area range` summarisation | ISR4331 | ⏳ | |
| P0-08c | OSPF `auto-cost reference-bandwidth`, point-to-point network type | ISR4331 | ⏳ | |
| P0-09 | EIGRP MD5 key chain | ISR4331, ISR2911 | ⏳ | |
| P0-10 | Redistribution with route-map tags | ISR4331 | ⏳ | |
| P0-11a | BGP on Router-PT | Router-PT | ⏳ | |
| P0-11b | iBGP between loopbacks (`update-source`, `next-hop-self`) | ISR4331 | ⏳ | |
| P0-11c | BGP outbound prefix-list | ISR4331 | ⏳ | |
| P0-11d | AS-path prepend via route-map | ISR4331 | ⏳ | |
| P0-11e | `default-originate` | Router-PT | ⏳ | |
| P0-11f | BGP neighbor password | ISR4331, Router-PT | ⏳ | |
| P0-11g | Inbound prefix-list rejects partner-edge default over iBGP | ISR4331 | ⏳ | Required for symmetric NAT failover |
| P0-12 | OSPF `default-information originate metric` | ISR4331 | ⏳ | |
| P0-13 | IP SLA + object tracking on static route | ISR4331 | ⏳ | |
| P0-14a | IKE `hash sha256` | ISR4331 | ⏳ | |
| P0-14b | IKE `group 14` | ISR4331 | ⏳ | |
| P0-14c | ESP `esp-sha256-hmac` | ISR4331 | ⏳ | |
| P0-14d | IPsec transport mode for GRE | ISR4331 | ⏳ | |
| P0-15a | Named ACL for NAT with deny exemption | ISR4331 | ⏳ | |
| P0-15b | ZBF (inspect HTTP/HTTPS/DNS/ICMP) | ISR4331 | ⏳ | |
| P0-15c | ZBF + NAT + crypto map + GRE on one router | ISR4331 | ⏳ | |
| P0-16a | TACACS+ with local fallback | ISR2911 | ⏳ | |
| P0-16b | `login block-for` | ISR2911 | ⏳ | |
| P0-16c | Authenticated NTP | ISR2911 | ⏳ | |
| P0-16d | SNMP community with ACL | ISR2911 | ⏳ | |
| P0-17a | MQC marking (`set ip dscp`) | ISR2911 | ⏳ | |
| P0-17b | MQC queuing (LLQ, CBWFQ, fair-queue) | ISR2911 | ⏳ | |
| P0-17c | MQC queuing | ISR4331 | ⏳ | |

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
| B-22 | Static route with `track` | LON-EDGE-02 | ⏳ | Covered by P0-13; fallback: untracked route (Chicago failover then needs EDGE-02 itself to fail) |
| B-23 | MCP device/interface names match the GUI and cabling plan | 3650-24PS, 5506-X, Router-PT, modular routers | ⏳ | Run the Phase 2 throwaway preflight; record any accepted MCP-to-GUI port mapping before production placement |

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
