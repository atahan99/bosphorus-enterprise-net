# Architecture decision records

Short records of the significant design decisions: context, decision, alternatives and consequences.

| ADR | Decision |
|---|---|
| [0001](0001-active-standby-internet-edge.md) | Active/standby internet edge instead of load-sharing |
| [0002](0002-hsrp-not-vrrp.md) | HSRP instead of VRRP |
| [0003](0003-single-redistribution-point.md) | One EIGRP/OSPF redistribution point |
| [0004](0004-vtp-transparent.md) | VTP transparent mode |
| [0005](0005-lacp-core-only.md) | LACP for the core bundle, no bundling to access switches |
| [0006](0006-gre-over-ipsec-istanbul.md) | GRE over IPsec for Istanbul |
| [0007](0007-policy-based-ipsec-chicago.md) | Policy-based IPsec and static routing for Chicago |
| [0008](0008-internet-breakout.md) | Local internet breakout for Istanbul and Chicago; Toronto through London |
| [0009](0009-local-dhcp-istanbul.md) | Local DHCP at the branches |
| [0010](0010-lab-grade-management.md) | SNMP v2c and type-7 password encryption in the lab |
| [0011](0011-stateless-inside-dmz.md) | Stateless policy between INSIDE and DMZ |
| [0012](0012-static-printer-dai.md) | Static printer on a DAI-protected switch |
| [0013](0013-branch-wifi-autonomous-aps.md) | Autonomous APs with local switching for branch Wi-Fi |
| [0014](0014-branch-switch-pairs.md) | Redundant switch pairs at the branches |
