# Lab 4 — Structured Troubleshooting

**Status:** ✅ Complete
**Date completed:** November 7, 2026

## Objective
Apply CompTIA's 7-step troubleshooting methodology (identify, theorize, test, plan, implement, verify, document) to deliberately introduced network faults, and capture a baseline of listening services on my workstation — mirroring how structured incident response and basic host awareness work together in practice.

## Hardware / Tools
- Windows PC (static/DHCP IP configuration)
- TP-Link TL-SG108E managed switch (port-based VLAN)
- TP-Link router (DHCP server, DHCP static IP reservations)
- A second Wi-Fi device, for the duplicate-IP scenario
- `ipconfig`, `ping`, `nslookup`, `arp`, `netstat`
- Windows Event Viewer

## What I did

### 1. Incorrect default gateway
Baseline: DHCP-assigned IP, gateway `192.168.100.1`, internet working normally.

Broke it by switching to a static IP and setting the default gateway to `192.168.100.99`, an address that doesn't exist on the LAN. Pinging the router (`192.168.100.1`) still succeeded, since that only requires Layer 2/same-subnet reachability, but pinging `8.8.8.8` failed completely — isolating the problem specifically to routing off the local subnet, not local connectivity.

Fixed by restoring the correct gateway. Verified both the router and `8.8.8.8` were reachable again.

### 2. DNS misconfiguration
Kept IP and gateway correct, changed the preferred DNS server to an unreachable address (`10.255.255.1`). `ping 8.8.8.8` still succeeded (routing was fine), but `ping google.com` failed with "could not find host," and `nslookup google.com` timed out against the broken DNS server, confirming the failure was isolated to name resolution.

Fixed by restoring the correct DNS server (`192.168.100.1`). Verified `ping google.com` and `nslookup` both succeeded, with the resolver returning a real IP.

### 3. Switch port / VLAN misassignment
Removed my PC's switch port (port 6) from VLAN 1 and placed it alone in a new VLAN 2, isolating it at Layer 2 from the router and everything else on the switch.

The first retest after applying the change showed inconsistent results: pinging the router returned partial timeouts followed by a reply, and `8.8.8.8` succeeded fully — suggesting the switch hadn't yet applied the new VLAN filtering to existing, already-learned MAC table entries. A clean retest moments later still showed both the router and internet fully reachable, confirming the isolation genuinely hadn't taken effect yet. Forcing a link-state change (unplugging and replugging the port) flushed the stale table, and the isolation then took full effect: both the router and internet became completely unreachable, and `ipconfig /all` showed a self-assigned APIPA address (`169.254.65.128`) with no gateway and no DHCP server reachable — a classic signature of being cut off below the IP layer, even though at first glance the symptom could be mistaken for something higher up the stack.

Fixed by re-adding port 6 to VLAN 1 and deleting VLAN 2, then flapping the link again to force a clean relearn. Verified a fresh, correct DHCP lease was obtained and both the router and internet were reachable.

### 4. Duplicate static IP
First attempted this via a router-side DHCP reservation, assigning a second device's MAC address the same IP already held by my PC (`192.168.100.16`). The conflict never materialized because the connecting phone presented a randomized MAC address on each connection — a privacy feature in modern mobile OSes — so the reservation, tied to a previously-seen MAC, never matched the device actually on the network, as confirmed by checking the router's client table.

Pivoted to a direct approach: manually set the phone's Wi-Fi IP configuration to static `192.168.100.16`, matching my PC's active address and bypassing DHCP entirely. This produced real, measurable instability — 25% packet loss on `ping 8.8.8.8` — but Windows Event Viewer showed zero occurrences of Event ID 4199 (the explicit duplicate-IP-address log event) during the conflict, meaning the issue had to be diagnosed from observed symptoms rather than a clean system alert.

Fixed by returning the phone to automatic/DHCP addressing. Verified `ping 8.8.8.8` returned to 0% packet loss.

### 5. Listening ports baseline
Ran `netstat -ano | findstr LISTENING` on my workstation to capture a snapshot of all active listening ports and their owning process IDs, as a basic host-awareness check independent of the break/fix scenarios above — the kind of quick sanity check worth doing on any system before assuming a connectivity issue is purely network-side.

## Findings
The most useful lesson across all four fault scenarios wasn't any single fix, it was how often the client-side configuration doesn't tell the whole story. A wrong gateway and wrong DNS both produced clean, predictable symptoms that `ipconfig` alone would reveal to a careful eye. But the VLAN and duplicate-IP scenarios were different: the VLAN break initially looked like it hadn't worked at all, due to switch-side MAC table caching, and once it did work, `ipconfig` showed the self-assigned APIPA signature rather than anything obviously "wrong." The duplicate-IP scenario produced real instability with no corresponding system log entry, and even the setup itself (a DHCP reservation) was quietly defeated by MAC randomization before the actual conflict could be tested. Both are a reminder that structured troubleshooting (testing a theory, re-verifying rather than trusting a first result) matters more than any single diagnostic command, since the same command can tell a different story depending on timing and what layer the problem is actually at.

## Screenshots / Captures
- `01-gateway-break-properties.jpg`
- `02-gateway-break-symptom-pings.png`
- `03-gateway-fix-verify-pings.png`
- `04-gateway-fix-properties.png`
- `05-dns-break-properties.png`
- `06-dns-break-symptom.png`
- `07-dns-fix-verify.png`
- `08-vlan-baseline.png`
- `09-vlan-break-config.png`
- `10-vlan-break-symptom-intermittent.png`
- `11-vlan-retest-router-worked.png`
- `12-vlan-intermittent-ipconfig.png`
- `13-vlan-retest-both-still-working.png`
- `14-vlan-linkflap-isolated-symptom.png`
- `15-vlan-discovery-utility-scanning-apipa.png`
- `16-vlan-apipa-ipconfig.png`
- `17-vlan-fix-verify-ipconfig.png`
- `18-vlan-fix-verify-pings.png`
- `19-duplicate-ip-reservation-config.png`
- `20-duplicate-ip-arp.png`
- `21-duplicate-ip-client-table-macrandomization.png`
- `22-duplicate-ip-adapter-details.png`
- `23-duplicate-ip-eventviewer-no-results.png`
- `24-duplicate-ip-fix-verify.png`
- `25-netstat-listening-ports.png`

## Skills demonstrated
`troubleshooting methodology` `TCP/IP configuration` `DNS resolution` `VLAN configuration` `DHCP` `MAC address tables` `Windows Event Viewer` `netstat` `root cause analysis`