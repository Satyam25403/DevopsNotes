# GCP VPC & Networking

GCP's Virtual Private Cloud (VPC) provides private, isolated networking for your GCP resources. Understanding VPC fundamentals is essential for DevOps — it controls how your services communicate securely.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **VPC Network** | A global, private network spanning all regions |
| **Subnet** | A regional IP address range within a VPC |
| **Firewall Rule** | Controls ingress/egress traffic using tags or service accounts |
| **Cloud Router** | Enables dynamic routing (BGP) for VPN and Interconnect |
| **Cloud NAT** | Allows VMs without public IPs to reach the internet |
| **Private Google Access** | Lets VMs with no public IP reach GCP APIs |
| **VPC Peering** | Private connectivity between two VPC networks |
| **Shared VPC** | Share one VPC network across multiple GCP projects |

### Key Difference from AWS
GCP VPCs are **global** — one VPC spans all regions. Subnets are regional. This means a single VPC can have subnets in `us-central1`, `europe-west1`, and `asia-east1`, all part of the same network. (AWS VPCs are regional.)

---

## Default vs Custom VPC

GCP creates a `default` VPC with subnets in every region automatically. For production, always create a **custom VPC** with explicit subnets and firewall rules.

```bash
# Create a custom VPC (no auto subnets)
gcloud compute networks create my-vpc \
  --subnet-mode=custom \
  --bgp-routing-mode=regional

# Create subnets
gcloud compute networks subnets create my-subnet-us \
  --network=my-vpc \
  --region=us-central1 \
  --range=10.0.0.0/20 \
  --enable-private-ip-google-access   # Allow VMs to reach GCP APIs

gcloud compute networks subnets create my-subnet-eu \
  --network=my-vpc \
  --region=europe-west1 \
  --range=10.1.0.0/20 \
  --enable-private-ip-google-access

# List VPCs and subnets
gcloud compute networks list
gcloud compute networks subnets list --network=my-vpc
```

---

## Firewall Rules

```bash
# Allow SSH (port 22) from your IP only
gcloud compute firewall-rules create allow-ssh \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=YOUR_IP/32 \
  --target-tags=allow-ssh

# Allow HTTP/HTTPS from anywhere
gcloud compute firewall-rules create allow-http-https \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp:80,tcp:443 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=web-server

# Allow internal VPC traffic on all ports
gcloud compute firewall-rules create allow-internal \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=all \
  --source-ranges=10.0.0.0/8

# Allow load balancer health checks
gcloud compute firewall-rules create allow-health-checks \
  --network=my-vpc \
  --direction=INGRESS \
  --priority=1000 \
  --action=ALLOW \
  --rules=tcp \
  --source-ranges=35.191.0.0/16,130.211.0.0/22 \
  --target-tags=web-server

# List firewall rules
gcloud compute firewall-rules list --filter="network=my-vpc"
```

> **Best practice**: Use **target tags** (not IP ranges) for intra-service communication. VMs communicate by applying tags, not by knowing each other's IPs.

---

## Cloud NAT (Outbound Internet for Private VMs)

VMs without public IPs can't reach the internet by default. Cloud NAT provides outbound internet access without exposing VMs to inbound connections.

```bash
# 1. Create a Cloud Router
gcloud compute routers create my-router \
  --network=my-vpc \
  --region=us-central1

# 2. Create a NAT gateway on the router
gcloud compute routers nats create my-nat \
  --router=my-router \
  --region=us-central1 \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips

# VMs in us-central1 subnets can now reach the internet via NAT
```

---

## Static / Reserved IP Addresses

```bash
# Reserve a global static IP (for Load Balancers)
gcloud compute addresses create my-lb-ip \
  --global \
  --ip-version=IPV4

# Reserve a regional static IP (for VMs, Cloud NAT)
gcloud compute addresses create my-vm-ip \
  --region=us-central1

# List reserved IPs
gcloud compute addresses list

# Assign static IP to a VM
gcloud compute instances create my-vm \
  --zone=us-central1-a \
  --address=my-vm-ip \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

---

## VPC Peering (Connect Two VPCs)

```bash
# Peer vpc-a with vpc-b (must be done from both sides)
gcloud compute networks peerings create vpc-a-to-vpc-b \
  --network=vpc-a \
  --peer-project=project-b \
  --peer-network=vpc-b \
  --auto-create-routes

