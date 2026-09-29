# ADR 0011: Stateless policy between INSIDE and DMZ

**Status:** Accepted

## Context

The DMZ gateway is an HSRP address shared by both edges, and it tracks the ISP-A link so that inbound internet traffic to the DMZ keeps working when ISP-A fails (ADR 0001). In that state the DMZ servers send replies to EDGE-02, while internal users' requests to the DMZ may still reach them through EDGE-01. A stateful firewall on EDGE-02 would see the replies without the session and drop them.

## Decision

The **INSIDE ↔ DMZ zone-pairs use `pass` with tight ACLs** instead of `inspect`:

- INSIDE → DMZ: only web (80/443) and DNS to the two servers, SSH to the DMZ switch from the MGMT subnet, ping from MGMT.
- DMZ → INSIDE: only replies to those flows (TCP `established` from the service ports, DNS responses, echo-replies).

OUTSIDE ↔ DMZ and INSIDE → OUTSIDE stay fully stateful (`inspect`), because those paths are always symmetric.

## Alternatives considered

- **Inspect in both directions:** breaks internal access to the DMZ during a single-ISP failure.
- **Don't track the ISP link in DMZ HSRP:** keeps internal access symmetric but breaks inbound internet access to the DMZ when only ISP-A fails.
- **Firewall state sync between edges:** not available on ISR routers.

## Consequences

- DMZ → INSIDE is still limited to reply packets from specific source ports; a compromised DMZ host can't open connections inside (test T17).
- The known weakness of stateless `established` matching (crafted ACK packets from the service port) is accepted and documented. A production design would put a stateful firewall pair with state sync in front of the DMZ.
