# ADR 0006: GRE over IPsec for Istanbul

**Status:** Accepted

## Context

Istanbul needs encrypted connectivity to London over the internet, with automatic failover between the two London edges.

## Decision

Build **GRE tunnels protected by IPsec** (crypto map in transport mode, matching GRE between the public addresses) to both London edges, and run **OSPF area 1** inside them. Tunnel0 (to EDGE-01) costs 10, Tunnel1 (to EDGE-02) costs 100.

## Alternatives considered

- **Plain policy-based IPsec:** no multicast, so no routing protocol; failover would need static routes and IP SLA.
- **DMVPN or IPsec VTI:** the production choice, but not available in Packet Tracer.

## Consequences

- Dynamic routing and failover over the VPN (test T05).
- Two ABRs summarise in each direction, so each side sees a single route for the other.
- Crypto maps with GRE are more configuration than VTI would be; the production-deltas list says so.
