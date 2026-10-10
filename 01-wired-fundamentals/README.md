# Lab 1 — Wired Fundamentals

**Status:** ✅ Complete
**Date completed:** October 10, 2026

## Objective
Demonstrate core wired networking skills: subnetting by hand, switch configuration inspection, cable diagnostics, and packet capture analysis (DNS resolution, TCP three-way handshake, DHCP DORA process).

## Hardware / Tools
- TP-Link TL-SG108E (8-port managed switch, firmware 1.0.0 Build 20211209, hardware v6.0)
- Main router (DHCP server / default gateway)
- Wireshark (captured directly on the Windows workstation's Ethernet NIC)

## What I did

### 1. Subnet calculation (by hand)
Pulled the DHCP configuration from the main router:
- Subnet Mask: `255.255.255.0`
- DHCP Pool: `192.168.100.2` – `192.168.100.254`

Worked out, without a calculator:
- Subnet mask in binary: `11111111.11111111.11111111.00000000` → **24 bits** → CIDR `/24`
- Network ID: `192.168.100.0`
- Broadcast address: `192.168.100.255`
- Full usable host range: `192.168.100.1` – `192.168.100.254` (254 addresses — formula: 2⁸ − 2)
- Confirmed the DHCP pool (`.2`–`.254`) is a subset of the full usable range, with `.1` reserved outside the pool for the router's own gateway address.

### 2. Switch configuration inspection
Logged into the TL-SG108E's web UI (`192.168.100.15`) and explored:
- **System Info**: confirmed device details, firmware/hardware version, management IP
- **Port Settings**: found ports 1, 6, 7, 8 active (LinkUp) at 1000MF / 1000MF / 100MF / 10MF respectively; ports 2–5 showing LinkDown (no device connected)
- **Port Statistics**: TX/RX packet counts per port, consistent with the active/inactive ports above
- **Cable Test**: ran on all 8 ports — ports 2–5 (disconnected) returned "Open" or "Open & Short" faults; ports 1, 6, 7, 8 (connected, active) returned "Normal"

**Note**: this switch model/firmware (TL-SG108E "Easy Smart" line) does **not** expose a MAC address table in the web UI — Monitoring only offers Port Statistics, Port Mirror, Cable Test, and Loop Prevention. This is a genuine hardware/firmware limitation of this entry-level switch tier, not a missed step.

### 3. PC-side network mapping
Ran `ipconfig /all`, `arp -a`, and `route print` on the workstation (plugged into switch port 8) and built a network map:

| Device | IP | MAC | Role |
|---|---|---|---|
| Main router | `192.168.100.1` | `80-60-36-49-4e-98` | Gateway / DHCP server / DNS server |
| TL-SG108E switch | `192.168.100.15` | — | Managed switch (mgmt interface) |
| This PC (wired) | `192.168.100.16` | `10-FF-E0-D8-56-17` | Workstation under test |

*(Other devices present on the LAN were observed in the ARP table but are omitted here — they belong to other household members/devices not relevant to this lab.)*

Confirmed via `route print` that the default route (`0.0.0.0/0`) points to `192.168.100.1` through the local interface, and that the full `192.168.100.0/24` subnet is marked "On-link" — both consistent with the subnet math above.

**Interesting finding**: the ARP table contained an entry for `192.168.100.12` mapped to this PC's own Wi-Fi adapter MAC address (`ac-36-1b-ed-7f-4d`), despite that adapter showing "Media disconnected" in `ipconfig`. This is a **stale ARP cache entry** — a leftover mapping from when the adapter last held an IP, which the OS hadn't flushed yet. It's a small but concrete illustration of why ARP is considered a "trust-based, unauthenticated" protocol: the cache can hold outdated or unverified information, which is exactly the weakness ARP spoofing exploits.

### 4. Packet capture — DNS
Captured on the Ethernet interface while running `nslookup google.com`. Filter: `dns`

- **Query** (txn `0xf9c2`): PC → router asking for the A record of `google.com`
- **Response** (txn `0xf9c2`): router → PC returning `A google.com → 142.250.181.142`
- Also observed a second response (txn `0x9204`) for the **AAAA** (IPv6) record of the same domain (`2a00:1450:4006:820::200e`), confirming the OS performs dual-stack (A + AAAA) resolution in parallel — "happy eyeballs" behavior.

### 5. Packet capture — TCP three-way handshake
Captured while generating HTTP/HTTPS traffic via browser activity. Filter: `tcp.flags.syn==1 && ip.addr==192.168.100.16`, then isolated one clean conversation via Wireshark's "Conversation Filter."

Captured conversation (port 80, HTTP, to `142.250.178.3`):
1. **Packet 998** — `[SYN]` PC → server, port 60994 → 80
2. **Packet 1002** — `[SYN, ACK]` server → PC, port 80 → 60994
3. **Packet 1003** — `[ACK]` PC → server — handshake complete
4. **Packet 1004** — `HTTP GET /wr2/75r4ZyA3vA0.crl` — the request that followed

All three handshake steps are visible, which confirmed the connection setup described in Module 3 notes. Interestingly, the GET request was for a `.crl` (Certificate Revocation List) file — background OS/browser activity checking a TLS certificate's revocation status, not something manually triggered. A useful reminder that a live capture shows *all* traffic, not just what you intentionally generate.

### 6. Packet capture — DHCP DORA process
Captured while running `ipconfig /release` followed by `ipconfig /renew`. Filter: `bootp`

- **Release** (txn `0x8e73a7e`): PC → router, giving back the current lease
- **Discover** (txn `0xfe41d095`): PC → broadcast, "is there a DHCP server?"
- **Offer** (txn `0xfe41d095`): router → broadcast, offering an IP
- **Request** (txn `0xfe41d095`): PC → broadcast, accepting the offer
- **ACK** (txn `0xfe41d095`): router → broadcast, confirming the lease

All four DORA packets share the same transaction ID, confirming they're one coherent exchange.

**Interesting finding**: the capture also contained repeating `DHCP ACK` broadcasts (transaction ID `0x948e3f03`) throughout the entire session, unrelated to the manual release/renew. These were the router periodically re-broadcasting lease ACKs in the background — a reminder that not everything in a packet capture comes from an action you deliberately took.

## Findings

This lab confirmed hands-on understanding of subnetting, switch administration on real hardware, and the ability to read and interpret live packet captures for three core protocols (DNS, TCP, DHCP). Two unplanned but genuinely useful discoveries came out of it: a stale ARP cache entry (illustrating ARP's trust-based weakness) and background DHCP/TLS-revocation traffic unrelated to the deliberate test actions — both reinforced that real network traffic includes much more than what you intentionally generate, which is itself a practical lesson for later log/capture analysis work.

## Screenshots / Captures
- `dhcp-settings.png`
- `switch-system-info.png`
- `port-settings.png`
- `port-statistics.png`
- `cable-test.png`
- `wireshark-dns.png`
- `wireshark-tcp-handshake.png`
- `wireshark-dhcp-dora.webp`

## Skills demonstrated
`subnetting` `CIDR` `switch administration` `cable diagnostics` `ARP` `routing tables` `Wireshark` `packet analysis` `TCP/IP` `DNS` `DHCP`