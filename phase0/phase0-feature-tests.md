# Phase 0: Packet Tracer feature validation

Before building the real topology, confirm which "needs testing" features Packet Tracer 9 actually supports. This avoids designing around a command that turns out to be `% Invalid input` halfway through the build.

**Time:** 2–3 hours. **Output:** every row in [`docs/pt-support-matrix.md`](../docs/pt-support-matrix.md) marked ✅ / ⚠️ / ❌, `phase0/phase0-scratch.pkt` saved, and labeled evidence captured in `phase0/evidence/`.

## How to run it

1. Build the scratch topology in §1 (9 devices).
2. Work through the tests in order. Each test has **commands to paste** and a **pass check**.
3. For each test, record the result in the support matrix:
   - ✅ all commands accepted and the pass check succeeds
   - ⚠️ accepted but doesn't behave as expected, or only a fallback syntax works
   - ❌ command rejected
4. When a command is rejected, copy the exact command and the `% Invalid input` line into the matrix's Notes column. If you get stuck, paste the output back to Claude.

Paste config in **global configuration mode** (`enable` → `configure terminal`) unless a block starts with `enable`. Credentials are lab-only.

> **Script/MCP CLI note:** explicitly send `exit` after every interface, router, crypto-policy or other configuration submode before entering the next one. Packet Tracer's script-driven CLI can otherwise keep later commands in the previous submode without an obvious failure; during this run that initially put multiple interface addresses on the first interface. Manual GUI/CLI pastes should also preserve the `exit` lines shown in the production configs.

## 1. Scratch topology

```
                     [R3 Router-PT] "ISP", AS 65001
                     Fa0/0 /       \ Fa1/0
                         /         \
            Gi0/0/0 [R1 ISR4331]─────[R2 ISR4331] Gi0/0/0
               AS 65100    Gi0/0/1  Gi0/0/1   AS 65100
                    Gi0/0/2 |              | Gi0/0/2
                  Gi1/0/24  |              | Gi0/0
                   [SW1 3650-24PS]     [R4 ISR2911]  EIGRP 100
                    Gi1/0/1 |    \ Gi1/0/2
                     Gi0/1  |    [PC2]
                    [SW2 2960-24TT]
                    Fa0/1 |    \ Fa0/2
                       [PC1]   [SRV1 Server-PT]
```

### Devices

| Device | Model | Prep |
|---|---|---|
| R1, R2 | ISR4331 | None |
| R3 | Router-PT | None |
| R4 | ISR2911 | None |
| SW1 | 3650-24PS | Add AC-POWER-SUPPLY |
| SW2 | 2960-24TT | None |
| PC1, PC2 | PC-PT | None |
| SRV1 | Server-PT | None |

### Cabling

| A | Port | B | Port | Cable |
|---|---|---|---|---|
| R1 | Gi0/0/0 | R3 | Fa0/0 | Cross-over |
| R2 | Gi0/0/0 | R3 | Fa1/0 | Cross-over |
| R1 | Gi0/0/1 | R2 | Gi0/0/1 | Cross-over |
| R2 | Gi0/0/2 | R4 | Gi0/0 | Cross-over |
| R1 | Gi0/0/2 | SW1 | Gi1/0/24 | Straight |
| SW1 | Gi1/0/1 | SW2 | Gi0/1 | Cross-over |
| SW1 | Gi1/0/2 | PC2 | Fa0 | Straight |
| SW2 | Fa0/1 | PC1 | Fa0 | Straight |
| SW2 | Fa0/2 | SRV1 | Fa0 | Straight |

### Addressing

