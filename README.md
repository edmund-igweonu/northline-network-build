# northline-network-build

A 30-day build of a small enterprise network for a fictional company, Northline Systems, with a head office (HQ) and one branch. It runs on real Cisco and TP-Link hardware, with the ISP simulated in GNS3. Each day is worked as a ticket (NL-1xx) with predictions, verification and a write-up of what went wrong.

## Topology

```
  HQ                                                         Branch

  NL-HQ-SW1 Fa0/1 ==trunk 10,20,99==  Fa0 NL-HQ-R1 Fa3 --VLAN 100 WAN-- Fa3 NL-BR-R1 Fa0  ==trunk 30,99== port 1 NL-BR-SW1
                                          Fa4                                Fa4
                                           |                                  |
  NL-HQ-SW1 Fa0/21 (VLAN 901) -------------+                                  |
  NL-HQ-SW1 Fa0/22 (VLAN 902) ------------------------------------------------+
  NL-HQ-SW1 Fa0/23 ==trunk 901,902== Acer USB adapter -- GNS3 ISP (c2691)
  NL-HQ-SW1 Fa0/24 (VLAN 99) ------- Acer enp1s0 (NOC mgmt)
```

## Devices

| Device | Hardware | Role |
|--------|----------|------|
| NL-HQ-R1 | Cisco 881 | HQ router: inter-VLAN routing, DHCP, OSPF, HQ ISP circuit |
| NL-HQ-SW1 | Cisco Catalyst 2950 | HQ access switch, also carries both ISP VLANs to the Acer |
| NL-BR-R1 | Cisco 881 | Branch router: inter-VLAN routing, DHCP, OSPF, branch ISP circuit |
| NL-BR-SW1 | TP-Link TL-SG105E | Branch access switch (web managed) |
| ISP | Cisco c2691 (Dynamips in GNS3) | Simulated ISP for both sites |

## Addressing

| Site | VLAN | Name | Subnet | Gateway |
|------|------|------|--------|---------|
| HQ | 10 | USERS | 10.10.10.0/24 | 10.10.10.1 |
| HQ | 20 | SERVERS | 10.10.20.0/24 | 10.10.20.1 |
| HQ | 99 | MGMT | 10.10.99.0/24 | 10.10.99.1 (NL-HQ-SW1 is .2) |
| Branch | 30 | USERS | 10.20.30.0/24 | 10.20.30.1 |
| Branch | 99 | MGMT | 10.20.99.0/24 | 10.20.99.1 |
| Both | 999 | PARKING | none | Native VLAN on trunks and unused ports |

| Link | Subnet | Addresses |
|------|--------|-----------|
| HQ to branch WAN (VLAN 100) | 172.16.0.0/30 | NL-HQ-R1 .1, NL-BR-R1 .2 |
| HQ ISP circuit (VLAN 901) | 203.0.113.0/30 | ISP .1, NL-HQ-R1 .2 |
| Branch ISP circuit (VLAN 902) | 203.0.113.4/30 | ISP .5, NL-BR-R1 .6 |
| Loopbacks / OSPF router IDs | /32 | NL-HQ-R1 10.255.0.1, NL-BR-R1 10.255.0.2 |
| Simulated internet | 198.51.100.1/32 | ISP Loopback0 |

DHCP pools on each router exclude .1 to .20 for gateways and static devices. OSPF runs in area 0 between the two routers over VLAN 100, with the LAN SVIs, loopbacks and Fa4 passive.

## Design notes

- Fate-sharing: the branch's ISP circuit physically runs through NL-HQ-SW1 at HQ, so if that switch fails both sites lose internet at the same time. A real branch would have its own local circuit. I accepted this as a lab compromise to save cables and Acer NICs.
- Management and the ISP trunk use separate links on the Acer (VLAN 99 on Fa0/24, ISP VLANs on Fa0/23 to the USB adapter), so changes to the ISP trunk can't cut off my management session.
- The TL-SG105E's management page answers from every port regardless of VLAN membership (Days 8 and 9). It's mitigated with a strong password for now.

## Daily notes

| Day | Ticket | Topic |
|-----|--------|-------|
| [1](notes/day-01.md) | NL-101 | Wipe gear and create repo |
| [2](notes/day-02.md) | NL-102 | Base config and passwords on all Cisco devices |
| [3](notes/day-03.md) | NL-103 | SSH only with local admin |
| [4](notes/day-04.md) | NL-104 | HQ VLANs and access ports on NL-HQ-SW1 |
| [5](notes/day-05.md) | NL-105 | Trunk to NL-HQ-R1 and inter-VLAN routing |
| [6](notes/day-06.md) | NL-106 | DHCP for HQ users and servers |
| [7](notes/day-07.md) | NL-107 | Break-fix: VLAN 10 DHCP pool exhausted |
| [8](notes/day-08.md) | NL-108 | Branch VLANs and tagged uplink on NL-BR-SW1 |
| [9](notes/day-09.md) | NL-109 | Branch trunk, SVIs and DHCP on NL-BR-R1 |
| [10](notes/day-10.md) | NL-110 | Private WAN link and static routes |
| [11](notes/day-11.md) | NL-111 | Replace static routes with OSPF area 0 |
| [12](notes/day-12.md) | NL-112 | ISP circuits in GNS3 via VLANs 901 and 902 |

## Configs

Running configs are in [configs/](configs/), with passwords and secrets removed. NL-BR-SW1.cfg is the TP-Link binary backup and can only be read by restoring it to the switch. Its VLAN and PVID tables are written out in the [Day 8 notes](notes/day-08.md).
