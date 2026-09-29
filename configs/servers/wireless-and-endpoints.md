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
| VLAN (management) | 10 |

The WLC's uplink (CORE-01 Gi1/0/10) is an 802.1Q trunk with native VLAN 999, so management must be tagged VLAN 10. If Packet Tracer doesn't let you tag the management interface, set the trunk's native VLAN to 10 on LON-CORE-01 instead (the note is in its config).

### 2. Web GUI

From **LON-MGMT-PC-01 → Desktop → Web Browser**, open `http://10.10.10.20` (try `https://` if that fails). On first access, create the admin account: `wlcadmin` / `Lab-Wlc-1`.

**Controller → Interfaces → New** (two dynamic interfaces):

| Interface name | VLAN | IP address | Mask | Gateway | Primary DHCP server |
|---|---|---|---|---|---|
| wifi-corp | 60 | 10.10.60.10 | 255.255.255.0 | 10.10.60.1 | 10.10.80.10 |
| wifi-guest | 70 | 10.10.70.10 | 255.255.255.0 | 10.10.70.1 | 10.10.80.10 |

**Security → AAA → RADIUS → Authentication → New:**

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

### 3. London access points

LON-AP-01 … LON-AP-04 (one per access switch, port Fa0/23): add a **power adapter** in the Physical tab. They take a DHCP address in VLAN 10, learn the WLC address from option 43 (the `WLC Address` field of `serverPool`) and join automatically.

**Check:** WLC GUI → Wireless → All APs lists all four.

---

## Branch access points (AccessPoint-PT)

Each branch has two autonomous APs, one per SSID and one on each switch of the pair ([ADR 0013](../../docs/adr/0013-branch-wifi-autonomous-aps.md)). They're bridges: the SSID lands in whatever VLAN the switch port is in. The 3650s supply PoE.

| AP | Switch port (VLAN) | Config → Port 1 |
|---|---|---|
| IST-AP-01, CHI-AP-01, TOR-AP-01 | SW-01 Gi1/0/21 (VLAN 60) | SSID `BSP-CORP`; Authentication **WPA2** (Enterprise), encryption AES, RADIUS server 10.10.80.11, shared secret `Lab-Radius-1` |
| IST-AP-02, CHI-AP-02, TOR-AP-02 | SW-02 Gi1/0/21 (VLAN 70) | SSID `BSP-GUEST`; Authentication **WPA2-PSK**, key `Lab-Guest-Wifi-1`, AES |

Use a different channel on the two APs at each site (e.g. 1 and 6) so they don't overlap.

**If branch WPA2-Enterprise doesn't work in Packet Tracer:** set the branch `BSP-CORP` APs to WPA2-PSK with key `Lab-Corp-Wifi-1`, update the branch clients to match, and record it as B-14 in the support matrix. London's corporate SSID stays WPA2-Enterprise.

---

## Wireless clients

| Device | Prep | Wireless settings | Expected address |
|---|---|---|---|
| LON-LAPTOP-01 | Physical tab: power off, remove the Ethernet NIC, insert **WPC300N**, power on | `BSP-CORP`, WPA2 (Enterprise), `wifi-user1` / `Lab-Wifi-1`, AES | 10.10.60.100+ |
| LON-SMARTPHONE-01 | None | `BSP-CORP`, WPA2 (Enterprise), `wifi-user2` / `Lab-Wifi-2` | 10.10.60.100+ |
| LON-TABLET-01 | None | `BSP-GUEST`, WPA2-PSK, `Lab-Guest-Wifi-1` | 10.10.70.100+, DNS 192.0.2.53 |
| IST-LAPTOP-01, CHI-LAPTOP-01, TOR-LAPTOP-01 | Swap NIC for **WPC300N** | `BSP-CORP`, WPA2 (Enterprise), `wifi-user3` / `Lab-Wifi-3` (or the PSK fallback) | 10.x.60.20+ |
| IST-TABLET-01, CHI-TABLET-01, TOR-TABLET-01 | None | `BSP-GUEST`, WPA2-PSK, `Lab-Guest-Wifi-1` | 10.x.70.100+, DNS 192.0.2.53 |

Place each branch client close to its site's APs in the workspace (Packet Tracer models range).

---

## IP phones (7960)

LON-PH-01, LON-PH-02, LON-PH-03: Physical tab → drag the **power adapter** onto the phone (London access switches have no PoE). IST-PH-01, CHI-PH-01, TOR-PH-01 are powered by the 3650's PoE; add an adapter only if one doesn't power up. Nothing else to configure: the phone learns the voice VLAN from the switch (CDP), gets an address and option 150 from its site's DHCP, and registers with LON-VOICE-01.

LON-PC-ENG-02 plugs into LON-PH-01's **PC port** (not into the switch).

**Check:** each phone's screen shows a line number (1001–1008). On LON-VOICE-01, `show ephone registered` lists all six.

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
