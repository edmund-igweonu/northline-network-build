# Day 13: Internet access with default routes and NAT overload (ticket NL-113)

## Rollback plan:
- Remove ip nat inside and outside configurations from all interfaces.
- Delete ACLs with permitted subnets. 
- Remove static default routes and OSPF default advertisement. 
- Delete global ip nat config command. 
- Check NAT statistics to ensure no mappings and routing table to match Day 12.

## Predictions:
- Once both routers advertise a default into OSPF, what will NL-HQ-R1 show as its gateway of last resort? Will the branch's OSPF default appear in HQ's routing table, its OSPF database, or both? Why?
0.0.0.0 via 203.0.113.1 will be the gateway of last resort on HQ. The branch default will not appear in HQ's routing table and not on its OSPF database because there will be no network statement on the branch router advertising that. (Partially correct)

Default-information originate injects default routes creating an external LSA and that floods across the whole OSPF domain. The branch's OSPF default will not appear in HQ's table because IOS prioritizes static routes (AD 1) over OSPF routes (AD 110). The branch default does appear in HQ's OSPF database, as the Type-5 output under How I verified it shows.


- An HQ user at 10.10.10.x pings 198.51.100.1. What source address does the ISP router see? When the reply comes back, how does NL-HQ-R1 know which inside host it belongs to?
The source address ISP router sees is 203.0.113.2:PORT. A reply to that user will be identified using the port number. (Mostly correct)

TCP and UDP sessions are tracked by port to identify the user but ICMP is by its id.
```
Pro Inside global         Inside local          Outside local         Outside global
udp 203.0.113.2:41641     10.10.10.24:41641     5.161.218.233:3478    5.161.218.233:3478
```

- You stop the ISP node in GNS3 and leave all the cabling in place. Does HQ's static default stay in the routing table? What happens to HQ users' internet traffic? (Think about what Fa4 is actually plugged into.) 
The static default will be lost because IOS only includes routes to link that are reachable by the router. HQ will lose access to the internet. (Partially correct)

The static default remains even with ISP node down. The default advertised by each router into OSPF remains but HQ and branch users lose connectivity to the internet. Fa4 on HQ and branch respectively plugs into the 2950's Fa0/21 and Fa0/22 ports which stay up regardless of GNS3 running or not. So Fa4 stays up/up, the connected route to 203.0.113.0/30  and the static default stays. IOS never checks if 203.0.113.1 or 203.0.113.5 is actually alive.

Fa4 on HQ-R1
```
FastEthernet4              203.0.113.2     YES NVRAM  up                    up      
```
Fa4 on BR-R1
```
FastEthernet4              203.0.113.6     YES NVRAM  up                    up      
```
Default route on HQ-R1
```
S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 10 subnets, 2 masks
```
Default route on BR-R1
```
S*    0.0.0.0/0 [1/0] via 203.0.113.5
      10.0.0.0/8 is variably subnetted, 9 subnets, 2 masks
```
## What I built:
- Configured static default route to each router's ISP i.e., HQ to 203.0.113.1 and branch to 203.0.113.5. 
- Advertised each routers default route conditionally into OSPF.
- Configured PAT for users from VLAN 10, 20 on HQ and 30 on branch (Vlan30 is configured but not verified) as inside interfaces to translate to own Fa4 address as outside interface. Excluded Mgmt VLANs (99 on both), WAN addresses (172.16.0.0/30) and loopbacks from translation.
- Left the ISP router untouched with no routes to 10.0.0.0/8 or 172.16.0.0/12.

## How I verified it:
- Confirmed PAT with show ip nat translations. Some entries were my Mac's background Apple traffic, because the default route pointed at Northline. Hence port 5223 showed up. 
```
Pro Inside global         Inside local          Outside local         Outside global
tcp 203.0.113.2:49606     10.10.10.24:49606     17.57.144.10:5223     17.57.144.10:5223
tcp 203.0.113.2:49607     10.10.10.24:49607     17.57.144.12:5223     17.57.144.12:5223
```

- Vlan20 host traffic translated the same way. 
```
tcp 203.0.113.2:49665     10.10.20.21:49665     185.34.3.207:443      185.34.3.207:443
tcp 203.0.113.2:49666     10.10.20.21:49666     162.248.221.215:443   162.248.221.215:443
```

- Verified accurate inside (Vlan10 and 20) and outside (Fa4) interfaces mapping "inside source" using NAT-INSIDE with show ip nat statistics.
```
NL-HQ-R1#show ip nat statistics 
Total active translations: 1995 (0 static, 1995 dynamic; 1995 extended)
Peak translations: 2280, occurred 00:08:28 ago
Outside interfaces:
  FastEthernet4
Inside interfaces: 
  Vlan10, Vlan20
Hits: 38582  Misses: 0
CEF Translated packets: 20287, CEF Punted packets: 18275
Expired translations: 4606
Dynamic mappings:
-- Inside Source
[Id: 4] access-list NAT-INSIDE interface FastEthernet4 refcount 1995

Total doors: 0
Appl doors: 0
Normal doors: 0
Queued Packets: 0
```

