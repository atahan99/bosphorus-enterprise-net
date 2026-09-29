# Design plan

The full design for bosphorus-enterprise-net. Addresses live in [addressing-plan.md](addressing-plan.md) and ports in [cabling-plan.md](cabling-plan.md); this document explains what is built and why.

## 1. Goals

The project has to show that I can:

- **Design** a multi-site enterprise network from requirements: addressing, segmentation, routing, redundancy.
- **Justify** it: each significant decision has a short ADR in [`adr/`](adr/).
- **Harden and verify** it: a written test plan with evidence, including deliberate failures and attacks.
- **Be honest about the tool**: a verified [Packet Tracer support matrix](pt-support-matrix.md) and a list of what production hardware would change.

## 2. Scope

| v1.0 (core) | v1.1+ (stretch) |
|---|---|
| London HQ, three remote sites, simulated internet with two ISPs | IPv6 dual-stack with OSPFv3 |
| VLANs, 802.1Q, Rapid PVST+, LACP, L2 security | Remote-access VPN (Easy VPN) for a teleworker |
| OSPF multi-area, EIGRP with redistribution, eBGP | PT Network Controller + Python in PT |
| HSRP, dual-ISP failover, backup VPN tunnel | Config linting and Batfish checks in GitHub Actions |
| GRE over IPsec, ASA IPsec, PPP/CHAP | OT zone using PT 9 industrial devices |
| Zone-based firewall, NAT/PAT, DMZ | |
| Wi-Fi at every site: WLC + lightweight APs in London, autonomous APs at branches | |
| CME voice for all four sites, QoS (if Phase 0 confirms support) | |
| AAA, SSH, NTP, Syslog, SNMP, TFTP backups | |

## 3. Scenario

A fictional professional-services company with **182 staff** across four sites and its own public /24.

| Site | Staff | Role | Why it looks the way it does |
|---|---|---|---|
| **London (LON)** | 80 | HQ | Dual-homed to two ISPs with its own address space, which is what makes **BGP** necessary. Redundant collapsed core, OSPF area 0, the server room, DMZ and call control for the whole company. |
| **Istanbul (IST)** | 30 | Regional office | Runs a routing protocol over its VPN: **OSPF area 1** over GRE/IPsec with a backup tunnel, extended to its two site switches. |
| **Chicago (CHI)** | 36 | Office behind a firewall | One ASA and one site summary (10.30.0.0/16): **static routes** are all it needs, with a backup VPN peer in London. |
| **Toronto (TOR)** | 36 | Recently acquired company | Kept its legacy **EIGRP** design and private circuits; integrated by **redistribution** at a single point. No local internet: it uses London's. |

The Toronto circuits are modelled as PT serial links. In reality they would be carrier Ethernet or MPLS services; the limitation is recorded in the support matrix.

## 4. Sizing

Headcounts were chosen so the equipment Packet Tracer offers (24-port switches) is used efficiently **and** every site is redundant: no single switch failure takes a site down. Packet Tracer holds a **representative sample** of endpoints (one or two per VLAN, enough to exercise every policy and test). Desk ports and DHCP scopes are sized for the full headcount; the branch AP layout and its port-security limit are demo-scale because AccessPoint-PT carries only one SSID per AP.

### Access ports

One switch port per desk: the PC plugs into the IP phone, the phone into the port.

| Site | Staff | Switches | Port budget | Desk ports | Spare |
|---|---|---|---|---|---|
| London | 80 (ENG 40, SALES 25, FIN-HR 15) | 4 × 2960-24TT access (dual-homed to the core pair) + services switch | Each: 20 desks, 1 AP, printer on two of them (uplinks are the separate Gi ports) | 80 | 10 |
| Istanbul | 30 | 2 × 3650-24PS (pair) | Each: Po1 (2), router link (1), AP (1), 20 desk-capable ports; printer on SW-02 | 30 of 39 | 9 |
| Chicago | 36 | 2 × 3650-24PS (pair) | Each: Po1 (2), ASA link (1), AP (1), 20 desk-capable ports; printer on SW-02 | 36 of 39 | 3 |
| Toronto | 36 | 2 × 3650-24PS (pair) | Same as Istanbul and Chicago | 36 of 39 | 3 |

