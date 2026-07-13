### Why IPv6 — The Numbers Problem

IPv4 gives you **32 bits** → 2³² = ~4.3 billion addresses.  
IPv6 gives you **128 bits** → 2¹²⁸ = **340 undecillion** addresses.

That number written out: **340,282,366,920,938,463,463,374,607,431,768,211,456**

Every person on Earth could have **45 quadrillion** addresses. Every device, every container, every IoT sensor — globally unique, no NAT required.

---

## Comparison of IPv4 and IPv6

| Feature | IPv4 | IPv6 |
|---------|------|------|
| Address Length | 32 bits | 128 bits |
| Address Format | Decimal (e.g., 192.168.1.1) | Hexadecimal (e.g., 2001:db8::1) |
| Address Space | ~4.3 billion | ~340 undecillion |
| Configuration | Manual / DHCP | SLAAC / DHCPv6 / Static |
| Security | Optional (IPsec available) | Mandatory in spec (IPsec integrated) |
| NAT | Required (address shortage workaround) | Not needed (every device globally routable) |
| Broadcast | Yes (subnet broadcast) | No (replaced entirely by multicast) |
| Header Size | Variable 20–60 bytes | Fixed 40 bytes (faster router processing) |
| Fragmentation | Routers can fragment packets | End-to-end only (PMTUD required) |
| Header Checksum | Yes (recalculated at every hop) | Removed (L2 and L4 checksums sufficient) |
| ARP | Uses ARP (broadcast-based) | Uses NDP via ICMPv6 (multicast-based) |
| Auto-configuration | APIPA (169.254.x.x, limited) | SLAAC (full addresses, always-on) |
| Loop / Hop Control | TTL (Time To Live, 8-bit) | Hop Limit (8-bit, same function) |
| IP Options | Variable options field in header | Extension headers (chained, flexible) |
| Flow Identification | No native flow label | Flow Label field (20-bit, built-in QoS) |
| Packet Size | Min 576 bytes (MTU) | Min 1280 bytes (MTU) |
| DNS Record Type | A record | AAAA record |
| Loopback Address | 127.0.0.1 | ::1 |
| Private Address Range | 10.x.x.x / 172.16.x.x / 192.168.x.x | fc00::/7 (Unique Local — fd::/8 in practice) |
| Link-Local Address | 169.254.x.x (APIPA only) | fe80::/10 (mandatory on every interface) |

---

### IPv6 Address Structure

An IPv6 address is **128 bits** written as **eight groups of four hexadecimal digits**, separated by colons.

```
Full address:    2001:0db8:85a3:0000:0000:8a2e:0370:7334
                 ─────────────────────────────────────────
                 8 groups × 16 bits = 128 bits total
                 Each group = 4 hex digits = 16 bits

Hex digits:  0-9, a-f (case-insensitive, lowercase preferred by RFC 5952)

Breaking it down by bit position:
  Group 1:  bits 127–112  →  2001
  Group 2:  bits 111–96   →  0db8
  Group 3:  bits 95–80    →  85a3
  Group 4:  bits 79–64    →  0000
  Group 5:  bits 63–48    →  0000
  Group 6:  bits 47–32    →  8a2e
  Group 7:  bits 31–16    →  0370
  Group 8:  bits 15–0     →  7334

Network portion vs Host portion (typically /64 prefix):
  2001:0db8:85a3:0000 : 0000:8a2e:0370:7334
  ──────────────────── ──────────────────────
  Network prefix       Interface ID
  (first 64 bits)      (last 64 bits)
```

---

### IPv6 Compression Rules — RFC 5952

IPv6 addresses can be shortened using two compression rules. Both can apply to the same address.

#### Rule 1 — Remove Leading Zeros Within Each Group

Leading zeros inside any group can be omitted (but at least one digit must remain).

```
RULE 1: Drop leading zeros in each group

  0db8  →  db8    (drop leading 0)
  0000  →  0      (drop three leading zeros)
  0001  →  1      (drop three leading zeros)
  00a3  →  a3     (drop two leading zeros)
  1234  →  1234   (no leading zeros, unchanged)

EXAMPLE:
  Full:       2001:0db8:0000:0000:0001:0000:0000:0001
  Rule 1:     2001:db8:0:0:1:0:0:1
  (Leading zeros in each group removed)
```

#### Rule 2 — Replace Consecutive All-Zero Groups with ::

**One or more** consecutive groups that are all zero (`0000`) can be replaced with `::` — but **only once** per address (RFC 5952).

