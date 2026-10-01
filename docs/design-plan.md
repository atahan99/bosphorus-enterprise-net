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
| GRE over IPsec, PPP/CHAP | OT zone using PT 9 industrial devices |
| Stateful firewall (CBAC + ACLs), NAT/PAT, DMZ | |
| Wi-Fi at every site: WLC + lightweight APs in London, autonomous APs at branches | |
| CME voice for all four sites, WAN QoS | |
| AAA, SSH, NTP, Syslog, SNMP, TFTP backups | |

## 3. Scenario

A fictional professional-services company with **182 staff** across four sites and its own public /24.

| Site | Staff | Role | Why it looks the way it does |
|---|---|---|---|
| **London (LON)** | 80 | HQ | Dual-homed to two ISPs with its own address space, which is what makes **BGP** necessary. Redundant collapsed core, OSPF area 0, the server room, DMZ and call control for the whole company. |
| **Istanbul (IST)** | 30 | Regional office | Runs a routing protocol over its VPN: **OSPF area 1** over GRE/IPsec with a backup tunnel, extended to its two site switches. |
| **Chicago (CHI)** | 36 | Office with its own internet breakout | Same pattern as Istanbul: **OSPF area 2** over GRE/IPsec to both London edges, extended to its two site switches. (First designed behind an ASA; replaced by a router, [ADR 0007](adr/0007-chicago-router-not-asa.md).) |
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
| Chicago | 36 | 2 × 3650-24PS (pair) | Each: Po1 (2), router link (1), AP (1), 20 desk-capable ports; printer on SW-02 | 36 of 39 | 3 |
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
| 1 | Istanbul: both GRE tunnels, IST-RTR-01 and IST-SW-01/02. EDGE-01 and EDGE-02 are the ABRs |
| 2 | Chicago: both GRE tunnels, CHI-RTR-01 and CHI-SW-01/02. EDGE-01 and EDGE-02 are the ABRs |

- `passive-interface default`, enabling only real adjacencies, except on routers with GRE tunnels: Packet Tracer can't name tunnels in `no passive-interface` (B-41), so those list their passive interfaces instead.
- `ip ospf network point-to-point` on routed /30s (tunnels are point-to-point by default; PT accepts no OSPF commands on them, B-40).
- `auto-cost reference-bandwidth 10000`.
- MD5 authentication in every area.
- Router IDs from Loopback0.
- Summarisation at the ABRs: `area 0 range 10.10.0.0 255.255.0.0`, so each branch sees one London route. The branches are **not** summarised into area 0: with two branch areas on the same ABRs, Packet Tracer generated only one of the two ranges, a different one on each load (B-48), so London carries the ~10 subnets of each branch instead. Production would add `area 1 range 10.20.0.0/16` and `area 2 range 10.30.0.0/16`.
- Tunnel costs can't be set in Packet Tracer (B-40). Both tunnels of a branch have the same cost; EDGE-02's cost-1000 core link makes EDGE-01 the primary path in both directions, and the EDGE-02 tunnel is a hot standby.

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

### Egress and ingress symmetry

- **Outbound:** only EDGE-01 runs `default-information originate` into OSPF (without `always`). Packet Tracer load-shares two equal E2 defaults and can't set their metric (matrix B-25), so LON-CORE-01/02 instead carry a floating static default (AD 250) towards EDGE-02.
- **Inbound:** longest match. The internet prefers EDGE-01's /25s via ISP-A over EDGE-02's /24 via ISP-B.
- **Why active/standby rather than load-sharing:** each edge translates to its own PAT address. Traffic that left via EDGE-01 and returned via EDGE-02 would hit a router with no NAT entry and break.
- **Failure behaviour:** if the ISP-A link or EDGE-01 fails, EDGE-01 loses its BGP default, stops originating the OSPF default and its /25s are withdrawn. The cores fall back to their floating statics, so both directions move to EDGE-02 together. Failback needs a manual step in Packet Tracer (B-37). The DMZ gateway (HSRP) can't track the ISP link in Packet Tracer, so it is moved by hand during the test. See [ADR 0001](adr/0001-active-standby-internet-edge.md).