The branch 3650s supply PoE, so branch phones and APs need no power adapters. London's 2960-24TTs don't, which is recorded as a PT limitation (production would use PoE access switches).

### Wireless

Planning assumption: 1.5 wireless devices per person (laptop + phone), about 30 clients per production AP. The four London APs reflect this capacity model. Each branch uses two single-SSID AccessPoint-PT devices to demonstrate corporate and guest segmentation, not to claim production RF/client capacity.

| Site | Wireless clients | APs |
|---|---|---|
| London | ~120 staff devices + guests | 4 lightweight APs (one per access switch), WLC |
| Istanbul | ~45 + guests | 2 (one per switch) |
| Chicago / Toronto | ~55 + guests | 2 each (one per switch) |

### DHCP scopes

All scopes are 151 addresses (.100–.250), except branch corporate Wi-Fi (231, .20–.250). The largest demand is London's corporate Wi-Fi at ~120 devices. At the branches each scope is split between the two switches, so either can keep handing out addresses alone.

### Call control

CME in London registers every phone in the company. The lab configures eight directory numbers (three London phones, one per branch, two spare); production would have one per employee (~180).

## 5. Device inventory

See [cabling-plan.md §1](cabling-plan.md#1-devices-and-modules) for models and modules.

**Naming:** `SITE-ROLE-NN` (e.g. `LON-CORE-01`, `IST-RTR-01`). Interface descriptions: `TO-<peer>-<peer-port>`.

## 6. Layer 2 (London)

**Topology.** Four access switches (two floors) and a services switch each uplink Gi0/1 to LON-CORE-01 and Gi0/2 to LON-CORE-02; STP blocks one of them per VLAN. The cores are joined by Po1, a two-link LACP bundle that trunks all VLANs and carries the VLAN 900 routed transit.

**VTP:** transparent everywhere, so no switch can overwrite the VLAN database by having a higher revision number.

**Trunks:** `switchport mode trunk`, `switchport nonegotiate`, native VLAN 999, explicit allowed-VLAN list on every trunk (access trunks only carry that switch's VLANs).

**Spanning tree:**

- `spanning-tree mode rapid-pvst` on every switch.
- Root primary/secondary per VLAN matching HSRP: CORE-01 root for 10/30/60/70 (60 and 70 because the WLC hangs off CORE-01), CORE-02 root for 20/40/50/80.
- Access ports: PortFast and BPDU guard.
- Core ports facing access switches: **root guard**.
- No loop guard and no automatic err-disable recovery: Packet Tracer has neither (matrix P0-02b/c, P0-03). A port err-disabled by BPDU guard is recovered with `shutdown` / `no shutdown` once the rogue switch is removed. Production would add both.

**Access ports:** `switchport mode access`, `switchport nonegotiate`, voice VLAN on phone ports. Unused ports in VLAN 998 and shut down.

**L2 security:**

- **Port security:** maximum 3 MACs (PC + phone), sticky, violation `restrict`.
- **DHCP snooping** on VLANs 10–70, uplinks trusted, and `no ip dhcp snooping information option`. Without that last line, the access switch inserts option 82 and the relay drops the requests: a classic gotcha worth a paragraph in the blog.
- **DAI** on every user access switch for its user and voice VLANs, uplinks trusted. Packet Tracer accepts the configuration (the 2960s must run IOS 15, matrix B-24) and shows DAI active, but does not drop invalid ARP (matrix P0-05a), so it is a documented control rather than a demonstrated one. Static printers' ports are trusted and locked to one sticky MAC ([ADR 0012](adr/0012-static-printer-dai.md)); LON-SRV-SW01 (static servers and CME) doesn't run DAI and relies on port security.
- **Wired 802.1X** was exercised in Phase 0: the IOS 15 2960 accepts and enforces it, but Packet Tracer's PC never completes the EAP exchange (matrix P0-06). It is not applied to the main lab access ports; wireless 802.1X demonstrates RADIUS, while production would add wired 802.1X with MAB and a staged rollout.
- **Rogue switch demo:** LON-ROGUE-SW is pre-configured with bridge priority 0 for VLAN 20 and left disconnected; it's cabled to LON-ACC-01 Fa0/24 (a spare desk port) only for tests T18–T19.

**Branches** use a smaller version of the same idea ([ADR 0014](adr/0014-branch-switch-pairs.md)): two 3650s acting as collapsed core and access, joined by a two-link LACP bundle (Po1) that carries every VLAN plus a routed transit VLAN 900. Each switch has its own routed uplink to the branch router or firewall, half the desks and one AP. SW-01 is STP root and HSRP active for VLANs 10, 20, 60; SW-02 for 50 and 70.

## 7. Layer 3

### Inter-VLAN routing

Every site routes on L3 switch SVIs with HSRP. (Router-on-a-stick is still exercised in Phase 0, but no longer used in the build: a single router trunk would be a single point of failure.)

### HSRP

- **London core and branch pairs:** version 2, preempt, group number = VLAN ID. Active switch priority 110, standby 100. Packet Tracer has no HSRP authentication or tracking (matrix P0-07): a switch that loses its uplink keeps its gateways and routes through its peer (OSPF in London, VLAN 900 at the branches). Production would add MD5 and uplink tracking.
- **DMZ:** LON-EDGE-01 and LON-EDGE-02 share 172.16.100.1 (group 100), EDGE-01 active. Without tracking the DMZ gateway doesn't follow an ISP-A failure; it is moved by hand in test T04 ([ADR 0001](adr/0001-active-standby-internet-edge.md), [ADR 0011](adr/0011-stateless-inside-dmz.md)).

### OSPF (process 1)

| Area | Members |
|---|---|
| 0 | LON-EDGE-01/02, LON-CORE-01/02, LON-WAN-01 |
| 1 | Both GRE tunnels, IST-RTR-01 and IST-SW-01/02. EDGE-01 and EDGE-02 are the ABRs |

- `passive-interface default`, enabling only real adjacencies.
- `ip ospf network point-to-point` on /30s and tunnels.
- `auto-cost reference-bandwidth 10000`.
- MD5 authentication on both areas.
- Router IDs from Loopback0.
- Summarisation at the ABRs: `area 0 range 10.10.0.0 255.255.0.0` (Istanbul sees one London route) and `area 1 range 10.20.0.0 255.255.0.0` (London sees one Istanbul route).
- Tunnel costs: Tunnel0 = 10, Tunnel1 = 100, so Tunnel1 is a hot standby.

### EIGRP AS 100 (Toronto) and redistribution

- TOR-RTR-01, TOR-SW-01/02 and LON-WAN-01's two serial interfaces run classic EIGRP, `no auto-summary`, MD5 via key chain (verified in Phase 0). The two circuits load-share and back each other up.
- The Toronto switches run EIGRP with `passive-interface default`, forming neighbors only on their uplink and VLAN 900 (Packet Tracer has no EIGRP stub, matrix B-03). TOR-RTR-01 **summarises** Toronto as 10.40.0.0/16 towards London on both circuits.
- **One redistribution point: LON-WAN-01.**
  - EIGRP → OSPF: `redistribute eigrp 100 subnets` (Toronto appears in London as one O E2 route, 10.40.0.0/16).
  - OSPF → EIGRP: `redistribute ospf 1 metric 2000 2000 255 1 1500 match internal external 1 external 2`. The seed metric is required (EIGRP can't use OSPF's cost); external routes are matched explicitly because London's default route is an OSPF external route.
  - No route tags: Packet Tracer has no route-maps (matrix P0-10). With a single point, loops can't form; production would tag each direction and deny re-import before adding a second point.

### BGP (London = AS 65100)

| Session | Type |
|---|---|
| LON-EDGE-01 ↔ ISP-A (AS 65001) | eBGP |
| LON-EDGE-02 ↔ ISP-B (AS 65002) | eBGP |
| ISP-A ↔ ISP-B ↔ INET-CORE (AS 65000) | Full mesh forming "the internet" |

Packet Tracer supports eBGP only: no iBGP, prefix-lists, route-maps, session passwords or `default-originate` (matrix P0-11). The edges therefore run no BGP between themselves, and the policy below uses only `network` statements.

- **Advertise:** EDGE-01 announces `203.0.113.0/25` and `203.0.113.128/25`; EDGE-02 announces `203.0.113.0/24`. Each is anchored by a static route to Null0.
- **Receive:** each ISP advertises `0.0.0.0/0` (static default to Null0 plus `network 0.0.0.0`) and the rest of the lab internet, including 192.0.2.0/24. Nothing is filtered.
- **Transit:** London only ever announces its own block, so it can't become a transit path between the ISPs even without an outbound filter.

### Egress and ingress: the key design decision

- **Outbound:** each edge runs `default-information originate` into OSPF (without `always`). Packet Tracer can't set its metric, so both are E2 metric 1. The EDGE-02 ↔ LON-CORE-02 link costs 1000, so EDGE-01 is the nearer ASBR and wins everywhere.
- **Inbound:** longest match. The internet prefers EDGE-01's /25s via ISP-A over EDGE-02's /24 via ISP-B.
- **Why active/standby rather than load-sharing:** each edge translates to its own PAT address. Traffic that left via EDGE-01 and returned via EDGE-02 would hit a router with no NAT entry and break.
- **Failure behaviour:** if the ISP-A link or EDGE-01 fails, EDGE-01 loses its BGP default, stops originating the OSPF default and its /25s are withdrawn. Both directions move to EDGE-02 together. The DMZ gateway (HSRP) can't track the ISP link in Packet Tracer, so it is moved by hand during the test. See [ADR 0001](adr/0001-active-standby-internet-edge.md).

## 8. WAN and VPN

| Link | Design |
|---|---|
| **IST ↔ LON** | GRE over IPsec. GRE sourced from the public interfaces (EDGE-01 198.51.100.2, EDGE-02 198.51.100.6, IST 198.51.100.10). Crypto map on the physical outside interface matching `permit gre host X host Y`. IKEv1 PSK, AES-256 with SHA-1 and DH group 5, ESP `esp-aes 256 esp-sha-hmac` in tunnel mode: the strongest Packet Tracer accepts (matrix P0-14; production would use SHA-256 and group 14+). OSPF runs inside the tunnels, which is the reason for GRE. |
| **CHI ↔ LON** | ASA crypto map ↔ IOS crypto map on LON-EDGE-02 (policy-based). Interesting traffic 10.30.0.0/16 ↔ 10.0.0.0/8. EDGE-02 has a static route for 10.30.0.0/16 via ISP-B and redistributes its statics into OSPF (`redistribute static subnets`). Packet Tracer can't track the route or filter the redistribution, so the route only goes when the ISP-B link itself drops, and the Null0 anchors for the public block are exported too (harmless, see ADR 0001). **Backup:** the ASA lists EDGE-01 as a second peer, and EDGE-01 has a floating static route (AD 250) that is only installed and advertised once EDGE-02's route is withdrawn. **NAT exemption on both ends** (deny line in the IOS NAT ACL, identity NAT on the ASA), otherwise traffic is PAT-translated before it reaches the crypto map. |
| **TOR ↔ LON** | Serial, `encapsulation ppp`, `ppp authentication chap`, per-peer `username` entries. LON-WAN-01 is DCE and sets `clock rate`. |

Chicago keeps working if EDGE-02 or ISP-B's link to it fails (test T23). The ASA itself and Chicago's single ISP remain single points of failure (production: ASA HA pair, second ISP or LTE).

## 9. Internet edge: NAT, zone-based firewall, DMZ

**NAT (each edge):** PAT for 10.0.0.0/8 to that edge's own 203.0.113.x, excluding site-to-site traffic; static NAT for the DMZ web and DNS servers, configured identically on both edges.

**Zones (each edge):**

| Traffic | Policy |
|---|---|
| INSIDE → OUTSIDE | Inspect HTTP, HTTPS, DNS, ICMP |
| INSIDE → DMZ | Pass (stateless): web and DNS to the two servers, SSH to the DMZ switch from MGMT |
| OUTSIDE → DMZ | Inspect HTTP, HTTPS, DNS to the static NAT hosts only |
| DMZ → INSIDE | Pass replies to the flows above only (TCP `established`, DNS responses, echo-replies); everything else dropped |
| DMZ → OUTSIDE | Inspect HTTP, HTTPS, DNS, ICMP |
| OUTSIDE → INSIDE | Drop (EDGE-02: pass Chicago VPN traffic) |

INSIDE ↔ DMZ is stateless because the DMZ gateway can move to EDGE-02 while internal traffic still enters the DMZ through EDGE-01; see [ADR 0011](adr/0011-stateless-inside-dmz.md). Tunnel interfaces and the core-facing link are in INSIDE. The self zone is left at IOS defaults so IKE, ESP, GRE, BGP and OSPF to the router keep working; this is documented. Istanbul reuses the template with INSIDE/OUTSIDE only. The Chicago ASA uses two inside interfaces (one to each site switch, security 100), outside (0), object NAT and no unsolicited inbound access; its route back to the site prefers CHI-SW-01 with a floating route via CHI-SW-02.

## 10. Internal security

### Inter-VLAN policy (named ACLs inbound on the core SVIs)

| From \ To | MGMT | ENG / SALES | FIN-HR | SERVERS | VOICE | Internet |
|---|---|---|---|---|---|---|
| MGMT | ✔ | ✔ | ✔ | ✔ | ✔ | ✔ |
| ENG / SALES | ✘ | ✔ | ✘ | ✔ (DNS, HTTP, mail) | ✘ | ✔ |
| FIN-HR | ✘ | ✘ | ✔ | ✔ | ✘ | ✔ |
| VOICE | ✘ | ✘ | ✘ | CME / TFTP only | ✔ | ✘ |
| WIFI-CORP | ✘ | ✔ | ✘ | ✔ | ✘ | ✔ |
| WIFI-GUEST | ✘ | ✘ | ✘ | DHCP relay only | ✘ | ✔ |

**Branches:** users, voice and corporate Wi-Fi are trusted within their own branch. Inbound ACLs on both branch gateway switches enforce off-site policy: user and corporate Wi-Fi can use approved London services and peer user networks but cannot reach London MGMT, FIN-HR or voice; voice can reach CME and the other voice subnets but has no internet access. None of them can reach their own site's management VLAN (10.x.10.0/24) or routed transit links (10.x.0.0/24); branch devices are managed only from London MGMT. Branch corporate Wi-Fi counts as a peer user network, so the London user and corporate Wi-Fi ACLs also permit traffic to 10.x.60.0/24; without that, sessions from branch Wi-Fi to London users would be allowed outbound and silently dropped on the reply. Guest Wi-Fi allows DHCP and internet only. The ACLs are installed on both HSRP peers so policy survives a gateway failover.

### Management plane (every device)

- Hostname, `ip domain-name corp.bosphorus.test`, 2048-bit RSA keys, `ip ssh version 2`.
- VTY: `transport input ssh`, `access-class` permitting 10.10.10.0/24 only.
- `aaa new-model`: login via TACACS+, falling back to local users if the server is unreachable.
- `enable secret`; `service password-encryption` (type 7 is weak and only stops shoulder-surfing, which the ADR says).
- `login block-for 120 attempts 3 within 60`, `exec-timeout 10 0`, MOTD banner.
- `no ip http server`; `no cdp enable` on internet-facing interfaces.
- Authenticated NTP from LON-SRV-OPS-01; Syslog to the same host with millisecond timestamps.
- SNMP v2c read-only. Packet Tracer rejects an ACL on the community (matrix P0-16d), so management access is limited by the VTY `access-class` instead (production would use v3 with an ACL).
- `copy running-config tftp` backups to LON-SRV-OPS-01.

## 11. Services

| Service | Where | Details |
|---|---|---|
| DHCP | LON-SRV-CORE-01 | One pool per London VLAN via `ip helper-address 10.10.80.10` on the SVIs. AP pool sets **WLC Address** (option 43). Voice pool sets **TFTP** (option 150) to 10.10.50.10. Branches serve their own DHCP on the two site switches, with split scopes so either switch can serve alone (ADR 0009). |
| Internal DNS | LON-SRV-CORE-01 | `corp.bosphorus.test` plus a few external records (PT DNS can't forward; documented) |
| Public DNS | LON-DMZ-DNS-01, INET-DNS-01 | `www.bosphorus.test` → 203.0.113.10 |
| AAA | LON-SRV-AAA-01 | TACACS+ for device logins and RADIUS for wireless. Wired 802.1X stays a production delta: PT's PC never completes EAP (P0-06). |
| Ops | LON-SRV-OPS-01 | Syslog, NTP, TFTP |
| Intranet, mail | LON-SRV-INTRA-01 | HTTP intranet, email between a few PCs |

## 12. Wireless

Every site has the same two SSIDs:

| SSID | Security | VLAN | Access |
|---|---|---|---|
| `BSP-CORP` | WPA2-Enterprise (RADIUS on LON-SRV-AAA-01) | 60 | Internal + internet |
| `BSP-GUEST` | WPA2-PSK | 70 | Internet only (ACL on the gateway) |

**London:** LON-WLC-01 (WLC-3504), management 10.10.10.20, trunk to LON-CORE-01 carrying VLANs 10, 60, 70, with dynamic interfaces for 60 and 70. Four lightweight APs (one per access switch) on VLAN 10 ports, joining via DHCP option 43 and CAPWAP.

**Branches:** autonomous APs (AccessPoint-PT), one per SSID and one per switch, each bridging its SSID into the site's VLAN 60 or 70 ([ADR 0013](adr/0013-branch-wifi-autonomous-aps.md)). This is a representative functional layout, not production capacity sizing. Wireless traffic stays local, so branch Wi-Fi keeps working if the WAN is down; only corporate authentication depends on the RADIUS server in London. If Packet Tracer's AccessPoint-PT can't do WPA2-Enterprise, the corporate SSID falls back to WPA2-PSK and the gap is recorded.

**Clients:** laptop and smartphone on corp and a tablet on guest in London; a laptop (corp) and tablet (guest) at each branch; plus one deliberate authentication failure.

**Documented single points of failure:** one WLC with one uplink in London (production: HA pair); branch APs in production would be lightweight APs in FlexConnect mode or cloud-managed APs, managed centrally with local switching.

## 13. Voice and QoS

**Voice:** LON-VOICE-01 (2811) runs CME for the whole company: `telephony-service`, `max-ephone`, `max-dn`, `ip source-address 10.10.50.10 port 2000`, `auto assign`, eight `ephone-dn`s (1001–1008). Every site has a voice VLAN whose DHCP scope points phones at CME (option 150): three phones in London and one at each branch, registering across the VPNs and the Toronto circuits. Tests: London↔London, and London↔each branch. Production would size CME/CUCM for ~180 phones and add SRST at the branches so they can still call locally when the WAN is down.

**QoS** (depends on Phase 0):

- If PT supports MQC on the routers: marking on LAN-facing router inputs (`PM-MARK-IN`, by ACL: phone RTP → EF, SCCP → CS3, branch ↔ server traffic → AF31, guest Wi-Fi → CS1), and queuing on WAN outputs (`PM-WAN-OUT`: EF priority 10%, CS3 5%, AF31 20%, CS1 1%, default fair-queue). Applied on both edges, LON-WAN-01, IST-RTR-01 and TOR-RTR-01. Classification is by ACL because switch trust and NBAR are likely missing in PT.
- If not: the design stays in this section and the QoS configs move to `configs/_design-only/`, clearly labelled as not validated in PT.

## 14. Redundancy summary

| Failure | Recovered by | Expected result |
|---|---|---|
| LON-CORE-01 down | HSRP, RPVST+, OSPF | CORE-02 takes all gateways; a few pings lost |
| One Po1 member down (London or branch) | LACP | No loss |
| London access uplink down | RPVST+ | Blocked uplink forwards |
| ISP-A or LON-EDGE-01 down | eBGP session loss, OSPF default withdrawal, /25 withdrawal | Egress and ingress move to EDGE-02 together, preserving NAT symmetry; the DMZ gateway is moved by hand (no HSRP tracking in PT) |
| Tunnel0 down | OSPF over Tunnel1 | Istanbul reconverges via EDGE-02 |
| One branch switch down | HSRP, the other switch's uplink, split DHCP | Only that switch's desks and AP are lost; gateways, DHCP and the WAN keep working |
| One branch switch's uplink down | Routing via VLAN 900 | It keeps its gateways (no HSRP tracking in PT) and forwards over VLAN 900; traffic leaves through the other uplink |
| One Toronto circuit down | EIGRP | All traffic on the remaining circuit |
| LON-EDGE-02 or its ISP-B link down | Tracked static, floating static, ASA backup peer | Chicago's VPN moves to EDGE-01 |
| LON-WAN-01 down | — | **Accepted SPOF** for Toronto (fix: terminate the second circuit on a second London router) |
| IST-RTR-01, CHI-FW-01, TOR-RTR-01 or a branch ISP down | — | **Accepted SPOF** per branch (fix: router/firewall HA pair, second ISP or LTE backup) |
| LON-WLC-01 down | — | **Accepted SPOF** (fix: HA pair) |
| LON-VOICE-01 down | — | No calls anywhere. **Accepted SPOF** (fix: CUCM cluster, SRST at branches) |

## 15. Build phases

Save the `.pkt` and tag the repo at the end of every phase.

| # | Phase | Done when | Est. |
|---|---|---|---|
| 0 | Feature validation ([procedure](../phase0/phase0-feature-tests.md)) | Every row in the support matrix is marked | 2–3 h |
| 1 | Design docs and diagrams | Addressing has no overlaps; diagrams match the tables | 3 h |
| 2 | Place and cable | Checklist in [cabling-plan.md §5](cabling-plan.md#5-done-when-for-phase-2) | 2–3 h |
| 3 | London switching | `show spanning-tree`, `show etherchannel summary`, `show interfaces trunk` as designed | 4 h |
| 4 | London routing and FHRP | Every London VLAN gets DHCP and reaches the servers | 4 h |
| 5 | Internet edge | Inside PC reaches 192.0.2.10; INET-HOST-01 reaches www via 203.0.113.10; edge failover works | 6 h |
| 6 | Remote sites and VPNs | Full site-to-site reachability; IPsec counters increasing | 6 h |
| 7 | Security and management | ACL matrix tests pass; Syslog receiving; NTP synced | 5 h |
| 8 | Wireless, voice, QoS | APs joined; corp/guest behave as designed; calls work | 5 h |
| 9 | Testing | All tests pass or deviations documented | 5 h |
| 10 | Docs and publishing | Repo public, blog published | 8–10 h |

## 16. Test plan

Full procedures are in [test-plan.md](test-plan.md). Each test is recorded with ID, purpose, procedure, expected result, actual result and screenshot filename (`screenshots/Txx-*.png`).

| ID | Test |
|---|---|
| T01 | Inter-VLAN and inter-site policy matrix: every cell in §10 checked with ping or HTTP, with ACL hit counters on the London cores and branch switches |
| T02 | Power off LON-CORE-01 during a continuous ping; HSRP and STP take over; count lost pings |
| T03 | Shut one Po1 member; zero loss |
| T04 | Shut the ISP-A link: BGP neighbor down, default moves, PAT uses 203.0.113.2, /25s withdrawn; after moving DMZ HSRP by hand, www still reachable |
| T05 | Shut Tunnel0; Istanbul routes move to Tunnel1 (route table and traceroute before/after) |
| T06 | Chicago VPN: ISAKMP and IPsec SAs up, counters increasing, ASA shows the tunnel |
| T07 | Wrong CHAP password on TOR-RTR-01: link stays down |
| T08 | OSPF key mismatch: adjacency drops |
| T09 | Port-security violation: counter increments, log message |
| T10 | Rogue DHCP server on an access port: clients ignore it (snooping bindings as evidence) |
| T11 | Guest wireless clients (London and a branch): internet works, 10.0.0.0/8 unreachable |
| T12 | Telnet refused, SSH works; AAA server off → local fallback works |
| T13 | Wireless client with wrong credentials rejected by RADIUS |
| T14 | Calls London↔London and London↔each branch |
| T15 | Syslog messages on LON-SRV-OPS-01; NTP synced on all devices |
| T16 | INET-HOST-01 tries to reach the internal network directly: blocked by ZBF |
| T17 | LON-DMZ-WEB-01 tries to reach the internal network: blocked |
| T18 | Cable LON-ROGUE-SW to LON-ACC-01 Fa0/24: BPDU guard err-disables the port, root unchanged, `%SPANTREE-2-BLOCK_BPDUGUARD` in Syslog |
| T19 | Same, with BPDU guard temporarily removed on that port: root guard on the core blocks the superior BPDUs, root still unchanged |
| T20 | Loop guard: not run, Packet Tracer has no loop guard (matrix P0-02b); kept as the production control |
| T21 | Power off a branch switch: the other switch keeps gateways, DHCP and the WAN up for its half of the office |
| T22 | Shut one Toronto circuit: EIGRP moves all traffic to the other, calls and pings continue |
| T23 | Shut LON-EDGE-02's ISP-B link: Chicago's VPN re-establishes to EDGE-01 and London routes Chicago traffic there |
| T24 | DAI shows active with counters on the access switches (PT does not drop invalid ARP, matrix P0-05a) |
| T25 | BGP advertisements: EDGE-01 exports only the two /25s, EDGE-02 only the /24; the internet prefers the /25s |
| T26 | Congest one Toronto circuit during a call: drops fall in class-default, voice class is protected (demonstration, not measurement) |

Simulation-mode captures (ZBF drop, IPsec encapsulation, HSRP hellos) go in the blog.

## 17. Documentation deliverables

- **README:** summary and hero diagram, features table, diagrams, addressing summary, how to open, test results, PT limitations, repo map.
- **Diagrams** ([`diagrams/`](../diagrams/), Mermaid so GitHub renders them): site overview; L2 VLANs, trunks, STP roots and blocked ports; L3 addressing, OSPF areas, EIGRP domain, BGP ASes; security zones; VPN overlay; wireless.
- **Blog** (drafted locally in `blog/`, git-ignored): "Building a four-site enterprise network in Packet Tracer, and what the simulator can't do." Sections: why rebuild without hardware; the scenario; three design problems (asymmetric NAT with two ISPs, the IPsec NAT exemption, integrating the EIGRP site); attacking my own Layer 2; break-it testing; the support matrix; what production would change (IKEv2/VTI, SD-WAN, 802.1X everywhere, SNMPv3, CoPP, VRRP).

## 18. Packet Tracer gotchas

- 3650 switches ship without a power supply: add AC-POWER-SUPPLY.
- Crypto and ZBF need the security license: on ISR4331 `license boot level securityk9`, on ISR2911 `license boot module c2900 technology-package securityk9`; then `write` and `reload`.
- Phones and APs on 2960-24TT need a power adapter (no PoE).
- Unsupported commands return `% Invalid input`: log them in the support matrix.
- Large topologies are slow: disable animations, use Fast Forward Time for convergence, save after every phase.
- `.pkt` files aren't backward-compatible: pin the PT build in the README.
- `copy running-config startup-config` on every device before saving the `.pkt`.
