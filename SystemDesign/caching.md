# 🗃️ Caching

> **Series:** System Design Notes  
> **Module:** 03 — Caching  
> **Prerequisites:** loadbalancing, Basic DB knowledge (reads/writes), Redis basics

---

## 📌 Table of Contents

1. [What is Caching?](#1-what-is-caching)
2. [Cache Terminology](#2-cache-terminology)
3. [Where Can You Cache? (Cache Layers)](#3-where-can-you-cache-cache-layers)
4. [Caching Patterns (Read)](#4-caching-patterns-read)
5. [Caching Patterns (Write)](#5-caching-patterns-write)
6. [Eviction Policies](#6-eviction-policies)
7. [Cache Invalidation](#7-cache-invalidation)
8. [Cache Problems & Solutions](#8-cache-problems--solutions)
9. [Distributed Caching](#9-distributed-caching)
10. [Redis vs Memcached](#10-redis-vs-memcached)
11. [Real-World Architectures](#11-real-world-architectures)
12. [Common Mistakes](#12-common-mistakes)
13. [Interview Cheatsheet](#13-interview-cheatsheet)

---

## 1. What is Caching?

> **Definition:** Caching is the technique of storing copies of frequently accessed data in a faster, temporary storage layer so future requests for that data can be served more quickly — avoiding the cost of recomputing or re-fetching from a slower source (e.g., a database or external API).

Think of it like a **sticky note on your desk** — instead of searching the filing cabinet every time, you write the key fact on a sticky note and read from that. Fast, local, temporary.

```
WITHOUT CACHE:
  User → App Server → Database → Response
                      (slow: 50–200ms disk I/O)

WITH CACHE:
  User → App Server → Cache (Redis) → Response
                      (fast: < 1ms in-memory)
                      
  Only on cache MISS:
  User → App Server → Cache MISS → Database → Cache → Response
```

**The fundamental trade-off:**

```
┌─────────────────┬────────────────────────────────────┐
│ Gain            │ Cost                               │
├─────────────────┼────────────────────────────────────┤
│ ✅ Speed        │ ⚠️  Stale data (consistency)       │
│ ✅ Less DB load │ ⚠️  Cache invalidation complexity  │
│ ✅ Lower cost   │ ⚠️  Cache failures → DB stampede   │
│ ✅ Scalability  │ ⚠️  Extra infrastructure to manage │
└─────────────────┴────────────────────────────────────┘
```

> **Phil Karlton's famous quote:**  
> *"There are only two hard things in Computer Science: cache invalidation and naming things."*

---

## 2. Cache Terminology

| Term | Definition |
|---|---|
| **Cache Hit** | Requested data IS found in cache → served immediately |
| **Cache Miss** | Requested data is NOT in cache → must fetch from DB |
| **Hit Ratio** | `Cache Hits / (Cache Hits + Cache Misses)` — higher is better |
| **TTL (Time-To-Live)** | Expiry time for a cache entry |
| **Eviction** | Removing data from cache when it's full |
| **Invalidation** | Removing/updating stale cache entries when source data changes |
| **Warm Cache** | Cache is populated and serving hits — "hot" |
| **Cold Cache** | Cache is empty/fresh — all requests are misses (happens on startup) |
| **Thundering Herd** | Many requests simultaneously hit a cold/expired key → all hammer the DB |
| **Hot Key** | A single cache key that receives a disproportionate number of requests |
| **Stale Data** | Cached data that no longer matches the source of truth |
| **Cache Pollution** | Cache fills up with data that's unlikely to be accessed again |

---

## 3. Where Can You Cache? (Cache Layers)

Caching can happen at multiple layers of the stack. Each layer has different scope, speed, and tradeoffs.

```
┌─────────────────────────────────────────────────────────────────┐
│                        REQUEST LIFECYCLE                        │
│                                                                 │
│  User                                                           │
│   ↓                                                             │
│  [ Browser Cache ]         ← Layer 1: Client-side              │
│   ↓ (miss)                                                      │
│  [ CDN Edge Cache ]        ← Layer 2: Network edge             │
│   ↓ (miss)                                                      │
│  [ Load Balancer ]                                              │
│   ↓                                                             │
│  [ App Server ]                                                 │
│   ↓ checks                                                      │
│  [ In-Process Cache ]      ← Layer 3: Local memory (L1)        │
│   ↓ (miss)                                                      │
│  [ External Cache (Redis)] ← Layer 4: Shared distributed cache │
│   ↓ (miss)                                                      │
│  [ Database ]              ← Source of truth                   │
└─────────────────────────────────────────────────────────────────┘
```

### Layer 1: Client-Side (Browser) Cache

```
Browser stores:  CSS, JS, images, HTML (via Cache-Control headers)

HTTP Response Header:
  Cache-Control: max-age=86400      ← cache for 24 hours
  ETag: "abc123"                    ← fingerprint for validation
  Last-Modified: Mon, 01 Jan 2024   ← timestamp for conditional GET
```

**Best for:** Static assets, public content, user-specific non-sensitive data

---

### Layer 2: CDN Cache (Edge Cache)

CDN servers distributed globally cache responses close to users.

```
User in Mumbai → Mumbai CDN PoP → Cache Hit → Response (5ms)
                                ↘ Miss → Origin Server in US → Response (200ms)
```

**Best for:** Static files (images, CSS, JS, fonts), public API responses, HTML pages  
**Covered in detail in:** `04_cdn.md`

---

### Layer 3: In-Process Cache (Local Memory / L1 Cache)

Cache lives inside the application process itself — no network hop.

```java
// Example: Guava Cache (Java)
LoadingCache<String, User> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> db.getUser(key));
```

```
Latency: ~microseconds (no network call)
Scope: Per process only — not shared across app server instances
Size: Limited by app's heap memory
```

| ✅ Pros | ❌ Cons |
|---|---|
| Fastest possible — no network | Not shared across multiple app servers |
| Zero serialization cost | Inconsistency between instances |
| No extra infrastructure | Lost on restart |

**Best for:** Config data, feature flags, tiny hot datasets (e.g., top 100 product IDs), read-only reference data

> ⚠️ **Gotcha:** With 10 app servers, you have 10 independent in-process caches. An update on one server doesn't propagate to the others. Use only for data that rarely or never changes.

---

### Layer 4: External Distributed Cache (Redis / Memcached)

A dedicated caching service shared by all app servers.

```
App Server 1 ─┐
App Server 2 ─┤──► [ Redis Cluster ] ──► (miss) → Database
App Server 3 ─┘
```

```
Latency: ~0.1–1ms (in-datacenter network)
Scope: Shared across ALL app server instances ✅
Size: GBs to TBs (distributed cluster)
Persistence: Optional (Redis supports AOF/RDB snapshots)
```

**This is the default answer in system design interviews.** When someone says "add a cache," they almost always mean Redis here.

---

## 4. Caching Patterns (Read)

### 4.1 Cache-Aside (Lazy Loading) ⭐ Default Pattern

> **The application manages the cache.** It checks cache first, then DB on miss, then populates cache.

```
READ:
  1. App checks cache for key
  2a. Cache HIT  → return data ✅
  2b. Cache MISS →
      3. App queries Database
      4. App writes result to Cache (with TTL)
      5. Return data to user

WRITE:
  App updates Database
  App DELETES (or updates) the cache key  ← invalidation
```

```
┌─────────────────────────────────────────────────────┐
│                  CACHE-ASIDE FLOW                   │
│                                                     │
│   Request                                           │
│      ↓                                              │
│   ┌──────────┐  HIT   ┌───────────┐                │
│   │  Cache   │───────►│  Return   │                │
│   │ (Redis)  │        │  to User  │                │
│   └──────────┘        └───────────┘                │
│       │ MISS                                        │
│       ↓                                             │
│   ┌──────────┐        ┌───────────┐                │
│   │ Database │───────►│  Write to │                │
│   │          │  data  │   Cache   │                │
│   └──────────┘        └─────┬─────┘                │
│                             │                       │
│                             ▼                       │
│                         Return to User              │
└─────────────────────────────────────────────────────┘
```

```javascript
// Node.js / Redis example
async function getUser(userId) {
  const key = `user:${userId}`;

  // 1. Check cache
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);  // Cache HIT

  // 2. Cache MISS → query DB
  const user = await db.query('SELECT * FROM users WHERE id = ?', [userId]);

  // 3. Populate cache with TTL
  await redis.setex(key, 3600, JSON.stringify(user));  // 1 hour TTL

  return user;
}

async function updateUser(userId, data) {
  await db.query('UPDATE users SET ... WHERE id = ?', [userId]);
  await redis.del(`user:${userId}`);  // Invalidate cache
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| **Most common, most resilient** | Cold start: first request always a miss |
| Cache only holds what's actually used | Race condition: two simultaneous misses can both hit DB |
| If Redis dies → fall back to DB gracefully | Stale data window between DB update and cache invalidation |
| Application controls what goes in cache | Extra code to manage cache in app |

**Best for:** Read-heavy workloads, general-purpose caching, when you want resilience if cache goes down

---

### 4.2 Read-Through

> **The cache manages fetching from DB.** On a miss, the cache itself calls the DB, populates, and returns data. App only talks to the cache.

```
App → Cache → (miss) → Cache calls DB → Cache stores → Returns to App
                              ↑
                    Cache layer handles this,
                    not the application code
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simpler app code (no DB logic in app) | Cache becomes a critical dependency |
| Cache handles its own population | If Redis dies → app can't read data at all |
| Good for CDN patterns | Less common in app-level caching with Redis |

**Best for:** CDN (this is literally how CDNs work), read-heavy with cache as the single read layer

> ℹ️ Redis does not natively support read-through. You need a library (Redisson in Java, Spring Cache) or proxy layer. In interviews, stick to Cache-Aside unless discussing CDNs.

---

## 5. Caching Patterns (Write)

### 5.1 Write-Through

> **Every write goes to cache AND database synchronously.** Cache is always up-to-date.

```
WRITE:
  App → Write to Cache
      → Write to Database  (same transaction / sequential)
      → Acknowledge to user only after BOTH succeed

READ:
  App → Cache → always up-to-date → Return
```

```
┌────────────────────────────────────────────────────┐
│               WRITE-THROUGH FLOW                   │
│                                                     │
│   Write Request                                     │
│        ↓                                            │
│   ┌─────────┐   1. Write   ┌──────────┐            │
│   │   App   │─────────────►│  Cache   │            │
│   │ Server  │              │ (Redis)  │            │
│   └─────────┘   2. Write   └──────────┘            │
│        │──────────────────►┌──────────┐            │
│        │                   │ Database │            │
│        │◄──────────────────└──────────┘            │
│        │   3. Ack (after BOTH succeed)              │
└────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Cache always consistent with DB | **Write latency doubles** (wait for both) |
| No stale reads | Cache can fill up with data never read again (cache pollution) |
| Simple reasoning about consistency | Dual-write problem: if one fails, inconsistency |

**Best for:** Financial data, inventory systems, anything where consistency > write speed

---

### 5.2 Write-Behind (Write-Back)

> **Write to cache first, acknowledge immediately, flush to DB asynchronously.** Optimise for write speed.

```
WRITE:
  App → Write to Cache → Acknowledge to user immediately ✅
         ↓ (async, later)
         Background job flushes cache → Database

READ:
  App → Cache (has the latest write) → Return
```

```
┌────────────────────────────────────────────────────────┐
│               WRITE-BEHIND FLOW                        │
│                                                         │
│  Write Request → Cache → ✅ Immediate ACK to user      │
│                     │                                   │
│                     │  (async, every N seconds)         │
│                     ▼                                   │
│               Background writer                         │
│                     │                                   │
│                     ▼                                   │
│                 Database                                 │
│                                                         │
│  ⚠️  If Redis crashes before flush → DATA LOSS          │
└────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| **Lowest write latency** | **Risk of data loss** if cache fails before flush |
| High write throughput (batch DB writes) | Complex failure recovery |
| Absorbs bursty writes | Requires Redis persistence (AOF/RDB) to mitigate |

**Best for:** IoT data ingestion, analytics counters, high-frequency logs, like/view counters

---

### 5.3 Write-Around

> **Writes go directly to the database, bypassing the cache.** Cache only gets populated on subsequent reads.

```
WRITE:
  App → Write directly to Database (cache is NOT updated)

READ:
  Cache Miss → Fetch from DB → Populate cache (Cache-Aside)
  Cache Hit  → Serve from cache
```

| ✅ Pros | ❌ Cons |
|---|---|
| Prevents cache pollution from write-heavy data | First read after a write is always a cache miss |
| Simple for write-once-read-rarely patterns | Slightly higher read latency for recently written data |

**Best for:** Log ingestion, time-series data, data written once and rarely read

---

### Write Pattern Summary

```
┌─────────────────────────────────────────────────────────────────────┐
│                  WRITE PATTERN DECISION TREE                        │
│                                                                     │
│  How important is write consistency?                                │
│  ├── Critical (finance, inventory)    → Write-Through               │
│  └── Eventual is OK                                                 │
│      ├── Write-heavy, speed matters  → Write-Behind (accept risk)   │
│      └── Writes rarely re-read       → Write-Around                 │
└─────────────────────────────────────────────────────────────────────┘
```

| Pattern | Write Latency | Read Latency | Data Safety | Consistency |
|---|---|---|---|---|
| **Cache-Aside** | Normal (DB only) | Slow on miss | ✅ DB always safe | Eventual |
| **Write-Through** | High (both writes) | Fast | ✅ Safe | Strong |
| **Write-Behind** | Very Low | Fast | ⚠️ Risk of loss | Eventual |
| **Write-Around** | Normal (DB only) | Slow on first read | ✅ DB always safe | Eventual |

---

## 6. Eviction Policies

> When the cache is **full**, something must be removed to make room. Eviction policies decide *what gets removed*.

> ⚠️ **Eviction ≠ Invalidation**  
> **Eviction** = remove to free up space (capacity problem)  
> **Invalidation** = remove because data is stale (correctness problem)

---

### LRU — Least Recently Used ⭐ Most Common

> Remove the item that hasn't been accessed for the longest time.  
> Assumption: *recently used items are likely to be used again soon (temporal locality)*

```
Cache state (most recent → least recent):
  [A, C, D, B, E]   ← A is most recently used, E is least

Cache FULL. New item F arrives:
  → Evict E (least recently used)
  → Insert F at front
  [F, A, C, D, B]
```

**Implementation:** HashMap (O(1) lookup) + Doubly Linked List (O(1) reorder)

| ✅ Pros | ❌ Cons |
|---|---|
| Works well for most web workloads | Fails for cyclic access patterns |
| Self-adjusting to recent access patterns | Overhead of tracking access order |

**Best for:** Web applications, databases, OS page cache, general-purpose

---

### LFU — Least Frequently Used

> Remove the item that has been accessed the fewest number of times total.  
> Assumption: *popular items should stay; unpopular items should go*

```
Frequency counters:
  A: 50 hits
  B:  3 hits  ← evict this
  C: 21 hits
  D:  7 hits

Cache FULL → Evict B (lowest frequency)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Keeps genuinely popular items | Expensive: must track frequency per key |
| Better than LRU for stable popularity patterns | **Cold start problem:** new items start at 0 → evicted too quickly |
| Great for CDNs, viral content | Old popular items "stuck" even if no longer trending |

**Best for:** CDNs, trending content, search result caches

---

### FIFO — First In, First Out

> Remove the oldest item regardless of how recently or frequently it was accessed.

```
Queue: [oldest: A, B, C, D :newest]

Cache FULL, add E:
  → Evict A (oldest)
  → [B, C, D, E]
```

| ✅ Pros | ❌ Cons |
|---|---|
| Very simple, no tracking needed | Ignores access patterns entirely |
| Predictable behavior | Evicts frequently-used items if they're old |

**Best for:** Queue-based systems, when simplicity > optimal hit rate

---

### MRU — Most Recently Used

> Evict the **most recently** accessed item.  
> This is the *opposite* of LRU — sounds counterintuitive, but useful for cyclic patterns.

```
Cyclic access pattern: A → B → C → A → B → C → (repeat)
With LRU: A is "newest" → kept. But A is immediately cycled out anyway.
With MRU: Evict A (just used) → keeps B, C which haven't been cycled yet.
```

**Best for:** Streaming media (last-played song unlikely to be re-requested), batch scans

---

### TTL-Based Expiration (Time-To-Live)

Not strictly an eviction *policy* — it's a **correctness mechanism** that also frees memory.

```
SET user:123 <data> EX 3600    ← key auto-deletes after 3600 seconds (1 hour)
```

> **TTL is a safety net, not a substitute for explicit invalidation.**  
> Use TTL on everything. It guarantees an upper bound on how stale data can get.

**TTL Jitter — prevent stampedes:**

```python
import random

# Without jitter: all keys created at the same time expire at the same time
# → mass cache miss at the same second → thundering herd
ttl = 3600

# With jitter: spread expirations randomly across a window
ttl_with_jitter = 3600 + random.randint(-300, 300)  # ±5 minutes
redis.setex(key, ttl_with_jitter, value)
```

---

### Eviction Policy Comparison

| Policy | Evicts | Overhead | Best For |
|---|---|---|---|
| **LRU** | Least recently used | Low (linked list) | General-purpose web apps |
| **LFU** | Least frequently used | Medium (counters) | CDN, stable popularity |
| **FIFO** | Oldest inserted | Minimal | Simple queues |
| **MRU** | Most recently used | Low | Cyclic scans, streaming |
| **TTL** | Expired by time | Minimal | All caches (use always) |
| **Random** | Random entry | None | Real-time, unpredictable |

### Redis Eviction Policies (Config)

```
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru    # or: volatile-lru, allkeys-lfu, etc.
```

| Redis Policy | Meaning |
|---|---|
| `noeviction` | Return error when full (default — usually wrong for caches) |
| `allkeys-lru` | LRU across all keys |
| `volatile-lru` | LRU only on keys with TTL set |
| `allkeys-lfu` | LFU across all keys |
| `volatile-lfu` | LFU only on keys with TTL set |
| `allkeys-random` | Random eviction |
| `volatile-ttl` | Evict key closest to expiry |

> **Recommended default for a cache:** `allkeys-lru`  
> **For CDN-like access patterns:** `allkeys-lfu`

---

## 7. Cache Invalidation

> **Definition:** Cache invalidation is the process of removing or updating cache entries when the underlying source data changes — to prevent stale data from being served.

This is the "two hard things" problem. Let's break down every approach.

---

### Strategy 1: TTL (Time-To-Live) — Passive Invalidation

```
Every key has an expiry. After TTL elapses → key is gone → next read hits DB.
```

```
SET product:42 <data> EX 300   ← expires in 5 minutes
```

```
Timeline:
  t=0    Write product:42 to cache
  t=100  User reads → cache hit ✅
  t=200  Product price updated in DB (cache still has old price ⚠️)
  t=300  TTL expires → cache miss → reads fresh data from DB ✅
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple, zero extra code | Data can be stale for up to TTL duration |
| Works as a safety net for missed invalidations | TTL too long → stale data. TTL too short → too many DB hits |

**Rule:** *Always set a TTL. It's the minimum safety net.*

---

### Strategy 2: Event-Driven Invalidation (Write Invalidation)

> When data is updated in the DB, explicitly delete or update the cache entry.

```
User updates profile:
  1. UPDATE users SET name='...' WHERE id=123  (DB write)
  2. DEL user:123                               (cache invalidation)
  
Next read:
  3. Cache miss → fetch fresh from DB → re-populate cache
```

```
┌─────────────────────────────────────────────────────┐
│            EVENT-DRIVEN INVALIDATION                │
│                                                     │
│  DB Write ──► App deletes cache key ──► Next read   │
│               (synchronous)               hits DB   │
│                                        (fresh data) │
└─────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Near-instant consistency | App must remember to invalidate on every write path |
| Precise — only affected keys removed | Race conditions possible |

---

### Strategy 3: Write-Through (Active Refresh)

Instead of deleting the key, update the cache simultaneously with the DB write.

```
DB Write + Cache Update (atomic or sequential):
  SET user:123 <new_data> EX 3600
  UPDATE users SET ... WHERE id=123
```

Avoids the cold miss that delete-on-write causes.

---

### Strategy 4: Tag-Based Invalidation

Group related cache entries under a tag. Invalidate all entries with a given tag at once.

```
Cache entries:
  product:42        → tagged: [products, category:electronics]
  product:42:reviews → tagged: [products, reviews]
  category:electronics:list → tagged: [category:electronics]

If product 42's price changes:
  INVALIDATE tag: products
  → Removes: product:42, product:42:reviews
  → Keeps: unrelated entries
```

**Best for:** CMS, e-commerce product catalogs, multi-tenant applications

---

### Strategy 5: Pub/Sub Invalidation (Distributed Caches)

Used when you have multiple cache nodes or multiple app servers with in-process caches.

```
Event: DB updated
  → App publishes to Redis channel: "invalidate:user:123"
  → All subscribed app servers receive message
  → Each server deletes its local in-process cache entry
```

```
App Server 1 ─┐                    ┌─ deletes local cache
App Server 2 ─┤─ publish ──► Redis ─┤─ deletes local cache
App Server 3 ─┘  channel           └─ deletes local cache
```

**Used by:** Facebook's Memcache, multi-region invalidation systems

---

## 8. Cache Problems & Solutions

### Problem 1: Cache Stampede / Thundering Herd ⚡

**What:** A popular cache key expires. Thousands of requests simultaneously get a cache miss and all hammer the database at once.

```
t=0       key "trending_feed" TTL expires
t=0.001   1000 concurrent requests → all get cache MISS
t=0.001   All 1000 requests query the DB simultaneously → DB overload → outage
```

**Solutions:**

#### Solution A: Mutex / Distributed Lock

```
On cache miss:
  Try to acquire lock for this key
  ├── Lock acquired → query DB, populate cache, release lock
  └── Lock NOT acquired → wait briefly → retry cache lookup
                                         (another request is repopulating it)
```

```python
lock_key = f"lock:{cache_key}"
if redis.set(lock_key, "1", nx=True, ex=10):   # nx = only if not exists
    try:
        data = db.query(...)
        redis.setex(cache_key, ttl, data)
        return data
    finally:
        redis.delete(lock_key)
else:
    time.sleep(0.1)
    return redis.get(cache_key)  # retry from cache
```

#### Solution B: TTL Jitter (Prevent Mass Expiry)

```python
# Add random offset so keys don't all expire at the same second
ttl = 3600 + random.randint(-300, 300)
redis.setex(key, ttl, value)
```

#### Solution C: Probabilistic Early Refresh (XFetch Algorithm)

Refresh the cache *before* it expires, with increasing probability as expiry approaches.

```python
remaining_ttl = redis.ttl(key)
if remaining_ttl < (total_ttl * 0.2):      # in last 20% of lifetime
    if random.random() < 0.3:               # 30% chance to refresh early
        data = db.query(...)
        redis.setex(key, ttl, data)
```

---

### Problem 2: Cache Penetration 🎯

**What:** Requests for data that **doesn't exist in the DB at all** (e.g., invalid user IDs). Every request is a cache miss → hits DB every time → DB abuse.

```
Attacker sends: GET /user/99999999 (doesn't exist)
  → Cache miss (nothing to cache)
  → DB query → nothing found
  → No caching → repeat forever → DB DoS
```

**Solutions:**

#### Solution A: Cache Null / Negative Caching

```python
user = db.query(user_id)
if user is None:
    redis.setex(f"user:{user_id}", 300, "NULL")  # cache the miss itself (short TTL)
    return None
```

#### Solution B: Bloom Filter

A probabilistic data structure that can definitively say "this key DOES NOT exist" without querying the DB.

```
Before any DB query:
  bloomFilter.contains(user_id)?
  ├── NO  → return 404 immediately (never existed)
  └── YES → proceed with cache-aside lookup
```

```
Memory: ~10 bits per element (~1.2 MB for 1 million items)
False positives: possible (might let a miss through)
False negatives: NEVER (if it says "no", it's definitely not there)
```

---

### Problem 3: Cache Avalanche ❄️

**What:** Large number of cache entries expire at nearly the same time (or the entire cache goes down), causing a massive surge of DB requests.

Different from stampede: stampede = one key, avalanche = many keys or full cache failure.

```
t=0   You deploy and warm cache. All keys set with TTL=3600.
t=3600  ALL keys expire simultaneously → 100% cache miss rate → DB overload
```

**Solutions:**

- **TTL Jitter** (most important): spread expirations over a window
- **Cache warming**: pre-populate cache before expiry using background jobs
- **Circuit Breaker**: if cache is down, rate-limit DB queries to prevent overload
- **Tiered cache**: L1 in-process cache as fallback

---

### Problem 4: Hot Key Problem 🔥

**What:** A single cache key is accessed by millions of requests per second. A single Redis node maxes out its CPU/network handling that one key.

```
"score:leaderboard" → 500,000 reads/sec → one Redis shard → bottleneck
```

**Solutions:**

#### Local Replica Caching

```
On each app server:
  L1 (in-process): cache hot key locally for 1–5 seconds
  L2 (Redis): falls back to Redis if L1 miss

Result: 100x reduction in Redis requests for that key
```

#### Key Replication / Sharding the Hot Key

```
Instead of: score:leaderboard → one Redis node
Use:        score:leaderboard:shard:0
            score:leaderboard:shard:1   → distribute across N Redis nodes
            score:leaderboard:shard:2
            ...
App randomly picks a shard to read from.
```

---

### Cache Problem Summary

| Problem | Cause | Key Solution |
|---|---|---|
| **Cache Stampede** | Popular key expires, many simultaneous misses | Mutex lock + TTL jitter |
| **Cache Penetration** | Querying keys that never exist | Bloom filter + null caching |
| **Cache Avalanche** | Mass expiry or cache node failure | Jitter + circuit breaker |
| **Hot Key** | One key hit millions of times/sec | Local L1 cache + key sharding |
| **Stale Data** | Cache not invalidated after DB write | Write-through + short TTL |
| **Memory Leak** | No TTL set, cache grows unbounded | Always set TTL, use eviction policy |

---

## 9. Distributed Caching

When a single Redis node isn't enough, you scale horizontally.

### Consistent Hashing

Maps keys to nodes so that when a node is added/removed, only ~1/N of keys need to be remapped (instead of all of them).

```
NAIVE hash (modulo):
  Key "user:42" → hash(42) % 3 = 0 → Node 0
  Add a 4th node → hash(42) % 4 = 2 → Node 2  (remapped! stale data if not migrated)

CONSISTENT HASH (ring):
  Nodes placed on a 0–2^32 ring
  Key maps to nearest node clockwise
  Add Node D: only the keys between Node C and Node D move to D
              ~25% remapped vs 100% with modulo
```

### Redis Cluster (Built-in Sharding)

```
Redis Cluster: 16,384 hash slots distributed across N master nodes

Key "user:42" → CRC16("user:42") % 16384 = slot 7643 → Node 2

Auto-failover:
  Each master has 1+ replicas
  If Master 2 dies → Replica 2 promoted → zero-downtime
```

```
┌──────────────────────────────────────────────┐
│              REDIS CLUSTER                   │
│                                              │
│  Master 1 (slots 0–5461)                     │
│    └── Replica 1                             │
│  Master 2 (slots 5462–10922)                 │
│    └── Replica 2                             │
│  Master 3 (slots 10923–16383)                │
│    └── Replica 3                             │
└──────────────────────────────────────────────┘
```

---

## 10. Redis vs Memcached

| Feature | Redis | Memcached |
|---|---|---|
| **Data Structures** | Strings, Lists, Sets, Sorted Sets, Hashes, Streams | Strings only |
| **Persistence** | ✅ AOF + RDB snapshots | ❌ In-memory only |
| **Pub/Sub** | ✅ Built-in | ❌ No |
| **Clustering** | ✅ Redis Cluster | ✅ Client-side sharding |
| **Lua Scripting** | ✅ | ❌ |
| **Replication** | ✅ Master-Replica | ❌ No built-in |
| **TTL** | ✅ Per-key TTL | ✅ Per-key TTL |
| **Multi-threading** | ⚠️ Single-threaded (I/O multi-threaded in v6+) | ✅ Multi-threaded |
| **Memory Overhead** | Higher (richer data structures) | Lower |
| **Use Cases** | Sessions, queues, pub/sub, leaderboards, rate limiting | Simple high-throughput KV cache |

> **Default choice:** **Redis**. Unless you have a specific high-throughput, simple KV use case where Memcached's multi-threaded model gives a measurable advantage.

**Redis Data Structure Use Cases:**

| Structure | Example Use Case |
|---|---|
| `String` | Session tokens, simple object cache |
| `Hash` | User profiles (`HGET user:42 name`) |
| `Sorted Set` | Leaderboards (`ZADD scores 1500 user:42`) |
| `List` | Activity feed, queue (LPUSH/RPOP) |
| `Set` | Unique visitors, tags |
| `Bitmap` | Daily active users (set bit for each userId) |
| `HyperLogLog` | Approximate unique count (very memory efficient) |
| `Stream` | Event log, message queue |

---

## 11. Real-World Architectures

### Twitter / X — Timeline Cache

```
Problem: Computing a user's timeline = fan-out of tweets from 1000+ followees
Too slow to compute on every request.

Solution: Precomputed timeline cache

Tweet posted by User A:
  → Fanout service identifies User A's followers
  → Push tweet ID to each follower's timeline cache (Redis List)
  → Each user's cache: redis.lpush("timeline:userB", tweet_id)

User B refreshes timeline:
  → LRANGE timeline:userB 0 99  (top 100 tweet IDs, in-memory, microseconds)
  → Fetch tweet content from separate tweet cache
```

### Facebook — Memcache at Scale

```
Problem: Billions of social graph queries per day

Solution: Multilayer cache
  L1: Per-datacenter Memcache cluster (hundreds of nodes)
  L2: Cross-datacenter cold cache
  
Invalidation:
  DB write → publishes invalidation message
  Cache servers subscribed to invalidation stream
  → Delete affected keys across all clusters

Scale: ~1 billion requests/sec to Memcache across all DCs
```

### Uber — Geolocation Cache

```
Problem: "What drivers are near me?" — expensive geospatial query every 4 seconds

Solution:
  Redis Geo commands (GEOADD, GEORADIUS)
  Driver app pings location → Redis GEOADD with TTL
  User requests nearby drivers → Redis GEORADIUS (in-memory, microseconds)
```

### Rate Limiting with Redis

```python
# Sliding window rate limiter using Redis
def is_rate_limited(user_id, limit=100, window_seconds=60):
    key = f"rate:{user_id}"
    now = time.time()
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window_seconds)  # remove old
    pipe.zadd(key, {now: now})                           # add current
    pipe.zcard(key)                                       # count in window
    pipe.expire(key, window_seconds)
    _, _, count, _ = pipe.execute()
    return count > limit
```

---

## 12. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **No TTL on cache keys** | Cache grows unbounded; stale data forever | Always set TTL, use `allkeys-lru` eviction |
| **Caching everything** | Cache pollution; evicts useful data | Cache only hot, expensive-to-compute data |
| **No invalidation on writes** | Stale data served until TTL expires | Delete/update cache key on every DB write |
| **Relying solely on cache (no DB fallback)** | Cache failure = complete outage | Cache-aside gracefully falls back to DB |
| **Not accounting for thundering herd** | Popular key expires → DB spike | Add mutex lock + TTL jitter |
| **Ignoring cold start** | Deployment → all cache misses → DB spike | Pre-warm cache after deploy |
| **Storing large objects in cache** | Wastes memory, slows serialization | Cache identifiers/IDs, not entire object graphs |
| **Single Redis node (no replication)** | SPOF — Redis fails → all traffic hits DB | Redis Sentinel (HA) or Redis Cluster |
| **Cache key collisions** | Two features using same key format | Namespace keys: `user:profile:123`, `user:settings:123` |
| **Not monitoring hit ratio** | Cache exists but isn't helping — you'd never know | Alert on hit ratio < 80% |

---

## 13. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Cache** | Fast temporary store to avoid slow repeated lookups |
| **Cache-Aside** | App checks cache, falls back to DB on miss, populates cache |
| **Write-Through** | Write to cache + DB simultaneously (strong consistency) |
| **Write-Behind** | Write to cache, async flush to DB (low latency, risk of loss) |
| **Write-Around** | Skip cache on write, only populate on read |
| **LRU** | Evict least recently used item |
| **LFU** | Evict least frequently used item |
| **TTL** | Auto-expire key after N seconds |
| **Thundering Herd** | Mass simultaneous cache misses on one key |
| **Bloom Filter** | Probabilistic check: "does this key exist?" |
| **Negative Caching** | Cache null results to prevent repeated DB misses |
| **Hot Key** | One cache key getting disproportionate traffic |

### When to Use What (Scenarios)

| Scenario | Recommendation |
|---|---|
| General web app caching | **Cache-Aside + LRU + TTL** |
| Financial data, inventory | **Write-Through** (consistency > speed) |
| IoT, analytics, counters | **Write-Behind** (throughput > safety) |
| Leaderboards | **Redis Sorted Set** |
| Session storage | **Redis String** with sliding TTL |
| Rate limiting | **Redis sliding window** (ZADD) |
| Invalid ID abuse (cache penetration) | **Bloom Filter** |
| Thundering herd on popular key | **Mutex lock + TTL jitter** |
| Stable popular content (CDN-like) | **LFU eviction** |
| Multi-region / multi-datacenter | **Pub/Sub invalidation** |
| Read-heavy, DB is bottleneck | **Cache-Aside + Redis** |
| Cache layer for microservices | **Sidecar pattern (Envoy + Redis)** |

### The Interview Answer Template

When caching comes up in a system design interview:

```
1. WHAT to cache:
   "I'll cache [user profiles / product data / search results] 
    because these are read-heavy and expensive to compute."

2. WHERE to cache:
   "External cache with Redis — shared across all app servers."

3. WHICH pattern:
   "Cache-aside for reads — it's resilient, and if Redis goes down, 
    we gracefully fall back to the DB."

4. WRITE pattern:
   "On writes, we'll invalidate the cache key immediately so the 
    next read gets fresh data."

5. EVICTION + TTL:
   "LRU eviction with a 1-hour TTL and ±5 minute jitter to prevent 
    stampedes."

6. FAILURE scenario:
   "If Redis goes down, all requests fall back to the DB. We'll add 
    a circuit breaker so the DB doesn't get overwhelmed."
```

### Must-Know Interview Points

- ☑ **Cache-Aside is the default.** Know it cold. Start here in interviews.
- ☑ **Always set a TTL** — it's the minimum safety net against stale data.
- ☑ **Thundering herd** = popular key expires → DB stampede. Fix: mutex lock + jitter.
- ☑ **Cache penetration** = querying non-existent keys. Fix: bloom filter + null caching.
- ☑ **Eviction ≠ Invalidation.** Know the difference.
- ☑ **Redis is not a database.** It's a cache. DB is still the source of truth.
- ☑ **Hit ratio** is your health metric. Below 80% → something is wrong.
- ☑ **Write-Through** = consistency. **Write-Behind** = performance. Know the tradeoff.
- ☑ **Hot keys** — L1 local cache per app server as a mitigation layer.
- ☑ **Jitter on TTL** is a simple but critical production practice. Always mention it.

---

*Sources: AlgoMaster Newsletter, HelloInterview, AWS Caching Whitepaper, System Design Handbook, DesignGurus, C# Corner, DEV Community — combined with first-principles system design knowledge.*