# Addressing plan

Single source of truth for every VLAN, subnet and interface address. Configs are written from this file; if the two ever disagree, this file wins and the config is fixed.

## 1. Address blocks

| Block | Use |
|---|---|
| 10.10.0.0/16 | London (HQ) |
| 10.20.0.0/16 | Istanbul |
| 10.30.0.0/16 | Chicago |
| 10.40.0.0/16 | Toronto |
| 10.x.0.0/24 (per branch) | Branch uplinks and transit (/30) |
| 10.255.0.0/24 | London internal point-to-point links (/30) |
| 10.255.1.0/24 | VPN tunnel interfaces (/30) |
| 10.255.2.0/24 | Toronto private circuits (/30) |
| 10.255.255.0/24 | Router and L3 switch loopbacks (/32) |
| 172.16.100.0/24 | London DMZ |
| 203.0.113.0/24 | Company-owned public block (RFC 5737), advertised in BGP |
| 198.51.100.0/24 | ISP-side links (RFC 5737) |
| 192.0.2.0/24 | Simulated internet services (RFC 5737) |
| 2001:db8:acad::/48 | IPv6 (stretch goal) |

Domain names: internal `corp.bosphorus.test`, public `bosphorus.test`.

**Convention at every site:** in each gateway VLAN, `.1` is the HSRP virtual IP, `.2` the first switch, `.3` the second switch. Static hosts use `.10–.49`; DHCP hands out `.100–.250` (corporate Wi-Fi at the branches `.20–.250`).

## 2. London VLANs

| VLAN | Name | Subnet | HSRP active / STP root | DHCP (LON-SRV-CORE-01) | Notes |
|---|---|---|---|---|---|
| 10 | MGMT | 10.10.10.0/24 | LON-CORE-01 | .100–.250 (APs only) | Switch SVIs, WLC, jump host |
| 20 | ENG | 10.10.20.0/24 | LON-CORE-02 | .100–.250 | 40 desks |
| 30 | SALES | 10.10.30.0/24 | LON-CORE-01 | .100–.250 | 25 desks; printer .20 |
| 40 | FIN-HR | 10.10.40.0/24 | LON-CORE-02 | .100–.250 | 15 desks; restricted; printer .20 |
| 50 | VOICE | 10.10.50.0/24 | LON-CORE-02 | .100–.250, option 150 → 10.10.50.10 | CME at .10 |
| 60 | WIFI-CORP | 10.10.60.0/24 | LON-CORE-01 | .100–.250 | WLC dynamic interface .10 |
| 70 | WIFI-GUEST | 10.10.70.0/24 | LON-CORE-01 | .100–.250, DNS → 192.0.2.53 | Internet only |
| 80 | SERVERS | 10.10.80.0/24 | LON-CORE-02 | none (static) | |
| 900 | TRANSIT | 10.255.0.8/30 | — | — | Core-to-core routed link over Po1 |
| 998 | PARKING | — | — | — | Unused ports, shut down |
| 999 | NATIVE | — | — | — | Native VLAN on all trunks, carries no traffic |

VLANs 10, 30, 60 and 70 are active on CORE-01 (60 and 70 because the WLC is attached to CORE-01); 20, 40, 50 and 80 on CORE-02. HSRP active and STP root are always on the same switch.

## 3. Branch VLANs

Every branch uses the same VLAN numbers. Gateways are HSRP pairs on the two branch switches; DHCP is served by the same two switches with split scopes (SW-01 the lower half, SW-02 the upper half).

