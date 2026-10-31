# Lab 3 — WAN / NAT

**Status:** ✅ Complete
**Date completed:** October 31, 2026

## Objective
Understand how NAT translates private LAN addresses to a public-facing address, and verify that understanding by configuring port forwarding and testing it from outside my own network, using only my existing home router.

## Hardware / Tools
- TP-Link router (WAN/NAT configuration, port mapping)
- Python `http.server` as a disposable test service
- Wireshark (packet capture)
- `canyouseeme.org` (external port reachability check)

## What I did

### 1. Baseline addressing
Checked my PC's private IP (`ipconfig`), my router's reported WAN IP (router Status page), and the actual public IP as seen from outside (`curl ifconfig.me`).

Private IP: `192.168.100.16`
Router's WAN IP: `10.143.124.117`
Real public IP: `203.0.113.40` *(redacted — actual value withheld for this write-up)*

The router's own "WAN IP" turned out to be a private address (`10.0.0.0/8` range), not a public one — my first sign that something more interesting than a single NAT hop was going on.

### 2. Recognizing CGNAT
The router's WAN IP and the actual public IP the internet sees are two different addresses entirely. This means my ISP is running Carrier-Grade NAT (CGNAT): a second NAT layer between my router and the real internet, common with ISPs that don't have enough public IPv4 addresses to assign one per customer. My router isn't holding a real public IP at all, it's holding another private address on the ISP's own internal network.

### 3. Testing port forwarding against that constraint
Set up a test service and a forwarding rule to see what CGNAT actually does to inbound connections in practice:
- Ran `python -m http.server 8080` on my PC
- Created an IPv4 Port Mapping rule on the router: external port 8080 → `192.168.100.16:8080`, TCP
- Tested reachability from `canyouseeme.org` against my real public IP, port 8080

Result: **connection timed out.** The rule was configured correctly, but nothing from the outside could reach it, confirming that my ISP's CGNAT layer isn't forwarding inbound connections for that port to me at all. This is the practical consequence of CGNAT: port forwarding on your own router is necessary but not sufficient, since you don't control the NAT layer that actually faces the internet.

### 4. Traceroute comparison, and an unplanned double-NAT discovery
Ran `tracert 192.168.100.1` (single hop, confirming it's my direct default gateway) and `tracert 8.8.8.8` to compare.

The first `tracert 8.8.8.8` came back strange: hop 1 was `10.82.158.230`, not my router, with `192.168.100.1` only showing up at hop 2. Since the direct trace had already proven `192.168.100.1` is my actual gateway, it should have been hop 1.

Checked `ipconfig /all` and found the cause: a second active network adapter, a phone connected over USB in internet-sharing/tethering mode, with its own default gateway (`10.82.158.230`). Windows was preferring that adapter's route for general internet traffic over my Ethernet connection, creating an unintended second NAT layer (PC → phone over USB → phone's own Wi-Fi link to my router → router → ISP CGNAT → internet) on top of the one I was actually trying to study.

Disconnected the phone and re-ran the trace. Hop 1 was correctly `192.168.100.1`, followed by `10.143.0.1` (matching the WAN subnet from step 1), then several more private-range hops (`172.20.x.x`) through what's clearly the ISP's own internal backbone, before reaching a public transit hop (GTT) and finally Google. This also reinforced the CGNAT finding from step 2: an ISP running that much of its internal routing on private addressing is a strong sign of large-scale CGNAT deployment, not a one-off for my connection specifically.

### 5. Packet capture, tying it back to NAT
Captured on my Ethernet interface while running `curl ifconfig.me`, filtered in Wireshark with `http.host == "ifconfig.me"`.

The GET request's source IP in the capture was `192.168.100.16`, my private address, confirming NAT translation hadn't happened yet at the point of capture, it happens downstream at the router (and again at the ISP's CGNAT layer). The response packet's body, visible directly in the packet detail pane, contained my actual public IP in plain text, giving a single clean screenshot showing both sides of the translation: private source going out, public identity coming back. *(Public IP redacted in the screenshot before upload.)*

## Findings
The most useful result here wasn't a successful port forward, it was understanding exactly why one failed. A correctly configured forwarding rule on my own router was not enough to make a service reachable from the internet, because my ISP's CGNAT sits between my router and the real public internet and I don't control that layer. This is a real-world constraint, not a misconfiguration, and it's increasingly common with fiber/mobile ISPs facing IPv4 exhaustion. The unplanned double-NAT from my phone's USB tethering was a useful reminder to check `ipconfig /all` for competing routes before trusting a traceroute's hop ordering at face value.

## Screenshots / Captures
- `01-wan-info-private-ip.png`
- `02-port-mapping-config.png`
- `03-canyouseeme-closed.png` *(public IP redacted)*
- `04-tracert-router-baseline.png`
- `05-tracert-google-doublenat.png`
- `06-ipconfig-all-two-adapters.png`
- `07-tracert-google-clean.png`
- `08-wireshark-tcp-conversation.png`
- `09-wireshark-response-body.png` *(public IP redacted)*

## Skills demonstrated
`NAT` `CGNAT` `PAT` `port forwarding` `traceroute` `routing tables` `Wireshark` `packet analysis` `troubleshooting` `attack surface awareness`