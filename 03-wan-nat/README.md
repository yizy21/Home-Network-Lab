# Lab 3 — WAN / NAT / Port Forwarding

**Status:** 🔲 Not started
**Date completed:** _TBD_

## Objective
Demonstrate understanding of NAT and port forwarding by intentionally exposing a test service to the internet, confirming reachability, then closing the exposure — showing the full lifecycle of "opening a port."

## Hardware / Tools
- TP-Link router (NAT / port forwarding config)
- Python `http.server` as a disposable test service

## What I did
1. _Stood up a test web server on port 8000 (`python3 -m http.server 8000`)_
2. _Found the device's local IP and the router's public IP_
3. _Configured a port forwarding rule on the TP-Link router_
4. _Confirmed reachability from outside the home network (cellular data)_
5. _Removed the rule_

## Findings
_One paragraph: what did this demonstrate about NAT's default "deny all inbound" behavior? Why does every forwarded port matter from a security standpoint?_

## Screenshots / Captures
- `port-forward-config.png`
- `external-access-confirmed.png`

## Skills demonstrated
`NAT` `PAT` `port forwarding` `attack surface awareness`
