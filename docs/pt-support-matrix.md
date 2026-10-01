# Packet Tracer support matrix

What Packet Tracer 9 supports for this design, verified hands-on rather than assumed. Tests are defined in [`phase0/phase0-feature-tests.md`](../phase0/phase0-feature-tests.md).

**Packet Tracer build tested:** 9.0.1.0858
**Images tested:** ISR4331 IOS XE (default image), ISR2911 IOS 15.1(4)M4 with `securityk9`, 3650-24PS (default image), 2960-24TT **IOS 15.0(2)SE4** (upgraded from the default 12.2(25)FX; see B-24), Router-PT (default image)
**Legend:** ✅ works as designed · ⚠️ partial or fallback syntax only · ❌ not supported

**What mattered most for this design:**

- **Edge failover:** PT load-shares two equal OSPF defaults and has no iBGP, route-maps or prefix-lists, so only EDGE-01 originates the default and the cores carry a floating static towards EDGE-02 (B-25). After ISP-A recovers, the default has to be re-originated by hand (B-37).
- **No HSRP tracking or authentication** (P0-07a/b): the DMZ gateway is moved by hand during an ISP failure.
- **Firewall:** the zone-based firewall can't pass GRE tunnels, so all four routers use CBAC plus ACLs (B-50, ADR 0015). PT's ASA can't complete IPsec with IOS, so Chicago uses a router (B-44, ADR 0007).
- **Layer 2 security is partly cosmetic:** DAI and root guard are accepted but not enforced on the data plane (P0-05a, P0-02a); loop guard and errdisable recovery don't exist (P0-02b/c, P0-03).
- **Wireless:** branch APs have no RADIUS (B-14), and the WLC uplink can't trunk, so London Wi-Fi clients share the management VLAN (B-53).
- **Stability:** PT 9.0.1 crashed repeatedly on this lab (B-49); save often.

A result counts only when the line is in `show running-config` afterwards. Script-driven runs can silently lose lines to a login prompt, an interactive prompt (the licence EULA) or the wrong submode; several first-run results were artefacts of exactly that and were corrected in the 2026-09-29 re-run.

## Phase 0 results