| Device | Interface | Address |
|---|---|---|
| R1 | Gi0/0/0 | 198.51.100.2/30 |
| | Gi0/0/1 | 10.0.12.1/30 |
| | Gi0/0/2 | 10.0.1.1/30 |
| | Loopback0 | 1.1.1.1/32 |
| R2 | Gi0/0/0 | 198.51.100.6/30 |
| | Gi0/0/1 | 10.0.12.2/30 |
| | Gi0/0/2 | 10.0.24.1/30 |
| | Loopback0 | 2.2.2.2/32 |
| | Loopback10 / 11 | 10.20.20.1/24, 10.20.21.1/24 |
| R3 | Fa0/0 | 198.51.100.1/30 |
| | Fa1/0 | 198.51.100.5/30 |
| | Loopback0 | 3.3.3.3/32 |
| R4 | Gi0/0 | 10.0.24.2/30 |
| | Loopback0 | 4.4.4.4/32 |
| | Loopback1 | 10.40.20.1/24 |
| SW1 | Gi1/0/24 (routed) | 10.0.1.2/30 |
| | Vlan20 | 10.10.20.2/24 (HSRP VIP .1) |
| SW2 | Vlan20 | 10.10.20.3/24 |
| SRV1 | Fa0 | 10.10.20.10/24, gateway 10.10.20.1 |
| PC1 | Fa0 | DHCP (from SW1) |
| PC2 | Fa0 | 10.10.20.20/24, gateway 10.10.20.1 |

## 2. Base configuration

### Step 0: security license on R1, R2, R4

Crypto and ZBF need it. Do this first because it requires a reload.

R1 and R2 (ISR4331):

```
enable
configure terminal
license boot level securityk9
end
write memory
reload
```

R4 (ISR2911):

```
enable
configure terminal
license boot module c2900 technology-package securityk9
end
write memory
reload
```

Check after reload: `show version` lists `securityk9` as active. **→ P0-01**

### R1

```
hostname R1
no ip domain-lookup
interface GigabitEthernet0/0/0
 ip address 198.51.100.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/0/1
 ip address 10.0.12.1 255.255.255.252
 no shutdown
interface GigabitEthernet0/0/2
 ip address 10.0.1.1 255.255.255.252
 no shutdown
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
ip route 10.10.20.0 255.255.255.0 10.0.1.2
```

### R2

```
hostname R2
no ip domain-lookup
interface GigabitEthernet0/0/0
 ip address 198.51.100.6 255.255.255.252
 no shutdown
interface GigabitEthernet0/0/1
 ip address 10.0.12.2 255.255.255.252
 no shutdown
interface GigabitEthernet0/0/2
 ip address 10.0.24.1 255.255.255.252
 no shutdown
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
interface Loopback10
 ip address 10.20.20.1 255.255.255.0
interface Loopback11
 ip address 10.20.21.1 255.255.255.0
```

### R3

```
hostname R3
no ip domain-lookup
interface FastEthernet0/0
 ip address 198.51.100.1 255.255.255.252
 no shutdown
interface FastEthernet1/0
 ip address 198.51.100.5 255.255.255.252
 no shutdown
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
```

### R4

```
hostname R4
no ip domain-lookup
interface GigabitEthernet0/0
 ip address 10.0.24.2 255.255.255.252
 no shutdown
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
interface Loopback1
 ip address 10.40.20.1 255.255.255.0
```

### SW1

```
hostname SW1
no ip domain-lookup
ip routing
vlan 20
 name ENG
interface GigabitEthernet1/0/24
 no switchport
 ip address 10.0.1.2 255.255.255.252
 no shutdown
interface GigabitEthernet1/0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
interface GigabitEthernet1/0/2
 switchport mode access
 switchport access vlan 20
interface Vlan20
 ip address 10.10.20.2 255.255.255.0
 no shutdown
ip route 0.0.0.0 0.0.0.0 10.0.1.1
ip dhcp excluded-address 10.10.20.1 10.10.20.99
ip dhcp pool VLAN20
 network 10.10.20.0 255.255.255.0
 default-router 10.10.20.1
 dns-server 10.10.20.10
```

If `switchport trunk encapsulation dot1q` is rejected, skip it (the 3650 may only support dot1q).

### SW2

```
hostname SW2
no ip domain-lookup
vlan 20
 name ENG
interface GigabitEthernet0/1
 switchport mode trunk
interface range FastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
interface Vlan20
 ip address 10.10.20.3 255.255.255.0
 no shutdown
ip default-gateway 10.10.20.1
```

### Hosts

