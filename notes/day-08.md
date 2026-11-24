# Day 8: VLANs 30 and 99 on NL-BR-SW1 through the web UI. Tagged uplink to router.
Ticket: NL-108

## What I built
On NL-BR-SW1:
- Made port 2 and 3 untagged members of VLAN 30 (USERS) with PVID 30.
- Created VLAN 99 (MGMT) and made port 4 an untagged member with PVID 99.
- Created VLAN 999 (PARKING) and made port 1 an untagged member of VLAN 999 with PVID 999 and a tagged member of VLANs 30 and 99.
- Made port 5 an untagged member of VLAN 999 with PVID 999 and disabled it.
- Changed the admin password and changed description to NL-BR-SW1. 

## How I verified it
- Using the web GUI. Check screenshots in notes (day-08-pvid-table and day-08-vlan-table).
- VLAN table shows VLAN 30, 99 and 999 with their respective assigned ports. 
- PVID table shows each port's PVID as configured. 
- Pings to 192.168.0.1 were successful from Acer on ports 1, 2, 3 and 4, confirming the quirk I highlighted earlier. 
- An end-to-end test whereby VLAN 30 host getting through the trunk will wait for NL-BR-R1. 

## What went wrong
- I tested whether VLAN membership isolates management but found out a limitation of the TL-SG105E.
- Port 2 had PVID 30 and was a member of VLAN 30 only and pings to 192.168.0.1 (VLAN 1) were successful. 
- This will allow users to access the admin login page from any port and presents a potential security risk. 
- Partially mitigated this by hardening the password on the web GUI. 

## Why it works
- PVID decides which VLAN untagged frames coming in the port belong to and tagging lets port 1 carry VLAN 30 and 99 over one cable. 
- Native VLAN 999 allows untagged frames coming into port 1 to default to that VLAN.
- Native VLAN 999 improves security as it has no gateway or hosts, so untagged frames on the trunk cannot reach anything. 
- USERS, MGMT and PARKING have designated ports to ensure frames designated for each VLAN will arrive as intended.

##Predictions
Skipped for today, missing notes. 


## Still to test
- Move mgmt IP to 10.20.99.2 on NL-BR-SW1.
- End-to-end trunk test once NL-BR-R1 is built. 
- NL-BR-R1 to match tagged VLANs 30 and 99 with native VLAN 999. 