| VLAN | Name | Istanbul | Chicago | Toronto | HSRP active | DHCP split (SW-01 / SW-02) | Options |
|---|---|---|---|---|---|---|---|
| 10 | MGMT | 10.20.10.0/24 | 10.30.10.0/24 | 10.40.10.0/24 | SW-01 | — (static) | |
| 20 | USERS | 10.20.20.0/24 | 10.30.20.0/24 | 10.40.20.0/24 | SW-01 | .100–.175 / .176–.250 | DNS 10.10.80.10; printer .20 |
| 50 | VOICE | 10.20.50.0/24 | 10.30.50.0/24 | 10.40.50.0/24 | SW-02 | .100–.175 / .176–.250 | Option 150 → 10.10.50.10 |
| 60 | WIFI-CORP | 10.20.60.0/24 | 10.30.60.0/24 | 10.40.60.0/24 | SW-01 | .20–.135 / .136–.250 | DNS 10.10.80.10 |
| 70 | WIFI-GUEST | 10.20.70.0/24 | 10.30.70.0/24 | 10.40.70.0/24 | SW-02 | .100–.175 / .176–.250 | DNS 192.0.2.53, internet only |
| 900 | TRANSIT | 10.20.0.8/30 | 10.30.0.8/30 | 10.40.0.8/30 | — | — | Switch-to-switch routed link over Po1 (SW-01 .9, SW-02 .10) |

## 4. Interface addressing

### London

| Device | Interface | Address | Connects to / purpose |
|---|---|---|---|
| LON-EDGE-01 | Gi0/0/0 | 198.51.100.2/30 | ISP-A (outside) |
| | Gi0/0/1 | 10.255.0.1/30 | LON-CORE-01 |
| | Gi0/0/2 | 172.16.100.2/24 | DMZ (HSRP VIP 172.16.100.1, active) |
| | Loopback0 | 10.255.255.1/32 | Router ID |
| | Tunnel0 | 10.255.1.1/30 | GRE/IPsec to IST (primary) |
| LON-EDGE-02 | Gi0/0/0 | 198.51.100.6/30 | ISP-B (outside) |
| | Gi0/0/1 | 10.255.0.5/30 | LON-CORE-02 |
| | Gi0/0/2 | 172.16.100.3/24 | DMZ (HSRP standby) |
| | Loopback0 | 10.255.255.2/32 | Router ID |
| | Tunnel1 | 10.255.1.5/30 | GRE/IPsec to IST (backup) |
| LON-CORE-01 | Gi1/0/20 (routed) | 10.255.0.2/30 | LON-EDGE-01 |
| | Gi1/0/21 (routed) | 10.255.0.13/30 | LON-WAN-01 |
| | Vlan900 | 10.255.0.9/30 | Core-to-core transit |
| | Loopback0 | 10.255.255.3/32 | Router ID |
| | Vlan10–80 | 10.10.x.2/24 | SVIs |
| LON-CORE-02 | Gi1/0/20 (routed) | 10.255.0.6/30 | LON-EDGE-02 |
| | Gi1/0/21 (routed) | 10.255.0.17/30 | LON-WAN-01 |
| | Vlan900 | 10.255.0.10/30 | Core-to-core transit |
| | Loopback0 | 10.255.255.4/32 | Router ID |
| | Vlan10–80 | 10.10.x.3/24 | SVIs |
| LON-WAN-01 | Gi0/0/1 | 10.255.0.14/30 | LON-CORE-01 |
| | Gi0/0/2 | 10.255.0.18/30 | LON-CORE-02 |
| | Se0/1/0 | 10.255.2.1/30 | TOR-RTR-01 circuit 1 (DCE) |
| | Se0/1/1 | 10.255.2.5/30 | TOR-RTR-01 circuit 2 (DCE) |
| | Loopback0 | 10.255.255.5/32 | Router ID |
| LON-ACC-01 … LON-ACC-04 | Vlan10 | 10.10.10.11 … 10.10.10.14 /24 | Management |
| LON-SRV-SW01 | Vlan10 | 10.10.10.15/24 | Management |
| LON-DMZ-SW01 | Vlan100 | 172.16.100.4/24 | Management (SSH from MGMT only) |
| LON-WLC-01 | Management | 10.10.10.20/24 | CAPWAP, GUI |
| | wifi-corp / wifi-guest | 10.10.60.10/24, 10.10.70.10/24 | Dynamic interfaces |
| LON-VOICE-01 | Fa0/0 | 10.10.50.10/24 | CME |