- **SRV1:** Desktop → IP Configuration: 10.10.20.10 / 255.255.255.0, gateway 10.10.20.1. Services → DHCP: **off**.
- **PC2:** static 10.10.20.20 / 255.255.255.0, gateway 10.10.20.1.
- **PC1:** leave on DHCP for now (tests P0-04 and P0-05 use it).

**Sanity check before testing:** R1 pings R2 (10.0.12.2), R3 (198.51.100.1) and SW1 (10.0.1.2); R2 pings R4 (10.0.24.2); PC2 pings SRV1. SW1 answers on 10.10.20.1 only once HSRP is configured in P0-07, so test the gateway after that.

## 3. Switching tests

### P0-02: Root guard and loop guard

SW1:

```
interface GigabitEthernet1/0/1
 spanning-tree guard root
```

SW2:

```
spanning-tree mode rapid-pvst
interface GigabitEthernet0/1
 spanning-tree guard loop
```

Also try the global form on SW2 (record separately if only one works):

```
spanning-tree loopguard default
```

**Pass:** commands accepted; `show spanning-tree interface GigabitEthernet0/1 detail` on SW2 mentions loop guard, and `show running-config interface Gi1/0/1` on SW1 shows root guard. Repeat the `spanning-tree guard loop` line on an SW1 port to check the 3650 as well.

### P0-03: Err-disable recovery for BPDU guard

SW2:

```
errdisable recovery cause bpduguard
errdisable recovery interval 30
```

**Pass:** accepted; `show errdisable recovery` lists bpduguard as enabled.

### P0-04: DHCP snooping with option 82 disabled

SW2:

```
ip dhcp snooping
ip dhcp snooping vlan 20
no ip dhcp snooping information option
interface GigabitEthernet0/1
 ip dhcp snooping trust
```

On PC1: Desktop → IP Configuration → DHCP.

**Pass:** PC1 gets an address in 10.10.20.100–.250; `show ip dhcp snooping binding` on SW2 lists PC1's MAC. If PC1 gets no address, remove the `no ... information option` line, retry, and note the difference (it helps explain the gotcha in the blog).

### P0-05: Dynamic ARP Inspection

SW2 (after P0-04 works):

```
ip arp inspection vlan 20
interface GigabitEthernet0/1
 ip arp inspection trust
```

**Pass, part 1:** PC1 (DHCP address) can still ping 10.10.20.2.
**Pass, part 2:** set PC1 to a static 10.10.20.99; its pings now fail, and `show ip arp inspection statistics` (or `show ip arp inspection vlan 20`) shows drops. Set PC1 back to DHCP afterwards.

**Part 3 (static hosts):** SRV1 has a static address and no snooping binding, so DAI now drops its ARP too. Try allowing it with an ARP ACL, which is how the real build handles the static printer:

```
arp access-list STATIC-HOSTS
 permit ip host 10.10.20.10 mac host <SRV1 MAC in xxxx.xxxx.xxxx form>
ip arp inspection filter STATIC-HOSTS vlan 20
```

Record whether it's accepted and whether PC2 can ping SRV1 again. **Then remove DAI** (`no ip arp inspection vlan 20`) so it can't interfere with P0-06.

### P0-06: Wired 802.1X

SRV1 → Services → AAA: service **On**. Add a network client: Client Name `SW2`, Client IP `10.10.20.3`, Secret `Lab-Radius-1`, Server Type **Radius**. Add a user: `dot1x-user` / `Lab-User-1`.

SW2:

```
aaa new-model
aaa authentication login default local
username admin privilege 15 secret Lab-Admin-1
radius-server host 10.10.20.10 key Lab-Radius-1
aaa authentication dot1x default group radius
dot1x system-auth-control
interface FastEthernet0/1
 authentication port-control auto
 dot1x pae authenticator
```

If `radius-server host` is rejected, try the newer form:

```
radius server SRV1
 address ipv4 10.10.20.10 auth-port 1812 acct-port 1813
 key Lab-Radius-1
```

SW2's console now asks for a login: `admin` / `Lab-Admin-1`.

