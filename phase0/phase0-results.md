# Phase 0 results

**Run:** 2026-09-29, re-run the same day after a review of the first results  
**Packet Tracer:** 9.0.1.0858  
**Gate:** complete

## Artifacts

- [`phase0-scratch.pkt`](phase0-scratch.pkt) — saved scratch topology with its result labels (re-run state).
- [`evidence/phase0-final-labeled.png`](evidence/phase0-final-labeled.png) — final labeled topology and summary.
- [`evidence/phase0-baseline.png`](evidence/phase0-baseline.png) — baseline topology before the test sequence.
- [`../docs/pt-support-matrix.md`](../docs/pt-support-matrix.md) — authoritative result and fallback for every test.

## Outcome

| Result | First run | Re-run | Meaning |
|---|---:|---:|---|
| ✅ | 16 | 16 | Feature worked as planned. |
| ⚠️ | 8 | 11 | A reduced Packet Tracer syntax or behavior works. |
| ❌ | 15 | 12 | Feature is unavailable in this Packet Tracer model/IOS. |
| **Total** | **39** | **39** | All planned Phase 0 checks completed. |

The base topology contains the nine intended lab devices plus Packet Tracer's automatically created power-distribution object. All nine links are up. After the re-run: OSPF R1–R2 FULL, EIGRP R2–R4 up, both eBGP sessions to R3 established, ISAKMP `QM_IDLE`, HSRP active on SW1. End-to-end pings: PC2 → 3.3.3.3 through NAT/ZBF 4/4, R1 → R2 over GRE/IPsec 4/5 (first packet lost to ARP), PC1 → PC2 across the SW1–SW2 trunk 4/4.

## What the re-run changed

The first run had three test-setup faults, not Packet Tracer limits:

| Fault | Effect | Fix |
|---|---|---|
| SW2 booted the default 2960 image, 12.2(25)FX LANBASE | DAI and every 802.1X command were missing | Copied `c2960-lanbasek9-mz.150-2.SE4.bin` from SRV1's TFTP service, set `boot system`, reloaded (matrix B-24) |
| The 2911 licence command asks `ACCEPT? [yes/no]`, which the script never answered | securityk9 stayed disabled | Answered `yes`, saved, reloaded; `show license feature` shows securityk9 enabled |
| SW1 ran PVST, SW2 Rapid PVST, and neither was set as root | Both claimed root for VLAN 20; SW1's uplink sat in `LIS` | Rapid PVST on both, SW1 priority 4096 for VLANs 1 and 20 |

Result changes: P0-01 ⚠️ → ✅; P0-05a, P0-06, P0-16d ❌ → ⚠️; P0-02a ✅ → ⚠️ (root guard blocks in STP only, not on the data plane). Every other ❌ was re-tested on IOS 15 (switches) and on the licensed 2911 (routers) and stayed ❌. The re-run also syntax-checked most of the "checked during the build" rows (B-01 to B-06, B-13, B-15 to B-17, B-21).

## Proven fallbacks for the main build

- **2960 access switches:** upgrade to IOS 15.0(2)SE4 before configuring them (B-24). The default image rejects DAI and 802.1X.
- **HSRP:** v2 with priority and preempt; no authentication and no interface/object tracking on either the 3650 or the ISR4331.
- **Layer 2 security:** keep root guard, BPDU guard, port security and DHCP snooping. Keep DAI and the 802.1X switch config as documented controls, knowing PT does not drop on DAI and PC-PT does not complete 802.1X. Omit loop guard, automatic err-disable recovery and ARP ACLs. After a root-guard event, bounce the port.
- **Redistribution:** one untagged OSPF/EIGRP redistribution point; route maps and tags are unavailable. `match internal external 1 external 2` works.
- **BGP:** eBGP only. No iBGP, prefix lists, route maps, AS-path prepending, passwords or `default-originate`. ISPs send the default with a Null0 static plus `network 0.0.0.0` (B-28); inbound preference comes from EDGE-01 announcing two /25s against EDGE-02's /24 ([ADR 0001](../docs/adr/0001-active-standby-internet-edge.md)).
- **Default route / tracking:** originate the OSPF default without a metric; EDGE-02's core link costs 1000 so EDGE-01's equal E2 default wins (B-25, verified in Phase 5). No IP SLA, `track` objects or tracked static routes. Plain `redistribute static subnets` works but can't be filtered (B-27).
- **3650:** no `switchport trunk encapsulation dot1q` (B-26).
- **GRE/IPsec:** IKE `hash sha`, `group 5`, AES-256; `esp-aes 256 esp-sha-hmac`; default tunnel mode. No `ip tcp adjust-mss`.
- **NAT/ZBF:** named NAT ACLs, PAT and ZBF work together with GRE/IPsec and eBGP; ZBF `pass` with an `established` ACL is accepted.
- **Management:** TACACS+ with local fallback, `login block-for`, authenticated NTP and MQC QoS work. Add a console-only local login list (`aaa authentication login CONSOLE local`), because a TACACS+ reject blocks the local fallback. SNMP communities work only without an ACL (`snmp-server community <name> RO`). `ip tacacs source-interface` and `eigrp stub` are unavailable.

## Config status

Every device config in [`configs/`](../configs/) now uses only commands Phase 0 accepted, with a `P0-xx` / `B-xx` comment where a fallback replaced the planned line. The Internet edge, both ISPs, LON-WAN-01's redistribution and the Chicago routing were redesigned around the missing BGP and route-map features ([ADR 0001](../docs/adr/0001-active-standby-internet-edge.md)); the design plan, test plan and ADRs were updated to match. The ASA lines (B-07 to B-10, B-18, B-19) are still untested until Phase 6.

## Automation notes

The optional MCP bridge placed devices, created links, configured the lab, queried state and saved the `.pkt`. Its static 3650 interface catalog did not match the running device: the working ports were `Gi1/0/1`–`Gi1/0/24`. Live interface discovery and raw link commands were used instead. ASA and modular-router mappings remain a Phase 2 preflight item.

Script-driven CLI pitfalls found in this phase:

- Send an explicit `exit` between configuration submodes; otherwise later commands can land in the previous submode.
- Check the prompt before every command. A `Username:` / `Password:` prompt swallows configuration lines silently, and an unanswered `ACCEPT?` or `[confirm]` prompt eats the next line. Several first-run results were artefacts of this.
- Count a command as accepted only when it appears in `show running-config` afterwards, not merely because no `% Invalid input` was printed.
