# ADR 0001: Active/standby internet edge instead of load-sharing

**Status:** Accepted

## Context

London has two edge routers, each with its own ISP. Each edge translates internal traffic to its own PAT address (203.0.113.1 and 203.0.113.2), and each runs a stateful zone-based firewall.

## Decision

Run the edge **active/standby**:

- **Outbound:** both edges originate a default route into OSPF (E2); EDGE-01 with metric 10, EDGE-02 with metric 100. Every internal router prefers EDGE-01.
- **Inbound:** EDGE-02 prepends AS 65100 twice towards ISP-B, so the internet prefers the path through ISP-A.
- **Failover:** `default-information originate` is configured without `always`, so when EDGE-01 loses its BGP default it stops originating the OSPF default and stops advertising the /24 to ISP-A. Both directions move to EDGE-02 at the same time.

## Alternatives considered

- **Load-share outbound** (equal OSPF metrics): rejected. Return traffic would arrive at whichever edge the internet prefers, which may not hold the NAT translation or the firewall session, so connections break.
- **Shared NAT address with stateful failover:** not possible on ISR routers in Packet Tracer (no NAT/firewall state sync).

## Consequences

- Paths are symmetric in every failure case, which NAT and a stateful firewall require.
- ISP-B's bandwidth is idle in normal operation. In production, this is where SD-WAN or an ASA/FTD pair with state sync would be considered.
- The DMZ gateway (HSRP) also tracks the ISP-A link so inbound DMZ traffic follows the same edge (see ADR 0011).
