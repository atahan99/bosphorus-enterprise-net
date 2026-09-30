# ADR 0015: CBAC and ACLs instead of a zone-based firewall

**Status:** Accepted (Phase 6, 2026-09-30). Replaces the zone-based firewall built in Phase 5.

## Context

Phase 5 built a zone-based firewall (ZBF) on both London edges (zones INSIDE, DMZ, OUTSIDE), and the branch routers used the same template with INSIDE/OUTSIDE. Phase 6 added GRE over IPsec tunnels from the edges to IST-RTR-01 and CHI-RTR-01.

In ZBF, traffic between an interface that is in a zone and one that isn't is dropped, so the tunnels had to be zone members. Packet Tracer's ISR4331 rejects `zone-member security` on a Tunnel interface ([matrix](../pt-support-matrix.md) B-40). The result was that every packet between a branch LAN and London was dropped twice: on the branch router (LAN in INSIDE → tunnel with no zone) and on the edge (tunnel with no zone → core link in INSIDE). Simulation mode showed the packets stopping at exactly those two points, and removing the zone membership from the branch LAN link moved the drop to the edge.

## Decision

Use the classic IOS firewall, **CBAC (`ip inspect`) plus ACLs**, on all four routers that had ZBF. It works per interface and doesn't need zones, so the tunnels need no special treatment. The Phase 5 policy is kept:

| Old zone-pair | Now |
|---|---|
| INSIDE → OUTSIDE (inspect) | `ip inspect CBAC-OUT out` on the ISP interface |
| OUTSIDE → INSIDE (drop) | `ACL-OUTSIDE-IN` on the ISP interface: only BGP from the ISP, IKE/ESP/GRE from the VPN peers and the published DMZ services; everything else denied |
| OUTSIDE → DMZ (inspect published services) | Permitted in `ACL-OUTSIDE-IN` (public addresses, checked before NAT) and in `ACL-TO-DMZ`; `ip inspect CBAC-OUT out` on the DMZ interface keeps it stateful |
| INSIDE → DMZ, DMZ → INSIDE (stateless pass, [ADR 0011](0011-stateless-inside-dmz.md)) | `ACL-TO-DMZ` out and `ACL-DMZ-IN` in on the DMZ interface, with the same lines |
| DMZ → OUTSIDE (inspect) | Permitted in `ACL-DMZ-IN`, inspected on the ISP interface |
| Self zone (default allow) | Router traffic is not filtered outbound; the inbound ACLs list BGP, IKE, ESP, GRE and HSRP explicitly |

The branch routers inspect outbound on their ISP interface and allow only IKE, ESP and GRE from the two London edges inbound.

## Alternatives considered

- **Keep ZBF and replace GRE with policy-based IPsec:** decrypted traffic would arrive on the OUTSIDE interface, where a zone-pair can pass it. But without GRE there is no OSPF over the VPN, so Istanbul and Chicago would lose dynamic failover and fall back to static routes and redistribution, which Packet Tracer handles badly (B-37, B-46).
- **Leave the firewall off on the tunnel paths:** simpler, but it leaves the edges and branches without a stateful policy towards the internet.

## Consequences

- Branch ↔ London traffic works (tested from all three branches), and the Phase 5 checks still pass: London browses the internet, the internet reaches the published DMZ site, inside users reach the DMZ, and HSRP keeps running on the DMZ.
- CBAC is the older IOS firewall model; in production the zone-based firewall (with tunnel interfaces as zone members) or a dedicated firewall would be used. The per-interface ACLs are longer than the zone-pair policies they replace.
- Traffic between the branches and London is not inspected (it wasn't inspected under ZBF either: the tunnels were effectively trusted).
- Packet Tracer rejects the `time-exceeded` ICMP keyword, so traceroutes through the branch routers show `*` for the hops beyond them.
