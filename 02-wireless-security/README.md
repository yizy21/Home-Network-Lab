# Lab 2 — Wireless Security

**Status:** ✅ Complete
**Date completed:** October 15, 2026

## Objective
Capture a WPA2 4-way handshake from my own home access point and recover the Wi-Fi password via a dictionary attack, using a monitor-mode-capable USB wireless adapter and the Aircrack-ng suite.

**Scope note:** every technique here (monitor mode, deauthentication, injection, dictionary cracking) was run only against hardware I own — my own home AP and my own client devices. Deauth attacks and unauthorized access attempts against networks you don't own are illegal in most jurisdictions, Jordan included.

## Hardware / Tools
- Qualcomm Atheros AR9271 USB Wi-Fi adapter (`ath9k_htc` driver)
- Kali Linux, live-boot USB via Ventoy (WSL2 can't do monitor mode; VM USB passthrough was unreliable for this adapter)
- Target: my own home AP — SSID `Umniah Fiber_C6B7`, BSSID `80:60:36:49:4E:A0`, channel 1, WPA2-PSK/CCMP
- Aircrack-ng suite (`airmon-ng`, `airodump-ng`, `aireplay-ng`, `aircrack-ng`), `tshark`

## What I did

### 1. Confirm adapter and baseline interfaces
Ran `lsusb`, `iwconfig`, and `ip a` to confirm the AR9271 adapter was present and `wlan1` was in Managed mode, not yet associated to anything.

### 2. Enable monitor mode
```bash
sudo airmon-ng check kill   # killed conflicting wpa_supplicant process
sudo airmon-ng start wlan1  # created monitor interface wlan1mon
```

### 3. Survey nearby networks
```bash
sudo airodump-ng wlan1mon
```
Identified three local APs on different channels, including the target (`Umniah Fiber_C6B7`, channel 1, WPA2 CCMP/PSK).

### 4. Targeted capture, locked to the AP's channel
```bash
sudo airodump-ng -c 1 --bssid 80:60:36:49:4E:A0 -w handshake wlan1mon
```
Left running continuously in its own terminal for the rest of the lab.

### 5. Deauthenticate a connected client
First attempt sent an untargeted broadcast deauth — ignored by the client. Modern PMF-aware devices largely disregard broadcast deauths, so I corrected to a targeted deauth against a specific client MAC:
```bash
sudo aireplay-ng -0 5 -a 80:60:36:49:4E:A0 -c <client_MAC> wlan1mon
```
This returned real ACKs from the client, confirming the frames were landing correctly.

### 6. Diagnosing a failed handshake capture
Getting ACKs back didn't mean a handshake had actually been captured, and working out why took several rounds:
- **AP vs. station MAC confusion** — I initially passed a client's MAC as the `-a` (AP) argument, correctly rejected with "No such BSSID available."
- **Channel mismatches, repeatedly** — `aireplay-ng` refused to send when `wlan1mon` wasn't on the AP's actual channel, and the interface drifted between invocations whenever nothing was holding it locked. Fixed by keeping a channel-locked `airodump-ng` capture running continuously in a separate terminal rather than relying on `aireplay-ng` alone.
- **Deauth "succeeded" but 0 handshake** — turned out I was checking a stale capture file. `airodump-ng` auto-increments its output filename on every new invocation (`handshake-01.cap`, `-02.cap`, ...), so I was repeatedly re-checking an old file instead of the live one.
- **Still 0 handshake on the correct file, despite EAPOL frames existing** — verified directly with `tshark -r handshake-0X.cap -Y eapol`, which showed only a partial handshake (Message 1 only, AP → client). The client's reply (Message 2) transmits at much lower power than the AP's — my adapter caught the AP's side cleanly but missed the client's weaker response.
- **Fix: physical proximity to the client, not the AP** — moved next to the phone, forced it to forget and manually rejoin the network (to bypass PMKSA/session-cache shortcuts that skip a full handshake), and captured a complete 4-way handshake on the next attempt.

### 7. Crack the captured handshake
```bash
aircrack-ng -w <wordlist> handshake-0X.cap
```
Result: `KEY FOUND!` — password recovered via dictionary attack using a custom wordlist containing the known password.

## Findings
A successful deauth (ACKs received) doesn't mean a handshake was captured — those are two separate things, and confirming an actual handshake requires checking the capture file itself (`aircrack-ng` or `tshark -Y eapol`), not just trusting that the deauth "worked." The most useful unplanned discovery was that client-side reply power is often the real bottleneck in handshake capture, not the deauth itself — something no walkthrough mentions, since it only shows up once you're troubleshooting a capture that looks like it should have worked but hasn't.

## Screenshots / Captures
- `01-adapter-lsusb.png`
- `02-iwconfig-baseline.png`
- `03-ip-a-baseline.png`
- `04-airmon-ng-monitor-mode.png`
- `05-survey-scan-three-aps.png`
- `05b-tplink-isolated-scan.png`
- `06-broadcast-deauth-failed.png`
- `06b-umniah-station-list.png`
- `07-targeted-deauth-acks.png`
- `08-aircrack-key-found.png` *(key value cropped out)*

## Skills demonstrated
`wireless security` `monitor mode` `802.11 frame analysis` `WPA2 four-way handshake` `airmon-ng` `airodump-ng` `aireplay-ng` `aircrack-ng` `tshark` `EAPOL` `dictionary attacks` `troubleshooting`