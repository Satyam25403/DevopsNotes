# 🌐 Content Delivery Networks (CDN)

> **Series:** System Design Notes  
> **Module:** 04 — CDN  
> **Prerequisites:** load_balancing, caching, Basic DNS knowledge

---

## 📌 Table of Contents

1. [What is a CDN?](#1-what-is-a-cdn)
2. [Core Terminology](#2-core-terminology)
3. [How a CDN Actually Works — Step by Step](#3-how-a-cdn-actually-works--step-by-step)
4. [Routing: How Users Find the Nearest Edge](#4-routing-how-users-find-the-nearest-edge)
5. [Push CDN vs Pull CDN](#5-push-cdn-vs-pull-cdn)
6. [What Can (and Cannot) Be Cached on a CDN](#6-what-can-and-cannot-be-cached-on-a-cdn)
7. [HTTP Cache-Control Headers](#7-http-cache-control-headers)
8. [CDN Cache Invalidation & Purging](#8-cdn-cache-invalidation--purging)
9. [Origin Shield](#9-origin-shield)
10. [Edge Computing (CDN Workers)](#10-edge-computing-cdn-workers)
11. [CDN Security Features](#11-cdn-security-features)
12. [CDN Providers & AWS Mapping](#12-cdn-providers--aws-mapping)
13. [Real-World Architectures](#13-real-world-architectures)
14. [Common Mistakes](#14-common-mistakes)
15. [Interview Cheatsheet](#15-interview-cheatsheet)

---

## 1. What is a CDN?

> **Definition:** A Content Delivery Network (CDN) is a globally distributed network of servers (called **edge servers**) that caches and delivers content to users from a location geographically close to them — reducing latency, offloading the origin server, and improving availability.

The key insight: **move the data closer to the user instead of making the user wait for data to travel from a faraway origin.**

```
WITHOUT CDN:
  User in Mumbai → Origin Server in US (Virginia)
  Round-trip: ~200ms latency + full TCP/TLS handshake cost

WITH CDN:
  User in Mumbai → CDN Edge Server in Mumbai
  Round-trip: ~5ms latency  ← 40x faster
  (TCP/TLS handshake happens locally; CDN talks to origin on optimized backbone)
```

> ⚠️ **Common misconception:** Many developers think CDNs are just "a place to store images."  
> Modern CDNs are sophisticated distributed systems that handle routing, TLS termination, DDoS protection, edge computing, and dynamic content acceleration.

---

## 2. Core Terminology

| Term | Definition |
|---|---|
| **Origin Server** | Your primary server — source of truth. DB, app logic, original files live here |
| **Edge Server** | A CDN server at a PoP; caches and delivers content to nearby users |
| **PoP (Point of Presence)** | A CDN data center. Contains edge servers, LBs, security appliances |
| **Cache Hit** | Request served from edge cache (no origin call) |
| **Cache Miss** | Edge doesn't have the content → fetches from origin or parent cache |
| **TTL (Time-To-Live)** | How long edge caches a response before checking origin |
| **Purge / Invalidation** | Force-removing cached content from edge before TTL expires |
| **Cache Warming** | Pre-loading content onto edge before users request it |
| **Origin Shield** | A dedicated mid-tier cache between edges and origin — shields origin from fan-out |
| **Anycast** | Routing technique where all PoPs share one IP; packets go to nearest node |
| **Cache Busting** | Technique to force fresh content (URL versioning, query strings) |
| **CDR (Cache Hit Ratio)** | `Cache Hits / Total Requests` — measure of CDN effectiveness |
| **TTFB** | Time To First Byte — what CDN primarily reduces |
| **IXP (Internet Exchange Point)** | Physical location where ISPs interconnect. CDN PoPs are often here |

---

## 3. How a CDN Actually Works — Step by Step

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CDN REQUEST LIFECYCLE                                │
│                                                                             │
│  Step 1: User types example.com                                             │
│  Step 2: DNS resolves cdn.example.com → returns nearest edge server IP      │
│  Step 3: User connects to nearest PoP (TCP + TLS handshake — very fast)     │
│  Step 4: Edge checks its local cache                                        │
│          ├── HIT  → Serve from edge immediately ✅                          │
│          └── MISS → Edge fetches from origin (or Origin Shield)             │
│  Step 5: Edge caches the response with TTL                                  │
│  Step 6: Future users in same region → HIT → served immediately             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The TCP + TLS Latency Problem (Why CDN Helps Even Without Cache Hits)

Modern web traffic requires:

```
TCP Handshake:    SYN → SYN-ACK → ACK   = 1 round trip
TLS Handshake:    Client Hello → Server Hello → Key Exchange → Finished = 2 round trips

Total: ~3 round trips BEFORE any data is sent
```

```
User in Mumbai → Origin in Virginia (RTT ≈ 200ms):
  TCP  handshake: 200ms
  TLS  handshake: 400ms  (2 × 200ms)
  Request:        200ms
  Total: ~800ms before first byte arrives

User in Mumbai → PoP in Mumbai (RTT ≈ 5ms):
  TCP  handshake:  5ms
  TLS  handshake: 10ms
  PoP → Origin:  200ms  (on CDN's own optimised backbone, only once)
  Total: ~215ms  ← 3.7x faster even on cache MISS
```

> **Key insight:** CDN accelerates even cache misses because TCP/TLS terminate at the local PoP. The slow origin round-trip only happens for the PoP→Origin leg, not for every user.

---

## 4. Routing: How Users Find the Nearest Edge

### Method 1: DNS-Based Routing

```
User requests: cdn.example.com
    ↓
CDN's authoritative DNS server receives query
DNS checks: where is this resolver/user located?
    ↓
DNS returns IP of nearest PoP
    ↓
User connects directly to that PoP
```

```
DNS returns:
  User in Mumbai   → 103.21.244.1  (Mumbai PoP)
  User in New York → 198.41.212.1  (New York PoP)
  User in London   → 141.101.64.1  (London PoP)
```

**Limitation:** DNS-based routing uses the **resolver's IP**, not the user's IP. A user in India using Google's DNS (8.8.8.8 in US) might get routed to a US PoP. Suboptimal.

---

### Method 2: Anycast Routing ⭐ Modern Standard

> With Anycast, **every PoP advertises the same IP address** to the internet via BGP. The internet's routing infrastructure automatically sends packets to the topologically closest PoP.

```
UNICAST (traditional):
  PoP Mumbai    → IP: 103.21.244.1
  PoP New York  → IP: 198.41.212.1
  PoP London    → IP: 141.101.64.1
  (DNS must pick the right one per user)

ANYCAST (modern):
  PoP Mumbai    → IP: 1.1.1.1  (same IP everywhere!)
  PoP New York  → IP: 1.1.1.1
  PoP London    → IP: 1.1.1.1
  (BGP routes packets to nearest advertising node automatically)
```

```
How BGP Anycast works:

Internet
  │
  ├─ Mumbai Router  → sees 3 paths to 1.1.1.1:
  │                    Path A: Mumbai PoP     (2 hops)   ← chosen ✅
  │                    Path B: Singapore PoP  (8 hops)
  │                    Path C: London PoP     (42 hops)
  │
  └─ London Router  → sees 3 paths to 1.1.1.1:
                       Path A: London PoP     (2 hops)   ← chosen ✅
                       Path B: Amsterdam PoP  (5 hops)
                       Path C: New York PoP   (15 hops)
```

**Benefits of Anycast:**

| Benefit | Explanation |
|---|---|
| **Automatic nearest-PoP routing** | BGP handles it — no DNS geolocation needed |
| **DDoS absorption** | Attack traffic spread across all PoPs globally — each absorbs a fraction |
| **Instant failover** | If Mumbai PoP goes down, BGP reconverges in seconds to next nearest |
| **No DNS TTL issues** | Single IP, no stale DNS entries pointing to dead PoPs |
| **Uses client IP** | Not resolver IP — more accurate geographic routing |

**Used by:** Cloudflare, Google, Fastly, AWS Global Accelerator

---

### Anycast vs DNS Routing Comparison

```
                DNS Routing           Anycast Routing
Latency         ~20–100ms DNS lookup  ~0ms (IP is known)
Failover speed  Minutes (DNS TTL)     Seconds (BGP reconvergence)
DDoS resilience Moderate             Excellent (traffic absorbed by all PoPs)
Accuracy        Resolver IP           Client IP
Complexity      Simple               Requires own BGP infrastructure
```

---

## 5. Push CDN vs Pull CDN

### Pull CDN (Lazy Loading) ⭐ Most Common

> Edge servers fetch content **on demand** — when the first user requests it. The edge "pulls" from origin, caches it, serves future requests from cache.

```
First user requests /images/logo.png:
  → Edge: cache miss
  → Edge fetches from origin
  → Edge caches with TTL
  → Returns to user  (slightly slower)

All subsequent users:
  → Edge: cache HIT
  → Served instantly ✅
```

```
┌─────────────────────────────────────────────────────────┐
│                     PULL CDN FLOW                       │
│                                                         │
│  User ──request──► Edge ──MISS──► Origin                │
│                      │              │                   │
│                      │◄──response───┘                   │
│                      │                                  │
│                      ├── Cache response                 │
│                      │                                  │
│  Next User ──────► Edge ──HIT──► Response ✅            │
└─────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Zero upfront setup — cache fills organically | First request per PoP is always a miss (cold start) |
| Edge only caches what's actually requested | If content is rarely accessed, cache fills/evicts repeatedly |
| Storage at edge = only popular content | Origin gets hit once per PoP per TTL cycle |

**Best for:** Most web applications, mixed static+dynamic, unpredictable traffic patterns

---

### Push CDN (Eager Loading)

> You **proactively push content** to edge servers before any user requests it. Content is pre-loaded at all PoPs.

```
You upload video → Push to all 300 PoPs globally → Every user gets cache HIT
```

```
┌─────────────────────────────────────────────────────────┐
│                     PUSH CDN FLOW                       │
│                                                         │
│  You deploy new content                                 │
│      ↓                                                  │
│  Push to CDN (API / rsync / webhook)                    │
│      ↓                                                  │
│  CDN propagates to all PoPs globally                    │
│      ↓                                                  │
│  Any user, any PoP → cache HIT immediately ✅           │
└─────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Zero cold-start misses — always a HIT | Storage cost: content stored at every PoP even if unpopular |
| Perfect for planned high-traffic events (game launch, product drop) | Operational overhead: must push and manage content manually |
| Consistent performance globally | Stale content risk if you forget to push updates |

**Best for:** Software downloads, game patches, large video files, known viral content, planned launch events (Steam game releases, Apple product pages)

---

### Pull vs Push Summary

| Aspect | Pull CDN | Push CDN |
|---|---|---|
| **Content loading** | On first request | Pre-loaded |
| **First request latency** | Slow (origin fetch) | Fast (always cached) |
| **Storage usage** | Only popular content | All content × all PoPs |
| **Management overhead** | Low (automatic) | High (manual push) |
| **Best for** | General web content | Large files, game patches, planned events |
| **Example** | Cloudflare with origin | AWS S3 → CloudFront with push |

---

## 6. What Can (and Cannot) Be Cached on a CDN

### Content Classification Matrix

```
┌────────────────────────────────────────────────────────────────────────┐
│                      CACHEABILITY SPECTRUM                             │
│                                                                        │
│  HIGHLY CACHEABLE           CONDITIONALLY CACHEABLE     NOT CACHEABLE  │
│  ─────────────────          ────────────────────────    ─────────────  │
│  Images, fonts              API responses (public)      Auth'd pages   │
│  CSS, JS bundles            HTML pages (public)         User dashboards│
│  Videos, audio              Search results              Shopping carts │
│  PDFs, docs                 Product listings            Payment flows  │
│  Static HTML                News articles               Real-time data │
│  Software downloads         Blog posts                  Private files  │
│                                                                        │
│  TTL: Days–Months           TTL: Seconds–Hours          TTL: 0 / no-store│
└────────────────────────────────────────────────────────────────────────┘
```

### What Makes Something Cacheable?

```
✅ Cache it if:
  - Same response for every user (public content)
  - Content is expensive to generate (DB query, render)
  - Content doesn't change frequently
  - No user-specific personalisation in the response

❌ Don't cache it if:
  - Response varies per user (auth tokens, user data)
  - Contains session state or cookies
  - Real-time / must-be-fresh (stock prices, live scores)
  - POST/PUT/DELETE (mutating requests — never cache)
  - Sensitive data (PII, payment info)
```

---

## 7. HTTP Cache-Control Headers

> Cache-Control headers are how your origin server **instructs** CDN edges and browsers on caching behaviour. Getting these right is the most impactful CDN optimisation you can make.

### The Header Hierarchy

```
CDN checks headers in this precedence order:
  1. X-Accel-Expires (nginx-specific)
  2. Cache-Control
  3. Expires  (legacy HTTP/1.0)
  4. CDN dashboard rules (override all)
```

### Core Directives

```http
# Cache publicly for 1 year — for versioned static assets (fingerprinted filenames)
Cache-Control: public, max-age=31536000, immutable

# Cache for 1 hour, but revalidate before serving stale
Cache-Control: public, max-age=3600

# Allow serving stale while fetching fresh in background (no user-visible delay)
Cache-Control: public, max-age=3600, stale-while-revalidate=86400

# Allow serving stale if origin is down
Cache-Control: public, max-age=3600, stale-if-error=86400

# Cache at CDN but NOT in browser (for frequently-updated public content)
Cache-Control: public, s-maxage=3600, no-store
                       ↑ CDN TTL    ↑ browser skips

# Must revalidate — check origin every time before serving
Cache-Control: no-cache

# Never cache anywhere (sensitive data, private user content)
Cache-Control: no-store, private
```

### Directive Reference

| Directive | Meaning |
|---|---|
| `public` | Can be cached by CDN and browser |
| `private` | Only browser cache, not CDN |
| `max-age=N` | Cache for N seconds (browser + CDN) |
| `s-maxage=N` | CDN-only TTL (overrides max-age for CDN) |
| `no-cache` | Always revalidate with origin before serving |
| `no-store` | Never cache anywhere |
| `immutable` | File will never change — skip revalidation checks |
| `stale-while-revalidate=N` | Serve stale for N seconds while fetching fresh in background |
| `stale-if-error=N` | Serve stale for N seconds if origin returns error |
| `must-revalidate` | Never serve stale (even if origin is down) |

### Validation Headers (Conditional Requests)

```
INITIAL RESPONSE (origin → browser/CDN):
  ETag: "abc123def"              ← fingerprint of this version
  Last-Modified: Mon, 01 Jan 2024 12:00:00 GMT

SUBSEQUENT REQUEST (browser/CDN → origin):
  If-None-Match: "abc123def"    ← "do you have a newer version?"
  If-Modified-Since: Mon, 01 Jan 2024 12:00:00 GMT

ORIGIN RESPONSE:
  If unchanged → 304 Not Modified (no body = saves bandwidth) ✅
  If changed   → 200 OK + new content + new ETag
```

### Recommended TTL Strategy by Content Type

| Content Type | Cache-Control | TTL | Invalidation Strategy |
|---|---|---|---|
| Versioned JS/CSS (`app.5f3c9.js`) | `public, max-age=31536000, immutable` | 1 year | New filename on deploy |
| Images (versioned) | `public, max-age=31536000, immutable` | 1 year | New filename |
| Images (unversioned) | `public, max-age=604800` | 7 days | API purge on update |
| Public HTML pages | `public, max-age=300, stale-while-revalidate=3600` | 5 min | API purge on publish |
| API responses (public) | `public, s-maxage=60, stale-while-revalidate=600` | 1 min | API purge |
| User-specific pages | `private, no-store` | Never | N/A |
| Real-time data | `no-store` | Never | N/A |

### `stale-while-revalidate` — Why It Matters

```
Without stale-while-revalidate:
  TTL expires → next user waits for origin fetch → visible latency spike

With stale-while-revalidate:
  TTL expires → next user gets stale (instant) → background fetch updates cache
  → zero latency impact on users, content stays fresh

Cache-Control: public, max-age=60, stale-while-revalidate=600
  ├── 0–60s:   Serve from cache (fresh)
  ├── 60–660s: Serve stale immediately + fetch fresh in background
  └── >660s:   Must fetch fresh (full miss)
```

---

## 8. CDN Cache Invalidation & Purging

> Getting fresh content to edge servers *before* TTL expires. This is the hardest operational problem in CDN.

### Strategy 1: TTL Expiry (Passive)

Let the cache expire naturally. Simple, zero cost. Content can be stale up to TTL duration.

```
Best for: Tolerant content (marketing pages, documentation, image assets)
Worst for: Breaking news, product prices, emergency bug fixes
```

---

### Strategy 2: URL / Path Purge (On-Demand)

Call CDN API to instantly evict specific URLs from all PoPs.

```bash
# Cloudflare — purge specific files
curl -X POST "https://api.cloudflare.com/client/v4/zones/ZONE_ID/purge_cache" \
     -H "Authorization: Bearer TOKEN" \
     -d '{"files": ["https://example.com/api/products/42", 
                    "https://example.com/images/product-42.jpg"]}'

# AWS CloudFront — create invalidation
aws cloudfront create-invalidation \
  --distribution-id DIST_ID \
  --paths "/products/42" "/images/product-42.jpg"
```

**Propagation time:**
- Sub-second to 30s: Modern CDNs with centralised metadata (Cloudflare, Fastly)
- 1–5 minutes: Large networks under high load
- 10+ minutes: Legacy systems or batch invalidation

**Cost:** CloudFront charges for invalidations after 1,000/month. Cloudflare is free.

---

### Strategy 3: Tag-Based / Surrogate Key Purge ⭐ Most Powerful

Tag responses with logical keys. Purge all content matching a tag in one API call.

```http
# Origin server sends this header
Surrogate-Key: product-42 category-electronics user-prices
Cache-Tag: product-42 category-electronics
```

```bash
# One call → purges all content tagged "product-42"
# Affects: product page HTML, product image, product API response, related listings
curl -X POST ".../purge_cache" -d '{"tags": ["product-42"]}'
```

```
Without tags:
  Product 42 updated → manually purge:
    /products/42
    /products/42/images
    /api/products/42
    /category/electronics   (listing page)
    /search?q=laptop        (might include it)
    ... (did you get them all?)

With tags:
  Product 42 updated → purge tag "product-42" → done ✅ (all pages auto-purged)
```

**Used by:** Fastly (Surrogate-Key), Cloudflare (Cache-Tag), Akamai (Edge Cache Tag)

---

### Strategy 4: Cache Busting (URL Versioning) ⭐ Best for Static Assets

> Instead of purging, change the URL of the file. Old URL stays cached harmlessly. New URL is a fresh cache miss with the new content.

```html
<!-- Old deploy -->
<script src="/app.js"></script>
<link href="/styles.css">

<!-- New deploy (Webpack/Vite hashes filenames automatically) -->
<script src="/app.5f3c9a2b.js"></script>
<link href="/styles.9e4d1c7f.css">
```

```
Benefits:
  ✅ No purge API call needed
  ✅ Old URL still cached — rollback is instant
  ✅ New URL served fresh immediately
  ✅ Long TTL (1 year) safe because URL changes with content
  ✅ Works across all browsers and CDNs with zero coordination

Cache-Control: public, max-age=31536000, immutable
```

**This is the gold standard for CSS/JS/images in modern frontend tooling (Webpack, Vite, Next.js all do this automatically).**

---

### Purge Strategy Decision Tree

```
Is content a static asset (CSS, JS, image)?
  └── YES → Use cache busting (versioned filenames) + long TTL
              Never need to purge
  └── NO  → Dynamic content

How many URLs are affected by the change?
  ├── 1–10 URLs → URL/path purge via API
  └── Many URLs across features/categories → Tag-based purge

How fast does fresh content need to reach users?
  ├── Minutes are OK → TTL expiry
  └── Seconds required → On-demand purge API
```

---

## 9. Origin Shield

> An **Origin Shield** (also called Mid-Tier Cache or Shield PoP) is an additional caching layer placed between the CDN edge servers and the origin server.

Without Origin Shield, every PoP that gets a cache miss independently calls your origin:

```
WITHOUT ORIGIN SHIELD:
  200 PoPs × 1 cache miss each on same file
  = 200 simultaneous requests to origin 🔥

Request pattern:
  PoP-Tokyo   → origin
  PoP-Mumbai  → origin
  PoP-London  → origin
  PoP-NYC     → origin
  ... (200 PoPs all hammer origin on TTL expiry)
```

With Origin Shield, all PoP misses converge to one Shield PoP, which makes a single request to origin:

```
WITH ORIGIN SHIELD:
  200 PoPs miss → all forward to Shield PoP
  Shield checks its own cache
    ├── HIT → serve all 200 PoPs from Shield (origin = 0 requests)
    └── MISS → Shield fetches from origin (1 request) → serves all 200 PoPs

  Origin: 1 request per TTL cycle instead of 200 ✅
```

```
┌──────────────────────────────────────────────────────────────────┐
│                     ORIGIN SHIELD TOPOLOGY                       │
│                                                                  │
│  Users worldwide                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │           EDGE LAYER (200+ PoPs globally)                │   │
│  │  Tokyo  Mumbai  London  Sydney  NYC  São Paulo ...        │   │
│  └──────────────────────┬───────────────────────────────────┘   │
│                         │ all misses converge here               │
│  ┌──────────────────────▼───────────────────┐                   │
│  │        ORIGIN SHIELD (1–3 Shield PoPs)   │  ← e.g. us-east  │
│  └──────────────────────┬───────────────────┘                   │
│                         │ single request on miss                 │
│  ┌──────────────────────▼───────────────────┐                   │
│  │              ORIGIN SERVER               │                   │
│  └──────────────────────────────────────────┘                   │
└──────────────────────────────────────────────────────────────────┘
```

**When to use Origin Shield:**

- ✅ Origin server is bandwidth-constrained or expensive
- ✅ Content is same for all users (public pages, media files)
- ✅ High traffic + globally distributed users
- ✅ AWS CloudFront Origin Shield, Fastly Shield, Cloudflare Tiered Cache

---

## 10. Edge Computing (CDN Workers)

> Modern CDNs let you run code **at the edge** — inside the PoP, before/after cache lookup, without hitting your origin. This extends CDN from pure caching to a programmable compute layer.

```
Traditional CDN:
  User → Edge (serve from cache or forward to origin) → User

Edge Computing:
  User → Edge → [Your code runs here at < 5ms latency] → User
                  Can: modify request/response, auth, A/B test,
                       personalise, rate-limit, geo-redirect
```

### Use Cases

```
┌─────────────────────────────────────────────────────────────────┐
│                    EDGE WORKER USE CASES                        │
├─────────────────────────────────────────────────────────────────┤
│  A/B Testing        → Split traffic without origin round-trip   │
│  Auth / JWT verify  → Validate tokens at edge, reject early     │
│  Geo-redirect       → Route /en, /de based on user country      │
│  Image resizing     → Serve right resolution per device         │
│  HTML personalise   → Inject user name into cached HTML         │
│  Rate limiting      → Block abusive IPs at edge                 │
│  Header rewriting   → Add/remove headers for legacy systems     │
│  Bot detection      → Fingerprint and block bots at edge        │
│  Request coalescing → Deduplicate identical in-flight origin    │
└─────────────────────────────────────────────────────────────────┘
```

### Cloudflare Worker Example

```javascript
// Edge worker: A/B test without hitting origin
addEventListener('fetch', event => {
  event.respondWith(handleRequest(event.request))
})

async function handleRequest(request) {
  const url = new URL(request.url)

  // 50/50 A/B split at the edge — no origin needed
  if (url.pathname === '/landing') {
    const variant = Math.random() < 0.5 ? 'a' : 'b'
    url.pathname = `/landing-${variant}`
    return fetch(url.toString())
  }

  return fetch(request)
}
```

### Edge vs Origin Processing

| Task | At Origin | At Edge |
|---|---|---|
| A/B test redirect | ~200ms (origin roundtrip) | ~1ms (edge logic) |
| JWT validation | ~200ms + DB lookup | ~2ms (crypto at edge) |
| Image resize | Server CPU + latency | Edge CDN transform |
| Personalised HTML | Full origin render | Cache + edge inject |

**Providers:** Cloudflare Workers, Fastly Compute@Edge, AWS Lambda@Edge / CloudFront Functions, Vercel Edge Functions

---

## 11. CDN Security Features

CDNs sit at the network edge — naturally positioned to absorb attacks and enforce security policies.

### DDoS Protection

```
WITHOUT CDN:
  100Gbps DDoS attack → hits your origin server → overwhelmed → down

WITH ANYCAST CDN:
  100Gbps DDoS attack → spread across 300 PoPs globally
  Each PoP absorbs ~330Mbps → manageable → origin unaffected

Cloudflare: absorbs 200+ Tbps of DDoS capacity
Akamai: 700+ Tbps globally
```

### TLS Termination at Edge

```
Client ──HTTPS──► Edge (TLS terminates here) ──HTTP──► Origin
                  ↑
  CDN manages:  Certificate rotation, OCSP stapling,
                TLS 1.3, HTTP/2 + HTTP/3 (QUIC) upgrades
```

**Benefits:**
- Origin doesn't need to handle TLS (offloads CPU)
- Modern TLS versions automatically, no origin changes needed
- Centralised certificate management (no per-server cert rotation)

### Web Application Firewall (WAF)

```
Edge WAF inspects HTTP requests and blocks:
  - SQL injection attempts
  - XSS payloads
  - OWASP Top 10 attacks
  - Bad bots (scrapers, credential stuffers)
  - Known malicious IP ranges

Request → WAF rules → Block / Challenge / Allow → Cache / Origin
```

### Bot Management & Rate Limiting

```
Cloudflare, Akamai, Fastly can:
  - Identify bot signatures (headless browsers, scrapers)
  - Rate-limit IPs or ASNs
  - Issue JS challenges (prove you're a browser)
  - Block by country (geo-blocking for compliance)
```

---

## 12. CDN Providers & AWS Mapping

| Provider | Type | Strengths | Notes |
|---|---|---|---|
| **Cloudflare** | Global CDN + Security | Anycast, DDoS, Workers, free tier | 300+ PoPs, most generous free plan |
| **Akamai** | Enterprise CDN | Largest network, media delivery | 4,000+ PoPs, enterprise pricing |
| **Fastly** | Programmable CDN | VCL, instant purge, Compute@Edge | Media, streaming, real-time invalidation |
| **AWS CloudFront** | Managed CDN | Deep AWS integration, Lambda@Edge | Best when already in AWS ecosystem |
| **Google Cloud CDN** | Managed CDN | Google backbone, low latency | Best with GCP |
| **Azure CDN** | Managed CDN | Azure integration | Best with Azure |
| **Vercel Edge** | Frontend CDN | Next.js native, edge functions | Best for frontend deployments |
| **Bunny CDN** | Budget CDN | Cheap egress, simple setup | Small/mid projects |

### AWS CloudFront Key Concepts

```
Distribution:        CloudFront endpoint (d1234.cloudfront.net)
Origin:              S3 bucket, ALB, EC2, API Gateway, custom HTTP
Behaviour:           Path-pattern → cache rules → origin mapping
Cache Policy:        TTL, headers to forward, query strings to include
Origin Request Policy: What to forward to origin (headers, cookies, QS)
Origin Shield:       Enable in specific AWS region
Invalidation:        API call, charged after 1000/month

Path-based routing example:
  /api/*      → ALB (no caching, forward all headers)
  /static/*   → S3 bucket (max-age=1 year)
  /images/*   → S3 bucket (max-age=7 days)
  /*          → ALB (max-age=60s)
```

---

## 13. Real-World Architectures

### Netflix — Video Streaming CDN (Open Connect)

```
Netflix built its OWN CDN (Open Connect) instead of using commercial CDNs.

Why:
  - Videos are huge (4K = 15–25 GB/title)
  - CDN egress costs at that scale = billions/year to commercial CDN
  - Netflix controls their own quality, peering, and cost

Architecture:
  - Open Connect Appliances (OCAs): Netflix-branded servers deployed
    IN partner ISPs' data centres — inside the network, not at IXPs
  - ISP gets Netflix content for free (saves ISP bandwidth)
  - Netflix serves users at near-zero latency (1–2 hops)
  
Content pre-positioning:
  - During off-peak hours (2am–6am): Netflix pushes new content
    to all OCAs proactively (Push CDN model)
  - By morning: all titles cached and ready globally
  
Scale: 15,000+ OCA servers in 1,000+ locations in 175 countries
```

### Cloudflare — Anycast Architecture

```
Infrastructure:
  - 300+ PoPs (2026)
  - Every PoP advertises the same /22 prefix via BGP
  - Traffic routes to nearest PoP by hop count

DDoS defence:
  - Any attack on a Cloudflare IP is spread across all 300+ PoPs
  - 200+ Tbps absorption capacity
  - L3/L4 scrubbing at line rate at the edge
  
Workers (edge compute):
  - Code runs in V8 isolates (not containers) — cold start < 0ms
  - 35M+ requests/second across the network
```

### E-Commerce (Product Catalogue)

```
Static Assets  → CloudFront + S3, max-age=1year, immutable (hashed filenames)
Product Pages  → CloudFront + ALB, s-maxage=300, stale-while-revalidate=3600
               → Tagged with "product-{id}" and "category-{id}"
               → On price update: purge tag "product-42" via API
API responses  → CloudFront + ALB, s-maxage=60 (public /products endpoint)
User cart      → No CDN (Cache-Control: private, no-store)

Deployment:
  CSS/JS: Webpack generates hashed filenames → deploy → zero purge needed
  New product images: upload to S3 → available at edge immediately
  Product update: API purge call in deploy pipeline → fresh content in ~5s
```

### Deployment Pipeline Integration

```yaml
# GitHub Actions — auto-purge on deploy
- name: Purge CDN cache
  run: |
    curl -X POST "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/purge_cache" \
      -H "Authorization: Bearer $CF_TOKEN" \
      -d '{"tags": ["deploy-$(date +%Y%m%d)"]}'
```

---

## 14. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **No Cache-Control headers** | CDN won't cache anything (or uses default short TTL) | Set explicit Cache-Control on every response type |
| **Same URL, changed content, long TTL** | Old content cached for days/weeks | Use versioned filenames (cache busting) or short TTL + purge |
| **Caching authenticated responses** | User A's private data served to User B 🔴 | Add `private` or `no-store` to any response with user-specific data |
| **Caching responses with Set-Cookie** | CDN may cache and replay stale sessions | Strip cookies on cacheable paths or use `s-maxage` without `max-age` |
| **No Origin Shield** | Every PoP hammers origin on TTL expiry | Enable Origin Shield / tiered cache |
| **Purging everything on deploy** | Massive origin traffic spike post-deploy | Use cache busting for assets; purge only dynamic paths |
| **Forgetting `Vary` header** | CDN serves wrong variant (mobile CSS to desktop) | Add `Vary: Accept-Encoding` or `Vary: User-Agent` where needed |
| **No fallback if CDN is down** | CDN outage = your site is down | Set origin as fallback; test CDN bypass path |
| **Single CDN (no redundancy)** | CDN outage = outage | Consider multi-CDN for critical workloads |
| **Not monitoring cache hit ratio** | CDN exists but is not helping — silent failure | Alert on CDR < 80% |

---

## 15. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **CDN** | Distributed edge servers that cache and serve content close to users |
| **PoP** | Physical CDN data center — contains edge servers |
| **Anycast** | All PoPs share one IP; BGP routes to nearest node |
| **Pull CDN** | Edge fetches from origin on first request, then caches |
| **Push CDN** | You proactively upload content to all PoPs |
| **Origin Shield** | Mid-tier cache that collapses all PoP misses into one origin request |
| **s-maxage** | CDN-specific TTL (overrides max-age for CDN only) |
| **Cache busting** | Changing URL of file to force fresh cache (versioned filenames) |
| **Surrogate Key** | Tag on cached responses for group-based purging |
| **stale-while-revalidate** | Serve stale + fetch fresh in background (no latency hit) |
| **Edge Worker** | Code running at the CDN PoP — for auth, A/B test, personalisation |

### When to Use CDN (Scenarios)

| Scenario | Recommendation |
|---|---|
| Static assets (CSS, JS, images) | CDN + versioned filenames + `max-age=1year, immutable` |
| Global video streaming | CDN + Push model (pre-position large files) |
| Public API responses | CDN + `s-maxage=60, stale-while-revalidate=600` |
| DDoS protection | CDN (Anycast absorbs attacks across PoPs) |
| TLS termination offload | CDN (terminate at edge, HTTP to origin) |
| A/B testing without origin | Edge Workers (Cloudflare Workers, Lambda@Edge) |
| E-commerce product pages | CDN + tag-based invalidation on price/stock change |
| User-authenticated pages | **No CDN** (`Cache-Control: private, no-store`) |
| Real-time data (live scores) | **No CDN** or very short TTL + `stale-while-revalidate` |
| Large file downloads (game patches) | CDN + Push model (pre-load before launch) |

### The Interview Answer Template

When CDN comes up in a system design interview:

```
1. WHAT to put on CDN:
   "Static assets with versioned filenames — year-long TTL, immutable.
    Public product/content pages — short TTL with stale-while-revalidate."

2. HOW routing works:
   "Anycast — all PoPs advertise the same IP via BGP.
    User's traffic naturally flows to the nearest PoP."

3. INVALIDATION strategy:
   "For static assets: cache busting via hashed filenames — no purge needed.
    For dynamic pages: tag-based purge on content updates via CI/CD pipeline."

4. ORIGIN protection:
   "Origin Shield collapses all PoP misses into a single origin request —
    critical when we have 300+ PoPs all potentially missing on TTL expiry."

5. SECURITY:
   "CDN handles DDoS (anycast absorption), TLS termination, and WAF —
    origin only sees clean, decrypted traffic."
```

### Must-Know Interview Points

- ☑ **CDN is not just for images.** It handles TLS, DDoS, routing, edge compute.
- ☑ **Anycast is how modern CDNs route.** All PoPs = same IP; BGP picks nearest.
- ☑ **TCP/TLS terminate at edge** — this is why CDN helps even on cache misses.
- ☑ **`stale-while-revalidate`** = best of both worlds: speed + freshness.
- ☑ **Cache busting** = versioned filenames. Never purge static assets manually.
- ☑ **`s-maxage` overrides `max-age` for CDN only** — lets you have different browser vs CDN TTLs.
- ☑ **Never cache `Set-Cookie` or auth'd responses** at CDN.
- ☑ **Origin Shield** collapses fan-out from hundreds of PoPs to one origin request.
- ☑ **Tag-based purge** is the right tool for content-managed / e-commerce sites.
- ☑ **Netflix built their own CDN (Open Connect)** — deployed inside ISPs. Know why.

---

*Sources: DesignGurus, Cloudflare Learning, AlgoMaster Newsletter, BlazingCDN Blog, KeyCDN Blog, DEV Community (CDN/Cloudflare Architecture), USAVPS CDN Optimization — combined with first-principles system design knowledge.*