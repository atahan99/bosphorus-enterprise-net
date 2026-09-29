# Packet Tracer support matrix

What Packet Tracer 9 supports for this design, verified hands-on rather than assumed. Tests are defined in [`phase0/phase0-feature-tests.md`](../phase0/phase0-feature-tests.md).

**Packet Tracer build tested:** 9.0.1.0858
**Images tested:** ISR4331 IOS XE (default image), ISR2911 IOS 15.1(4)M4 with `securityk9`, 3650-24PS (default image), 2960-24TT **IOS 15.0(2)SE4** (upgraded from the default 12.2(25)FX; see B-24), Router-PT (default image)
**Legend:** ✅ works as designed · ⚠️ partial or fallback syntax only · ❌ not supported · ⏳ not yet tested

A result counts only when the line is in `show running-config` afterwards. Script-driven runs can silently lose lines to a login prompt, an interactive prompt (the licence EULA) or the wrong submode; several first-run results were artefacts of exactly that and were corrected in the 2026-09-29 re-run.

## Phase 0 results

| ID | Feature | Device(s) | Result | Notes (rejected command / fallback used) |
|---|---|---|---|---|
| P0-01 | `securityk9` license | ISR4331, ISR2911 | ✅ | Active on both ISR4331s and on the ISR2911. The 2911 prompts `ACCEPT? [yes/no]` for the EULA; the first run never answered it, so the licence was not set. Answer `yes`, `write memory`, reload; `show license feature` then shows securityk9 enabled. |
| P0-02a | Root guard | 3650 | ⚠️ | Persists as `spanning-tree guard root`. With SW1 root (priority 4096) and SW2 forced to priority 0, SW1 logged `%SPANTREE-2-ROOTGUARDBLOCK` and showed Gi1/0/1 `BLK` for VLAN 20, but PC2 → SRV1 across that port still returned 4/4. PT enforces it in STP state only, not on the data plane. The block also does not clear by itself after the superior BPDUs stop: VLAN 20 stayed split between SW1 and SW2 until Gi1/0/1 was bounced with `shutdown` / `no shutdown`. The first run also had SW1 on PVST and SW2 on Rapid PVST with neither set as root; always set the mode and root priority explicitly. |
| P0-02b | Loop guard (interface) | 2960 (IOS 15), 3650 | ❌ | `spanning-tree guard loop` → `% Invalid input` on both. |
| P0-02c | Loop guard (global default) | 2960 (IOS 15), 3650 | ❌ | `spanning-tree loopguard default` → `% Invalid input` on both. |
| P0-03 | Err-disable recovery for bpduguard | 2960 (IOS 15), 3650 | ❌ | `errdisable recovery cause ...` and `errdisable recovery interval` → `% Invalid input` on both. Recover ports with `shutdown` / `no shutdown`. BPDU guard itself (`spanning-tree bpduguard enable`) is accepted. |
| P0-04 | DHCP snooping + `no ip dhcp snooping information option` | 2960, 3650 | ✅ | PC1 received `10.10.20.100` and appears in the snooping binding table (re-verified on IOS 15). All snooping lines are also accepted on the 3650. |
| P0-05a | Dynamic ARP Inspection | 2960 (IOS 15) | ⚠️ | IOS 15 accepts `ip arp inspection vlan`, `validate src-mac dst-mac ip` and interface `trust` (the 12.2 image lacks the global command). VLAN 20 shows `Active` and the counters increase, but an unbound static host (SRV1) and a spoofed IP (PC1 set to static .99) were both permitted: PT does not drop. Keep it in the configs as a demonstrable control, not an enforced one. |
| P0-05b | DAI ARP ACL filter for static hosts | 2960 (IOS 15) | ❌ | `arp access-list` and `ip arp inspection filter` → `% Invalid input`. |
| P0-06 | Wired 802.1X with RADIUS | 2960 (IOS 15) + Server-PT | ⚠️ | On IOS 15 all of these are accepted: `aaa new-model`, `radius-server host ... key`, `aaa authentication dot1x default group radius`, `dot1x system-auth-control`, interface `authentication port-control auto` and `dot1x pae authenticator`. `dot1x port-control auto` and the `radius server` / `address ipv4` block form are rejected. With the port in auto mode, simulation shows SW2 dropping PC1's DHCP with `The PDU is dropped because the port is in unauthorized state`. The successful-login path never completed: with Server-PT AAA set up (client SW2 / 10.10.20.3 / Radius / key, user `dot1x-user`, port 1645) and PC1's Desktop → IP Configuration → 802.1X set to MD5, PC1 never sent EAPOL. Re-ticking the setting, bouncing the port and renewing DHCP did not help. Treat 802.1X as a switch-side control that PT's PC-PT cannot authenticate against. The 12.2 image rejects every dot1x command. |
| P0-07a | HSRP v2 with authentication | 3650, ISR4331 | ⚠️ | HSRP v2, VIP, priority and preempt persist and `10.10.20.1` answers. `standby N authentication md5 key-string`, `authentication text` and the bare string form are all rejected on both platforms. Use unauthenticated HSRP. |
| P0-07b | HSRP object tracking | 3650, ISR4331 | ❌ | `track N interface ... line-protocol`, `standby N track N decrement` and the legacy `standby N track <interface> <decrement>` are all rejected. |
| P0-08a | OSPF MD5 per area | ISR4331 | ✅ | Neighbor reached FULL; a forced wrong key dropped it and restoring the key recovered it. |
| P0-08b | OSPF `area range` summarisation | ISR4331 | ✅ | R1 learns only `10.20.0.0/16`, not the component /24s. |
| P0-08c | OSPF `auto-cost reference-bandwidth`, point-to-point network type | ISR4331 | ✅ | Both persisted; interface reported POINT-TO-POINT and cost 10. |
| P0-09 | EIGRP MD5 key chain | ISR4331, ISR2911 | ✅ | Neighbor formed and `10.40.20.0/24` was learned; a wrong key dropped the neighbor and restoring it recovered it. |
| P0-10 | Redistribution with route-map tags | ISR4331, ISR2911 | ⚠️ | `route-map` is invalid on both (2911 re-tested with securityk9 active). Plain two-way redistribution works, with tag 0; keep one redistribution point and no tag-based loop prevention. |
| P0-11a | BGP on Router-PT | Router-PT | ✅ | Two eBGP neighbors established and `203.0.113.0/24` was learned. |
| P0-11b | iBGP between loopbacks (`update-source`, `next-hop-self`) | ISR4331, ISR2911 | ❌ | PT reports `does not support internal BGP ... Only external neighbors are supported`; the neighbor never appears in the running config. `update-source` → `% Invalid input`. |
| P0-11c | BGP outbound prefix-list | ISR4331, ISR2911 | ❌ | `ip prefix-list` and neighbor `prefix-list` → `% Invalid input`. |
| P0-11d | AS-path prepend via route-map | ISR4331, ISR2911 | ❌ | `route-map` and `set as-path prepend` → `% Invalid input`. |
| P0-11e | `default-originate` | Router-PT, ISR4331, ISR2911 | ❌ | Neighbor `default-originate` → `% Invalid input`. |
| P0-11f | BGP neighbor password | Router-PT, ISR4331, ISR2911 | ❌ | Neighbor `password` → `% Invalid input`. |
| P0-11g | Inbound prefix-list rejects partner-edge default over iBGP | ISR4331 | ❌ | Neither iBGP nor prefix-lists are available; the production BGP policy must be simplified for PT. |
| P0-12 | OSPF `default-information originate metric` | ISR4331, ISR2911 | ⚠️ | `metric`, `metric-type 1` and `always` are invalid. Plain `default-information originate` works and produces an E2 default with metric 1. |
| P0-13 | IP SLA + object tracking on static route | ISR4331, ISR2911 | ❌ | `ip sla`, `track ... ip sla` and `ip route ... track` → `% Invalid input` on both. |
| P0-14a | IKE `hash sha256` | ISR4331, ISR2911 | ⚠️ | `sha256` is invalid; `hash sha` works. |
| P0-14b | IKE `group 14` | ISR4331, ISR2911 | ⚠️ | Group 14 is invalid; group 5 works. |
| P0-14c | ESP `esp-sha256-hmac` | ISR4331, ISR2911 | ⚠️ | SHA-256 HMAC is invalid; `esp-sha-hmac` works with AES-256. |
| P0-14d | IPsec transport mode for GRE | ISR4331, ISR2911 | ⚠️ | `mode transport` is unavailable; default tunnel mode works. GRE ping succeeds (4/5 on the re-run, the first packet lost to ARP) and ISAKMP is `QM_IDLE`. |
| P0-15a | Named ACL for NAT with deny exemption | ISR4331 | ✅ | Named extended ACL and PAT worked; PC2 reached `3.3.3.3` through R1 (4/4 on the re-run). |
| P0-15b | ZBF (inspect HTTP/HTTPS/DNS/ICMP) | ISR4331 | ✅ | Zone pair and inspect policy persisted; ICMP counter increased by four packets. |
| P0-15c | ZBF + NAT + crypto map + GRE on one router | ISR4331 | ✅ | PC2 Internet ping, GRE/IPsec and eBGP all remained operational together. |
| P0-16a | TACACS+ with local fallback | ISR2911 | ✅ | `group tacacs+ local` falls back to the local user only when the server is **unreachable**. When Server-PT answers with a reject, login fails with `% Login invalid` and the local user is never tried. Add `aaa authentication login CONSOLE local` plus `line con 0` / `login authentication CONSOLE` so the console cannot be locked out. |
| P0-16b | `login block-for` | ISR2911 | ✅ | Persisted with the requested thresholds and blocks after three failures. |
| P0-16c | Authenticated NTP | ISR2911 | ✅ | Authentication key, trusted key and keyed server all persisted. |
| P0-16d | SNMP community with ACL | ISR4331, ISR2911, 2960, 3650 | ⚠️ | `snmp-server community <name> RO` works on every platform (`%SNMP-5-WARMSTART`). The ACL-bound form `... RO 99` → `% Invalid input`. Drop the ACL from the community and restrict management access with the VTY `access-class` and interface ACLs instead. |
| P0-17a | MQC marking (`set ip dscp`) | ISR2911 | ✅ | Input policy attached and reported `QoS Set dscp ef`. |
| P0-17b | MQC queuing (LLQ, CBWFQ, fair-queue) | ISR2911 | ✅ | Output policy contains priority, bandwidth and fair-queue classes. |
| P0-17c | MQC queuing | ISR4331 | ✅ | Output policy attached to Gi0/0/1 and reported LLQ/CBWFQ classes. |