```
RULE 2: Replace ONE sequence of consecutive zero groups with ::

  2001:db8:0:0:1:0:0:1
              ─────
              Two consecutive zero groups (groups 3 and 4)

  →  2001:db8::1:0:0:1   (:: replaces groups 3-4)

OR alternatively:
              ─────────
              Groups 6 and 7 are also consecutive zeros

  →  2001:db8:0:0:1::1   (:: replaces groups 6-7)

RFC 5952 rule: When two sequences of equal length exist,
               use :: for the FIRST one.
               When unequal, use :: for the LONGEST one.

EXAMPLES — applying both rules together:

  Full:     2001:0db8:0000:0000:0000:0000:0000:0001
  Rule 1:   2001:db8:0:0:0:0:0:1
  Rule 2:   2001:db8::1                      ← Final compressed form

  Full:     fe80:0000:0000:0000:021a:2bff:fe3c:4d5e
  Rule 1:   fe80:0:0:0:21a:2bff:fe3c:4d5e
  Rule 2:   fe80::21a:2bff:fe3c:4d5e         ← Final

  Full:     0000:0000:0000:0000:0000:0000:0000:0001
  Rule 1:   0:0:0:0:0:0:0:1
  Rule 2:   ::1                              ← Loopback! (like 127.0.0.1)

  Full:     0000:0000:0000:0000:0000:0000:0000:0000
  Rule 2:   ::                               ← Unspecified address (like 0.0.0.0)
```

#### Decompression — Reading a Compressed Address

To expand `::` back to full form:

```
Step 1: Count the groups present (not counting ::)
Step 2: :: fills in (8 - count) groups of all zeros
Step 3: Fill with 0000 to restore 8 groups

EXAMPLE: 2001:db8::1
  Groups present: 2001, db8, 1  →  count = 3
  :: replaces = 8 - 3 = 5 groups of zeros
  Expanded: 2001:0db8:0000:0000:0000:0000:0000:0001

EXAMPLE: fe80::1
  Groups present: fe80, 1  →  count = 2
  :: replaces = 8 - 2 = 6 groups of zeros
  Expanded: fe80:0000:0000:0000:0000:0000:0000:0001

COMMON MISTAKE — :: used twice (INVALID):
  INVALID: 2001::db8::1   ← Cannot determine where each :: goes
  Computers cannot parse this — RFC forbids it
```

#### Compression Practice Table

```
Full Address                              Compressed
──────────────────────────────────────    ─────────────────────────
2001:0db8:0000:0000:0000:0000:0000:0001   2001:db8::1
fe80:0000:0000:0000:021a:2bff:fe3c:4d5e   fe80::21a:2bff:fe3c:4d5e
0000:0000:0000:0000:0000:0000:0000:0001   ::1
0000:0000:0000:0000:0000:0000:0000:0000   ::
ff02:0000:0000:0000:0000:0000:0000:0001   ff02::1
2001:0db8:0001:0002:0003:0004:0005:0006   2001:db8:1:2:3:4:5:6
```

---

### IPv6 Address Types

Unlike IPv4's unicast/multicast/broadcast, IPv6 has three transmission types and no broadcast at all.

```
IPv6 Transmission Types:
  UNICAST:    One sender → one specific receiver
  MULTICAST:  One sender → multiple interested receivers (group)
  ANYCAST:    One sender → nearest member of a group (routing-based)

  NO BROADCAST:  IPv6 abolished broadcast completely.
                 Replaced by specific multicast groups.
```

#### Unicast Address Types

```
┌─────────────────────────────────────────────────────────────────────┐
│                     UNICAST ADDRESSES                               │
├──────────────────────┬──────────────────────┬───────────────────────┤
│ Type                 │ Prefix               │ Description           │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ Global Unicast       │ 2000::/3             │ Public routable       │
│                      │ (2000:: to 3fff::)   │ Like IPv4 public IPs  │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ Link-Local           │ fe80::/10            │ On-link only          │
│                      │ (fe80:: to febf::)   │ Not routable          │
│                      │                      │ Auto-configured       │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ Unique Local         │ fc00::/7             │ Private (like RFC1918)│
│                      │ fc00:: or fd00::     │ Routable within org   │
│                      │                      │ Not internet-routed   │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ Loopback             │ ::1/128              │ Like 127.0.0.1        │
│                      │                      │ Self-reference only   │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ Unspecified          │ ::/128               │ Like 0.0.0.0          │
│                      │                      │ "No address" / source │
│                      │                      │ before address assign │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ IPv4-mapped IPv6     │ ::ffff:0:0/96        │ Embedded IPv4 in IPv6 │
│                      │ e.g. ::ffff:192.0.2.1│ Dual-stack compat     │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ IPv4-compatible      │ ::IPv4 address       │ Deprecated (RFC 4291) │
│                      │ e.g. ::192.0.2.1     │ Do not use            │
└──────────────────────┴──────────────────────┴───────────────────────┘
```

**Global Unicast — The Public IPv6 Address**

