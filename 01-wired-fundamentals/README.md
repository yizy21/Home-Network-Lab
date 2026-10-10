# Lab 1 — Wired Fundamentals

**Status:** 🔲 Not started
**Date completed:** _TBD_

## Objective
Demonstrate core wired networking skills: subnetting by hand, switch MAC table inspection, port mirroring configuration, and packet capture analysis (TCP handshake, DNS, DHCP).

## Hardware / Tools
- TP-Link TL-SG108E (8-port managed switch)
- TP-Link router
- Wireshark
- Kali Linux

## What I did
1. _Calculated the home network's subnet by hand from the DHCP range and mask — show the math._
2. _Viewed the MAC address table on the TL-SG108E — screenshot._
3. _Configured port mirroring (source port → mirror port) — screenshot of config._
4. _Captured traffic with Wireshark and identified:_
   - _A TCP three-way handshake (filter: `tcp.flags.syn==1`)_
   - _A DNS query/response pair (filter: `dns`)_
   - _A DHCP release/renew exchange_

## Findings
_One paragraph: what did the capture actually show? Any plaintext traffic observed? What surprised you?_

## Screenshots / Captures
- `subnet-calc.png`
- `mac-table.png`
- `port-mirror-config.png`
- `wireshark-tcp-handshake.png`
- `wireshark-dns-query.png`
- `wireshark-dhcp-dora.png`

## Skills demonstrated
`subnetting` `switching` `MAC address tables` `port mirroring` `Wireshark` `TCP/IP` `DNS` `DHCP`
