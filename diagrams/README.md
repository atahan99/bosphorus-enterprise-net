# Diagrams

Mermaid diagrams render directly on GitHub. Addresses and ports are in [addressing-plan.md](../docs/addressing-plan.md) and [cabling-plan.md](../docs/cabling-plan.md).

## 1. Site overview

```mermaid
flowchart TB
  INET["INET-CORE (AS 65000)<br/>192.0.2.0/24 web, DNS, test host"]
  ISPA["ISP-A (AS 65001)"]
  ISPB["ISP-B (AS 65002)"]
  INET --- ISPA
  INET --- ISPB
  ISPA --- ISPB
  subgraph LON["London HQ: 80 staff, 10.10.0.0/16"]
    E1["LON-EDGE-01"]
    E2["LON-EDGE-02"]
    DMZ["DMZ 172.16.100.0/24"]
    CORE["LON-CORE-01 / 02<br/>Po1 LACP"]
    ACC["LON-ACC-01 to 04 (80 desks)<br/>LON-SRV-SW01, WLC + 4 APs"]
    WAN["LON-WAN-01"]
    E1 --- DMZ
    E2 --- DMZ
    E1 --- CORE
    E2 --- CORE
    CORE --- ACC
    CORE --- WAN
  end
  ISPA --- E1
  ISPB --- E2
  ISPA --- IST["Istanbul: 30 staff, 10.20.0.0/16<br/>IST-RTR-01, IST-SW-01/02 pair, 2 APs"]
  ISPB --- CHI["Chicago: 36 staff, 10.30.0.0/16<br/>CHI-FW-01 (ASA), CHI-SW-01/02 pair, 2 APs"]
  WAN -. "two private circuits (serial, PPP/CHAP)" .- TOR["Toronto: 36 staff, 10.40.0.0/16<br/>TOR-RTR-01, TOR-SW-01/02 pair, 2 APs"]
```

Port-level cabling is in [cabling-plan.md](../docs/cabling-plan.md); the Packet Tracer workspace screenshot (`screenshots/phase2-topology-full.png`) is the physical view.

## 2. Layer 2 in London: VLANs, STP roots, HSRP

```mermaid
flowchart TB
  C1["LON-CORE-01<br/>STP root + HSRP active:<br/>VLAN 10, 30, 60, 70"]
  C2["LON-CORE-02<br/>STP root + HSRP active:<br/>VLAN 20, 40, 50, 80"]
  C1 == "Po1 LACP, trunk all VLANs + 900" === C2
  A1["LON-ACC-01 / 02<br/>ENG desks<br/>VLAN 10, 20, 50"]
  A3["LON-ACC-03<br/>SALES desks<br/>VLAN 10, 30, 50"]
  A4["LON-ACC-04<br/>SALES + FIN-HR desks<br/>VLAN 10, 30, 40, 50"]
  S1["LON-SRV-SW01<br/>servers, CME, MGMT PC<br/>VLAN 10, 50, 80"]
  C1 --- A1 & A3 & A4 & S1
  C2 --- A1 & A3 & A4 & S1
  C1 -- "trunk 10, 60, 70" --- WLC["LON-WLC-01"]
```

Every downlink is dual-homed: root guard on the core side, loop guard on the access side. Per VLAN, each access switch forwards on the uplink to that VLAN's root and blocks the other (on LON-ACC-01, VLAN 10 forwards on Gi0/1 and blocks Gi0/2; VLAN 20 the reverse). All trunks: native VLAN 999, DTP off, explicit allowed lists.

## 2b. Branch layout (every branch)

```mermaid
flowchart TB
  UP["IST-RTR-01 (OSPF) / CHI-FW-01 (static) / TOR-RTR-01 (EIGRP)"]
  S1["SW-01 (3650)<br/>HSRP active 10, 20, 60<br/>15-18 desks, AP BSP-CORP"]
  S2["SW-02 (3650)<br/>HSRP active 50, 70<br/>15-18 desks, printer, AP BSP-GUEST"]
  UP -- "routed /30, Gi1/0/22" --- S1
  UP -- "routed /30, Gi1/0/22" --- S2
  S1 == "Po1 LACP (Gi1/0/23-24): all VLANs + transit VLAN 900" === S2
```

