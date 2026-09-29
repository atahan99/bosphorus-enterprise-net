# ADR 0005: LACP for the core bundle, no bundling to access switches

**Status:** Accepted

## Context

The two cores need a high-capacity, resilient interconnect. Access switches have one uplink to each core.

## Decision

Bundle the two core-to-core links as **Po1 with LACP** (`mode active`). Access switches keep two separate uplinks, one blocked by STP per VLAN.

## Alternatives considered

- **PAgP:** Cisco-only; LACP is the standard.
- **Multi-chassis EtherChannel to access switches (VSS/StackWise Virtual):** removes STP blocking, but isn't available in Packet Tracer.

## Consequences

- The core link survives a member failure with no STP change (test T03).
- STP still blocks one access uplink per VLAN; load is split by alternating root/HSRP per VLAN.
