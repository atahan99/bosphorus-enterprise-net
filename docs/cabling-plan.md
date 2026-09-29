# Device and cabling plan

Everything needed for Phase 2: which device to drag in, which modules to add, and exactly which port connects to which. Addresses are in [addressing-plan.md](addressing-plan.md); sizing is explained in [design-plan.md §4](design-plan.md#4-sizing).

Port names below are what Packet Tracer 9 shows for these models. The MCP catalog can expose different names for some models. Run the build guide's throwaway MCP preflight first; if a name differs, keep the physical position and record the accepted MCP-to-GUI mapping here before building the production file.

## 1. Devices and modules

Power off a device (Physical tab) before adding modules, then power it back on.

### Network devices

| Hostname | PT model | Modules / physical prep |
|---|---|---|
| LON-EDGE-01, LON-EDGE-02 | ISR4331 | None. `securityk9` licence (build guide Phase 2) |
| LON-CORE-01, LON-CORE-02 | 3650-24PS | **AC-POWER-SUPPLY** |
| LON-WAN-01 | ISR4331 | **NIM-2T** in slot 0/1 → Serial0/1/0–1 |
| LON-ACC-01 … LON-ACC-04 | 2960-24TT | None |
| LON-SRV-SW01 | 2960-24TT | None |
| LON-DMZ-SW01 | 2960-24TT | None |
| LON-ROGUE-SW | 2960-24TT | None. Left **disconnected** except during tests T18–T19 |
| LON-WLC-01 | WLC-3504 | None |
| IST-RTR-01 | ISR4331 | None. `securityk9` licence |
| IST-SW-01, IST-SW-02 | 3650-24PS | **AC-POWER-SUPPLY** |
| CHI-FW-01 | ASA 5506-X | None |
| CHI-SW-01, CHI-SW-02 | 3650-24PS | **AC-POWER-SUPPLY** |
| TOR-RTR-01 | ISR2911 | **HWIC-2T** in GUI slot 0 → Serial0/0/0–1. With MCP, install it using slot `0/0` |
| TOR-SW-01, TOR-SW-02 | 3650-24PS | **AC-POWER-SUPPLY** |
| ISP-A, ISP-B, INET-CORE | Router-PT | None (FastEthernet0/0, 1/0, 4/0, 5/0). If Phase 0 shows Router-PT lacks a BGP feature, swap for ISR4331 + NIM-ES2-4 |
| INET-SW | 2960-24TT | None |

### Wireless

| Hostname | PT model | Prep |
|---|---|---|
| LON-AP-01 … LON-AP-04 | LAP (3702i) | **Power adapter** (no PoE on 2960-24TT) |
| IST-AP-01, CHI-AP-01, TOR-AP-01 | AccessPoint-PT | SSID `BSP-CORP`, on SW-01 (see [wireless-and-endpoints.md](../configs/servers/wireless-and-endpoints.md)) |
| IST-AP-02, CHI-AP-02, TOR-AP-02 | AccessPoint-PT | SSID `BSP-GUEST`, on SW-02 |

### Servers and endpoints

The lab places a **representative sample** of endpoints: enough to test every policy, not one per employee. Every desk port for the full headcount is still configured on the switches.

| Device | PT model | Prep |
|---|---|---|
| LON-SRV-CORE-01, -AAA-01, -OPS-01, -INTRA-01 | Server-PT | None |
| LON-DMZ-WEB-01, LON-DMZ-DNS-01 | Server-PT | None |
| INET-WEB-01, INET-DNS-01 | Server-PT | None |
| LON-VOICE-01 | 2811 | None |
| LON-PH-01 … 03 | IP Phone 7960 | **Power adapter** (London access switches have no PoE) |
| IST-PH-01, CHI-PH-01, TOR-PH-01 | IP Phone 7960 | None: powered by the 3650's PoE (add a power adapter only if it doesn't power up) |
| LON-PC-ENG-01 … 03, LON-PC-SALES-01 … 02, LON-PC-FIN-01 … 02, LON-MGMT-PC-01 | PC-PT | None |
| IST-PC-01 … 02, CHI-PC-01 … 02, TOR-PC-01 … 02, INET-HOST-01 | PC-PT | None |
| LON-PRN-01, LON-PRN-02 | Printer-PT | None (branch printer ports are configured but not populated) |
| LON-LAPTOP-01, IST-LAPTOP-01, CHI-LAPTOP-01, TOR-LAPTOP-01 | Laptop-PT | Swap NIC for **WPC300N** |
| LON-SMARTPHONE-01 | Smartphone-PT | None |
| LON-TABLET-01, IST-TABLET-01, CHI-TABLET-01, TOR-TABLET-01 | TabletPC-PT | None |

