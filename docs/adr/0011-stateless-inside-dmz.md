# ADR 0011: Stateless policy between INSIDE and DMZ

**Status:** Accepted (since Phase 6 implemented with ACLs on the DMZ interface, [ADR 0015](0015-cbac-not-zbf.md))

## Context

The DMZ gateway is an HSRP address shared by both edges. After an ISP-A failure it moves to EDGE-02 so that inbound internet traffic to the DMZ keeps working (ADR 0001). The design had HSRP track the ISP-A link; Packet Tracer has no HSRP tracking, so the move is made by hand (test T04), but the resulting state is the same. In that state the DMZ servers send replies to EDGE-02, while internal users' requests to the DMZ may still reach them through EDGE-01. A stateful firewall on EDGE-02 would see the replies without the session and drop them.

## Decision

Traffic between the inside and the DMZ is **passed statelessly with tight ACLs** instead of being inspected (first as ZBF `pass` zone-pairs, since Phase 6 as `ACL-TO-DMZ` and `ACL-DMZ-IN` on the DMZ interface):

- INSIDE → DMZ: only web (80/443) and DNS to the two servers, SSH to the DMZ switch from the MGMT subnet, ping from MGMT.
- DMZ → INSIDE: only replies to those flows (TCP `established` from the service ports, DNS responses, echo-replies).

OUTSIDE ↔ DMZ and INSIDE → OUTSIDE stay fully stateful (`inspect`), because those paths are always symmetric.

## Alternatives considered

- **Inspect in both directions:** breaks internal access to the DMZ during a single-ISP failure.
- **Leave the DMZ gateway on EDGE-01 after an ISP-A failure:** keeps internal access symmetric but breaks inbound internet access to the DMZ when only ISP-A fails.
- **Firewall state sync between edges:** not available on ISR routers.

## Consequences

- DMZ → INSIDE is still limited to reply packets from specific source ports; a compromised DMZ host can't open connections inside (test T17).
- The known weakness of stateless `established` matching (crafted ACK packets from the service port) is accepted and documented. A production design would put a stateful firewall pair with state sync in front of the DMZ.
