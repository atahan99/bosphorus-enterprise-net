# Server configuration (GUI)

Server-PT devices are configured in the GUI, so this file is their "config". Work through each server in order. On every server, **turn off every service that isn't listed** for it (Services tab → each service → Off). That's part of the hardening story.

Addresses are in [addressing-plan.md](../../docs/addressing-plan.md). Credentials are in [lab-credentials.md](../../docs/lab-credentials.md).

---

## LON-SRV-CORE-01: DHCP and internal DNS

**Desktop → IP Configuration:** Static, 10.10.80.10 / 255.255.255.0, gateway 10.10.80.1, DNS 10.10.80.10.

### Services → DHCP (On)

The default pool `serverPool` can't be deleted. Leave it alone: its default gateway stays 0.0.0.0, so no relayed request ever matches it. **Add** the pools below, including a separate `VLAN10-MGMT` pool for the APs (in the Phase 4 build this was scripted, and the scripting API can't set the WLC option on an existing pool). Every pool: subnet mask 255.255.255.0, maximum users 151 (addresses .100–.250). The branches run their own DHCP on their switch pairs (in their configs), so only London's scopes live here.

| Pool name | Default gateway | DNS server | Start IP | TFTP server | WLC address |
|---|---|---|---|---|---|
| VLAN10-MGMT | 10.10.10.1 | 10.10.80.10 | 10.10.10.100 | — | **10.10.10.20** |
| VLAN20-ENG | 10.10.20.1 | 10.10.80.10 | 10.10.20.100 | — | — |
| VLAN30-SALES | 10.10.30.1 | 10.10.80.10 | 10.10.30.100 | — | — |
| VLAN40-FIN-HR | 10.10.40.1 | 10.10.80.10 | 10.10.40.100 | — | — |
| VLAN50-VOICE | 10.10.50.1 | 10.10.80.10 | 10.10.50.100 | **10.10.50.10** | — |
| VLAN60-WIFI-CORP | 10.10.60.1 | 10.10.80.10 | 10.10.60.100 | — | — |
| VLAN70-WIFI-GUEST | 10.10.70.1 | **192.0.2.53** | 10.10.70.100 | — | — |

The server picks the pool whose gateway matches the relaying SVI's subnet, so the gateway field must be exactly right. Guests get the public DNS server so they never query internal DNS.

### Services → DNS (On)