**Count:** 9 routers/firewalls (3 of them the simulated internet), 8 L3 switches (2 London cores + 3 branch pairs), 7 connected L2 switches plus 1 disconnected rogue switch, 1 WLC, 10 APs, 8 servers, 1 call manager, ~35 endpoints.

## 2. Infrastructure links

Cable rule: **Copper Cross-Over** between like devices (router↔router, switch↔switch, router↔ASA); **Copper Straight-Through** between unlike devices (router↔switch, switch↔host, switch↔AP). For serial, use **Serial DCE** and click the DCE end first.

### London

| # | Device A | Port A | Device B | Port B | Cable | Notes |
|---|---|---|---|---|---|---|
| L1 | LON-EDGE-01 | Gi0/0/0 | ISP-A | Fa0/0 | Cross | Outside |
| L2 | LON-EDGE-02 | Gi0/0/0 | ISP-B | Fa0/0 | Cross | Outside |
| L3 | LON-EDGE-01 | Gi0/0/1 | LON-CORE-01 | Gi1/0/20 | Straight | Routed /30 |
| L4 | LON-EDGE-02 | Gi0/0/1 | LON-CORE-02 | Gi1/0/20 | Straight | Routed /30 |
| L5 | LON-EDGE-01 | Gi0/0/2 | LON-DMZ-SW01 | Gi0/1 | Straight | DMZ |
| L6 | LON-EDGE-02 | Gi0/0/2 | LON-DMZ-SW01 | Gi0/2 | Straight | DMZ |
| L7 | LON-CORE-01 | Gi1/0/23 | LON-CORE-02 | Gi1/0/23 | Cross | Po1 member |
| L8 | LON-CORE-01 | Gi1/0/24 | LON-CORE-02 | Gi1/0/24 | Cross | Po1 member |
| L9 | LON-CORE-01 | Gi1/0/1 | LON-ACC-01 | Gi0/1 | Cross | Trunk |
| L10 | LON-CORE-02 | Gi1/0/1 | LON-ACC-01 | Gi0/2 | Cross | Trunk |
| L11 | LON-CORE-01 | Gi1/0/2 | LON-ACC-02 | Gi0/1 | Cross | Trunk |
| L12 | LON-CORE-02 | Gi1/0/2 | LON-ACC-02 | Gi0/2 | Cross | Trunk |
| L13 | LON-CORE-01 | Gi1/0/3 | LON-ACC-03 | Gi0/1 | Cross | Trunk |
| L14 | LON-CORE-02 | Gi1/0/3 | LON-ACC-03 | Gi0/2 | Cross | Trunk |
| L15 | LON-CORE-01 | Gi1/0/4 | LON-ACC-04 | Gi0/1 | Cross | Trunk |
| L16 | LON-CORE-02 | Gi1/0/4 | LON-ACC-04 | Gi0/2 | Cross | Trunk |
| L17 | LON-CORE-01 | Gi1/0/5 | LON-SRV-SW01 | Gi0/1 | Cross | Trunk |
| L18 | LON-CORE-02 | Gi1/0/5 | LON-SRV-SW01 | Gi0/2 | Cross | Trunk |
| L19 | LON-CORE-01 | Gi1/0/21 | LON-WAN-01 | Gi0/0/1 | Straight | Routed /30 |
| L20 | LON-CORE-02 | Gi1/0/21 | LON-WAN-01 | Gi0/0/2 | Straight | Routed /30 |
| L21 | LON-CORE-01 | Gi1/0/10 | LON-WLC-01 | Gi1 (port 1) | Straight | Trunk (VLAN 10, 60, 70) |
| L22 | LON-WAN-01 | Se0/1/0 | TOR-RTR-01 | Se0/0/0 | Serial DCE | Circuit 1. Click LON-WAN-01 first (DCE) |
| L23 | LON-WAN-01 | Se0/1/1 | TOR-RTR-01 | Se0/0/1 | Serial DCE | Circuit 2. Click LON-WAN-01 first (DCE) |

