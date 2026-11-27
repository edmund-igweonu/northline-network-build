# Day 12: ISP circuits in GNS3 (ticket NL-112)

## Predictions:
- After you cable and address Fa4 but before the ISP node exists, what state will Fa4 show on each router, and why? up/up, as Fa4 has a live link to an enabled 2950 port and the ISP router does not need to exist for that.

- In that same state, you ping 203.0.113.1 from NL-HQ-R1 while capturing on the Acer. What will you see in the capture, if anything, and what will the tag be? (Did not test in this lab)

- With everything built, can NL-HQ-R1 ping 203.0.113.6? Explain using what is and isn't in its routing table. No, the pings will not get through as the routing table does not have routes covering 203.0.113.4/30. (Correct)
```
NL-HQ-R1#ping 203.0.113.6
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 203.0.113.6, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```
```
NL-HQ-R1#sh ip route 203.0.113.6
% Subnet not in table
```

## What I built:
- Created two ISP VLANs on NL-HQ-SW1 (901 and 902 for ISP-HQ and ISP-BR respectively).
- Fa0/21 and Fa0/22 access ports for VLAN 901 and 902 respectively.
- Fa0/23 trunk port to Acer USB Ethernet adapter carrying VLANs 901 and 902 only. 
- Fa0/21 - 23 all no shut with relevant descriptions. 
- Created a GNS3 Project with Cloud node on the USB adapter linking one subinterface per ISP VLAN (203.0.113.1/30 for HQ circuit and 203.0.113.5/30 for BR circuit). Used c2691 Dynamips with subint Fa0/0.901 and .902 after deciding against the FRR routers for Cisco continuity.
- Gave Fa4 on HQ and branch routers ip addresses (203.0.113.2/30 for HQ-R1 and 203.0.113.6/30 for BR-R1).
- Gave ISP node loopback address 198.51.100.1/32 to simulate the internet.
- Disabled CDP on int Fa4 on both HQ and branch routers and issued passive-interface commands.


## How I verified it:
- Verified on NL-HQ-SW1 the Fa0/21 and Fa0/22 with proper description and VLANs with show int status.
```
Fa0/21    ISP-HQ             connected    901        a-full  a-100 10/100BaseTX
Fa0/22    ISP-BR             connected    902        a-full  a-100 10/100BaseTX
Fa0/23    ISP-TRUNK          connected    trunk      a-full  a-100 10/100BaseTX
Fa0/24    NOC Mgmt           connected    99         a-full  a-100 10/100BaseTX
```
- Verified on NL-HQ-SW1 the proper trunk configuration with Fa0/23 carrying only VLANs 901 and 902 and Fa0/1 now carrying neither with show int trunk.
```
NL-HQ-SW1#show int trunk

Port        Mode         Encapsulation  Status        Native vlan
Fa0/1       on           802.1q         trunking      999
Fa0/23      on           802.1q         trunking      999

Port      Vlans allowed on trunk
Fa0/1       10,20,99
Fa0/23      901-902

Port        Vlans allowed and active in management domain
Fa0/1       10,20,99
Fa0/23      901-902

Port        Vlans in spanning tree forwarding state and not pruned
Fa0/1       10,20,99
Fa0/23      901-902
NL-HQ-SW1#

```
- Before opening GNS3 project, confirmed the Acer was receiving tagged frames from VLANs 901 and 902 sent out by Fa0/23 on HQ-SW1 with tcpdump on Acer. Received PVST+ BPDUs, one per VLAN every 2 seconds. 
```
22:13:40.988333 00:13:80:49:78:97 > 01:00:0c:cc:cc:cd, ethertype 802.1Q (0x8100), length 68: vlan 901, p 7, 802.3LLC, dsap SNAP (0xaa) Individual, ssap SNAP (0xaa) Command, ctrl 0x03: oui Cisco (0x00000c), pid PVST (0x010b), length 42: STP 802.1d, Config, Flags [none], bridge-id 8385.00:13:80:49:78:80.8017, length 42
22:13:41.206829 00:13:80:49:78:97 > 01:00:0c:cc:cc:cd, ethertype 802.1Q (0x8100), length 68: vlan 902, p 7, 802.3LLC, dsap SNAP (0xaa) Individual, ssap SNAP (0xaa) Command, ctrl 0x03: oui Cisco (0x00000c), pid PVST (0x010b), length 42: STP 802.1d, Config, Flags [none], bridge-id 8386.00:13:80:49:78:80.8017, length 42
```

- Confirmed Fa4 on both HQ-R1 and BR-R1 had their assigned ip addresses with show ip int brief.
```
FastEthernet4              203.0.113.2     YES manual up                    up      
```
```
FastEthernet4              203.0.113.6     YES manual up                    up      
```

