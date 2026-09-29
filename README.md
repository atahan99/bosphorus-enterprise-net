# bosphorus-enterprise-net

A four-site enterprise network built and tested in **Cisco Packet Tracer 9.0.1**: a London headquarters with dual ISPs, a DMZ and campus redundancy, plus regional sites in Istanbul, Chicago and Toronto connected over IPsec VPNs and a private circuit.

The scenario is fictional. Every design choice (which routing protocol runs where, how failover works, what each zone may reach) is justified in the docs and verified by a written test plan, including tests that deliberately break things.

> **Status:** design, configs and test plan complete; lab build in progress. See [Build progress](#build-progress).

---

## Topology at a glance

```
                        [INET-CORE AS65000]──(192.0.2.0/24 "internet")
                         /               \
                [ISP-A AS65001]────────[ISP-B AS65002]
                 /        \               /        \
          eBGP  /     IST WAN        eBGP /     CHI WAN
               |          |              |          |
        [LON-EDGE-01]==iBGP==[LON-EDGE-02]      [CHI-FW-01 ASA]
           |    \   DMZ (HSRP)   /   |             /     \
           |     [LON-DMZ-SW01]      |      [CHI-SW-01]=[CHI-SW-02]
           |      WEB   DNS          |
        [LON-CORE-01]===Po1 LACP===[LON-CORE-02]   [IST-RTR-01]
           |   \                        /  |        /     \
           |  [LON-ACC-01..04]  [LON-SRV-SW01]  [IST-SW-01]=[IST-SW-02]
           |   80 desks, WLC + 4 APs, servers, CME
        [LON-WAN-01]═══2 serial circuits═══[TOR-RTR-01]
                                             /     \
                                     [TOR-SW-01]=[TOR-SW-02]

Branch pairs (=): two 3650s, LACP bundle, HSRP, one uplink each
Overlays:  GRE/IPsec Tunnel0  LON-EDGE-01 <-> IST  (primary)
           GRE/IPsec Tunnel1  LON-EDGE-02 <-> IST  (backup)
           ASA crypto map     CHI <-> LON-EDGE-02  (EDGE-01 as backup peer)
```

| Site | Staff | Role | WAN | Routing |
|---|---|---|---|---|
| **London (LON)** | 80 | HQ: redundant core, 4 access switches, servers, DMZ, Wi-Fi, voice, internet edge | ISP-A primary, ISP-B backup | OSPF area 0, eBGP/iBGP |
| **Istanbul (IST)** | 30 | Regional office, redundant switch pair | ISP-A + GRE over IPsec to both London edges | OSPF area 1 |
| **Chicago (CHI)** | 36 | Office behind an ASA, redundant switch pair | ISP-B + ASA IPsec (primary EDGE-02, backup EDGE-01) | Static |
| **Toronto (TOR)** | 36 | Acquired site on a legacy design, redundant switch pair | Two private serial circuits to HQ, no local internet | EIGRP AS 100, redistributed at HQ |

Every site has user, voice, corporate Wi-Fi and guest Wi-Fi VLANs, and no single switch failure takes a site down. Switch ports and DHCP scopes are sized for all 182 staff; Packet Tracer holds a representative sample of endpoints, and the branch AP layout is demo-scale rather than a production RF design (see [sizing](docs/design-plan.md#4-sizing)).

## What's covered

| Area | Features |
|---|---|
| Switching | VLANs, 802.1Q trunks (native VLAN 999, DTP off), VTP transparent, Rapid PVST+ with per-VLAN root placement, LACP EtherChannel |
| L2 security | PortFast + BPDU guard, root guard, loop guard, port security, DHCP snooping, DAI, rogue-switch attack demo |
| Routing | Inter-VLAN routing on L3 switch SVIs, OSPF multi-area with MD5 and summarisation, EIGRP, tagged redistribution, eBGP/iBGP with prefix filtering and AS-path prepending |
| Redundancy | HSRP at every site, redundant switch pairs at the branches, dual ISP failover with symmetric NAT, backup VPN tunnel and backup VPN peer, dual Toronto circuits |
| WAN / VPN | GRE over IPsec, ASA-to-IOS policy-based IPsec, PPP with CHAP |
| Edge security | Zone-based firewall, PAT and static NAT, DMZ |
| Internal security | Inter-VLAN ACL policy matrix, AAA (TACACS+ / RADIUS), SSH-only management, NTP, Syslog, SNMP |
| Services | DHCP, DNS, HTTP, email, TFTP config backups |
| Wireless | Every site: WLC-3504 with lightweight APs in London, autonomous APs at the branches; WPA2-Enterprise corporate SSID, isolated guest SSID |
| Voice / QoS | CME IP telephony for all four sites, WAN QoS (subject to Phase 0 validation) |

## Repository map

| Path | Contents |
|---|---|
| [`docs/design-plan.md`](docs/design-plan.md) | Full design: requirements, every subsystem, redundancy, build phases |
| [`docs/addressing-plan.md`](docs/addressing-plan.md) | VLANs, subnets and every interface IP |
| [`docs/cabling-plan.md`](docs/cabling-plan.md) | Device models, modules and port-to-port cabling |
| [`docs/build-guide.md`](docs/build-guide.md) | Phase-by-phase build steps with verification commands |
| [`docs/test-plan.md`](docs/test-plan.md) | 26 tests, including failure and attack scenarios, with results |
| [`docs/adr/`](docs/adr/) | 14 architecture decision records |
| [`docs/pt-support-matrix.md`](docs/pt-support-matrix.md) | What Packet Tracer 9 does and doesn't support, verified hands-on |
| [`docs/lab-credentials.md`](docs/lab-credentials.md) | Every lab secret in one place |
| [`configs/`](configs/) | Every device's configuration, by site, sectioned by build phase |
| [`configs/servers/`](configs/servers/) | Server, WLC, phone and endpoint settings (GUI-configured) |
| [`diagrams/`](diagrams/) | Topology, Layer 2, routing, security-zone, VPN and wireless diagrams |
| [`phase0/`](phase0/) | Feature-validation lab and procedure |
| [`blog/`](blog/) | Write-up of the design and what I learned |
| [`pkt/`](pkt/) | The Packet Tracer file |
| [`screenshots/`](screenshots/) | Test evidence, named by test ID |

## Build progress

| Phase | Scope | Status |
|---|---|---|
| 0 | Packet Tracer feature validation | ✅ |
| 1 | Design documents, configs and diagrams | ✅ |
| 2 | Place and cable devices | ⏳ |
| 3 | London switching | ⏳ |
| 4 | London routing and first-hop redundancy | ⏳ |
| 5 | Internet edge (BGP, NAT, ZBF, DMZ) | ⏳ |
| 6 | Remote sites and VPNs | ⏳ |
| 7 | Security and management | ⏳ |
| 8 | Wireless, voice, QoS | ⏳ |
| 9 | Testing | ⏳ |
| 10 | Documentation and blog | ⏳ |

## Opening the lab

- Built and tested in **Cisco Packet Tracer 9.0.1 (build 9.0.1.0858)**. Open it with this version or newer; Packet Tracer files are not backward-compatible.
- Credentials in this lab are lab-only and are not used anywhere else.

## Optional: MCP-assisted building

The lab can be built manually from the documentation; no automation is required. As an optional add-on, [MCP-Packet-Tracer by Mats2208](https://github.com/Mats2208/MCP-Packet-Tracer) can automate parts of device placement, cabling and configuration through Packet Tracer's script module.

Install the third-party MCP server in an isolated Python virtual environment such as `.venv-packet-tracer-mcp/` so it can be removed without affecting the system Python installation. Follow the upstream setup instructions, start the `MCP-BUILDER` script module in Packet Tracer, and confirm that its bridge reports online before sending commands.

The MCP device catalog and the Packet Tracer GUI can expose different interface names. Before building the real topology, run the throwaway device/module/link preflight in the [build guide](docs/build-guide.md#phase-2-place-and-cable) and record any accepted port mappings in the cabling plan. The written plans and device configurations remain the canonical reference if automation is unavailable.

## Packet Tracer limitations

Packet Tracer implements a subset of IOS. Where a production network would use something PT lacks (VRRP, DMVPN/VTI, SNMPv3, CoPP, UDLD), the design uses the closest supported equivalent and the gap is recorded in [`docs/pt-support-matrix.md`](docs/pt-support-matrix.md). Convergence times measured in PT are not representative of real hardware.

---

*All company, site and domain names in this project are fictional. Addressing uses RFC 1918 and RFC 5737 documentation ranges; the internal domain uses the reserved `.test` TLD.*
