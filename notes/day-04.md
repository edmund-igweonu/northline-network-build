# Day 4: HQ VLANs and access ports on NL-HQ-SW1
Ticket: NL-104

## What I built
- set up VTP to transparent mode 
- set up VLANs on NL-HQ-SW1 including User, Servers, Mgmt. PARKING 
- VLAN 1 interfaces (shutdown) and a description on every port
- Put access ports in access mode using range command and enabled Portfast (allows for faster convergence on access ports, going straight to forwarding and skipping listening and learning states)
- Set up Mgmt interface ip, default gateway on the 2950 and config acer NIC with a ip in same subnet 

## How I verified it
- show vlan br - to ensure all access ports are in the right VLANs
- show int status - to ensure unused ports are disabled
- show vtp status - to ensure transparent mode is active 
- ping from Acer on f0/24 to VLAN 99 interface (10.10.99.2) succeeded

## What went wrong
- The 2950 does not have k9 image; no SSH. Thus only Telnet for now. 

## Why it works
- Transparent mode allows vtp messages to pass through the devices without sharing its VLANs and thus has its VLANs saved in the running-config


## Still to test
- Lock down Telnet to the management VLAN with an ACL (Day 15).