```
Structure: 2001:0db8:85a3:0001:0000:8a2e:0370:7334/64
           ─────────────────────  ──────────────────────
           Routing prefix         Interface ID
           (network portion)      (host identifier)
           
           Broken down further:
           2001:0db8  = Global routing prefix (ISP assigned)
           :85a3:0001 = Subnet ID (organization assigns)
           :0000:8a2e:0370:7334 = Interface ID (auto or static)

Range: 2000:: to 3fff:ffff:ffff:ffff:ffff:ffff:ffff:ffff
       Any address with first 3 bits = 001

Current IANA allocation:
  2001::/32           IANA special purposes
  2001:db8::/32       Documentation (like 192.0.2.0/24) — NEVER use in prod!
  2001:20::/28        ORCHIDv2 (cryptographic identifiers)
  2002::/16           6to4 transition mechanism
  2600::/12           ARIN (North America)
  2a00::/12           RIPE NCC (Europe/Middle East)
  2400::/12           APNIC (Asia-Pacific)
```

**Link-Local — The Mandatory Auto-Generated Address**

```
Every IPv6 interface MUST have a link-local address.
Generated automatically when interface comes up.
Never needs configuration — always there.

Prefix: fe80::/10
Range:  fe80:: to febf::  (but fe80::/64 is always used in practice)

How it's generated (EUI-64 method):
  MAC address:  00:1a:2b:3c:4d:5e
  Step 1: Split at middle:  00:1a:2b | 3c:4d:5e
  Step 2: Insert ff:fe:      00:1a:2b:ff:fe:3c:4d:5e
  Step 3: Flip bit 7 of first byte:
          00 = 0000 0000  → flip bit 7 → 0000 0010 = 02
  Result: 02:1a:2b:ff:fe:3c:4d:5e
  Link-local: fe80::21a:2bff:fe3c:4d5e

PRIVACY ISSUE: MAC address is embedded in the address!
               Anyone can correlate your device across networks.
               
RFC 4941 Privacy Extensions:
  Generate RANDOM interface ID instead of EUI-64
  Preferred (temporary) address: random, changes periodically
  Deprecated address: old random ID, kept for existing connections
  
  Check on Linux:
  sysctl net.ipv6.conf.eth0.use_tempaddr  (1=prefer temp, 2=mandatory)

Link-local SCOPE — zone IDs:
  Link-local is only meaningful on a specific interface
  If a host has multiple interfaces, must specify which:
  
  ping6 fe80::1%eth0    (zone ID: the interface name after %)
  ping6 fe80::1%2       (zone ID: interface index)
  
  In URLs: http://[fe80::1%25eth0]/  (%25 = URL-encoded %)
```

**Unique Local — The Private IPv6 Address**

```
Equivalent to RFC 1918 private addresses, but routable within your org.
Generated with a pseudo-random 40-bit global ID — extremely unlikely to
collide when two organizations merge (unlike 10.0.0.0/8 everywhere).

Prefix structure:
  fc00::/7  =  fc00:: or fd00::
  
  Bit 8 (L bit):
    fc00::/8  (L=0): Defined, not implemented — don't use
    fd00::/8  (L=1): ALWAYS use fd::/8 for actual unique local
  
  Full structure:
  fd[40-bit random]::[16-bit subnet]:[64-bit interface ID]
    │  ─────────────  ─────────────  ────────────────────
    │  Global ID      Subnet ID      Interface ID
    │  (pseudo-random, (per subnet)   (EUI-64 or random)
    │   org-unique)
    │
    fd prefix (marks unique local)

Generate a proper ULA prefix:
  python3 -c "
  import os, struct
  rand = os.urandom(5)  # 40 random bits
  prefix = 'fd' + rand.hex()[:10]
  groups = [prefix[i:i+4] for i in range(0, len(prefix), 4)]
  print('fd' + ':'.join([prefix[i:i+4] for i in range(0, 10, 4)]) + '::/48')
  "
  # Example output: fd3a:9b2c:4e1f::/48

  Your org gets /48, subnet within it (/64 subnets), hosts within those.
  
Security note: 
  Unique local is NOT internet-routable (ISPs filter fc00::/7)
  But unlike link-local, it CAN route between internal sites
  Use for internal services that should never be internet-accessible
```

#### Multicast Address Types

