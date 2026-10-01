# Wireless, phones and endpoints (GUI)

GUI-only configuration for the WLC, access points, wireless clients, IP phones and PCs. Port positions are in [cabling-plan.md §3](../../docs/cabling-plan.md#3-access-port-plan).

---

## London: LON-WLC-01 (WLC-3504)

### 1. Management interface (Config tab)

| Field | Value |
|---|---|
| IP address | 10.10.10.20 |
| Subnet mask | 255.255.255.0 |
| Default gateway | 10.10.10.1 |
| VLAN (management) | `0` (untagged) |

PT treats the WLC's GigabitEthernet1 as an access port, not a trunk, so it can't tag anything. Management stays untagged and LON-CORE-01 Gi1/0/10 uses native VLAN 10 (matrix B-11). Side effect: the dynamic interfaces below never get onto the wire, and every London Wi-Fi client lands in VLAN 10 (B-53). Setting the management VLAN to 10 makes the WLC unreachable; don't.

### 2. Web GUI

From **LON-MGMT-PC-01 → Desktop → Web Browser**, open `http://10.10.10.20` (try `https://` if that fails). On first access, create the admin account: `wlcadmin` / `Lab-Wlc-1`. The setup wizard asks for the system name, NTP `10.10.80.12`, the management address above, and one employee network: create `BSP-CORP` there (WPA2 Enterprise, RADIUS `10.10.80.11` / `Lab-Radius-1`, VLAN left on Management VLAN, guest toggle off), then add the rest below. **Type** passwords: pasting into the PT browser crashed PT on macOS twice (see the build guide).

**Controller → Interfaces → New** (two dynamic interfaces):

| Interface name | VLAN | IP address | Mask | Gateway | Primary DHCP server |
|---|---|---|---|---|---|
| wifi-corp | 60 | 10.10.60.10 | 255.255.255.0 | 10.10.60.1 | 10.10.80.10 |
| wifi-guest | 70 | 10.10.70.10 | 255.255.255.0 | 10.10.70.1 | 10.10.80.10 |

**Security → AAA → RADIUS → Authentication** (the wizard creates this entry; check it):

| Field | Value |
|---|---|
| Server IP | 10.10.80.11 |
| Shared secret | Lab-Radius-1 |
| Port | 1812 |
| Status | Enabled |

**WLANs → Create New → Go** (twice):

| Setting | BSP-CORP | BSP-GUEST |
|---|---|---|
| Profile name / SSID | BSP-CORP | BSP-GUEST |
| General → Status | Enabled | Enabled |
| General → Interface | wifi-corp | wifi-guest |
| Security → Layer 2 | WPA2, AES, **802.1X** | WPA2, AES, **PSK** |
| PSK | — | `Lab-Guest-Wifi-1` |
| Security → AAA Servers | 10.10.80.11 | — |

Apply and save the configuration (**Save Configuration** at the top of the GUI).

Also set **LON-SRV-AAA-01 → Services → AAA → Radius Port** to `1812` (default 1645), otherwise the WLC's RADIUS requests go unanswered and WPA2-Enterprise clients end up on 169.254.x.x.

### 3. London access points

LON-AP-01 … LON-AP-04 (one per access switch, port Fa0/23) have a power adapter from Phase 2. **Config → GigabitEthernet0 → IP Configuration → DHCP**: the 3702i ships set to Static with no address and never asks for one. They then take a DHCP address in VLAN 10, learn the WLC address from option 43 (the `WLC Address` field of the `VLAN10-MGMT` pool) and join.

**Check:** WLC GUI → Wireless → All APs lists all four.

---

## Branch access points (AccessPoint-PT)

Each branch has two autonomous APs, one per SSID and one on each switch of the pair ([ADR 0013](../../docs/adr/0013-branch-wifi-autonomous-aps.md)). They're bridges: the SSID lands in whatever VLAN the switch port is in. The 3650s supply PoE.

| AP | Switch port (VLAN) | Config → Port 1 |
|---|---|---|
| IST-AP-01, CHI-AP-01, TOR-AP-01 | SW-01 Gi1/0/21 (VLAN 60) | SSID `BSP-CORP`; **WPA2-PSK**, key `Lab-Corp-Wifi-1`, AES, channel 1 |
| IST-AP-02, CHI-AP-02, TOR-AP-02 | SW-02 Gi1/0/21 (VLAN 70) | SSID `BSP-GUEST`; **WPA2-PSK**, key `Lab-Guest-Wifi-1`, AES, channel 6 |

AccessPoint-PT only offers Disabled, WEP, WPA-PSK and WPA2-PSK, so branch `BSP-CORP` is PSK (matrix B-14). London's corporate SSID stays WPA2-Enterprise.

## Physical view: one city per site

Wi-Fi range in PT is physical distance. With every device in the default office, branch clients joined London's or another branch's APs. The Physical view therefore has five cities on the Intercity map, each with an office building and a wiring closet:

| City | Building | Wiring closet | Office floor |
|---|---|---|---|
| London | London HQ | all LON network gear (existing racks) | LON PCs, phones, printers, Wi-Fi clients |
| Istanbul / Chicago / Toronto | `<City> Office` | `<SITE>-RTR-01`, `-SW-01`, `-SW-02` | PCs, phone, both APs, laptop, tablet |
| Internet | ISP Datacenter | ISP-A, ISP-B, INET-CORE, INET-SW, INET-WEB-01, INET-DNS-01 | INET-HOST-01 |

Cities, buildings and closets are created and named in the GUI (New City / New Building / New Closet); devices can be moved with **Move Object** or by script (`moveOutOfCurrentObject()` up to Intercity, then `moveIntoObject("<container name>")` level by level). Links stay up when devices move between cities.

---

## Wireless clients

| Device | Prep | Wireless settings | Expected address |
|---|---|---|---|
| LON-LAPTOP-01 | Physical tab: power off, remove the Ethernet NIC, insert **WPC300N**, power on | `BSP-CORP`, **WPA2** (this is Enterprise in PT: User ID / Password fields), `wifi-user1` / `Lab-Wifi-1`, AES | 10.10.10.100+ (B-53; design 10.10.60.x) |
| LON-SMARTPHONE-01 | None | `BSP-CORP`, WPA2, `wifi-user2` / `Lab-Wifi-2` | 10.10.10.100+ (B-53) |
| LON-TABLET-01 | None | `BSP-GUEST`, WPA2-PSK, `Lab-Guest-Wifi-1` | 10.10.10.100+ (B-53; design 10.10.70.x) |
| IST-LAPTOP-01, CHI-LAPTOP-01, TOR-LAPTOP-01 | Swap NIC for **WPC300N** | `BSP-CORP`, WPA2-PSK, `Lab-Corp-Wifi-1`, AES (B-14) | 10.x.60.20+ |
| IST-TABLET-01, CHI-TABLET-01, TOR-TABLET-01 | None | `BSP-GUEST`, WPA2-PSK, `Lab-Guest-Wifi-1` | 10.x.70.100+, DNS 192.0.2.53 |

On every client, first tick **Config → Wireless0 → Port Status: On** (the radios were switched off in Phase 2, matrix B-32). Branch clients only join their own site's APs once each site is in its own city in the Physical view (above).

---

## IP phones (7960)

LON-PH-01, LON-PH-02, LON-PH-03: Physical tab → drag the **power adapter** onto the phone (London access switches have no PoE). IST-PH-01, CHI-PH-01, TOR-PH-01 are powered by the 3650's PoE; add an adapter only if one doesn't power up. Nothing else to configure: the phone learns the voice VLAN from the switch (CDP), gets an address and option 150 from its site's DHCP, and registers with LON-VOICE-01.

LON-PC-ENG-02 plugs into LON-PH-01's **PC port** (not into the switch).

**Check:** each phone's screen shows a line number. On LON-VOICE-01, `show ephone` lists five phones: TOR-PH-01 1001, CHI-PH-01 1002, LON-PH-02 1003, LON-PH-01 1004, IST-PH-01 1005 (`show ephone registered` prints nothing in PT). The lobby phone LON-PH-03 doesn't complete DHCP and stays unregistered (matrix B-52).

---

## PCs and printers

| Device | IP configuration | Extra |
|---|---|---|
| LON-PC-ENG-01 … 03, LON-PC-SALES-01 … 02, LON-PC-FIN-01 … 02 | DHCP | Email client on ENG-01 (`alice`) and SALES-01 (`bob`): server `mail.corp.bosphorus.test`, domain `corp.bosphorus.test` |
| LON-MGMT-PC-01 | 10.10.10.50 / 24, gw 10.10.10.1, DNS 10.10.80.10 | Jump host for SSH (Desktop → Command Prompt → `ssh -l netadmin <ip>`) |
| LON-PRN-01 | 10.10.30.20 / 24, gw 10.10.30.1, DNS 10.10.80.10 | Static (see ADR 0012) |
| LON-PRN-02 | 10.10.40.20 / 24, gw 10.10.40.1, DNS 10.10.80.10 | Static |
| IST-PC-01 … 02, CHI-PC-01 … 02, TOR-PC-01 … 02 | DHCP | |
| INET-HOST-01 | 192.0.2.100 / 24, gw 192.0.2.1, DNS 192.0.2.53 | The "outside user / attacker" for tests T04 and T16 |
