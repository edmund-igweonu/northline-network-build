# Day 10: Private WAN link between the routers. Static routes. Verify HQ to Branch reachability.

Ticket: NL-110

## Predictions

- Suppose you only add routes on NL-HQ-R1. You run a plain ping from NL-HQ-R1 to 10.20.30.1 with no source specified. Does it work? What source address does the router use, and why does that matter? Answer: The pings will not work, even though the destination address is stated correctly, the router will send pings out but will not get response. NL-BR-R1 cannot reach 10.20.30.0/24 subnet without a static route. (Incorrect)
The ping works. HQ source address will be the interface the echo request goes out of, which is VLAN100 at 172.16.0.1. This works because 172.16.0.0/30 is directly connected so the branch can reply w/o static routes.

- Same half-built state, but you ping from NL-HQ-R1 sourced from its VLAN 10 SVI. Does it work, and where exactly does it fail if not? Answer: This will not work because on the other router, there is no routes for the ICMP echo reply to return from NL-BR-R1 to NL-HQ-R1. (Correct)

- How many static routes will each router need in total? Answer: NL-HQ-R1 will need 4 routes and NL-BR-R1 will need 5 routes. (Incorrect)
The actual number of static routes were 3 and 4 respectively. I counted the WAN link but 172.16.0.0/30 is already directly connected on both routers, so no static routes needed.


## What I built

- Direct link between NL-HQ-R1 and NL-BR-R1.
- Configured unique transit VLAN 100 with same VLAN ID on both routers and description.
- Configured ip addresses 172.16.0.1 on HQ and 172.16.0.2 on BR on the transit VLANs and loopbacks 10.255.0.1 and 10.255.0.2 on both respectively. 
- Added static routes so every HQ subnet can reach every branch subnet and vice versa, 3 on HQ and 4 on the branch.

## How I verified it

- Confirmed pings on each HQ SVI could reach every branch SVI successfully. 
```
NL-HQ-R1#ping 10.20.99.1 source vlan 20
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.20.99.1, timeout is 2 seconds:
Packet sent with a source address of 10.10.20.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms

NL-BR-R1#ping 10.10.20.1 source vlan 30   
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.10.20.1, timeout is 2 seconds:
Packet sent with a source address of 10.20.30.1 
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/4 ms
```
- Confirmed routes were installed successfully. show ip route 
e.g, NL-HQ-R1
```
S        10.20.30.0/24 [1/0] via 172.16.0.2
S        10.20.99.0/24 [1/0] via 172.16.0.2
C        10.255.0.1/32 is directly connected, Loopback0
S        10.255.0.2/32 [1/0] via 172.16.0.2
      172.16.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.16.0.0/30 is directly connected, Vlan100
L        172.16.0.1/32 is directly connected, Vlan100
```
- Traceroute from NL-HQ-R1 to Vlan30 on NL-BR-R1 replies as 172.16.0.2 because a router sources replies from the exit interface and the * comes from ICMP rate limiting.
```
NL-HQ-R1#traceroute 10.20.30.1        
Type escape sequence to abort.
Tracing the route to 10.20.30.1
VRF info: (vrf in name/id, vrf out name/id)
  1 172.16.0.2 0 msec *  0 msec
NL-HQ-R1#
```
- Pings from a host on Vlan30 on the branch can reach Vlan20 on HQ after adding temporary route 10.10.0.0/16 via 10.20.30.1, because nl-dhcp is never-default. 
```
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ ping -c 4 10.10.20.1
traceroute -n 10.10.20.1
PING 10.10.20.1 (10.10.20.1) 56(84) bytes of data.
64 bytes from 10.10.20.1: icmp_seq=1 ttl=254 time=1.12 ms
64 bytes from 10.10.20.1: icmp_seq=2 ttl=254 time=0.990 ms
64 bytes from 10.10.20.1: icmp_seq=3 ttl=254 time=0.634 ms
64 bytes from 10.10.20.1: icmp_seq=4 ttl=254 time=1.00 ms

--- 10.10.20.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3021ms
rtt min/avg/max/mdev = 0.634/0.936/1.121/0.182 ms
traceroute to 10.10.20.1 (10.10.20.1), 30 hops max, 60 byte packets
 1  10.20.30.1  0.687 ms  0.933 ms  1.207 ms
 2  172.16.0.1  1.380 ms * *
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ 
```

## What went wrong

- Routes were configured on both routers to each others SVIs but did not show up after entering show ip route. 
- I verified the routes were actually configured in the running-config with show run | include ip route. 
```

NL-HQ-R1#show run | include ip route
ip route 10.20.30.0 255.255.255.0 172.16.0.2
ip route 10.20.99.0 255.255.255.0 172.16.0.2
ip route 10.255.0.2 255.255.255.255 172.16.0.2
NL-HQ-R1#
```
- Fa3 was still in VLAN 1, so VLAN 100 had no up port and hence the Vlan100 SVI stayed down; meaning there was no connected route to 172.16.0.0/30. I corrected this with "switchport access vlan 100" on int Fa3.
- That made the next hop unreachable and IOS will not install the routes.
- The branch router's clock reset again, so DHCP failed on Acer until I set it.
- The router traceroute failed because NL-BR-SW1 was unplugged, so Fa0 went down and Vlan30 SVI went down with it. 

## Why it works

- Each router only knows its connected subnets until you tell it about the remote ones.
- Static routes tell the router to "reach this subnet through this next hop", and the next hop has to be reachable through a connected route. 
- Traffic needs routes in both directions. The request needs a route there, and the reply needs a route back.

## Still to test

- Summary routes on both routers to replace the multiple routes on each. 10.10.0.0/16 covers all three HQ subnets on the branch side and 10.20.0.0/16 covers all branch subnets on the HQ side. 
- Routes to loopback interfaces to remain since 10.255.0.x falls outside of 10.10.0.0/16 and 10.20.0.0/16.
- NTP to be configured after day 12.
