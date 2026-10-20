# ADR 0016: RIPv2 at the Toronto warehouse, redistributed at TOR-RTR-01

**Status:** Accepted (Phase 11, 2026-10-20)

## Context

Toronto's warehouse (TOR-WH) came with the acquisition and still runs RIPv2: one router, one switch, one cable to TOR-RTR-01. It has to reach London's servers, and its cameras have to reach the London IoT server (Phase 12).

[ADR 0003](0003-single-redistribution-point.md) keeps redistribution in one place, LON-WAN-01, because Packet Tracer has no route-maps to tag and filter routes between two redistribution points. Joining the warehouse means adding a second point.

## Decision

Run RIPv2 only between TOR-RTR-01 and the warehouse, and redistribute **one way, at TOR-RTR-01**:

- RIP → EIGRP: `redistribute rip metric 1000000 10 255 1 1500`. Toronto's switches learn the warehouse as D EX routes.
- EIGRP → RIP: nothing. The warehouse gets only a default route (`default-information originate` under `router rip`).
- RIP is `version 2`, `no auto-summary`, `passive-interface default`, with only the warehouse link active.
- Warehouse address space is 10.40.100.0/22, inside Toronto's 10.40.0.0/16. TOR-RTR-01 already summarises 10.40.0.0/16 towards London on both circuits, so LON-WAN-01, OSPF and BGP see no change. The warehouse router has no Loopback0 for the same reason: a 10.255.255.x/32 would fall outside the summary and appear in London.
- The warehouse routes its VLANs on the router (router-on-a-stick). The rest of the design avoids that because a router trunk is a single point of failure; here the router already is the whole site, so it adds none.

The new point is safe without tags because no route can come back. The warehouse has a single exit, so it can never be a transit path between the two domains. It receives only a default, so it has nothing from EIGRP or OSPF to re-advertise. And the RIP routes never reach LON-WAN-01 as separate routes, so they can't be redistributed into OSPF and back.

## Alternatives considered

- **Migrate the warehouse to EIGRP:** the cleanest answer, but the scenario is a site that still runs its legacy protocol.
- **Static routes at TOR-RTR-01 and a static default at the warehouse:** works, but shows no routing protocol, and new warehouse subnets would need manual routes.
- **RIP in the core (on LON-WAN-01 or the circuits):** would put a third protocol into London's redistribution point and require redesigning ADR 0003. Much more change for no gain.

## Consequences

- Two redistribution points now exist, and both are one-way with respect to anything that could loop. If the warehouse ever gets a second exit (to London or to a Toronto switch), this ADR no longer holds: the route-map tags from ADR 0003 would be needed first.
- RIP has no authentication: Packet Tracer has no `ip rip` interface commands on the ISR4331 or the 2911 ([matrix](../pt-support-matrix.md) B-56). Exposure is one cable between two company routers, with RIP passive everywhere else. Production would add `ip rip authentication mode md5`.
- The warehouse is a single point of failure for itself (one router, one switch, one link), accepted for a stub site.
