# Amazon Route 53

Amazon Route 53 is AWS's highly available and scalable **DNS (Domain Name System)** web service. Named after port 53 — the standard port for DNS traffic — it handles domain registration, DNS routing, and health monitoring for your infrastructure.

---

## What Route 53 Does

- **Domain Registration**: Register new domain names directly through AWS
- **DNS Management**: Create and manage DNS records that map domain names to AWS resources or any IP
- **Traffic Routing**: Route users to the right endpoint using intelligent routing policies
- **Health Checks**: Monitor endpoints and automatically reroute traffic when something goes down

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Hosted Zone** | A container for DNS records for a specific domain (e.g., `myapp.com`) |
| **Record Set** | A DNS record within a hosted zone (A, CNAME, MX, etc.) |
| **Alias Record** | AWS-native record type that points to AWS resources (S3, CloudFront, ALB) — free and resolves faster than CNAME |
| **TTL** | Time To Live — how long DNS resolvers cache a record before re-querying |
| **Health Check** | Route 53 monitors an endpoint and can trigger failover if it becomes unhealthy |

---

## DNS Record Types

| Type | Purpose | Example |
|------|---------|---------|
| **A** | Maps domain to IPv4 address | `myapp.com → 1.2.3.4` |
| **AAAA** | Maps domain to IPv6 address | `myapp.com → 2001:db8::1` |
| **CNAME** | Alias to another domain | `www.myapp.com → myapp.com` |
| **Alias** | AWS-native, points to AWS resource | `myapp.com → ALB DNS name` |
| **MX** | Mail server records | For email routing |
| **TXT** | Text records | Domain verification, SPF records |
| **NS** | Name server records | Delegates DNS for a zone |

> **Alias vs CNAME**: Use **Alias** for AWS resources (it's free, faster, and works at the zone apex). Use CNAME for non-AWS targets.

---

## Setting Up a Domain

### Option 1: Register in Route 53
1. Go to **Route 53 → Registered Domains → Register domain**
2. Search for and purchase your domain
3. Route 53 automatically creates a hosted zone

### Option 2: Transfer Existing Domain
If your domain is registered elsewhere (Namecheap, GoDaddy, etc.):
1. Create a **Hosted Zone** in Route 53 for your domain
2. Note the **Name Servers (NS records)** provided
3. Update the name servers at your registrar to point to Route 53's NS records
4. DNS propagation takes up to 48 hours

---

## Common DNS Setups

### Point Domain to an EC2 Instance
```
myapp.com  →  A record  →  <EC2 Elastic IP>
```

### Point Domain to a Load Balancer
```
myapp.com  →  Alias A record  →  <ALB DNS name>
```

### Point Domain to CloudFront
```
myapp.com  →  Alias A record  →  <CloudFront distribution domain>
www.myapp.com  →  Alias A record  →  <CloudFront distribution domain>
```

### Subdomain Routing
```
app.myapp.com    →  A record / Alias  →  app server
api.myapp.com    →  A record / Alias  →  API server
blog.myapp.com   →  CNAME             →  blog.myapp.com.s3-website-us-east-1.amazonaws.com
```

---

## Routing Policies

Route 53 supports intelligent traffic routing beyond simple DNS:

| Policy | Use Case |
|--------|----------|
| **Simple** | Single endpoint — basic DNS |
| **Weighted** | Split traffic between endpoints (e.g., 90/10 for canary deployments) |
| **Latency-based** | Route to the AWS region with lowest latency for the user |
| **Geolocation** | Route based on user's country or continent |
| **Geoproximity** | Route based on geographic distance with bias settings |
| **Failover** | Primary/secondary — failover to backup if primary is unhealthy |
| **Multivalue Answer** | Return up to 8 healthy records for basic client-side load balancing |

### Weighted Routing Example (Blue/Green or Canary)
```
myapp.com  →  Weight: 90  →  v1 server
myapp.com  →  Weight: 10  →  v2 server  (canary: 10% of traffic)
```

### Failover Routing Example
```
myapp.com  →  PRIMARY  →  us-east-1 ALB  (health check enabled)
myapp.com  →  SECONDARY →  us-west-2 ALB  (only used if primary fails)
```

---

## Health Checks

Route 53 can monitor your endpoints and automatically remove unhealthy records from DNS responses.

1. Go to **Route 53 → Health Checks → Create**
2. Specify:
   - Protocol (HTTP, HTTPS, TCP)
   - Endpoint (IP or domain)
   - Path (e.g., `/health`)
   - Threshold (how many failures = unhealthy)
3. Attach the health check to a DNS record

**Common health check endpoint:**
```javascript
// Express.js example
app.get('/health', (req, res) => {
  res.status(200).json({ status: 'ok', timestamp: new Date().toISOString() });
});
```

---

## CLI Commands

```bash
# List hosted zones
aws route53 list-hosted-zones

# List records in a hosted zone
aws route53 list-resource-record-sets \
  --hosted-zone-id Z1234567890ABC

# Create/update a record (upsert)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "1.2.3.4" }]
      }
    }]
  }'

# Check DNS propagation
dig myapp.com
nslookup myapp.com 8.8.8.8

# List health checks
aws route53 list-health-checks
```

---

## Route 53 + CloudFront + ACM (Full HTTPS Setup)

The typical production setup for a custom domain with HTTPS:

1. **Register domain** in Route 53
2. **Request ACM certificate** in `us-east-1` for `myapp.com` and `*.myapp.com`
3. **Validate certificate** via DNS (Route 53 can do this automatically)
4. **Create CloudFront distribution** with:
   - Origin: S3 bucket or ALB
   - Alternate domain names: `myapp.com`, `www.myapp.com`
   - SSL certificate: the ACM cert
5. **Create Route 53 Alias records** pointing `myapp.com` → CloudFront distribution

---

## Private Hosted Zones

Route 53 also supports **private hosted zones** — DNS that only resolves within your VPC:
```
db.internal  →  RDS endpoint  (only accessible inside the VPC)
cache.internal  →  ElastiCache endpoint
```

This allows internal service discovery without exposing resources publicly.

---

## Cost

- **Hosted zones**: $0.50/month per hosted zone (first 25 zones free)
- **DNS queries**: $0.40 per million queries (standard), $0.60 for latency/geo routing
- **Health checks**: ~$0.50/month per endpoint
- **Domain registration**: varies by TLD (`.com` is ~$12/year)