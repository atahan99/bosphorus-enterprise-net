# ADR 0001: Active/standby internet edge instead of load-sharing

**Status:** Accepted (mechanisms revised after Phase 0, 2026-09-29)

## Context

London has two edge routers, each with its own ISP. Each edge translates internal traffic to its own PAT address (203.0.113.1 and 203.0.113.2), and each runs a stateful zone-based firewall.

Phase 0 showed that Packet Tracer lacks the tools the first version of this decision relied on: iBGP, prefix-lists, route-maps (so no AS-path prepend), neighbor `default-originate`, and `metric` / `metric-type` on `default-information originate` (see [`pt-support-matrix.md`](../pt-support-matrix.md) P0-10 to P0-13).

## Decision

Run the edge **active/standby**, with EDGE-01 active:

- **Outbound:** both edges originate a default into OSPF with `default-information originate`, so both are E2 with metric 1. The tie is broken by distance to the ASBR: the EDGE-02 ↔ LON-CORE-02 link has `ip ospf cost 1000` on both ends, so every internal router prefers EDGE-01.
- **Inbound:** EDGE-01 announces the two halves `203.0.113.0/25` and `203.0.113.128/25`; EDGE-02 announces the whole `203.0.113.0/24`. Longest match sends inbound traffic to ISP-A while EDGE-01's session is up.
- **ISP default:** each ISP router advertises `0.0.0.0/0` with a static default to Null0 and `network 0.0.0.0 mask 0.0.0.0` (verified on Router-PT in Phase 0).
- **Failover:** `default-information originate` has no `always`, so when EDGE-01 loses its BGP session it stops originating the OSPF default and its /25s disappear from the internet. Outbound moves to EDGE-02's default, inbound to the /24 via ISP-B. Both directions move together.
- The edges run no BGP between themselves.

## Alternatives considered

- **Load-share outbound** (equal OSPF metrics and costs): rejected. Return traffic would arrive at whichever edge the internet prefers, which may not hold the NAT translation or the firewall session, so connections break.
- **Shared NAT address with stateful failover:** not possible on ISR routers in Packet Tracer (no NAT/firewall state sync).
- **AS-path prepend and OSPF default metrics 10/100** (the original mechanisms): the standard production approach, not available in Packet Tracer.

## Consequences

- Paths are symmetric in the tested failure (ISP-A link or EDGE-01 down), which NAT and a stateful firewall require.
- ISP-B's bandwidth is idle in normal operation. In production, this is where SD-WAN or an ASA/FTD pair with state sync would be considered.
- The outbound tie-break relies on OSPF's forward-metric rule for equal E2 routes. It is verified during Phase 5 (matrix B-25).
- Packet Tracer has no HSRP tracking, so the DMZ gateway (HSRP 100) no longer follows an ISP-A failure by itself. After failover, published DMZ services stay reachable only once EDGE-01's DMZ interface is shut or its HSRP priority lowered by hand (test T04).
- Because the static redistribution for the Chicago backup can't be filtered, the Null0 anchors for the public block are also exported into OSPF. Internal hosts can't reach the company's public addresses from inside, which no designed flow needs.
- London accepts every prefix the ISPs send and exports only what its `network` statements list; there are no inbound or outbound BGP filters and no session passwords.