# From project-b:
gcloud compute networks peerings create vpc-b-to-vpc-a \
  --network=vpc-b \
  --peer-project=project-a \
  --peer-network=vpc-a \
  --auto-create-routes
```

---

## Shared VPC (Multi-Project Networking)

Shared VPC lets multiple GCP projects share a single VPC network. The "host project" owns the network; "service projects" use it.

```bash
# In the host project: enable Shared VPC
gcloud compute shared-vpc enable my-host-project

# Associate a service project
gcloud compute shared-vpc associated-projects add my-service-project \
  --host-project=my-host-project

# Grant the service project's service accounts permission to use subnets
gcloud projects add-iam-policy-binding my-host-project \
  --member="serviceAccount:SERVICE_PROJECT_NUMBER@cloudservices.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

---

## Private Service Connect (Access GCP Services Privately)

Access GCP APIs and managed services (Cloud SQL, Pub/Sub, etc.) privately without traffic leaving your VPC:

```bash
# Create a Private Service Connect endpoint for Google APIs
gcloud compute addresses create my-psc-address \
  --global \
  --purpose=PRIVATE_SERVICE_CONNECT \
  --addresses=10.0.0.100 \
  --network=my-vpc

gcloud compute forwarding-rules create my-psc-endpoint \
  --global \
  --address=my-psc-address \
  --target-google-apis-bundle=all-apis \
  --network=my-vpc
```

---

## Cloud VPN (Site-to-Site)

Connect your on-premise network to GCP securely over the internet:

```bash
# Create a VPN gateway
gcloud compute vpn-gateways create my-vpn-gateway \
  --network=my-vpc \
  --region=us-central1

# Create a Cloud Router for dynamic routing
gcloud compute routers create my-vpn-router \
  --network=my-vpc \
  --region=us-central1 \
  --asn=65001

# Create VPN tunnels (HA VPN requires 2 tunnels)
gcloud compute vpn-tunnels create tunnel-1 \
  --peer-gcp-gateway=PEER_GATEWAY \
  --vpn-gateway=my-vpn-gateway \
  --interface=0 \
  --region=us-central1 \
  --ike-version=2 \
  --shared-secret=MY_SHARED_SECRET \
  --router=my-vpn-router
```

---

## Network Architecture Patterns

### Three-Tier Web App
```
Internet
    ↓
Global Load Balancer (HTTPS, anycast IP)
    ↓
Web tier (Cloud Run / GKE) — tagged: web-server
    ↓ (internal only, no public IP)
App tier (GKE) — tagged: app-server
    ↓ (internal only)
DB tier (Cloud SQL, private IP) — tagged: db-server
```

```bash
# Firewall rules for this pattern:
# LB → web: allow tcp:8080 from 130.211.0.0/22,35.191.0.0/16 (LB health check IPs)
# web → app: allow tcp:8080 from source-tags=web-server to target-tags=app-server
# app → db: allow tcp:5432 from source-tags=app-server to target-tags=db-server
# All VMs: NO public IPs — use Cloud NAT for outbound internet access
```

---

## Useful Network Commands

```bash
# Show VPC details
gcloud compute networks describe my-vpc

# Show routing table
gcloud compute routes list --filter="network=my-vpc"

# Check firewall rules affecting a VM
gcloud compute firewall-rules list \
  --filter="targetTags:web-server OR sourceRanges:0.0.0.0/0" \
  --format="table(name, direction, priority, sourceRanges, targetTags, allowed)"

# Test connectivity (requires gcloud)
gcloud compute ssh my-vm --zone=us-central1-a --command="curl -s ifconfig.me"    # external IP via NAT
gcloud compute ssh my-vm --zone=us-central1-a --command="curl -s db.internal"     # internal DNS
```