| Name | Type | Address | Why |
|---|---|---|---|
| intranet.corp.bosphorus.test | A | 10.10.80.13 | Intranet |
| mail.corp.bosphorus.test | A | 10.10.80.13 | Mail |
| cme.corp.bosphorus.test | A | 10.10.50.10 | Call manager |
| www.bosphorus.test | A | 172.16.100.10 | **Split DNS:** inside users reach the DMZ web server directly |
| www.example.test | A | 192.0.2.10 | External site (Packet Tracer DNS can't forward, so external names are added here) |

---

## LON-SRV-AAA-01: TACACS+ and RADIUS

**IP:** 10.10.80.11 / 255.255.255.0, gateway 10.10.80.1, DNS 10.10.80.10.

### Services → AAA (On)

**Network configuration (clients).** Packet Tracer has no `ip tacacs source-interface` (matrix B-01), so each device's TACACS+ request comes from the interface it leaves by. Register that address. Where a second path exists, add the backup-path address as another client so logins still work after a failover.

| Client name | Client IP | Also add (backup path) | Secret | Type |
|---|---|---|---|---|
| LON-EDGE-01 | 10.255.0.1 | — | Lab-Tacacs-1 | Tacacs |
| LON-EDGE-02 | 10.255.0.5 | — | Lab-Tacacs-1 | Tacacs |
| LON-CORE-01 | 10.10.80.2 | — | Lab-Tacacs-1 | Tacacs |
| LON-CORE-02 | 10.10.80.3 | — | Lab-Tacacs-1 | Tacacs |
| LON-WAN-01 | 10.255.0.14 | 10.255.0.18 | Lab-Tacacs-1 | Tacacs |
| IST-RTR-01 | 10.255.1.2 | 10.255.1.6 | Lab-Tacacs-1 | Tacacs |
| TOR-RTR-01 | 10.255.2.2 | 10.255.2.6 | Lab-Tacacs-1 | Tacacs |
| LON-ACC-01 … LON-ACC-04 | 10.10.10.11 … 10.10.10.14 (one entry each) | — | Lab-Tacacs-1 | Tacacs |
| LON-SRV-SW01 | 10.10.10.15 | — | Lab-Tacacs-1 | Tacacs |
| LON-VOICE-01 | 10.10.50.10 | — | Lab-Tacacs-1 | Tacacs |
| IST-SW-01, IST-SW-02 | 10.20.0.2, 10.20.0.6 | — | Lab-Tacacs-1 | Tacacs |
| TOR-SW-01, TOR-SW-02 | 10.40.0.2, 10.40.0.6 | — | Lab-Tacacs-1 | Tacacs |
| CHI-SW-01, CHI-SW-02 | 10.30.0.2, 10.30.0.6 | 10.30.0.9, 10.30.0.10 (VLAN 900) | Lab-Tacacs-1 | Tacacs |
| LON-WLC-01 | 10.10.10.20 | — | Lab-Radius-1 | Radius |

If a login shows `% Login invalid` although the password is right, the server answered from a client IP it doesn't know. IOS only falls back to the local user when the server is unreachable, not when it rejects, so every config also has a console-only local list (`aaa authentication login CONSOLE local`) as a way back in.

LON-DMZ-SW01 and CHI-FW-01 use local accounts only (see their configs).

**Branch APs (RADIUS):** AccessPoint-PT has no management IP, so there's no fixed client address to register. Add one Radius client per branch using the AP's **gateway-side** subnet guess first (e.g. `IST-AP-01` → 10.20.60.2, `CHI-AP-01` → 10.30.60.2, `TOR-AP-01` → 10.40.60.2, the SW-01 address in VLAN 60, secret `Lab-Radius-1`). If branch WPA2-Enterprise still fails, switch the branch corporate SSID to WPA2-PSK (`Lab-Corp-Wifi-1`) and record it as B-14 in the support matrix.

**User setup:**

| Username | Password | Used by |
|---|---|---|
| netadmin | Lab-Tacacs-User-1 | Device logins via TACACS+ |
| wifi-user1 | Lab-Wifi-1 | `BSP-CORP` wireless (laptop) |
| wifi-user2 | Lab-Wifi-2 | `BSP-CORP` wireless (smartphone) |
| wifi-user3 | Lab-Wifi-3 | `BSP-CORP` wireless (branch laptops) |

The TACACS+ password is deliberately different from the local fallback password (`Lab-Admin-1`), so test T12 can prove which one authenticated you.

---

## LON-SRV-OPS-01: Syslog, NTP, TFTP

**IP:** 10.10.80.12 / 255.255.255.0, gateway 10.10.80.1, DNS 10.10.80.10.

| Service | Setting |
|---|---|
| SYSLOG | On |
| NTP | On. **Authentication: Enable**, Key 1, Password `Lab-Ntp-1`. Set the date and time to the real current time |
| TFTP | On (config backups land here, see build guide Phase 10) |

---

## LON-SRV-INTRA-01: intranet and mail

**IP:** 10.10.80.13 / 255.255.255.0, gateway 10.10.80.1, DNS 10.10.80.10.

| Service | Setting |
|---|---|
| HTTP / HTTPS | On. Edit `index.html`: replace the body with `<h1>Intranet</h1><p>corp.bosphorus.test - internal only</p>` |
| EMAIL | On. Domain name `corp.bosphorus.test`. Users: `alice` / `Lab-Mail-1`, `bob` / `Lab-Mail-2` |

---

## LON-DMZ-WEB-01: public website

**IP:** 172.16.100.10 / 255.255.255.0, gateway **172.16.100.1** (the DMZ HSRP address), DNS 172.16.100.53.

| Service | Setting |
|---|---|
| HTTP / HTTPS | On. `index.html`: `<h1>www.bosphorus.test</h1><p>Public site, served from the London DMZ.</p>` |

## LON-DMZ-DNS-01: public DNS

**IP:** 172.16.100.53 / 255.255.255.0, gateway 172.16.100.1.

| Service | Setting |
|---|---|
| DNS | On. `www.bosphorus.test` A `203.0.113.10` |

---

## Internet servers

| Server | IP / gateway | Services |
|---|---|---|
| INET-WEB-01 | 192.0.2.10 / 24, gw 192.0.2.1, DNS 192.0.2.53 | HTTP on, `index.html`: `<h1>www.example.test</h1><p>Somewhere on the internet.</p>` |
| INET-DNS-01 | 192.0.2.53 / 24, gw 192.0.2.1 | DNS on: `www.example.test` A `192.0.2.10`, `www.bosphorus.test` A `203.0.113.10` |

INET-DNS-01 holds the public `www.bosphorus.test` record because Packet Tracer DNS can't delegate to the company's own DNS server. The DMZ DNS server is still reachable at 203.0.113.53 (test with `nslookup www.bosphorus.test 203.0.113.53` from INET-HOST-01).