### Istanbul

| Device | Interface | Address | Connects to / purpose |
|---|---|---|---|
| IST-RTR-01 | Gi0/0/0 | 198.51.100.10/30 | ISP-A (outside) |
| | Gi0/0/1 | 10.20.0.1/30 | IST-SW-01 |
| | Gi0/0/2 | 10.20.0.5/30 | IST-SW-02 |
| | Tunnel0 | 10.255.1.2/30 | To LON-EDGE-01 |
| | Tunnel1 | 10.255.1.6/30 | To LON-EDGE-02 |
| | Loopback0 | 10.255.255.11/32 | Router ID |
| IST-SW-01 | Gi1/0/22 (routed) | 10.20.0.2/30 | IST-RTR-01 |
| | Vlan10/20/50/60/70 | 10.20.x.2/24 | Gateways (HSRP .1) |
| | Loopback0 | 10.255.255.12/32 | Router ID, TACACS+ source |
| IST-SW-02 | Gi1/0/22 (routed) | 10.20.0.6/30 | IST-RTR-01 |
| | Vlan10/20/50/60/70 | 10.20.x.3/24 | Gateways |
| | Loopback0 | 10.255.255.13/32 | Router ID, TACACS+ source |

### Chicago

| Device | Interface | Address | Connects to / purpose |
|---|---|---|---|
| CHI-FW-01 | Gi1/1 (outside, sec 0) | 198.51.100.14/30 | ISP-B |
| | Gi1/2 (inside, sec 100) | 10.30.0.1/30 | CHI-SW-01 |
| | Gi1/3 (inside2, sec 100) | 10.30.0.5/30 | CHI-SW-02 |
| CHI-SW-01 | Gi1/0/22 (routed) | 10.30.0.2/30 | CHI-FW-01 inside |
| | Vlan10/20/50/60/70 | 10.30.x.2/24 | Gateways (HSRP .1); Vlan10 is the TACACS+ source |
| CHI-SW-02 | Gi1/0/22 (routed) | 10.30.0.6/30 | CHI-FW-01 inside2 |
| | Vlan10/20/50/60/70 | 10.30.x.3/24 | Gateways; Vlan10 is the TACACS+ source |

### Toronto

| Device | Interface | Address | Connects to / purpose |
|---|---|---|---|
| TOR-RTR-01 | Se0/0/0 | 10.255.2.2/30 | LON-WAN-01 circuit 1 (DTE) |
| | Se0/0/1 | 10.255.2.6/30 | LON-WAN-01 circuit 2 (DTE) |
| | Gi0/0 | 10.40.0.1/30 | TOR-SW-01 |
| | Gi0/1 | 10.40.0.5/30 | TOR-SW-02 |
| | Loopback0 | 10.255.255.21/32 | Router ID |
| TOR-SW-01 | Gi1/0/22 (routed) | 10.40.0.2/30 | TOR-RTR-01 |
| | Vlan10/20/50/60/70 | 10.40.x.2/24 | Gateways (HSRP .1) |
| | Loopback0 | 10.255.255.22/32 | Router ID, TACACS+ source |
| TOR-SW-02 | Gi1/0/22 (routed) | 10.40.0.6/30 | TOR-RTR-01 |
| | Vlan10/20/50/60/70 | 10.40.x.3/24 | Gateways |
| | Loopback0 | 10.255.255.23/32 | Router ID, TACACS+ source |

### Simulated internet