```
Multicast replaces broadcast in IPv6. ALL broadcasts from IPv4 are now
specific multicast groups in IPv6.

Prefix: ff00::/8

Structure: ff[flags][scope]::[group ID]
              ──────  ─────
              4 bits  4 bits

Scope values:
  1  = Interface-local (loopback only)
  2  = Link-local      (same link/VLAN)
  4  = Admin-local     (configured boundary)
  5  = Site-local      (entire site)
  8  = Organization    (entire org)
  e  = Global          (internet-wide)

Key well-known multicast groups:
┌─────────────────┬────────────────────────────────────────────────┐
│ Address         │ Purpose                                        │
├─────────────────┼────────────────────────────────────────────────┤
│ ff02::1         │ All nodes on link (like IPv4 broadcast)        │
│ ff02::2         │ All routers on link                            │
│ ff02::5         │ OSPF routers (like 224.0.0.5)                  │
│ ff02::6         │ OSPF designated routers                        │
│ ff02::9         │ RIPng routers                                  │
│ ff02::a         │ EIGRP routers                                  │
│ ff02::d         │ PIM routers                                    │
│ ff02::16        │ MLDv2 routers (Multicast Listener Discovery)   │
│ ff02::fb        │ mDNS (Bonjour/Avahi — like 224.0.0.251)        │
│ ff02::1:2       │ DHCPv6 relay/server agents                     │
│ ff02::1:3       │ DHCPv6 server only                             │
│ ff02::1:ff__:__ │ Solicited-Node Multicast (NDP, replaces ARP)   │
│ ff05::1:3       │ DHCPv6 server (site-local)                     │
└─────────────────┴────────────────────────────────────────────────┘

Solicited-Node Multicast (critical for NDP/neighbor discovery):
  Format: ff02::1:ff[last 24 bits of unicast address]
  
  Example: host with 2001:db8::1a2b:3c4d:5e6f
  Last 24 bits:                          :3c:4d:5e6f → :3c4d:5e6f → last 24 = 4d:5e:6f
  Wait: last 24 bits of 5e6f = 5e6f's last 24 bits...
  
  Actually: take last 6 hex chars of the address = 4d5e6f
  Solicited-node: ff02::1:ff4d:5e6f
  
  All hosts join this group for their own unicast/anycast addresses
  NDP Neighbor Solicitation sent HERE instead of broadcasting ARP
  → Dramatically reduces "ARP broadcast storm" equivalent in IPv6
```

#### Anycast Addresses

```
ANYCAST: Same address assigned to MULTIPLE devices.
         Packets delivered to the NEAREST one (by routing metric).

No special prefix — anycast uses the same address space as unicast.
Must be explicitly configured as anycast on the interface.

Use cases:
  DNS root servers (k.root-servers.net = anycast → nearest root)
  CDN edge nodes (same IP, nearest PoP serves you)
  IPv6 subnet-router anycast (mandatory built-in)

Subnet-router anycast (automatic, per RFC 4291):
  Network prefix + all-zeros interface ID
  Example: prefix 2001:db8:1::/64
  Subnet-router anycast = 2001:db8:1:: (host part all zeros)
  Automatically assigned to all routers on that subnet
  
Security note:
  Anycast makes IP-based geolocation inaccurate
  An attacker could route anycast traffic to themselves
  (BGP anycast hijacking — same issue as with CDN IPs)
```

---

### IPv6 Prefix Notation and Subnetting

#### Prefix Length (CIDR in IPv6)

IPv6 uses CIDR notation identically to IPv4. The `/prefix-length` specifies how many bits are the network portion.

```
Notation: 2001:db8:1::/48
          └──────────────┘ ├─┘
          Network address  Prefix length

Common prefix lengths:
  /128  Single host route (like IPv4 /32)
  /64   Standard LAN segment (the universal default!)
  /48   Site/organization allocation (from ISP)
  /32   ISP allocation to large customer
  /19   Regional internet registry (RIR) to ISP
  /12   IANA to RIR (huge block)

WHY /64 IS THE UNIVERSAL STANDARD:
  SLAAC (auto-configuration) requires /64
  EUI-64 interface ID generation requires exactly 64 bits for host part
  RFC 4291 defines /64 as the standard subnet size
  Breaking this convention breaks auto-configuration

  EXCEPTION: Point-to-point links use /127 (RFC 6164)
  (Like IPv4's /30 or /31 for P2P links)
  
  EXCEPTION: Loopback uses /128 (single address)
```

#### IPv6 Subnetting — Worked Example

```
SCENARIO: ISP assigns 2001:db8:abcd::/48 to your organization
          You need to subdivide it for different sites and VLANs.

Your block:  2001:0db8:abcd:0000:0000:0000:0000:0000 /48
                                ────  ←  16 bits available for subnetting
             2001:0db8:abcd:[SUBNET]::[INTERFACE-ID]

With /48 you control bits 49-64 (the subnet field = 16 bits)
That gives you 2^16 = 65,536 possible /64 subnets!

Subnet allocation plan:
  Site HQ (first 256 subnets: 0000–00ff):
    2001:db8:abcd:0000::/64  VLAN 10 - Users
    2001:db8:abcd:0001::/64  VLAN 20 - Voice
    2001:db8:abcd:0002::/64  VLAN 30 - Servers
    2001:db8:abcd:0003::/64  VLAN 40 - Management
    2001:db8:abcd:0004::/64  VLAN 50 - Guest WiFi
    2001:db8:abcd:00ff::/64  Reserved

  Site Branch-A (subnets 0100–01ff):
    2001:db8:abcd:0100::/64  Branch-A Users
    2001:db8:abcd:0101::/64  Branch-A Servers
    ...

  Site Branch-B (subnets 0200–02ff):
    2001:db8:abcd:0200::/64  Branch-B Users
    ...

  Point-to-Point links (subnet ffxx):
    2001:db8:abcd:ff00::/127  HQ ↔ Branch-A router link
    2001:db8:abcd:ff02::/127  HQ ↔ Branch-B router link
    2001:db8:abcd:ff04::/127  HQ ↔ Internet router
    (Each /127 gives exactly 2 addresses for P2P)

BENEFIT OVER IPv4:
  With IPv4 you'd be rationing addresses carefully.
  With IPv6 you have 65,536 entire /64 subnets — plan generously!
```