## 8. WAN and VPN

| Link | Design |
|---|---|
| **IST ↔ LON, CHI ↔ LON** | GRE over IPsec, same pattern for both branches (Istanbul area 1, Chicago area 2). GRE sourced from the public interfaces (EDGE-01 198.51.100.2, EDGE-02 198.51.100.6, IST 198.51.100.10, CHI 198.51.100.14). Crypto map on the physical outside interface matching `permit gre host X host Y`. IKEv1 PSK, AES-256 with SHA-1 and DH group 5, ESP `esp-aes 256 esp-sha-hmac` in tunnel mode: the strongest Packet Tracer accepts (matrix P0-14; production would use SHA-256 and group 14+). OSPF runs inside the tunnels, which is the reason for GRE. Packet Tracer's tunnel interfaces take no OSPF settings (B-40); EDGE-01 is the preferred path because EDGE-02's core link costs 1000. |
| **TOR ↔ LON** | Serial, `encapsulation ppp`, `ppp authentication chap`, per-peer `username` entries. LON-WAN-01 is DCE and sets `clock rate`. |

Both branches keep working if either London edge or its ISP fails: OSPF moves to the other tunnel (tests T05, T23). Each branch router and its single ISP remain single points of failure (production: router pair, second ISP or LTE). Chicago was first designed around an ASA 5506-X; Packet Tracer's ASA could not complete IPsec with the IOS edges, so it was replaced by an ISR4331 ([ADR 0007](adr/0007-chicago-router-not-asa.md)).

## 9. Internet edge: NAT, firewall, DMZ

**NAT (each edge):** PAT for 10.0.0.0/8 to that edge's own 203.0.113.x, excluding site-to-site traffic; static NAT for the DMZ web and DNS servers, configured identically on both edges.

**Firewall (each edge): CBAC + ACLs.** Phase 5 built this as a zone-based firewall; Phase 6 replaced it because Packet Tracer can't put GRE tunnels in a zone and ZBF then drops all branch traffic ([ADR 0015](adr/0015-cbac-not-zbf.md)). The policy is unchanged:

| Traffic | Policy | Where |
|---|---|---|
| Inside → internet | Stateful (inspect TCP, UDP, ICMP) | `ip inspect CBAC-OUT out` on Gi0/0/0 |
| Internet → inside | Drop; only BGP from the ISP and IKE/ESP/GRE from the VPN peers reach the router | `ACL-OUTSIDE-IN` in on Gi0/0/0 |
| Inside → DMZ | Pass (stateless): web and DNS to the two servers, SSH to the DMZ switch and ping from MGMT | `ACL-TO-DMZ` out on Gi0/0/2 |
| Internet → DMZ | Web and DNS to the static NAT hosts only, stateful | `ACL-OUTSIDE-IN` + `ACL-TO-DMZ`, inspected out on Gi0/0/2 |
| DMZ → inside | Replies to the flows above only (TCP `established`, DNS responses, echo-replies); everything else dropped | `ACL-DMZ-IN` in on Gi0/0/2 |
| DMZ → internet | Web and DNS, stateful | `ACL-DMZ-IN`, inspected out on Gi0/0/0 |

Inside ↔ DMZ is stateless because the DMZ gateway can move to EDGE-02 while internal traffic still enters the DMZ through EDGE-01; see [ADR 0011](adr/0011-stateless-inside-dmz.md). The core link and the GRE tunnels are unfiltered (the old INSIDE zone). `ACL-DMZ-IN` also permits HSRP between the edges. IST-RTR-01 and CHI-RTR-01 use the same model: inspect outbound on the ISP interface, and inbound only IKE, ESP and GRE from the two London edges.

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
| `BSP-CORP` | WPA2-Enterprise (RADIUS on LON-SRV-AAA-01) in London; WPA2-PSK at the branches | 60 | Internal + internet |
| `BSP-GUEST` | WPA2-PSK | 70 | Internet only (ACL on the gateway) |