### Branches

Every branch pair uses the same ports: Gi1/0/22 uplink, Gi1/0/23–24 the LACP bundle between the two switches.

| # | Device A | Port A | Device B | Port B | Cable | Notes |
|---|---|---|---|---|---|---|
| B1 | IST-RTR-01 | Gi0/0/0 | ISP-A | Fa1/0 | Cross | Outside |
| B2 | IST-RTR-01 | Gi0/0/1 | IST-SW-01 | Gi1/0/22 | Straight | Routed /30 |
| B3 | IST-RTR-01 | Gi0/0/2 | IST-SW-02 | Gi1/0/22 | Straight | Routed /30 |
| B4 | IST-SW-01 | Gi1/0/23 | IST-SW-02 | Gi1/0/23 | Cross | Po1 member |
| B5 | IST-SW-01 | Gi1/0/24 | IST-SW-02 | Gi1/0/24 | Cross | Po1 member |
| B6 | CHI-FW-01 | Gi1/1 | ISP-B | Fa1/0 | Cross | Outside |
| B7 | CHI-FW-01 | Gi1/2 | CHI-SW-01 | Gi1/0/22 | Straight | Routed /30 (inside) |
| B8 | CHI-FW-01 | Gi1/3 | CHI-SW-02 | Gi1/0/22 | Straight | Routed /30 (inside2) |
| B9 | CHI-SW-01 | Gi1/0/23 | CHI-SW-02 | Gi1/0/23 | Cross | Po1 member |
| B10 | CHI-SW-01 | Gi1/0/24 | CHI-SW-02 | Gi1/0/24 | Cross | Po1 member |
| B11 | TOR-RTR-01 | Gi0/0 | TOR-SW-01 | Gi1/0/22 | Straight | Routed /30 |
| B12 | TOR-RTR-01 | Gi0/1 | TOR-SW-02 | Gi1/0/22 | Straight | Routed /30 |
| B13 | TOR-SW-01 | Gi1/0/23 | TOR-SW-02 | Gi1/0/23 | Cross | Po1 member |
| B14 | TOR-SW-01 | Gi1/0/24 | TOR-SW-02 | Gi1/0/24 | Cross | Po1 member |

### Simulated internet

| # | Device A | Port A | Device B | Port B | Cable |
|---|---|---|---|---|---|
| I1 | ISP-A | Fa4/0 | ISP-B | Fa4/0 | Cross |
| I2 | ISP-A | Fa5/0 | INET-CORE | Fa0/0 | Cross |
| I3 | ISP-B | Fa5/0 | INET-CORE | Fa1/0 | Cross |
| I4 | INET-CORE | Fa4/0 | INET-SW | Fa0/1 | Straight |

## 3. Access-port plan

Each desk has one switch port: the PC plugs into the phone, the phone into the port (data VLAN + voice VLAN 50). Ports not listed are VLAN 998 and shut down.

### London (80 desks)

Every London access switch uses the same layout: Fa0/1–20 desks, Fa0/21 printer (where present), Fa0/23 AP, the rest spare.