Either switch can fail: the other keeps every gateway (HSRP), its own uplink and its half of the DHCP scopes.

## 3. Routing domains

```mermaid
flowchart LR
  subgraph BGP["eBGP"]
    ISPA["ISP-A AS 65001"]
    ISPB["ISP-B AS 65002"]
  end
  subgraph AS["AS 65100 (iBGP between edge loopbacks)"]
    subgraph A0["OSPF area 0"]
      E1["EDGE-01<br/>ABR, ASBR<br/>default metric 10"]
      E2["EDGE-02<br/>ABR, ASBR<br/>default metric 100<br/>+ static 10.30.0.0/16"]
      CORE["CORE-01 / CORE-02"]
      WAN["LON-WAN-01<br/>ASBR"]
    end
    subgraph A1["OSPF area 1"]
      IST["IST-RTR-01 + IST-SW-01/02"]
    end
    subgraph EIGRP["EIGRP AS 100"]
      TOR["TOR-RTR-01<br/>summary 10.40.0.0/16"]
      TORC["TOR-SW-01/02 (stubs)"]
      TOR --- TORC
    end
  end
  ISPA -- "eBGP: /24 out, default in" --- E1
  ISPB -- "eBGP: /24 prepended x2" --- E2
  E1 --- CORE --- E2
  CORE --- WAN
  E1 -- "Tunnel0 cost 10" --- IST
  E2 -- "Tunnel1 cost 100" --- IST
  WAN -- "2 circuits; redistribution, tags 100/110" --- TOR
  E2 -. "IPsec primary (static)" .- CHI["CHI-FW-01 + CHI-SW-01/02"]
  E1 -. "IPsec backup" .- CHI
```

Summaries at the ABRs: `10.10.0.0/16` into area 1, `10.20.0.0/16` into area 0.

## 4. Security zones (London edge)

```mermaid
flowchart LR
  IN["INSIDE<br/>core link, tunnels"]
  DMZ["DMZ<br/>172.16.100.0/24"]
  OUT["OUTSIDE<br/>ISP"]
  IN -- "inspect: http, https, dns, icmp<br/>(EDGE-02: pass Chicago VPN)" --> OUT
  IN -- "pass: web/DNS to servers, SSH from MGMT" --> DMZ
  DMZ -- "pass: replies only" --> IN
  OUT -- "inspect: web, DNS to published hosts" --> DMZ
  DMZ -- "inspect: http, https, dns, icmp" --> OUT
  OUT -. "drop<br/>(EDGE-02: pass Chicago VPN)" .-> IN
```

## 5. VPN overlay

```mermaid
flowchart LR
  E1["LON-EDGE-01<br/>198.51.100.2"]
  E2["LON-EDGE-02<br/>198.51.100.6"]
  IST["IST-RTR-01<br/>198.51.100.10"]
  CHI["CHI-FW-01<br/>198.51.100.14"]
  E1 == "GRE/IPsec Tunnel0 10.255.1.0/30<br/>primary, OSPF" === IST
  E2 -- "GRE/IPsec Tunnel1 10.255.1.4/30<br/>backup, OSPF" --- IST
  E2 -- "IPsec crypto map (primary)<br/>10.0.0.0/8 to 10.30.0.0/16" --- CHI
  E1 -. "IPsec backup peer" .- CHI
```

## 6. Wireless

```mermaid
flowchart LR
  subgraph LONW["London: central"]
    AP["LON-AP-01 to 04<br/>VLAN 10, DHCP option 43"] -- CAPWAP --> WLC["LON-WLC-01<br/>10.10.10.20"]
    WLC -- "BSP-CORP to VLAN 60" --> CORP["Corp clients"]
    WLC -- "BSP-GUEST to VLAN 70" --> GUEST["Guests, internet only"]
  end
  subgraph BR["Each branch: local switching"]
    BAP1["AP-01 on SW-01: BSP-CORP"] --> V60["VLAN 60"]
    BAP2["AP-02 on SW-02: BSP-GUEST"] --> V70["VLAN 70, guest ACL"]
  end
  WLC -- "RADIUS" --> AAA["LON-SRV-AAA-01"]
  BAP1 -- "RADIUS over VPN / circuit" --> AAA
```
