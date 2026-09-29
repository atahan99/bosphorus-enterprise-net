# ADR 0002: HSRP instead of VRRP

**Status:** Accepted

## Context

The campus needs a first-hop redundancy protocol on the core SVIs and in the DMZ. VRRP is the open standard and my usual preference.

## Decision

Use **HSRP version 2** with MD5 authentication, preemption and interface tracking.

## Alternatives considered

- **VRRP:** not implemented in Packet Tracer.
- **GLBP:** not implemented in Packet Tracer, and load-balancing gateways would complicate the symmetric-path design anyway.

## Consequences

- Functionally equivalent for this design (active/standby per VLAN, load split across VLANs).
- Recorded in the support matrix; a multi-vendor production network would use VRRP.
