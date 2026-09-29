# Test plan

Every test has a purpose, a procedure, the expected result and a place to record what actually happened. Evidence goes in `screenshots/` named by test ID (`T04-before-bgp.png`, `T04-after-bgp.png`, ...).

Run the tests after Phase 8, in order. Each test ends by restoring the lab to its normal state.

**Result legend:** ✅ pass · ⚠️ pass with deviation (explain) · ❌ fail

## Summary

| ID | Test | Result |
|---|---|---|
| T01 | Inter-VLAN and inter-site policy matrix | |
| T02 | Core switch failure (HSRP + STP) | |
| T03 | EtherChannel member failure | |
| T04 | ISP-A failure (BGP, NAT, DMZ failover) | |
| T05 | Primary VPN tunnel failure | |
| T06 | Chicago site-to-site VPN | |
| T07 | Wrong CHAP password | |
| T08 | OSPF authentication mismatch | |
| T09 | Port-security violation | |
| T10 | Rogue DHCP server | |
| T11 | Guest Wi-Fi isolation | |
| T12 | Management access: SSH only, TACACS+ with local fallback | |
| T13 | Wi-Fi with wrong credentials | |
| T14 | Voice calls in London and to every branch | |
| T15 | Syslog and NTP | |
| T16 | Unsolicited inbound traffic from the internet | |
| T17 | Compromised DMZ host can't reach inside | |
| T18 | Rogue switch: BPDU guard | |
| T19 | Rogue switch: root guard as the second layer | |
| T20 | Loop guard on a blocked uplink | |
| T21 | Branch switch failure | |
| T22 | Toronto circuit failure | |
| T23 | Chicago backup VPN | |
| T24 | Dynamic ARP Inspection blocks an unbound static host | |
| T25 | BGP export and iBGP default filters | |
| T26 | QoS under congestion on a Toronto circuit | |

---

## T01: Inter-VLAN and inter-site policy matrix