| ID | Feature | Device(s) | Result | Notes (rejected command / fallback used) |
|---|---|---|---|---|
| P0-01 | `securityk9` license | ISR4331, ISR2911 | ✅ | Active on both ISR4331s and on the ISR2911. The 2911 prompts `ACCEPT? [yes/no]` for the EULA; the first run never answered it, so the licence was not set. Answer `yes`, `write memory`, reload; `show license feature` then shows securityk9 enabled. |
| P0-02a | Root guard | 3650 | ⚠️ | Accepted (`spanning-tree guard root`) and logs `%SPANTREE-2-ROOTGUARDBLOCK` with the port `BLK`, but traffic still crosses the port: PT enforces it in STP state only. The block doesn't clear by itself; bounce the port. Set STP mode and root priority explicitly on both switches. |
| P0-02b | Loop guard (interface) | 2960 (IOS 15), 3650 | ❌ | `spanning-tree guard loop` → `% Invalid input` on both. |
| P0-02c | Loop guard (global default) | 2960 (IOS 15), 3650 | ❌ | `spanning-tree loopguard default` → `% Invalid input` on both. |
| P0-03 | Err-disable recovery for bpduguard | 2960 (IOS 15), 3650 | ❌ | `errdisable recovery cause ...` and `errdisable recovery interval` → `% Invalid input` on both. Recover ports with `shutdown` / `no shutdown`. BPDU guard itself (`spanning-tree bpduguard enable`) is accepted. |
| P0-04 | DHCP snooping + `no ip dhcp snooping information option` | 2960, 3650 | ✅ | PC1 received `10.10.20.100` and appears in the snooping binding table (re-verified on IOS 15). All snooping lines are also accepted on the 3650. |
| P0-05a | Dynamic ARP Inspection | 2960 (IOS 15) | ⚠️ | IOS 15 accepts `ip arp inspection vlan`, `validate src-mac dst-mac ip` and `trust`; the VLAN shows `Active`, but an unbound host and a spoofed IP were both forwarded. Kept as a visible control, not an enforced one. |
| P0-05b | DAI ARP ACL filter for static hosts | 2960 (IOS 15) | ❌ | `arp access-list` and `ip arp inspection filter` → `% Invalid input`. |
| P0-06 | Wired 802.1X with RADIUS | 2960 (IOS 15) + Server-PT | ⚠️ | IOS 15 accepts the switch side (`aaa new-model`, `dot1x system-auth-control`, `authentication port-control auto`, `dot1x pae authenticator`) and blocks an unauthenticated port, but PC-PT never sends EAPOL, so no client can log in. `dot1x port-control auto` and the `radius server` block form are rejected; 12.2 rejects all dot1x. Not used in the lab. |
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
| B-06 | ZBF `pass` with ACL class-map including `established` | ISR4331 | ✅ | `permit tcp any any established` in a named ACL, `match access-group name`, and `pass` in an inspect policy all persisted. Traffic behaviour is checked in the DMZ phase. No longer used: the ZBF was replaced by CBAC in Phase 6 (B-50). |
| B-07 | ASA twice NAT (`nat (inside,outside) source static ... destination static ...`) | ASA 5506-X | ❌ | `% Invalid input`: the PT ASA has object NAT only, so no NAT exemption for the VPN. Moot since Phase 6: the ASA was replaced ([ADR 0007](adr/0007-chicago-router-not-asa.md)). |
| B-08 | ASA `crypto ikev1 policy` with `group 14` | ASA 5506-X | ❌ | Only groups 1, 2 and 5 exist. `encryption aes-256` is accepted but not stored (the running config shows no `encr` line); only `encr aes` sticks. ASA replaced in Phase 6. |
| B-09 | ASA `inspect icmp` in global_policy | ASA 5506-X | ✅ | Accepted and shown under `class inspection_default` (Phase 6, before the ASA was replaced). |
| B-10 | ASA `management-access inside`, `logging host`, `ntp server` | ASA 5506-X | — | Not tested: the ASA was replaced in Phase 6 before its management plane was built. |
| B-11 | WLC management interface tagged VLAN 10 | WLC-3504 | ❌ | PT treats the WLC's uplink as an access port, so it can't tag. Setting the management VLAN to 10 left the WLC unreachable. Fallback used: management VLAN `0` (untagged) and native VLAN 10 on LON-CORE-01 Gi1/0/10. Consequence for the dynamic interfaces: B-53. |
| B-12 | WPA2-Enterprise on Laptop-PT / Smartphone-PT | Wireless clients | ✅ | Through LON-WLC-01 and RADIUS on LON-SRV-AAA-01: in PT the client's **WPA2** option (with User ID / Password) is Enterprise. Needed the AAA Radius Port changed from the default 1645 to 1812; before that the clients associated and fell back to 169.254.x.x. |
| B-13 | HSRP track on ISR4331 (`track 2 interface ... line-protocol`) | ISR4331 | ❌ | `track` objects and `standby N track` are rejected (see P0-07b). Failover relies on HSRP hello loss only. |
| B-14 | WPA2-Enterprise (RADIUS) on AccessPoint-PT | Branch APs | ❌ | AccessPoint-PT Port 1 offers only Disabled, WEP, WPA-PSK and WPA2-PSK: no RADIUS server field. Fallback used: branch `BSP-CORP` is WPA2-PSK (AES) with key `Lab-Corp-Wifi-1`; London's `BSP-CORP` on the WLC stays WPA2-Enterprise. |
| B-15 | IOS DHCP server on 3650 (split scopes) | Branch switch pairs | ✅ | SW1's `ip dhcp pool` served PC1 in Phase 0 (`10.10.20.100`). Split scopes are the same feature with different ranges. |
| B-16 | `ip summary-address eigrp` on serial | ISR2911 | ✅ | Accepted and persisted (tested on Ethernet; the command is interface-type independent). |
| B-17 | EIGRP stub + `passive-interface default` on 3650 | TOR-SW-01/02 | ⚠️ | `passive-interface default` works; `eigrp stub` is rejected (B-03). |
| B-18 | ASA second inside interface + floating static route | CHI-FW-01 | ❌ | Two routes to 10.30.0.0/16 via different inside interfaces collapse into one (shown as `inside2` even for a next hop on `inside`), and two network objects with the same subnet share one NAT rule. ASA replaced in Phase 6. |
| B-19 | ASA crypto map with two peers | CHI-FW-01 | ❌ | Accepted, but the ASA alternated between peers and the peer list could not be edited cleanly; even with one peer, Phase 2 never completed (B-44). ASA replaced in Phase 6. |
| B-20 | PoE from 3650-24PS to IP phones and AccessPoint-PT | Branch switches | ✅ | Branch phones and APs were cabled to the 3650s without power adapters in Phase 6; all three branch phones register with CME and all six branch APs serve clients (Phase 8). |
| B-21 | DHCP snooping and DAI on 3650 | Branch switches | ⚠️ | All snooping lines and `ip arp inspection vlan` are accepted. DAI is expected to behave as on the 2960 (P0-05a): active but not dropping. |
| B-22 | Static route with `track` | LON-EDGE-02 | ❌ | Covered by P0-13; use the documented untracked-route fallback. |
| B-23 | MCP device/interface names match the GUI and cabling plan | Every model | ✅ | Every model's live ports were read in Phase 2: the GUI names in the cabling plan are right, the MCP's static catalog is wrong for the 3650 and Router-PT ([cabling-plan appendix](cabling-plan.md#appendix-port-names-for-mcp-automation-checked-2026-09-29)). |
| B-24 | 2960-24TT on IOS 15 | Every 2960 | ✅ | A new 2960-24TT boots `c2960-lanbase-mz.122-25.FX.bin`, which lacks DAI and 802.1X. Server-PT's TFTP service has `c2960-lanbasek9-mz.150-2.SE4.bin`: `copy tftp: flash:` → `boot system flash:c2960-lanbasek9-mz.150-2.SE4.bin` → `write memory` → `reload`. `show version` then reports 15.0(2)SE4 and the running config survives. |
| B-25 | Equal E2 defaults resolved by distance to the ASBR (`ip ospf cost 1000` on EDGE-02's core link) | LON core | ❌ | With both edges originating, the cores load-share the two E2 defaults (PT ignores the metric), which breaks NAT: traffic leaving as EDGE-02's PAT address returns via EDGE-01. `default-information originate` takes no options and `distance bgp` is rejected. Used instead: only EDGE-01 originates; the cores carry a floating static (AD 250): CORE-02 to EDGE-02 (10.255.0.5), CORE-01 via CORE-02 (10.255.0.10). Phase 9 (T04) found CORE-01's static pointing at LON-WAN-01, a black hole; corrected. |
| B-26 | `switchport trunk encapsulation dot1q` | 3650 | ❌ | `% Invalid input`: the 3650 is dot1q-only. Removed from the configs. |
| B-27 | Plain `redistribute static subnets` into OSPF | ISR4331 | ✅ | Accepted (statics appear as O E2), but it also exports Null0 statics and can't be filtered (`metric` and `distribute-list ... out static` rejected). No longer used: only the ASA-based Chicago design needed it. |
| B-28 | ISP default via `ip route 0.0.0.0 0.0.0.0 Null0` + `network 0.0.0.0 mask 0.0.0.0` | Router-PT | ✅ | Replaces `neighbor ... default-originate` (P0-11e). R1 learned `0.0.0.0/0` from R3 in its BGP table. Advertising a /25 with `network ... mask 255.255.255.128` and a Null0 anchor also works on the ISR4331. |
| B-29 | ASA factory DHCP pool (`no dhcpd enable inside`) | ASA 5506-X | ⚠️ | PT's 5506-X starts with no DHCP pool (`show running-config dhcpd` is empty) and rejects the `no dhcpd ...` lines. Moot since the ASA was replaced in Phase 6. |
| B-30 | `securityk9` on the ISR4331 | ISR4331 | ✅ | A new ISR4331 in PT 9 already shows `securityk9  Permanent` as the current technology package; `license boot level securityk9` is accepted but no reload is needed. |
| B-31 | Router-PT expansion slots 4 and 5 | Router-PT | ⚠️ | They hold fiber FastEthernet (PT-ROUTER-NM-1FFE), which refuses copper cables. Swapped for PT-ROUTER-NM-1CFE (copper) on ISP-A, ISP-B and INET-CORE; the port names FastEthernet4/0 and 5/0 are unchanged. |
| B-32 | Wireless clients before Phase 8 | Laptop-PT, TabletPC-PT, SMARTPHONE-PT | ⚠️ | With every AP on its factory open SSID, clients associate with whichever AP answers, including other sites' APs. Their `Wireless0` radios are powered off from Phase 2 until the SSIDs are configured in Phase 8. |
| B-33 | LACP Po1 between two 3650s | 3650-24PS | ⚠️ | Po1 stays `(SD)` after the first paste. Re-entering `channel-group 1 mode active` on both switches (shut, remove, re-add, no shut) brings it to `(SU)`; it then survives save and reopen. Needed on the London cores and all three branch pairs. |
| B-34 | `ip ospf network point-to-point` on an SVI | 3650-24PS | ❌ | `% Invalid input` on `interface Vlan900` (routed ports accept it). The VLAN 900 adjacency between the cores runs as the broadcast type (`FULL/DR`, `FULL/BDR`); both ends match, so it forms normally. Removed from the LON-CORE and IST-SW Vlan900 configs. |
| B-35 | `neighbor X description` | Router-PT, ISR4331 | ❌ | `% Invalid input` in `router bgp`. Removed from ISP-A, ISP-B, INET-CORE and LON-EDGE-01/02. |
| B-36 | `description` under `zone security` | ISR4331 | ❌ | `% Invalid input`; the zones themselves work. Removed from LON-EDGE-01/02. |
| B-37 | `default-information originate` follows the default route | ISR4331 | ⚠️ | Withdrawal works (the E2 LSA is flushed when the BGP default disappears), but origination doesn't restart by itself when the default returns or after a reload: re-enter `no default-information originate` / `default-information originate`. |
| B-38 | Routes with the same address and different masks | Router-PT, ISR4331, 3650 | ⚠️ | BGP keeps only one of 203.0.113.0/24 and /25, whichever arrived first. Harmless; after a failover, `clear ip bgp *` on LON-EDGE-02 brings EDGE-01's /25 back. OSPF: see B-47. |
| B-39 | BGP session after the peer's link flaps | ISR4331, Router-PT | ⚠️ | The session stays in a stale "established" state with no routes until the hold timer expires in simulated time. Press **Fast Forward Time** (realtime toolbar) a few times; `clear ip bgp *` alone doesn't help. |
| B-40 | Tunnel interface commands | ISR4331 | ⚠️ | Tunnel interfaces accept only `ip address`, `tunnel ...` and `shutdown`; `description`, `ip mtu`, `ip ospf ...` and `zone-member security` are rejected. OSPF still forms over them (point-to-point); EDGE-01 stays primary because EDGE-02's core link costs 1000. |
| B-41 | `no passive-interface Tunnel0` | ISR4331 | ❌ | Tunnels are not offered in `passive-interface`; the line is rejected, or accepted at runtime and then lost on reload, so `passive-interface default` silences the tunnels. Routers with tunnels list their passive interfaces instead. |
| B-42 | `ip dhcp snooping trust` / `ip arp inspection trust` on Port-channel1 | 3650-24PS | ❌ | `% Invalid input` on the port-channel; accepted on the member ports, where they now sit. |
| B-43 | EIGRP MD5 on an SVI (`ip authentication mode eigrp 100 md5` on Vlan900) | 3650-24PS | ❌ | `% Invalid input` on the SVI; accepted on routed ports. The Toronto switch-to-switch adjacency on VLAN 900 is unauthenticated. |
| B-44 | ASA ↔ IOS policy-based IPsec | ASA 5506-X, ISR4331 | ❌ | IKE phase 1 comes up, but quick mode fails with "The responder does not have a policy matching initiator's policies" for every transform, peer and proxy combination tried. Chicago's ASA was replaced by an ISR4331 with GRE over IPsec ([ADR 0007](adr/0007-chicago-router-not-asa.md)). |
| B-45 | ASA operations | ASA 5506-X | ⚠️ | No `clear crypto ikev1 sa` / `clear crypto ipsec sa`; IKE is reset by unbinding and rebinding the crypto map on the interface. The console times out to `>` after Fast Forward Time. |
| B-46 | Removing redistribution or a summary | ISR4331 | ⚠️ | After `no redistribute static`, `no default-information originate` or a readdressing, the old LSAs stay in the database (even after `clear ip ospf process`) until the lab is saved and reopened. |
| B-47 | OSPF `area range` whose address is also a component subnet | ISR4331, 3650 | ⚠️ | With `area 2 range 10.30.0.0 255.255.0.0` and a 10.30.0.0/30 link inside the area, the ABRs advertised both 10.30.0.0/16 and 10.30.0.0/30, and London kept only the /30, losing every Chicago VLAN. The branch router ↔ SW-01 links were moved to 10.x.0.12/30 so no component starts at the summary address. |
| B-48 | Two branch `area range` statements on the same ABRs | ISR4331 | ❌ | With `area 1 range` and `area 2 range` on both edges, each edge generates only one of the two summaries, and which one changes between loads. Branch ranges removed: London learns branch subnets from both edges and prefers EDGE-01 by cost. `area 0 range 10.10.0.0/16` towards the branches works. |
| B-49 | Packet Tracer stability with the full lab | PT 9.0.1 (macOS) | ⚠️ | PT 9.0.1 segfaulted repeatedly inside its own engine once all sites were up: OSPF route install, ARP clears after DHCP and HSRP timers, ZBF session expiry, EIGRP MD5 hashing during failover tests, and opening a file through the MCP bridge. On macOS it also crashed when other apps queried its window through the accessibility API (clipboard managers, window switchers). Mitigation: save after every step, one change at a time, no Fast Forward bursts, close such apps. |
| B-50 | Zone-based firewall with GRE tunnels | ISR4331 | ❌ | Tunnels can't be zone members (B-40), and ZBF drops traffic between a zone and a non-zone interface, so all branch ↔ London traffic was dropped. Replaced on all four routers by CBAC (`ip inspect`) plus ACLs; in ACLs `eq bgp` is rejected (use `eq 179`) ([ADR 0015](adr/0015-cbac-not-zbf.md)). |
| B-51 | Management-plane hardening lines | ISR4331, 3650, 2960, 2811 | ⚠️ | `no ip http server` and `logging trap informational` are rejected on every model, and the 2960s reject `login block-for`. Everything else in Phase 7 is accepted and works (SSH v2, VTY ACL, TACACS+ with local fallback, NTP MD5, syslog, SNMP). |
| B-52 | 7960 lobby phone on a port whose access VLAN is also the voice VLAN | 2960 (IOS 15) + 7960 | ⚠️ | LON-PH-03, on a port where access and voice VLAN are both 50, never completes DHCP: with `voice vlan 50` it learns the VLAN over CDP and the server sends an Offer, but the lease never finishes. Moving the data VLAN to 998 makes DHCP snooping drop the Discover (no trusted port carries 998). Desk phones (data VLAN ≠ voice VLAN) work. Open. |
| B-53 | WLC dynamic interfaces (VLAN 60/70) for London Wi-Fi clients | WLC-3504 + 3702i LAPs | ❌ | PT treats the WLC uplink as an access port, so its dynamic interfaces (VLANs 60/70) never reach the wire and every London Wi-Fi client gets a VLAN 10 address. Tagging management VLAN 10 and forcing the port to trunk from a script made the WLC reachable but not the dynamic interfaces, and sent London STP into re-convergence; reverted to native VLAN 10. London Wi-Fi works, without corp/guest separation; branch Wi-Fi is separated. |
| B-54 | IPsec to the secondary London edge after reopening the .pkt | ISR4331 (IST-RTR-01 ↔ LON-EDGE-02) | ⚠️ | After reopening the `.pkt`, the IST ↔ EDGE-02 IKE session can sit in `MM_NO_STATE` with nothing encrypted, leaving Istanbul without its backup tunnel; the configs match and `clear crypto` does nothing in PT. Fix: `no crypto map CM-WAN` / `crypto map CM-WAN` on IST-RTR-01's and EDGE-02's outside interfaces. Check `show crypto isakmp sa` on both edges after every reopen. |
| B-55 | DHCP snooping rate limit after a switch reboot | 3650 (branch switch pairs) | ⚠️ | After a switch reboot, a PC and phone on the same port sent DHCP on two VLANs at once and PT's built-in snooping rate limit err-disabled the port (`dhcp-rate-limit error`); a PC on 169.254.x.x re-tripped it after every bounce. `ip dhcp snooping limit rate 100` fixes it; now on every untrusted access port of the branch switches. |

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

Items outside what PT can show, listed so the gaps are explicit: SNMPv3, CoPP, UDLD, loop guard, automatic err-disable recovery, HSRP authentication and object tracking, IP SLA-tracked routes, iBGP and BGP route policy (prefix-lists, route-maps, AS-path prepend, MD5 session passwords), SHA-256/DH group 14+ IPsec, IKEv2 with VTI or DMVPN, DAI with ARP ACLs that actually drop, MAB for phones and printers alongside 802.1X, NetFlow to a collector, WLC HA pair, FlexConnect or cloud-managed branch APs, 48-port PoE access switches, SRST at the branches, a multi-chassis/stateful core system with redundant control planes, an internet or LTE backup path for Toronto, corporate and guest Wi-Fi VLAN separation on the London WLC (a trunked WLC uplink), and SD-WAN in place of hand-built tunnels.
