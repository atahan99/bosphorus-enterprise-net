# ADR 0003: One EIGRP/OSPF redistribution point

**Status:** Accepted (route tags dropped after Phase 0, 2026-09-29)

## Context

Toronto is an acquired site that runs EIGRP. Re-addressing and migrating it to OSPF on day one is out of scope, so the two routing domains have to be joined.

## Decision

Redistribute in **one place only, LON-WAN-01**, in both directions:

- EIGRP → OSPF: `subnets`. Toronto appears in London as O E2.
- OSPF → EIGRP: seed metric for a 2 Mb/s link (required, since EIGRP can't use OSPF's cost), explicitly including OSPF external routes (`match internal external 1 external 2`) so Toronto also learns the default route.

The original design also tagged each direction (100 and 110) and used route-maps to deny re-import of the other protocol's routes. Packet Tracer has no route-maps ([matrix](../pt-support-matrix.md) P0-10), so the tags are gone.

## Alternatives considered

- **Static routes both ways:** simpler, but Toronto wouldn't learn about new London subnets and the default route would need manual failover.
- **Migrate Toronto to OSPF:** the right long-term answer, but it's a separate project.

## Consequences

- With a single point, routing loops can't form, with or without tags.
- Toronto has two circuits into LON-WAN-01, so a circuit failure is covered; LON-WAN-01 itself is a single point of failure for Toronto, accepted and documented. Moving the second circuit to a second London router would need a second redistribution point, which is only safe with the tag-and-deny route-maps; in production they would come back first.
- Phase 11 added a second, one-way point at TOR-RTR-01 for the RIP warehouse. It needs no tags because the warehouse has one exit and only receives a default ([ADR 0016](0016-rip-warehouse-second-redistribution.md)).
