# Test plan

Every test has a purpose, a procedure, the expected result and a place to record what actually happened. Evidence goes in `screenshots/` named by test ID (`T04-before-bgp.png`, `T04-after-bgp.png`, ...).

Run the tests after Phase 8, in order. Each test ends by restoring the lab to its normal state.

**Result legend:** ✅ pass · ⚠️ pass with deviation (explain) · ❌ fail

## Summary

| ID | Test | Result |
|---|---|---|
| T01 | Inter-VLAN and inter-site policy matrix | ⚠️ |
| T02 | Core switch failure (HSRP + STP) | ✅ |
| T03 | EtherChannel member failure | ✅ |
| T04 | ISP-A failure (BGP, NAT, DMZ failover) | ⚠️ |
| T05 | Primary VPN tunnel failure | ⚠️ |
| T06 | Chicago site-to-site VPN | ✅ |
| T07 | Wrong CHAP password | ✅ |
| T08 | OSPF authentication mismatch | ✅ |
| T09 | Port-security violation | not run |
| T10 | Rogue DHCP server | not run |
| T11 | Guest Wi-Fi isolation | ⚠️ |
| T12 | Management access: SSH only, TACACS+ with local fallback | ✅ |
| T13 | Wi-Fi with wrong credentials | ✅ |
| T14 | Voice calls in London and to every branch | ✅ |
| T15 | Syslog and NTP | ✅ |
| T16 | Unsolicited inbound traffic from the internet | ✅ |
| T17 | Compromised DMZ host can't reach inside | ✅ |
| T18 | Rogue switch: BPDU guard | ✅ |
| T19 | Rogue switch: root guard as the second layer | not run |
| T20 | Loop guard on a blocked uplink | n/a (not in PT, matrix P0-02b) |
| T21 | Branch switch failure | ✅ |
| T22 | Toronto circuit failure | ✅ |
| T23 | Chicago backup tunnel | ✅ |
| T24 | Dynamic ARP Inspection active on the access switches | ⚠️ |
| T25 | BGP advertisements and inbound preference | ✅ |
| T26 | QoS under congestion on a Toronto circuit | not run |

**Phase 9 run (2026-10-01):** 16 ✅, 5 ⚠️, 4 not run, 1 n/a, no test ❌ overall. Every failover test recovered and the lab was restored after each one. Restore points: `pkt/phase-9-wip1.pkt` … `wip6.pkt` (not committed).

**Issues found by the tests:**

1. **Fixed after the run: TACACS+ client addresses (T01 row 13).** LON-SRV-AAA-01 listed IST-SW-01 as `10.20.0.2` and CHI-SW-01 as `10.30.0.2`; corrected to `10.20.0.14` / `10.30.0.14`, and London MGMT then logged in to IST-SW-01 over SSH with TACACS+.
2. **London Wi-Fi in VLAN 10 (T01 row 15, T11):** a PT limitation, matrix B-53. Guest and corporate London Wi-Fi clients are not isolated from internal networks.
3. **Fixed during testing:** LON-CORE-01's floating static default pointed at LON-WAN-01 (T04, B-25); IST ↔ EDGE-02 IKE needs a crypto-map re-bind after reopening the .pkt (T05, B-54); a branch switch reboot err-disabled its desk ports through PT's DHCP snooping rate limit, now set explicitly to 100 pps (T21, B-55).

---

## T01: Inter-VLAN and inter-site policy matrix