PC1 → Config → FastEthernet0 → 802.1X: enable, method MD5, user `dot1x-user`, password `Lab-User-1`.

**Pass:** PC1 is blocked before authenticating and gets DHCP after; `show dot1x all` (or `show authentication sessions`) shows the port authorised. With a wrong password, the port stays unauthorised.

Afterwards, disable 802.1X on Fa0/1 (`no authentication port-control auto`) so it doesn't interfere with later tests.

### P0-07: HSRP v2 with MD5 and tracking on the 3650

SW1:

```
track 1 interface GigabitEthernet1/0/24 line-protocol
interface Vlan20
 standby version 2
 standby 20 ip 10.10.20.1
 standby 20 priority 110
 standby 20 preempt
 standby 20 authentication md5 key-string Lab-Hsrp-1
 standby 20 track 1 decrement 20
```

If `track 1 interface ...` or `standby 20 track 1 decrement 20` is rejected, try the older interface-tracking syntax:

```
interface Vlan20
 standby 20 track GigabitEthernet1/0/24 20
```

**Pass:** `show standby brief` shows Active with priority 110. Shut Gi1/0/24 and priority drops to 90 (`show standby`). Bring it back up. PC2 can now ping 10.10.20.1.

## 4. Routing tests

### P0-08: OSPF with MD5, point-to-point, reference bandwidth and area range

R1:

```
router ospf 1
 router-id 1.1.1.1
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface GigabitEthernet0/0/1
 area 0 authentication message-digest
 network 10.0.12.0 0.0.0.3 area 0
 network 1.1.1.1 0.0.0.0 area 0
interface GigabitEthernet0/0/1
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 Lab-Ospf-1
```

R2:

```
router ospf 1
 router-id 2.2.2.2
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface GigabitEthernet0/0/1
 area 0 authentication message-digest
 area 1 authentication message-digest
 area 1 range 10.20.0.0 255.255.0.0
 network 10.0.12.0 0.0.0.3 area 0
 network 2.2.2.2 0.0.0.0 area 0
 network 10.20.20.0 0.0.0.255 area 1
 network 10.20.21.0 0.0.0.255 area 1
interface GigabitEthernet0/0/1
 ip ospf network point-to-point
 ip ospf message-digest-key 1 md5 Lab-Ospf-1
```

**Pass:** `show ip ospf neighbor` on R1 shows R2 FULL; `show ip route ospf` on R1 contains `O IA 10.20.0.0/16` and **not** the two /24s. Then change R1's key to `Wrong-Key`: the adjacency drops (useful preview of test T08). Change it back.

### P0-09: EIGRP with MD5 key chain

R2:

```
key chain EIGRP-KEYS
 key 1
  key-string Lab-Eigrp-1
router eigrp 100
 no auto-summary
 network 10.0.24.0 0.0.0.3
interface GigabitEthernet0/0/2
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

R4:

```
key chain EIGRP-KEYS
 key 1
  key-string Lab-Eigrp-1
router eigrp 100
 no auto-summary
 network 10.0.24.0 0.0.0.3
 network 10.40.20.0 0.0.0.255
 passive-interface Loopback1
interface GigabitEthernet0/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP-KEYS
```

**Pass:** `show ip eigrp neighbors` on R2 lists R4; R2 has `D 10.40.20.0/24`. Change one key-string: the neighbor drops. Change it back.

### P0-10: Redistribution with route tags

R2:

```
route-map EIGRP-TO-OSPF deny 10
 match tag 110
route-map EIGRP-TO-OSPF permit 20
 set tag 100
route-map OSPF-TO-EIGRP deny 10
 match tag 100
route-map OSPF-TO-EIGRP permit 20
 set tag 110
router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP-TO-OSPF
router eigrp 100
 redistribute ospf 1 metric 1000000 10 255 1 1500 route-map OSPF-TO-EIGRP
