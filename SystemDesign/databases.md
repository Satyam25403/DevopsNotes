# System Design: Databases

Comprehensive guide to database selection, design patterns, indexing strategies, and scaling with decision trees for real-world systems.

## Table of Contents
- [Database Selection Framework](#database-selection-framework)
- [SQL vs NoSQL Decision Making](#sql-vs-nosql-decision-making)
- [Database Types Deep Dive](#database-types-deep-dive)
- [Indexing Strategies](#indexing-strategies)
- [Data Modeling Patterns](#data-modeling-patterns)
- [Database Scaling](#database-scaling)
- [Real-World Use Cases](#real-world-use-cases)

---

## Database Selection Framework

### The Core Question

**Before choosing a database, answer these questions:**

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              DATABASE SELECTION DECISION TREE                       │
│                                                                     │
│  What is your data structure?                                       │
│  ├── Structured, fixed schema         → SQL (PostgreSQL, MySQL)     │
│  ├── Semi-structured (JSON, XML)      → Document DB (MongoDB)       │
│  ├── Key-Value pairs only             → Redis, DynamoDB             │
│  ├── Graph relationships               → Neo4j, Neptune             │
│  └── Time-series data                  → InfluxDB, TimescaleDB      │
│                                                                     │
│  What are your consistency needs?                                   │
│  ├── ACID required (finance, inventory) → SQL                       │
│  ├── Eventual consistency OK           → NoSQL (Cassandra)          │
│  └── Strong consistency + scale        → NewSQL (CockroachDB)       │
│                                                                     │
│  What is your scaling pattern?                                      │
│  ├── Vertical scaling sufficient       → Traditional SQL            │
│  ├── Horizontal scaling required       → NoSQL or NewSQL            │
│  └── Hybrid (read-heavy)               → SQL + Read Replicas        │
│                                                                     │
│  What are your query patterns?                                      │
│  ├── Complex joins, analytics          → SQL (PostgreSQL)           │
│  ├── Simple lookups by ID              → Key-Value (Redis)          │
│  ├── Full-text search                  → Elasticsearch              │
│  ├── Aggregations on large data        → Column-store (ClickHouse)  │
│  └── Graph traversal                   → Graph DB (Neo4j)           │
│                                                                     │
│  What is your write/read ratio?                                     │
│  ├── Read-heavy (90% reads)           → SQL + caching               │
│  ├── Write-heavy (analytics, logs)    → LSM-based (Cassandra)       │
│  └── Balanced                          → SQL or Document DB         │
└─────────────────────────────────────────────────────────────────────┘
```

### Critical Trade-offs Matrix

```
┌──────────────────┬──────────┬──────────┬──────────┬──────────┐
│ Requirement      │   SQL    │ NoSQL-Doc│ Key-Value│  Graph   │
├──────────────────┼──────────┼──────────┼──────────┼──────────┤
│ ACID Guarantees  │    ✅   │     ⚠️   │    ❌   │    ⚠️    │
│ Schema Flex      │    ❌   │     ✅   │    ✅   │    ✅    │
│ Horizontal Scale │    ⚠️   │     ✅   │    ✅   │    ⚠️    │
│ Complex Queries  │    ✅   │     ⚠️   │    ❌   │    ✅    │
│ Write Speed      │    ⚠️   │     ✅   │    ✅   │    ⚠️    │
│ Read Speed       │    ✅   │     ✅   │    ✅   │    ⚠️    │
│ Maturity         │    ✅   │     ✅   │    ✅   │    ⚠️    │
│ Learning Curve   │   Easy   │  Medium  │   Easy   │   Hard   │
└──────────────────┴──────────┴──────────┴──────────┴──────────┘

Legend:
✅ Excellent  ⚠️ Acceptable  ❌ Poor/Limited
```

---

## SQL vs NoSQL Decision Making

### When SQL is the Right Choice

**Use SQL when:**

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                    SQL DATABASE INDICATORS                          │
│                                                                     │
│  ✅ Your data has clear relationships                              │
│     Example: Users → Orders → Products → Inventory                  │
│     Why: Foreign keys, joins are natural                            │
│                                                                     │
│  ✅ Transactions are critical                                       │
│     Example: Banking (transfer $100 from A to B)                    │
│     Why: ACID guarantees prevent inconsistency                      │
│                                                                     │
│  ✅ Complex queries and analytics needed                            │
│     Example: "Show revenue by product category in Q4"               │
│     Why: SQL excels at JOINs, aggregations, window functions        │
│                                                                     │
│  ✅ Data structure is well-defined and stable                        │
│     Example: HR system (employee, department, salary)               │
│     Why: Schema enforces data integrity                             │
│                                                                     │
│  ✅ Vertical scaling is acceptable                                   │
│     Example: Internal tools, < 100K concurrent users                │
│     Why: Single powerful server handles load                        │
└─────────────────────────────────────────────────────────────────────┘
```

**Real-World SQL Example:**

```sql
-- E-commerce order system
-- Strong relationships, need ACID

-- Transaction: Create order (all-or-nothing)
BEGIN TRANSACTION;

  -- Check inventory
  SELECT stock FROM products WHERE id = 123 FOR UPDATE;
  
  -- If stock > 0:
  -- 1. Create order
  INSERT INTO orders (user_id, total) VALUES (456, 99.99);
  
  -- 2. Add order items
  INSERT INTO order_items (order_id, product_id, quantity)
  VALUES (LAST_INSERT_ID(), 123, 1);
  
  -- 3. Decrement inventory
  UPDATE products SET stock = stock - 1 WHERE id = 123;
  
  -- 4. Create payment record
  INSERT INTO payments (order_id, amount, status)
  VALUES (LAST_INSERT_ID(), 99.99, 'pending');

COMMIT;
-- Either ALL succeed or ALL fail (ACID)
```

### When NoSQL is the Right Choice

**Use NoSQL when:**

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                   NOSQL DATABASE INDICATORS                         │
│                                                                     │
│  ✅ Schema changes frequently                                        │
│     Example: SaaS with custom fields per customer                   │
│     Why: No migrations needed, add fields on-the-fly                │
│                                                                      │
│  ✅ Massive scale is required                                         │
│     Example: Social media (billions of posts)                       │
│     Why: Horizontal scaling is native                               │
│                                                                      │
│  ✅ High write throughput needed                                      │
│     Example: IoT sensors (millions of writes/sec)                   │
│     Why: LSM trees optimize writes                                  │
│                                                                      │
│  ✅ Data is naturally document-based                                  │
│     Example: User profiles with varying attributes                  │
│     Why: Store entire document, no joins needed                     │
│                                                                      │
│  ✅ Geographical distribution required                                │
│     Example: Global app with data in multiple regions               │
│     Why: Multi-datacenter replication built-in                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Real-World NoSQL Example:**

```javascript
// MongoDB - User profile with flexible schema
// Different users have different fields

// User 1: Basic profile
{
  "_id": "user123",
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2024-01-15",
  "preferences": {
    "theme": "dark",
    "language": "en"
  }
}

// User 2: Premium with extra fields
{
  "_id": "user456",
  "name": "Jane Smith",
  "email": "jane@example.com",
  "created_at": "2024-01-15",
  "preferences": {
    "theme": "light",
    "language": "es"
  },
  // Extra fields - no schema change needed!
  "premium": true,
  "subscription_end": "2025-01-15",
  "payment_method": {
    "type": "credit_card",
    "last_four": "1234"
  },
  "custom_fields": {
    "company": "Acme Corp",
    "department": "Engineering"
  }
}

// No ALTER TABLE needed!
// No migration required!
// Just insert new fields
```

### The Gray Area: When Both Could Work

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              CONSIDER BOTH SQL AND NOSQL WHEN:                      │
│                                                                     │
│  Scenario: E-commerce application                                   │
│                                                                     │
│  SQL Approach:                                                       │
│  ├─ Products, orders, inventory in PostgreSQL                       │
│  ├─ Transactions for order processing                               │
│  ├─ Complex analytics queries                                        │
│  └─ Challenge: Scaling product catalog to millions                  │
│                                                                     │
│  NoSQL Approach:                                                     │
│  ├─ Product catalog in MongoDB                                      │
│  ├─ Easy schema changes for new product types                       │
│  ├─ Better horizontal scaling                                       │
│  └─ Challenge: Complex reporting, order transactions                │
│                                                                     │
│  Hybrid Solution (Often Best):                                      │
│  ├─ PostgreSQL: Orders, payments, inventory (ACID critical)         │
│  ├─ MongoDB: Product catalog, user reviews (flexible schema)        │
│  ├─ Redis: Session store, cache (fast reads)                        │
│  └─ Elasticsearch: Product search (full-text)                       │
│                                                                     │
│  Rule: Use the right tool for each job!                             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Database Types Deep Dive

### 1. Relational Databases (SQL)

**Core Concept:** Structured data in tables with relationships

**Architecture:**
```
┌─────────────────────────────────────────┐
│        Relational Database              │
│                                         │
│  ┌──────────┐      ┌──────────┐         │
│  │  users   │──┐   │  orders  │         │
│  ├──────────┤  │   ├──────────┤         │
│  │ id (PK)  │  └───│ user_id  │         │
│  │ name     │      │ total    │         │
│  │ email    │      │ date     │         │
│  └──────────┘      └──────┬───┘         │
│                           │             │
│              ┌────────────┘             │
│              ↓                          │
│       ┌──────────────┐                  │
│       │ order_items  │                  │
│       ├──────────────┤                  │
│       │ order_id(FK) │                  │
│       │ product_id   │                  │
│       │ quantity     │                  │
│       └──────────────┘                  │
└─────────────────────────────────────────┘

Foreign Keys enforce referential integrity
```

**Popular Options:**

**PostgreSQL:**
```markdown
Best For:
✅ Complex queries (CTEs, window functions)
✅ JSON data with SQL (JSONB type)
✅ Full-text search (built-in)
✅ Geographic data (PostGIS)

Weaknesses:
❌ Write-heavy workloads
❌ Horizontal scaling (requires sharding)

When to Choose:
- General-purpose OLTP
- Need both SQL and JSON
- Open-source preferred
```

**MySQL:**
```markdown
Best For:
✅ Read-heavy workloads
✅ Simple transactions
✅ Wide ecosystem support

Weaknesses:
❌ Less advanced SQL features
❌ Limited JSON support (vs PostgreSQL)

When to Choose:
- Proven at scale (Facebook, Twitter used it)
- Simple CRUD operations
- Large community support
```

**Decision Tree:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              POSTGRESQL VS MYSQL DECISION                           │
│                                                                     │
│  Need advanced SQL features (CTEs, window functions)?               │
│  ├── YES → PostgreSQL                                               │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Working with JSON data extensively?                                │
│  ├── YES → PostgreSQL (JSONB is superior)                           │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Read-heavy, simple queries?                                        │
│  ├── YES → MySQL (slightly faster reads)                            │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Need GIS/spatial data?                                             │
│  ├── YES → PostgreSQL + PostGIS                                     │
│  └── NO  → Either works, prefer PostgreSQL for flexibility          │
└─────────────────────────────────────────────────────────────────────┘
```

### 2. Document Databases

**Core Concept:** Store data as JSON-like documents

**Architecture:**
```
┌─────────────────────────────────────────┐
│      Document Database (MongoDB)        │
│                                         │
│  Collection: users                      │
│  ┌─────────────────────────────────┐    │
│  │ {                               │    │
│  │   "_id": "123",                 │    │
│  │   "name": "John",               │    │
│  │   "email": "john@example.com",  │    │
│  │   "addresses": [                │    │
│  │     {                           │    │
│  │       "type": "home",           │    │
│  │       "street": "123 Main St"   │    │
│  │     }                           │    │
│  │   ],                            │    │
│  │   "orders": [                   │    │
│  │     {                           │    │
│  │       "id": "order1",           │    │
│  │       "total": 99.99,           │    │
│  │       "items": [...]            │    │
│  │     }                           │    │
│  │   ]                             │    │
│  │ }                               │    │
│  └─────────────────────────────────┘    │
│                                         │
│  No foreign keys - embed or reference   │
└─────────────────────────────────────────┘
```

**MongoDB:**
```markdown
Best For:
✅ Rapidly changing schemas
✅ Document-centric data (user profiles, blog posts)
✅ Horizontal scaling
✅ Real-time analytics

Weaknesses:
❌ Complex joins (must denormalize)
❌ Transactions across documents (limited)

When to Choose:
- Content management systems
- Mobile app backends
- Real-time analytics
- Flexible schema needed
```

**Decision: Embed vs Reference:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│            MONGODB: EMBED VS REFERENCE DECISION                     │
│                                                                     │
│  How often do you access this data together?                        │
│  ├── Always (user + addresses)     → EMBED                          │
│  └── Sometimes (user + orders)     → REFERENCE                      │
│                                                                     │
│  How big will the embedded array grow?                              │
│  ├── Small (<100 items)            → EMBED                          │
│  ├── Medium (100-1000)             → Consider both                  │
│  └── Large (>1000 items)           → REFERENCE                      │
│                                                                     │
│  Does data need to exist independently?                             │
│  ├── YES (products, orders)        → REFERENCE                      │
│  └── NO (comments on post)         → EMBED                          │
│                                                                     │
│  Do you need to query this separately?                              │
│  ├── YES (search all orders)       → REFERENCE                      │
│  └── NO (just with parent)         → EMBED                          │
└─────────────────────────────────────────────────────────────────────┘

Example - User Profile:
{
  "user_id": "123",
  "addresses": [...],      // EMBED: Always need together
  "preferences": {...},    // EMBED: Small, always needed
  "order_ids": [...]       // REFERENCE: Large, query separately
}
```

### 3. Key-Value Stores

**Core Concept:** Simple hash table, map keys to values

**Architecture:**
```
┌─────────────────────────────────────────┐
│       Key-Value Store (Redis)           │
│                                         │
│  Key: "user:123:session"                │
│  Value: "eyJhbGciOiJIUzI1NiIsInR5..."   │
│                                         │
│  Key: "product:456:price"               │
│  Value: "99.99"                         │
│                                         │
│  Key: "cache:homepage"                  │
│  Value: "<html>...</html>"              │
│                                         │
│  Simple: O(1) lookup                    │
│  No queries, just GET/SET               │
└─────────────────────────────────────────┘
```

**Redis:**
```markdown
Best For:
✅ Caching (most common use)
✅ Session storage
✅ Real-time leaderboards
✅ Rate limiting
✅ Pub/sub messaging

Weaknesses:
❌ Limited query capabilities
❌ In-memory (expensive for large data)
❌ No complex relationships

When to Choose:
- Need sub-millisecond latency
- Caching layer
- Session store
- Simple data structures
```

**DynamoDB (AWS):**
```markdown
Best For:
✅ Serverless applications
✅ Predictable performance at any scale
✅ Single-digit millisecond latency

Weaknesses:
❌ Expensive at high scale
❌ Limited query flexibility
❌ Vendor lock-in

When to Choose:
- AWS-native applications
- Serverless architecture
- Need guaranteed performance
```

**Decision: Which Key-Value Store:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│           KEY-VALUE STORE SELECTION DECISION                        │
│                                                                     │
│  What is your primary use case?                                     │
│  ├── Caching                       → Redis (in-memory)             │
│  ├── Session storage               → Redis                         │
│  ├── Durable key-value storage     → DynamoDB                      │
│  └── Distributed cache             → Memcached                     │
│                                                                     │
│  Do you need persistence?                                           │
│  ├── NO (cache only)               → Memcached (simpler)           │
│  └── YES                            → Redis or DynamoDB            │
│                                                                     │
│  Do you need data structures (lists, sets, sorted sets)?            │
│  ├── YES                            → Redis                        │
│  └── NO                             → Memcached or DynamoDB        │
│                                                                     │
│  Are you in AWS ecosystem?                                          │
│  ├── YES, serverless               → DynamoDB                      │
│  ├── YES, but self-managed OK      → Redis on EC2/ElastiCache     │
│  └── NO (on-premise/other cloud)  → Redis                         │
└─────────────────────────────────────────────────────────────────────┘
```

### 4. Graph Databases

**Core Concept:** Nodes and edges (relationships are first-class citizens)

**Architecture:**
```
┌─────────────────────────────────────────┐
│        Graph Database (Neo4j)           │
│                                         │
│     (User)                              │
│       ↓                                 │
│     [John]───FRIEND_OF───→[Jane]        │
│       │                      │          │
│       │                      │          │
│   LIKES                   LIKES         │
│       │                      │          │
│       ↓                      ↓          │
│   (Product)             (Product)       │
│     [Phone]←──BOUGHT───[Laptop]         │
│                                         │
│  Relationships are fast to traverse     │
│  No JOIN tables needed                  │
└─────────────────────────────────────────┘
```

**Neo4j:**
```markdown
Best For:
✅ Social networks (friends of friends)
✅ Recommendation engines
✅ Fraud detection
✅ Knowledge graphs

Weaknesses:
❌ Not for transactional data
❌ Limited horizontal scaling
❌ Steep learning curve (Cypher query language)

When to Choose:
- Deeply connected data
- Recommendations
- Network analysis
- Relationship-centric queries
```

**Query Example:**
```cypher
// Find friends of friends who like same products
// In SQL: Complex multi-level JOIN
// In Neo4j: Natural graph traversal

MATCH (me:User {name: 'John'})
  -[:FRIEND_OF*1..2]->(friend)
  -[:LIKES]->(product)
WHERE (me)-[:LIKES]->(product)
RETURN friend.name, product.name
ORDER BY COUNT(product) DESC
LIMIT 10

// Fast even with millions of relationships!
```

**Decision: Do You Need a Graph DB:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              GRAPH DATABASE DECISION TREE                           │
│                                                                     │
│  How many relationship hops do you query?                            │
│  ├── 0-1 levels (user → orders)    → SQL is fine                   │
│  ├── 2-3 levels (friends of friends) → Consider Graph DB           │
│  └── 4+ levels (network analysis)   → Definitely Graph DB          │
│                                                                     │
│  Is relationship data your primary concern?                          │
│  ├── YES (social network, fraud)   → Graph DB                      │
│  └── NO (transactional data)       → SQL                           │
│                                                                     │
│  Do you need to query relationships dynamically?                     │
│  ├── YES ("find shortest path")    → Graph DB                      │
│  └── NO (fixed relationships)      → SQL with joins                │
│                                                                     │
│  Real-world test:                                                    │
│  If your SQL queries have > 3 JOINs regularly → Consider Graph DB   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Indexing Strategies

### Index Type Selection Framework

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                INDEX TYPE SELECTION DECISION                        │
│                                                                     │
│  What type of queries do you run?                                   │
│  ├── Exact match (WHERE id = ?)        → B-Tree or Hash           │
│  ├── Range (WHERE date BETWEEN)        → B-Tree only              │
│  ├── Pattern match (WHERE name LIKE)   → B-Tree or GIN            │
│  ├── Full-text search                  → GIN/Full-text index      │
│  └── Geospatial (nearby locations)     → GiST/R-Tree              │
│                                                                     │
│  What is your read/write ratio?                                     │
│  ├── Read-heavy (90%+ reads)           → B-Tree (fast reads)      │
│  ├── Write-heavy (logs, metrics)       → LSM-Tree (fast writes)   │
│  └── Balanced                           → B-Tree                   │
│                                                                     │
│  How large is your dataset?                                         │
│  ├── Small (fits in memory)            → Hash index OK            │
│  ├── Medium-Large (on disk)            → B-Tree                   │
│  └── Huge (TB+, write-heavy)           → LSM-Tree                 │
│                                                                     │
│  Do you need to support multiple query patterns?                    │
│  ├── YES (flexible queries)            → B-Tree (most versatile)  │
│  └── NO (single pattern)               → Specialized index        │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. B-Tree Index (Default Choice)

**How It Works:**
```
B-Tree Structure:
                    [40, 80]
                   /    |    \
         [10,20,30]  [50,60,70]  [90]
         /  |  |  \    |  |  |     \
      [5][15][25][35][45][55][65][85][95]

Characteristics:
- Balanced tree (all leaves same depth)
- Sorted data (enables range queries)
- O(log n) lookup time
- Self-balancing on inserts

Example with 1 million rows:
- Tree depth: ~20 levels
- Lookups: ~20 disk reads
- vs Full scan: 1 million reads!
```

**When to Use:**
```javascript
// ✅ GOOD: Exact match queries
SELECT * FROM users WHERE id = 12345;
// B-Tree: Navigate down tree to find 12345

// ✅ GOOD: Range queries
SELECT * FROM orders WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31';
// B-Tree: Find start point, scan sorted sequence

// ✅ GOOD: Sorting
SELECT * FROM products ORDER BY price;
// B-Tree: Already sorted, no extra work

// ✅ GOOD: Prefix matching
SELECT * FROM users WHERE email LIKE 'john%';
// B-Tree: Find 'john', scan until no longer matches

// ❌ BAD: Suffix matching
SELECT * FROM users WHERE email LIKE '%@gmail.com';
// B-Tree can't help - must scan all

// ❌ BAD: Function on column
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';
// B-Tree indexes email, not LOWER(email)
// Solution: Create index on LOWER(email)
```

**Implementation:**
```sql
-- PostgreSQL
CREATE INDEX idx_users_email ON users(email);

-- Composite index (order matters!)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- This index helps:
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';
SELECT * FROM orders WHERE user_id = 123;

-- This index does NOT help:
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Why? user_id is first column, must be in WHERE clause
```

### 2. Hash Index

**How It Works:**
```
Hash Table Structure:
Key "user_123" → Hash Function → Bucket 47
Key "user_456" → Hash Function → Bucket 12
Key "user_789" → Hash Function → Bucket 47 (collision!)

hash("user_123") % num_buckets = bucket_index

Characteristics:
- O(1) lookup for exact matches
- No ordering (can't do range queries)
- Collisions require resolution
```

**When to Use:**
```javascript
// ✅ PERFECT: Exact equality, high frequency
// Session tokens (UUIDs)
CREATE INDEX idx_sessions_token USING HASH (token);
SELECT * FROM sessions WHERE token = 'abc123...';
// Hash: Instant O(1) lookup

// ❌ USELESS: Range queries
SELECT * FROM users WHERE age > 25;
// Hash destroys ordering information

// ❌ USELESS: Prefix matching
SELECT * FROM users WHERE email LIKE 'john%';
// Hash can't do partial matches
```

**Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              B-TREE VS HASH INDEX DECISION                          │
│                                                                     │
│  Do you EVER need range queries on this column?                     │
│  ├── YES → B-Tree (hash can't do ranges)                           │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Do you EVER need sorting by this column?                           │
│  ├── YES → B-Tree (hash can't sort)                                │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Do you EVER need prefix matching (LIKE 'abc%')?                    │
│  ├── YES → B-Tree (hash can't do prefixes)                         │
│  └── NO  → Continue                                                 │
│                                                                     │
│  Is this column used ONLY for exact equality?                       │
│  ├── YES → Hash index might be faster                              │
│  └── NO  → B-Tree is safer choice                                  │
│                                                                     │
│  Is table size huge (> 100M rows) and queries super frequent?       │
│  ├── YES → Hash index worth benchmarking                           │
│  └── NO  → B-Tree is sufficient and more flexible                  │
│                                                                     │
│  Real-world advice: Use B-Tree unless you have measured proof       │
│  that Hash index provides significant benefit!                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3. LSM-Tree (Log-Structured Merge Tree)

**How It Works:**
```
Write Path (Optimized for writes):
1. Write → MemTable (in-memory, sorted)
2. When full → Flush to disk as SSTable (immutable)
3. Background compaction merges SSTables

┌──────────────────────────────────┐
│         MemTable (RAM)           │
│  sorted: k1→v1, k2→v2, k3→v3    │
└─────────────┬────────────────────┘
              │ (flush when full)
              ↓
┌──────────────────────────────────┐
│      Level 0 SSTables (Disk)    │
│  SSTable1  SSTable2  SSTable3    │
└─────────────┬────────────────────┘
              │ (compact)
              ↓
┌──────────────────────────────────┐
│      Level 1 SSTables           │
│  SSTable_merged                 │
└──────────────────────────────────┘

Read Path:
1. Check MemTable first (fastest)
2. Check recent SSTables
3. Check older levels
4. Bloom filters skip checking irrelevant SSTables
```

**Used By:** Cassandra, RocksDB, LevelDB, HBase

**Trade-offs:**
```markdown
Write Performance:
✅ Excellent (append-only, sequential writes)
   - No random disk seeks
   - Write to memory first (fast)
   - 10x-100x faster writes than B-Tree

Read Performance:
⚠️ Good but slower than B-Tree
   - Must check multiple SSTables
   - Bloom filters help
   - More compaction = faster reads

Space:
⚠️ Higher temporarily (write amplification)
   - Multiple copies during compaction
   - Compaction reduces over time
```

**When to Use:**
```javascript
// ✅ PERFECT: Write-heavy workloads
// Time-series data, logs, metrics
INSERT INTO metrics (timestamp, value) VALUES (...);
// LSM: Fast append-only writes

// ✅ GOOD: Large datasets with updates
// User activity logs
UPDATE events SET processed = true WHERE id = ...;
// LSM: Append new version, compact later

// ❌ AVOID: Read-heavy, need fastest reads
// Leaderboards requiring instant ranking
SELECT * FROM scores ORDER BY score DESC LIMIT 10;
// B-Tree faster for this access pattern
```

**Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              B-TREE VS LSM-TREE DECISION                            │
│                                                                     │
│  What is your write/read ratio?                                     │
│  ├── >50% writes (logs, metrics, events)  → LSM-Tree              │
│  ├── >70% reads (user queries)            → B-Tree                │
│  └── Balanced                              → B-Tree (safer)        │
│                                                                     │
│  What is your data retention?                                       │
│  ├── Time-series, delete old data         → LSM-Tree              │
│  ├── Permanent storage                    → Either works           │
│  └── Frequent updates to same keys        → LSM-Tree              │
│                                                                     │
│  Can you tolerate slightly higher read latency?                     │
│  ├── YES (analytics, batch jobs)          → LSM-Tree              │
│  ├── NO (user-facing queries)             → B-Tree                │
│  └── Mixed workload                        → Benchmark both        │
│                                                                     │
│  Examples:                                                          │
│  LSM: Cassandra (write-heavy), RocksDB (key-value store)          │
│  B-Tree: PostgreSQL (balanced), MySQL (read-heavy)                │
└─────────────────────────────────────────────────────────────────────┘
```

### Composite Index Strategy

**The Order Matters Rule:**
```sql
-- Index: (user_id, created_at, status)
CREATE INDEX idx_orders_multi ON orders(user_id, created_at, status);

-- ✅ Uses index (leftmost prefix)
SELECT * FROM orders WHERE user_id = 123;
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01';
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2024-01-01' AND status = 'paid';

-- ⚠️ Partially uses index
SELECT * FROM orders WHERE user_id = 123 AND status = 'paid';
-- Uses user_id part only, skips created_at, can't use status efficiently

-- ❌ Does NOT use index
SELECT * FROM orders WHERE created_at > '2024-01-01';
SELECT * FROM orders WHERE status = 'paid';
-- user_id not in WHERE clause, index useless
```

**Composite Index Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│          COMPOSITE INDEX COLUMN ORDER DECISION                      │
│                                                                     │
│  Step 1: Identify your most common WHERE clause patterns            │
│  Examples:                                                           │
│  - WHERE user_id = ? AND date = ?                                   │
│  - WHERE user_id = ?                                                 │
│  - WHERE user_id = ? AND status = ?                                 │
│                                                                     │
│  Step 2: Order columns by selectivity (most selective first)        │
│  Selectivity = Number of distinct values                            │
│                                                                     │
│  user_id:    1M distinct values → High selectivity                 │
│  status:     5 distinct values → Low selectivity                    │
│  date:       365 distinct values → Medium selectivity               │
│                                                                     │
│  Recommended order: (user_id, date, status)                         │
│  Why? user_id filters most rows first                               │
│                                                                     │
│  Step 3: Consider query frequency                                   │
│  If 80% queries use: WHERE user_id = ?                              │
│  If 15% queries use: WHERE user_id = ? AND date = ?                 │
│  If 5% queries use: WHERE user_id = ? AND date = ? AND status = ?   │
│                                                                     │
│  One index (user_id, date, status) handles ALL three patterns!      │
│                                                                     │
│  Step 4: Exception - Equality before ranges                         │
│  WHERE user_id = ? AND date > ?                                      │
│  Correct: (user_id, date) - equality first, range second            │
│  Wrong: (date, user_id) - range prevents using user_id              │
└─────────────────────────────────────────────────────────────────────┘
```

### Index Maintenance Decision

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              INDEX MAINTENANCE DECISION TREE                        │
│                                                                     │
│  Should I add this index?                                           │
│  ├── Is this column in WHERE clause frequently?                    │
│  │   └── NO  → Don't index                                         │
│  │   └── YES → Continue                                            │
│  │                                                                  │
│  ├── Is table frequently updated/inserted?                         │
│  │   └── YES → Index has cost (slower writes)                      │
│  │   └── NO  → Index is cheaper                                    │
│  │                                                                  │
│  ├── How many rows does query return?                              │
│  │   └── >20% of table → Full scan might be faster                │
│  │   └── <20% of table → Index helps                              │
│  │                                                                  │
│  ├── Is column highly selective?                                   │
│  │   └── NO (status: 3 values) → Index might not help             │
│  │   └── YES (user_id: 1M values) → Index helps                   │
│  │                                                                  │
│  └── Can existing index cover this query?                          │
│      └── YES → Reuse existing                                      │
│      └── NO  → Consider adding                                     │
│                                                                     │
│  General Rules:                                                     │
│  ✅ Index foreign keys (for joins)                                  │
│  ✅ Index columns in WHERE, ORDER BY, GROUP BY                      │
│  ✅ Start with 3-5 indexes per table                                │
│  ❌ Don't index everything ("over-indexing")                        │
│  ❌ Don't index low-selectivity columns alone (status, gender)     │
│  ❌ Remove unused indexes (check query logs)                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Data Modeling Patterns

### Normalization vs Denormalization

**Decision Framework:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│          NORMALIZATION VS DENORMALIZATION DECISION                  │
│                                                                     │
│  What is your read/write ratio?                                     │
│  ├── Write-heavy, few reads       → Normalize (avoid duplicate writes)│
│  ├── Read-heavy, few writes       → Denormalize (fast reads)       │
│  └── Balanced                      → Hybrid approach               │
│                                                                     │
│  How complex are your joins?                                        │
│  ├── Simple (2-3 tables)          → Normalize                      │
│  ├── Complex (5+ tables)          → Consider denormalizing         │
│  └── Performance-critical queries → Denormalize those paths        │
│                                                                     │
│  How often does data change?                                        │
│  ├── Frequently (prices, inventory) → Normalize (single update)   │
│  ├── Rarely (user names, addresses) → Denormalize OK              │
│  └── Static (product categories)   → Denormalize freely           │
│                                                                     │
│  Can you handle eventual consistency?                               │
│  ├── NO (financial data)          → Normalize                      │
│  ├── YES (analytics, caching)     → Denormalize                    │
│  └── Mixed                         → Hybrid                        │
└─────────────────────────────────────────────────────────────────────┘
```

**Example - E-commerce:**

**Normalized (3NF):**
```sql
-- Normalized: No redundancy
-- Advantage: Single source of truth
-- Disadvantage: Requires JOINs

-- Users table
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(100),
  email VARCHAR(255)
);

-- Products table
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(200),
  price DECIMAL(10,2)
);

-- Orders table
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT REFERENCES users(id),
  created_at TIMESTAMP
);

-- Order items table
CREATE TABLE order_items (
  id INT PRIMARY KEY,
  order_id INT REFERENCES orders(id),
  product_id INT REFERENCES products(id),
  quantity INT,
  price DECIMAL(10,2)  -- Price at time of order
);

-- Query requires 4-way JOIN
SELECT 
  u.name,
  o.id AS order_id,
  p.name AS product_name,
  oi.quantity,
  oi.price
FROM users u
JOIN orders o ON u.id = o.user_id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON p.id = oi.product_id
WHERE u.id = 123;
```

**Denormalized:**
```sql
-- Denormalized: Duplicate data for performance
-- Advantage: Fast reads, no JOINs
-- Disadvantage: Data duplication, consistency challenges

CREATE TABLE orders (
  id INT PRIMARY KEY,
  
  -- User data denormalized
  user_id INT,
  user_name VARCHAR(100),
  user_email VARCHAR(255),
  
  -- Order items denormalized (JSONB in PostgreSQL)
  items JSONB,
  /* Example items value:
  [
    {
      "product_id": 456,
      "product_name": "Laptop",
      "quantity": 1,
      "price": 999.99
    },
    {
      "product_id": 789,
      "product_name": "Mouse",
      "quantity": 2,
      "price": 29.99
    }
  ]
  */
  
  total DECIMAL(10,2),
  created_at TIMESTAMP
);

-- Single table query - FAST!
SELECT * FROM orders WHERE user_id = 123;

-- Trade-off:
-- ✅ 10x faster reads
-- ❌ If user changes email, must update all their orders
-- ❌ If product price changes, historical orders unaffected (might be desired!)
```

**Hybrid Approach (Best Practice):**
```sql
-- Hybrid: Normalize foreign keys, denormalize frequently accessed data

CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT REFERENCES users(id),  -- Normalized (foreign key)
  
  -- Denormalized snapshot (at time of order)
  user_name_snapshot VARCHAR(100),
  user_email_snapshot VARCHAR(255),
  
  items JSONB,  -- Denormalized order items
  total DECIMAL(10,2),
  created_at TIMESTAMP
);

-- Benefits:
-- ✅ Fast reads (no JOINs for display)
-- ✅ Can still JOIN to users table for current data
-- ✅ Historical snapshot preserved
-- ✅ Referential integrity maintained
```

---

## Database Scaling

### Scaling Decision Framework

```markdown
┌─────────────────────────────────────────────────────────────────────┐
│              DATABASE SCALING DECISION TREE                         │
│                                                                     │
│  What is your current bottleneck?                                   │
│  ├── CPU maxed out                                                  │
│  │   └─ Check: Missing indexes? Slow queries?                      │
│  │       ├─ YES → Optimize first (cheaper!)                        │
│  │       └─ NO  → Scale vertically or horizontally                 │
│  │                                                                  │
│  ├── Memory maxed out                                               │
│  │   └─ Scale vertically (add RAM) - quick win                     │
│  │                                                                  │
│  ├── Disk I/O saturated                                             │
│  │   └─ Options:                                                    │
│  │       ├─ Add read replicas (if read-heavy)                      │
│  │       ├─ Add caching layer (Redis)                              │
│  │       └─ Upgrade to SSD/NVMe                                    │
│  │                                                                  │
│  └── Network bandwidth                                              │
│      └─ Geographic distribution needed → Sharding by region        │
│                                                                     │
│  What is your read/write ratio?                                     │
│  ├── 90%+ reads                                                     │
│  │   └─ Solution: Read replicas (5-10 replicas)                    │
│  │       └─ Cost: $$$, Complexity: Low                             │
│  │                                                                  │
│  ├── 50/50 read/write                                               │
│  │   └─ Solution: Vertical scaling + caching                       │
│  │       └─ Cost: $$, Complexity: Low                              │
│  │                                                                  │
│  └── Write-heavy                                                    │
│      └─ Solution: Sharding or different DB (Cassandra)             │
│          └─ Cost: $$$$, Complexity: High                           │
│                                                                     │
│  How big is your database?                                          │
│  ├── < 100 GB     → Single server OK                               │
│  ├── 100 GB - 1 TB → Vertical + replicas                           │
│  ├── 1 TB - 10 TB  → Consider sharding                             │
│  └── > 10 TB       → Must shard                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### Read Replica Pattern

**When to Use:**
```markdown
Indicators:
✅ 80%+ read queries
✅ Analytics/reports slow down main database
✅ Geographic distribution (users worldwide)
✅ Different read patterns (OLTP vs analytics)

Implementation:
Primary (writes) → Replicas (reads)
    ↓                ↓  ↓  ↓
  [Write]       [Read][Read][Read]
```

**Application Code:**
```javascript
// Separate connection pools
const primary = new Pool({ host: 'primary.db.com' });
const replica = new Pool({ host: 'replica.db.com' });

// Route based on operation
async function getUser(id) {
  // Read from replica
  return replica.query('SELECT * FROM users WHERE id = $1', [id]);
}

async function updateUser(id, data) {
  // Write to primary
  return primary.query('UPDATE users SET ... WHERE id = $1', [id, data]);
}

// Read-after-write consistency
async function createUser(data) {
  // Write to primary
  const user = await primary.query('INSERT INTO users ...');
  
  // Read from primary (not replica!) to ensure consistency
  return primary.query('SELECT * FROM users WHERE id = $1', [user.id]);
  
  // Replication lag: replica might not have new user yet!
}
```

**Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│            READ REPLICA CONFIGURATION DECISION                      │
│                                                                     │
│  How many replicas do you need?                                     │
│  ├── Read QPS / Single server capacity = Base number               │
│  └── Add 1-2 extra for high availability                           │
│                                                                     │
│  Example calculation:                                               │
│  - Total read QPS: 10,000                                           │
│  - Single server: 2,000 QPS                                         │
│  - Needed: 10,000 / 2,000 = 5 replicas                             │
│  - Add HA: 5 + 2 = 7 replicas total                                │
│                                                                     │
│  Where to place replicas?                                           │
│  ├── Same datacenter → Low latency, single point of failure        │
│  ├── Multiple AZs     → High availability                           │
│  └── Multiple regions → Global latency optimization                │
│                                                                     │
│  Replication lag tolerance?                                         │
│  ├── Real-time required → Read from primary                        │
│  ├── Few seconds OK     → Normal async replication                 │
│  └── Minutes OK         → Can use more aggressive caching          │
└─────────────────────────────────────────────────────────────────────┘
```

### Sharding Pattern

**Sharding Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│                 SHARDING DECISION FRAMEWORK                         │
│                                                                     │
│  Do you REALLY need sharding?                                       │
│  ├── Database < 1 TB              → NO, not yet                    │
│  ├── Single server can handle     → NO, optimize first            │
│  ├── Read replicas sufficient     → NO, try replicas first        │
│  └── Truly need horizontal scale  → YES, consider sharding        │
│                                                                     │
│  Warning: Sharding adds massive complexity!                         │
│  - Cross-shard queries difficult                                    │
│  - Transactions across shards impossible                            │
│  - Rebalancing shards is painful                                    │
│  - Application logic becomes complex                                │
│                                                                     │
│  Choose sharding key carefully (cannot change easily):              │
│                                                                     │
│  What is your primary access pattern?                               │
│  ├── By user (social app)         → Shard by user_id              │
│  ├── By tenant (SaaS)              → Shard by tenant_id            │
│  ├── By geography                  → Shard by region               │
│  └── By time (logs)                → Shard by date range           │
│                                                                     │
│  Sharding Strategy Decision:                                        │
│                                                                     │
│  Range-Based Sharding:                                              │
│  ├── user_id 1-1M      → Shard 0                                   │
│  ├── user_id 1M-2M     → Shard 1                                   │
│  └── user_id 2M-3M     → Shard 2                                   │
│  Pros: Easy to add shards, range queries work                      │
│  Cons: Hotspots (new users all in latest shard)                   │
│                                                                     │
│  Hash-Based Sharding:                                               │
│  ├── hash(user_id) % 4 = 0 → Shard 0                              │
│  ├── hash(user_id) % 4 = 1 → Shard 1                              │
│  └── hash(user_id) % 4 = 2 → Shard 2                              │
│  Pros: Even distribution                                            │
│  Cons: Hard to add shards (requires rehashing all data!)          │
│                                                                     │
│  Consistent Hashing (Best for dynamic shards):                     │
│  └── Minimizes data movement when adding/removing shards           │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation Example:**
```javascript
// Sharding implementation
class ShardedDatabase {
  constructor() {
    this.shards = [
      new Pool({ host: 'shard0.db.com', port: 5432 }),
      new Pool({ host: 'shard1.db.com', port: 5432 }),
      new Pool({ host: 'shard2.db.com', port: 5432 }),
      new Pool({ host: 'shard3.db.com', port: 5432 })
    ];
  }
  
  // Determine which shard to use
  getShardForUser(userId) {
    const shardIndex = userId % this.shards.length;
    return this.shards[shardIndex];
  }
  
  async getUser(userId) {
    const shard = this.getShardForUser(userId);
    return shard.query('SELECT * FROM users WHERE id = $1', [userId]);
  }
  
  async getUserOrders(userId) {
    // ✅ GOOD: Orders sharded same way as users
    const shard = this.getShardForUser(userId);
    return shard.query(
      'SELECT * FROM orders WHERE user_id = $1',
      [userId]
    );
    // Works! User and their orders on same shard
  }
  
  async getOrderStats() {
    // ❌ PROBLEM: Need to query ALL shards
    const results = await Promise.all(
      this.shards.map(shard => 
        shard.query('SELECT COUNT(*) as total FROM orders')
      )
    );
    
    // Aggregate in application code
    const total = results.reduce((sum, r) => sum + r.rows[0].total, 0);
    return { total };
    // Slow! Must query all 4 shards
  }
  
  async getTotalRevenue() {
    // ❌ PROBLEM: Cross-shard aggregation
    // Solution: Replicate aggregated data to separate analytics DB
  }
}
```

**Sharding Challenges:**
```markdown
Challenge 1: Cross-Shard Queries
Problem: "Find all orders with total > $1000"
         Must query ALL shards, aggregate in app
Solution: 
- Maintain separate analytics database
- Use map-reduce pattern
- Pre-aggregate important metrics

Challenge 2: Distributed Transactions
Problem: Transfer money user A (shard 0) → user B (shard 1)
         Can't use database ACID across shards!
Solution:
- Saga pattern (compensating transactions)
- Eventual consistency
- Two-phase commit (slow, avoid if possible)

Challenge 3: Joins Across Shards
Problem: JOIN users (sharded by user_id) with products (sharded differently)
Solution:
- Denormalize data
- Application-level joins
- Shard related data together

Challenge 4: Rebalancing
Problem: Shard 0 has 2M users, Shard 1 has 100K users (unbalanced!)
Solution:
- Consistent hashing (minimizes moves)
- Add shards during low-traffic periods
- Plan capacity ahead of time
```

---

## Real-World Use Cases

### Use Case 1: Social Media Platform

**Requirements:**
```
- 100M+ users
- 1B+ posts
- Real-time feed generation
- User connections (friends/followers)
- High write volume (posts, likes, comments)
- Global distribution
```

**Database Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│            SOCIAL MEDIA DATABASE ARCHITECTURE                       │
│                                                                     │
│  User Profiles & Auth:                                              │
│  ├── Database: PostgreSQL                                           │
│  ├── Why: ACID for auth, complex queries for user management       │
│  └── Scaling: Shard by user_id                                     │
│                                                                     │
│  Posts & Comments:                                                   │
│  ├── Database: Cassandra (NoSQL)                                    │
│  ├── Why: High write volume, horizontal scaling, eventual OK       │
│  └── Sharding: By user_id (posts with their author)                │
│                                                                     │
│  Social Graph (friends/followers):                                  │
│  ├── Database: Neo4j (Graph)                                        │
│  ├── Why: Efficient friend-of-friend queries, recommendations      │
│  └── Alternative: Redis (if simple following only)                 │
│                                                                     │
│  Timeline/Feed:                                                      │
│  ├── Database: Redis (Cache)                                        │
│  ├── Why: Fast reads, TTL for recent posts only                    │
│  └── Backing store: Cassandra for complete history                 │
│                                                                     │
│  Analytics:                                                          │
│  ├── Database: ClickHouse (Column-store)                           │
│  ├── Why: Fast aggregations, trending topics                       │
│  └── Data pipeline: Stream from Cassandra via Kafka                │
│                                                                     │
│  Search:                                                             │
│  ├── Database: Elasticsearch                                        │
│  ├── Why: Full-text search for posts, users, hashtags             │
│  └── Sync: Real-time indexing from Cassandra                       │
└─────────────────────────────────────────────────────────────────────┘

Data Flow:
User posts → Cassandra (durable) 
          → Kafka (stream)
          → Redis (feed cache)
          → Elasticsearch (search index)
          → ClickHouse (analytics)
```

### Use Case 2: E-commerce Platform

**Requirements:**
```
- Product catalog (millions of SKUs)
- Inventory management (real-time stock)
- Order processing (ACID critical)
- Search & filters
- Personalized recommendations
```

**Database Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│            E-COMMERCE DATABASE ARCHITECTURE                         │
│                                                                     │
│  Orders & Payments:                                                  │
│  ├── Database: PostgreSQL                                           │
│  ├── Why: ACID mandatory for financial transactions                │
│  ├── Pattern: Sharded by user_id (user's orders together)         │
│  └── Scale: Read replicas for order history queries                │
│                                                                     │
│  Inventory:                                                          │
│  ├── Database: PostgreSQL + Redis                                   │
│  ├── Why: ACID for stock updates, Redis for fast reads             │
│  ├── Pattern:                                                        │
│  │   - PostgreSQL: Source of truth                                  │
│  │   - Redis: Cache with TTL for product availability              │
│  └── Critical: Use transactions for inventory decrements!          │
│                                                                     │
│  Product Catalog:                                                    │
│  ├── Database: MongoDB                                              │
│  ├── Why: Flexible schema (different product types)                │
│  ├── Example: Electronics have specs, Clothing has sizes           │
│  └── Scale: Shard by category or product_id                        │
│                                                                     │
│  Product Search:                                                     │
│  ├── Database: Elasticsearch                                        │
│  ├── Why: Full-text search, faceted filters, typo tolerance       │
│  └── Sync: Index from MongoDB via change streams                   │
│                                                                     │
│  User Sessions:                                                      │
│  ├── Database: Redis                                                │
│  ├── Why: Fast, TTL for carts and sessions                        │
│  └── Pattern: Cart persistence with 24h TTL                        │
│                                                                     │
│  Recommendations:                                                    │
│  ├── Database: Neo4j or specialized (Amazon Personalize)          │
│  ├── Why: Graph for "users who bought X also bought Y"            │
│  └── Alternative: Pre-computed in ClickHouse, served from Redis    │
└─────────────────────────────────────────────────────────────────────┘
```

**Critical Transaction Example:**
```sql
-- Order placement (MUST be ACID)
BEGIN TRANSACTION;

-- Lock inventory row
SELECT stock FROM inventory 
WHERE product_id = 123 
FOR UPDATE;

-- Check stock available
IF stock >= quantity THEN
  -- Create order
  INSERT INTO orders (...) VALUES (...);
  
  -- Decrease inventory
  UPDATE inventory 
  SET stock = stock - quantity 
  WHERE product_id = 123;
  
  -- Create payment record
  INSERT INTO payments (...) VALUES (...);
  
  COMMIT;
ELSE
  ROLLBACK;
  RAISE EXCEPTION 'Out of stock';
END IF;

-- Either ALL succeed or ALL fail
-- No partial orders!
```

### Use Case 3: Analytics Platform

**Requirements:**
```
- Billions of events per day
- Real-time dashboards
- Historical queries (years of data)
- Complex aggregations
- Time-series data
```

**Database Decision:**
```markdown
┌─────────────────────────────────────────────────────────────────────┐
│            ANALYTICS PLATFORM DATABASE ARCHITECTURE                 │
│                                                                     │
│  Real-Time Events:                                                   │
│  ├── Ingestion: Apache Kafka                                        │
│  ├── Why: High throughput, durable, replay capability              │
│  └── Pattern: Topic per event type                                 │
│                                                                     │
│  Hot Data (Recent 7 days):                                          │
│  ├── Database: ClickHouse                                           │
│  ├── Why: Blazing fast aggregations, column-store                  │
│  ├── Retention: 7 days in ClickHouse                               │
│  └── Query: Real-time dashboards hit this                          │
│                                                                     │
│  Warm Data (8-90 days):                                             │
│  ├── Database: TimescaleDB (PostgreSQL extension)                  │
│  ├── Why: SQL interface, good compression, slower OK               │
│  └── Use: Weekly/monthly reports                                   │
│                                                                     │
│  Cold Data (>90 days):                                              │
│  ├── Storage: S3 (Parquet format)                                  │
│  ├── Query: AWS Athena (serverless SQL on S3)                      │
│  ├── Why: Cheapest storage, query on-demand                        │
│  └── Use: Compliance, historical analysis                          │
│                                                                     │
│  Data Pipeline:                                                      │
│  Kafka → ClickHouse (real-time) ───┐                               │
│       → TimescaleDB (hourly batch) │                               │
│       → S3 (daily archive) ─────────┴→ Athena (query as needed)    │
│                                                                     │
│  Cost Optimization:                                                  │
│  - Hot data (7 days): $$$$ (fast, expensive)                       │
│  - Warm data (90 days): $$ (medium cost)                           │
│  - Cold data (years): $ (cheap storage)                            │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Quick Reference

### Database Selection Cheat Sheet

```markdown
Choose PostgreSQL when:
✅ Complex queries with JOINs
✅ ACID transactions required
✅ Mixed workload (OLTP + light analytics)
✅ Need both relational AND JSON data
✅ Budget for managed service (RDS, Cloud SQL)

Choose MongoDB when:
✅ Schema changes frequently
✅ Document-centric data (user profiles, catalogs)
✅ Rapid development (no migrations)
✅ Horizontal scaling needed
✅ JSON-first API

Choose Redis when:
✅ Caching layer
✅ Session storage
✅ Real-time leaderboards
✅ Rate limiting
✅ Pub/sub messaging

Choose Cassandra when:
✅ Massive write volume
✅ Time-series data
✅ High availability critical
✅ Linear scalability needed
✅ Eventual consistency acceptable

Choose Neo4j when:
✅ Graph relationships are primary
✅ Social networks
✅ Recommendation engines
✅ Fraud detection
✅ Complex relationship queries
```

### Common Mistakes to Avoid

```markdown
❌ Mistake 1: Over-indexing
Problem: Adding index on every column
Impact: Slow writes, wasted storage
Solution: Index only frequently queried columns

❌ Mistake 2: Premature sharding
Problem: Sharding with 100GB database
Impact: Unnecessary complexity
Solution: Try vertical scaling + replicas first

❌ Mistake 3: Using wrong database type
Problem: Storing graphs in SQL, transactions in NoSQL
Impact: Performance suffers, complex code
Solution: Choose right tool for the job

❌ Mistake 4: No caching layer
Problem: Every request hits database
Impact: Database overload
Solution: Add Redis cache for hot data

❌ Mistake 5: Ignoring query patterns
Problem: Schema doesn't match access patterns
Impact: Slow queries, full table scans
Solution: Design schema based on how you'll query

❌ Mistake 6: No monitoring
Problem: Don't know when DB is struggling
Impact: Surprise outages
Solution: Monitor query times, connection pools, disk space
```

### Performance Optimization Checklist

```markdown
Before scaling, optimize:
□ Add appropriate indexes
□ Analyze slow queries (EXPLAIN)
□ Remove unused indexes
□ Optimize table schema
□ Add caching layer (Redis)
□ Use connection pooling
□ Implement query result caching
□ Archive old data
□ Partition large tables
□ Update database statistics

Then consider scaling:
□ Vertical scaling (more RAM/CPU)
□ Read replicas (if read-heavy)
□ Sharding (if > 1TB and optimized)
□ Different database type (if fundamentally mismatched)
```

---

This comprehensive database guide provides decision trees and design considerations for choosing, implementing, and scaling databases in production systems.