- Ensured HQ-to-branch traffic is not translated, after a successful ping from Vlan10 to Vlan30 did not show an entry on translation table. Traffic goes in Vlan10 and out Vlan100 with no NAT role so the packets never cross from inside to outside.

- Verified Vlan99 does not get internet access by failed pings to 198.51.100.1 from an Acer terminal through nl-mgmt with source ip 10.10.99.10. The router forwarded it, but it was not translated because no ACL match (capture shows 10.10.99.10 leaving untranslated) and the ISP had no route back.
```
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ ping -c 4 198.51.100.1
PING 198.51.100.1 (198.51.100.1) 56(84) bytes of data.

--- 198.51.100.1 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3074ms
```
```
14:08:14.272107 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.99.10 > 198.51.100.1: ICMP echo request, id 9344, seq 3, length 64
14:08:15.295440 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.99.10 > 198.51.100.1: ICMP echo request, id 9344, seq 4, length 64
```

- Confirmed default route on both routers with show ip route.
NL-HQ-R1
```
Gateway of last resort is 203.0.113.1 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.1
      10.0.0.0/8 is variably subnetted, 10 subnets, 2 masks
```
NL-BR-R1
```
Gateway of last resort is 203.0.113.5 to network 0.0.0.0

S*    0.0.0.0/0 [1/0] via 203.0.113.5
      10.0.0.0/8 is variably subnetted, 9 subnets, 2 masks
```

- Verified HQ and branch routers injected default routes into OSPF with show ip ospf database.
```
		Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
0.0.0.0         10.255.0.1      1604        0x80000001 0x00DACC 1
0.0.0.0         10.255.0.2      1419        0x80000001 0x00D4D1 1
```

## What went wrong:
- Router-sourced pings were not translated. A ping from NL-HQ-R1 sourced from Vlan10 left Fa4 with its original 10.10.10.1 source and got no reply.
- I expected the source addresses to be translated to Fa4 address but that did not happen.
```
11:51:04.390705 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.10.1 > 203.0.113.1: ICMP echo request, id 4, seq 2, length 80
11:51:06.390533 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.10.1 > 203.0.113.1: ICMP echo request, id 4, seq 3, length 80
```

- Pings generated from the router itself never actually enter the inside interface and the ISP has no route back to the 10.x address. 
- A plain ping from NL-HQ-R1 used Fa4's own address and got replies.
```
11:52:59.496001 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.2 > 203.0.113.1: ICMP echo request, id 5, seq 4, length 80
11:52:59.525003 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:e8, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.1 > 203.0.113.2: ICMP echo reply, id 5, seq 4, length 80
```

- IOS accepted the command as a different kind of NAT rule that ignores the inside and outside role. 
- I initially configured "ip nat source list NAT-INSIDE int Fa4 overload". This command creates a NAT Virtual Interface rule which does not use the inside and outside interface roles. 
```
12:06:56.101801 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.10.24 > 198.51.100.1: ICMP echo request, id 54835, seq 2, length 64
12:06:57.105945 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 10.10.10.24 > 198.51.100.1: ICMP echo request, id 54835, seq 3, length 64
```

- I found this because the statistics showed "Outside Destination" with zero hits and misses.
```
Hits: 0  Misses: 0
CEF Translated packets: 0, CEF Punted packets: 0
Expired translations: 0
Dynamic mappings:
-- Outside Destination
[Id: 2] access-list NAT-INSIDE interface FastEthernet4 refcount 0
```

- I fixed this by removing the earlier ip nat command and issuing "ip nat inside source list NAT-INSIDE interface Fa4 overload".
```
12:15:03.545458 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.2 > 198.51.100.1: ICMP echo request, id 32308, seq 3, length 64
12:15:03.568331 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:e8, ethertype 802.1Q (0x8100), length 102: vlan 901, p 0, ethertype IPv4 (0x0800), 198.51.100.1 > 203.0.113.2: ICMP echo reply, id 32308, seq 3, length 64
```

## Why it works:
- Port address translation allows hosts on a private network to access the internet using a single outward facing address.
- Users on VLANs 10, 20 at HQ and VLAN 30 at branch have their DHCP issued addresses translated to Fa4 address on each router, depending on the source. 
- PAT tracks sessions using the inside local plus port or id which is mapped to inside global plus port or id.
- HQ to branch traffic is not translated as it never crosses from an inside interface to an outside one. 
- Each router prefers its own static default over the OSPF one because of lower AD.
- Each router only advertises a default route while the router has its own configured otherwise there is no default route to inject into OSPF domain. 


## Still to test:
- Branch NAT (NL-BR-R1's NAT statistics, plus VLAN 30 ping with a capture).