```

**Pass:**
- R1: `show ip route 10.40.20.0` → learned via OSPF as **E2**, and the detail shows **tag 100** (or `show ip ospf database external` shows tag 100).
- R4: `show ip route` contains `D EX 1.1.1.1/32`.

If the route-maps are rejected but plain redistribution works, record ⚠️: the design then uses untagged redistribution at a single point.

### P0-11: BGP on Router-PT, iBGP, prefix-list, prepend, passwords

R3 (Router-PT, the "ISP"):

```
router bgp 65001
 bgp router-id 3.3.3.3
 network 3.3.3.3 mask 255.255.255.255
 neighbor 198.51.100.2 remote-as 65100
 neighbor 198.51.100.2 password Lab-Bgp-1
 neighbor 198.51.100.2 default-originate
 neighbor 198.51.100.6 remote-as 65100
 neighbor 198.51.100.6 password Lab-Bgp-1
 neighbor 198.51.100.6 default-originate
```

R1:

```
ip route 203.0.113.0 255.255.255.0 Null0
ip prefix-list PL-OUT seq 5 permit 203.0.113.0/24
ip prefix-list PL-IBGP-IN seq 5 deny 0.0.0.0/0
ip prefix-list PL-IBGP-IN seq 10 permit 3.3.3.3/32
router bgp 65100
 bgp router-id 1.1.1.1
 network 203.0.113.0 mask 255.255.255.0
 neighbor 198.51.100.1 remote-as 65001
 neighbor 198.51.100.1 password Lab-Bgp-1
 neighbor 198.51.100.1 prefix-list PL-OUT out
 neighbor 2.2.2.2 remote-as 65100
 neighbor 2.2.2.2 update-source Loopback0
 neighbor 2.2.2.2 next-hop-self
 neighbor 2.2.2.2 prefix-list PL-IBGP-IN in
```

R2:

```
ip route 203.0.113.0 255.255.255.0 Null0
ip prefix-list PL-OUT seq 5 permit 203.0.113.0/24
ip prefix-list PL-IBGP-IN seq 5 deny 0.0.0.0/0
ip prefix-list PL-IBGP-IN seq 10 permit 3.3.3.3/32
route-map RM-PREPEND permit 10
 set as-path prepend 65100 65100
router bgp 65100
 bgp router-id 2.2.2.2
 network 203.0.113.0 mask 255.255.255.0
 neighbor 198.51.100.5 remote-as 65001
 neighbor 198.51.100.5 password Lab-Bgp-1
 neighbor 198.51.100.5 prefix-list PL-OUT out
 neighbor 198.51.100.5 route-map RM-PREPEND out
 neighbor 1.1.1.1 remote-as 65100
 neighbor 1.1.1.1 update-source Loopback0
 neighbor 1.1.1.1 next-hop-self
 neighbor 1.1.1.1 prefix-list PL-IBGP-IN in
```

Wait about a minute (or use Fast Forward Time), then check:

| Check | Where | Expected |
|---|---|---|
| eBGP up on Router-PT | R3 `show ip bgp summary` | Both neighbors with a prefix count (not Idle/Active) |
| iBGP between loopbacks | R1 `show ip bgp summary` | 2.2.2.2 established |
| Prefix-list works | R3 `show ip bgp` | Only 203.0.113.0/24 received from London, nothing else |
| Prepend works | R3 `show ip bgp` | Path via 198.51.100.6 shows `65100 65100 65100`; best path (`>`) via 198.51.100.2 |
| default-originate | R1 `show ip route` | `B* 0.0.0.0/0` via 198.51.100.1 |
| iBGP default filter | Shut R1 Gi0/0/0, then R1 `show ip route 0.0.0.0` | No default remains through R2; `PL-IBGP-IN` rejects it. Restore `no shutdown` |
| Password | Change R1's password to `Wrong-Bgp` | Session to R3 drops; restore it |

Record each row separately in the matrix (P0-11a to P0-11g). If Router-PT can't do BGP at all, place an ISR4331 in R3's position and re-run; the design then uses ISR4331 + NIM-ES2-4 for the ISP routers.

### P0-12: OSPF default-information originate with metric

R1:

```
router ospf 1
 default-information originate metric 10
```

**Pass:** R2 `show ip ospf database external` shows a 0.0.0.0 LSA from 1.1.1.1 with metric 10.

### P0-13: IP SLA and object tracking

R1:

```
ip sla 1
 icmp-echo 198.51.100.1 source-interface GigabitEthernet0/0/0
 frequency 10
