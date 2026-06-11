# Google Cloud DNS

Google Cloud DNS is GCP's highly available and scalable **DNS (Domain Name System)** service. It provides authoritative DNS serving with 100% SLA uptime — the GCP equivalent of AWS Route 53.

---

## What Cloud DNS Does

- **DNS Management**: Create and manage DNS records that map domain names to GCP resources or any IP
- **Private DNS Zones**: Internal DNS for resources within your VPC (no public exposure)
- **DNS Peering**: Share DNS zones across VPC networks
- **Public DNS**: Serve DNS to the internet from Google's global anycast network
- **Low Latency**: Anycast routing — queries routed to the nearest Google nameserver

> **Domain Registration**: Unlike Route 53, Cloud DNS does not register domain names. Use Google Domains, Namecheap, GoDaddy, etc. to register, then point nameservers to Cloud DNS.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Managed Zone** | A container for DNS records for a specific domain (e.g., `myapp.com`) |
| **Record Set** | A DNS record within a zone (A, CNAME, MX, TXT, etc.) |
| **Public Zone** | Serves DNS to the internet |
| **Private Zone** | Serves DNS only within one or more VPC networks |
| **TTL** | Time To Live — how long DNS resolvers cache a record (in seconds) |
| **Name Servers** | Google's anycast nameservers assigned to your zone (configure at your registrar) |

---

## DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| `A` | Maps domain to IPv4 address | `myapp.com → 34.102.136.180` |
| `AAAA` | Maps domain to IPv6 address | `myapp.com → 2001:db8::1` |
| `CNAME` | Alias to another domain | `www → myapp.com` |
| `MX` | Mail server routing | `myapp.com → mail.google.com` |
| `TXT` | Arbitrary text (SPF, DMARC, verification) | `"v=spf1 include:_spf.google.com ~all"` |
| `NS` | Nameserver records for a subdomain delegation | `sub.myapp.com → ns1.example.com` |
| `SOA` | Start of Authority — zone metadata | Auto-managed by Cloud DNS |
| `PTR` | Reverse DNS (IP → domain) | Used in private zones for internal resolution |
| `SRV` | Service location records | Used by Kubernetes, SIP, etc. |
| `CAA` | Certificate Authority Authorization | `myapp.com CAA 0 issue "letsencrypt.org"` |

---

## CLI Commands

### Create a Managed Zone

```bash
# Public zone (internet-facing)
gcloud dns managed-zones create my-zone \
  --dns-name=myapp.com. \
  --description="My app public DNS zone" \
  --visibility=public

# Private zone (VPC-internal only)
gcloud dns managed-zones create my-private-zone \
  --dns-name=internal.myapp.com. \
  --description="Internal DNS for VPC" \
  --visibility=private \
  --networks=default
```

### Get Nameservers (configure at your registrar)
```bash
gcloud dns managed-zones describe my-zone --format="value(nameServers)"
# Output: ns-cloud-a1.googledomains.com. ns-cloud-a2.googledomains.com. ...
# Paste these 4 nameservers into your domain registrar's NS records
```

### Add DNS Records

```bash
# Start a transaction
gcloud dns record-sets transaction start --zone=my-zone

# A record (apex domain)
gcloud dns record-sets transaction add 34.102.136.180 \
  --name=myapp.com. \
  --type=A \
  --ttl=300 \
  --zone=my-zone

# www CNAME
gcloud dns record-sets transaction add myapp.com. \
  --name=www.myapp.com. \
  --type=CNAME \
  --ttl=300 \
  --zone=my-zone

# Subdomain A record
gcloud dns record-sets transaction add 34.102.136.180 \
  --name=api.myapp.com. \
  --type=A \
  --ttl=60 \
  --zone=my-zone

# TXT record (for domain verification or SPF)
gcloud dns record-sets transaction add '"v=spf1 include:_spf.google.com ~all"' \
  --name=myapp.com. \
  --type=TXT \
  --ttl=3600 \
  --zone=my-zone

# Commit the transaction
gcloud dns record-sets transaction execute --zone=my-zone
```

### Modify / Delete Records

