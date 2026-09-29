# ADR 0002: HSRP instead of VRRP

**Status:** Accepted

## Context

The campus needs a first-hop redundancy protocol on the core SVIs and in the DMZ. VRRP is the open standard and my usual preference.

## Decision

Use **HSRP version 2** with preemption. MD5 authentication and interface tracking were planned, but Packet Tracer rejects every form of both on the 3650 and the ISR4331 ([matrix](../pt-support-matrix.md) P0-07).

## Alternatives considered

- **VRRP:** not implemented in Packet Tracer.
- **GLBP:** not implemented in Packet Tracer, and load-balancing gateways would complicate the symmetric-path design anyway.

## Consequences

- Functionally equivalent for this design (active/standby per VLAN, load split across VLANs).
- Recorded in the support matrix; a multi-vendor production network would use VRRP.
- Without tracking, a gateway switch that loses its uplink stays active and routes through its peer instead of handing over (one extra hop). Production would add MD5 and uplink tracking.