ip sla schedule 1 life forever start-time now
track 10 ip sla 1 reachability
ip route 3.3.3.3 255.255.255.255 198.51.100.1 track 10
```

**Pass:** `show ip sla statistics` shows successes; `show track 10` shows Up; the static route to 3.3.3.3 is in the table. Shut R3 Fa0/0: the track goes Down and the route disappears. Restore it.

Not required for v1.0 (the design uses BGP/OSPF for edge failover), but if it works it's a cleaner way to track ISP reachability for DMZ HSRP.

## 5. Security and VPN tests

### P0-14: GRE over IPsec with strong IKE settings

Both routers need to reach each other's public address through R3. R1:

```
ip route 198.51.100.6 255.255.255.255 198.51.100.1
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key Lab-Ike-1 address 198.51.100.6
crypto ipsec transform-set TS-AES256 esp-aes 256 esp-sha256-hmac
 mode transport
access-list 110 permit gre host 198.51.100.2 host 198.51.100.6
crypto map CM-WAN 10 ipsec-isakmp
 set peer 198.51.100.6
 set transform-set TS-AES256
 match address 110
interface GigabitEthernet0/0/0
 crypto map CM-WAN
interface Tunnel0
 ip address 10.255.1.1 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.6
```

R2: mirror image.

```
ip route 198.51.100.2 255.255.255.255 198.51.100.5
crypto isakmp policy 10
 encryption aes 256
 hash sha256
 authentication pre-share
 group 14
 lifetime 86400
crypto isakmp key Lab-Ike-1 address 198.51.100.2
crypto ipsec transform-set TS-AES256 esp-aes 256 esp-sha256-hmac
 mode transport
access-list 110 permit gre host 198.51.100.6 host 198.51.100.2
crypto map CM-WAN 10 ipsec-isakmp
 set peer 198.51.100.2
 set transform-set TS-AES256
 match address 110
interface GigabitEthernet0/0/0
 crypto map CM-WAN
interface Tunnel0
 ip address 10.255.1.2 255.255.255.252
 tunnel source GigabitEthernet0/0/0
 tunnel destination 198.51.100.2
```

**Record each option separately** (P0-14a to P0-14d). If rejected, use the fallback and note it:

| Option | Fallback |
|---|---|
| `hash sha256` | `hash sha` |
| `group 14` | `group 5`, then `group 2` |
| `esp-sha256-hmac` | `esp-sha-hmac` |
| `mode transport` | remove the line (tunnel mode) |

**Pass:** from R1, `ping 10.255.1.2` succeeds (the first ping or two may fail while IKE negotiates). `show crypto isakmp sa` shows QM_IDLE; `show crypto ipsec sa` shows encaps/decaps counters increasing.

### P0-15: NAT and ZBF on the same router as crypto

R1:

```
interface GigabitEthernet0/0/2
 ip nat inside
interface GigabitEthernet0/0/0
 ip nat outside
ip access-list extended NAT-ACL
 deny ip 10.0.0.0 0.255.255.255 10.0.0.0 0.255.255.255
 permit ip 10.0.0.0 0.255.255.255 any
ip nat inside source list NAT-ACL interface GigabitEthernet0/0/0 overload

class-map type inspect match-any CM-INSIDE-OUT
 match protocol http
 match protocol https
 match protocol dns
 match protocol icmp
policy-map type inspect PM-INSIDE-OUT
 class type inspect CM-INSIDE-OUT
  inspect
 class class-default
  drop
zone security INSIDE
zone security OUTSIDE
zone-pair security ZP-IN-OUT source INSIDE destination OUTSIDE
 service-policy type inspect PM-INSIDE-OUT
interface GigabitEthernet0/0/2
 zone-member security INSIDE
interface GigabitEthernet0/0/1
 zone-member security INSIDE
interface Tunnel0
 zone-member security INSIDE
interface GigabitEthernet0/0/0
 zone-member security OUTSIDE
