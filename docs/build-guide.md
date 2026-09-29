# Build guide

Step-by-step instructions for building the lab in Packet Tracer 9.0.1. Each phase says what to do in the GUI, which config sections to paste, how to check it worked, and which screenshots to take.

All device configs are in [`configs/`](../configs/), split into `! ===== Phase N` sections. Server, WLC and endpoint settings are in [`configs/servers/`](../configs/servers/).

## How to paste configs

1. Click the device → **CLI** tab → press Enter → `enable` → `configure terminal`.
2. Copy the section(s) for the current phase from the `.cfg` file and paste. `!` lines are comments and are ignored.
3. Scroll back and look for `% Invalid input` or `% Incomplete command`. If you see one, check the `P0-xx` comment next to that line for the fallback, and note it in [`pt-support-matrix.md`](pt-support-matrix.md).
4. `end` → `copy running-config startup-config` (press Enter to confirm).

Tips:

- **Paste one phase at a time** rather than a whole file. Errors are easier to spot and later phases rely on earlier ones being verified.
- Packet Tracer can be slow with long pastes. If lines seem to be dropped, paste in smaller chunks.
- Use **Fast Forward Time** (bottom-left) to skip waiting for STP, OSPF and BGP to converge.
- Turn off packet animation in Simulation mode except when capturing screenshots.

## After every phase

- Add a visible note box to the workspace with the phase number, site/subnet scope, active protocols and any Packet Tracer fallback used. Keep notes clear of devices and links.
- Save a phase checkpoint as `pkt/phase-N.pkt`, then also refresh `pkt/bosphorus-enterprise-net.pkt` as the latest canonical copy.
- Take the listed screenshots and save them in `screenshots/` with the given names. Capture the whole canvas as `phaseN-topology-full.png` **and** close-ups of the areas the phase touched as `phaseN-topology-part1-<area>.png`, `part2-…`, and so on: the whole workspace is too large to read on its own, and the MCP's single-image capture fails once it exceeds the bridge's transfer limit.
- If this folder is a Git repository, commit and tag: `git add -A && git commit -m "Phase N: <summary>" && git tag phase-N`. If it is not, initialize Git once before Phase 0 or skip the Git checkpoint; saving the `.pkt` remains required.

---

## Phase 0: feature validation

Follow [`phase0/phase0-feature-tests.md`](../phase0/phase0-feature-tests.md) before placing the production topology. Phase 0 is a required compatibility gate: record each result in [`pt-support-matrix.md`](pt-support-matrix.md) and apply the documented fallback before using a rejected command in the main lab.

**Checkpoint:** save the scratch lab as `phase0/phase0-scratch.pkt`, with a labeled summary box on the canvas. Capture `phase0/evidence/phase0-final-labeled.png` and record the run in `phase0/phase0-results.md`.

---

## Phase 2: place and cable

**MCP preflight:** in a throwaway file, place one of each distinct model, install the planned modules, and create one representative link per interface family. Confirm the interface names accepted by the MCP before placing the production devices. In particular, verify the 3650 uplinks, ASA interfaces and Router-PT expansion interfaces; the MCP catalog may name them differently from the Packet Tracer GUI. Delete the throwaway file only after recording any mapping changes in the cabling plan.

**GUI:**