| Device | ASN | Interface | Address | Connects to |
|---|---|---|---|---|
| ISP-A | 65001 | Fa0/0 | 198.51.100.1/30 | LON-EDGE-01 |
| | | Fa1/0 | 198.51.100.9/30 | IST-RTR-01 |
| | | Fa4/0 | 198.51.100.17/30 | ISP-B |
| | | Fa5/0 | 198.51.100.21/30 | INET-CORE |
| ISP-B | 65002 | Fa0/0 | 198.51.100.5/30 | LON-EDGE-02 |
| | | Fa1/0 | 198.51.100.13/30 | CHI-FW-01 |
| | | Fa4/0 | 198.51.100.18/30 | ISP-A |
| | | Fa5/0 | 198.51.100.25/30 | INET-CORE |
| INET-CORE | 65000 | Fa0/0 | 198.51.100.22/30 | ISP-A |
| | | Fa1/0 | 198.51.100.26/30 | ISP-B |
| | | Fa4/0 | 192.0.2.1/24 | INET-SW |

The ISPs advertise their connected /30s into BGP, so the branch routers and HQ edges can reach each other's public addresses for VPN peering.

### Loopbacks

| Device | Loopback0 | Device | Loopback0 |
|---|---|---|---|
| LON-EDGE-01 | 10.255.255.1 | IST-RTR-01 | 10.255.255.11 |
| LON-EDGE-02 | 10.255.255.2 | IST-SW-01 | 10.255.255.12 |
| LON-CORE-01 | 10.255.255.3 | IST-SW-02 | 10.255.255.13 |
| LON-CORE-02 | 10.255.255.4 | TOR-RTR-01 | 10.255.255.21 |
| LON-WAN-01 | 10.255.255.5 | TOR-SW-01 | 10.255.255.22 |
| | | TOR-SW-02 | 10.255.255.23 |

## 5. Public addresses and NAT

| Public address | Translates to | On |
|---|---|---|
| 203.0.113.1 | PAT for 10.0.0.0/8 | LON-EDGE-01 |
| 203.0.113.2 | PAT for 10.0.0.0/8 | LON-EDGE-02 |
| 203.0.113.10 | 172.16.100.10 (LON-DMZ-WEB-01) | Both edges (static NAT) |
| 203.0.113.53 | 172.16.100.53 (LON-DMZ-DNS-01) | Both edges (static NAT) |
| 198.51.100.10 | PAT for 10.20.0.0/16 | IST-RTR-01 (interface overload) |
| 198.51.100.14 | PAT for 10.30.0.0/16 | CHI-FW-01 (interface PAT) |

Both edges hold `ip route 203.0.113.0 255.255.255.0 Null0` so the /24 can be advertised with a BGP `network` statement. VPN traffic between sites is exempted from NAT. Toronto has no internet connection and is translated at the London edge.

## 6. Static hosts

| Host | Address | VLAN / segment |
|---|---|---|
| LON-SRV-CORE-01 (DHCP, internal DNS) | 10.10.80.10 | 80 |
| LON-SRV-AAA-01 (TACACS+, RADIUS) | 10.10.80.11 | 80 |
| LON-SRV-OPS-01 (Syslog, NTP, TFTP) | 10.10.80.12 | 80 |
| LON-SRV-INTRA-01 (HTTP, email) | 10.10.80.13 | 80 |
| LON-MGMT-PC-01 (jump host) | 10.10.10.50 | 10 |
| LON-PRN-01 | 10.10.30.20 | 30 |
| LON-PRN-02 | 10.10.40.20 | 40 |
| IST-PRN-01, CHI-PRN-01, TOR-PRN-01 | 10.20.20.20, 10.30.20.20, 10.40.20.20 | 20 |
| LON-DMZ-WEB-01 | 172.16.100.10 | DMZ |
| LON-DMZ-DNS-01 | 172.16.100.53 | DMZ |
| INET-WEB-01 | 192.0.2.10 | Internet |
| INET-DNS-01 | 192.0.2.53 | Internet |
| INET-HOST-01 (external test host) | 192.0.2.100 | Internet |