```

**Pass (all four needed for ✅):**

1. PC2 pings 3.3.3.3 (inspected ICMP through NAT).
2. R1 `show ip nat translations` shows PC2's 10.10.20.20 translated to 198.51.100.2.
3. R1 `show policy-map type inspect zone-pair sessions` shows the session (or the command is accepted with counters).
4. The Tunnel0 ping from P0-14 **still works** and the BGP session to R3 stays up (the self zone isn't blocking IKE, ESP, GRE or BGP).

If the named NAT ACL is rejected, use `access-list 1 permit 10.0.0.0 0.255.255.255` with `ip nat inside source list 1 ...` and note it.

### P0-16: Management-plane hardening commands

R4:

```
ip domain-name corp.bosphorus.test
username admin privilege 15 secret Lab-Admin-1
crypto key generate rsa general-keys modulus 2048
ip ssh version 2
aaa new-model
tacacs-server host 10.10.20.10 key Lab-Tacacs-1
aaa authentication login default group tacacs+ local
login block-for 120 attempts 3 within 60
ntp authentication-key 1 md5 Lab-Ntp-1
ntp authenticate
ntp trusted-key 1
ntp server 10.10.20.10 key 1
access-list 99 permit 10.10.10.0 0.0.0.255
snmp-server community BSP-RO RO 99
logging host 10.10.20.10
service timestamps log datetime msec
line vty 0 4
 transport input ssh
 access-class 99 in
 exec-timeout 10 0
```

If `tacacs-server host` is rejected, try the newer form:

```
tacacs server SRV1
 address ipv4 10.10.20.10
 key Lab-Tacacs-1
```

**Pass:** every line accepted (R4 can't actually reach the server; this test only checks syntax). Record any rejected line.

From now on R4's console asks for a login. The TACACS+ server is unreachable, so it falls back to the local user: `admin` / `Lab-Admin-1`. That fallback working is itself worth noting in the matrix.

## 6. QoS test

### P0-17: MQC classification, marking and queuing

R4 (ISR2911):

```
ip access-list extended ACL-VOICE
 permit udp any any range 16384 32767
class-map match-any CM-MARK-VOICE
 match access-group name ACL-VOICE
policy-map PM-LAN-IN
 class CM-MARK-VOICE
  set ip dscp ef
class-map match-any CM-VOICE
 match ip dscp ef
class-map match-any CM-SIGNALING
 match ip dscp cs3
policy-map PM-WAN-OUT
 class CM-VOICE
  priority percent 10
 class CM-SIGNALING
  bandwidth percent 5
 class class-default
  fair-queue
interface GigabitEthernet0/0
 service-policy input PM-LAN-IN
 service-policy output PM-WAN-OUT
```

Repeat the `PM-WAN-OUT` part on R1 Gi0/0/1 to check the ISR4331.

**Pass:** accepted; `show policy-map interface GigabitEthernet0/0` shows the classes with counters. Record marking (input policy) and queuing (output policy) separately, and the 2911 and the 4331 separately.

## 7. Items verified during the build

These are known to work in Packet Tracer and are confirmed in the build phase that uses them, rather than here:

| Feature | Confirmed in phase |
|---|---|
| ASA 5506-X site-to-site IPsec (IKEv1 crypto map) with an IOS peer | 6 |
| PPP with CHAP on serial | 6 |
| WLC-3504 with LAPs, WPA2-Enterprise via RADIUS | 8 |
| CME on a 2811 with 7960 phones | 8 |
| LACP, VTP transparent, Rapid PVST+, port security, HSRP on ISR4331 | 3–5 |

## 8. Deferred (stretch goals)

Not tested in Phase 0: NetFlow, Easy VPN remote access, IPv6/OSPFv3, Network Controller API. These are tested if and when the stretch goals are picked up.

## 9. When you're done

1. Save the scratch lab as `phase0/phase0-scratch.pkt`.
2. Fill in every row in [`docs/pt-support-matrix.md`](../docs/pt-support-matrix.md).
3. Send the matrix (or just the ❌ and ⚠️ rows) back to Claude. The final device configs are written against the confirmed feature set.
