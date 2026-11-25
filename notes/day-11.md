# Day 11: Replace the static routes with OSPF 

Ticket NL-111

## Rollback plan (written after the change)

- What I would restore: the Day 10 static routes on both routers, then remove OSPF process.
- How I would confirm it: the sourced pings and the empty-then-full static table.

## Predictions

- OSPF is up and FULL, but the Day 10 statics are still in place. Which code (S or O) shows up in the routing table for 10.20.30.0/24 on NL-HQ-R1, and why? Do the OSPF routes show up anywhere at all? Answer: Static routes have a lower AD so they are prioritized in routing tables. Static AD is 1 and OSPF AD 110. This only matters when two sources offer the same prefix. show ip route ospf would show the ospf routes. (Partially correct)

The routes did not show up in routing table and sat in the LSDB until the static routes were gone. 

- Before you set the WAN link to point-to-point, which router would become DR on that segment, and what decides it? Answer: The NL-HQ-R1 will become the DR, this is because it will have the lower router-id. (Incorrect)
I did not capture this before switching to point-to-point. However, by rule, BR-R1 should have been the DR due to higher router ID.
The DR rule is: highest interface priority which is 1 by default, then the highest router ID. I clearly mixed up highest for lowest.

- NL-HQ-R1 uses process ID 1 and NL-BR-R1 uses process ID 50. Do they still become neighbors? No, the routers need matching process ID to form adjacency. **correction** I just remembered the process ID need not match for adjacency, its the OSPF area ID that is required to match. As well as subnet and mask on shared link, hello and dead timers, auth type and key, stub area flags and router-id's must be different. 
Not tested: configured process ID 1 on both routers, never verified on the lab. 

## What I built

- Configured OSPF on both routers with area 0.
- Set each router ID to the router's loopback ip address, i.e., 10.255.0.1 on NL-HQ-R1 and 10.255.0.2 on NL-BR-R1. 
- Network statements advertising every subnet on each router, i.e., HQ SVIs (VLANs 10, 20, and 99), Branch SVIs (VLANs 30 and 99), WAN transit subnet and both loopback (router-ID).
- Made every LAN facing SVI passive-interfaces, i.e., VLANs 10, 20, 30, 99 and loopbacks on both routers. 
- Configured the WAN link as point-to-point to bypass DR/BDR elections. 
- Removed static routes configured the day before. 

## How I verified it

- Confirmed the adjacency formed as FULL with show ip ospf neighbor 
```
NL-HQ-R1#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.2        0   FULL/  -        00:00:38    172.16.0.2      Vlan100
NL-HQ-R1#
```
```
NL-BR-R1#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.1        0   FULL/  -        00:00:36    172.16.0.1      Vlan100
NL-BR-R1#
```
- Confirmed router ID on NL-HQ-R1 with show ip protocols
```
  Router ID 10.255.0.1
```
- Ensured all LAN facing interfaces were passive with show ip protocols 
```
  Passive Interface(s):
    Loopback0
    Vlan10
    Vlan20
    Vlan99
```
```
  Passive Interface(s):
    Loopback0
    Vlan30
    Vlan99
```
and show ip ospf int vlan 10 on NL-HQ-R1 
```
    No Hellos (Passive interface) 
```
- Ensured static routes were replaced with OSPF routes with show ip route
```
NL-HQ-R1#sh ip route 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 10 subnets, 2 masks
C        10.10.10.0/24 is directly connected, Vlan10
L        10.10.10.1/32 is directly connected, Vlan10
C        10.10.20.0/24 is directly connected, Vlan20
L        10.10.20.1/32 is directly connected, Vlan20
C        10.10.99.0/24 is directly connected, Vlan99
L        10.10.99.1/32 is directly connected, Vlan99
O        10.20.30.0/24 [110/2] via 172.16.0.2, 00:45:19, Vlan100
O        10.20.99.0/24 [110/2] via 172.16.0.2, 00:44:54, Vlan100
C        10.255.0.1/32 is directly connected, Loopback0
O        10.255.0.2/32 [110/2] via 172.16.0.2, 00:44:31, Vlan100
      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.16.0.0/30 is directly connected, Vlan100
L        172.16.0.1/32 is directly connected, Vlan100
NL-HQ-R1#
```
```
NL-BR-R1#show ip route static 
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area 
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

NL-BR-R1#
```
- Verified the network statements covered necessary subnets on each router with show ip protocols 
```
  Routing for Networks:
    10.10.10.0 0.0.0.255 area 0
    10.10.20.0 0.0.0.255 area 0
    10.10.99.0 0.0.0.255 area 0
    10.255.0.1 0.0.0.0 area 0
    172.16.0.0 0.0.0.3 area 0
```
- Verified both routers could reach each other through OSPF by:
pinging Vlan30 SVI from HQ Vlan10
```
NL-HQ-R1#ping 10.20.30.1 source vlan 10
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.20.30.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.10.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms
NL-HQ-R1#
```
pinging Vlan20 SVI from Branch Vlan30
```
NL-BR-R1#ping 10.10.20.1 source vlan 30
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.20.1, timeout is 2 seconds:
Packet sent with a source address of 10.20.30.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms
NL-BR-R1#
```
traceroute from NL-BR-R1 sourced from VLAN 30 to 10.10.20.1
```
NL-BR-R1#traceroute 10.10.20.1 source vlan 30
Type escape sequence to abort.
Tracing the route to 10.10.20.1
VRF info: (vrf in name/id, vrf out name/id)
  1 172.16.0.1 4 msec *  0 msec
NL-BR-R1#
```
This shows 172.16.0.1 instead of 10.10.20.1 because the destination is an address on HQ-R1 itself and hence the router reached is the first and last one and it answers from the interface facing you. 


## What went wrong

- I initially tried to configure point-to-point on the routers interfaces using the ospfv3 command which did not affect the ospfv2 already being configured. 
- I then tried the ip ospf on Fa3 as opposed to the Vlan on which ospf was configured. 
- The Acer had no route to the branch until I added a route to the nl-mgmt. 
- I skipped the rollback plan before making the change.


## Why it works

- Neighbors form over the WAN SVI, exchange LSAs to build identical databases before running SPF and installing the best paths. 
- Making passive interfaces ensures no hello messages are sent while still advertising the specific subnet. 



## Still to test

- Test mismatched process IDs.
- DR/BDR election on a broadcast link. 
