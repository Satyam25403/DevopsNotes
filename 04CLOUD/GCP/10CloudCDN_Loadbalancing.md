# Google Cloud CDN & Load Balancing

Google Cloud CDN and Cloud Load Balancing work together to accelerate content delivery and distribute traffic globally. Cloud CDN caches content at Google's edge, while Cloud Load Balancing routes traffic to healthy backends — together they're the GCP equivalent of AWS CloudFront + ALB/NLB.

---

## Why CDNs and Load Balancers Exist

A server in `us-central1` is physically far from a user in Tokyo. Every request travels thousands of kilometers — adding latency. CDNs solve this by caching content at **edge locations** distributed globally, so users get data from a nearby server instead of your origin.

**Benefits of Cloud CDN + Load Balancing:**
- Dramatically reduced latency for end users
- Reduced load and egress cost on your origin backends
- Built-in DDoS protection via Google Cloud Armor
- Automatic HTTPS with managed TLS certificates
- Global anycast IP — one IP address, routes to the nearest Google edge

---

## How Cloud CDN Works

```
User → Google Edge (PoP) → [Cache Hit] → Serve immediately
                          → [Cache Miss] → Fetch from Backend → Cache → Serve
```

1. User requests content (image, API response, HTML page)
2. Routed to the nearest of 100+ global Points of Presence (PoPs)
3. If cached (cache hit) → served instantly from the edge
4. If not cached (cache miss) → fetches from your backend (Cloud Storage, Cloud Run, GCE, GKE)

---

## Cloud Load Balancer Types

| Type | Layer | Use Case |
|------|-------|----------|
| **Global External Application LB** | L7 (HTTP/S) | HTTPS apps, CDN, path routing — most common |
| **Regional External Application LB** | L7 (HTTP/S) | Single-region HTTPS apps |
| **External Network LB (TCP/UDP)** | L4 | Non-HTTP TCP/UDP, preserve client IPs |
| **Internal Application LB** | L7 (HTTP/S) | Internal microservices |
| **Internal TCP/UDP LB** | L4 | Internal TCP/UDP services |

> For most web apps and APIs, use the **Global External Application Load Balancer** — it includes CDN, SSL termination, and global anycast routing.

---

## Setting Up HTTPS Load Balancer + CDN

### Via CLI (full setup)

```bash
# 1. Create a backend bucket (for static assets from Cloud Storage)
gcloud compute backend-buckets create my-static-backend \
  --gcs-bucket-name=my-static-bucket \
  --enable-cdn

# 2. Create a backend service (for dynamic traffic to Cloud Run / GCE / GKE)
gcloud compute backend-services create my-app-backend \
  --protocol=HTTPS \
  --port-name=https \
  --global

# 3. Create a URL map (route rules)
gcloud compute url-maps create my-url-map \
  --default-service=my-app-backend

# Add path-based routing (serve static assets from CDN, rest from backend)
gcloud compute url-maps import my-url-map --global << 'EOF'
defaultService: my-app-backend
hostRules:
  - hosts: ["mysite.com", "www.mysite.com"]
    pathMatcher: main-paths
pathMatchers:
  - name: main-paths
    defaultService: my-app-backend
    pathRules:
      - paths: ["/static/*", "/images/*"]
        service: my-static-backend
EOF

# 4. Provision a managed SSL certificate
gcloud compute ssl-certificates create my-cert \
  --domains=mysite.com,www.mysite.com \
  --global

# 5. Create a target HTTPS proxy
gcloud compute target-https-proxies create my-https-proxy \
  --url-map=my-url-map \
  --ssl-certificates=my-cert \
  --global

# 6. Create a global forwarding rule (the public IP)
gcloud compute forwarding-rules create my-https-rule \
  --address-type=EXTERNAL \
  --global \
  --target-https-proxy=my-https-proxy \
  --ports=443

# Redirect HTTP → HTTPS
gcloud compute url-maps create http-redirect \
  --default-redirect-response-code=301 \
  --redirect-to-https

gcloud compute target-http-proxies create http-redirect-proxy \
  --url-map=http-redirect

gcloud compute forwarding-rules create http-to-https-rule \
  --address-type=EXTERNAL \
  --global \
  --target-http-proxy=http-redirect-proxy \
  --ports=80
```