## Checked during the build

Commands the configs rely on that aren't covered by Phase 0. Each config has the fallback next to the line; record what happened here. Rows marked with a result were syntax-probed in the Phase 0 re-run (line present in `show running-config` afterwards).

| ID | Feature | Device(s) | Result | Notes |
|---|---|---|---|---|
| B-01 | `ip tacacs source-interface Loopback0` | ISR4331, ISR2911 | ❌ | `% Invalid input` on both. Register the outgoing interface IP as the AAA client. |
| B-02 | `redistribute ospf ... match internal external 1 external 2` into EIGRP | ISR4331, ISR2911 | ✅ | Accepted and persisted on both. |
| B-03 | `eigrp stub connected` | ISR2911, 3650 | ❌ | `eigrp stub` and `eigrp stub connected` → `% Invalid input`. The Toronto switches stay normal EIGRP neighbors, limited by `passive-interface default`. |
| B-04 | `ip tcp adjust-mss` on tunnels | ISR4331, ISR2911 | ❌ | `% Invalid input` on tunnel and physical interfaces. Omit it; PT does not fragment-test. |
| B-05 | `aaa authentication ppp default local` | ISR4331, ISR2911 | ✅ | Accepted and persisted on both. |
| B-06 | ZBF `pass` with ACL class-map including `established` | ISR4331 | ✅ | `permit tcp any any established` in a named ACL, `match access-group name`, and `pass` in an inspect policy all persisted. Traffic behaviour is checked in the DMZ phase. |
| B-07 | ASA twice NAT (`nat (inside,outside) source static ... destination static ...`) | ASA 5506-X | ⏳ | VPN NAT exemption |
| B-08 | ASA `crypto ikev1 policy` with `group 14` | ASA 5506-X | ⏳ | Fallback group 5 on both ends (the IOS peers only support group 5; P0-14b). |
| B-09 | ASA `inspect icmp` in global_policy | ASA 5506-X | ⏳ | |
| B-10 | ASA `management-access inside`, `logging host`, `ntp server` | ASA 5506-X | ⏳ | |
| B-11 | WLC management interface tagged VLAN 10 | WLC-3504 | ⏳ | Fallback: native VLAN 10 on the WLC trunk |
| B-12 | WPA2-Enterprise on Laptop-PT / Smartphone-PT | Wireless clients | ⏳ | |
| B-13 | HSRP track on ISR4331 (`track 2 interface ... line-protocol`) | ISR4331 | ❌ | `track` objects and `standby N track` are rejected (see P0-07b). Failover relies on HSRP hello loss only. |
| B-14 | WPA2-Enterprise (RADIUS) on AccessPoint-PT | Branch APs | ⏳ | Fallback: WPA2-PSK `Lab-Corp-Wifi-1` on branch corporate SSID |
| B-15 | IOS DHCP server on 3650 (split scopes) | Branch switch pairs | ✅ | SW1's `ip dhcp pool` served PC1 in Phase 0 (`10.10.20.100`). Split scopes are the same feature with different ranges. |
| B-16 | `ip summary-address eigrp` on serial | ISR2911 | ✅ | Accepted and persisted (tested on Ethernet; the command is interface-type independent). |
| B-17 | EIGRP stub + `passive-interface default` on 3650 | TOR-SW-01/02 | ⚠️ | `passive-interface default` works; `eigrp stub` is rejected (B-03). |
| B-18 | ASA second inside interface + floating static route | CHI-FW-01 | ⏳ | Fallback: single inside link to CHI-SW-01 (Chicago loses switch-level WAN redundancy) |
| B-19 | ASA crypto map with two peers | CHI-FW-01 | ⏳ | Fallback: single peer 198.51.100.6 |
| B-20 | PoE from 3650-24PS to IP phones and AccessPoint-PT | Branch switches | ⏳ | Fallback: power adapters |
| B-21 | DHCP snooping and DAI on 3650 | Branch switches | ⚠️ | All snooping lines and `ip arp inspection vlan` are accepted. DAI is expected to behave as on the 2960 (P0-05a): active but not dropping. |
| B-22 | Static route with `track` | LON-EDGE-02 | ❌ | Covered by P0-13; use the documented untracked-route fallback. |
| B-23 | MCP device/interface names match the GUI and cabling plan | Every model | ✅ | Phase 2 preflight read every model's live ports: the GUI names in the cabling plan are right, and the MCP's static catalog is wrong for the 3650, ASA 5506-X and Router-PT (mapping and module slots in [cabling-plan.md](cabling-plan.md#mcp-preflight-results-2026-09-29)). All 83 links were created with the live names. |
| B-24 | 2960-24TT on IOS 15 | Every 2960 | ✅ | A new 2960-24TT boots `c2960-lanbase-mz.122-25.FX.bin`, which lacks DAI and 802.1X. Server-PT's TFTP service has `c2960-lanbasek9-mz.150-2.SE4.bin`: `copy tftp: flash:` → `boot system flash:c2960-lanbasek9-mz.150-2.SE4.bin` → `write memory` → `reload`. `show version` then reports 15.0(2)SE4 and the running config survives. |
| B-25 | Equal E2 defaults resolved by distance to the ASBR (`ip ospf cost 1000` on EDGE-02's core link) | LON core | ⏳ | `ip ospf cost` is accepted. The tie-break itself needs three routers in OSPF and is checked in Phase 5 (test T04, "Also check"). If PT load-shares the two defaults instead, shut EDGE-02's `default-information originate` and turn it on by hand during T04. |
| B-26 | `switchport trunk encapsulation dot1q` | 3650 | ❌ | `% Invalid input`: the 3650 is dot1q-only. Removed from the configs. |
| B-27 | Plain `redistribute static subnets` into OSPF | ISR4331 | ✅ | Accepted; statics appear as O E2 metric 20. It also exports Null0 statics (seen in `show ip ospf database external`). `redistribute static ... metric N` and `distribute-list N out static` are both `% Invalid input`, so the edges can't filter it (ADR 0001). |
| B-28 | ISP default via `ip route 0.0.0.0 0.0.0.0 Null0` + `network 0.0.0.0 mask 0.0.0.0` | Router-PT | ✅ | Replaces `neighbor ... default-originate` (P0-11e). R1 learned `0.0.0.0/0` from R3 in its BGP table. Advertising a /25 with `network ... mask 255.255.255.128` and a Null0 anchor also works on the ISR4331. |
| B-29 | ASA factory DHCP pool (`no dhcpd enable inside`) | ASA 5506-X | ⚠️ | PT's 5506-X starts with no DHCP pool (`show running-config dhcpd` is empty) and rejects the `no dhcpd ...` lines. Nothing to remove; the lines were dropped from CHI-FW-01.cfg. |
| B-30 | `securityk9` on the ISR4331 | ISR4331 | ✅ | A new ISR4331 in PT 9 already shows `securityk9  Permanent` as the current technology package; `license boot level securityk9` is accepted but no reload is needed. |
| B-31 | Router-PT expansion slots 4 and 5 | Router-PT | ⚠️ | They hold fiber FastEthernet (PT-ROUTER-NM-1FFE), which refuses copper cables. Swapped for PT-ROUTER-NM-1CFE (copper) on ISP-A, ISP-B and INET-CORE; the port names FastEthernet4/0 and 5/0 are unchanged. |
| B-32 | Wireless clients before Phase 8 | Laptop-PT, TabletPC-PT, SMARTPHONE-PT | ⚠️ | With every AP on its factory open SSID, clients associate with whichever AP answers, including other sites' APs. Their `Wireless0` radios are powered off from Phase 2 until the SSIDs are configured in Phase 8. |
| B-33 | LACP Po1 between two 3650s | 3650-24PS | ⚠️ | After pasting the same `channel-group 1 mode active` config on both cores, Po1 stayed `(SD)` with both members `(s)` and no LACP neighbour; bouncing the members didn't help. Removing and re-adding the channel-group on both switches (`shutdown`, `no channel-group 1`, `channel-group 1 mode active`, `no shutdown`) brought Po1 up as `(SU)` with both members `(P)`. |
| B-34 | `ip ospf network point-to-point` on an SVI | 3650-24PS | ❌ | `% Invalid input` on `interface Vlan900` (routed ports accept it). The VLAN 900 adjacency between the cores runs as the broadcast type (`FULL/DR`, `FULL/BDR`); both ends match, so it forms normally. Removed from the LON-CORE and IST-SW Vlan900 configs. |

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

Items outside what PT can show, listed so the gaps are explicit: SNMPv3, CoPP, UDLD, loop guard, automatic err-disable recovery, HSRP authentication and object tracking, IP SLA-tracked routes, iBGP and BGP route policy (prefix-lists, route-maps, AS-path prepend, MD5 session passwords), SHA-256/DH group 14+ IPsec, IKEv2 with VTI or DMVPN, DAI with ARP ACLs that actually drop, MAB for phones and printers alongside 802.1X, NetFlow to a collector, WLC HA pair, FlexConnect or cloud-managed branch APs, 48-port PoE access switches, SRST at the branches, a multi-chassis/stateful core system with redundant control planes, a second VPN peer for Chicago and Toronto, a second circuit (or LTE backup) for Toronto, and SD-WAN in place of hand-built tunnels.
