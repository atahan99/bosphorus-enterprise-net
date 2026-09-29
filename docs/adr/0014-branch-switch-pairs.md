# ADR 0014: Redundant switch pairs at the branches

**Status:** Accepted

## Context

Branches have 30–36 staff. The first design used a single router or L3 core switch with access switches hanging off it, which made one switch (or one router trunk) a single point of failure for the whole office. Packet Tracer's switches have 24 ports, so the design also has to use ports efficiently.

## Decision

Every branch uses **two 3650-24PS switches acting as collapsed core and access**:

- Joined by a two-link LACP bundle (Po1) carrying all VLANs and a routed transit VLAN 900.
- HSRP for every gateway VLAN (SW-01 active for 10, 20, 60; SW-02 for 50, 70). Packet Tracer has no HSRP tracking, so a switch that loses its uplink keeps its gateways and forwards over VLAN 900 to its peer.
- Each switch has its **own routed uplink** to the branch router (Istanbul, Toronto) or firewall (Chicago), so the WAN survives the loss of either switch.
- Desks, APs and DHCP scopes are split across the pair.
- Istanbul runs OSPF area 1 between the router and both switches; Toronto runs EIGRP (no stubs: Packet Tracer lacks `eigrp stub`, so `passive-interface default` limits neighbors to the uplink and VLAN 900); Chicago uses static routes with floating backups.

## Alternatives considered

- **Single L3 core + access switches:** one more box per site and a site-wide single point of failure in the core.
- **Router-on-a-stick with chained access switches:** fewest devices, but both the router trunk and the first switch are single points of failure.
- **A router/firewall pair per branch:** removes the last single point of failure, but doubles WAN equipment and needs a second ISP to be worth it at this size.

## Consequences

- A switch failure costs only half the desks at a branch; the rest of the office, Wi-Fi on the surviving switch, DHCP and the WAN keep working (test T21).
- The branch router or firewall and its ISP remain single points of failure, documented as production upgrades.
- Headcounts (30 and 36) were chosen so each pair's 40 desk-capable ports are used with a small margin.
- Router-on-a-stick is no longer part of the build; it stays in the Phase 0 tests.