| Switch | Location | Desk ports | Other ports | Populated in PT |
|---|---|---|---|---|
| LON-ACC-01 | Floor 1 | Fa0/1–20: ENG (20) | Fa0/23 LON-AP-01; Fa0/24 spare desk port (VLAN 20, used for tests T10, T18, T19) | Fa0/1 LON-PC-ENG-01; Fa0/2 LON-PH-01 with LON-PC-ENG-02 on its PC port |
| LON-ACC-02 | Floor 1 | Fa0/1–20: ENG (20) | Fa0/23 LON-AP-02 | Fa0/1 LON-PC-ENG-03 |
| LON-ACC-03 | Floor 2 | Fa0/1–20: SALES (20) | Fa0/21 LON-PRN-01; Fa0/23 LON-AP-03 | Fa0/1 LON-PC-SALES-01; Fa0/2 LON-PH-02 |
| LON-ACC-04 | Floor 2 | Fa0/1–5: SALES (5); Fa0/6–20: FIN-HR (15) | Fa0/21 LON-PRN-02; Fa0/23 LON-AP-04 | Fa0/1 LON-PC-SALES-02; Fa0/6 LON-PC-FIN-01; Fa0/7 LON-PC-FIN-02 |
| LON-SRV-SW01 | Server room | — | Fa0/1–4 servers (VLAN 80); Fa0/5 LON-VOICE-01 (VLAN 50); Fa0/6 LON-PH-03 lobby phone (VLAN 50); Fa0/20 LON-MGMT-PC-01 (VLAN 10) | All |
| LON-DMZ-SW01 | Server room | — | Fa0/1 LON-DMZ-WEB-01; Fa0/2 LON-DMZ-DNS-01 (VLAN 100) | All |

Totals: 40 ENG + 25 SALES + 15 FIN-HR = **80 desk ports** (96 FE ports, 10 spare), 4 APs, 2 printers.

### Branches (switch pairs)

| Switch | Desk ports | Gi1/0/20 | Gi1/0/21 | Populated in PT |
|---|---|---|---|---|
| IST-SW-01 | Gi1/0/1–15 (15) | spare | IST-AP-01 (VLAN 60, `BSP-CORP`) | Gi1/0/1 IST-PC-01; Gi1/0/2 IST-PH-01; AP |
| IST-SW-02 | Gi1/0/1–15 (15) | IST printer | IST-AP-02 (VLAN 70, `BSP-GUEST`) | Gi1/0/1 IST-PC-02; AP |
| CHI-SW-01 / TOR-SW-01 | Gi1/0/1–18 (18) | spare | AP-01 (VLAN 60, `BSP-CORP`) | Gi1/0/1 PC-01; Gi1/0/2 PH-01; AP |
| CHI-SW-02 / TOR-SW-02 | Gi1/0/1–18 (18) | printer | AP-02 (VLAN 70, `BSP-GUEST`) | Gi1/0/1 PC-02; AP |

Totals: Istanbul 30 desks (9 spare), Chicago and Toronto 36 desks each (3 spare). Desks and APs are split across the pair, so losing one switch takes out only its half.

### Internet

| Switch | Port | Device |
|---|---|---|
| INET-SW | Fa0/2 | INET-WEB-01 |
| | Fa0/3 | INET-DNS-01 |
| | Fa0/4 | INET-HOST-01 |

Wireless clients have no cables: see [wireless-and-endpoints.md](../configs/servers/wireless-and-endpoints.md).

## 4. Workspace layout

Suggested placement so screenshots read left to right:

- **Top band:** INET-CORE, ISP-A, ISP-B and the internet servers.
- **Centre:** London. Edge pair and DMZ at the top; the core pair below; the four access switches and the services switch in a row underneath, with APs, phones and PCs below them.
- **Right:** Istanbul under ISP-A, Chicago under ISP-B, each switch pair side by side.
- **Bottom left:** LON-WAN-01 and Toronto.
- Use PT's rectangle/note tools to draw a labelled box around each site. Use **Logical workspace clusters** (one per site) if the canvas gets crowded.

## 5. Done-when for Phase 2

- [ ] Every link in §2 is green on both ends once interfaces are enabled.
- [ ] Hostnames set on all devices.
- [ ] Every infrastructure interface has a description in the form `TO-<peer>-<peer-port>`.
- [ ] `show cdp neighbors` on each router and switch matches this file.