- Confirmed Fa4 on both HQ-R1 and BR-R1 were not sending CDP packets or OSPF hellos with show cdp interfaces and show ip protocols.
```
  Passive Interface(s):
    FastEthernet4
    Loopback0
    Vlan10
    Vlan20
    Vlan99
```
```
  Passive Interface(s):
    FastEthernet4
    Loopback0
    Vlan30
    Vlan99

```
```
NL-HQ-R1#show cdp interface 
FastEthernet0 is up, line protocol is up
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet1 is down, line protocol is down
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet2 is down, line protocol is down
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet3 is up, line protocol is up
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds

 cdp enabled interfaces : 4
 interfaces up          : 2
 interfaces down        : 2
```
```
NL-BR-R1#sh cdp interface 
FastEthernet0 is up, line protocol is up
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet1 is down, line protocol is down
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet2 is down, line protocol is down
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds
FastEthernet3 is up, line protocol is up
  Encapsulation ARPA
  Sending CDP packets every 60 seconds
  Holdtime is 180 seconds

 cdp enabled interfaces : 4
 interfaces up          : 2
 interfaces down        : 2
```

- Verified the pings from HQ-R1 and BR-R1 to their respective ISP subinterface were going through as intended by observing the tcpdump on the USB Ethernet.
```
23:19:17.267363 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.2 > 203.0.113.1: ICMP echo request, id 7, seq 0, length 80
23:19:17.292492 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:e8, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.1 > 203.0.113.2: ICMP echo reply, id 7, seq 0, length 80
23:19:17.293894 7c:0e:ce:d0:c3:e8 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.2 > 203.0.113.1: ICMP echo request, id 7, seq 1, length 80
23:19:17.323071 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:e8, ethertype 802.1Q (0x8100), length 118: vlan 901, p 0, ethertype IPv4 (0x0800), 203.0.113.1 > 203.0.113.2: ICMP echo reply, id 7, seq 1, length 80
```
```
22:39:38.891177 7c:0e:ce:d0:c3:f6 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 902, p 0, ethertype IPv4 (0x0800), 203.0.113.6 > 203.0.113.5: ICMP echo request, id 4, seq 0, length 80
22:39:38.900203 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:f6, ethertype 802.1Q (0x8100), length 118: vlan 902, p 0, ethertype IPv4 (0x0800), 203.0.113.5 > 203.0.113.6: ICMP echo reply, id 4, seq 0, length 80
22:39:38.901268 7c:0e:ce:d0:c3:f6 > c0:01:71:14:00:00, ethertype 802.1Q (0x8100), length 118: vlan 902, p 0, ethertype IPv4 (0x0800), 203.0.113.6 > 203.0.113.5: ICMP echo request, id 4, seq 1, length 80
22:39:38.931001 c0:01:71:14:00:00 > 7c:0e:ce:d0:c3:f6, ethertype 802.1Q (0x8100), length 118: vlan 902, p 0, ethertype IPv4 (0x0800), 203.0.113.5 > 203.0.113.6: ICMP echo reply, id 4, seq 1, length 80
```

- Verified ISP MAC on ARP table on both HQ and branch routers with show arp. 
```
Internet  203.0.113.1            45   c001.7114.0000  ARPA   FastEthernet4
```
```
Internet  203.0.113.5            43   c001.7114.0000  ARPA   FastEthernet4
```

- Confirmed OSPF neighbor list is unchanged on both HQ and Branch routers with show ip ospf neighbor.
```
NL-HQ-R1#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.2        0   FULL/  -        00:00:31    172.16.0.2      Vlan100
```
```
NL-BR-R1#show ip ospf neighbor 

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.255.0.1        0   FULL/  -        00:00:31    172.16.0.1      Vlan100
NL-BR-R1#
```

- Pings to internet ip 198.51.100.1 fail as intended because there are no routes to that network in either routing tables. 
```
NL-HQ-R1#ping 198.51.100.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 198.51.100.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```
```
NL-BR-R1#ping 198.51.100.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 198.51.100.1, timeout is 2 seconds:
.....
Success rate is 0 percent (0/5)
```

## Stretch: 
- Stopped and started the c2691 ISP node on GNS3 and confirmed pings still worked to HQ-R1 and BR-R1. 
```
ISP#ping 203.0.113.2

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 203.0.113.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 32/40/56 ms
ISP#ping 203.0.113.6

Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 203.0.113.6, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 32/38/44 ms
```

## What went wrong:
- Switching from FRR routers to c2691 was a design choice and not a mistake. 

## Why it works:
- 802.1Q tags carry each circuit through one physical link. 
- The subinterface encapsulation matches each tag to a /30. 

## Still to test:
- NL-BR-SW1 mgmt at 10.20.99.2 not answering pings. 

## Design notes:
- Fate-sharing: The branch's ISP circuit physically runs through NL-HQ-SW1 at HQ, so if that switch fails, both sites lose internet at the same time. A real branch would have its own local circuit. I accepted this as a lab compromise to save cables and Acer NICs.

- Separate Acer links: Management (VLAN 99 on Fa0/24 to enp1s0) and the ISP trunk (901 and 902 on Fa0/23 to the USB adapter) use separate links. That way the simulated ISP is never one subinterface away from the management network, and changes to the ISP trunk can't cut off my management session.
