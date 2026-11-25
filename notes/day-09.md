# Day 9: SVIs and DHCP on NL-BR-R1 for the branch
Ticket: NL-109

## Predictions
- Untagged frames from the router should land in VLAN 999 where there are no active ports (confirmed). 
- Plugging into the VLAN 99 mgmt will show port 4 sending Discover broadcasts but with no DHCP set up on VLAN 99, it will show the discovery counter going up (not tested).


## What I built
On NL-BR-R1:
- Configured int Fa0 as a trunk allowing only necessary VLANs (Cisco 881 rejects unless default VLANs 1-2 and 1002-1005 are in it).
- Configured SVIs for VLAN 30 and 99, and also made native VLAN 999.

```
NL-BR-R1#sh interfaces trunk 

Port      Mode         Encapsulation  Status        Native vlan
Fa0       on           802.1q         trunking      999

Port      Vlans allowed on trunk
Fa0       1-2,30,99,999,1002-1005

Port      Vlans allowed and active in management domain
Fa0       1,30,99,999

Port      Vlans in spanning tree forwarding state and not pruned
Fa0       1,30,99,999
NL-BR-R1#
```



- Excluded IP addresses in VLAN 30 from .1 to .20 for gateways and static devices.
- Created a pool for USERS with defined default gateway, DNS server, domain name and lease time.
- Configured the clock to current time to prevent dhcp leasing errors. 

## How I verified it
- show ip dhcp pool 
```
NL-BR-R1#show ip dhcp pool

Pool USERS :
 Utilization mark (high/low)    : 100 / 0
 Subnet size (first/next)       : 0 / 0 
 Total addresses                : 254
 Leased addresses               : 2
 Pending event                  : none
 1 subnet is currently in the pool :
 Current index        IP address range                    Leased addresses
 10.20.30.23          10.20.30.1       - 10.20.30.254      2
NL-BR-R1#
```
- show run | section dhcp
```
 NL-BR-R1#show run | section dhcp 
ip dhcp excluded-address 10.20.30.1 10.20.30.20
ip dhcp pool USERS
 network 10.20.30.0 255.255.255.0
 default-router 10.20.30.1 
 dns-server 1.1.1.1 
 domain-name northline.local
NL-BR-R1#
```
- No lease time displayed as that is the default on IOS and sh run does not display defaults.

- show ip int br
```
NL-BR-R1#show ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Cellular0                  unassigned      YES NVRAM  up                    up      
FastEthernet0              unassigned      YES unset  up                    up      
FastEthernet1              unassigned      YES unset  down                  down    
FastEthernet2              unassigned      YES unset  down                  down    
FastEthernet3              unassigned      YES unset  down                  down    
FastEthernet4              unassigned      YES NVRAM  administratively down down    
Vlan1                      unassigned      YES unset  up                    up      
Vlan30                     10.20.30.1      YES manual up                    up      
Vlan99                     10.20.99.1      YES manual up                    up      
NL-BR-R1#
```
- show ip dhcp binding - showed the Acer's MAC with an expiry date of tomorrow
- the Acer got 10.20.30.22 in VLAN 30, with the gateway, DNS and domain from DHCP details.
- Pings to both SVIs were successful as shown below:
```
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ ping 10.20.30.1
PING 10.20.30.1 (10.20.30.1) 56(84) bytes of data.
64 bytes from 10.20.30.1: icmp_seq=1 ttl=255 time=1.25 ms
64 bytes from 10.20.30.1: icmp_seq=2 ttl=255 time=0.769 ms
64 bytes from 10.20.30.1: icmp_seq=3 ttl=255 time=0.981 ms
64 bytes from 10.20.30.1: icmp_seq=4 ttl=255 time=1.03 ms
^C
--- 10.20.30.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3016ms
rtt min/avg/max/mdev = 0.769/1.005/1.245/0.169 ms
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ ping 10.20.99.1
PING 10.20.99.1 (10.20.99.1) 56(84) bytes of data.
64 bytes from 10.20.99.1: icmp_seq=1 ttl=255 time=0.498 ms
64 bytes from 10.20.99.1: icmp_seq=2 ttl=255 time=0.634 ms
64 bytes from 10.20.99.1: icmp_seq=3 ttl=255 time=0.551 ms
64 bytes from 10.20.99.1: icmp_seq=4 ttl=255 time=0.765 ms
^C
--- 10.20.99.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3109ms
rtt min/avg/max/mdev = 0.498/0.612/0.765/0.100 ms
```


## What went wrong
- I configured the USERS pool with a 7 day lease instead of 1 day and corrected it promptly. 
- Ping to 10.20.99.1 from the Acer failed at first until I added a temporary route via 10.20.30.1. nl-dhcp has never-default set and no default route, so traffic to other subnets went out the Wi-Fi. 
- The TP-Link switch took the ip address 10.20.30.21 in VLAN 30, proving the mgmt is not isolated from users on the switch. 
- The switch's mgmt plane sent its DHCP requests in VLAN 30, out port 1 tagged. 
```
eddy-igweonu@eddy-igweonu-Aspire-A515-54:~$ curl -s https://api.macvendors.com/18:69:45

TP-Link Systems Inc.
```
```
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.20.30.1              -   7c0e.ced0.c3f2  ARPA   Vlan30
Internet  10.20.30.21             0   1869.4542.2075  ARPA   Vlan30
```
```
IP address          Client-ID/              Lease expiration        Type
                    Hardware address/
                    User name
10.20.30.21         0118.6945.4220.75       Dec 01 2026 07:01 PM    Automatic
10.20.30.22         01b4.a9fc.7900.d3       Nov 25 2026 07:06 PM    Automatic
```

## Why it works
- Excludes addresses that may be needed for default gateways and static devices.
- The router picks the pool whose network matches the subnet of the SVI the Discover arrived on.
- Trunk from the router carried tagged frames for VLANs 30 and 99, untagged frames go to native VLAN 999 and the router sends each tagged frame to the matching SVI. 
- Both Vlans 30 and 99 are up/up. Vlan 99 is up/up with no hosts as the VLAN exists and the port carrying it (Fa0 trunk) is up and forwarding.


## Still to test
- Static mgmt IP 10.20.99.2 on the switch and whether it really moves mgmt out of VLAN 30.
- Turn off DHCP On the switch and clear stale .21 binding.
- Clock likely resets on reboot, have to config NTP (Day 20).