---

## CDN Cache Control

### Cache Headers from Origin
Cloud CDN respects standard HTTP cache headers from your backend:
```
Cache-Control: public, max-age=3600          # Cache for 1 hour
Cache-Control: no-store                       # Don't cache
Cache-Control: public, s-maxage=86400        # CDN caches for 24h, browser for less
```

### Override TTL via Cache Mode

```bash
gcloud compute backend-services update my-app-backend \
  --global \
  --cache-mode=CACHE_ALL_STATIC    # Cache all static assets, use headers for others
  # Options: USE_ORIGIN_HEADERS | CACHE_ALL_STATIC | FORCE_CACHE_ALL

# Set a default TTL for cache misses when no Cache-Control header exists
gcloud compute backend-services update my-app-backend \
  --global \
  --default-ttl=3600 \
  --max-ttl=86400 \
  --client-ttl=600
```

### Cache Key Customization

```bash
# Include query string parameters in the cache key (e.g., ?version=2)
gcloud compute backend-services update my-app-backend \
  --global \
  --cache-key-query-string-whitelist=version,lang
```

---

## Cache Invalidation

```bash
# Invalidate a specific URL
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --path="/images/logo.png" \
  --global

# Invalidate all paths under a prefix
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --path="/static/*" \
  --global

# Invalidate everything
gcloud compute url-maps invalidate-cdn-cache my-url-map \
  --path="/*" \
  --global
```

---

## Cloud Armor (DDoS + WAF)

Cloud Armor provides DDoS protection and a Web Application Firewall (WAF) for your load balancer.

```bash
# Create a security policy
gcloud compute security-policies create my-waf-policy \
  --description="WAF policy for my app"

# Block a specific IP range
gcloud compute security-policies rules create 1000 \
  --security-policy=my-waf-policy \
  --src-ip-ranges=192.0.2.0/24 \
  --action=deny-403

# Allow only traffic from specific countries (geo-restriction)
gcloud compute security-policies rules create 2000 \
  --security-policy=my-waf-policy \
  --expression="origin.region_code != 'IN' && origin.region_code != 'US'" \
  --action=deny-403

# Enable OWASP Top 10 preconfigured WAF rules
gcloud compute security-policies rules create 3000 \
  --security-policy=my-waf-policy \
  --expression="evaluatePreconfiguredExpr('sqli-stable')" \
  --action=deny-403

# Attach policy to backend service
gcloud compute backend-services update my-app-backend \
  --global \
  --security-policy=my-waf-policy
```

---

## Load Balancer Health Checks

```bash
# Create an HTTPS health check
gcloud compute health-checks create https my-health-check \
  --port=443 \
  --request-path=/health \
  --check-interval=10s \
  --timeout=5s \
  --healthy-threshold=2 \
  --unhealthy-threshold=3

# Attach to backend service
gcloud compute backend-services update my-app-backend \
  --global \
  --health-checks=my-health-check
```

---

## Viewing CDN Cache Hit Rate (Monitoring)

```bash
# View CDN metrics via Cloud Monitoring
# Key metrics to monitor:
# - loadbalancing.googleapis.com/https/request_count (split by cache_result)
# - loadbalancing.googleapis.com/https/total_latencies
# - loadbalancing.googleapis.com/https/backend_latencies

# Quick log-based query for cache hits/misses
gcloud logging read \
  'resource.type="http_load_balancer" AND httpRequest.cacheHit=true' \
  --limit=10
```

---

## Cloud CDN vs Using Cloud Storage Directly

| Feature | Cloud CDN + LB | Cloud Storage Direct |
|---------|---------------|---------------------|
| HTTPS + Custom domain | ✅ Easy | ❌ Requires LB |
| Global caching | ✅ 100+ PoPs | ❌ Single region |
| DDoS protection | ✅ Cloud Armor | ❌ |
| URL routing | ✅ Path-based | ❌ |
| Cost | LB + CDN fees | Storage + egress only |

> For production use with a custom domain, always front Cloud Storage with a Load Balancer + Cloud CDN.