# Linux Networking

Comprehensive guide to networking, connectivity, DNS, ports, services, routing, and firewalls in Linux.

## Table of Contents
- [Network Interfaces](#network-interfaces)
- [Network Connectivity](#network-connectivity)
- [DNS Configuration](#dns-configuration)
- [Ports and Services](#ports-and-services)
- [Service Management](#service-management)
- [Routing](#routing)
- [Firewalls](#firewalls)
- [Best Practices](#best-practices)

---

## Network Interfaces

### Viewing Network Interfaces

```bash
# Legacy command (still works)
ifconfig                               # Shows interfaces, IPs, MAC, status

# Modern replacement (recommended)
ip a                                   # Show all interfaces
ip addr                                # Same as ip a
ip link                                # Show interface status
```

**Example output:**
```
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
    inet 10.10.10.10/24 brd 10.10.10.255 scope global eth0
    inet6 fe80::a00:27ff:fe4e:66a1/64 scope link
```

### Configuring IP Addresses

⚠️ **Note:** These changes are temporary (lost after reboot)

#### Using ifconfig (legacy)

```bash
# Assign IP address
sudo ifconfig eth0 10.10.10.10 netmask 255.255.255.0

# Bring interface up/down
sudo ifconfig eth0 up
sudo ifconfig eth0 down
```

#### Using ip (modern)

```bash
# Assign IP address
sudo ip a add 10.10.10.10/24 dev eth0
# 10.10.10.10 → IP address
# /24 → subnet mask (255.255.255.0)
# eth0 → interface name

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down

# Remove IP address
sudo ip a del 10.10.10.10/24 dev eth0
```

### CIDR Notation

| CIDR | Subnet Mask | Available IPs |
|------|-------------|---------------|
| /24 | 255.255.255.0 | 256 (254 usable) |
| /16 | 255.255.0.0 | 65,536 |
| /8 | 255.0.0.0 | 16,777,216 |

**Example:**
```bash
# /24 network
10.10.10.0/24
# Network: 10.10.10.0
# Usable: 10.10.10.1 - 10.10.10.254
# Broadcast: 10.10.10.255
```

### Making IP Configuration Permanent

**Debian/Ubuntu:** Edit `/etc/netplan/*.yaml`
```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 10.10.10.10/24
      gateway4: 10.10.10.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

Apply changes:
```bash
sudo netplan apply
```

**RHEL/CentOS:** Edit `/etc/sysconfig/network-scripts/ifcfg-eth0`
```
DEVICE=eth0
BOOTPROTO=static
IPADDR=10.10.10.10
NETMASK=255.255.255.0
GATEWAY=10.10.10.1
DNS1=8.8.8.8
ONBOOT=yes
```

Restart networking:
```bash
sudo systemctl restart network
```

---

## Network Connectivity

### ping - Test Connectivity

Test network connectivity and DNS resolution.

```bash
# Test internet connectivity
ping google.com                        # DNS resolution + reachability

# Test specific IP
ping 8.8.8.8                          # Network only (no DNS)

# Limited packets
ping -c 5 google.com                  # Send 5 packets and stop

# Interval between packets
ping -c 10 -i 2 google.com            # 10 packets, 2 second interval

# Ping flood (DoS testing - needs root)
sudo ping -f google.com               # Send packets as fast as possible
```

**Interpreting results:**
```
64 bytes from 142.250.185.46: icmp_seq=1 ttl=117 time=15.2 ms
                                       ↑        ↑         ↑
                                    sequence  hops   latency
```

**Troubleshooting:**
```bash
# Domain ping works but IP doesn't → Network issue
ping google.com                        # ✅ Works
ping 8.8.8.8                          # ❌ Fails → Network problem

# IP ping works but domain doesn't → DNS issue
ping 8.8.8.8                          # ✅ Works
ping google.com                        # ❌ Fails → DNS problem
```

### traceroute - Trace Network Path

Show hops (routers) between source and destination.

```bash
traceroute google.com                  # Trace route to Google
traceroute 8.8.8.8                    # Trace to specific IP
traceroute -n google.com              # Don't resolve hostnames (faster)
```

**Use cases:**
- Detect network latency
- Identify routing issues
- Analyze packet path

**Example output:**
```
 1  192.168.1.1 (192.168.1.1)  1.234 ms    # Local router
 2  10.0.0.1 (10.0.0.1)  5.678 ms          # ISP gateway
 3  * * *                                   # Timeout (firewall?)
 4  142.250.185.46  15.234 ms              # Google server
```

### netstat / ss - Network Connections

View active network connections, listening ports, and services.

```bash
# Modern replacement for netstat
ss -tulnp                              # Recommended

# Legacy netstat (same flags)
netstat -tulnp
```

**Flag meanings:**
- `t` → TCP connections
- `u` → UDP connections
- `l` → Listening ports
- `n` → Numeric (don't resolve names)
- `p` → Process using port

**Example output:**
```
Proto  Local Address    Foreign Address  State       PID/Program
tcp    0.0.0.0:22       0.0.0.0:*        LISTEN      1234/sshd
tcp    0.0.0.0:80       0.0.0.0:*        LISTEN      5678/nginx
tcp    10.0.0.5:54321   142.250.185.46:443  ESTABLISHED  9012/firefox
```

**Common uses:**
```bash
# All listening ports
ss -tuln

# All established connections
ss -tupn

# Specific port
ss -tuln | grep :80

# Count connections per state
ss -ant | awk '{print $1}' | sort | uniq -c
```

---

## DNS Configuration

### DNS Basics

**DNS (Domain Name System):** Translates domain names to IP addresses

**Common DNS servers:**
- Google: `8.8.8.8`, `8.8.4.4`
- Cloudflare: `1.1.1.1`, `1.0.0.1`
- Quad9: `9.9.9.9`

### DNS Lookup Tools

```bash
# Basic DNS lookup
nslookup google.com                    # Query DNS server

# Detailed DNS lookup
dig google.com                         # More detailed output
dig google.com +short                  # Just the IP
dig @8.8.8.8 google.com               # Query specific DNS server

# Reverse DNS lookup
dig -x 142.250.185.46                 # IP to domain
```

**Example dig output:**
```
;; ANSWER SECTION:
google.com.  300  IN  A  142.250.185.46
             ↑        ↑
          TTL (seconds)  Record type
```

### Configuring DNS

**View DNS configuration:**
```bash
cat /etc/resolv.conf
```

**Example /etc/resolv.conf:**
```
nameserver 8.8.8.8                     # Primary DNS
nameserver 1.1.1.1                     # Secondary DNS
search example.com                     # Domain search suffix
```

**Add DNS server:**
```bash
sudo nano /etc/resolv.conf
# Add:
nameserver 8.8.8.8
nameserver 1.1.1.1
```

⚠️ **Note:** Changes may be overwritten by network manager. For permanent changes:

**Debian/Ubuntu (systemd-resolved):**
```bash
sudo nano /etc/systemd/resolved.conf
# Set:
DNS=8.8.8.8 1.1.1.1

sudo systemctl restart systemd-resolved
```

**RHEL/CentOS:**
```bash
# Edit interface config
sudo nano /etc/sysconfig/network-scripts/ifcfg-eth0
# Add:
DNS1=8.8.8.8
DNS2=1.1.1.1
```

### DNS Troubleshooting

```bash
# Test DNS resolution
ping google.com                        # ✅ DNS + network
ping 8.8.8.8                          # ✅ Network only

# If IP ping works but domain ping fails → DNS issue

# Check DNS servers
cat /etc/resolv.conf

# Test specific DNS server
dig @8.8.8.8 google.com

# Clear DNS cache
sudo systemd-resolve --flush-caches    # systemd-resolved
sudo systemctl restart nscd            # nscd
```

---

## Ports and Services

### Understanding Ports

**Port:** Logical endpoint for services on an IP address
- **Range:** 0–65535
- **Well-known ports:** 0–1023 (require root)
- **Registered ports:** 1024–49151
- **Dynamic/Private:** 49152–65535

### Common Ports

| Service | Port | Protocol |
|---------|------|----------|
| SSH | 22 | TCP |
| FTP | 21 | TCP |
| HTTP | 80 | TCP |
| HTTPS | 443 | TCP |
| DNS | 53 | UDP/TCP |
| SMTP | 25 | TCP |
| MySQL | 3306 | TCP |
| PostgreSQL | 5432 | TCP |
| Redis | 6379 | TCP |
| MongoDB | 27017 | TCP |
| Docker | 2375/2376 | TCP |

### Checking Open Ports

```bash
# List all open/listening ports
ss -tulnp                              # Modern way
netstat -tulnp                         # Legacy way

# Check specific port
ss -tulnp | grep :80                   # Port 80
lsof -i :8080                          # Processes on port 8080

# All TCP ports
ss -tln

# All UDP ports
ss -uln
```

### Finding Process Using Port

```bash
# Find process on specific port
lsof -i :8080                          # Detailed info
sudo lsof -t -i :8080                  # Just PID

# Example output:
COMMAND  PID  USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nginx    1234 root   6u   IPv4  12345      0t0  TCP *:8080 (LISTEN)
```

### Killing Process on Port

```bash
# Find and kill process on port 8080
sudo lsof -t -i :8080 | xargs kill -9

# Safer kill (SIGTERM first)
sudo lsof -t -i :8080 | xargs kill

# Manual way
sudo lsof -i :8080                     # Get PID
sudo kill -9 <PID>                     # Force kill
```

**Explanation:**
```bash
lsof -i :8080      # Find process on port
-t                 # Print only PID
| xargs            # Pass PID to next command
kill -9            # Force kill (SIGKILL)
```

---

## Service Management

### systemd Services

**Service:** Background process managed by systemd

```bash
# Check service status
systemctl status nginx                 # Detailed status
systemctl is-active nginx              # Just active/inactive

# Control services
systemctl start nginx                  # Start service
systemctl stop nginx                   # Stop service
systemctl restart nginx                # Restart service
systemctl reload nginx                 # Reload config without restart

# Boot configuration
systemctl enable nginx                 # Start on boot
systemctl disable nginx                # Don't start on boot
systemctl is-enabled nginx             # Check boot status

# List services
systemctl list-units --type=service    # All services
systemctl list-units --type=service --state=running  # Running only
systemctl list-units --type=service --state=failed   # Failed only
```

### Service Logs

```bash
# View service logs
journalctl -u nginx                    # All logs for nginx
journalctl -u nginx -f                 # Follow logs
journalctl -u nginx --since today      # Today's logs
journalctl -u nginx --since "1 hour ago"  # Last hour
journalctl -u nginx -n 100             # Last 100 lines
```

---

## Routing

### Understanding Routing

**Routing:** Determines where network packets should be sent

**Route table:** Maps destination networks to gateways

### Viewing Routes

```bash
# View routing table
ip route                               # Modern way
route -n                               # Legacy way
netstat -rn                            # Alternative
```

**Example output:**
```
default via 192.168.1.1 dev eth0       # Default gateway (router)
10.0.0.0/24 via 192.168.1.1 dev eth0   # Specific route
192.168.1.0/24 dev eth0 scope link     # Directly connected network
```

**Route components:**
- `default via 192.168.1.1` → Default gateway (router)
- `10.0.0.0/24` → Destination network
- `via 192.168.1.1` → Next hop (gateway)
- `dev eth0` → Interface to use

### Adding Routes

⚠️ **Note:** Temporary (lost after reboot)

```bash
# Add route
sudo ip route add 10.0.0.0/24 via 192.168.1.1

# Explanation:
# - Traffic destined for 10.0.0.0 – 10.0.0.255
# - Should be sent via gateway 192.168.1.1

# Add default gateway
sudo ip route add default via 192.168.1.1

# Before adding, verify gateway is reachable
ping 192.168.1.1
```

### Deleting Routes

```bash
# Delete specific route
sudo ip route del 10.0.0.0/24

# Delete default gateway
sudo ip route del default
```

### Making Routes Permanent

**Debian/Ubuntu (Netplan):**
```yaml
# /etc/netplan/*.yaml
network:
  version: 2
  ethernets:
    eth0:
      routes:
        - to: 10.0.0.0/24
          via: 192.168.1.1
        - to: default
          via: 192.168.1.1
```

**RHEL/CentOS:**
```bash
# /etc/sysconfig/network-scripts/route-eth0
10.0.0.0/24 via 192.168.1.1
```

---

## Firewalls

### Firewall Concepts

**Firewall:** Security layer that filters network traffic based on rules

**Filter criteria:**
- IP addresses
- Ports
- Protocols
- Direction (inbound/outbound)

### Types of Firewalls

#### Stateful Firewall (Recommended ✅)

**Tracks connection states:**
- NEW - New connection request
- ESTABLISHED - Existing connection
- RELATED - Related connection (FTP data, ICMP)
- INVALID - Malformed packets

**Advantages:**
- Remembers active connections
- Automatically allows response traffic
- More secure
- Easier to configure

**Example:**
```bash
# Only need to allow incoming
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# Responses are automatically allowed
```

#### Stateless Firewall

**Examines each packet independently**
- No connection tracking
- Must explicitly allow both directions
- Less secure
- Higher overhead

**Example:**
```bash
# Must allow both directions
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A OUTPUT -p tcp --sport 80 -j ACCEPT
```

### Linux Firewall Stack

```
Application
   ↓
iptables / nftables
   ↓
firewalld / ufw (frontends)
   ↓
Kernel (netfilter)
```

---

## UFW - Uncomplicated Firewall

Simple firewall for Ubuntu/Debian.

### Basic UFW Commands

```bash
# Enable/disable firewall
sudo ufw enable
sudo ufw disable

# Check status
sudo ufw status
sudo ufw status numbered               # Show rule numbers
sudo ufw status verbose                # Detailed status
```

### Allowing and Denying

```bash
# Allow services by name
sudo ufw allow ssh                     # Port 22
sudo ufw allow http                    # Port 80
sudo ufw allow https                   # Port 443
sudo ufw allow OpenSSH                 # SSH safely

# Allow by port
sudo ufw allow 8080                    # All traffic on 8080
sudo ufw allow 8080/tcp                # TCP only on 8080
sudo ufw allow 8080/udp                # UDP only on 8080

# Deny traffic
sudo ufw deny 3306                     # Block MySQL
sudo ufw deny ftp                      # Block FTP
```

### Advanced UFW Rules

```bash
# Allow from specific IP
sudo ufw allow from 192.168.1.100

# Allow from IP to specific port
sudo ufw allow from 192.168.1.100 to any port 22

# Allow from subnet
sudo ufw allow from 10.0.0.0/24

# Deny from network
sudo ufw deny from 203.0.113.0/24

# Outbound traffic
sudo ufw allow out 443                 # Allow outbound HTTPS
sudo ufw deny out 25                   # Block outbound SMTP
```

### Managing Rules

```bash
# List rules with numbers
sudo ufw status numbered

# Delete rule by number
sudo ufw delete 3

# Delete rule by specification
sudo ufw delete allow 8080

# Insert rule at position
sudo ufw insert 1 allow from 10.0.0.0/24
```

### Default Policies

```bash
# Set defaults
sudo ufw default deny incoming         # Block all incoming
sudo ufw default allow outgoing        # Allow all outgoing
sudo ufw default deny forward          # Block routing

# Verify
sudo ufw status verbose
```

### UFW Best Practices

```bash
# 1. Allow SSH before enabling firewall
sudo ufw allow OpenSSH
sudo ufw enable

# 2. Allow necessary services
sudo ufw allow http
sudo ufw allow https

# 3. Set sensible defaults
sudo ufw default deny incoming
sudo ufw default allow outgoing

# 4. Reload after changes
sudo ufw reload
```

---

## iptables - Advanced Firewall

Lower-level firewall with more control.

### iptables Concepts

**Tables:**
- `filter` - Default, allow/block traffic
- `nat` - Port forwarding, NAT
- `mangle` - Modify packets
- `raw` - Disable connection tracking

**Chains:**
- `INPUT` - Incoming traffic
- `OUTPUT` - Outgoing traffic
- `FORWARD` - Routed traffic

**Rules evaluated TOP → BOTTOM; first match wins**

### Viewing iptables Rules

```bash
# List all rules
sudo iptables -L -n -v                 # Numeric, verbose

# List specific table
sudo iptables -t nat -L                # NAT table
sudo iptables -t mangle -L             # Mangle table

# List with line numbers
sudo iptables -L --line-numbers
```

### Default Policies

```bash
# Set default policies
sudo iptables -P INPUT DROP            # Drop all incoming
sudo iptables -P FORWARD DROP          # Block routing
sudo iptables -P OUTPUT ACCEPT         # Allow outgoing
```

### Basic iptables Rules

```bash
# Allow SSH (port 22) - DO THIS FIRST!
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP and HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Allow from specific IP
sudo iptables -A INPUT -s 192.168.1.100 -j ACCEPT

# Block specific IP
sudo iptables -A INPUT -s 203.0.113.5 -j DROP

# Block specific port
sudo iptables -A INPUT -p tcp --dport 3306 -j DROP
```

### Stateful Firewall (Recommended)

```bash
# Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# This enables:
# - Web browsing responses
# - SSH session continuation
# - Package download responses
```

### Essential Rules

```bash
# 1. Allow loopback (localhost communication)
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A OUTPUT -o lo -j ACCEPT

# 2. Allow established connections
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

# 3. Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# 4. Allow ICMP (ping)
sudo iptables -A INPUT -p icmp -j ACCEPT

# 5. Drop everything else (if default policy is DROP)
# (Automatically handled by default policy)
```

### Advanced iptables

```bash
# SSH only from specific IP
sudo iptables -A INPUT -p tcp --dport 22 -s 203.0.113.5 -j ACCEPT

# Rate limit SSH (prevent brute force)
sudo iptables -A INPUT -p tcp --dport 22 -m limit --limit 5/min -j ACCEPT

# Log dropped packets
sudo iptables -A INPUT -j LOG --log-prefix "IPTABLES-DROP: "
sudo iptables -A INPUT -j DROP

# Limit logging (prevent disk fill)
sudo iptables -A INPUT -m limit --limit 5/min -j LOG --log-prefix "IPTABLES-DROP: "
sudo iptables -A INPUT -j DROP
```

### Managing iptables Rules

```bash
# Delete rule by number
sudo iptables -L --line-numbers
sudo iptables -D INPUT 3               # Delete rule 3

# Flush all rules (CAREFUL!)
sudo iptables -F                       # Clear filter table
sudo iptables -X                       # Delete custom chains
sudo iptables -t nat -F                # Clear NAT table

# Insert rule at position
sudo iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT
```

### Saving iptables Rules

**Temporary rules** (lost after reboot) need to be saved.

```bash
# Save rules (Debian/Ubuntu)
sudo iptables-save > /etc/iptables/rules.v4
sudo ip6tables-save > /etc/iptables/rules.v6

# Restore rules
sudo iptables-restore < /etc/iptables/rules.v4
```

### Making iptables Persistent

#### Method 1: iptables-persistent

```bash
# Install package
sudo apt install iptables-persistent

# Save current rules
sudo netfilter-persistent save

# Rules saved to:
# /etc/iptables/rules.v4
# /etc/iptables/rules.v6
```

#### Method 2: rc.local (Legacy)

```bash
# Make executable
sudo chmod +x /etc/rc.local

# Edit file
sudo nano /etc/rc.local

# Add:
#!/bin/bash
/sbin/iptables-restore < /etc/iptables/rules.v4
/sbin/ip6tables-restore < /etc/iptables/rules.v6
exit 0
```

### iptables Actions

| Action | Meaning |
|--------|---------|
| `ACCEPT` | Allow packet |
| `DROP` | Silently discard |
| `REJECT` | Discard + send response |
| `LOG` | Log packet |

**DROP vs REJECT:**
- `DROP` - Silent (use for internet-facing)
- `REJECT` - Sends response (use for internal networks)

---

## Best Practices

### Network Configuration

1. **Use modern tools**
   - Prefer `ip` over `ifconfig`
   - Use `ss` instead of `netstat`

2. **Document changes**
   - Keep records of IP assignments
   - Document routing rules
   - Note firewall changes

3. **Test before production**
   ```bash
   # Test connectivity
   ping gateway
   ping 8.8.8.8
   ping google.com
   ```

### Firewall Security

1. **Default deny policy**
   ```bash
   sudo ufw default deny incoming
   sudo iptables -P INPUT DROP
   ```

2. **Always allow SSH first**
   ```bash
   sudo ufw allow OpenSSH
   sudo ufw enable
   # Prevents lockout
   ```

3. **Use stateful firewall**
   ```bash
   sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
   ```

4. **Limit logging**
   ```bash
   # Prevent disk fill
   iptables -A INPUT -m limit --limit 5/min -j LOG
   ```

5. **Regular audits**
   ```bash
   sudo ufw status numbered
   sudo iptables -L -n -v
   ```

### Troubleshooting Workflow

1. **Check connectivity**
   ```bash
   ping 8.8.8.8                        # Network layer
   ping google.com                     # DNS + network
   ```

2. **Check DNS**
   ```bash
   cat /etc/resolv.conf
   dig google.com
   ```

3. **Check routes**
   ```bash
   ip route
   traceroute google.com
   ```

4. **Check firewall**
   ```bash
   sudo ufw status
   sudo iptables -L -n -v
   ```

5. **Check services**
   ```bash
   systemctl status service-name
   ss -tulnp
   ```

---

## Quick Reference

### Network Commands

```bash
ip a                                   # Show interfaces
ip route                               # Show routes
ping google.com                        # Test connectivity
dig google.com                         # DNS lookup
ss -tulnp                              # Open ports
lsof -i :8080                          # Process on port
systemctl status nginx                 # Service status
```

### UFW Commands

```bash
sudo ufw enable                        # Enable firewall
sudo ufw allow 80                      # Allow port
sudo ufw deny 3306                     # Block port
sudo ufw status numbered               # List rules
sudo ufw delete 3                      # Remove rule
```

### iptables Commands

```bash
sudo iptables -L -n -v                 # List rules
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT  # Allow HTTP
sudo iptables -D INPUT 3               # Delete rule 3
sudo netfilter-persistent save         # Save rules
```

---

This comprehensive guide covers essential Linux networking concepts for system administration and DevOps operations.