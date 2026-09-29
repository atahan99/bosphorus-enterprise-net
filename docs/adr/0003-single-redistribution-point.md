# ADR 0003: One EIGRP/OSPF redistribution point, with route tags

**Status:** Accepted

## Context

Toronto is an acquired site that runs EIGRP. Re-addressing and migrating it to OSPF on day one is out of scope, so the two routing domains have to be joined.

## Decision

Redistribute in **one place only, LON-WAN-01**, in both directions:

- EIGRP → OSPF: `subnets`, tag 100. Toronto appears in London as O E2.
- OSPF → EIGRP: seed metric for a 2 Mb/s link (required, since EIGRP can't use OSPF's cost), tag 110, explicitly including OSPF external routes so Toronto also learns the default route.
- Route-maps deny routes carrying the other protocol's tag, so nothing is fed back into the protocol it came from.

## Alternatives considered

- **Static routes both ways:** simpler, but Toronto wouldn't learn about new London subnets and the default route would need manual failover.
- **Migrate Toronto to OSPF:** the right long-term answer, but it's a separate project.

## Consequences

- With a single point, routing loops can't form; the tags are there so a second redistribution point (for circuit redundancy) can be added safely.
- Toronto has two circuits into LON-WAN-01, so a circuit failure is covered; LON-WAN-01 itself is a single point of failure for Toronto, accepted and documented. Moving the second circuit to a second London router would need a second redistribution point, which is exactly what the tags are there for.
