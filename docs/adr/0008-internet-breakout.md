# ADR 0008: Local internet breakout for Istanbul and Chicago; Toronto through London

**Status:** Accepted

## Context

Each branch needs internet access.

## Decision

Istanbul and Chicago **break out locally** through their own ISP connection (with PAT and a firewall). Toronto has **no internet connection** and reaches the internet through London.

## Alternatives considered

- **Everything through London (backhaul):** central inspection, but wastes VPN bandwidth and adds latency; it also makes London's edge a single point of failure for every site.
- **Local breakout for Toronto:** would need a new ISP circuit and firewall at an acquired site that isn't integrated yet.

## Consequences

- Each branch needs its own firewall policy (done with ZBF on IST-RTR-01 and the ASA in Chicago).
- Toronto's internet traffic uses London's edge and is covered by London's failover design.
