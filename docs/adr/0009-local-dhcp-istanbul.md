# ADR 0009: Local DHCP at the branches

**Status:** Accepted

## Context

London hands out addresses from a central DHCP server via relay. Istanbul could do the same over the tunnel.

## Decision

Every branch serves **local DHCP** for all its VLANs from its two site switches, with **split scopes**: SW-01 hands out the lower half of each range, SW-02 the upper half. Either switch can keep the whole office addressed if the other fails.

## Alternatives considered

- **Relay to the London server:** single place to manage scopes, but a WAN outage would stop new devices getting addresses at the branch.

## Consequences

- Branch offices keep working locally when both tunnels are down.
- Scopes are managed per switch; acceptable at this size, and every branch uses the same VLAN numbers and scope layout.
- Each half-scope must be large enough for the whole site on its own (75 addresses for ≤ 36 desks, 115 for corporate Wi-Fi).
