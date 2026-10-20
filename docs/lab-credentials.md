# Lab credentials

Every secret in this lab, in one place. **These are lab-only values**, published deliberately so the lab can be opened and tested. They are not used anywhere else, and a real network would keep secrets in a vault, not a repository.

| Purpose | Value | Where |
|---|---|---|
| Enable secret | `Lab-Enable-1` | All IOS devices and ISP routers |
| Local admin (fallback) | `netadmin` / `Lab-Admin-1` | All IOS devices |
| TACACS+ admin | `netadmin` / `Lab-Tacacs-User-1` | LON-SRV-AAA-01; used whenever the AAA server is reachable |
| TACACS+ shared key | `Lab-Tacacs-1` | Devices ↔ LON-SRV-AAA-01 |
| RADIUS shared key | `Lab-Radius-1` | LON-WLC-01 ↔ LON-SRV-AAA-01 (port 1812) |
| Corporate Wi-Fi users | `wifi-user1` / `Lab-Wifi-1`, `wifi-user2` / `Lab-Wifi-2`, `wifi-user3` / `Lab-Wifi-3` | London `BSP-CORP` (WPA2-Enterprise) |
| Corporate Wi-Fi PSK | `Lab-Corp-Wifi-1` | Branch `BSP-CORP` APs (AccessPoint-PT has no RADIUS) |
| Guest Wi-Fi PSK | `Lab-Guest-Wifi-1` | `BSP-GUEST` at every site |
| WLC admin | `wlcadmin` / `Lab-Wlc-1` | LON-WLC-01 web GUI |
| OSPF MD5 key 1 | `Lab-Ospf-1` | All OSPF interfaces |
| EIGRP key chain | `Lab-Eigrp-1` | LON-WAN-01 ↔ TOR-RTR-01 ↔ TOR-SW-01/02 |
| BGP neighbor password | `Lab-Bgp-1` | Not configured: Packet Tracer has no BGP session passwords (matrix P0-11f) |
| HSRP MD5 | `Lab-Hsrp-1` | Not configured: Packet Tracer has no HSRP authentication (matrix P0-07a) |
| PPP CHAP | `Lab-Chap-1` | LON-WAN-01 ↔ TOR-RTR-01 (both circuits) |
| IKE PSK (Istanbul) | `Lab-Ike-1` | London edges ↔ IST-RTR-01 |
| IKE PSK (Chicago) | `Lab-Ike-2` | London edges ↔ CHI-RTR-01 |
| NTP key 1 | `Lab-Ntp-1` | All devices ↔ LON-SRV-OPS-01 |
| SNMP RO community | `BSP-RO-Lab` | All IOS devices (restricted to the MGMT subnet) |
| Mail users | `alice` / `Lab-Mail-1`, `bob` / `Lab-Mail-2` | LON-SRV-INTRA-01 |
| IoT server account | `cctv` / `Lab-Iot-1` | LON-SRV-IOT-01 dashboard; every CCTV camera registers with it |
| RIP authentication | — | Not configured: Packet Tracer has no RIP authentication (matrix B-56) |

## Console access

Console logins always use the **local** account (`netadmin` / `Lab-Admin-1`), so a broken AAA server can never lock you out of the console. SSH (VTY) logins try TACACS+ first and fall back to the local account only when the server is unreachable.
