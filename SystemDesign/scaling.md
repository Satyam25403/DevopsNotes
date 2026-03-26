# System Design: Scalability

Comprehensive guide to scaling systems from single server to millions of users with real-world patterns and trade-offs.

## Table of Contents
- [What is Scalability](#what-is-scalability)
- [Vertical vs Horizontal Scaling](#vertical-vs-horizontal-scaling)
- [Single Server to Million Users](#single-server-to-million-users)
- [Database Scaling Patterns](#database-scaling-patterns)
- [Scaling Challenges](#scaling-challenges)
- [Real-World Examples](#real-world-examples)

---

## What is Scalability?

### Definition

**Scalability** is a system's ability to handle increased load by adding resources while maintaining performance, reliability, and cost efficiency.

**Key Question:**
> "Can your system serve 10x more users tomorrow without breaking?"

### Why Scalability Matters

**Business Impact:**
```
❌ Non-Scalable System:
- Black Friday traffic spike → Site crashes
- Lost revenue: $1M+ per hour
- Customer trust damaged
- Competitors gain market share

✅ Scalable System:
- Traffic increases 50x → System handles it
- Revenue opportunity captured
- Customer satisfaction high
- Business grows confidently
```

**Real-World Scenario:**
```
Startup Journey:

Month 1: 100 users
- Single server works fine
- Response time: 50ms
- Cost: $50/month

Month 6: 10,000 users
- Same server struggling
- Response time: 2000ms
- Users complaining

Month 12: 100,000 users
- Must scale or die
- Redesign required
- Lost customers already
```

### Scalability vs Performance

```
┌────────────────────┬──────────────────┬──────────────────┐
│ Aspect             │ Performance      │ Scalability      │
├────────────────────┼──────────────────┼──────────────────┤
│ Definition         │ How fast         │ Handle more load │
│ Focus              │ Single request   │ Many requests    │
│ Example            │ Query in 10ms    │ 1M queries/sec   │
│ Solution           │ Optimize code    │ Add resources    │
│ Metrics            │ Latency, speed   │ Throughput, QPS  │
└────────────────────┴──────────────────┴──────────────────┘

Example:
Performance: Optimize database query from 100ms to 10ms
Scalability: Handle 100 concurrent queries without degradation
```

### Measuring Scalability

**Key Metrics:**

**1. Throughput**
```
Definition: Requests handled per second

Example:
System A: 1,000 requests/sec
System B: 10,000 requests/sec
→ System B has 10x better throughput

Measurement:
throughput = successful_requests / time_period
```

**2. Latency**
```
Definition: Time to complete single request

Metrics:
- P50 (median): 50% of requests faster than this
- P95: 95% of requests faster than this
- P99: 99% of requests faster than this

Example:
P50: 100ms  ← Half of users see this
P95: 500ms  ← 5% see worse than this
P99: 2000ms ← 1% see terrible experience

Why P99 matters:
If 1% of users have bad experience
At 1M users = 10,000 unhappy users!
```

**3. Availability**
```
Definition: % of time system is operational

Formula:
availability = (uptime / (uptime + downtime)) × 100

Standard levels:
99% ("two nines")    = 7.2 hours downtime/month
99.9% ("three nines") = 43 minutes downtime/month
99.99% ("four nines") = 4.3 minutes downtime/month
99.999% ("five nines") = 26 seconds downtime/month

Cost increases exponentially with each nine!
```

---

## Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)

**Definition:** Add more power to existing machine

**Visual:**
```
Before:
┌─────────────┐
│  Server     │
│  4 CPU      │
│  8 GB RAM   │
│  100 GB SSD │
└─────────────┘
Handles: 1,000 req/sec

After Vertical Scaling:
┌─────────────┐
│  Server     │
│  16 CPU     │ ← Upgraded
│  64 GB RAM  │ ← Upgraded
│  1 TB SSD   │ ← Upgraded
└─────────────┘
Handles: 5,000 req/sec
```

**Implementation:**
```bash
# AWS Example
# Current: t3.medium (2 vCPU, 4GB RAM)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Change instance type
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --instance-type t3.2xlarge  # 8 vCPU, 32GB RAM

aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

**Advantages:**
```
✅ Simple to implement
   - Stop server
   - Upgrade hardware
   - Start server
   - No code changes

✅ No architecture changes
   - Same application code
   - Same database schema
   - Same deployment process

✅ Maintains data locality
   - Everything on one machine
   - No network latency
   - ACID transactions easy
```

**Disadvantages:**
```
❌ Hardware limits
   - Maximum CPU: ~128 cores
   - Maximum RAM: ~4 TB
   - Can't scale infinitely

❌ Single point of failure
   - Server dies = entire system down
   - No redundancy
   - High risk

❌ Downtime required
   - Must stop server to upgrade
   - Minutes to hours offline
   - Users can't access system

❌ Expensive
   - Large servers cost exponentially more
   - Example:
     4 CPU server: $100/month
     64 CPU server: $2000/month (20x cost for 16x power)
```

**When to Use:**
```
✅ Good for:
- Legacy applications (can't distribute)
- Databases requiring ACID (PostgreSQL, MySQL)
- Quick fixes (temporary solution)
- Small to medium scale (< 10,000 users)

❌ Avoid for:
- Massive scale (millions of users)
- High availability requirements (99.99%+)
- Rapid growth (unpredictable traffic)
```

### Horizontal Scaling (Scale Out)

**Definition:** Add more machines

**Visual:**
```
Before:
┌─────────────┐
│  Server 1   │
│  4 CPU      │
│  8 GB RAM   │
└─────────────┘
Handles: 1,000 req/sec

After Horizontal Scaling:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Server 1   │  │  Server 2   │  │  Server 3   │
│  4 CPU      │  │  4 CPU      │  │  4 CPU      │
│  8 GB RAM   │  │  8 GB RAM   │  │  8 GB RAM   │
└─────────────┘  └─────────────┘  └─────────────┘
Total Capacity: 3,000 req/sec
```

**Implementation:**
```
With Load Balancer:

        ┌──────────────────┐
        │  Load Balancer   │
        └────────┬─────────┘
                 │
     ┌───────────┼───────────┐
     ↓           ↓           ↓
┌─────────┐ ┌─────────┐ ┌─────────┐
│Server 1 │ │Server 2 │ │Server 3 │
└─────────┘ └─────────┘ └─────────┘

Traffic distributed across all servers
```

**Advantages:**
```
✅ Unlimited scaling
   - Add as many servers as needed
   - No hardware ceiling
   - Linear scaling possible

✅ High availability
   - Server 1 dies? Server 2 & 3 continue
   - Redundancy built-in
   - Fault tolerant

✅ No downtime
   - Add servers while system running
   - Rolling updates possible
   - Zero-downtime deployments

✅ Cost effective
   - Use commodity hardware
   - Pay for what you need
   - Auto-scaling reduces costs
```

**Disadvantages:**
```
❌ Complex architecture
   - Need load balancer
   - Session management
   - Data consistency challenges

❌ Data distribution
   - Shared state is hard
   - Database scaling complex
   - Caching strategies needed

❌ Network overhead
   - Servers communicate over network
   - Latency introduced
   - Increased complexity

❌ Operational complexity
   - More servers to manage
   - Monitoring multiple instances
   - Deployment complexity
```

**When to Use:**
```
✅ Good for:
- Web applications (stateless)
- Microservices architecture
- Cloud-native applications
- High availability requirements
- Unpredictable growth

❌ Challenging for:
- Stateful applications
- Legacy monoliths
- Applications requiring strong consistency
```

### Comparison Table

```
┌─────────────────┬──────────────────┬──────────────────┐
│ Aspect          │ Vertical Scaling │ Horizontal       │
├─────────────────┼──────────────────┼──────────────────┤
│ Cost            │ Expensive        │ Cost-effective   │
│ Limit           │ Hardware limit   │ Unlimited        │
│ Complexity      │ Simple           │ Complex          │
│ Downtime        │ Yes              │ No               │
│ Availability    │ Single point     │ Highly available │
│ Scaling speed   │ Hours            │ Minutes/seconds  │
│ Data locality   │ Local            │ Distributed      │
│ Code changes    │ None             │ Required         │
└─────────────────┴──────────────────┴──────────────────┘
```

---

## Single Server to Million Users

### Evolution Journey

**Stage 1: Single Server (1-100 users)**

```
Everything on one machine:

┌─────────────────────────────┐
│         Server              │
│  ┌──────────────────────┐   │
│  │  Web Application     │   │
│  │  (Node.js/Python)    │   │
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │  Database            │   │
│  │  (PostgreSQL/MySQL)  │   │
│  └──────────────────────┘   │
│  ┌──────────────────────┐   │
│  │  File Storage        │   │
│  └──────────────────────┘   │
└─────────────────────────────┘

Characteristics:
- Simple deployment
- Low cost ($50-100/month)
- Easy to develop
- Fast (no network calls)

Problems:
- Single point of failure
- Can't scale beyond ~100 users
- Resource contention (CPU/memory)
```

**Stage 2: Separate Database (100-1,000 users)**

```
Split application and database:

┌─────────────────┐         ┌─────────────────┐
│  Web Server     │ ←────→  │  Database       │
│  (Application)  │  TCP    │  (PostgreSQL)   │
│                 │         │                 │
│  4 CPU          │         │  8 CPU          │
│  8 GB RAM       │         │  16 GB RAM      │
└─────────────────┘         └─────────────────┘

Benefits:
- Independent scaling
- Optimize each server
- Web server: More CPU
- Database: More RAM

Cost: ~$200/month
```

**Stage 3: Add Load Balancer + Multiple Servers (1,000-10,000 users)**

```
Horizontal scaling begins:

        User
         ↓
    ┌──────────┐
    │   DNS    │
    └────┬─────┘
         ↓
┌─────────────────┐
│ Load Balancer   │
└────────┬────────┘
    ┌────┴────┐
    ↓         ↓
┌────────┐ ┌────────┐
│ Web 1  │ │ Web 2  │
└────┬───┘ └────┬───┘
     └─────┬────┘
           ↓
    ┌──────────────┐
    │   Database   │
    └──────────────┘

New Components:
1. DNS (Route 53, Cloudflare)
2. Load Balancer (nginx, HAProxy, ALB)
3. Multiple web servers (auto-scaling)

Benefits:
- High availability
- Handle traffic spikes
- No downtime deployments

Challenge:
- Session management (use Redis)
- Database becomes bottleneck

Cost: ~$500/month
```

**Stage 4: Add Caching (10,000-100,000 users)**

```
Reduce database load:

┌──────────────┐
│ Load Balancer│
└──────┬───────┘
   ┌───┴───┐
   ↓       ↓
┌────┐  ┌────┐
│Web1│  │Web2│
└─┬──┘  └─┬──┘
  │       │
  ├───────┼──────┐
  ↓       ↓      ↓
┌────┐ ┌─────┐ ┌──────────┐
│DB  │ │Cache│ │ CDN      │
│    │ │Redis│ │Cloudflare│
└────┘ └─────┘ └──────────┘

Layers:
1. CDN: Static files (images, CSS, JS)
2. Redis: Application cache (sessions, popular data)
3. Database: Source of truth

Performance improvement:
- Page load: 2000ms → 200ms
- Database queries: 100/sec → 10/sec

Cost: ~$1,000/month
```

**Stage 5: Database Replication (100,000-1M users)**

```
Read/write separation:

┌──────────────┐
│ Load Balancer│
└──────┬───────┘
   ┌───┴───┐
┌────┐  ┌────┐
│Web1│  │Web2│
└─┬──┘  └─┬──┘
  │       │
  │ Writes│
  └───┬───┘
      ↓
┌──────────────┐
│ Primary DB   │ (writes)
└──────┬───────┘
       │ Replication
   ┌───┼────┐
   ↓   ↓    ↓
┌────┐ ┌────┐ ┌────┐
│Rep1│ │Rep2│ │Rep3│ (reads)
└────┘ └────┘ └────┘

Pattern:
- All writes → Primary
- All reads → Replicas
- Replication lag: <100ms

Scaling math:
If 90% reads, 10% writes:
- 1 primary handles 10,000 writes/sec
- 3 replicas handle 90,000 reads/sec
- Total: 100,000 req/sec

Cost: ~$3,000/month
```

**Stage 6: Add Message Queue (1M+ users)**

```
Async processing:

┌──────────────┐
│ Load Balancer│
└──────┬───────┘
   ┌───┴───┐
┌────┐  ┌────┐
│Web1│  │Web2│
└─┬──┘  └─┬──┘
  │       │
  └───┬───┘
      ↓
┌─────────────┐
│Message Queue│
│ (RabbitMQ/  │
│   Kafka)    │
└──────┬──────┘
   ┌───┴────┐
   ↓        ↓
┌────┐  ┌────────┐
│DB  │  │Workers │
└────┘  └────────┘

Use cases:
- Email sending (async)
- Image processing
- Report generation
- Analytics

Benefits:
- Decouple services
- Handle traffic spikes
- Background processing
- Retry failed jobs

Example:
User uploads photo → Response instant
Queue processes:
  - Resize (3 sizes)
  - Apply filters
  - Generate thumbnails
  - Store in S3
  - Update database

Cost: ~$5,000/month
```

**Stage 7: Microservices + Sharding (10M+ users)**

```
Full distributed system:

┌──────────────────────┐
│   API Gateway        │
└──────────┬───────────┘
      ┌────┴────┐
      ↓         ↓
┌──────────┐ ┌──────────┐
│User Svc  │ │Order Svc │
└────┬─────┘ └────┬─────┘
     │            │
┌────┴───┐   ┌───┴────┐
│User DB │   │Order DB│
│Shard 1 │   │Shard 1 │
└────────┘   └────────┘
     │            │
┌────┴───┐   ┌───┴────┐
│User DB │   │Order DB│
│Shard 2 │   │Shard 2 │
└────────┘   └────────┘

Sharding strategy:
User shard = user_id % 4
- users 1,5,9... → Shard 0
- users 2,6,10... → Shard 1
- users 3,7,11... → Shard 2
- users 4,8,12... → Shard 3

Benefits:
- Independent scaling
- Isolated failures
- Team autonomy

Challenges:
- Distributed transactions
- Data consistency
- Operational complexity

Cost: ~$50,000/month
```

---

## Database Scaling Patterns

### Read Replicas

**Pattern:**
```
Primary (Write)     Replicas (Read)
      ↓                ↓  ↓  ↓
[Write Queries] → [Primary DB]
                       ↓
                  Replication
                       ↓
              ┌────────┼────────┐
              ↓        ↓        ↓
          [Replica1][Replica2][Replica3]
              ↑        ↑        ↑
         [Read Queries from App]
```

**Configuration Example (PostgreSQL):**
```sql
-- Primary server
-- postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_segments = 64

-- Replica server
-- recovery.conf
standby_mode = 'on'
primary_conninfo = 'host=primary-db port=5432'
trigger_file = '/tmp/postgresql.trigger.5432'
```

**Application Code:**
```javascript
// Separate read/write connections
const writeDB = new Pool({
  host: 'primary.db.example.com',
  port: 5432
});

const readDB = new Pool({
  host: 'replica.db.example.com',
  port: 5432
});

// Route queries
async function getUser(id) {
  // Read from replica
  return readDB.query('SELECT * FROM users WHERE id = $1', [id]);
}

async function updateUser(id, data) {
  // Write to primary
  return writeDB.query('UPDATE users SET ... WHERE id = $1', [id, ...]);
}
```

**Replication Lag:**
```
Problem:
14:30:00.000 - Write to primary: user.name = "John"
14:30:00.100 - Replication lag: 100ms
14:30:00.050 - Read from replica: user.name = "Jane" (stale!)

Solutions:
1. Read from primary after write
2. Add timestamp check
3. Sticky sessions (same user → same replica)
4. Accept eventual consistency
```

### Database Sharding

**Horizontal Partitioning:**
```
Single Database (Before):
users table: 100M rows
┌────┬──────┬───────┐
│ id │ name │ email │
├────┼──────┼───────┤
│  1 │John  │john@  │
│  2 │Jane  │jane@  │
│... │...   │...    │
│100M│Kate  │kate@  │
└────┴──────┴───────┘
Queries slow, storage full

Sharded (After):
Shard by user_id % 4

Shard 0 (id % 4 = 0):
┌────┬──────┬───────┐
│  4 │Mike  │mike@  │
│  8 │Anna  │anna@  │
│ 12 │Tom   │tom@   │
└────┴──────┴───────┘

Shard 1 (id % 4 = 1):
┌────┬──────┬───────┐
│  1 │John  │john@  │
│  5 │Lisa  │lisa@  │
│  9 │Bob   │bob@   │
└────┴──────┴───────┘

Shard 2 (id % 4 = 2):
┌────┬──────┬───────┐
│  2 │Jane  │jane@  │
│  6 │Mark  │mark@  │
│ 10 │Sara  │sara@  │
└────┴──────┴───────┘

Shard 3 (id % 4 = 3):
┌────┬──────┬───────┐
│  3 │Paul  │paul@  │
│  7 │Emma  │emma@  │
│ 11 │Dave  │dave@  │
└────┴──────┴───────┘
```

**Sharding Strategies:**

**1. Range-Based Sharding:**
```
Shard by ID ranges:
Shard 0: user_id 1-25M
Shard 1: user_id 25M-50M
Shard 2: user_id 50M-75M
Shard 3: user_id 75M-100M

Pros: Simple, easy to add shards
Cons: Uneven distribution, hotspots
```

**2. Hash-Based Sharding:**
```
shard = hash(user_id) % num_shards

Pros: Even distribution
Cons: Hard to add shards (rehashing needed)
```

**3. Geographic Sharding:**
```
Shard by region:
Shard US-East: US users
Shard EU-West: European users
Shard Asia: Asian users

Pros: Low latency, data locality
Cons: Uneven distribution, complex routing
```

**Implementation:**
```javascript
class ShardedDatabase {
  constructor() {
    this.shards = [
      new Pool({ host: 'shard0.db.com' }),
      new Pool({ host: 'shard1.db.com' }),
      new Pool({ host: 'shard2.db.com' }),
      new Pool({ host: 'shard3.db.com' })
    ];
  }
  
  getShard(userId) {
    const shardIndex = userId % this.shards.length;
    return this.shards[shardIndex];
  }
  
  async getUser(userId) {
    const shard = this.getShard(userId);
    return shard.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
  
  async getUserOrders(userId) {
    // Problem: Orders might be on different shard!
    // Solution: Shard orders by user_id too
    const shard = this.getShard(userId);
    return shard.query('SELECT * FROM orders WHERE user_id = $1', [userId]);
  }
}
```

**Challenges:**
```
1. Cross-shard queries:
   SELECT * FROM users JOIN orders...
   ❌ Can't join across shards!
   ✅ Solution: Denormalize or app-level joins

2. Resharding:
   Adding 5th shard → data must move
   ❌ Downtime required
   ✅ Solution: Consistent hashing

3. Transactions:
   BEGIN; UPDATE users...; UPDATE orders...; COMMIT;
   ❌ Users and orders on different shards
   ✅ Solution: Saga pattern, eventual consistency
```

---

## Scaling Challenges

### State Management

**Problem:**
```
User session stored in server memory:

Request 1 → Load Balancer → Server 1 (login, session created)
Request 2 → Load Balancer → Server 2 (no session! user logged out?)

Solution: Stateless servers
```

**Redis Session Store:**
```javascript
// Before (stateful):
app.use(session({
  store: new MemoryStore(), // ❌ In server memory
  secret: 'secret'
}));

// After (stateless):
app.use(session({
  store: new RedisStore({  // ✅ Shared Redis
    host: 'redis.example.com',
    port: 6379
  }),
  secret: 'secret'
}));

Flow:
User login → Session saved to Redis
Request 1 → Server 1 → Reads session from Redis
Request 2 → Server 2 → Reads same session from Redis
```

### Data Consistency

**CAP Theorem:**
```
Can only have 2 of 3:
- Consistency (all nodes see same data)
- Availability (system always responds)
- Partition Tolerance (works despite network failures)

Examples:
CP (Consistency + Partition Tolerance):
- HBase, MongoDB (strong consistency mode)
- Sacrifices availability during partition

AP (Availability + Partition Tolerance):
- Cassandra, DynamoDB
- Sacrifices consistency (eventual consistency)

CA (Consistency + Availability):
- Single-node databases
- ❌ Not realistic in distributed systems
```

### Cost vs Performance

```
Optimization Example:

Current: 10 servers @ $100/month = $1,000/month
Handles: 10,000 req/sec

Option 1: Add more servers
Cost: 20 servers @ $100 = $2,000/month
Result: 20,000 req/sec
Cost per req: Same

Option 2: Optimize code
Investment: $10,000 (engineer time)
Result: 5x improvement = 50,000 req/sec on same 10 servers
Cost per req: 80% reduction!

Lesson: Optimize before scaling!
```

---

## Real-World Examples

### Twitter's Scaling Journey

```
2006: Single Rails server, MySQL
Users: 1,000

2008: Multiple app servers, read replicas
Users: 1M

2010: Sharded MySQL, memcached
Users: 50M

2012: Cassandra, custom tweet timeline service
Users: 200M

2020: Microservices, Manhattan (distributed DB)
Users: 350M
Tweets: 500M/day
```

### Instagram's Architecture

```
Scale: 1B+ users

Key Decisions:
1. PostgreSQL sharded by user_id
2. Cassandra for feeds
3. Redis for caching
4. CDN for images/videos

Sharding Strategy:
- 4,000 PostgreSQL shards
- Each shard: ~250,000 users
- Can add shards without downtime
```

---

## Quick Reference

### When to Scale

```
Vertical: Quick fix, < 10K users, legacy apps
Horizontal: Long-term, > 10K users, cloud-native

Signals to scale:
- Response time > 1 second
- CPU > 80% sustained
- Memory > 90%
- Database connections maxed out
```

### Scaling Checklist

```
✅ Separate concerns (web, DB, cache)
✅ Use load balancer
✅ Implement caching
✅ Add read replicas
✅ Make servers stateless
✅ Consider sharding at 100M+ rows
✅ Use CDN for static content
✅ Monitor everything
```

---

This guide covers scalability fundamentals. Next topics: Load Balancing, Caching, and CDN deep dives in separate files.