# ADR 0004: VTP transparent mode

**Status:** Accepted

## Context

Several switches share VLAN definitions.

## Decision

Run **VTP transparent** everywhere and define VLANs on each switch in its config file.

## Alternatives considered

- **VTP server/client:** less typing, but a switch with a higher configuration revision can wipe the VLAN database of the whole domain. That risk isn't worth it on a network this size.
- **VTPv3:** not available in Packet Tracer.

## Consequences

- Each switch only has the VLANs it needs, which also keeps trunk allowed-lists honest.
- VLAN changes are made per switch, which config files make easy to track.
