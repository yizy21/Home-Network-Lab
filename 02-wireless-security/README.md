# Lab 2 — Wireless Security

**Status:** 🔲 Not started
**Date completed:** _TBD_

> ⚠️ **All actions in this lab were performed exclusively against my own access point and my own devices.** Deauthentication and handshake-capture techniques are illegal against networks you do not own or have explicit authorization to test. See the repo-level note in the main README.

## Objective
Demonstrate the full wireless security workflow: monitor mode, passive reconnaissance, targeted handshake capture, and an offline dictionary attack against a known password — to understand the mechanics of WPA2 security and why strong passphrases matter.

## Hardware / Tools
- Atheros AR9271 USB wireless adapter (`ath9k_htc` chipset)
- Kali Linux (aircrack-ng suite)
- My own WPA2-PSK access point

## What I did
1. _Enabled monitor mode: `airmon-ng check kill` + `airmon-ng start wlan1`_
2. _Surveyed nearby networks with `airodump-ng` — identified my own AP's BSSID/channel_
3. _Ran a targeted capture locked to my AP's channel_
4. _Triggered a handshake by deauthing my own connected device_
5. _Verified handshake capture_
6. _Cracked the handshake offline against a known password using `aircrack-ng` + rockyou.txt_

## Findings
_One paragraph: how long did the crack take against the known password? What does this demonstrate about passphrase entropy? What would change with WPA3?_

## Screenshots / Captures
- `airodump-survey.png`
- `airodump-targeted-capture.png` (showing "WPA handshake" captured)
- `aireplay-deauth.png`
- `aircrack-result.png`

## Skills demonstrated
`802.11 wireless protocols` `monitor mode` `aircrack-ng suite` `WPA2 4-way handshake` `offline dictionary attacks` `wireless security evaluation`