#### IPv6 Address in URLs and Configuration

```
IPv6 addresses in URLs must be wrapped in square brackets:
  http://[2001:db8::1]/                    ← Global unicast
  http://[::1]/                             ← Loopback (localhost)
  http://[fe80::1%25eth0]/                  ← Link-local (% encoded)
  http://[2001:db8::1]:8080/                ← With port number

Configuration files:
  /etc/hosts:
    ::1         localhost
    2001:db8::1 myserver.example.com
    
  Nginx config:
    listen [::]:443 ssl;  ← Listen on all IPv6 interfaces, port 443
    
  SSH:
    ssh user@2001:db8::1
    ssh user@[fe80::1%eth0]  ← Link-local requires interface
```

---

### IPv6 Address Configuration Methods

```
METHOD 1: STATIC (Manual)
  ip -6 addr add 2001:db8:1::50/64 dev eth0
  ip -6 route add default via 2001:db8:1::1
  
  In /etc/network/interfaces:
    iface eth0 inet6 static
    address 2001:db8:1::50
    netmask 64
    gateway 2001:db8:1::1

METHOD 2: SLAAC (Stateless Address AutoConfiguration)
  No DHCPv6 server needed!
  
  1. Interface generates link-local: fe80::[EUI-64 or random]
  2. Sends Router Solicitation (RS) to ff02::2 (all routers)
  3. Router replies with Router Advertisement (RA):
     - Prefix: 2001:db8:1::/64
     - Default gateway: router's link-local
     - Flags: A=1 (SLAAC), M=0 (no DHCPv6 for address)
  4. Host builds address: prefix + EUI-64 or privacy random
  5. Performs Duplicate Address Detection (DAD) via NDP
  6. Address is ready!
  
  Check SLAAC on Linux:
  radvd (router side) or:
  sysctl net.ipv6.conf.eth0.autoconf  (should be 1)

METHOD 3: DHCPv6 (Stateful)
  Like DHCPv4 but for IPv6
  Provides: address, DNS, domain search list, NTP
  
  DHCPv6 port: UDP 546 (client), UDP 547 (server)
  Client sends to ff02::1:2 (all DHCPv6 relay/servers)
  
  DHCPv6 message types:
    Solicit      ← Client: "I need an address"
    Advertise    ← Server: "I offer this"
    Request      ← Client: "I accept"
    Reply        ← Server: "Confirmed"
    (Simpler than DORA but same concept)
    
  RA flags control which method is used:
    M flag = 1: Use DHCPv6 for address (stateful)
    O flag = 1: Use DHCPv6 for other info (DNS, NTP) but not address
    A flag = 1: SLAAC ok
    
  Most common combo:
    M=0, O=1, A=1 = SLAAC for address + DHCPv6 for DNS
    M=1, O=1, A=0 = Full DHCPv6 (like IPv4 DHCP)

METHOD 4: Privacy Extensions (RFC 4941)
  Temporary random address for outbound connections
  Permanent address (EUI-64 or static) for inbound/server use
  
  Linux config:
  sysctl -w net.ipv6.conf.eth0.use_tempaddr=2
  # 0=disabled, 1=enable but prefer public, 2=prefer temporary
```

---

### IPv4 vs IPv6 — Complete Equivalents Reference

