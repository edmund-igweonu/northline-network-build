# Day 5: Trunk to NL-HQ-R1 and inter-VLAN routing 
Ticket: NL-105

## What I built
On NL-HQ-SW1:
- Made fa0/1 a trunk and disabled DTP trunk negotiation on HQ switch
- Set up native VLAN 999 and allowed VLANs 10,20,99 across trunk
On NL-HQ-R1:
- Created VLANs 10,20,99 on HQ router
- Made FE0 trunk with native and allowed VLANs
- Created routing interfaces for each VLAN (first usable in each subnet)

## How I verified it
- show interfaces trunk - to verify native and allowed VLANs
- show ip interface brief - Vlan10, Vlan20 and Vlan99 all up/up
- ping from Acer on f0/2 to VLAN 99 interface (10.10.99.2) succeeded verifying inter-VLAN routing 
- ssh admin@10.10.99.1 (NL-HQ-R1) succeeded 
- tracepath 10.10.99.2
 1?: [LOCALHOST]                      pmtu 1500
 1:  10.10.10.1                                            1.023ms 
 1:  10.10.10.1                                            0.996ms 
 2:  10.10.99.2                                            3.667ms reached
     Resume: pmtu 1500 hops 2 back 2 

## What went wrong
- Cisco 881 router did not accept just adding the relevant VLANs to the trunk on FE0 and required adding 1-2 and 1002-1005 as well. [Command rejected: Bad VLAN allowed list. You have to include all default vlans, e.g. 1-2,1002-1005.] Added VLANs 1-2,1002-1005 with VLANs 10,20,99
- The switch side only allows 10,20,99 so the extra default VLANs on the router side never cross the trunk.
- Ubuntu SSH client turns off older encryption methods by default could not ssh into Cisco 881 router initially. Explicitly allowed SHA-1 key exchange for only 10.10.x and 10.20.x hosts, as older IOS does not support newer methods on my Acer.

## Why it works
- Set up native VLAN 999 to allow for untagged frames to be dropped into this by the trunk. VLAN 999 is unused so as to prevent malicious untagged frames from causing harm to the network, while also blocking double-tagging VLAN hopping attack. 
- Pinging VLAN 99 from VLAN 10 indicates inter-VLAN routing works as intended as messages for VLAN 99 are sent through the trunk to the router which then sends it back to the correct VLAN.


## Still to test
- SSH to NL-BR-R1 pending. Will be done once WAN link exists on Day 10.
