# ADR 0010: SNMP v2c and type-7 password encryption in the lab

**Status:** Accepted

## Context

Management-plane hardening is part of the design, but Packet Tracer limits what can be shown.

## Decision

- **SNMP v2c**, read-only. The plan restricted it by ACL to the MGMT subnet, but Packet Tracer rejects an ACL on the community ([matrix](../pt-support-matrix.md) P0-16d); management access is limited by the VTY `access-class` instead.
- `service password-encryption` (type 7) to keep plain-text passwords off the screen; all real secrets use `secret` (type 5/8/9 where available).

## Alternatives considered

- **SNMPv3 (authPriv):** the production choice, not available in Packet Tracer.

## Consequences

- Type 7 is reversible and only stops shoulder-surfing, which this ADR states explicitly.
- Production would use SNMPv3, keep secrets out of configs (vault or TACACS+ only), and add CoPP.