1. Place every device from [cabling-plan.md §1](cabling-plan.md#1-devices-and-modules), adding modules and power supplies first (power off → add module → power on). On ISP-A, ISP-B and INET-CORE, replace the fiber modules in slots 4 and 5 with PT-ROUTER-NM-1CFE (matrix B-31).
2. Set each device's display name to its hostname.
3. Cable every link in [cabling-plan.md §2–3](cabling-plan.md#2-infrastructure-links). For the serial link, choose Serial DCE and click LON-WAN-01 first.
4. Draw labelled boxes around each site and the internet (see [cabling-plan.md §4](cabling-plan.md#4-workspace-layout)).
5. Power off the `Wireless0` port of every laptop, tablet and smartphone (Config tab → Wireless0 → Port Status). Until the SSIDs exist in Phase 8 they associate with any AP, including other sites' (matrix B-32).

**Security licence** on LON-EDGE-01, LON-EDGE-02 and IST-RTR-01. In PT 9.0.1 a new ISR4331 already has `securityk9` as its current permanent package (matrix B-30), so check `show version` first; if it isn't listed, run:

```
enable
configure terminal
license boot level securityk9
end
write memory
reload
```

**IOS 15 on every 2960-24TT** (LON-ACC-01 … 04, LON-SRV-SW01, LON-DMZ-SW01, LON-ROGUE-SW, INET-SW). A new 2960 boots 12.2(25)FX, which rejects the DAI lines in the Phase 3 configs ([matrix](pt-support-matrix.md) B-24). Do this before pasting any config:

1. Place a temporary **Server-PT** named `TMP-TFTP`, static `192.168.99.1/24`. Its TFTP service is on by default and holds the image.
2. Cable `TMP-TFTP` FastEthernet0 to the switch's **Fa0/24** (straight-through).
3. On the switch:

   ```
   enable
   configure terminal
   interface Vlan1
    ip address 192.168.99.2 255.255.255.0
    no shutdown
   end
   copy tftp: flash:
   192.168.99.1
   c2960-lanbasek9-mz.150-2.SE4.bin
   c2960-lanbasek9-mz.150-2.SE4.bin
   configure terminal
   boot system flash:c2960-lanbasek9-mz.150-2.SE4.bin
   interface Vlan1
    no ip address
    shutdown
   end
   write memory
   reload
   ```

4. After the reload `show version | include Version 15` must print `Version 15.0(2)SE4`. Move the cable to the next 2960 (`copy tftp:` again) and repeat.
5. Delete `TMP-TFTP` and its cable when every 2960 is done.

   Shortcut used in the Phase 2 build: before any config every London switch is in VLAN 1 and joined through the cores, so one temporary server on LON-ACC-01 Fa0/24 serves all five London 2960s at once (give each a different 192.168.99.x address, and wait about 30 s for spanning tree). LON-DMZ-SW01, INET-SW and LON-ROGUE-SW are isolated and each need their own temporary server.

**Paste:** the `Phase 2: base` section on every router and switch that has one, and `Phase 2: base` on CHI-FW-01. The ISP routers are configured in Phase 5.

**Verify:**

| Command | Where | Expected |
|---|---|---|
| `show version \| include securityk9` | EDGE-01/02, IST | securityk9 listed as the current licence |
| `show version \| include Version 15` | Every 2960-24TT | `Version 15.0(2)SE4` |
| `show cdp neighbors` | Any router or switch | Neighbors and ports match cabling-plan.md |

**Screenshots:** `phase2-topology-full.png` (whole workspace, zoomed to fit).

---

## Phase 3: London switching

**Paste `Phase 3` sections (both `switching` and `Layer 2 security`) on:** LON-CORE-01, LON-CORE-02, LON-ACC-01 … LON-ACC-04, LON-SRV-SW01, LON-DMZ-SW01.

Paste the two cores first, then the access switches.

If `show etherchannel summary` then shows `Po1(SD)` with the members `(s)` (matrix B-33), re-apply the channel-group on **both** cores:

```
interface range GigabitEthernet1/0/23 - 24
 shutdown
 no channel-group 1
 channel-group 1 mode active
 no shutdown
```

Po1 should come up as `(SU)` with both members `(P)` within a few seconds.

**Verify:**

| Command | Where | Expected |
|---|---|---|
| `show vlan brief` | Each switch | Its VLANs exist; access ports in the right VLAN; unused ports in 998 |
| `show interfaces trunk` | Cores, access switches | Trunks up, native VLAN 999, allowed lists as configured |
| `show etherchannel summary` | Cores | `Po1(SU)`, members `(P)` |
| `show spanning-tree root` | LON-ACC-01 | Root for VLAN 10 = CORE-01's MAC; root for 20, 50 = CORE-02's MAC |
| `show spanning-tree vlan 10` | LON-ACC-01 | Gi0/1 root port, Gi0/2 **alternate (blocked)** |
| `show spanning-tree vlan 20` | LON-ACC-01 | Gi0/2 root port, Gi0/1 alternate |
| `show interfaces status` | LON-ACC-01 … 04 | Desk ports in their VLAN (connected only where a device is placed), spare ports disabled |
| `show ip arp inspection vlan 20` | Access switches | DAI Enabled / Active on the user and voice VLANs (IOS 15 required) |
| `show port-security` | Access switches | Access ports listed |
| `show ip dhcp snooping` | Access switches | Enabled on the listed VLANs, uplinks trusted |

**Screenshots:** `phase3-etherchannel.png`, `phase3-stp-acc01-vlan10.png`, `phase3-stp-acc01-vlan20.png`, `phase3-trunks-core01.png`, plus the whole canvas `phase3-topology-full.png` and close-ups `phase3-topology-part1-london-edge-core.png`, `part2-london-access.png`, `part3-phase-notes.png`.

---

## Phase 4: London routing, HSRP, DHCP

**GUI:** configure LON-SRV-CORE-01, LON-SRV-AAA-01, LON-SRV-OPS-01 and LON-SRV-INTRA-01 from [servers.md](../configs/servers/servers.md). Set the London PCs and printer from [wireless-and-endpoints.md](../configs/servers/wireless-and-endpoints.md#pcs-and-printer).

**Paste `Phase 4` sections on:** LON-CORE-01, LON-CORE-02, LON-EDGE-01, LON-EDGE-02, LON-WAN-01.

Inter-VLAN ACLs aren't applied until Phase 7, so at this point every London VLAN can reach every other.

**Verify:**

| Command / action | Where | Expected |
|---|---|---|
| `show standby brief` | CORE-01 | Active for 10, 30, 60, 70; Standby for 20, 40, 50, 80 (CORE-02 the reverse) |
| `show ip ospf neighbor` | CORE-01 | FULL with EDGE-01, WAN-01 and CORE-02 (via Vlan900) |
| `show ip route ospf` | EDGE-01 | All 10.10.x.0/24 VLANs, loopbacks, point-to-point links |
| PC → IP Configuration → DHCP | Every London PC | Address in .100–.250 of their VLAN, gateway .1 |
| `ping 10.10.80.13`, browse `http://intranet.corp.bosphorus.test` | LON-PC-ENG-01 | Works |

**Screenshots:** `phase4-hsrp-core01.png`, `phase4-ospf-neighbors-core01.png`, `phase4-dhcp-pc-eng01.png`, plus `phase4-topology-full.png` and the close-ups `phase4-topology-part1-london-edge-core.png` … `part4-phase-notes.png`.

---

## Phase 5: internet edge

**GUI:** configure LON-DMZ-WEB-01, LON-DMZ-DNS-01, INET-WEB-01, INET-DNS-01 and INET-HOST-01 ([servers.md](../configs/servers/servers.md)).

**Paste:**

1. ISP-A, ISP-B, INET-CORE, INET-SW (whole files).
2. The `Phase 5` section on LON-EDGE-01 and LON-EDGE-02.

**Verify:**

| Command / action | Where | Expected |
|---|---|---|
| `show ip bgp summary` | EDGE-01, EDGE-02 | Their ISP neighbor established, prefixes received (including 0.0.0.0/0) |
| `show ip bgp` | INET-CORE | 203.0.113.0/25 and 203.0.113.128/25 via ISP-A (`65001 65100`); 203.0.113.0/24 via ISP-B (`65002 65100`) |
| `show ip route 0.0.0.0` | CORE-01 and CORE-02 | O*E2 towards LON-EDGE-01 (CORE-02 via CORE-01, not via the cost-1000 link to EDGE-02) |
| `show standby brief` | EDGE-01 | DMZ group 100 Active |
| Browse `http://www.example.test` | LON-PC-ENG-01 | Internet page loads |
| `show ip nat translations` | EDGE-01 | PC's address translated to 203.0.113.1 |
| `show policy-map type inspect zone-pair sessions` | EDGE-01 | Inspected session for the browse |
| Browse `http://203.0.113.10` and `http://www.bosphorus.test` | INET-HOST-01 | Public site loads |
| Browse `http://www.bosphorus.test` | LON-PC-ENG-01 | Loads via split DNS (172.16.100.10) |

**Screenshots:** `phase5-bgp-summary-edge01.png`, `phase5-bgp-table-inet-core.png`, `phase5-nat-edge01.png`, `phase5-browse-from-internet.png`.

---

## Phase 6: remote sites and VPNs

**GUI:** add AC power supplies to all six branch 3650s before powering them on. Set every branch PC to DHCP. Branch phones and APs are powered by the 3650s (PoE).

**Paste:**

1. **Istanbul:** IST-RTR-01 (licence already reloaded in Phase 2): both `Phase 6` sections. IST-SW-01 and IST-SW-02: `Phase 2` and all `Phase 6` sections.
2. **London VPN hub:** the `Phase 6` section on LON-EDGE-01 and LON-EDGE-02 (both now carry the Chicago VPN: EDGE-02 primary, EDGE-01 backup).
3. **Chicago:** CHI-FW-01: its `Phase 6` sections. CHI-SW-01 and CHI-SW-02: `Phase 2` and all `Phase 6` sections.
4. **Toronto:** the `Phase 6` section on LON-WAN-01 (both circuits). TOR-RTR-01: `Phase 6`. TOR-SW-01 and TOR-SW-02: `Phase 2` and all `Phase 6` sections.

**Verify:**

| Command / action | Where | Expected |
|---|---|---|
| `show etherchannel summary` | Any branch switch | `Po1(SU)`, both members `(P)` |
| `show standby brief` | IST-SW-01 | Active for 10, 20, 60; Standby for 50, 70 (SW-02 the reverse) |
| `show ip dhcp binding` | Both switches of a branch | Leases split between them (lower half on SW-01, upper half on SW-02) |
| `show crypto isakmp sa` | IST-RTR-01 | Two SAs (198.51.100.2 and .6), `QM_IDLE` |
| `show crypto ipsec sa` | IST-RTR-01 | encaps/decaps counters increasing on both peers |
| `show ip ospf neighbor` | IST-RTR-01 | FULL on Tunnel0, Tunnel1, Gi0/0/1 and Gi0/0/2 |
| `show ip route ospf` | IST-SW-01 | `O IA 10.10.0.0/16`; static default to 10.20.0.1 in the table |
| `show ip route 10.20.0.0` | LON-CORE-01 | `O IA 10.20.0.0/16` via EDGE-01 |
| Ping 10.10.80.10 | CHI-PC-01 | Works after the first one or two time out (IKE negotiation) |
| `show crypto isakmp sa`, `show crypto ipsec sa` | CHI-FW-01 | Tunnel to 198.51.100.6 up, counters increasing |
| `show ip route 10.30.0.0` | LON-CORE-01 | `O E2 10.30.0.0/16` via EDGE-02 |
| `show interfaces serial0/0/0` and `serial0/0/1` | TOR-RTR-01 | Both up/up, `LCP Open` |
| `show ip eigrp neighbors` | TOR-RTR-01 | LON-WAN-01 twice (10.255.2.1 and .5) and both Toronto switches |
| `show ip route` | TOR-SW-01 | `D*EX 0.0.0.0/0` and London routes as `D EX` |
| `show ip route 10.40.0.0` | LON-CORE-01 | `O E2 10.40.0.0/16` (the Toronto summary; no route tag, PT has no route-maps) |
| Browse `http://www.example.test` | TOR-PC-01 | Works (via London's internet edge) |
| Browse `http://www.example.test` | IST-PC-01, CHI-PC-01 | Works (local breakout) |

**Screenshots:** `phase6-branch-hsrp.png`, `phase6-ist-crypto-sa.png`, `phase6-ist-ospf-routes.png`, `phase6-asa-vpn.png`, `phase6-tor-eigrp-neighbors.png`, `phase6-core-toronto-route.png`.

---

## Phase 7: security and management

**Paste the `Phase 7` sections on every device that has one** (all four sites), including CHI-FW-01. The simulated ISP and internet devices do not have Phase 7 sections.

From now on, console login asks for `netadmin` / `Lab-Admin-1` (see [lab-credentials.md](lab-credentials.md)).

**Verify:**

| Command / action | Where | Expected |
|---|---|---|
| `ssh -l netadmin 10.10.10.11` (password `Lab-Tacacs-User-1`) | LON-MGMT-PC-01 | Logs in via TACACS+ |
| `telnet 10.10.10.11` | LON-MGMT-PC-01 | Refused |
| `show ntp status` | Any IOS device | `Clock is synchronized` (can take a few minutes; use Fast Forward) |
| Services → SYSLOG | LON-SRV-OPS-01 | Messages arriving from devices |
| `show access-lists` | CORE-01 | Hit counters increasing |
| Ping LON-PC-FIN-01 | LON-PC-ENG-01 | **Fails** (policy) |
| Browse `http://intranet.corp.bosphorus.test` | LON-PC-ENG-01 | Works |

**Screenshots:** `phase7-ssh-tacacs.png`, `phase7-syslog-server.png`, `phase7-ntp-status.png`, `phase7-acl-counters-core01.png`.

---

## Phase 8: wireless, voice, QoS

**GUI:**

1. LON-WLC-01, the four London APs, the six branch APs and the wireless clients ([wireless-and-endpoints.md](../configs/servers/wireless-and-endpoints.md)).
2. Power adapters on LON-PH-01/02/03 (the branch phones were done in Phase 6).

**Paste:**

1. LON-VOICE-01: `Phase 2`, `Phase 8` and `Phase 7` sections.
2. The `Phase 8: QoS` section on LON-EDGE-01, LON-EDGE-02, LON-WAN-01, IST-RTR-01, TOR-RTR-01.

**Verify:**

| Command / action | Where | Expected |
|---|---|---|
| Wireless → All APs | WLC GUI | Four APs joined |
| IP configuration | LON-LAPTOP-01, LON-SMARTPHONE-01 | 10.10.60.x |
| IP configuration | LON-TABLET-01 | 10.10.70.x, DNS 192.0.2.53 |
| IP configuration | IST/CHI/TOR laptops and tablets | 10.x.60.x (corp) and 10.x.70.x (guest) from the site's own DHCP |
| `show ephone registered` | LON-VOICE-01 | Six phones |
| Call between two London phones, then London ↔ each branch | Phones | Rings, connects |
| `show policy-map interface GigabitEthernet0/0/0` | EDGE-01 | Classes listed; EF counters increase during a London ↔ Istanbul call |

**Screenshots:** `phase8-wlc-aps.png`, `phase8-wlans.png`, `phase8-branch-wifi.png`, `phase8-ephones.png`, `phase8-call-lon-tor.png`, `phase8-qos-serial.png`.

---

## Phase 9: testing

Run every test in [test-plan.md](test-plan.md) and record the results there, with screenshots named by test ID.

---

## Phase 10: documentation and publishing

1. **Export final configs:** on each device, `show running-config`, copy the output into `configs/<site>/<HOSTNAME>.running.txt`. Also run the TFTP backup once from each IOS device to prove it works: `copy running-config tftp:` → `10.10.80.12` → accept the default filename.
2. Fill in the Phase 0 matrix rows you didn't test, and anything you learned during the build.
3. Update the README's build-progress table and test-results summary.
4. Finish the blog draft in [`blog/`](../blog/) with the screenshots.
5. Push to GitHub, set the repo description and topics (`cisco`, `packet-tracer`, `networking`, `ccna`, `ccnp`, `network-security`, `bgp`, `ospf`, `ipsec`).
