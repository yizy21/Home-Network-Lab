# Lab 2: Wireless Security — WPA2 Handshake Capture & Crack

## Objective
Capture a WPA2 4-way handshake from my own home access point and recover the
Wi-Fi password via a dictionary attack, using a monitor-mode-capable USB
wireless adapter and the Aircrack-ng suite.

**Scope note:** Every technique here (monitor mode, deauthentication,
injection, dictionary cracking) was run only against hardware I own — my own
home AP and my own client devices. Deauth attacks and unauthorized access
attempts against networks you don't own are illegal in most jurisdictions,
Jordan included.

## Setup
- **Adapter:** Qualcomm Atheros AR9271 USB Wi-Fi adapter (`ath9k_htc` driver)
- **Environment:** Kali Linux, live-boot USB via Ventoy (WSL2 can't do
  monitor mode; VM USB passthrough was unreliable for this adapter)
- **Target:** My own home AP — SSID `Umniah Fiber_C6B7`, BSSID
  `80:60:36:49:4E:A0`, channel 1, WPA2-PSK/CCMP

## Steps

### 1. Confirm adapter and baseline interfaces
`lsusb` / `iwconfig` / `ip a` — confirmed the AR9271 adapter present,
`wlan1` in Managed mode, not yet associated.
*(Evidence: 01-adapter-lsusb.png, 02-iwconfig-baseline.png, 03-ip-a-baseline.png)*

### 2. Enable monitor mode
```bash
sudo airmon-ng check kill   # killed conflicting wpa_supplicant process
sudo airmon-ng start wlan1  # created monitor interface wlan1mon
```
*(Evidence: 04-airmon-ng-monitor-mode.png)*

### 3. Survey nearby networks
```bash
sudo airodump-ng wlan1mon
```
Identified three local APs on different channels, including the target
(`Umniah Fiber_C6B7`, channel 1, WPA2 CCMP/PSK).
*(Evidence: 05-survey-scan-three-aps.png)*

### 4. Targeted capture, locked to the AP's channel
```bash
sudo airodump-ng -c 1 --bssid 80:60:36:49:4E:A0 -w handshake wlan1mon
```
Left running continuously in its own terminal for the rest of the lab.

### 5. Deauthenticate a connected client to force a handshake
```bash
sudo aireplay-ng -0 5 -a 80:60:36:49:4E:A0 -c <client_MAC> wlan1mon
```
First attempt used a broadcast deauth — ignored by the client (see
Troubleshooting). Corrected to a targeted deauth against a specific client
MAC, confirmed with real ACKs returned.
*(Evidence: 06-broadcast-deauth-failed.png, 07-targeted-deauth-acks.png)*

### 6. Crack the captured handshake
```bash
aircrack-ng -w <wordlist> handshake-0X.cap
```
Result: `KEY FOUND!` — password recovered via dictionary attack using a
custom wordlist containing the known password.
*(Evidence: 08-aircrack-key-found.png — key value cropped out)*

## Troubleshooting (the actual process, not the clean version)
- **Broadcast deauth failed.** Untargeted broadcast deauths are widely
  ignored by modern PMF-aware clients — had to target a specific client
  with `-c <client_MAC>`.
- **AP vs. station MAC confusion.** Passed a client's MAC as the `-a` (AP)
  argument once, correctly rejected with "No such BSSID available."
- **Channel mismatches, repeatedly.** `aireplay-ng` refused to send when
  `wlan1mon` wasn't on the AP's actual channel, and the interface drifted
  between invocations when nothing held it locked. Fixed by keeping a
  channel-locked `airodump-ng` capture running continuously in a separate
  terminal.
- **Deauth "succeeded" (real ACKs) but 0 handshake.** Was checking a stale
  capture file — `airodump-ng` auto-increments its output filename on every
  new invocation (`handshake-01.cap`, `-02.cap`, ...).
- **Still 0 handshake on the right file, despite EAPOL frames existing.**
  Verified directly with `tshark -r handshake-0X.cap -Y eapol`, which showed
  only a partial handshake (Message 1 only, AP → client). The client's
  reply (Message 2) transmits at much lower power than the AP's — my
  adapter caught the AP's side but missed the client's weaker response.
- **Fix: physical proximity to the client, not the AP.** Moved next to the
  phone, forced it to forget and manually rejoin the network (bypassing
  PMKSA/session-cache shortcuts that skip a full handshake), and captured a
  complete 4-way handshake on the next attempt.

## Key takeaway
A successful deauth (ACKs received) doesn't mean a handshake was captured —
confirming an actual handshake requires checking the capture file itself
(`aircrack-ng` or `tshark -Y eapol`), not just trusting that the deauth
"worked."

## Evidence
- [x] Adapter detection + monitor-mode setup
- [x] Unfiltered survey scan (three APs, channels visible)
- [x] Broadcast-deauth failure, before the fix
- [x] Targeted deauth with real ACKs against correct AP/client
- [x] `aircrack-ng` success output (password redacted/cropped)