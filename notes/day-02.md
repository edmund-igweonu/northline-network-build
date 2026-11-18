# Day 2: Base config and passwords on all Cisco devices
Ticket: NL-102

## What I built
- Configured hostnames NL-HQ-SW1, NL-HQ-R1, NL-BR-R1, enable secret, console and VTY passwords on all Cisco devices
- MOTD banner, no ip domain-lookup, logging synchronous 

## How I verified it
- Reconnected on console: banner shown and password prompt worked 
- show run | include hostname|secret|banner

## What went wrong
- Set up enable secret but was unable to get into privilege daccess mode, turns out there was a space I included unintentionally which became part of the password
- Fix: entered the password with the trailing space then reset the secret
- Lesson: IOS counts trailing spaces as part of password

## Why it works
- Every device now has a hostname and a basic level of security
- enable secret stores the password as a one-way hash, while password is store in plain text

## Still to test
- Nothing 
