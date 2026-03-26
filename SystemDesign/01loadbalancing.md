# ⚖️ Load Balancing

> **Series:** System Design Notes  
> **Module:** 02 — Load Balancing  
> **Prerequisites:** Networking basics (OSI Model, TCP/UDP), Horizontal vs Vertical Scaling

---

## 📌 Table of Contents

1. [What is a Load Balancer?](#1-what-is-a-load-balancer)
2. [Why Do We Need It?](#2-why-do-we-need-it)
3. [How It Works — Big Picture](#3-how-it-works--big-picture)
4. [Load Balancing Algorithms](#4-load-balancing-algorithms)
5. [L4 vs L7 Load Balancing](#5-l4-vs-l7-load-balancing)
6. [Health Checks](#6-health-checks)
7. [Sticky Sessions (Session Persistence)](#7-sticky-sessions-session-persistence)
8. [TLS Termination](#8-tls-termination)
9. [High Availability of the Load Balancer Itself](#9-high-availability-of-the-load-balancer-itself)
10. [Real-World Tools & AWS Mapping](#10-real-world-tools--aws-mapping)
11. [Real-World Architectures](#11-real-world-architectures)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview Cheatsheet](#13-interview-cheatsheet)

---

## 1. What is a Load Balancer?

> **Definition:** A Load Balancer is a component that sits between clients and backend servers, distributing incoming traffic across a pool of machines so that no single server becomes a bottleneck or single point of failure.

Think of it as a **traffic cop at a highway toll plaza** — it decides which lane (server) each car (request) goes to, keeps things moving, and reroutes if a lane closes.

```
         Clients
        /   |   \
       /    |    \
      ↓     ↓     ↓
  ┌─────────────────────┐
  │     LOAD BALANCER   │
  └─────────────────────┘
       /    |    \
      ↓     ↓     ↓
  Server1 Server2 Server3
```

---

## 2. Why Do We Need It?

### The Problem: Single Server Setup

```
Client → [ Server ] → Response
```

| Problem | Impact |
|---|---|
| **Single Point of Failure** | If server crashes → entire app goes down |
| **Limited Scalability** | One machine can only handle so many requests |
| **Performance Degradation** | As traffic grows, latency increases for everyone |
| **No Redundancy** | Maintenance window = downtime |

### The Solution: Load Balancer + Server Pool

```
                    ┌──────────────┐
Client Requests →   │ LOAD BALANCER│  ← Health Monitor
                    └──────┬───────┘
              ┌────────────┼────────────┐
              ↓            ↓            ↓
          Server A     Server B     Server C
          (Healthy)    (Healthy)   (Healthy ✓)
```

**What you gain:**

- ✅ **High Availability** — if one server fails, traffic reroutes automatically
- ✅ **Horizontal Scalability** — just add more servers to the pool
- ✅ **Better Performance** — no single machine is overwhelmed
- ✅ **Zero-Downtime Deployments** — take servers out of rotation one by one (rolling deploy)

---

## 3. How It Works — Big Picture

```
Step 1: Client sends request to Load Balancer's Virtual IP (VIP)
Step 2: LB runs its algorithm → picks a backend server
Step 3: LB forwards the request (either by NAT or Proxy)
Step 4: Server processes and responds
Step 5: Response returns to client (through LB or directly)
```

> **Virtual IP (VIP):** A single IP address that represents the entire server pool. Clients never know individual server IPs — they only see the VIP.

---

## 4. Load Balancing Algorithms

### 4.1 Round Robin

Requests are distributed to servers in sequential rotation.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  ← cycle repeats
Request 5 → Server B
```

```
LB ──► A ──► B ──► C ──► A ──► B ──► C ...
       ↑ equal share for each server ↑
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to implement | Doesn't account for server load |
| Predictable distribution | Slow requests on one server don't affect routing |
| Works well for equal-capacity servers | |

**Best for:** Homogeneous servers, stateless services, uniform request sizes

---

### 4.2 Weighted Round Robin

Each server gets a weight proportional to its capacity. Heavier machines take more requests.

```
Server A (weight=3): ████████
Server B (weight=2): █████
Server C (weight=1): ███
```

```
Pattern per 6 requests: A, A, A, B, B, C
```

| ✅ Pros | ❌ Cons |
|---|---|
| Works for mixed instance types (2vCPU, 4vCPU, 8vCPU) | Not real-time load aware |
| Simple extension of Round Robin | Slow/noisy server still gets its share |

**Best for:** Heterogeneous server environments (mixed instance sizes)

---

### 4.3 Least Connections

Routes each new request to the server with the **fewest active connections** at that moment. Dynamic — responds to actual load.

```
Incoming request...

Server A: 10 active connections
Server B:  5 active connections  ← request goes HERE
Server C:  8 active connections
```

| ✅ Pros | ❌ Cons |
|---|---|
| Adapts to varying request durations | Needs to track connection state per server |
| Naturally handles long-running requests | Slight overhead vs Round Robin |
| Avoids overloading slower servers | |

**Best for:** Workloads where requests vary significantly in duration (DB queries, file uploads, streaming)

---

### 4.4 Weighted Least Connections

Combines Least Connections with server weights.

```
Score = Active Connections / Weight

Server A: 10 connections, weight=5  →  Score = 2.0  ← WINNER (lowest score)
Server B:  6 connections, weight=2  →  Score = 3.0
Server C:  4 connections, weight=1  →  Score = 4.0

→ Next request goes to Server A
```

**Best for:** Heterogeneous environments with variable request durations

---

### 4.5 IP Hash (Sticky by IP)

Hash the client's IP address → deterministically map to a server. Same client always hits the same server.

```
hash(192.168.1.10) % 3 = 1  →  Server B
hash(192.168.1.20) % 3 = 0  →  Server A
hash(192.168.1.30) % 3 = 2  →  Server C
```

| ✅ Pros | ❌ Cons |
|---|---|
| No cookies needed for session persistence | Uneven distribution if IPs aren't spread uniformly |
| Stateless — no extra bookkeeping | Adding/removing servers redistributes all clients |
| | NAT'd clients (e.g. corporate proxies) all hit same server |

**Best for:** Basic session affinity, legacy apps without cookie support

---

### 4.6 Least Response Time

Routes to the server with the **lowest combination of response latency + active connections**. LB continuously measures avg response time per server.

```
Server A: avg 80ms,  10 connections  →  Score high
Server B: avg 20ms,   3 connections  →  Score LOW ← gets request
Server C: avg 50ms,   7 connections  →  Score medium
```

| ✅ Pros | ❌ Cons |
|---|---|
| Optimizes for perceived performance | Highest operational complexity |
| Can detect degraded servers early | Can "overreact" to noise (feedback loops) |

**Best for:** Latency-sensitive apps (real-time, gaming, financial trading)

---

### Algorithm Decision Tree

```
Are all servers identical capacity?
├── YES → Round Robin (simple) or Least Connections (dynamic)
└── NO (mixed sizes)
    ├── Requests mostly uniform duration? → Weighted Round Robin
    └── Requests vary in duration?       → Weighted Least Connections

Need same user → same server?
├── Don't mind cookie overhead → Cookie-based Sticky Sessions (L7)
└── Can't use cookies          → IP Hash

Latency is the #1 concern?
└── Least Response Time
```

---

## 5. L4 vs L7 Load Balancing

This is the **most important concept** for system design interviews.

### OSI Model Context

```
Layer 7 — Application   (HTTP, HTTPS, WebSocket, gRPC)   ← L7 LB lives here
Layer 6 — Presentation
Layer 5 — Session
Layer 4 — Transport     (TCP, UDP)                        ← L4 LB lives here
Layer 3 — Network       (IP)
Layer 2 — Data Link
Layer 1 — Physical
```

---

### L4 Load Balancer (Transport Layer)

> Works like a **postal courier** — it knows the **address and destination port**, but never opens the envelope. It routes packets without looking at content.

```
Client TCP SYN
       ↓
[ L4 Load Balancer ]
  Sees: Source IP, Dest IP, Source Port, Dest Port, Protocol
  Does NOT see: HTTP headers, cookies, URL paths
       ↓
Backend Server (via NAT or DSR)
```

**What it can route on:**
- Source/Destination IP
- Source/Destination Port
- Protocol (TCP/UDP)

**What it CANNOT do:**
- Route based on URL paths (`/api` vs `/static`)
- SSL termination (packet contents opaque)
- Cookie inspection
- A/B testing or canary releases

**Characteristics:**

| Property | Value |
|---|---|
| Latency | ~50–100 µs |
| Throughput | 10–40 Gbps per node |
| Connection model | Packet forwarding (NAT or DSR) |
| SSL Termination | ❌ No |
| Content-aware routing | ❌ No |
| Protocol support | Any TCP/UDP |

**AWS equivalent:** `Network Load Balancer (NLB)`  
**Self-hosted:** `HAProxy (TCP mode)`, `IPVS/LVS (Linux kernel)`

---

### L7 Load Balancer (Application Layer)

> Works like a **smart receptionist** — it reads your request fully, understands what you want, and routes you to the right department.

```
Client HTTP Request
        ↓
[ L7 Load Balancer ]  ← TERMINATES the client connection
  Parses: URL path, HTTP headers, cookies, query params, body
  Can inspect: Host header, Authorization, Content-Type
        ↓ (new connection opened to backend)
Backend Server
```

**What it can route on:**
- URL paths (`/api/*` → API servers, `/static/*` → CDN origin)
- HTTP methods (GET vs POST)
- Headers (`Host: api.example.com`)
- Cookies / Session tokens
- Query parameters

**Deep Packet Inspection (DPI):** L7 LBs read the actual request content — this is how content-aware routing works.

**Characteristics:**

| Property | Value |
|---|---|
| Latency | ~0.5–3 ms |
| Throughput | 1–5 Gbps per node |
| Connection model | Full proxy (terminate + re-originate) |
| SSL Termination | ✅ Yes |
| Content-aware routing | ✅ Yes |
| Protocol support | HTTP, HTTPS, HTTP/2, gRPC, WebSocket |

**AWS equivalent:** `Application Load Balancer (ALB)`  
**Self-hosted:** `NGINX (HTTP mode)`, `HAProxy (HTTP mode)`, `Envoy`, `Traefik`

---

### L4 vs L7 Comparison Table

| Feature | L4 | L7 |
|---|---|---|
| **OSI Layer** | Transport | Application |
| **Routing Basis** | IP + Port | URL, Headers, Cookies |
| **Connection Model** | Packet forwarding | Full proxy |
| **Latency** | ~50–100 µs | ~0.5–3 ms |
| **SSL Termination** | ❌ | ✅ |
| **Content-based routing** | ❌ | ✅ |
| **Sticky Sessions (Cookie)** | ❌ | ✅ |
| **Rate Limiting** | ❌ | ✅ |
| **A/B Testing / Canary** | ❌ | ✅ |
| **Throughput** | Very High | High |
| **AWS Service** | NLB | ALB |
| **Use Case** | TCP/UDP, game servers, raw throughput | Web apps, APIs, microservices |

---

### Two-Tier Architecture (Production Pattern)

Most large systems use **both layers together:**

```
Internet
    ↓
[ L4 Load Balancer ]  ← Edge. Handles DDoS, raw TCP, anycast VIPs.
    ↓                    Fast. Stateless. 10–40 Gbps.
[ L7 Load Balancer ]  ← Smart routing. TLS termination. Cookie handling.
    ↓                    Path-based routing to microservices.
Microservices / App Servers
```

> **Netflix:** L4 regional balancers → L7 Zuul/Envoy gateways  
> **Google:** Maglev (L4, consistent hashing) → GFE (L7, TLS + routing)  
> **AWS pattern:** NLB (L4, static IP, ultra-low latency) → ALB (L7, host/path routing)

---

### NGINX L7 Config Example

```nginx
http {
  upstream api_service {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
  }

  upstream web_service {
    server 10.0.0.20:80;
    server 10.0.0.21:80;
  }

  server {
    listen 80;

    # Route /api/* → API servers
    location /api/ {
      proxy_pass http://api_service;
    }

    # Route everything else → Web servers
    location / {
      proxy_pass http://web_service;
    }
  }
}
```

---

## 6. Health Checks

Without health checks, the LB would keep sending traffic to dead servers. Health checks are how the LB knows who's alive.

### Types of Health Checks

#### Active Health Checks (LB probes the servers)

```
Every N seconds, LB sends a probe to each server:
    ↓
L4 Check: Can I open a TCP connection to port 8080?
              → Success = server is alive (network level)

L7 Check: Does GET /health return HTTP 200?
              → Success = app is running correctly (app level)
```

#### Passive Health Checks (LB watches real traffic)

```
LB observes responses to real requests:
    If server returns 5xx errors repeatedly
    OR response time > threshold
    → Mark server as unhealthy, stop sending traffic
```

### Health Check Flow

```
┌─────────────────────────────────────────────────────────┐
│                    HEALTH CHECK CYCLE                   │
│                                                         │
│  LB polls servers every 5s                             │
│                                                         │
│  Server A: ✅ 200 OK  →  Keep in rotation              │
│  Server B: ✅ 200 OK  →  Keep in rotation              │
│  Server C: ❌ Timeout →  Mark UNHEALTHY                 │
│                           Stop sending traffic          │
│                           Keep polling...               │
│                           ✅ 200 OK → Mark HEALTHY      │
│                           Re-add to rotation            │
└─────────────────────────────────────────────────────────┘
```

### Key Health Check Parameters

| Parameter | Typical Value | Meaning |
|---|---|---|
| `interval` | 5–10s | How often to check |
| `timeout` | 3–5s | Max wait for response |
| `healthy_threshold` | 2 | Consecutive successes to mark healthy |
| `unhealthy_threshold` | 3 | Consecutive failures to mark unhealthy |
| `path` | `/health` or `/ping` | Endpoint to hit (L7) |

> **Pro tip:** Design a dedicated `/health` endpoint that checks your app's actual dependencies (DB connection, cache) — not just "is the process running."

---

## 7. Sticky Sessions (Session Persistence)

### The Problem

```
User adds item to cart → request hits Server A → cart saved in Server A's memory
User clicks checkout   → request hits Server B → Server B has no cart data → 💥
```

This happens in **stateful applications** where session data is stored in server memory instead of a shared store.

### The Solution: Sticky Sessions

Sticky sessions ensure that **all requests from the same user always go to the same backend server.**

#### Method 1: Cookie-Based (L7 only)

```
1. User makes first request → LB picks Server 1
2. LB injects cookie: Set-Cookie: SERVERID=server1; Path=/
3. Browser stores cookie
4. All subsequent requests include Cookie: SERVERID=server1
5. LB reads cookie → routes to Server 1 every time
```

```
AWS ALB cookie: AWSALB=<hash>
                AWSALBCORS=<hash>
```

#### Method 2: IP Hash (L4 or L7)

```
hash(client_IP) % num_servers = server_index
```

Simple but problematic with NAT (entire office → same server).

### Sticky Session Tradeoffs

```
Without Stickiness:             With Stickiness:
┌────────────────┐              ┌──────────────────────┐
│ Any server can │              │ Server 1 gets ALL     │
│ handle any req │              │ requests from User A  │
│ True stateless │              │ → Can become hotspot  │
└────────────────┘              └──────────────────────┘
    ✅ Even load                  ⚠️ Uneven load
    ✅ Easy failover              ❌ If Server 1 dies,
    ✅ Scale freely                   User A loses session
```

> ⚠️ **Modern Best Practice:** Avoid sticky sessions if possible. Store sessions externally in **Redis** or **Memcached** so any server can handle any request. This enables true stateless, horizontally scalable architecture.

```
With External Session Store:

User A → Server 1 → Read/Write session from Redis
User A → Server 2 → Same session from Redis  ✅
User A → Server 3 → Same session from Redis  ✅

No stickiness needed. Any server, same data.
```

---

## 8. TLS Termination

L7 load balancers handle SSL/TLS so backend servers don't have to.

```
Client ──HTTPS──► [ Load Balancer ] ──HTTP──► Backend Servers
                   (TLS terminated here)       (plaintext internally)
```

### Benefits

- ✅ **Offloads CPU** from backend servers (TLS handshake is expensive)
- ✅ **Centralised certificate management** — update certs in one place
- ✅ **LB can now inspect and route traffic** (since it's decrypted)
- ✅ **Connection pooling** — LB reuses persistent HTTP/2 connections to backends

### mTLS (Mutual TLS) for Backend

```
Client ──HTTPS──► [ LB ] ──mTLS──► Backend Servers
                            ↑
               Backend must present valid cert too
               (Zero Trust / service mesh pattern)
```

---

## 9. High Availability of the Load Balancer Itself

> **The problem:** If the load balancer is a single machine, it's now a single point of failure. Who balances the load balancer?

### Solution: Active-Passive Failover (Heartbeat)

```
              ┌──── VIP: 10.0.0.1 ────┐
              │                        │
       [ Primary LB ]           [ Standby LB ]
       (Active, handles         (Passive, monitors
        all traffic)             primary via heartbeat)
              │
              ↓
          Backends

If Primary fails:
   Standby detects via missed heartbeat
   Standby claims the VIP
   Traffic seamlessly continues
```

**Technology:** Keepalived (VRRP), Pacemaker, AWS handles this automatically.

### Solution: Active-Active (DNS Round Robin)

```
DNS: lb.example.com → [ 10.0.0.1, 10.0.0.2 ]
                              ↓           ↓
                       [ LB Node 1 ] [ LB Node 2 ]
                         /    \          /    \
                       S1      S2      S1      S2
```

Both nodes handle traffic simultaneously. If one goes down, DNS TTL determines how fast clients switch (keep TTL at 30–60s for fast failover).

> **AWS:** ALB/NLB are inherently multi-AZ and managed — you don't worry about LB HA.

---

## 10. Real-World Tools & AWS Mapping

| Tool | Layer | Type | Best For |
|---|---|---|---|
| **AWS ALB** | L7 | Managed | HTTP/HTTPS, microservices, path routing |
| **AWS NLB** | L4 | Managed | TCP/UDP, ultra-low latency, static IPs |
| **AWS CLB** | L4/L7 | Managed (legacy) | Old workloads, avoid for new projects |
| **NGINX** | L7 | Self-hosted | Reverse proxy, simple web LB |
| **HAProxy** | L4/L7 | Self-hosted | High performance, advanced health checks |
| **Envoy** | L7 | Self-hosted | Service mesh, gRPC, microservices |
| **Traefik** | L7 | Self-hosted | Container-native, auto service discovery |
| **Cloudflare LB** | L7 | Cloud | Global, anycast, DDoS-protected |

### AWS ALB Path-Based Routing Example

```
ALB Listener Rule:
  /api/*       → Target Group: api-servers (EC2)
  /auth/*      → Target Group: auth-service (ECS)
  /static/*    → Target Group: cdn-origin (S3)
  default      → Target Group: web-servers (EC2)
```

---

## 11. Real-World Architectures

### Netflix

```
Internet → L4 Regional LBs (AWS NLB)
               ↓
           Zuul / Envoy (L7 Gateway)
           - Routes to 100s of microservices
           - Handles auth, rate limiting, retries
               ↓
           Microservice Pods (EC2/ECS)
```

- Uses **P2C (Power of Two Choices)** algorithm in Zuul
- Servers returning errors enter **probation** — receive reduced traffic until recovery
- **Zone-aware routing:** If Zone A has 90% success rate and Zone B has 70%, traffic shifts to Zone A

### Google

```
Internet → Maglev (L4, anycast VIPs + consistent hashing)
               ↓
           GFE - Google Frontend (L7, TLS termination)
               ↓
           Backend Services
```

- Maglev uses consistent hashing → `< 5% connection remaps` when nodes added/removed
- Failover converges in seconds

### Stripe

```
L7 Load Balancer
  → Routes by API version header
  → v1 requests → v1 backend pool
  → v2 requests → v2 backend pool
  → Enables safe API version rollouts
```

---

## 12. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **No health checks** | Traffic flows to dead servers | Always configure active + passive checks |
| **Sticky sessions without external state** | One server restart = session loss | Use Redis/Memcached for session storage |
| **High DNS TTL on LB** | After failover, clients hit dead endpoint for hours | Keep TTL at 30–60s |
| **No observability at LB** | 502s and connection resets go unnoticed until outage | Log + alert on LB-level errors |
| **L7 LB for raw TCP** | Unnecessary parsing overhead | Use L4 for non-HTTP protocols |
| **Overloaded backends not caught by health checks** | Backend responds but is saturated → cascade failure | Use passive checks + circuit breakers |
| **Sticky sessions on horizontally-scaled services** | Uneven load, defeats the purpose of scaling | Prefer stateless design |

---

## 13. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Load Balancer** | Distributes traffic across a server pool |
| **VIP (Virtual IP)** | Single IP representing the entire server pool |
| **L4 LB** | Routes based on IP/Port, no content inspection |
| **L7 LB** | Routes based on HTTP content, terminates TLS |
| **Health Check** | Periodic probe to detect unhealthy backends |
| **Sticky Session** | Binding a user's requests to the same server |
| **TLS Termination** | LB decrypts HTTPS, communicates HTTP to backend |
| **Least Connections** | Route to server with fewest active connections |
| **Weighted Round Robin** | Round Robin but heavier servers get more traffic |
| **Connection Draining** | Let existing connections finish before removing server |

### When to Use What

| Scenario | Recommendation |
|---|---|
| REST API, web app, microservices | **AWS ALB (L7)** |
| TCP game server, MQTT, raw throughput | **AWS NLB (L4)** |
| Need static IP for allowlisting | **AWS NLB** |
| Path-based routing to different services | **L7 LB** |
| Session state must persist | **External Redis** > Sticky sessions |
| Heterogeneous server pool | **Weighted Round Robin / Weighted Least Connections** |
| Long-lived connections (WebSockets, DB) | **Least Connections** |
| Low latency, uniform requests | **Round Robin** |
| Global traffic distribution | **Global LB + Anycast (Cloudflare, AWS Global Accelerator)** |

### Must-Know Interview Points

- ☑ **L4 is fast but dumb. L7 is smart but slower.** Production uses both.
- ☑ **AWS ALB uses a flow hash** considering protocol, src/dst IP, port, and TCP sequence number.
- ☑ **Sticky sessions = antipattern** for stateless apps. Use shared session store.
- ☑ **DNS TTL matters for failover speed.** Keep it at 30–60s.
- ☑ **Health checks must verify app logic**, not just TCP connectivity.
- ☑ **Connection draining** allows graceful removal — existing connections finish before server is deregistered.
- ☑ **The LB itself can be a SPOF** — solved via Active-Passive failover (VRRP) or Active-Active (DNS).


---

*Sources: AlgoMaster Newsletter, System Overflow, Medium (Codeshbhai, Naveena Chinta), sujeet.pro, oneuptime.com — combined with first-principles system design knowledge.*