# ADR 0012: Static printer on a DAI-protected switch

**Status:** Accepted

## Context

Dynamic ARP Inspection validates ARP against the DHCP snooping table. Printers (two in London, one per branch) have static addresses, so they have no binding and DAI would drop their ARP.

## Decision

Trust DAI on each printer's port only (Fa0/21 on London access switches, Gi1/0/20 on branch switches), and lock the port to **one sticky MAC** with port security.

## Alternatives considered

- **ARP ACL (`arp access-list` + `ip arp inspection filter`):** stricter, but needs the printer's MAC hard-coded in the config. Tested in Phase 0 (P0-05b); a good production option.
- **DHCP reservation:** Packet Tracer's DHCP server doesn't support reservations.

## Consequences

- A different device/MAC is blocked by port security. This is a lab compromise, not complete anti-spoofing: a device that copies the printer's permitted MAC, or a compromised printer, could still send forged ARP.
- One exception to document instead of a network-wide weakening of DAI.
