# Phase 0 results

**Run:** 2026-09-29  
**Packet Tracer:** 9.0.1.0858  
**Gate:** complete

## Artifacts

- [`phase0-scratch.pkt`](phase0-scratch.pkt) — saved scratch topology with its result labels.
- [`evidence/phase0-final-labeled.png`](evidence/phase0-final-labeled.png) — final labeled topology and summary.
- [`evidence/phase0-baseline.png`](evidence/phase0-baseline.png) — baseline topology before the test sequence.
- [`../docs/pt-support-matrix.md`](../docs/pt-support-matrix.md) — authoritative result and fallback for every test.

## Outcome

| Result | Count | Meaning |
|---|---:|---|
| ✅ | 16 | Feature worked as planned. |
| ⚠️ | 8 | A reduced Packet Tracer syntax or behavior works. |
| ❌ | 15 | Feature is unavailable in this Packet Tracer model/IOS. |
| **Total** | **39** | All planned Phase 0 checks completed. |

The base topology contained the nine intended lab devices plus Packet Tracer's automatically created power-distribution object. All nine intended links came up. Baseline addressing was verified, and the final end-to-end PC2-to-SRV1 ping returned 4/4.

## Proven fallbacks for the main build

- Use HSRP v2 with priority and preempt, but without MD5 or interface/object tracking on 3650 switches.
- Keep root guard, BPDU guard, port security and DHCP snooping; omit loop guard, automatic err-disable recovery, DAI and wired 802.1X where Phase 0 rejected them.
- Use one untagged OSPF/EIGRP redistribution point. Route maps and route tags are unavailable.
- Use eBGP only. Packet Tracer rejected iBGP, prefix lists, AS-path prepending, BGP passwords and neighbor `default-originate`.
- Originate the OSPF default without an explicit metric. IP SLA and tracked static routes are unavailable.
- For GRE/IPsec use IKE SHA, DH group 5, AES-256 with `esp-sha-hmac`, and the default IPsec tunnel mode.
- Keep named NAT ACLs, PAT and ZBF; all three worked alongside GRE/IPsec and eBGP.
- TACACS+ local fallback, login blocking, authenticated NTP and MQC QoS worked. The tested 2911 did not retain the SNMP community command.

## Automation note

The optional MCP bridge successfully placed devices, created links, configured the lab, queried state and saved the `.pkt`. Its static 3650 interface catalog did not match the running device: the working ports were `Gi1/0/1`–`Gi1/0/24`. Live interface discovery and raw link commands were used instead. ASA and modular-router mappings remain a Phase 2 preflight item.

Packet Tracer's script-driven CLI also requires explicit `exit` commands between configuration submodes. Without them, later interface commands can be interpreted inside the previous interface context.
