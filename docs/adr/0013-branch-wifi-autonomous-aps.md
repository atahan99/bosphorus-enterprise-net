# ADR 0013: Autonomous APs with local switching for branch Wi-Fi

**Status:** Accepted

## Context

Every site needs corporate and guest Wi-Fi. London has a wireless controller (WLC-3504) with lightweight APs. The branches (30–36 staff) connect to London over VPNs and two 2 Mb/s circuits.

## Decision

Each branch gets **autonomous APs (AccessPoint-PT)**, one per SSID and one on each of the two site switches, that bridge `BSP-CORP` into the site's VLAN 60 and `BSP-GUEST` into VLAN 70. Traffic is switched locally; the branch's own gateway enforces guest isolation, and corporate clients join with a pre-shared key (see Consequences).

## Alternatives considered

- **Lightweight APs joined to the London WLC, central switching:** one place to manage, but every branch Wi-Fi packet (including guests) would hairpin through London over the VPN or circuit, and a WAN outage would take branch Wi-Fi down with it.
- **FlexConnect (local switching, central management):** the production answer. Not available in Packet Tracer.
- **A WLC per branch:** far too much for 30–50 people.

## Consequences

- Branch Wi-Fi works during a WAN outage.
- APs are managed one by one. Production would use FlexConnect or cloud-managed APs.
- AccessPoint-PT carries one SSID each, so losing a branch switch loses one of the two SSIDs. Production would use dual-SSID APs, one per switch, so either switch failure leaves both SSIDs up.
- The two APs per branch and the switch-port limit of ten learned wireless MACs are sufficient for the representative lab clients, not the full headcount. A production RF survey would determine the AP count and client limits.
- AccessPoint-PT has no RADIUS option, so the branch corporate SSID uses WPA2-PSK (`Lab-Corp-Wifi-1`), recorded as B-14 in the support matrix. In production these would be WPA2-Enterprise APs managed centrally.
