# ADR 0006: GRE over IPsec for Istanbul

**Status:** Accepted

## Context

Istanbul needs encrypted connectivity to London over the internet, with automatic failover between the two London edges.

## Decision

Build **GRE tunnels protected by IPsec** (crypto map matching GRE between the public addresses; PT only offers tunnel mode, P0-14d) to both London edges, and run **OSPF area 1** inside them. Packet Tracer's ISR4331 accepts no `ip ospf cost` on a tunnel (B-40), so both tunnels have the same cost; EDGE-01 is still the primary path in both directions because EDGE-02's core link costs 1000. Chicago uses the same pattern with area 2 ([ADR 0007](0007-chicago-router-not-asa.md)).

## Alternatives considered

- **Plain policy-based IPsec:** no multicast, so no routing protocol; failover would need static routes and IP SLA.
- **DMVPN or IPsec VTI:** the production choice, but not available in Packet Tracer.

## Consequences

- Dynamic routing and failover over the VPN (test T05).
- The ABRs summarise London (10.10.0.0/16) towards the branches. Summarising the branches into area 0 was dropped because PT generates it unreliably with two branch areas (B-48); London carries each branch's subnets.
- Crypto maps with GRE are more configuration than VTI would be; the production-deltas list says so.
