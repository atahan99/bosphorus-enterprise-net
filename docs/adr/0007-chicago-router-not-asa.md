# ADR 0007: Chicago uses an ISR4331 with GRE over IPsec, not an ASA

**Status:** Accepted (revised in Phase 6, 2026-09-30). Supersedes the first version of this ADR, "Policy-based IPsec and static routing for Chicago".

## Context

Chicago is a 36-person office with a redundant pair of L3 switches. It needs its own internet access, a stateful firewall, and encrypted reachability to the other sites through London.

The first design put an **ASA 5506-X (CHI-FW-01)** at the edge, running a policy-based IPsec tunnel (ASA crypto map) to an IOS crypto map on LON-EDGE-02, with EDGE-01 as a backup peer and static routing. The point of the ASA was platform variety: London and Istanbul already show IOS routers with a zone-based firewall, and Chicago would show the other common branch pattern, a dedicated firewall with a site-to-site VPN.

When Chicago was built in Phase 6, Packet Tracer's ASA model could not deliver that design ([matrix](../pt-support-matrix.md) B-07, B-18, B-19, B-44, B-45):

- **Phase 2 never completed.** IKE Phase 1 came up (`QM_IDLE`), but LON-EDGE-02 answered every quick-mode proposal with "The responder does not have a policy matching initiator's policies" (seen in Simulation mode). Matching AES-256, AES-128 and 3DES transform sets, single and dual peers, equal SA lifetimes, narrower proxy ACLs, crypto-map reordering, rebinding the map and a full network restart all failed. A single ping crossed the tunnel once and could not be reproduced.
- **No NAT exemption.** The ASA rejects manual (twice) NAT, so VPN traffic could not be excluded from PAT.
- **No floating route over the second inside interface.** Two routes to 10.30.0.0/16 on different interfaces collapse into one, and so do two network objects with the same subnet, so the second switch link could not be a backup.
- **Smaller gaps.** IKE `encryption aes-256` is silently not stored (only `encr aes` works), there is no `clear crypto`, and two-peer failover alternated peers unpredictably.

## Decision

Replace CHI-FW-01 with an **ISR4331, CHI-RTR-01**, built like IST-RTR-01 ([ADR 0006](0006-gre-over-ipsec-istanbul.md)):

- The same three links: Gi0/0/0 to ISP-B (198.51.100.14), Gi0/0/1 and Gi0/0/2 to the two site switches (10.30.0.13 and 10.30.0.5; the SW-01 link avoids 10.30.0.0/30, matrix B-47).
- Local internet breakout with interface PAT and a stateful firewall (CBAC outbound, VPN peers only inbound, [ADR 0015](0015-cbac-not-zbf.md)).
- **GRE over IPsec to both London edges** (Tunnel0 to EDGE-01, Tunnel1 to EDGE-02) with **OSPF area 2** inside the tunnels. The edges summarise 10.10.0.0/16 into area 2; Chicago's subnets reach area 0 unsummarised (B-48).
- The Chicago switches run OSPF area 2 on their router link and VLAN 900, and keep a static default to the router for local breakout.

## Alternatives considered

- **Keep debugging ASA ↔ IOS IPsec:** no configuration change moved it past quick mode; the cause is inside Packet Tracer's ASA/IOS interoperability.
- **Keep the ASA for internet and add an ISR4331 behind it for the VPN:** keeps an ASA in the lab, but the GRE/IPsec traffic would have to cross the ASA through a static NAT (NAT-T), another untested PT path on the same weak device model, for extra devices and links.
- **Drop the Chicago VPN:** Chicago would still reach the internet but not the other sites.

## Consequences

- Chicago gains automatic, routed failover between the London edges (both tunnels up, OSPF chooses), which the ASA design only approximated with a backup peer and floating statics.
- All three branches now use one of two proven patterns (GRE/IPsec + OSPF for Istanbul and Chicago, serial + EIGRP for Toronto). The lab no longer shows an ASA; in production a branch firewall appliance would be a normal choice.
- EDGE-02 no longer needs a static route to 10.30.0.0/16 or `redistribute static`, so the unfilterable export of Null0 anchors described in [ADR 0001](0001-active-standby-internet-edge.md) disappears.
- The edges no longer need a Chicago-specific firewall rule: branch traffic arrives decrypted on the tunnels, which are unfiltered (ADR 0015).
- CHI-RTR-01 and Chicago's single ISP remain single points of failure, as for Istanbul.
