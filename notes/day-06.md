# Day 6: DHCP for HQ users and servers on NL-HQ-R1
Ticket: NL-106

## What I built
On NL-HQ-R1:
- Excluded IP addresses in VLAN 10 and 20 from .1 to .20 for gateways and static devices
- Created a pool for USERS and SERVERS with defined default gateway, DNS server, domain name and lease time

## How I verified it
- show ip dhcp pool - showed the pool configured
- show run | section dhcp - showed the network address, default-router, dns-server, domain-name and lease time 
- show ip dhcp binding - showed the Acer's MAC with an expiry date of tomorrow
- the Acer got 10.10.10.29 on VLAN 10 and 10.10.20.21 in VLAN 20, with the gateway, DNS and domain from DHCP details


## What went wrong
- DHCP timed out on the Acer
- tcpdump on the Acer showed the Offer arriving but no ACK
- The router debug said "Sending DHCPACK", so the ACK was lost after the router decided to send it
- The binding's 1900 expire was the clue: the clock sat just before the Feb 2036 rollover, so the lease expiry wrapped to 1900
- Fixed it by entering privileged exec mode to correct the clock with the "clock set" command and verified using show clock
- The saved profiles replaced the vanishing manual addresses

## Why it works
- Excludes addresses that may be needed for default gateways and static devices
- Creates a pool of address for devices in specific VLANs, matching the subnet of the interface that the Discover arrived on
- Discover broadcast messages sent by a host on a network looking for an IP
- Offer messages sent by a router suggesting an IP address from a particular subnet
- Request message sent back to the router by the host to ask for particular IP
- Acknowledge message sent by router confirming the IP lease for that device and the router maps its MAC address to that specific IP on its DHCP binding table


## Still to test
- Router clock likely to reset post-reboot so will have to set up NTP