```
╔══════════════════════════════════════════════════════════════════════════════╗
║              IPv4 TO IPv6 COMPLETE EQUIVALENTS TABLE                        ║
╠══════════════════════╦════════════════════════════╦══════════════════════════╣
║ IPv4                 ║ IPv6 Equivalent            ║ Notes                   ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ ADDRESSES                                                                   ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ 0.0.0.0              ║ ::  (::/128)               ║ Unspecified             ║
║ 127.0.0.1            ║ ::1  (::1/128)             ║ Loopback                ║
║ 255.255.255.255       ║ No equivalent              ║ Broadcast abolished     ║
║ 10.0.0.0/8           ║ fc00::/7 (ULA fd::/8)      ║ Private/unique local    ║
║ 172.16.0.0/12        ║ fc00::/7 (ULA fd::/8)      ║ Private/unique local    ║
║ 192.168.0.0/16       ║ fc00::/7 (ULA fd::/8)      ║ Private/unique local    ║
║ 169.254.0.0/16       ║ fe80::/10                  ║ Link-local              ║
║ 192.0.2.0/24         ║ 2001:db8::/32              ║ Documentation only!     ║
║ 198.51.100.0/24      ║ 2001:db8::/32              ║ Documentation only!     ║
║ 203.0.113.0/24       ║ 2001:db8::/32              ║ Documentation only!     ║
║ Public addresses      ║ 2000::/3                   ║ Global unicast          ║
║ 224.0.0.0/4          ║ ff00::/8                   ║ Multicast               ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ PROTOCOLS & MECHANISMS                                                       ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ ARP                  ║ NDP (Neighbor Discovery)   ║ ICMPv6 Types 135/136   ║
║ RARP                 ║ DHCPv6 / SLAAC             ║ Address resolution      ║
║ DHCP (DHCPv4)        ║ DHCPv6 / SLAAC             ║ Port 547/546 UDP        ║
║ IGMP                 ║ MLD (Multicast Listener    ║ ICMPv6 Types 130-132   ║
║                      ║ Discovery)                 ║                         ║
║ ICMP                 ║ ICMPv6 (expanded role)     ║ Cannot fully block!     ║
║ OSPF (224.0.0.5/6)   ║ OSPFv3 (ff02::5/6)         ║ Protocol same, addr diff║
║ RIP (224.0.0.9)      ║ RIPng (ff02::9)            ║                         ║
║ EIGRP (224.0.0.10)   ║ EIGRPv6 (ff02::a)          ║                         ║
║ IP fragmentation     ║ End-to-end only (no router ║ Routers don't fragment  ║
║ (routers can frag.)  ║ fragmentation)             ║ PMTUD mandatory!        ║
║ IP header checksum   ║ No header checksum         ║ Removed (L2+L4 check)  ║
║ IP options           ║ Extension headers          ║ More flexible           ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ IMPORTANT MULTICAST REPLACEMENTS FOR BROADCAST                               ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ 255.255.255.255      ║ ff02::1 (all nodes)        ║ Limited broadcast→mcast ║
║ Subnet broadcast     ║ ff02::1 (all nodes link)   ║ No directed broadcast   ║
║ 224.0.0.1 (all hosts)║ ff02::1                    ║ All nodes on link       ║
║ 224.0.0.2 (routers)  ║ ff02::2                    ║ All routers on link     ║
║ DHCP broadcast disc. ║ ff02::1:2 (DHCPv6 srv)     ║ DHCPv6 Solicit target   ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ SUBNET SIZING                                                                ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ /32  (1 host)        ║ /128 (1 host)              ║ Host route              ║
║ /30  (2 hosts P2P)   ║ /127 (2 hosts P2P)         ║ RFC 6164 P2P links      ║
║ /29  (6 hosts)       ║ /64 (18 quintillion hosts) ║ Smallest standard LAN   ║
║ /24  (254 hosts)     ║ /64 (standard segment)     ║ Always /64 for LANs     ║
║ /16  (65534 hosts)   ║ /48 (65536 × /64 subnets)  ║ Site allocation         ║
║ /8   (16M hosts)     ║ /32 (4B × /64 subnets)     ║ ISP allocation          ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ SECURITY & NAT                                                               ║
╠══════════════════════╬════════════════════════════╬══════════════════════════╣
║ NAT (RFC 1918 reason)║ No NAT needed (by design)  ║ Every device routable   ║
║ IPsec optional       ║ IPsec integrated (optional ║ Built into IPv6 spec    ║
║                      ║ in practice)               ║ but still negotiated    ║
║ APIPA (169.254/16)   ║ Link-local (fe80::/10)     ║ Auto-assign on no-DHCP  ║
╚══════════════════════╩════════════════════════════╩══════════════════════════╝
```

---

### IPv6 Header Comparison with IPv4

