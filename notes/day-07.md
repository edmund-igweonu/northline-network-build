# Day 7: Break-fix, VLAN 10 DHCP pool exhausted by unauthorized exclusion

Ticket: NL-107

## Summary

An unauthorized DHCP exclusion on NL-HQ-R1 left the USERS pool with no addresses, so VLAN 10 clients couldn't get leases. I removed it and service came back. Outage ran from about 09:24 to 10:07.

## Impact

New clients and renewals on VLAN 10 failed. Existing leases kept working until they expired. VLAN 20 and 99 were fine.

## Detection

User report. Nothing alerted.

## Timeline

- 09:24 Fault planted
- 09:34 Symptom reproduced on Fa0/2 with nl-dhcp
- 10:00 Root cause found
- 10:05 Fix applied
- 10:07 Verified

## Predictions

1. USERS pool has no addresses left. **Correct.**
2. Native VLAN mismatch on the trunk. **Wrong.** VLAN 10 is tagged, and VLAN 20 and 99 crossed the same trunk fine.

My first hunch, that DHCP messages weren't getting across, was also wrong. The Discover reached the router, it just had nothing to offer.

## Investigation

Since VLAN 20 and 99 share the same trunk, router and DHCP service, the fault had to be specific to VLAN 10. I found it in show run on NL-HQ-R1:

```
ip dhcp excluded-address 10.10.10.21 10.10.10.254
```

Next time I'll work the path with operational show commands instead of going straight to the config.

## Root cause

The new exclusion plus my original .1 to .20 exclusion covered every usable address in 10.10.10.0/24. It went in without a change request.

## Resolution

Removed the extra exclusion and kept the original .1 to .20.

```
no ip dhcp excluded-address 10.10.10.21 10.10.10.254
```

## Verification

The Acer on Fa0/2 got a lease again, and `show ip dhcp binding` listed its MAC.

## Prevention

- Change control: no change request, no change.
- Syslog config change messages plus Netmiko backups and git diff (Week 4) would catch the unexpected line.
- LibreNMS pool monitoring over SNMP, once I confirm the 881 exposes it.
