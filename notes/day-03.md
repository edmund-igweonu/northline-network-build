# Day 3: SSH only on all Cisco devices
Ticket: NL-103

## What I built
- SSH v2 on NL-HQ-R1 and NL-BR-R1, local admin user, Telnet disabled on VTY
- 2950 image has no k9, so no SSH there. Local login only for now.

## How I verified it
- show ip ssh: "SSH Enabled - version 2.0"
- show run | section line vty: transport input ssh

## What went wrong
- Key generation failed until I set ip domain-name

## Why it works
The RSA key is built from the hostname and domain name, so both must exist first.

## Still to test
- SSH in from the Acer once management IPs exist (Day 5)