```
IPv4 HEADER (20 bytes minimum, up to 60 bytes with options):
┌────┬────┬────────────────────┬──────────────────────────────────┐
│Ver │IHL │Type of Service     │          Total Length             │
│ 4b │ 4b │        8b          │               16b                 │
├────┴────┴────────────────────┼──────────────────────────────────┤
│         Identification       │ Flags │      Fragment Offset      │
│              16b             │  3b   │           13b             │
├──────────────────────────────┴───────┴───────────────────────────┤
│        TTL        │  Protocol  │          Header Checksum        │
│         8b        │     8b     │               16b               │
├───────────────────┴────────────┴─────────────────────────────────┤
│                       Source Address (32 bits)                   │
├──────────────────────────────────────────────────────────────────┤
│                    Destination Address (32 bits)                 │
├──────────────────────────────────────────────────────────────────┤
│                    Options (variable, 0-40 bytes)                │
└──────────────────────────────────────────────────────────────────┘
Total: variable 20–60 bytes. Complicated. Every router must parse options.

IPv6 HEADER (FIXED 40 bytes, always):
┌────┬────────────────────────────────┬─────────────────────────────┐
│Ver │  Traffic Class  │          Flow Label                        │
│ 4b │       8b        │              20b                          │
├────┴─────────────────┴──────────────────────────────────────────┤
│          Payload Length (16b)      │ Next Header (8b) │Hop Limit │
│                                    │                  │    8b    │
├────────────────────────────────────┴──────────────────┴──────────┤
│                                                                   │
│                     Source Address (128 bits)                    │
│                                                                   │
├───────────────────────────────────────────────────────────────────┤
│                                                                   │
│                  Destination Address (128 bits)                   │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
Total: ALWAYS exactly 40 bytes. Simple. Routers process faster.

KEY FIELD CHANGES:
  TTL → Hop Limit         (same function, better name)
  Protocol → Next Header  (can chain extension headers)
  TOS → Traffic Class     (same function, DSCP/ECN)
  New: Flow Label         (20 bits for QoS flow identification)
  REMOVED: Header Checksum (saves processing at every hop)
  REMOVED: Fragmentation fields (moved to extension header)
  REMOVED: IHL (header always 40 bytes — no options field needed)

EXTENSION HEADERS (chained via Next Header field):
  0  = Hop-by-Hop Options (every router reads this)
  43 = Routing Header (loose/strict source routing)
  44 = Fragment Header (end-to-end fragmentation only)
  50 = ESP (IPsec Encapsulating Security Payload)
  51 = AH (IPsec Authentication Header)
  59 = No Next Header (end of chain)
  60 = Destination Options (only destination reads)
  135 = Mobility Header (mobile IPv6)
  
  Chain example: IPv6 → HopByHop → Fragment → ESP → TCP → Data
                 Each header has its own Next Header field pointing to next
```

---

### IPv6 Hands-On Commands Reference

```bash
# ── VIEWING IPv6 ADDRESSES ────────────────────────────────────────
ip -6 addr show                          # All IPv6 addresses
ip -6 addr show dev eth0                 # IPv6 on specific interface
ip -6 addr show scope global             # Only global unicast
ip -6 addr show scope link               # Only link-local

# ── PING AND REACHABILITY ─────────────────────────────────────────
ping6 ::1                                # Ping loopback
ping6 fe80::1%eth0                       # Ping link-local (scope needed!)
ping6 2001:db8::1                        # Ping global address
ping -6 2001:db8::1                      # Alternative syntax

# ── ROUTING ───────────────────────────────────────────────────────
ip -6 route show                         # IPv6 routing table
ip -6 route add 2001:db8:1::/48 via 2001:db8::1 dev eth0
ip -6 route add default via fe80::1%eth0 # Default route via link-local GW

# ── DNS ───────────────────────────────────────────────────────────
dig AAAA google.com                      # Get IPv6 address for domain
dig -t AAAA google.com @2001:4860:4860::8888  # Query Google's IPv6 DNS
nslookup -type=AAAA google.com           # Alternative

# ── TRACEROUTE ────────────────────────────────────────────────────
traceroute6 2001:4860:4860::8888         # Trace to Google DNS
tracepath6 2001:db8::1                   # No root needed

# ── NEIGHBOR DISCOVERY (NDP = IPv6 ARP) ───────────────────────────
ip -6 neigh show                         # Neighbor cache (like ARP table)
ip -6 neigh show dev eth0                # For specific interface
# Entries: REACHABLE = confirmed, STALE = needs revalidation
#          INCOMPLETE = pending NDP reply, FAILED = no response

# Manually trigger NDP (like "arping" for IPv6):
ndisc6 2001:db8::1 eth0                  # Solicited neighbor discovery

# ── SECURITY CHECKS ───────────────────────────────────────────────
# Check if link-local is properly configured:
ip -6 addr show scope link | grep -v "tentative\|dadfailed"

# Check for DAD failures (Duplicate Address Detection):
ip -6 addr show | grep dadfailed         # If any: address conflict!

# Check which privacy extension level is set:
sysctl net.ipv6.conf.eth0.use_tempaddr   # 0=off,1=prefer stable,2=prefer temp
sysctl net.ipv6.conf.eth0.autoconf      # 1=SLAAC enabled

# Check if IPv6 forwarding (routing) is enabled:
sysctl net.ipv6.conf.all.forwarding     # 1=router, 0=host

# ── NETWORK ENUMERATION ────────────────────────────────────────────
# Nmap IPv6 host scan (specify -6):
nmap -6 -sn 2001:db8:1::/64             # Ping sweep (may miss SLAAC hosts)
nmap -6 -sV 2001:db8::1                  # Service detection

# Discover all on-link hosts via NDP:
ping6 ff02::1%eth0                       # Ping all-nodes multicast
ip -6 neigh show                         # Check who responded

# Alternative multicast discovery:
sudo nmap -6 --script=targets-ipv6-multicast-slaac --script-args 'newtargets,interface=eth0'

# ── PYTHON FOR IPv6 ───────────────────────────────────────────────
python3 << 'EOF'
import ipaddress

# Parse and expand IPv6 address
addr = ipaddress.IPv6Address("2001:db8::1")
print(f"Full:      {addr.exploded}")    # 2001:0db8:0000:0000:0000:0000:0000:0001
print(f"Compressed:{addr.compressed}")  # 2001:db8::1
print(f"Is private:{addr.is_private}")  # True for ULA/link-local
print(f"Is global: {addr.is_global}")   # True for 2000::/3

# Work with networks/subnets
net = ipaddress.IPv6Network("2001:db8:abcd::/48")
print(f"\nNetwork: {net.network_address}")
print(f"Prefix:  {net.prefixlen}")
print(f"Hosts:   {net.num_addresses:,}")

# List /64 subnets within /48 (first 5):
subnets = list(net.subnets(new_prefix=64))
for s in subnets[:5]:
    print(f"  Subnet: {s}")

# Check containment:
host = ipaddress.IPv6Address("2001:db8:abcd:1::50")
print(f"\n{host} in {net}: {host in net}")

# Compress an address:
full = "2001:0db8:0000:0000:0000:0000:0000:0001"
print(f"\nFull:       {full}")
print(f"Compressed: {ipaddress.IPv6Address(full).compressed}")
EOF
```