**London:** LON-WLC-01 (WLC-3504), management 10.10.10.20, trunk to LON-CORE-01 carrying VLANs 10, 60, 70, with dynamic interfaces for 60 and 70 (as built, PT can't trunk the WLC uplink, so London Wi-Fi clients land in VLAN 10: [matrix B-53](pt-support-matrix.md)). Four lightweight APs (one per access switch) on VLAN 10 ports, joining via DHCP option 43 and CAPWAP.

**Branches:** autonomous APs (AccessPoint-PT), one per SSID and one per switch, each bridging its SSID into the site's VLAN 60 or 70 ([ADR 0013](adr/0013-branch-wifi-autonomous-aps.md)). This is a representative functional layout, not production capacity sizing. Wireless traffic stays local, so branch Wi-Fi keeps working if the WAN is down. AccessPoint-PT has no RADIUS option, so the branch corporate SSID uses WPA2-PSK (matrix B-14).

**Clients:** laptop and smartphone on corp and a tablet on guest in London; a laptop (corp) and tablet (guest) at each branch; plus one deliberate authentication failure.

**Documented single points of failure:** one WLC with one uplink in London (production: HA pair); branch APs in production would be lightweight APs in FlexConnect mode or cloud-managed APs, managed centrally with local switching.

## 13. Voice and QoS

**Voice:** LON-VOICE-01 (2811) runs CME for the whole company: `telephony-service`, `max-ephone`, `max-dn`, `ip source-address 10.10.50.10 port 2000`, `auto assign`, eight `ephone-dn`s (1001–1008). Every site has a voice VLAN whose DHCP scope points phones at CME (option 150): three phones in London and one at each branch, registering across the VPNs and the Toronto circuits (as built, the London lobby phone doesn't complete DHCP: matrix B-52). Tests: London↔London, and London↔each branch. Production would size CME/CUCM for ~180 phones and add SRST at the branches so they can still call locally when the WAN is down.

**QoS** (MQC, validated in Phase 0 as P0-17):

- Marking on LAN-facing router inputs (`PM-MARK-IN`, by ACL: phone RTP → EF, SCCP → CS3, branch ↔ server traffic → AF31, guest Wi-Fi → CS1), and queuing on WAN outputs (`PM-WAN-OUT`: EF priority 10%, CS3 5%, AF31 20%, CS1 1%, default fair-queue). Applied on both edges, LON-WAN-01 and all three branch routers. Classification is by ACL because switch trust and NBAR are missing in PT.

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
| LON-EDGE-01 or LON-EDGE-02 down | OSPF over the other tunnel | Istanbul and Chicago reconverge via the remaining edge |
| LON-WAN-01 down | none | **Accepted SPOF** for Toronto (fix: terminate the second circuit on a second London router) |
| IST-RTR-01, CHI-RTR-01, TOR-RTR-01 or a branch ISP down | none | **Accepted SPOF** per branch (fix: router/firewall HA pair, second ISP or LTE backup) |
| LON-WLC-01 down | none | **Accepted SPOF** (fix: HA pair) |
| LON-VOICE-01 down | none | No calls anywhere. **Accepted SPOF** (fix: CUCM cluster, SRST at branches) |

## 15. Build, test and documentation

The lab was built in ten phases, each ending with a saved `.pkt` checkpoint and a Git tag: [build-guide.md](build-guide.md) has the steps and verification commands. The 26 tests, with procedures and results, are in [test-plan.md](test-plan.md). Everything Packet Tracer couldn't do, and the fallback used, is in [pt-support-matrix.md](pt-support-matrix.md).