```bash
# List all records in a zone
gcloud dns record-sets list --zone=my-zone

# Update a record (transaction required)
gcloud dns record-sets transaction start --zone=my-zone
gcloud dns record-sets transaction remove OLD_IP \
  --name=myapp.com. --type=A --ttl=300 --zone=my-zone
gcloud dns record-sets transaction add NEW_IP \
  --name=myapp.com. --type=A --ttl=300 --zone=my-zone
gcloud dns record-sets transaction execute --zone=my-zone

# Or use the simpler update command (no transaction)
gcloud dns record-sets update myapp.com. \
  --type=A \
  --ttl=300 \
  --rrdatas=NEW_IP \
  --zone=my-zone

# Delete a record
gcloud dns record-sets delete api.myapp.com. \
  --type=A \
  --zone=my-zone
```

---

## Private DNS Zones (Internal VPC DNS)

Private zones allow GCP resources within your VPC to resolve internal hostnames — without exposing them publicly.

```bash
# Create a private zone
gcloud dns managed-zones create internal-zone \
  --dns-name=internal. \
  --visibility=private \
  --networks=my-vpc

# Add internal A records
gcloud dns record-sets transaction start --zone=internal-zone
gcloud dns record-sets transaction add 10.0.0.5 \
  --name=db.internal. --type=A --ttl=300 --zone=internal-zone
gcloud dns record-sets transaction add 10.0.0.10 \
  --name=redis.internal. --type=A --ttl=300 --zone=internal-zone
gcloud dns record-sets transaction execute --zone=internal-zone

# Now VMs in the VPC can resolve:
# db.internal → 10.0.0.5
# redis.internal → 10.0.0.10
```

---

## DNS Peering

Share a private zone across multiple VPC networks:

```bash
# Allow "consumer-vpc" to use DNS records from a zone owned by "producer-vpc"
gcloud dns managed-zones update internal-zone \
  --dns-name=internal. \
  --visibility=private \
  --networks=producer-vpc \
  --target-network=consumer-vpc \
  --target-project=consumer-project
```

---

## Routing Policies (Traffic Steering)

Cloud DNS supports routing policies for failover, geolocation, and weighted routing:

```bash
# Weighted round-robin (split traffic between two IPs)
gcloud dns record-sets create api.myapp.com. \
  --type=A --ttl=30 \
  --zone=my-zone \
  --routing-policy-type=WRR \
  --routing-policy-data="0.8=1.2.3.4;0.2=5.6.7.8"    # 80%/20% split

# Geolocation routing
gcloud dns record-sets create api.myapp.com. \
  --type=A --ttl=30 \
  --zone=my-zone \
  --routing-policy-type=GEO \
  --routing-policy-data="us-east1=1.2.3.4;europe-west1=5.6.7.8"

# Failover (primary/backup)
gcloud dns record-sets create api.myapp.com. \
  --type=A --ttl=30 \
  --zone=my-zone \
  --routing-policy-type=FAILOVER \
  --routing-policy-primary-data=1.2.3.4 \
  --routing-policy-backup-data=5.6.7.8 \
  --health-check=my-health-check
```

---

## Typical Setup: Custom Domain on Cloud Run / Load Balancer

```bash
# 1. Get the IP of your Load Balancer
gcloud compute forwarding-rules describe my-https-rule --global \
  --format="value(IPAddress)"

# 2. Create an A record pointing to it
gcloud dns record-sets update myapp.com. \
  --type=A --ttl=300 \
  --rrdatas=LB_IP \
  --zone=my-zone

# 3. For Cloud Run (use CNAME to ghs.googlehosted.com)
# First, map domain in Cloud Run:
gcloud run domain-mappings create \
  --service=my-service \
  --domain=myapp.com \
  --region=us-central1

# Then add the CNAME Cloud Run shows you
gcloud dns record-sets update www.myapp.com. \
  --type=CNAME --ttl=300 \
  --rrdatas=ghs.googlehosted.com. \
  --zone=my-zone
```

---

## Testing DNS

```bash
# Test DNS resolution
dig myapp.com A
dig www.myapp.com CNAME
nslookup myapp.com 8.8.8.8

# Check propagation (TTL affects how quickly changes spread globally)
# Temporarily lower TTL to 60s before making changes, then restore after
```