---

### IPv6 Security Considerations (Beyond Section 12)

```
1. FIREWALL RULES MUST COVER IPv6 SEPARATELY:
   iptables = IPv4 only
   ip6tables = IPv6 only
   nftables = handles both (preferred)
   
   Common mistake: All iptables rules, no ip6tables rules
   → IPv6 traffic completely unfiltered!
   
   ip6tables -A INPUT -p tcp --dport 22 -j ACCEPT  # IPv6 SSH rule
   ip6tables -A INPUT -j DROP                       # IPv6 default drop
   
   Or with nftables (covers both):
   table inet filter {  # inet = applies to IPv4 AND IPv6
     chain input { ... }
   }

2. EXTENSION HEADER ABUSE:
   Hop-by-Hop header processed by EVERY router → DoS risk
   Long extension header chains can confuse firewalls
   Some implementations process headers in wrong order
   
   Filtering: Drop packets with unusual extension headers
   (Routing Header type 0 is banned — RFC 5095)

3. NDP SPOOFING (IPv6 ARP poisoning):
   Attacker sends fake Neighbor Advertisement
   → Redirects traffic to attacker (IPv6 MITM)
   
   Tools: parasite6, fake_router6 (THC-IPv6 toolkit)
   
   Defense: RA Guard on switches (RFC 6105)
             SEND (Secure Neighbor Discovery, RFC 3971)
             Dynamic ND inspection on managed switches

4. ROGUE ROUTER ADVERTISEMENT (RA):
   Anyone can send RA on the network
   → All hosts update their default gateway
   → Attacker = IPv6 MITM for entire segment
   
   This is worse than rogue DHCP! No authentication needed.
   RA Guard: switch drops RA from non-router ports.
   
5. TUNNELING BYPASSES IPv4-ONLY FIREWALLS:
   6to4 (192.88.99.0/24): IPv6 over IPv4 protocol 41
   Teredo: IPv6 over UDP 3544 (through NAT!)
   ISATAP: intra-site tunneling
   
   If your firewall only filters TCP/UDP, IPv6 protocol 41
   tunnels create unfiltered IPv6 paths through IPv4 networks.
   
   Block: protocol 41 at edge firewall
          UDP port 3544 (Teredo)
          If not using these tunnels: safe to block

6. PRIVACY — DEVICE TRACKING:
   EUI-64 embeds MAC in IPv6 address → track device across networks
   RFC 4941 privacy extensions: use temporary random interface IDs
   
   But: many organizations disable privacy extensions for logging!
   Trade-off: privacy vs auditability
```

---

*This section is an addition to `Networking_Deep_Dive_TCPIP_and_Devices.md`, inserted between Section 4 (Subnetting) and Section 5 (CIDR). It complements the IPv6 architecture content in Section 12 of the same document with full addressing details.*

*Standards: RFC 4291 (IPv6 Addressing Architecture), RFC 5952 (IPv6 Text Representation),
RFC 4193 (Unique Local IPv6 Unicast Addresses), RFC 4862 (SLAAC),
RFC 6164 (Using /127 Prefix Length Between Routers),
RFC 4941 (Privacy Extensions), RFC 3315 (DHCPv6)*