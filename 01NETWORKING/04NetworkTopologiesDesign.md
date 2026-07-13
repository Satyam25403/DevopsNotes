# Network Topologies & Design — Field Manual
### Physical & Logical Topologies | Modern Architectures | Network Partitioning | Admin Best Practices

> **Series Supplement:** Companion to `Networking_Deep_Dive_TCPIP_and_Devices.md` (Module 2)
> **Audience:** Network administrators, security engineers, systems architects, cybersecurity students
> **Purpose:** Understand how networks are physically and logically structured, how to design them correctly, and how to segment and manage them securely — the foundation every attacker studies and every defender must master.

---

## Table of Contents

### PART 1 — PHYSICAL TOPOLOGIES
1. [What Physical Topology Means](#1-what-physical-topology-means)
2. [Bus Topology](#2-bus-topology)
3. [Ring Topology](#3-ring-topology)
4. [Star Topology](#4-star-topology)
5. [Mesh Topology (Full & Partial)](#5-mesh-topology)
6. [Tree / Hierarchical Topology](#6-tree--hierarchical-topology)
7. [Hybrid Topology](#7-hybrid-topology)
8. [Point-to-Point Topology](#8-point-to-point-topology)

### PART 2 — LOGICAL TOPOLOGIES
9. [What Logical Topology Means — Physical vs Logical](#9-physical-vs-logical-topology)
10. [Logical Bus (Ethernet/CSMA-CD)](#10-logical-bus)
11. [Logical Ring (Token Ring, SONET)](#11-logical-ring)
12. [Logical Star (Switched Ethernet)](#12-logical-star)
13. [Logical Mesh (BGP/OSPF Routing)](#13-logical-mesh)

### PART 3 — MODERN NETWORK ARCHITECTURES
14. [Three-Tier Architecture (Core / Distribution / Access)](#14-three-tier-architecture)
15. [Two-Tier (Collapsed Core) Architecture](#15-two-tier-collapsed-core)
16. [Spine-Leaf Architecture (Data Center)](#16-spine-leaf-architecture)
17. [Software-Defined Networking (SDN)](#17-software-defined-networking-sdn)
18. [Campus Network Design](#18-campus-network-design)
19. [Data Center Network Design](#19-data-center-network-design)
20. [WAN Topologies — MPLS, SD-WAN, Internet VPN](#20-wan-topologies)

### PART 4 — NETWORK PARTITIONING
21. [Why Segment a Network](#21-why-segment-a-network)
22. [VLANs — Layer 2 Segmentation](#22-vlans--layer-2-segmentation)
23. [Subnetting — Layer 3 Segmentation](#23-subnetting--layer-3-segmentation)
24. [DMZ Design — Isolating Public-Facing Services](#24-dmz-design)
25. [Zero-Trust Micro-Segmentation](#25-zero-trust-micro-segmentation)
26. [Air-Gapped Networks](#26-air-gapped-networks)

### PART 5 — ROUTING STRATEGIES
27. [Inter-VLAN Routing — Router on a Stick vs L3 Switch](#27-inter-vlan-routing)
28. [Route Summarization and Hierarchy](#28-route-summarization-and-hierarchy)
29. [Policy-Based Routing](#29-policy-based-routing)

### PART 6 — NETWORK ADMIN BEST PRACTICES
30. [IP Addressing Design and Planning](#30-ip-addressing-design-and-planning)
31. [Network Documentation Standards](#31-network-documentation-standards)
32. [Change Management](#32-change-management)
33. [Monitoring and Observability](#33-monitoring-and-observability)
34. [Network Security Hardening](#34-network-security-hardening)
35. [Redundancy and High Availability](#35-redundancy-and-high-availability)

---

# PART 1 — PHYSICAL TOPOLOGIES

---

## 1. What Physical Topology Means

**Physical topology** describes how devices are **physically connected** — the actual cable runs, the locations of switches and routers, the physical layout on a floor or campus. It answers: if you traced the wire from Device A, where would it physically go?

**Logical topology** (Part 2) describes how **data flows** regardless of physical layout. A network can look like a star physically but behave like a bus logically.

```
QUICK COMPARISON:

Physical topology = the MAP of cables and hardware
Logical topology  = the FLOW of data through the network

Example:
  Physical: Every device plugged into a central switch (STAR)
  Logical:  Data broadcast to all devices (BUS — old Ethernet behavior)
  
  Same physical setup, different logical behavior depending on protocol.
```

---

## 2. Bus Topology

```
PHYSICAL DIAGRAM:

Device A    Device B    Device C    Device D
   │           │           │           │
───┼───────────┼───────────┼───────────┼───
            Single Shared Cable (backbone)
                    │
                Terminator at each end (prevents signal reflection)

HOW IT WORKS:
  All devices connect to ONE shared cable (the "bus")
  When any device transmits, signal travels entire length of cable
  All devices receive every transmission
  Devices must wait for cable to be free before transmitting
  (CSMA/CD — Carrier Sense Multiple Access with Collision Detection)
```

```
BENEFITS:
  ✓ Simple and cheap to install (one cable run)
  ✓ Easy to extend (tap into the backbone)
  ✓ Works well for small, simple networks
  ✓ If one device fails, rest of network unaffected
  ✓ Low cable cost (no central hub needed)

LIMITATIONS:
  ✗ Single point of failure: cable break = ENTIRE network down
  ✗ Performance degrades as devices are added (shared bandwidth)
  ✗ Difficult to troubleshoot (hard to isolate faults)
  ✗ Cable length limits (Ethernet: 10BASE-2 max 185m, 10BASE-5 max 500m)
  ✗ Collisions increase with more devices (poor scalability)
  ✗ Security risk: every device sees every packet (promiscuous mode capture)
  ✗ Adding/removing devices disrupts entire network

WHEN USED TODAY:
  Almost never in modern LANs. Legacy coaxial networks only.
  Conceptually lives on in shared wireless medium (Wi-Fi is a logical bus).
  Used in some industrial control/automotive contexts (CAN bus).
  
REAL-WORLD: Early Ethernet (10BASE-2 "cheapernet", 10BASE-5 "thicknet")
```

---

## 3. Ring Topology

```
PHYSICAL DIAGRAM:

        Device A
       /         \
  Device D       Device B
       \         /
        Device C

Each device connects to exactly TWO neighbors
Data travels in ONE direction around the ring (or both in dual-ring)

TOKEN RING (logical ring on physical ring):
  A "token" travels around the ring continuously
  Device that wants to transmit must CAPTURE the token first
  No collisions possible (only one device transmits at a time)
  After transmitting: releases token to next device

SONET/SDH (dual ring):
        Device A
       // \\
  Device D   Device B     ← Two counter-rotating rings
       \\ //
        Device C
  Primary ring (clockwise) + Protection ring (counter-clockwise)
  If primary ring breaks: protection ring activates automatically
```

```
BENEFITS:
  ✓ Predictable performance: no collisions (token-based)
  ✓ Equal access for all devices
  ✓ Well-suited for larger networks than bus
  ✓ Dual-ring provides redundancy (SONET self-healing ring)
  ✓ Deterministic latency (important for real-time applications)
  ✓ Easy to add devices (just insert into ring)

LIMITATIONS:
  ✗ Single ring failure = entire network down (single ring only)
  ✗ Troubleshooting is complex
  ✗ Adding/removing device temporarily disrupts network
  ✗ Slower than star topologies for point-to-point communication
  ✗ Entire ring depends on each device functioning correctly
  ✗ Token passing overhead reduces effective throughput

WHEN USED TODAY:
  SONET/SDH rings: still backbone of many telco/carrier networks
  FDDI (Fiber Distributed Data Interface): campus backbones in 1990s
  Token Ring (IBM): legacy enterprise, mostly gone
  MSTP/G.8032: Ethernet ring protection in service provider networks
  RPR (Resilient Packet Ring): metro Ethernet rings
```

---

## 4. Star Topology

```
PHYSICAL DIAGRAM:

  Device A    Device B    Device C
      \           |           /
       \          |          /
        ┌─────────────────────┐
        │   Central Switch    │
        │   (or Hub)          │
        └─────────────────────┘
              /     |     \
        Device D  Device E  Device F

THE DOMINANT TOPOLOGY IN MODERN LANs
Every device has a DEDICATED point-to-point link to the central device
The central device (switch) manages all communication

HUB-BASED STAR (legacy, logical bus):
  Hub receives signal → broadcasts to ALL ports
  All devices share bandwidth
  
SWITCH-BASED STAR (modern, logical point-to-point):
  Switch receives frame → learns MAC → forwards ONLY to correct port
  Each port gets DEDICATED bandwidth (full-duplex!)
  No collisions
  This is what every modern LAN uses
```

```
BENEFITS:
  ✓ Easy to install and configure
  ✓ Easy to troubleshoot (isolate to individual cable or port)
  ✓ One device/cable failure doesn't affect others
  ✓ Easy to add/remove devices (just plug/unplug)
  ✓ Centralized management and monitoring
  ✓ With switches: full dedicated bandwidth per device, no collisions
  ✓ Supports different cable types/speeds on different ports
  ✓ Scalable (just add more switches)

LIMITATIONS:
  ✗ Central switch = single point of failure (mitigated with redundant switches)
  ✗ More cable needed than bus (dedicated run to every device)
  ✗ Switch cost (central device required)
  ✗ Performance bounded by switch capacity
  ✗ Physical distance limited by cable type (Cat6: 100m per segment)

WHEN USED TODAY:
  Universally — every modern LAN, office, data center, campus
  Wi-Fi uses logical star (AP = center, clients = endpoints)
  
BEST PRACTICE:
  Always use managed switches (not unmanaged/hubs)
  Enable STP/RSTP to prevent loops if using multiple switches
  Consider stacking or chassis switches for high density
  Plan cable runs during building construction (much cheaper)
```

---

## 5. Mesh Topology

```
FULL MESH DIAGRAM (every device connects to every other):

Device A ────── Device B
  │  ╲            │  ╲
  │    ╲          │    ╲
  │     Device C──┼─────Device D
  │       │       │       │
  └───────┴───────┴───────┘
  
  Formula: n(n-1)/2 links for n devices
  4 devices = 4×3/2 = 6 links
  10 devices = 45 links  ← expensive!
  100 devices = 4,950 links ← impractical

PARTIAL MESH (only some devices are fully meshed):

Device A ────── Device B
  │                │
  │           Device C
  │
Device D ────── Device E
(Core devices fully meshed, edge devices star-connected to core)

INTERNET = PARTIAL MESH (logical):
  Core ISP routers are heavily meshed (many BGP peering links)
  Edge routers connect to core via fewer links
  This is how global routing redundancy works
```

```
BENEFITS (Full Mesh):
  ✓ Maximum redundancy: path exists even if multiple links fail
  ✓ No single point of failure
  ✓ Traffic can take optimal path between any two nodes
  ✓ High fault tolerance
  ✓ Good performance (dedicated links between pairs)

LIMITATIONS (Full Mesh):
  ✗ Extremely expensive: n(n-1)/2 links
  ✗ Complex to configure and manage
  ✗ Physical cabling is impractical at scale
  ✗ Every device needs n-1 interfaces
  ✗ Difficult to troubleshoot

PARTIAL MESH BENEFITS:
  ✓ Balance between cost and redundancy
  ✓ Critical links meshed, less critical use hub-and-spoke
  ✓ Used in: WAN design, data center core, ISP backbones
  
WHEN USED TODAY:
  Full mesh: Small critical networks (2-4 core routers in small DCs)
  Partial mesh: WAN design, ISP backbones, data center interconnects
  Logical mesh: BGP routing on the internet (underlying topology is partial mesh)
  SD-WAN virtual mesh: Full mesh tunnels over internet without physical links
```

---

## 6. Tree / Hierarchical Topology

```
PHYSICAL DIAGRAM:

                    [Root Switch/Router]
                           │
              ┌────────────┼────────────┐
              │            │            │
        [Building A]  [Building B]  [Building C]
        Distribution  Distribution  Distribution
              │            │            │
          ┌───┴───┐    ┌───┴───┐    ┌───┴───┐
       [Floor 1][Floor 2] [F1][F2] [F1][F2][F3]
        Access   Access
          │         │
       Devices   Devices

THIS IS THE THREE-TIER MODEL:
  Root = Core Layer       (high-speed backbone)
  Building = Distribution Layer (policy, routing, aggregation)
  Floor = Access Layer    (end-device connectivity)

Physical topology: extended star (stars connected to stars)
Logical topology: hierarchical routing/switching
```

```
BENEFITS:
  ✓ Highly scalable — add branches without redesigning
  ✓ Easy to manage in sections (fault isolation per branch)
  ✓ Clear traffic flow patterns (up/across/down the hierarchy)
  ✓ Supports large organizations
  ✓ Enables hierarchical addressing (each branch = subnet block)
  ✓ Policy enforcement at each tier

LIMITATIONS:
  ✗ Root node failure affects entire network
  ✗ Multiple levels add latency
  ✗ Redundancy at each level increases cost
  ✗ Complex configuration at scale

WHEN USED TODAY:
  EVERYWHERE — campus networks, enterprise networks
  The three-tier model is the reference architecture for enterprise LANs
  Most Fortune 500 companies run some variation of this

BEST PRACTICE:
  Redundant root/core nodes (two core switches, stacked or L2 bonded)
  ECMP or port-channels between distribution and core
  Spanning Tree only at access layer (L3 as high as possible)
```

---

## 7. Hybrid Topology

```
DIAGRAM (Star-Bus Hybrid — common in older enterprise):

   Building 1 (Star):              Building 2 (Star):
   Switch ─── PC1                  Switch ─── PC1
   Switch ─── PC2                  Switch ─── PC2
   Switch ─── PC3                  Switch ─── PC3
       │                               │
       └──────────── Bus ──────────────┘
                (backbone cable)

MODERN HYBRID (Star-Mesh — actual enterprise design):

         [Core Switch A]════[Core Switch B]  ← MESH (fully redundant)
              ║                    ║
    ┌─────────╫──────┐   ┌────────╫──────────┐
    │    Dist SW 1   │   │   Dist SW 2        │
    │   (Building A) │   │  (Building B)      │
    └───┬────────────┘   └────────────┬───────┘
        │                             │
   Access SW (Star)            Access SW (Star)
   ┌────┼────┐                 ┌─────┼──────┐
  PC1  PC2  PC3               PC1   PC2    PC3

Core layer: MESH (full redundancy)
Distribution layer: dual uplinks to both core switches (partial mesh)
Access layer: STAR (simpler, cheaper at the edge)

This is the dominant real-world enterprise topology.
```

```
BENEFITS:
  ✓ Combines advantages of multiple topologies
  ✓ Redundancy where it matters (core), simplicity where it's fine (edge)
  ✓ Flexible to match organizational needs
  ✓ Most practical topology for real networks

LIMITATIONS:
  ✗ Complex to design and document
  ✗ Requires more expertise to manage
  ✗ Troubleshooting requires understanding the full hybrid design

WHEN USED TODAY:
  Every large enterprise network is a hybrid topology
  The star-mesh hybrid (described above) is the industry standard
```

---

## 8. Point-to-Point Topology

```
DIAGRAM:
  
  Site A                               Site B
  [Router A] ─── dedicated link ──── [Router B]
  
  Direct connection between exactly two nodes
  Can be: fiber, leased line, microwave, satellite, VPN tunnel
  
  WAN EXAMPLES:
  Headquarters ─── MPLS Circuit ─── Branch Office
  Headquarters ─── IPsec VPN  ─── Remote Worker
  Data Center A ─── Dark Fiber ─── Data Center B
```

```
BENEFITS:
  ✓ Simple: only two endpoints, clear responsibility
  ✓ Full bandwidth dedicated to that connection
  ✓ Low latency (no sharing)
  ✓ Easy to secure (only two points to protect)
  ✓ Predictable performance

LIMITATIONS:
  ✗ Doesn't scale (need one link per pair of sites)
  ✗ Expensive for many sites (becomes hub-and-spoke or mesh)
  ✗ No redundancy in single P2P link

WHEN USED TODAY:
  Everywhere for WAN connections:
  Leased lines, MPLS circuits, fiber interconnects
  VPN tunnels between sites (logically P2P over internet)
  Storage replication links between data centers
  Carrier interconnects (peering links between ISPs)
```

---

# PART 2 — LOGICAL TOPOLOGIES

---

## 9. Physical vs Logical Topology

```
THE CRITICAL DISTINCTION:

PHYSICAL TOPOLOGY: What you can see and touch
  Where are the cables?
  Where are the switches, routers, APs?
  What connectors are used?
  
LOGICAL TOPOLOGY: How data actually flows
  Which devices can communicate with which?
  How does a packet get from A to B?
  What protocol governs access to the shared medium?

CLASSIC EXAMPLE — WHY THEY DIFFER:

Physical: All PCs connect to a central HUB (star topology)
          [Hub receives frame → broadcasts out ALL ports]
Logical:  ALL PCs on same collision domain (BUS behavior!)
          Any frame reaches every device — hub is invisible in logic

Physical: All PCs connect to a central SWITCH (star topology)  
Logical:  Each port is its own collision domain (point-to-point!)
          Switch forwards only to correct destination — logical star

Physical cables don't change — the device in the middle (hub vs switch)
changes the logical behavior entirely.

ANOTHER EXAMPLE — VLAN:

Physical: PC1, PC2, PC3, PC4 all on same switch (star)
Logical:  PC1 and PC2 on VLAN 10 (isolated network A)
          PC3 and PC4 on VLAN 20 (isolated network B)
Physical star → two logical networks that can't communicate
```

---

## 10. Logical Bus

```
BEHAVIOR: All devices share a single communication channel
          One speaks, all hear
          Devices must arbitrate access to prevent collisions

PROTOCOLS THAT CREATE LOGICAL BUS:
  CSMA/CD (Ethernet with hubs): devices detect collisions, back off
  CSMA/CA (Wi-Fi): devices sense medium, avoid collisions before transmitting
  
WHY WI-FI IS A LOGICAL BUS:
  All Wi-Fi clients within range share the same radio frequency
  When one client transmits, all others must wait
  The AP manages this with CSMA/CA
  This is why many Wi-Fi clients = degraded performance for each
  
  Client A transmitting → all other clients detect busy medium → wait
  Same as original Ethernet over coax — just wireless instead of wire

IMPLICATION FOR PERFORMANCE:
  More devices = more contention = more wait time per device
  High-density Wi-Fi environments need careful cell planning
  (Smaller cells, more APs, correct channel assignment, band steering)
```

---

## 11. Logical Ring

```
TOKEN RING (IEEE 802.5):
  Physical star (all connect to hub/MAU)
  Logical ring (token passes from device to device in sequence)
  
  The MAU (Media Attachment Unit) wires the ports in ring sequence:
  Port 1 → Port 2 → Port 3 → Port 4 → back to Port 1
  Looks like star, behaves like ring
  
  Advantage: deterministic — each device gets a turn
  Maximum latency is bounded (number of devices × max hold time)

SONET LOGICAL RING:
  Two counter-rotating fiber rings
  Primary ring: traffic flows clockwise
  Protection ring: traffic flows counter-clockwise, normally idle
  
  ON FAILURE:
  If fiber cut between Node A and Node B:
  All traffic for that section automatically rerouted via protection ring
  Self-healing in <50ms (faster than most TCP sessions notice!)
  
  [A]═════[B]═════[C]
   ║                ║
   ╚═══════[D]══════╝
   
  Cut between B-C: A→B→(normal) then A→D→C (protection ring)
  
  SONET protection switching: <50ms restoration
  Still backbone of many carrier networks globally
```

---

## 12. Logical Star (Switched Ethernet)

```
HOW MODERN ETHERNET ACTUALLY WORKS:

Each switch port = its own collision domain
Full-duplex links: transmit AND receive simultaneously
Switches make point-to-point forwarding decisions (learn MAC from frames)

SWITCH LEARNING PROCESS:
  1. Frame arrives on Port 3 from MAC aa:bb:cc:11:22:33
  2. Switch records: aa:bb:cc:11:22:33 → Port 3 (in CAM table)
  3. Destination MAC dd:ee:ff:44:55:66 — check CAM table
  4. Found in CAM: forward only to Port 7
  5. If NOT found: flood to all ports (like a hub, temporarily)
  
  After learning period: all traffic unicast directly to correct port
  No device sees traffic not addressed to it (unlike hub!)

IMPLICATIONS FOR SECURITY:
  MAC flooding attack: flood switch with random MACs → fill CAM table
  When CAM table full: switch behaves like hub (floods all) → sniff everything
  Defense: Port Security (limit MACs per port), Dynamic ARP Inspection

LOGICAL STAR = BASIS FOR ALL MODERN NETWORKS:
  Every modern LAN is a logical star
  VLANs divide the star into isolated segments
  Layer 3 routing connects the segments
```

---

## 13. Logical Mesh (BGP/OSPF Routing)

```
THE INTERNET'S LOGICAL TOPOLOGY:

Physical: complex partial mesh of cables, fiber, wireless links
Logical: every AS (Autonomous System) can potentially reach every other
         via BGP path selection → effective logical mesh

BGP PEERING (logical mesh between ISPs):
  ISP A peers with ISP B, C, D (full mesh at major internet exchanges)
  ISP B peers with A, C, E, F
  Traffic finds optimal path via BGP attributes
  
  Internet Exchange Points (IXPs) are where ISPs physically connect:
  AMS-IX (Amsterdam), LINX (London), DE-CIX (Frankfurt), etc.
  At an IXP: hundreds of ISPs in one building, all on shared switch
  Logical: full mesh of BGP sessions between all participants

OSPF LOGICAL MESH (within one organization):
  All OSPF routers exchange link-state information
  Every router builds complete map of the network (LSDB)
  SPF algorithm finds shortest path to every destination
  Effectively creates a logical mesh of routing knowledge

MPLS VIRTUAL MESH (WAN):
  Customer sites connected point-to-point to provider edge
  Provider MPLS core creates any-to-any logical connectivity
  Site A can reach Site B via label-switched paths through provider core
  Physical: hub-and-spoke (sites to provider PEs)
  Logical: full mesh (any site can reach any other site)
```

---

# PART 3 — MODERN NETWORK ARCHITECTURES

---

## 14. Three-Tier Architecture (Core / Distribution / Access)

```
FULL THREE-TIER ENTERPRISE DIAGRAM:

┌──────────────────────────────────────────────────────────────────┐
│                        CORE LAYER                                │
│   [Core-SW-1] ═══════════════════════ [Core-SW-2]               │
│   (High speed, L3, no STP, redundant pair)                       │
└───────────────────┬────────────────────────┬─────────────────────┘
                    │                        │
        ╔═══════════╧══════╗      ╔═══════════╧══════╗
        ║  DISTRIBUTION    ║      ║  DISTRIBUTION    ║
        ║  Building A      ║      ║  Building B      ║
        ║  [Dist-A-1]      ║      ║  [Dist-B-1]      ║
        ║  [Dist-A-2]      ║      ║  [Dist-B-2]      ║
        ║  (Policy, ACL,   ║      ║  (Same)          ║
        ║   routing,VLAN)  ║      ║                  ║
        ╚════╤═══════╤═════╝      ╚════╤═══════╤═════╝
             │       │                │       │
    ┌────────┘       └────────┐  ┌────┘       └────┐
    │  ACCESS LAYER           │  │  ACCESS LAYER   │
    │  [Acc-A-1][Acc-A-2]     │  │  [Acc-B-1]      │
    │  Floor 1    Floor 2     │  │  Floor 1        │
    │  48 ports   48 ports    │  │  48 ports       │
    └──┬──────────┬───────────┘  └──────┬──────────┘
       │          │                     │
  PCs, Phones, APs, Printers       PCs, Phones, APs

CORE LAYER — The backbone:
  Fastest switching, minimal features
  L3 routing only (no STP loops — use L3 everywhere)
  Redundant pair (HSRP/VRRP for gateway redundancy)
  No access control here — just speed
  Typical: 10GbE, 40GbE, 100GbE uplinks
  Hardware: Cisco Catalyst 9500, Nexus 9300, Juniper EX9200

DISTRIBUTION LAYER — The policy tier:
  Inter-VLAN routing
  QoS policy application
  ACLs and security policies
  Summarize routes before advertising to core
  Redundant uplinks to both core switches
  Typical: 10GbE uplinks, 1/10GbE downlinks
  Hardware: Cisco Catalyst 9300/9400, Juniper EX4600

ACCESS LAYER — Where devices connect:
  Endpoint connectivity (PCs, phones, APs, printers)
  PoE for phones and APs (802.3af/at/bt)
  VLAN assignment per port
  Port security, 802.1X authentication
  STP portfast/BPDU guard on user ports
  Typical: 1GbE ports, 10GbE uplinks
  Hardware: Cisco Catalyst 9200, Juniper EX3400
```

```
BENEFITS:
  ✓ Clear separation of functions (speed, policy, access)
  ✓ Scalable: add access switches without redesigning core
  ✓ Fault isolation: access switch failure affects only its floor
  ✓ Consistent policy application at distribution
  ✓ Predictable traffic paths (up to dist, across at core, down)

LIMITATIONS:
  ✗ Three tiers = potential three hops minimum per any communication
  ✗ Higher cost (more devices)
  ✗ More complex to manage
  ✗ Three-tier overkill for small/medium organizations

WHEN TO USE THREE-TIER:
  Large campus: 1000+ users, multiple buildings
  When you need tight policy control at distribution
  When access and core requirements are very different
```

---

## 15. Two-Tier (Collapsed Core) Architecture

```
DIAGRAM:

        [Core/Dist-1] ══════ [Core/Dist-2]
        (Core AND Distribution merged)
              ║                    ║
    ┌─────────╫──────┐   ┌────────╫──────────┐
  [Acc-1]  [Acc-2]  [Acc-3]  [Acc-4]  [Acc-5]
  Floor 1  Floor 2  Floor 1  Floor 2  Floor 3
  
  Core and Distribution merged into two switches
  Access layer still separate

WHEN TO USE TWO-TIER:
  Small/medium organizations (< 500 users)
  Single building
  When three-tier would over-engineer the solution
  
BENEFITS:
  ✓ Lower cost (one less tier of hardware)
  ✓ Simpler configuration and management
  ✓ Fewer hops (better latency)
  ✓ Easier to understand and troubleshoot
  
LIMITATIONS:
  ✗ Less separation of policy and core functions
  ✗ Core/dist switches carry more configuration burden
  ✗ Harder to scale beyond initial design
```

---

## 16. Spine-Leaf Architecture (Data Center)

### Layman's Terms
Traditional three-tier doesn't work well in data centers because **east-west traffic** (server to server within the DC) dominates over north-south (user to server). Spine-leaf is specifically designed for this: every leaf (server-facing switch) connects to every spine (aggregation), giving any server equal-distance access to any other server.

```
SPINE-LEAF DIAGRAM:

┌─────────────────────────────────────────────────────────────┐
│                     SPINE LAYER                             │
│    [Spine-1]       [Spine-2]       [Spine-3]               │
│        ║               ║               ║                    │
│   Every Spine connects to Every Leaf (full mesh)            │
└────╥───────╥───────────╥───────────────╥─────────────────────┘
     ║       ╚═══╗       ║       ╗═══════╝
  [Leaf-1]  [Leaf-2]  [Leaf-3]  [Leaf-4]
  │││││││   │││││││   │││││││   │││││││
  Servers   Servers   Servers   Servers
  
RULES:
  Leaf ↔ Spine connections: ALL (every leaf to every spine)
  Leaf ↔ Leaf connections: NONE (leaves never connect directly)
  Spine ↔ Spine connections: NONE (spines never connect directly)
  
  Any server → any other server: exactly 2 hops (leaf → spine → leaf)
  PREDICTABLE, EQUAL LATENCY regardless of location!

UNDERLAY vs OVERLAY:
  Underlay: physical IP routing between spine and leaf (OSPF or BGP)
  Overlay: VXLAN tunnels carrying tenant traffic over the underlay
  
  VXLAN allows: same L2 segment stretched across multiple leaves
  Server on Leaf-1 and Server on Leaf-4 in same "VLAN" → VXLAN
```

```
BENEFITS:
  ✓ Equal latency between any two servers (2 hops always)
  ✓ Highly scalable: add leaves for more servers, add spines for more bandwidth
  ✓ No STP (pure L3, ECMP for load balancing)
  ✓ Massive aggregate bandwidth (every spine-leaf link carries traffic)
  ✓ Fault tolerant: losing one spine reduces bandwidth, not connectivity
  ✓ Multi-path (ECMP across all spines simultaneously)

LIMITATIONS:
  ✗ Complex configuration (BGP EVPN, VXLAN)
  ✗ Expensive: many switches, many cables
  ✗ Requires expertise to design and operate
  ✗ Fixed scale formula: max servers = leaves × ports per leaf

WHEN USED TODAY:
  Every modern hyperscale data center (Google, AWS, Meta, Azure)
  Cloud provider networks
  Enterprise data centers replacing three-tier
  Any environment with significant east-west traffic
```

---

## 17. Software-Defined Networking (SDN)

```
TRADITIONAL NETWORKING:
  Control plane and data plane ON EACH device
  Router/switch decides WHERE traffic goes (control) AND moves it (data)
  Configuration: log into each device, configure locally
  
SDN ARCHITECTURE:
  Control plane SEPARATED from data plane
  Centralized SDN Controller knows entire network topology
  Devices (switches, routers) become simple forwarding elements
  
  ┌─────────────────────────────────────┐
  │          SDN Controller             │
  │  (OpenFlow, NETCONF, RESTCONF)      │
  │  - Network-wide view               │
  │  - Calculates forwarding tables     │
  │  - Pushes rules to devices          │
  └────────────┬────────────────────────┘
               │ Control Channel
    ┌──────────┼──────────────┐
    │          │              │
  [SW-1]     [SW-2]        [SW-3]
  Simple      Simple        Simple
  Forwarding  Forwarding    Forwarding
  (OpenFlow)  (OpenFlow)    (OpenFlow)
  
NORTHBOUND API: Applications talk to controller
  (orchestration platforms, security apps, analytics)
SOUTHBOUND API: Controller talks to devices
  (OpenFlow, NETCONF/YANG, gRPC)
EAST-WEST: Controllers talk to each other
  (clustering, federation)

MODERN SDN EXAMPLES:
  Cisco ACI (Application Centric Infrastructure): data center SDN
  VMware NSX: network virtualization for VM environments
  OpenDaylight: open-source SDN controller
  SD-WAN: SDN principles applied to WAN (Cisco Viptela, VMware SD-WAN)
```

```
BENEFITS:
  ✓ Centralized network management (single pane of glass)
  ✓ Programmable: automate changes via API
  ✓ Network-wide optimization (controller sees everything)
  ✓ Rapid service deployment (no per-device CLI changes)
  ✓ Consistent policy application across all devices
  ✓ Enables network automation and DevOps/NetDevOps practices
  ✓ Better visibility and analytics

LIMITATIONS:
  ✗ Controller = new single point of failure (must be HA)
  ✗ Vendor lock-in (Cisco ACI, VMware NSX proprietary)
  ✗ Complexity in initial design and migration
  ✗ Requires operator retraining (new mental model)
  ✗ Northbound API security critical (compromise controller = own network)
```

---

## 18. Campus Network Design

```
COMPLETE CAMPUS NETWORK DESIGN:

┌─────────────────────────────────────────────────────────────────┐
│                      INTERNET / WAN                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────────┐
│                       INTERNET EDGE                             │
│  [Firewall-1 (Active)] ═══════════ [Firewall-2 (Standby)]      │
│  [IPS/IDS]  [Load Balancer]  [Email Gateway]                    │
│  [DMZ: Web Servers, VPN Concentrator, DNS]                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────────┐
│                       CORE LAYER                                │
│  [Core-1] (10/40GbE) ═════════════════ [Core-2] (10/40GbE)    │
└──────────┬───────────────────────────────────────┬──────────────┘
           │                                       │
┌──────────┴──────────┐               ┌────────────┴────────────┐
│  DISTRIBUTION A     │               │  DISTRIBUTION B         │
│  Building A         │               │  Building B             │
│  [Dist-A-1][Dist-A-2]              │  [Dist-B-1][Dist-B-2]   │
│  VLANs: 10,20,30   │               │  VLANs: 40,50,60        │
└──┬─────────────┬────┘               └───┬──────────────┬───────┘
   │             │                        │              │
[Acc-A-1]  [Acc-A-2]                [Acc-B-1]      [Acc-B-2]
Floor 1    Floor 2                  Floor 1        Floor 2

VLAN DESIGN FOR CAMPUS:
  VLAN 10: Management (network devices only, restricted access)
  VLAN 20: Corporate users (domain-joined PCs)
  VLAN 30: Voice (IP phones, QoS priority)
  VLAN 40: Wireless corporate (authenticated users)
  VLAN 50: Wireless guest (internet only, isolated)
  VLAN 60: Servers (application servers, file servers)
  VLAN 70: DMZ (web servers, mail relay)
  VLAN 80: Security cameras (isolated, outbound only)
  VLAN 90: IoT/printers (restricted, quarantine capable)
  VLAN 100: Quarantine (for NAC failures, restricted)

RECOMMENDED ADDRESSING:
  10.0.0.0/8 total campus space
  10.Building.VLAN.0/24 per subnet
  Example:
    10.1.10.0/24 = Building 1, VLAN 10 (management)
    10.1.20.0/24 = Building 1, VLAN 20 (users)
    10.2.10.0/24 = Building 2, VLAN 10 (management)
  Consistent, hierarchical, summarizable
```

---

## 19. Data Center Network Design

```
MODERN DATA CENTER — THREE-ZONE MODEL:

┌─────────────────────────────────────────────────────────────┐
│                    INTERNET EDGE ZONE                       │
│  [Border Routers] ← BGP peering with ISPs                  │
│  [DDoS Scrubbing] [Load Balancers] [WAF] [CDN Integration] │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                       DMZ ZONE                              │
│  Public-facing: Web Servers, API Gateways, Mail Relays      │
│  [Dedicated firewalls + IPS between Internet Edge and DMZ]  │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────┴─────────────────────────────────┐
│                    INTERNAL DC ZONE                         │
│  [Dedicated firewalls + IPS between DMZ and Internal]       │
│                                                             │
│  SPINE-LEAF FABRIC:                                         │
│  [Spine-1][Spine-2][Spine-3][Spine-4]                      │
│       Connected to every leaf below                         │
│  [L1][L2][L3][L4][L5][L6][L7][L8]                         │
│  │   │   │   │   │   │   │   │                             │
│  App DB  DB  Cache MQ  App App Storage                      │
│  Servers Servers Servers Servers Servers                    │
│                                                             │
│  VXLAN overlays create isolated tenant networks:            │
│  Tenant A: VNI 10001 (Web+App+DB, isolated L2 domain)     │
│  Tenant B: VNI 10002 (separate isolated environment)       │
│  Management: VNI 10999 (OOB management)                    │
└─────────────────────────────────────────────────────────────┘

EAST-WEST FIREWALL (micro-segmentation):
  Traffic between tiers (web→app→db) controlled by:
  Physical firewalls at tier boundaries (legacy approach)
  OR distributed firewall in hypervisor (VMware NSX DFW)
  OR Kubernetes NetworkPolicy (cloud-native apps)
```

---

## 20. WAN Topologies

```
1. HUB-AND-SPOKE (Star WAN):

   Branch A ─── MPLS ───┐
   Branch B ─── MPLS ───┤
   Branch C ─── MPLS ───┼─── HQ
   Branch D ─── MPLS ───┤
   Branch E ─── MPLS ───┘
   
   All branch-to-branch traffic goes THROUGH HQ
   Simple, cheap, central control
   Bottleneck at HQ, HQ failure = all branches isolated
   Used: small organizations with most resources at HQ

2. PARTIAL MESH WAN:

   HQ ════ Branch-A ════ Branch-B
   ║                        ║
   ╚════ Branch-C ═══════════╝
   
   Critical sites directly connected
   Reduces HQ bottleneck for heavy inter-branch traffic
   Higher cost, more circuits

3. FULL MESH WAN:
   Every site to every other site
   Maximum redundancy, maximum cost
   Only practical for very few sites (2-5)

4. SD-WAN (MODERN):
   Physical: Any internet connection (broadband, LTE, MPLS)
   Logical: Full mesh of encrypted tunnels between ALL sites
   
   Branch A ─── Internet/LTE ───┐
   Branch B ─── MPLS+Internet───┤
   Branch C ─── Internet ───────┼─── SD-WAN Orchestrator
   Branch D ─── 4G LTE ─────────┤    (cloud-based)
   HQ ────── MPLS+Internet ─────┘
   
   Advantages:
   - Any site can reach any site (logical full mesh)
   - Uses cheapest available circuit
   - Automatic failover between circuits
   - Application-aware routing (Office 365 breaks out locally, critical apps via MPLS)
   - Zero-touch provisioning (new sites self-configure)
   
   Technologies: Cisco Viptela/Meraki, VMware SD-WAN, Fortinet SD-WAN, Aryaka
```

---

# PART 4 — NETWORK PARTITIONING

---

## 21. Why Segment a Network

```
THE PROBLEM WITH A FLAT NETWORK (no segmentation):

  [PC1][PC2][PC3]...[PC500][Server1][Server2][DB1][PRINTER][CAMERA]
  ─────────────────────────────────────────────────────────────────
                      ALL ON THE SAME FLAT NETWORK
  
  PROBLEMS:
  1. Broadcast domain = ALL 500 devices
     One device ARP storms → 500 devices process the traffic
     
  2. Security: compromised PC1 can directly reach:
     - Every other PC (lateral movement)
     - Every server (including domain controllers!)
     - Every database
     - Even cameras and printers
     
  3. Compliance: PCI-DSS requires isolation of payment card systems
     HIPAA requires isolation of health records
     Flat network makes compliance impossible
     
  4. Performance: all traffic in one collision domain
     No QoS enforcement possible (voice vs bulk data)

SEGMENTATION GOALS:
  Contain broadcast storms (limit blast radius)
  Restrict lateral movement (attacker can't reach everything)
  Enforce least-privilege networking (only needed traffic flows)
  Enable compliance (isolated environments for regulated data)
  Improve performance (smaller domains, better QoS control)
  Simplify troubleshooting (isolate faults to segments)
```

---

## 22. VLANs — Layer 2 Segmentation

```
VLAN CONCEPT:
  Logically divide one physical switch into multiple virtual switches
  Each VLAN = separate broadcast domain
  Devices in different VLANs cannot communicate without routing
  
  Physical: all connected to same switch
  Logical: VLAN 10 devices can't see VLAN 20 devices

VLAN CONFIGURATION (Cisco IOS):

! Create VLANs:
vlan 10
 name CORPORATE_USERS
vlan 20
 name VOICE
vlan 30
 name SERVERS
vlan 40
 name GUEST_WIFI
vlan 99
 name MANAGEMENT

! Access port (one VLAN, for end devices):
interface GigabitEthernet0/1
 description PC-on-Floor1
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate          ! Disable DTP (security)
 spanning-tree portfast          ! Skip STP listening/learning
 spanning-tree bpduguard enable  ! Drop BPDUs (user port)
 no shutdown

! Trunk port (multiple VLANs, for switch-to-switch or switch-to-router):
interface GigabitEthernet0/24
 description UPLINK-TO-DIST-SW
 switchport mode trunk
 switchport trunk encapsulation dot1q
 switchport trunk allowed vlan 10,20,30,40,99  ! Explicit allow list!
 ! NEVER: switchport trunk allowed vlan all     ! Too permissive!
 switchport trunk native vlan 999  ! Non-routable native VLAN
 no shutdown

! Voice VLAN (data + voice on same port, different VLANs):
interface GigabitEthernet0/5
 description IP-Phone-with-PC-behind
 switchport mode access
 switchport access vlan 10    ! PC traffic
 switchport voice vlan 20     ! Phone traffic (auto 802.1p tagged)
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

VLAN HOPPING ATTACK PREVENTION:
  Attack: Send double-tagged 802.1Q frames to escape native VLAN
  
  Defense 1: Set native VLAN to unused ID (e.g., VLAN 999)
  switchport trunk native vlan 999
  
  Defense 2: Never use VLAN 1 as native (it's the default, attackers know it)
  
  Defense 3: Disable trunk on all access ports
  switchport mode access  (not dynamic auto/desirable)
  switchport nonegotiate  (disable DTP entirely)
  
  Defense 4: Explicit VLAN allow lists on trunks (never "all")
```

---

## 23. Subnetting — Layer 3 Segmentation

```
SUBNET DESIGN PRINCIPLES:

1. SIZE SUBNETS TO MATCH ACTUAL NEED (+ growth room):
   /24 (254 hosts) = standard floor/department subnet
   /25 (126 hosts) = smaller department
   /26 (62 hosts)  = small team or server segment
   /28 (14 hosts)  = point-to-point WAN links, small groups
   /30 (2 hosts)   = traditional P2P link
   /31 (2 hosts)   = modern P2P link (RFC 3021, saves addresses)
   
   Don't use /24 everywhere "to be safe" — wastes addresses,
   harder to track, harder to summarize
   
2. HIERARCHICAL ADDRESSING (enables summarization):
   10.1.0.0/16    = Building 1 entire address space
   10.1.10.0/24   = Building 1, VLAN 10 (users)
   10.1.20.0/24   = Building 1, VLAN 20 (voice)
   10.1.30.0/24   = Building 1, VLAN 30 (servers)
   
   10.2.0.0/16    = Building 2 entire address space
   10.2.10.0/24   = Building 2, VLAN 10 (users)
   
   ADVERTISEMENT: Building 1 router advertises 10.1.0.0/16 to core
   Core doesn't need to know about individual /24s → cleaner routing table

3. RESERVE RANGES FOR SPECIFIC PURPOSES:
   10.0.0.0/8     = Internal (RFC 1918)
   172.16.0.0/12  = Internal (RFC 1918)  
   192.168.0.0/16 = Internal (RFC 1918)
   
   Assign different RFC 1918 ranges to different functions:
   10.x.x.x      = Campus LAN
   172.16.x.x    = Data Center
   192.168.x.x   = Lab/dev environments
   Separation makes routing policies and ACLs cleaner

SUBNET CHEAT SHEET:
   CIDR  Mask              Hosts  
   /24   255.255.255.0     254    ← Standard LAN segment
   /25   255.255.255.128   126
   /26   255.255.255.192   62
   /27   255.255.255.224   30
   /28   255.255.255.240   14     ← Small server group
   /29   255.255.255.248   6
   /30   255.255.255.252   2      ← P2P WAN link
   /31   255.255.255.254   2      ← P2P (no broadcast needed)
   /32   255.255.255.255   1      ← Host route, loopback
```

---

## 24. DMZ Design — Isolating Public-Facing Services

```
DMZ ARCHITECTURE PATTERNS:

PATTERN 1: THREE-LEGGED FIREWALL (single firewall, three interfaces):

                [Internet]
                     │
             ┌───────┴───────┐
             │    Firewall   │
             │ (3 interfaces)│
             └──┬────┬────┬──┘
                │    │    │
           [DMZ] [LAN]  [WAN-backup]
           Web   Internal
           Mail  Users
           
  Pros: Simple, single device to manage
  Cons: Firewall = single point of failure
        Compromise of firewall = breach all three zones
        Not suitable for high compliance requirements

PATTERN 2: DUAL-FIREWALL DMZ (recommended for enterprise):

  [Internet]
       │
  [Outer FW]  ← Controls internet→DMZ and internet→internal
       │
    [DMZ]     ← Web servers, mail relay, VPN concentrators, DNS
       │
  [Inner FW]  ← Controls DMZ→internal (MUCH stricter rules)
       │
  [Internal LAN] ← Protected internal network

  Outer FW rules: Allow HTTPS/443 in to DMZ web servers
                  Allow SMTP/25 in to DMZ mail relay
                  Deny everything else inbound
                  
  Inner FW rules: DMZ web server → App server port 8080 ONLY
                  DMZ → internal: almost nothing allowed
                  No direct internet → internal
                  
  BENEFIT: Attacker who compromises DMZ web server
           still faces inner firewall to reach internal network
           Two firewalls from different vendors = different vuln profile

PATTERN 3: PARALLEL DMZ (separate DMZ per security zone):

  [Internet]
       │
  [Border FW]
    ┌──┴──┐
    │     │
  [DMZ1] [DMZ2]  ← Separate DMZs for different risk levels
  Public  Partner  
  web     extranets
    └──┬──┘
       │
  [Internal FW]
       │
  [Internal LAN]

WHAT GOES IN THE DMZ:
  ✓ Web servers (reverse proxy preferred — not direct app servers)
  ✓ Mail relay/gateway (filter spam before it hits internal mail)
  ✓ VPN concentrator (terminate VPN here, then auth to internal)
  ✓ External DNS server (authoritative for public domains)
  ✓ FTP/SFTP servers for external file exchange
  ✓ API gateways facing internet
  
WHAT NEVER GOES IN THE DMZ:
  ✗ Domain controllers (never expose AD to DMZ)
  ✗ Application servers with direct database access from internet
  ✗ File servers with sensitive data
  ✗ Any server with access to internal-only resources without firewall control
```

---

## 25. Zero-Trust Micro-Segmentation

```
TRADITIONAL MODEL ("castle and moat"):
  Trust everything inside the network perimeter
  Don't trust anything outside
  Problem: Attacker who gets inside has free movement

ZERO-TRUST MODEL:
  "Never trust, always verify"
  No implicit trust based on network location
  Every connection authenticated and authorized regardless of source
  
MICRO-SEGMENTATION:
  Apply per-workload, per-application security policies
  Even if two servers are on the same subnet, they can't communicate
  unless explicitly permitted by policy
  
IMPLEMENTATION APPROACHES:

1. HOST-BASED FIREWALL (Windows Defender Firewall, iptables):
   Policy on every host: only accept connections from authorized sources
   Example: Database server only accepts port 5432 from app servers
   App servers only accept port 443 from load balancers
   
2. HYPERVISOR-BASED (VMware NSX Distributed Firewall):
   Firewall rules enforced at each VM's virtual NIC
   No traffic bypass possible (enforced in kernel)
   Manage from central NSX Manager
   
3. KUBERNETES NETWORK POLICY:
   Ingress/egress rules per pod/namespace
   Default: deny all (explicit allow required)
   Example:
   
   apiVersion: networking.k8s.io/v1
   kind: NetworkPolicy
   metadata:
     name: allow-only-frontend
     namespace: production
   spec:
     podSelector:
       matchLabels:
         app: backend-api
     ingress:
     - from:
       - podSelector:
           matchLabels:
             app: frontend  # Only frontend can reach backend
       ports:
       - protocol: TCP
         port: 8080

4. IDENTITY-BASED SEGMENTATION:
   Policies based on user/workload identity, not IP address
   SPIFFE/SPIRE for workload identity
   Service mesh (Istio) for mTLS between services
   
BENEFITS OF ZERO-TRUST:
  ✓ Lateral movement is extremely difficult (every hop requires auth)
  ✓ Breach blast radius is minimal (attacker can't pivot freely)
  ✓ Compliance-friendly (data access is explicit and logged)
  ✓ Works in hybrid/multi-cloud (location-independent policy)
  
LIMITATIONS:
  ✗ Complex to implement correctly
  ✗ Application discovery required (what talks to what?)
  ✗ Legacy applications may not support mTLS or modern auth
  ✗ Operational overhead (every change requires policy update)
```

---

## 26. Air-Gapped Networks

```
DEFINITION:
  A network with NO connection to other networks or the internet
  Physically isolated — no cables, no Wi-Fi, no Bluetooth links
  to any external network
  
USE CASES:
  Nuclear facility control systems
  Military classified systems (SIPRNET, JWICS)
  Industrial control systems / SCADA for critical infrastructure
  Financial trading systems (some cases)
  High-security research environments
  
PHYSICAL ISOLATION REQUIREMENTS:
  No wired connections to other networks
  No wireless (Wi-Fi, Bluetooth, cellular disabled or removed)
  Faraday cage consideration for very high security
  Strict physical access controls to the room/building
  
HOW DATA MOVES IN/OUT (the air gap challenge):
  USB drives (controlled, scanned before use — biggest attack vector!)
  Optical media (CD/DVD — read-only media preferred)
  Data diodes (one-way hardware enforcement — only OUT, never IN)
  Secure file transfer stations (isolated, scanned)
  
AIR GAP ATTACKS (theoretical and demonstrated):
  Stuxnet: spread via USB drive across air gap to Iranian nuclear facility
  TEMPEST: electromagnetic emanation from cables/monitors can leak data
  Acoustic: ultrasonic exfiltration via PC speakers/fans (demonstrated in research)
  
IMPLEMENTATION BEST PRACTICES:
  Strict USB port control (hardware disable or device whitelist)
  All media scanned at isolated scanning station before use
  Physical security: badge access logs, cameras, escorts required
  No consumer devices (phones, tablets) inside the air-gapped zone
  Regular security audits of physical connections
  Monitor for unauthorized wireless signals (RF scanning)
```

---

# PART 5 — ROUTING STRATEGIES

---

## 27. Inter-VLAN Routing

```
PROBLEM: Devices in different VLANs cannot communicate without routing
         even if on the same physical switch

SOLUTION 1: ROUTER ON A STICK (legacy, small networks)

  Switch ─── Trunk link ─── Router
  
  One physical router link, multiple logical sub-interfaces:
  
  Router config:
  interface GigabitEthernet0/0.10
   encapsulation dot1Q 10
   ip address 10.1.10.1 255.255.255.0
   description GATEWAY-FOR-VLAN10
  
  interface GigabitEthernet0/0.20
   encapsulation dot1Q 20
   ip address 10.1.20.1 255.255.255.0
   description GATEWAY-FOR-VLAN20
  
  interface GigabitEthernet0/0.30
   encapsulation dot1Q 30
   ip address 10.1.30.1 255.255.255.0
   description GATEWAY-FOR-VLAN30
  
  LIMITATION: All inter-VLAN traffic passes through ONE physical link
              Creates bottleneck as VLANs grow
              Only suitable for very small networks (<50 users total)

SOLUTION 2: LAYER 3 SWITCH SVI (recommended for enterprise)

  Switch performs routing internally — no external router needed
  Each VLAN gets a Switched Virtual Interface (SVI) = virtual Layer 3 port
  
  L3 Switch config:
  ip routing  ! Enable L3 switching
  
  vlan 10
  vlan 20
  vlan 30
  
  interface Vlan10
   ip address 10.1.10.1 255.255.255.0
   description GATEWAY-VLAN10-USERS
   no shutdown
  
  interface Vlan20
   ip address 10.1.20.1 255.255.255.0
   description GATEWAY-VLAN20-VOICE
   no shutdown
  
  interface Vlan30
   ip address 10.1.30.1 255.255.255.0
   description GATEWAY-VLAN30-SERVERS
   no shutdown
  
  ! ACL example: prevent users from directly reaching DB servers
  ip access-list extended VLAN10-TO-VLAN30
   permit tcp 10.1.10.0 0.0.0.255 10.1.30.0 0.0.0.255 eq 443
   permit tcp 10.1.10.0 0.0.0.255 10.1.30.0 0.0.0.255 eq 80
   deny   ip  10.1.10.0 0.0.0.255 10.1.30.0 0.0.0.255
   permit ip any any
  
  interface Vlan10
   ip access-group VLAN10-TO-VLAN30 out
  
  BENEFIT: Wire-speed routing within the switch hardware
           No external router bottleneck
           ACLs applied at line rate
```

---

## 28. Route Summarization and Hierarchy

```
WHY SUMMARIZATION MATTERS:

Without summarization (access to all routes visible everywhere):
  Core router routing table: 500 specific /24 routes
  Each route = memory entry + processing for each update
  Route flap in one access switch → update propagates everywhere
  
With summarization (aggregate at distribution, summary to core):
  Access switch: advertises /24 routes to distribution
  Distribution summarizes: advertises 10.1.0.0/16 to core
  Core routing table: 20 summary routes instead of 500 specifics
  Route flap in access switch → only distribution sees it
  Core stable and fast
  
SUMMARIZATION EXAMPLE:

Building 1 has these subnets:
  10.1.10.0/24
  10.1.20.0/24
  10.1.30.0/24
  10.1.40.0/24
  10.1.50.0/24
  
All fall within 10.1.0.0/16
Distribution router advertises: 10.1.0.0/16 summary to core
Core only sees ONE route instead of five!

OSPF SUMMARIZATION:
  router ospf 1
   area 1 range 10.1.0.0 255.255.0.0  ! Summarize at ABR
  
BGP SUMMARIZATION:
  router bgp 65001
   aggregate-address 10.1.0.0 255.255.0.0 summary-only
  ! summary-only suppresses more-specific advertisements

RULE OF THUMB:
  Summarize at each tier boundary
  Each distribution block should fit in one summary address block
  Plan your IP addressing scheme BEFORE you assign addresses
  (Retrofit is extremely painful)
```

---

## 29. Policy-Based Routing

```
WHAT IT IS:
  Normal routing: packets routed based ONLY on destination IP
  Policy-based routing: route based on SOURCE IP, port, protocol, or other criteria
  
USE CASES:
  1. Route high-priority traffic via expensive/fast MPLS link
     Route bulk traffic via cheap internet connection
     
  2. Route customer A's traffic via Provider X
     Route customer B's traffic via Provider Y
     
  3. Force certain users through inspection/proxy regardless of destination
  
CISCO IOS EXAMPLE:

  ! Route all traffic from VoIP VLAN via MPLS (better QoS)
  ! Route all other traffic via internet
  
  ip access-list extended VOIP-TRAFFIC
   permit udp 10.1.20.0 0.0.0.255 any  ! Voice VLAN source
  
  route-map PBR-VOICE permit 10
   match ip address VOIP-TRAFFIC
   set ip next-hop 10.0.0.1  ! MPLS gateway
  
  route-map PBR-VOICE permit 20
   ! No match = all other traffic
   set ip next-hop 203.0.113.1  ! Internet gateway
  
  interface Vlan20
   ip policy route-map PBR-VOICE  ! Apply to Voice VLAN SVI
  
MODERN APPROACH: SD-WAN application-aware routing
  (Does this automatically based on application recognition)
```

---

# PART 6 — NETWORK ADMIN BEST PRACTICES

---

## 30. IP Addressing Design and Planning

```
STEP-BY-STEP IP ADDRESSING PLAN:

STEP 1: ALLOCATE TOP-LEVEL BLOCKS BY FUNCTION
  10.0.0.0/8      = Entire organization (private space)
  10.0.0.0/11     = Campus A (10.0.0.0 – 10.31.255.255)
  10.32.0.0/11    = Campus B (10.32.0.0 – 10.63.255.255)
  10.64.0.0/11    = Data Center (10.64.0.0 – 10.95.255.255)
  10.96.0.0/11    = Cloud/Azure/AWS (10.96.0.0 – 10.127.255.255)
  10.128.0.0/11   = WAN/P2P links (10.128.0.0 – 10.159.255.255)
  10.160.0.0/11   = Spare/future (10.160.0.0 – 10.191.255.255)
  10.192.0.0/10   = Lab/dev (10.192.0.0 – 10.255.255.255)

STEP 2: ALLOCATE PER SITE/BUILDING
  Campus A: 10.0.0.0/11
  Building 1: 10.0.0.0/16  (10.0.0.0 – 10.0.255.255)
  Building 2: 10.1.0.0/16  (10.1.0.0 – 10.1.255.255)
  Building 3: 10.2.0.0/16  (10.2.0.0 – 10.2.255.255)

STEP 3: ALLOCATE PER VLAN WITHIN EACH BUILDING
  Building 1:
  10.0.10.0/24  = VLAN 10 (Users)
  10.0.20.0/24  = VLAN 20 (Voice)
  10.0.30.0/24  = VLAN 30 (Servers)
  10.0.40.0/24  = VLAN 40 (Wireless)
  10.0.99.0/24  = VLAN 99 (Management)

STEP 4: STANDARD ADDRESSING WITHIN EACH SUBNET
  .1   = Default gateway (switch SVI or router)
  .2   = Secondary gateway (HSRP/VRRP standby, if applicable)
  .3   = Reserved
  .4   = Reserved
  .5-.9 = Network infrastructure (APs, switches management IPs)
  .10-199 = DHCP pool for end devices
  .200-250 = Static assignments (servers, printers, fixed devices)
  .251-254 = Reserved
  .255 = Broadcast (unusable)

DOCUMENT EVERYTHING IN IPAM:
  Use: Infoblox, NetBox (open source), phpIPAM, or even a spreadsheet
  Record: Subnet, VLAN, Location, Gateway, DHCP range, Purpose
  Assigned IPs: Device name, MAC, location, responsible team
  Keep it CURRENT — stale IPAM is worse than no IPAM
```

---

## 31. Network Documentation Standards

```
MINIMUM DOCUMENTATION SET:

1. NETWORK TOPOLOGY DIAGRAMS (logical AND physical):
   Logical: VLANs, IP subnets, routing protocol, firewall zones
   Physical: Building maps, cable paths, port assignments, rack diagrams
   Tools: draw.io, Lucidchart, Visio, NetBox (automatic topology)
   Update: every time you make a change

2. IP ADDRESS MANAGEMENT (IPAM):
   Every subnet, every assigned address
   Include: hostname, location, VLAN, owner, purpose
   
3. CABLE DOCUMENTATION:
   Source port → destination port for every cable
   Include: cable type, length, color code (if used)
   Critical for troubleshooting and moves/adds/changes
   
4. DEVICE INVENTORY:
   Hostname, model, serial number, firmware/OS version
   Location (building, floor, rack, U position)
   Support contract, end-of-life date
   Configuration backup location
   
5. CHANGE LOG:
   Date, who made change, what was changed, why, result
   Keep forever — invaluable for debugging "it worked last week"
   
6. RUNBOOKS AND PROCEDURES:
   How to add a new user port
   How to add a new VLAN
   How to failover the firewall
   How to perform firmware upgrades
   Emergency contacts and escalation paths

NAMING CONVENTIONS (be consistent):
  Switches:    SW-{BUILDING}-{FLOOR}-{NUMBER}   → SW-HQ-2-01
  Routers:     RTR-{SITE}-{FUNCTION}-{NUMBER}   → RTR-HQ-CORE-01
  Firewalls:   FW-{SITE}-{ZONE}-{NUMBER}        → FW-HQ-EDGE-01
  APs:         AP-{BUILDING}-{FLOOR}-{NUMBER}   → AP-HQ-2-14
  Servers:     {FUNCTION}-{SITE}-{NUMBER}       → WEBSVR-HQ-01
  Ports:       Description required on every port!
```

---

## 32. Change Management

```
WHY CHANGE MANAGEMENT MATTERS:
  Most network outages are caused by CHANGES (not hardware failure)
  Undocumented changes cause impossible-to-debug problems
  "Who changed what last Tuesday?" = critical question in every incident
  
CHANGE CONTROL PROCESS:

1. REQUEST:
   What is changing?
   Why is this change needed? (business justification)
   What systems/services are affected?
   What is the rollback plan if it fails?
   
2. REVIEW:
   Peer review by another engineer
   Impact assessment
   Testing plan (preferably in lab first)
   
3. APPROVAL:
   Change Advisory Board (CAB) for major changes
   Direct manager approval for standard changes
   Pre-approved templates for routine changes (adding ports, etc.)
   
4. SCHEDULING:
   Maintenance window (define times when changes are allowed)
   Communicate to affected teams before the window
   Emergency process for urgent issues (still document!)
   
5. IMPLEMENTATION:
   Follow the written procedure exactly
   Someone else watching (or on-call standby)
   Verify the change worked immediately after
   
6. DOCUMENTATION:
   Update network diagrams, IPAM, device documentation
   Record what was done, when, by whom
   
MAINTENANCE WINDOWS:
  Define a recurring window (e.g., Tuesday 10pm-2am)
  All changes go in that window unless emergency
  Users know to expect possible disruption during window
  
ROLLBACK PLAN = MANDATORY:
  Before ANY change: document how to undo it
  Configuration backup before changes: mandatory
  Test rollback procedure in lab if possible
  Set a "go/no-go" time: if not working by X, rollback immediately
```

---

## 33. Monitoring and Observability

```
WHAT TO MONITOR:

INFRASTRUCTURE AVAILABILITY:
  Ping (ICMP) monitoring of all devices → know immediately if device goes down
  SNMP polls for interface status → know when a link goes down
  Syslog collection → events and errors from all devices
  
PERFORMANCE METRICS:
  Interface utilization (%) → identify congestion before users complain
  CPU and memory on routers/switches → capacity planning
  Error counters (CRC errors, input/output drops) → cabling or config issues
  Latency between key points → detect routing issues
  
TOOLS:
  Nagios/Zabbix/LibreNMS: open-source monitoring (ICMP + SNMP)
  PRTG: commercial, easy to set up, Windows-based
  Prometheus + Grafana: modern metrics collection + visualization
  Elastic Stack: log aggregation and analysis
  SolarWinds: enterprise (expensive but comprehensive)
  
SNMP CONFIGURATION (Cisco IOS):
  ! Read-only SNMPv3 (preferred over v1/v2c which send community in cleartext)
  snmp-server group MONITORING v3 priv
  snmp-server user NETMON MONITORING v3 auth sha AUTH_PASS priv aes 128 PRIV_PASS
  snmp-server host 10.0.99.10 version 3 priv NETMON
  
  ! If SNMPv2c required (legacy tools):
  snmp-server community PUBLIC_NAME RO           ! Read-only, restrict by ACL
  snmp-server community PRIVATE_NAME RW ACL 99  ! Read-write MUST be ACL-restricted
  
SYSLOG CONFIGURATION:
  ! Send logs to centralized syslog server:
  logging host 10.0.99.11
  logging trap informational  ! Log level (emergency/alert/critical/error/warning/notice/info/debug)
  logging source-interface Loopback0  ! Consistent source IP for logs
  service timestamps log datetime msec  ! Add timestamps to all log messages
  
NETFLOW (traffic analysis):
  ! Understand what traffic is flowing where
  ip flow-export destination 10.0.99.12 9996
  ip flow-export version 9
  ip flow-cache timeout active 1
  interface GigabitEthernet0/0
   ip flow ingress
   ip flow egress
  ! Use Ntopng, ElastiFlow, or SolarWinds NTA to visualize
  
ALERTING RULES:
  Interface down → immediate alert
  Interface utilization > 80% for 5 min → warning
  Interface utilization > 95% for 1 min → critical
  Device unreachable → immediate alert (after 3 consecutive ping failures)
  Log error rate spike → alert
  New device on network → alert (detect rogue devices!)
```

---

## 34. Network Security Hardening

```
SWITCH HARDENING CHECKLIST:

MANAGEMENT PLANE:
  □ Change default hostname (no identifying vendor info)
  □ Set strong enable secret (not enable password — it's weaker):
    enable secret 9 $9$STRONG_HASH  (type 9 = most secure)
  □ Use SSH v2 ONLY (disable Telnet completely):
    line vty 0 15
     transport input ssh
     login local
    ip ssh version 2
    no service telnet
  □ Create local admin with strong password:
    username admin privilege 15 secret STRONG_PASSWORD
  □ Restrict management access to management VLAN only:
    ip access-class MGMT-ACL in (on VTY lines)
  □ Set session timeout:
    line vty 0 15
     exec-timeout 10 0  (10 minutes idle timeout)
  □ Disable unused services:
    no service finger
    no service tcp-small-servers
    no service udp-small-servers
    no ip http server  (use HTTPS only)
    ip http secure-server
    no cdp run  (on edge/access ports if not needed)
  □ NTP with authentication:
    ntp server 10.0.99.20 key 1
    ntp authenticate
    
DATA PLANE:
  □ Disable unused ports, put in unused VLAN:
    interface range Gi0/10-24
     shutdown
     switchport access vlan 999  (black hole VLAN with no routing)
  □ Enable BPDU Guard on ALL user ports:
    spanning-tree portfast bpduguard default
  □ Enable port security where needed:
    switchport port-security maximum 3  (limit MAC addresses)
    switchport port-security violation restrict  (log but don't shut)
  □ Enable DAI (Dynamic ARP Inspection) per VLAN:
    ip arp inspection vlan 10,20,30
    ! Mark trusted ports (uplinks to switches):
    interface Gi0/24
     ip arp inspection trust
  □ Enable DHCP snooping:
    ip dhcp snooping
    ip dhcp snooping vlan 10,20,30
    ! Trust only legitimate DHCP server port:
    interface Gi0/24
     ip dhcp snooping trust
  □ Enable IP Source Guard on untrusted ports:
    ip verify source  (on access ports, prevents IP spoofing)
  □ Explicit trunk VLAN allow lists (never allow all):
    switchport trunk allowed vlan 10,20,30,99
    
ROUTER HARDENING:
  □ BCP38: Block spoofed source IPs at edge (uRPF):
    interface GigabitEthernet0/0 (internet-facing)
     ip verify unicast source reachable-via rx  (strict uRPF)
  □ No IP directed broadcasts:
    no ip directed-broadcast (blocks smurf amplification)
  □ Enable logging for all ACL denies:
    ip access-list extended EDGE-IN
     deny ip any any log  (explicit deny with logging)
  □ Rate-limit ICMP to prevent ping floods:
    interface Gi0/0
     rate-limit input access-group 100 8000 1500 2000 conform-action continue exceed-action drop
```

---

## 35. Redundancy and High Availability

```
REDUNDANCY LEVELS:

ACTIVE/STANDBY (Cold Standby):
  Primary device active, standby idle
  On failure: manual or automatic failover
  Failover time: seconds to minutes
  Cost: low (standby does nothing)
  Used: simple environments, lower criticality

ACTIVE/ACTIVE:
  Both devices handling traffic simultaneously
  On failure: remaining device absorbs all traffic
  Failover time: sub-second (no state to transfer)
  Cost: higher (both fully loaded in steady state)
  Used: high-traffic environments, maximum performance

FIRST HOP REDUNDANCY (gateway redundancy):

  HSRP (Cisco proprietary):
  standby 10 ip 10.1.10.1  (virtual gateway IP)
  standby 10 priority 110   (higher = active)
  standby 10 preempt        (take back active if it comes back)
  standby 10 track 1 decrement 20  (reduce priority if uplink fails)
  
  VRRP (open standard, RFC 5798):
  vrrp 10 ip 10.1.10.1
  vrrp 10 priority 110
  vrrp 10 preempt
  
  Both provide: single virtual IP that users use as gateway
  If primary switch fails: standby takes over virtual IP in <1 second
  
LINK REDUNDANCY (Port-Channel / LAG):
  Bundle multiple physical links into one logical link
  Provides: redundancy + bandwidth aggregation
  
  LACP (standard, IEEE 802.3ad):
  interface Port-channel1
   description UPLINK-TO-CORE
   switchport mode trunk
  
  interface GigabitEthernet0/23
   channel-group 1 mode active  (LACP active)
  
  interface GigabitEthernet0/24
   channel-group 1 mode active
  
  Result: 2×1GbE = 2Gbps effective (if traffic hashes across both)
  If one link fails: other carries all traffic (no downtime!)

REDUNDANT POWER SUPPLY:
  All critical switches/routers: dual PSUs from different circuits
  Different circuits from different breaker panels
  Ideally different utility feeds into the building
  UPS on all network equipment
  Generator for extended outages

SPANNING TREE FOR REDUNDANCY:
  Always have two uplinks from access to distribution
  STP blocks one to prevent loops
  If active uplink fails: STP unblocks the other (30-50 second convergence)
  
  Use RSTP (not classic STP) for faster convergence:
  spanning-tree mode rapid-pvst
  
  Best: Use L3 at distribution (no STP blocking at all):
  Two uplinks, both routing (ECMP load sharing)
  Both active simultaneously
  One fails: OSPF/EIGRP reconverges in <1 second
  FAR better than STP convergence
```

---

## Summary: Network Design Decision Reference

```
╔══════════════════════════════════════════════════════════════════╗
║              TOPOLOGY SELECTION GUIDE                           ║
╠══════════════════════════════════════════════════════════════════╣
║  SMALL OFFICE (<50 users, 1 floor):                             ║
║  Physical: Star (1-2 switches)                                  ║
║  Logical: L3 switch with SVIs, basic VLANs                      ║
║  WAN: ISP router + firewall                                     ║
║                                                                  ║
║  MEDIUM CAMPUS (50-500 users, 1-2 buildings):                   ║
║  Physical: Hierarchical star (collapsed core / 2-tier)          ║
║  Logical: SVIs for inter-VLAN, OSPF or static routes           ║
║  WAN: Redundant ISPs + firewall HA pair                         ║
║                                                                  ║
║  LARGE ENTERPRISE (500+ users, multiple buildings):             ║
║  Physical: Hierarchical (3-tier core/dist/access)               ║
║  Logical: OSPF with hierarchy, HSRP/VRRP gateways              ║
║  WAN: MPLS + internet + SD-WAN overlay                         ║
║                                                                  ║
║  DATA CENTER (servers, virtualization):                         ║
║  Physical: Spine-leaf                                           ║
║  Logical: BGP EVPN + VXLAN overlay                             ║
║  East-West: Micro-segmentation                                  ║
║                                                                  ║
║  SEGMENTATION APPROACH BY RISK:                                 ║
║  Low risk (SOHO): Basic VLANs (user/IoT/guest)                 ║
║  Medium risk (SMB): VLANs + firewall between segments          ║
║  High risk (enterprise): DMZ + internal zones + micro-seg      ║
║  Critical (compliance): Zero-trust + air-gap where required    ║
║                                                                  ║
║  ALWAYS (regardless of size):                                   ║
║  □ Separate management VLAN (admin access isolated)            ║
║  □ Guest network completely isolated                            ║
║  □ IoT/printers on separate VLAN                               ║
║  □ SSH only (never Telnet)                                      ║
║  □ DHCP snooping + DAI on access switches                      ║
║  □ Redundant uplinks at every tier                              ║
║  □ Document everything                                          ║
║  □ Monitor everything                                           ║
╚══════════════════════════════════════════════════════════════════╝
```

---

*Cross-references in this series:*
- *Switching mechanisms (STP, VLANs, trunking): `Networking_Deep_Dive_TCPIP_and_Devices.md` Section 2*
- *Firewall configuration deep dive: `Networking_Deep_Dive_TCPIP_and_Devices.md` Section 5*
- *Cloud VPC networking: `Cloud_Networking_Sections_18_to_36.md`*
- *How attackers exploit flat networks: `Active_Directory_RedTeam_Field_Manual.md` (lateral movement)*
- *VLAN hopping attack: `Ports_Protocols_RedTeam_Field_Manual.md`*
- *Network monitoring (Zeek, Snort): `Networking_Deep_Dive_TCPIP_and_Devices.md` Section 8*

*Standards references: IEEE 802.1Q (VLANs), IEEE 802.3ad (LACP), RFC 1918 (private addressing),*
*RFC 5798 (VRRP), Cisco Three-Tier design guide, NIST SP 800-41 (firewall guidelines)*