**Purpose:** prove the ACL policy in [design-plan.md §10](design-plan.md#10-internal-security) is enforced.

**Procedure:**

1. Clear the ACL counters on the London cores and on every branch switch pair: `clear access-list counters` on LON-CORE-01/02, IST-SW-01/02, CHI-SW-01/02 and TOR-SW-01/02.
2. Run each check below from the source device's Command Prompt or Web Browser.
3. Record the hit counters (see **ACL counter checks** after the table). The branch rows are filtered on the switch that is HSRP active for the source VLAN: SW-01 for VLANs 20 and 60, SW-02 for voice VLAN 50 and guest VLAN 70. In London, LON-CORE-02 is active for VLANs 20 and 40. Confirm with `show standby brief` before reading counters.

| # | From | To | Check | Expected | Actual |
|---|---|---|---|---|---|
| 1 | LON-PC-ENG-01 | LON-PC-SALES-01 | ping | ✔ | |
| 2 | LON-PC-ENG-01 | LON-PC-FIN-01 | ping | ✘ | |
| 3 | LON-PC-ENG-01 | LON-MGMT-PC-01 (10.10.10.50) | ping | ✘ | |
| 4 | LON-PC-ENG-01 | 10.10.80.12 | ping | ✔ | |
| 5 | LON-PC-ENG-01 | intranet.corp.bosphorus.test | browse | ✔ | |
| 6 | LON-PC-ENG-01 | www.bosphorus.test | browse | ✔ (DMZ via split DNS) | |
| 7 | LON-PC-ENG-01 | www.example.test | browse | ✔ | |
| 8 | LON-PC-ENG-01 | IST-PC-01 | ping | ✔ | |
| 9 | LON-PC-FIN-01 | 10.10.80.13 | browse intranet | ✔ | |
| 10 | LON-PC-FIN-01 | LON-PC-SALES-01 | ping | ✘ | |
| 11 | LON-PC-FIN-01 | www.example.test | browse | ✔ | |
| 12 | LON-MGMT-PC-01 | LON-PC-FIN-01 | ping | ✔ | |
| 13 | LON-MGMT-PC-01 | 10.20.10.2 (IST-SW-01) | ssh | ✔ | |
| 14 | LON-TABLET-01 (guest) | www.example.test | browse | ✔ | |
| 15 | LON-TABLET-01 (guest) | 10.10.80.10 | ping | ✘ | |
| 16 | LON-LAPTOP-01 (corp) | intranet.corp.bosphorus.test | browse | ✔ | |
| 17 | IST-PC-01 | LON-PC-FIN-01 | ping | ✘ | |
| 18 | TOR-PC-01 | 10.10.80.13 | browse intranet | ✔ | |
| 19 | CHI-PC-01 | 10.10.80.13 | browse intranet | ✔ | |
| 20 | IST-TABLET-01 (branch guest) | www.example.test | browse | ✔ | |
| 21 | IST-TABLET-01 (branch guest) | 10.20.20.1 and 10.10.80.10 | ping | ✘ | |
| 22 | TOR-LAPTOP-01 (branch corp) | intranet.corp.bosphorus.test | browse | ✔ | |
| 23 | TOR-LAPTOP-01 (branch corp) | LON-PC-FIN-01 | ping | ✘ | |
| 24 | IST-PH-01 | LON-VOICE-01 / a London phone | register / call | ✔ | |
| 25 | IST-PH-01 | www.example.test | browse or ping | ✘ | |
| 26 | IST-PC-01 | 10.20.10.2 and 10.20.10.3 (own site's switch management) | ping, ssh | ✘ | |
| 27 | IST-PC-01 | 10.20.0.1 (IST-RTR-01 transit) | ping | ✘ | |
| 28 | TEST-PC on an Istanbul VLAN 50 port (IP phones can't originate pings) | 10.20.10.2 | ping | ✘ | |
| 29 | TOR-LAPTOP-01 (branch corp) | LON-PC-ENG-01 | ping | ✔ | |
| 30 | LON-PC-ENG-01 | TOR-LAPTOP-01 (branch corp) | ping | ✔ | |
| 31 | LON-LAPTOP-01 (corp) | TOR-LAPTOP-01 (branch corp) | ping | ✔ | |

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

---

## T02: Core switch failure

**Purpose:** gateway and Layer 2 failover when LON-CORE-01 dies.

**Procedure:**

1. On LON-PC-SALES-01 (VLAN 30, whose gateway is active on CORE-01): `ping -n 100 10.10.80.13`.
2. While it runs, power off LON-CORE-01 (Physical tab → power switch).
3. When the ping finishes, power CORE-01 back on and wait for it to boot (Fast Forward Time).

**Expected:** a few pings lost, then replies resume. On CORE-02 `show standby brief` shows **Active for all eight VLANs**; on LON-ACC-03 (where SALES-01 is connected) `show spanning-tree vlan 30` shows Gi0/2 as root port. After CORE-01 returns, preempt makes it Active again for 10/30/60/70.

**Evidence:** `T02-ping-loss.png`, `T02-standby-core02-after.png`, `T02-stp-acc01-after.png`.

**Actual:**

---

## T03: EtherChannel member failure

**Procedure:** on LON-PC-SALES-01 `ping -n 50 10.10.80.13`. This flow crosses Po1: VLAN 30's gateway is CORE-01, but the server VLAN's forwarding path to LON-SRV-SW01 is through CORE-02 (its STP root). While it runs, on LON-CORE-01: `interface GigabitEthernet1/0/23` → `shutdown`. Afterwards `no shutdown`.

**Expected:** no pings lost. `show etherchannel summary` shows Gi1/0/23 down and Po1 still `(SU)` with one member.

**Evidence:** `T03-etherchannel-one-member.png`. **Actual:**

---

## T04: ISP-A failure

**Purpose:** the key design test. Outbound, inbound and DMZ traffic all move to LON-EDGE-02 together, and stay symmetric.

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
| `show ip bgp summary` | EDGE-01 | ISP-A neighbor down; iBGP still up |
| `show ip route 0.0.0.0` | EDGE-01 | No default route; EDGE-02's default is rejected by `PL-IBGP-IN` |
| `show ip route 0.0.0.0` | CORE-01 | O*E2 via EDGE-02, metric 100 |
| Browse www.example.test, then `show ip nat translations` | LON-PC-ENG-01, then EDGE-02 | Works; translated to **203.0.113.2** |
| `show standby brief` | EDGE-02 | DMZ group 100 **Active** |
| `show ip bgp 203.0.113.0` | INET-CORE | Only path via ISP-B |
| Browse http://203.0.113.10 | INET-HOST-01 | Public site still loads |
| `show ip ospf neighbor` | IST-RTR-01 | Tunnel0 neighbor gone after the dead timer, Tunnel1 FULL |

**Restore:** `no shutdown` on ISP-A Fa0/0. Everything returns to EDGE-01 (DMZ HSRP preempts, OSPF default via EDGE-01 again).

**Evidence:** `T04-before-*.png`, `T04-after-*.png`. **Actual:**

---

## T05: Primary VPN tunnel failure

**Procedure:** on IST-PC-01 `tracert 10.10.80.13`. On LON-EDGE-01: `interface Tunnel0` → `shutdown`. On IST-RTR-01 `show ip route ospf`, then repeat the tracert. Restore with `no shutdown`.

**Expected:** before, the path goes via 10.255.1.1 (EDGE-01); after, via 10.255.1.5 (EDGE-02), and IST-RTR-01's routes point at Tunnel1.

**Evidence:** `T05-tracert-before.png`, `T05-tracert-after.png`, `T05-ist-routes-after.png`. **Actual:**

---

## T06: Chicago site-to-site VPN

**Procedure:** CHI-PC-01 `ping 10.10.80.10` (repeat if the first attempts time out). Then:

| Command | Where | Expected |
|---|---|---|
| `show crypto isakmp sa` | LON-EDGE-02 | Peer 198.51.100.14, QM_IDLE |
| `show crypto ipsec sa` | LON-EDGE-02 | Peer .14 encaps/decaps increasing |
| `show crypto isakmp sa`, `show crypto ipsec sa` | CHI-FW-01 | Peer 198.51.100.6, counters increasing |
| Ping CHI-PC-01 from LON-MGMT-PC-01 | London | Works (London-initiated through the tunnel) |

**Evidence:** `T06-edge02-ipsec.png`, `T06-asa-ipsec.png`. **Actual:**

---

## T07: Wrong CHAP password

**Procedure:** on TOR-RTR-01: `username LON-WAN-01 password Wrong-Chap-1`, then `interface Serial0/0/0` → `shutdown` → `no shutdown`. Optionally `debug ppp authentication` (turn off with `undebug all`).

**Expected:** `show interfaces serial0/0/0` → line protocol **down**; EIGRP neighbor gone; debug shows CHAP failure. Restore `username LON-WAN-01 password Lab-Chap-1` and bounce the interface.

**Evidence:** `T07-serial-down.png`. **Actual:**

---

## T08: OSPF authentication mismatch

**Procedure:** on LON-CORE-01: `interface GigabitEthernet1/0/20` → `ip ospf message-digest-key 1 md5 Wrong-Key` (remove the old key first: `no ip ospf message-digest-key 1`).

**Expected:** the adjacency with EDGE-01 drops after the dead timer (`show ip ospf neighbor`); traffic reroutes via CORE-02. Restore the correct key.

**Evidence:** `T08-ospf-neighbor-down.png`. **Actual:**

---

## T09: Port-security violation

**Procedure:**

1. On LON-ACC-03: `interface FastEthernet0/1` → `switchport port-security maximum 1`. The sticky MAC of LON-PC-SALES-01 is already learned.
2. Add a new PC (TEST-PC), disconnect LON-PC-SALES-01 from Fa0/1 and connect TEST-PC instead. Give TEST-PC a static 10.10.30.99 and ping 10.10.30.1.

**Expected:** pings fail; `show port-security interface fa0/1` shows the violation count increasing; a `%PORT_SECURITY-2-PSECURE_VIOLATION` message appears in the log and on LON-SRV-OPS-01.

**Restore:** reconnect LON-PC-SALES-01, delete TEST-PC, `switchport port-security maximum 3`.

**Evidence:** `T09-port-security.png`, `T09-syslog.png`. **Actual:**

---

## T10: Rogue DHCP server

**Procedure:** add a Server-PT named ROGUE-DHCP, static 10.10.20.254/24, DHCP service on with a pool for 10.10.20.0/24 whose gateway is 10.10.20.254. Connect it to LON-ACC-01 **Fa0/24** (the spare VLAN 20 desk port). On LON-PC-ENG-01, switch IP configuration to Static and back to DHCP to renew.

**Expected:** LON-PC-ENG-01 still gets a lease from 10.10.80.10 with gateway 10.10.20.1. `show ip dhcp snooping statistics` (or binding table) on LON-ACC-01 shows the rogue offers dropped on the untrusted port.

**Restore:** delete ROGUE-DHCP.

**Evidence:** `T10-pc-lease.png`, `T10-snooping.png`. **Actual:**

---

## T11: Guest Wi-Fi isolation

**Procedure:**

1. On LON-TABLET-01: browse `http://www.example.test`, `ping 10.10.80.10`, `ping 10.10.10.1`, `ping 172.16.100.10`.
2. On TOR-TABLET-01 (a branch guest whose internet goes through London): browse `http://www.example.test`, `ping 10.40.20.1`, `ping 10.10.80.10`.

**Expected:** browsing works at both sites; every ping to an internal address fails.

**Evidence:** `T11-guest-london.png`, `T11-guest-toronto.png`. **Actual:**

---

## T12: Management access

**Procedure** (from LON-MGMT-PC-01 unless stated):

1. `telnet 10.10.10.11` → refused.
2. `ssh -l netadmin 10.10.10.11`, password `Lab-Admin-1` (the local one) → **rejected**, because the TACACS+ server is up and has a different password.
3. Same with `Lab-Tacacs-User-1` → **accepted** (TACACS+).
4. On LON-SRV-AAA-01, turn the AAA service **off**. SSH again with `Lab-Admin-1` → **accepted** (local fallback).
5. From LON-PC-ENG-01: `ssh -l netadmin 10.10.10.11` → fails (VLAN 20 can't reach MGMT, and the VTY access-class only allows 10.10.10.0/24).

**Restore:** turn the AAA service back on.

**Evidence:** `T12-telnet-refused.png`, `T12-ssh-tacacs.png`, `T12-ssh-local-fallback.png`. **Actual:**

---

## T13: Wi-Fi with wrong credentials

**Procedure:** on LON-LAPTOP-01, change the `BSP-CORP` password to `Wrong-Wifi-1`.

**Expected:** the laptop doesn't associate and gets no address. Restore `Lab-Wifi-1`.

**Evidence:** `T13-wifi-rejected.png`. **Actual:**

---

## T14: Voice calls

**Procedure:** `show ephone registered` on LON-VOICE-01 to see which number each phone got. Call London → London (e.g. LON-PH-01 → LON-PH-02), then London → Istanbul, London → Chicago and London → Toronto. During the Toronto call, `show policy-map interface Serial0/1/0` on LON-WAN-01. Behaviour under congestion is tested separately in T26.

**Expected:** every call rings and connects; EF (voice) class counters increase on the Toronto circuit during that call.

**Evidence:** `T14-call-local.png`, `T14-call-branches.png`, `T14-qos-serial.png`. **Actual:**

---

## T15: Syslog and NTP

**Procedure:** on any IOS device `show ntp status` and `show clock`. On LON-SRV-OPS-01, open Services → SYSLOG. Generate a message (e.g. `shutdown`/`no shutdown` an unused port).

**Expected:** `Clock is synchronized`, stratum from 10.10.80.12; the syslog server shows the message with the device's IP and a millisecond timestamp.

**Evidence:** `T15-ntp.png`, `T15-syslog.png`. **Actual:**

---

## T16: Unsolicited inbound traffic

**Procedure** (from INET-HOST-01):

1. Browse `http://203.0.113.10` → works (published).
2. `ping 203.0.113.53` → fails (DNS server only publishes DNS).
3. `telnet 203.0.113.10` → fails (only web is published).
4. `ping 203.0.113.1` → fails (PAT address, nothing listening).

On LON-EDGE-01: `show policy-map type inspect zone-pair ZP-OUT-DMZ` (or `show zone-pair security`) shows drops in class-default.

**Evidence:** `T16-internet-host.png`, `T16-zbf-drops.png`. **Actual:**

---

## T17: Compromised DMZ host

**Procedure** (from LON-DMZ-WEB-01's Command Prompt): `ping 10.10.80.10`, `ping 10.10.10.50`, and browse `http://10.10.80.13`.

**Expected:** all fail. The DMZ → INSIDE policy only passes replies to connections opened from inside.

**Evidence:** `T17-dmz-blocked.png`. **Actual:**

---

## T18: Rogue switch vs BPDU guard

**Before:** `show spanning-tree root` on LON-ACC-01 and LON-CORE-01.

**Procedure:** cable LON-ROGUE-SW Gi0/1 → LON-ACC-01 Fa0/24 (copper cross-over).

**Expected:** Fa0/24 goes err-disabled immediately (`show interfaces status err-disabled`); `%SPANTREE-2-BLOCK_BPDUGUARD` appears in the log and on the syslog server; the root bridge is unchanged for every VLAN. With errdisable recovery configured, the port retries after 60 s and is disabled again while the rogue switch is attached.

**Restore:** remove the cable; the port recovers within 60 s (or `shutdown` / `no shutdown`).

**Evidence:** `T18-err-disabled.png`, `T18-syslog.png`, `T18-root-unchanged.png`. **Actual:**

---

## T19: Rogue switch vs root guard

**Purpose:** show the second layer of defence. BPDU guard is removed from the edge port on purpose.

**Procedure:**

1. On LON-ACC-01: `interface FastEthernet0/24` → `no spanning-tree bpduguard enable` → `no spanning-tree portfast`.
2. Cable LON-ROGUE-SW as in T18 and wait for STP to converge.

**Expected:** LON-ACC-01 accepts the rogue as root for VLAN 20 and sends superior BPDUs upstream, but the cores' root guard blocks them: `show spanning-tree inconsistentports` on LON-CORE-01 and LON-CORE-02 lists Gi1/0/1 as **Root Inconsistent** for VLAN 20. The cores stay root; the rogue only affects its own corner of the network.

**Restore:** remove the cable, re-enable `spanning-tree portfast` and `spanning-tree bpduguard enable` on Fa0/24.

**Evidence:** `T19-root-inconsistent-core01.png`. **Actual:**

---

## T20: Loop guard

**Purpose:** a blocked uplink that stops receiving BPDUs (as with a one-way link failure) must not start forwarding.

**Procedure:**

1. On LON-ACC-01, confirm `show spanning-tree vlan 10` shows Gi0/2 (to CORE-02) as **alternate/blocking** (VLAN 10's root is CORE-01).
2. Simulate the failure: on LON-CORE-02 `interface GigabitEthernet1/0/1` → `spanning-tree bpdufilter enable`. CORE-02 stops sending BPDUs on that link, but the link stays up.
3. Wait for the max-age timer (Fast Forward Time).

**Expected:** on LON-ACC-01, `show spanning-tree inconsistentports` shows Gi0/2 **Loop Inconsistent** instead of moving to forwarding.

**Restore:** `no spanning-tree bpdufilter enable` on CORE-02 Gi1/0/1; the port recovers automatically.

**Evidence:** `T20-loop-inconsistent.png`. **Actual:**

---

## T21: Branch switch failure

**Purpose:** a branch survives the loss of either switch of its pair.

**Procedure:** on IST-PC-02 (connected to IST-SW-02) run `ping -n 100 10.10.80.13`. While it runs, power off **IST-SW-01** (the active gateway for VLAN 20). Then renew IST-PC-02's DHCP lease and browse `http://www.example.test`.

**Expected:** a few pings lost while HSRP moves, then replies resume. On IST-SW-02, `show standby brief` shows Active for every VLAN. The renewed lease comes from IST-SW-02's upper-half scope (.176–.250). Browsing works through IST-SW-02's own router link. IST-PC-01 (on the failed switch) is offline, as expected.

**Restore:** power IST-SW-01 back on; it pre-empts VLANs 10, 20, 60.

**Evidence:** `T21-ping-loss.png`, `T21-standby-sw02.png`, `T21-dhcp-upper-half.png`. **Actual:**

---

## T22: Toronto circuit failure

**Procedure:** on TOR-PC-01 run `ping -n 100 10.10.80.13`. While it runs, on LON-WAN-01: `interface Serial0/1/0` → `shutdown`.

**Expected:** little or no loss. `show ip eigrp neighbors` on TOR-RTR-01 shows only the 10.255.2.5 neighbor on circuit 2; `show ip route 10.10.0.0` points at Serial0/0/1. Calls from TOR-PH-01 still work.

**Restore:** `no shutdown`; EIGRP load-shares across both circuits again.

**Evidence:** `T22-eigrp-one-circuit.png`. **Actual:**

---

## T23: Chicago backup VPN

**Procedure:** on LON-EDGE-02: `interface GigabitEthernet0/0/0` → `shutdown` (EDGE-02 loses ISP-B). Wait for the ASA to give up on the primary peer (use Fast Forward Time), then ping 10.10.80.10 from CHI-PC-01.

**Expected:**

| Check | Where | Expected |
|---|---|---|
| `show ip route 10.30.0.0` | LON-EDGE-02 | Tracked static gone |
| `show ip route 10.30.0.0` | LON-EDGE-01 | Floating static (AD 250) now installed |
| `show ip route 10.30.0.0` | LON-CORE-01 | O E2 via EDGE-01, metric 200 |
| `show crypto isakmp sa` | CHI-FW-01 | SA with 198.51.100.2 |
| Ping 10.10.80.10 | CHI-PC-01 | Works |

**Restore:** `no shutdown` on EDGE-02 Gi0/0/0. The ASA stays on EDGE-01 until that SA expires or is cleared (`clear crypto ikev1 sa` on the ASA), then returns to EDGE-02.

**Evidence:** `T23-edge01-floating-route.png`, `T23-asa-backup-peer.png`. **Actual:**

---

## T24: Dynamic ARP Inspection

**Purpose:** prove that DAI does more than appear in the running configuration.

**Procedure:** connect TEST-PC to an unused, untrusted VLAN 20 access port on LON-ACC-01. Give it a static address of `10.10.20.240/24`, gateway `10.10.20.1`, and try to ping the gateway. Then inspect `show ip arp inspection statistics` and `show ip arp inspection log` on LON-ACC-01.

**Expected:** the static host has no DHCP snooping binding, so its ARP is dropped and the ping fails; the DAI drop counter increases. A normal DHCP client on the same VLAN still works.

**Restore:** disconnect TEST-PC and return the port to its documented test/spare state.

**Evidence:** `T24-dai-drop.png`, `T24-dhcp-client-works.png`. **Actual:**

---

## T25: BGP filters

**Purpose:** prove that the enterprise cannot become internet transit and that an edge cannot retain its partner's default route over iBGP.

**Procedure:**

1. On both London edges, inspect `show ip bgp neighbors <ISP-address> advertised-routes`; only `203.0.113.0/24` should be exported.
2. On EDGE-01, temporarily add `ip route 10.10.0.0 255.255.0.0 Null0`, then add `network 10.10.0.0 mask 255.255.0.0` under BGP. Recheck advertised routes toward ISP-A.
3. Remove that temporary BGP `network` statement and Null0 route.
4. Repeat the ISP-A outage from T04 and inspect `show ip route 0.0.0.0` and `show ip bgp 0.0.0.0` on EDGE-01.

**Expected:** the private aggregate is never advertised because `PL-BGP-OUT` permits only `203.0.113.0/24`. During the outage, EDGE-01 has no default learned over iBGP because `PL-IBGP-IN` rejects it; EDGE-02 alone originates the enterprise default.

**Restore:** ensure ISP-A Fa0/0 is up and both temporary statements—`network 10.10.0.0 mask 255.255.0.0` and `ip route 10.10.0.0 255.255.0.0 Null0`—are absent.

**Evidence:** `T25-export-filter.png`, `T25-ibgp-default-filter.png`. **Actual:**

---

## T26: QoS under congestion

**Purpose:** show that `PM-WAN-OUT` protects voice when a WAN link is actually full, not just that the class counters move. Packet Tracer's queuing model is approximate, so treat the result as a demonstration of the policy's behaviour, not a measurement of latency or jitter.

**Prerequisites:** P0-17a/b passed (MQC marking and queuing accepted on the ISR2911 and ISR4331). If QoS was moved to `configs/_design-only/`, mark this test ❌ with that reason.

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

**Evidence:** `T26-congested-interface.png`, `T26-policy-map-drops.png`, `T26-call-during-congestion.png`, and `T26-no-policy-contrast.png` if step 7 was run. **Actual:**
