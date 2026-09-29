# ADR 0007: Policy-based IPsec and static routing for Chicago

**Status:** Accepted

## Context

Chicago is a 36-person office behind an ASA, with a redundant pair of L3 switches routing its VLANs. It needs reachability to the other sites and its own internet access, and the whole site summarises to one prefix, 10.30.0.0/16.

## Decision

Use a **policy-based IPsec tunnel** (ASA crypto map to IOS crypto map on LON-EDGE-02) for 10.30.0.0/16 ↔ 10.0.0.0/8, a static route for 10.30.0.0/16 on EDGE-02 (tracked against its ISP link) redistributed into OSPF, NAT exemption on both ends, and static default routes from both site switches to the ASA. **LON-EDGE-01 is a backup peer:** the ASA's crypto map lists both edges, and EDGE-01 holds a floating static route (AD 250) that only enters routing when EDGE-02's route is withdrawn.

## Alternatives considered

- **GRE + OSPF like Istanbul:** the ASA doesn't terminate GRE, and with a single summary prefix a dynamic protocol adds little.
- **GRE tunnels to both edges with a routing protocol:** automatic in both directions, but the ASA can't terminate GRE.
- **Single peer only:** simpler, but an EDGE-02 or ISP-B failure would cut Chicago off from the other sites.

## Consequences

- Simple and typical for a branch behind a firewall.
- Failover relies on the ASA noticing the primary peer is gone and on EDGE-02's route being withdrawn when its ISP link drops. A failure that keeps EDGE-02's link up but breaks the path beyond it isn't detected (production: IP SLA tracking, or a route-based VPN with a routing protocol).
- The ASA and Chicago's single ISP remain single points of failure.
- The zone-based firewall on EDGE-02 passes Chicago traffic statelessly in both directions (trusted site).