**Purpose:** prove the ACL policy in [design-plan.md §10](design-plan.md#10-internal-security) is enforced.

**Procedure:**

1. Clear the ACL counters on the London cores and on every branch switch pair: `clear access-list counters` on LON-CORE-01/02, IST-SW-01/02, CHI-SW-01/02 and TOR-SW-01/02.
2. Run each check below from the source device's Command Prompt or Web Browser.
3. Record the hit counters (see **ACL counter checks** after the table). The branch rows are filtered on the switch that is HSRP active for the source VLAN: SW-01 for VLANs 20 and 60, SW-02 for voice VLAN 50 and guest VLAN 70. In London, LON-CORE-02 is active for VLANs 20 and 40. Confirm with `show standby brief` before reading counters.

| # | From | To | Check | Expected | Actual |
|---|---|---|---|---|---|
| 1 | LON-PC-ENG-01 | LON-PC-SALES-01 | ping | ✔ | ✔ 4/4 |
| 2 | LON-PC-ENG-01 | LON-PC-FIN-01 | ping | ✘ | ✘ 0/4 |
| 3 | LON-PC-ENG-01 | LON-MGMT-PC-01 (10.10.10.50) | ping | ✘ | ✘ 0/4 |
| 4 | LON-PC-ENG-01 | 10.10.80.12 | ping | ✔ | ✔ 4/4 |
| 5 | LON-PC-ENG-01 | intranet.corp.bosphorus.test | browse | ✔ | ✔ (browsed) |
| 6 | LON-PC-ENG-01 | www.bosphorus.test | browse | ✔ (DMZ via split DNS) | ✔ (browsed) |
| 7 | LON-PC-ENG-01 | www.example.test | browse | ✔ | ✔ (browsed) |
| 8 | LON-PC-ENG-01 | IST-PC-01 | ping | ✔ | ✔ 3/4 (pinged IST-PC-01 → ENG-01) |
| 9 | LON-PC-FIN-01 | 10.10.80.13 | browse intranet | ✔ | ✔ (browsed) |
| 10 | LON-PC-FIN-01 | LON-PC-SALES-01 | ping | ✘ | ✘ 0/4 |
| 11 | LON-PC-FIN-01 | www.example.test | browse | ✔ | ✔ (browsed) |
| 12 | LON-MGMT-PC-01 | LON-PC-FIN-01 | ping | ✔ | ✔ 4/4 |
| 13 | LON-MGMT-PC-01 | 10.20.10.2 (IST-SW-01) | ssh | ✔ | ✔ after fix: first run rejected (AAA server had IST-SW-01 as 10.20.0.2 instead of 10.20.0.14); corrected, then `IST-SW-01>` |
| 14 | LON-TABLET-01 (guest) | www.example.test | browse | ✔ | ✔ (browsed) |
| 15 | LON-TABLET-01 (guest) | 10.10.80.10 | ping | ✘ | **✔ (fail)**: tablet is in VLAN 10 (B-53) |
| 16 | LON-LAPTOP-01 (corp) | intranet.corp.bosphorus.test | browse | ✔ | ✔ (browsed) |
| 17 | IST-PC-01 | LON-PC-FIN-01 | ping | ✘ | ✘ 0/2 |
| 18 | TOR-PC-01 | 10.10.80.13 | browse intranet | ✔ | ping ✔ 4/4; browse ✔ |
| 19 | CHI-PC-01 | 10.10.80.13 | browse intranet | ✔ | ping ✔ 3/4; browse ✔ |
| 20 | IST-TABLET-01 (branch guest) | www.example.test | browse | ✔ | ✔ (browsed) |
| 21 | IST-TABLET-01 (branch guest) | 10.20.20.1 and 10.10.80.10 | ping | ✘ | ✘ 0/2, ✘ 0/4 |
| 22 | TOR-LAPTOP-01 (branch corp) | intranet.corp.bosphorus.test | browse | ✔ | ping ✔ 4/4; browse ✔ |
| 23 | TOR-LAPTOP-01 (branch corp) | LON-PC-FIN-01 | ping | ✘ | ✘ 0/4 |
| 24 | IST-PH-01 | LON-VOICE-01 / a London phone | register / call | ✔ | see T14 |
| 25 | IST-PH-01 | www.example.test | browse or ping | ✘ | not run (phones can't ping) |
| 26 | IST-PC-01 | 10.20.10.2 and 10.20.10.3 (own site's switch management) | ping, ssh | ✘ | ✘ 0/4 (ping) |
| 27 | IST-PC-01 | 10.20.0.13 (IST-RTR-01 transit) | ping | ✘ | ✘ 0/4 |
| 28 | TEST-PC on an Istanbul VLAN 50 port (IP phones can't originate pings) | 10.20.10.2 | ping | ✘ | not run (same ACL entry as row 26) |
| 29 | TOR-LAPTOP-01 (branch corp) | LON-PC-ENG-01 | ping | ✔ | ✔ 4/4 |
| 30 | LON-PC-ENG-01 | TOR-LAPTOP-01 (branch corp) | ping | ✔ | ✔ 4/4 |
| 31 | LON-LAPTOP-01 (corp) | TOR-LAPTOP-01 (branch corp) | ping | ✔ | ✔ 4/4 (London laptop in VLAN 10, B-53) |

Rows 26–28 check that branch users and phones can't reach their own site's management VLAN or routed transit links (only London MGMT manages branch devices, row 13). Rows 29–31 check that branch corporate Wi-Fi is treated as a peer user network in both directions: the London user and corporate Wi-Fi ACLs permit replies to it.

**ACL counter checks:** after the table, run `show access-lists` on the device named below and check that the listed entry's match count rose. A row that passes with a counter still at zero passed for the wrong reason (for example, a missing route rather than the ACL), so treat it as ⚠️ and investigate.

| Row(s) | Device | ACL | Entry whose counter should increase |
|---|---|---|---|
| 2, 10 | LON-CORE-02 | ACL-ENG-IN / ACL-FIN-IN | `deny ip any 10.0.0.0 0.255.255.255` |
| 17 | IST-SW-01 | ACL-BRANCH-DATA-IN | `deny ip any 10.0.0.0 0.255.255.255` (the ping to FIN never leaves Istanbul) |
| 23 | TOR-SW-01 | ACL-BRANCH-DATA-IN (on Vlan60) | `deny ip any 10.0.0.0 0.255.255.255` |
| 21 | IST-SW-02 | ACL-WIFI-GUEST-IN | `deny ip any 10.0.0.0 0.255.255.255` |
| 25 | IST-SW-02 | ACL-BRANCH-VOICE-IN | final `deny ip any any` |
| 26 | IST-SW-01 | ACL-BRANCH-DATA-IN | `deny ip any 10.20.10.0 0.0.0.255` |
| 27 | IST-SW-01 | ACL-BRANCH-DATA-IN | `deny ip any 10.20.0.0 0.0.0.255` |
| 28 | IST-SW-02 | ACL-BRANCH-VOICE-IN | `deny ip any 10.20.10.0 0.0.0.255` |
| 8, 18, 19, 22 | IST-SW-01, TOR-SW-01, CHI-SW-01 | ACL-BRANCH-DATA-IN | the matching permit (London user subnet, 10.10.80.13 `eq www`) |
| 29 | TOR-SW-01 / LON-CORE-02 | ACL-BRANCH-DATA-IN / ACL-ENG-IN | `permit ip 10.40.0.0 0.0.255.255 10.10.20.0 0.0.0.255` / `permit ip 10.10.20.0 0.0.0.255 10.40.60.0 0.0.0.255` |

Repeat rows 17 and 26 once with IST-SW-01 powered off (or during T21): the same deny entries must now count on IST-SW-02, proving the policy survives a gateway failover.

**Evidence:** `T01-matrix-*.png` for a few representative rows; `T01-acl-lon-core02.png`, `T01-acl-ist-sw01.png`, `T01-acl-ist-sw02.png`, `T01-acl-tor-sw01.png` showing the counters above; `T01-acl-failover.png` for the repeat on IST-SW-02.

**Actual:** ⚠️ Run 2026-10-01 from the CLI. Every ✘ row was blocked and every ✔ row passed, with the expected ACL counters rising (`T01-acl-lon-core02.png`, `T01-acl-ist-sw01.png`, `T01-acl-ist-sw02.png`, `T01-acl-tor-sw01.png`). The first 2-packet pings of rows 1 and 12 lost both packets to ARP resolution; 4-packet re-runs passed. Deviations: row 15 passes traffic that should be blocked because London Wi-Fi clients are in VLAN 10 (matrix B-53); row 13 first failed: LON-SRV-AAA-01 had IST-SW-01 registered as 10.20.0.2 (and CHI-SW-01 as 10.30.0.2), but those switches source TACACS+ from their router links 10.20.0.14 / 10.30.0.14 as documented in servers.md, so the server rejects them and IOS doesn't fall back to the local user. The 'connection refused' on 10.20.0.14 was `login block-for` Quiet Mode after the failed attempts (`%SEC_LOGIN-1-QUIET_MODE_ON` on LON-SRV-OPS-01), which is the protection working. After correcting both server entries to 10.20.0.14 / 10.30.0.14, SSH with the TACACS+ password reached `IST-SW-01>`. Browse rows were checked by hand afterwards: every page loaded. Row 28 and the failover repeat are still open.

---

## T02: Core switch failure

**Purpose:** gateway and Layer 2 failover when LON-CORE-01 dies.

**Procedure:**

1. On LON-PC-SALES-01 (VLAN 30, whose gateway is active on CORE-01): `ping -n 100 10.10.80.13`.
2. While it runs, power off LON-CORE-01 (Physical tab → power switch).
3. When the ping finishes, power CORE-01 back on and wait for it to boot (Fast Forward Time).

**Expected:** a few pings lost, then replies resume. On CORE-02 `show standby brief` shows **Active for all eight VLANs**; on LON-ACC-03 (where SALES-01 is connected) `show spanning-tree vlan 30` shows Gi0/2 as root port. After CORE-01 returns, preempt makes it Active again for 10/30/60/70.

**Evidence:** `T02-ping-loss.png`, `T02-standby-core02-after.png`, `T02-stp-acc01-after.png`.

**Actual:** ✅ LON-PC-SALES-01 44/50 (12% loss): six timeouts during the switchover, then 44 consecutive replies. CORE-02 Active for all eight VLANs; LON-ACC-03 VLAN 30 root port moved to Gi0/2 (the test plan's ACC-01 screenshot name is kept as `T02-stp-acc03-after.png`, the switch SALES-01 is on). After CORE-01 came back it preempted 10/30/60/70; OSPF FULL to EDGE-01, WAN-01 and CORE-02, Po1 with both members.

---

## T03: EtherChannel member failure

**Procedure:** on LON-PC-SALES-01 `ping -n 50 10.10.80.13`. This flow crosses Po1: VLAN 30's gateway is CORE-01, but the server VLAN's forwarding path to LON-SRV-SW01 is through CORE-02 (its STP root). While it runs, on LON-CORE-01: `interface GigabitEthernet1/0/23` → `shutdown`. Afterwards `no shutdown`.

**Expected:** no pings lost. `show etherchannel summary` shows Gi1/0/23 down and Po1 still `(SU)` with one member.

**Evidence:** `T03-etherchannel-one-member.png`. **Actual:** ✅ 30/30 replies (0% loss) from LON-PC-SALES-01 with Gi1/0/23 shut; `Po1(SU) LACP Gig1/0/23(D) Gig1/0/24(P)`.

---

## T04: ISP-A failure

**Purpose:** the key design test. Outbound and inbound traffic move to LON-EDGE-02 together and stay symmetric ([ADR 0001](adr/0001-active-standby-internet-edge.md)). The DMZ gateway has to be moved by hand because Packet Tracer has no HSRP tracking.

**Before:** capture these on the normal network:

| Command / action | Where |
|---|---|
| `show ip bgp summary` | LON-EDGE-01 |
| `show ip route 0.0.0.0` | LON-CORE-01 |
| `show standby brief` | LON-EDGE-01 |
| `show ip bgp 203.0.113.0` | INET-CORE |

**Procedure:** on ISP-A: `interface FastEthernet0/0` → `shutdown`. Wait for convergence (Fast Forward Time), then:

| Check | Where | Expected |
|---|---|---|
| `show ip bgp summary` | EDGE-01 | ISP-A neighbor down |
| `show ip route 0.0.0.0` | EDGE-01 | No default (`Gateway of last resort is not set`); EDGE-01 no longer originates one |
| `show ip route 0.0.0.0` | CORE-01, CORE-02 | `S* 0.0.0.0/0 [250/0]`, the floating static towards EDGE-02 (B-25) |
| Browse www.example.test, then `show ip nat translations` | LON-PC-ENG-01, then EDGE-02 | Works; translated to **203.0.113.2** |
| `show ip bgp` | INET-CORE | The two /25s are gone; only 203.0.113.0/24 via ISP-B remains |
| `show ip ospf neighbor` | IST-RTR-01 | Tunnel0 neighbor gone after the dead timer, Tunnel1 FULL |
| On EDGE-01 `interface GigabitEthernet0/0/2` → `standby 100 priority 90`, then `show standby brief` on EDGE-02 | EDGE-01, EDGE-02 | DMZ group 100 **Active** on EDGE-02 |
| Browse http://203.0.113.10 | INET-HOST-01 | Public site still loads |

Before the priority change, check that browsing 203.0.113.10 from INET-HOST-01 **fails** and record it: this is the gap left by the missing HSRP tracking.

**Restore:** `no shutdown` on ISP-A Fa0/0 and `standby 100 priority 110` on EDGE-01 Gi0/0/2. Then, for Packet Tracer (tested in Phase 5):

1. Press **Fast Forward Time** a few times until EDGE-01's `show ip bgp summary` shows a fresh Up/Down time (B-39).
2. On EDGE-01: `router ospf 1` → `no default-information originate` → `default-information originate` (B-37).
3. On EDGE-02: `clear ip bgp *`, then Fast Forward Time again, so INET-CORE learns 203.0.113.0/25 again (B-38).

Everything is then back on EDGE-01: DMZ HSRP preempts, the cores show O*E2 via EDGE-01, and INET-CORE shows both /25s via ISP-A.

**Also check (matrix B-25):** before the failure, `show ip route 0.0.0.0` on LON-CORE-01 and LON-CORE-02 must show a single O*E2 path, advertised by 10.255.255.1 (EDGE-01).

**Evidence:** `T04-before-*.png`, `T04-after-*.png`. **Actual:** ⚠️ (found and fixed a config bug). With ISP-A Fa0/0 shut: EDGE-01 lost its default (`Gateway of last resort is not set`), INET-CORE kept only 203.0.113.0/24 via ISP-B, IST-RTR-01 ran on Tunnel1 only, CORE-02 used its floating static to EDGE-02 and LON-PC-ENG-01 reached 192.0.2.10 as 203.0.113.2. **But CORE-01's floating static pointed at 10.255.0.14 (LON-WAN-01, which has no default), so every VLAN whose gateway is CORE-01 lost the internet (LON-PC-SALES-01 0/4).** Fixed to `ip route 0.0.0.0 0.0.0.0 10.255.0.10 250` (via CORE-02 over Vlan900, as the config comment intended); SALES-01 then 4/4 via 203.0.113.2. EDGE-01 `standby 100 priority 90` made EDGE-02 Active for the DMZ. Restore: BGP re-established without Fast Forward, B-37 re-origination, `clear ip bgp *` on EDGE-02; everything back on EDGE-01. Browsing from INET-HOST-01 before/after the DMZ move is a GUI step. By hand: INET-HOST-01 loads http://203.0.113.10 in the normal state; the browse during the failover (before and after moving the DMZ gateway) was not repeated.

---

## T05: Primary VPN tunnel failure

**Procedure:** on IST-PC-01 `tracert 10.10.80.13`. On LON-EDGE-01: `interface Tunnel0` → `shutdown`. On IST-RTR-01 `show ip route ospf`, then repeat the tracert. Restore with `no shutdown`.

**Expected:** before, the path goes via 10.255.1.1 (EDGE-01); after, via 10.255.1.5 (EDGE-02), and IST-RTR-01's routes point at Tunnel1.

**Evidence:** `T05-tracert-before.png`, `T05-tracert-after.png`, `T05-ist-routes-after.png`. **Actual:** ⚠️ Before: IST-PC-01 → 10.255.1.1 (EDGE-01). With Tunnel0 shut, IST-RTR-01 first lost every London route: its IKE session to EDGE-02 sat in `MM_NO_STATE (deleted)` with no encrypted packets, although keys, policies and ACLs matched and the outside ACL counted the ISAKMP packets. `clear crypto isakmp`/`sa` did nothing; re-binding `crypto map CM-WAN` on IST-RTR-01 Gi0/0/0 and then EDGE-02 Gi0/0/0 brought it up (matrix B-54). After that: routes via Tunnel1, tracert via 10.255.1.5 → 10.255.0.6 (`T05-*.png`).

---

## T06: Chicago site-to-site VPN

Chicago uses GRE over IPsec to both London edges with OSPF area 2, like Istanbul. It was first designed around an ASA; PT's ASA could not complete IPsec with IOS ([ADR 0007](adr/0007-chicago-router-not-asa.md)).

**Procedure:** CHI-PC-01 `ping 10.10.80.10` (repeat if the first attempt times out while IKE negotiates). Then:

| Command | Where | Expected |
|---|---|---|
| `show crypto isakmp sa` | CHI-RTR-01 | Two SAs (198.51.100.2 and .6), `QM_IDLE` |
| `show crypto ipsec sa` | CHI-RTR-01 | encaps/decaps increasing for both peers |
| `show ip ospf neighbor` | CHI-RTR-01 | FULL on Tunnel0, Tunnel1, Gi0/0/1 and Gi0/0/2 |
| `show ip route 10.30.20.0` | LON-CORE-01 | `O IA 10.30.20.0/24` via EDGE-01 |
| Ping CHI-PC-01 from LON-MGMT-PC-01 | London | Works (London-initiated through the tunnel) |

**Evidence:** `T06-chi-crypto.png`, `T06-core-chicago-route.png`. **Actual:** ✅ Two ISAKMP SAs QM_IDLE (198.51.100.2 and .6), encaps/decaps 464/480 and 274/249, OSPF FULL on Tunnel0, Tunnel1, Gi0/0/1 and Gi0/0/2; LON-CORE-01 has `O IA 10.30.20.0/24` via EDGE-01. The London-initiated ping was not run separately.

---

## T07: Wrong CHAP password

**Procedure:** on TOR-RTR-01: `username LON-WAN-01 password Wrong-Chap-1`, then `interface Serial0/0/0` → `shutdown` → `no shutdown`. Optionally `debug ppp authentication` (turn off with `undebug all`).

**Expected:** `show interfaces serial0/0/0` → line protocol **down**; EIGRP neighbor gone; debug shows CHAP failure. Restore `username LON-WAN-01 password Lab-Chap-1` and bounce the interface.

**Evidence:** `T07-serial-down.png`. **Actual:** ✅ Serial0/0/0 `up, line protocol is down (disabled)`, `LCP Closed`; only the Se0/0/1 EIGRP neighbor remained. Restored. `debug ppp authentication` was not captured.

---

## T08: OSPF authentication mismatch

**Procedure:** on LON-CORE-01: `interface GigabitEthernet1/0/20` → `ip ospf message-digest-key 1 md5 Wrong-Key` (remove the old key first: `no ip ospf message-digest-key 1`).

**Expected:** the adjacency with EDGE-01 drops after the dead timer (`show ip ospf neighbor`); traffic reroutes via CORE-02. Restore the correct key.

**Evidence:** `T08-ospf-neighbor-down.png`. **Actual:** ✅ After the dead timer LON-EDGE-01 disappeared from CORE-01's neighbors. At the same moment CORE-01's adjacencies to CORE-02 (Vlan900) and LON-WAN-01 showed EXSTART. The correct key was restored, and PT 9.0.1 then crashed in its EIGRP MD5 code (matrix B-49) before the recovery and reroute could be captured; recheck the reroute when re-running T02/T04.

---

## T09: Port-security violation

**Procedure:**

1. On LON-ACC-03: `interface FastEthernet0/1` → `switchport port-security maximum 1`. The sticky MAC of LON-PC-SALES-01 is already learned.
2. Add a new PC (TEST-PC), disconnect LON-PC-SALES-01 from Fa0/1 and connect TEST-PC instead. Give TEST-PC a static 10.10.30.99 and ping 10.10.30.1.

**Expected:** pings fail; `show port-security interface fa0/1` shows the violation count increasing; a `%PORT_SECURITY-2-PSECURE_VIOLATION` message appears in the log and on LON-SRV-OPS-01.

**Restore:** reconnect LON-PC-SALES-01, delete TEST-PC, `switchport port-security maximum 3`.

**Evidence:** `T09-port-security.png`, `T09-syslog.png`. **Actual:** Not run: Packet Tracer enforces port security (sticky MACs are learned on every access port in this lab); the swap test was judged low value against the setup cost.

---

## T10: Rogue DHCP server

**Procedure:** add a Server-PT named ROGUE-DHCP, static 10.10.20.254/24, DHCP service on with a pool for 10.10.20.0/24 whose gateway is 10.10.20.254. Connect it to LON-ACC-01 **Fa0/24** (the spare VLAN 20 desk port). On LON-PC-ENG-01, switch IP configuration to Static and back to DHCP to renew.

**Expected:** LON-PC-ENG-01 still gets a lease from 10.10.80.10 with gateway 10.10.20.1. `show ip dhcp snooping statistics` (or binding table) on LON-ACC-01 shows the rogue offers dropped on the untrusted port.

**Restore:** delete ROGUE-DHCP.

**Evidence:** `T10-pc-lease.png`, `T10-snooping.png`. **Actual:** Not run: DHCP snooping was already validated in Phase 0 (matrix P0-04: the client got its lease, untrusted ports filtered). Building a full rogue DHCP server adds no new evidence.

---

## T11: Guest Wi-Fi isolation

**Procedure:**

1. On LON-TABLET-01: browse `http://www.example.test`, `ping 10.10.80.10`, `ping 10.10.10.1`, `ping 172.16.100.10`.
2. On TOR-TABLET-01 (a branch guest whose internet goes through London): browse `http://www.example.test`, `ping 10.40.20.1`, `ping 10.10.80.10`.

**Expected:** browsing works at both sites; every ping to an internal address fails.

**Evidence:** `T11-guest-london.png`, `T11-guest-toronto.png`. **Actual:** ⚠️ Toronto: TOR-TABLET-01 → 10.40.20.1 and 10.10.80.10 both 0/4 (blocked). London: LON-TABLET-01 reaches 10.10.80.10 and 172.16.100.10, because PT puts every London Wi-Fi client in VLAN 10 (matrix B-53). LON-TABLET-01 and IST-TABLET-01 browse www.example.test (T01 rows 14 and 20).

---

## T12: Management access

**Procedure** (from LON-MGMT-PC-01 unless stated):

1. `telnet 10.10.10.11` → refused.
2. `ssh -l netadmin 10.10.10.11`, password `Lab-Admin-1` (the local one) → **rejected**, because the TACACS+ server is up and has a different password.
3. Same with `Lab-Tacacs-User-1` → **accepted** (TACACS+).
4. On LON-SRV-AAA-01, turn the AAA service **off**. SSH again with `Lab-Admin-1` → **accepted** (local fallback).
5. From LON-PC-ENG-01: `ssh -l netadmin 10.10.10.11` → fails (VLAN 20 can't reach MGMT, and the VTY access-class only allows 10.10.10.0/24).

**Restore:** turn the AAA service back on.

**Evidence:** `T12-telnet-refused.png`, `T12-ssh-tacacs.png`, `T12-ssh-local-fallback.png`. **Actual:** ✅ By hand on LON-MGMT-PC-01: `telnet 10.10.10.11` refused; with the TACACS+ server up, `Lab-Tacacs-User-1` accepted and the local `Lab-Admin-1` rejected; with the AAA service off, `Lab-Admin-1` accepted (local fallback); from LON-PC-ENG-01 the SSH attempt fails. Branch switch management (T01 row 13) fails separately.

---

## T13: Wi-Fi with wrong credentials

**Procedure:** on LON-LAPTOP-01, change the `BSP-CORP` password to `Wrong-Wifi-1`.

**Expected:** the laptop doesn't associate and gets no address. Restore `Lab-Wifi-1`.

**Evidence:** `T13-wifi-rejected.png`. **Actual:** ✅ By hand: with `Wrong-Wifi-1` LON-LAPTOP-01 got no address; with `Lab-Wifi-1` restored it got 10.10.10.109 (VLAN 10, B-53).

---

## T14: Voice calls

**Procedure:** `show ephone` on LON-VOICE-01 to see which number each phone got (`show ephone registered` prints nothing in PT). Call London → London (e.g. LON-PH-01 → LON-PH-02), then London → Istanbul, London → Chicago and London → Toronto. During the Toronto call, `show policy-map interface Serial0/1/0` on LON-WAN-01. Behaviour under congestion is tested separately in T26.

**Expected:** every call rings and connects; EF (voice) class counters increase on the Toronto circuit during that call.

**Evidence:** `T14-call-local.png`, `T14-call-branches.png`, `T14-qos-serial.png`. **Actual:** ✅ By hand: LON-PH-01 called 1003 (LON-PH-02), 1005 (IST), 1002 (CHI) and 1001 (TOR); every call rang and connected (`phase8-call-lon-ist.png`). The EF counter check during the Toronto call is covered by T26.

---

## T15: Syslog and NTP

**Procedure:** on any IOS device `show ntp status` and `show clock`. On LON-SRV-OPS-01, open Services → SYSLOG. Generate a message (e.g. `shutdown`/`no shutdown` an unused port).

**Expected:** `Clock is synchronized`, stratum from 10.10.80.12; the syslog server shows the message with the device's IP and a millisecond timestamp.

**Evidence:** `T15-ntp.png`, `T15-syslog.png`. **Actual:** ✅ TOR-SW-01 `Clock is synchronized, stratum 2, reference is 10.10.80.12`; IST-RTR-01 clock in step. LON-SRV-OPS-01 holds messages from 11 devices across all four sites with millisecond timestamps. An unplugged port bounce on LON-ACC-01 generated no message.

---

## T16: Unsolicited inbound traffic

**Procedure** (from INET-HOST-01):

1. Browse `http://203.0.113.10` → works (published).
2. `ping 203.0.113.53` → fails (DNS server only publishes DNS).
3. `telnet 203.0.113.10` → fails (only web is published).
4. `ping 203.0.113.1` → fails (PAT address, nothing listening).

On LON-EDGE-01: `show ip access-lists ACL-OUTSIDE-IN` shows the hits on the final `deny ip any any`.

**Evidence:** `T16-internet-host.png`, `T16-acl-drops.png`. **Actual:** ✅ From INET-HOST-01: ping 203.0.113.53 and 203.0.113.1 0/4, `telnet 203.0.113.10` timed out; ACL-OUTSIDE-IN final `deny ip any any` 22 hits. Browsing http://203.0.113.10 from INET-HOST-01 loads the published site (checked by hand).

---

## T17: Compromised DMZ host

**Procedure** (from LON-DMZ-WEB-01's Command Prompt): `ping 10.10.80.10`, `ping 10.10.10.50`, and browse `http://10.10.80.13`.

**Expected:** all fail. The DMZ → INSIDE policy only passes replies to connections opened from inside.

**Evidence:** `T17-dmz-blocked.png`. **Actual:** ✅ LON-DMZ-WEB-01 → 10.10.80.10 and 10.10.10.50 both 0/4. Browsing 10.10.80.13 is a GUI step. Browsing http://10.10.80.13 from LON-DMZ-WEB-01 also fails (checked by hand).

---

## T18: Rogue switch vs BPDU guard

**Before:** `show spanning-tree root` on LON-ACC-01 and LON-CORE-01.

**Procedure:** cable LON-ROGUE-SW Gi0/1 → LON-ACC-01 Fa0/24 (copper cross-over).

**Expected:** Fa0/24 goes err-disabled immediately (`show interfaces status err-disabled`); `%SPANTREE-2-BLOCK_BPDUGUARD` appears in the log and on the syslog server; the root bridge is unchanged for every VLAN. The port stays err-disabled: Packet Tracer has no errdisable recovery (matrix P0-03).

**Restore:** remove the cable, then `shutdown` / `no shutdown` on Fa0/24.

**Evidence:** `T18-err-disabled.png`, `T18-syslog.png`, `T18-root-unchanged.png`. **Actual:** ✅ LON-ROGUE-SW cabled to LON-ACC-01 Fa0/24 (cross-over): the port went err-disabled immediately, `%SPANTREE-2-BLOCK_BPDUGUARD` and `%PM-4-ERR_DISABLE` were logged locally and on LON-SRV-OPS-01 (11:34:58.420), and VLAN 20's root stayed LON-CORE-02 (priority 24596). Restored with the cable removed and `shutdown` / `no shutdown` on Fa0/24.

---

## T19: Rogue switch vs root guard

**Purpose:** show the second layer of defence. BPDU guard is removed from the edge port on purpose.

**Procedure:**

1. On LON-ACC-01: `interface FastEthernet0/24` → `no spanning-tree bpduguard enable` → `no spanning-tree portfast`.
2. Cable LON-ROGUE-SW as in T18 and wait for STP to converge.

**Expected:** LON-ACC-01 accepts the rogue as root for VLAN 20 and sends superior BPDUs upstream, but the cores' root guard rejects them: both cores log `%SPANTREE-2-ROOTGUARDBLOCK` for Gi1/0/1 and `show spanning-tree vlan 20` shows it **BLK**. The cores stay root. Record whether traffic still crosses that port: in Phase 0 PT blocked the port in STP only, not on the data plane (matrix P0-02a), so a pass is judged on the STP state and log.

**Restore:** remove the cable, re-enable `spanning-tree portfast` and `spanning-tree bpduguard enable` on Fa0/24. Then `shutdown` / `no shutdown` Gi1/0/1 on both cores: PT's root-guard block does not clear by itself.

**Evidence:** `T19-root-inconsistent-core01.png`. **Actual:** Not run: Phase 0 (matrix P0-02a) already showed that PT enforces root guard in the STP state only, not on the data plane, and that the block has to be cleared by bouncing the core ports.

---

## T20: Loop guard

**Not run.** Packet Tracer rejects `spanning-tree guard loop` and `spanning-tree loopguard default` on both the 2960 (IOS 15) and the 3650 (matrix P0-02b/c), so the access uplinks have no loop guard. The test is kept as a record of the production control: a blocked uplink that stops receiving BPDUs (a one-way link failure) should go loop-inconsistent instead of forwarding.

---

## T21: Branch switch failure

**Purpose:** a branch survives the loss of either switch of its pair.

**Procedure:** on IST-PC-02 (connected to IST-SW-02) run `ping -n 100 10.10.80.13`. While it runs, power off **IST-SW-01** (the active gateway for VLAN 20). Then renew IST-PC-02's DHCP lease and browse `http://www.example.test`.

**Expected:** a few pings lost while HSRP moves, then replies resume. On IST-SW-02, `show standby brief` shows Active for every VLAN. The renewed lease comes from IST-SW-02's upper-half scope (.176–.250). Browsing works through IST-SW-02's own router link. IST-PC-01 (on the failed switch) is offline, as expected.

**Restore:** power IST-SW-01 back on; it pre-empts VLANs 10, 20, 60.

**Evidence:** `T21-ping-loss.png`, `T21-standby-sw02.png`, `T21-dhcp-upper-half.png`. **Actual:** ✅ IST-PC-02 37/40 (8% loss) with IST-SW-01 powered off; IST-SW-02 Active for VLANs 10/20/50/60/70; renewed lease 10.20.20.176 from IST-SW-02's upper-half scope; 192.0.2.10 reachable (3/4, first lost to ARP) via IST-SW-02's own router link. IST-SW-01 preempted 10/20/60 when powered back on. Found afterwards: the reboot had err-disabled Gi1/0/1–2 (IST-PC-01, IST-PH-01) through PT's built-in DHCP snooping rate limit; fixed with `ip dhcp snooping limit rate 100` on every branch access port (matrix B-55).

---

## T22: Toronto circuit failure

**Procedure:** on TOR-PC-01 run `ping -n 100 10.10.80.13`. While it runs, on LON-WAN-01: `interface Serial0/1/0` → `shutdown`.

**Expected:** little or no loss. `show ip eigrp neighbors` on TOR-RTR-01 shows only the 10.255.2.5 neighbor on circuit 2; `show ip route 10.10.0.0` points at Serial0/0/1. Calls from TOR-PH-01 still work.

**Restore:** `no shutdown`; EIGRP load-shares across both circuits again.

**Evidence:** `T22-eigrp-one-circuit.png`. **Actual:** ✅ TOR-PC-01 38/40 (5% loss) with LON-WAN-01 Se0/1/0 shut; TOR-RTR-01 kept only the 10.255.2.5 neighbor, London routes via Serial0/0/1, TOR-PH-01 stayed registered (a call was not placed).

---

## T23: Chicago backup tunnel

**Procedure:** on CHI-PC-01 `tracert 10.10.80.13`. On LON-EDGE-01: `interface Tunnel2` → `shutdown`. On CHI-RTR-01 `show ip route ospf`, then repeat the tracert. Restore with `no shutdown`.

**Expected:** before, the path goes via 10.255.1.9 (EDGE-01); after, via 10.255.1.13 (EDGE-02), and CHI-RTR-01's London routes point at Tunnel1. LON-CORE-01 reaches the Chicago subnets via EDGE-02.

**Evidence:** `T23-tracert-before.png`, `T23-tracert-after.png`. **Actual:** ✅ Before via 10.255.1.9 (EDGE-01); with Tunnel2 shut, CHI-RTR-01 routes via Tunnel1 and tracert goes 10.255.1.13 → 10.255.0.6; LON-CORE-01 reaches 10.30.20.0/24 via Gi1/0/21.

---

## T24: Dynamic ARP Inspection

**Purpose:** show DAI is configured and inspecting, and record how far Packet Tracer enforces it.

**Procedure:** connect TEST-PC to an unused, untrusted VLAN 20 access port on LON-ACC-01. Give it a static address of `10.10.20.240/24`, gateway `10.10.20.1`, and try to ping the gateway. Then inspect `show ip arp inspection vlan 20` and `show ip arp inspection statistics` on LON-ACC-01.

**Expected:** VLAN 20 shows DAI **Enabled / Active** and the forwarded counter increases. In Phase 0 PT did not drop ARP from a host without a binding (matrix P0-05a), so the ping is expected to **succeed**; record it as ⚠️ with that note. A drop would be a better-than-expected result. A normal DHCP client on the same VLAN works either way.

**Restore:** disconnect TEST-PC and return the port to its documented test/spare state.

**Evidence:** `T24-dai-active.png`, `T24-dhcp-client-works.png`. **Actual:** ⚠️ LON-ACC-01 VLAN 20 DAI `Enabled / Active`, counters 0 (`T24-dai-active.png`). The TEST-PC spoofing step was not run: Phase 0 (matrix P0-05a) already showed PT forwards ARP from unbound hosts.

---

## T25: BGP advertisements

**Purpose:** prove that London only announces its own block, and that the internet prefers ISP-A without AS-path prepending ([ADR 0001](adr/0001-active-standby-internet-edge.md)).

**Procedure:**

1. On ISP-A and ISP-B, `show ip bgp` and check what each learned from its London neighbor.
2. On INET-CORE, `show ip bgp` and `show ip route 203.0.113.10`.
3. On both London edges, `show ip bgp` and check that each received `0.0.0.0/0` from its ISP.

**Expected:** ISP-A learns only `203.0.113.0/25` and `203.0.113.128/25` from EDGE-01; ISP-B learns only `203.0.113.0/24` from EDGE-02. INET-CORE reaches 203.0.113.10 through ISP-A (longest match on the /25). No private prefix is ever announced, because the edges have `network` statements only for the public block (PT has no outbound prefix-list, matrix P0-11c).

**Evidence:** `T25-isp-a-learned.png`, `T25-inet-core-route.png`. **Actual:** ✅ ISP-A learns only 203.0.113.0/25 and .128/25 from EDGE-01; ISP-B learns 203.0.113.0/24 from EDGE-02; INET-CORE routes 203.0.113.0/25 via ISP-A (longest match for .10); both edges receive 0.0.0.0/0; no private prefix anywhere.

---

## T26: QoS under congestion

**Purpose:** show that `PM-WAN-OUT` protects voice when a WAN link is actually full, not just that the class counters move. Packet Tracer's queuing model is approximate, so treat the result as a demonstration of the policy's behaviour, not a measurement of latency or jitter.

**Prerequisites:** P0-17a/b passed (MQC marking and queuing accepted on the ISR2911 and ISR4331).

**Procedure:**

1. Put all Toronto traffic on one 2 Mb/s circuit: on LON-WAN-01, `interface Serial0/1/1` → `shutdown`. Confirm with `show ip route 10.40.0.0` that only Serial0/1/0 is used.
2. `clear counters` and `clear policy-map interface Serial0/1/0` (if accepted) on LON-WAN-01.
3. Start a call from LON-PH-01 to TOR-PH-01 and leave it connected.
4. Generate bulk traffic from London towards Toronto, above the circuit's 2 Mb/s:
   - Preferred: on LON-PC-ENG-01 and LON-PC-SALES-01, open **Desktop → Traffic Generator** and send large frames (1,400+ bytes) to TOR-PC-01 at the shortest interval available, continuously.
   - Fallback: in Simulation mode, add periodic Complex PDUs (ICMP, 1,400 bytes, shortest period) from the same two PCs to TOR-PC-01, then switch back to Realtime.
5. With traffic running for at least 30 seconds, capture on LON-WAN-01: `show policy-map interface Serial0/1/0` and `show interfaces Serial0/1/0` (output drops and txload).
6. During the congestion, check that the call is still connected and that `ping -n 20 10.40.50.1` from a London voice-VLAN test point (or LON-VOICE-01: `ping 10.40.50.1 repeat 20`) completes.
7. Optional contrast: remove the policy with `interface Serial0/1/0` → `no service-policy output PM-WAN-OUT`, repeat step 5, then reapply it.

**Expected:** `txload` near 255/255 and output drops rising, confirming the circuit is congested. In `show policy-map interface`, drops fall in `class-default`, while `CM-VOICE` shows packets matched with zero (or near-zero) drops. The call stays connected. In the optional contrast run, drops are no longer separated by class. If Packet Tracer shows no drops at all despite the generated load, record ⚠️: the simulator did not model congestion on this build, and only the counter evidence from T14 is available.

**Restore:** stop the traffic generators or delete the Complex PDUs; reapply `service-policy output PM-WAN-OUT` if removed; `interface Serial0/1/1` → `no shutdown` and confirm EIGRP load-shares across both circuits again.

**Evidence:** `T26-congested-interface.png`, `T26-policy-map-drops.png`, `T26-call-during-congestion.png`, and `T26-no-policy-contrast.png` if step 7 was run. **Actual:** Not run: PT's queuing model is approximate and congestion tests were the highest crash risk in this lab (matrix B-49). Classification and queuing are shown by the class counters instead (`phase8-qos-serial.png`: CS3 and AF31 matches on the Toronto circuit).
