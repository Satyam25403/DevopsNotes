# AWS Databases

AWS offers purpose-built databases for every workload type — relational, NoSQL, in-memory, graph, time-series, and more. Choosing the right one is key to building performant, cost-efficient applications.

---

## Quick Reference

| Database | Type | Best For |
|----------|------|----------|
| RDS | Relational (SQL) | Traditional apps with structured data |
| Aurora | Relational (cloud-native) | High-performance MySQL/PostgreSQL |
| DynamoDB | NoSQL (key-value + document) | High-speed, serverless, scalable |
| Redshift | Data warehouse | Analytics, BI, large SQL queries |
| ElastiCache | In-memory cache | Speed boost, session storage |
| DocumentDB | Document (MongoDB-compatible) | JSON document storage |
| Neptune | Graph | Relationships, fraud detection |
| Timestream | Time-series | IoT, monitoring, metrics |
| QLDB | Ledger | Immutable audit trails |
| Keyspaces | Wide-column (Cassandra) | Large-scale, low-latency |

---

## 1. Amazon RDS (Relational Database Service)

Fully managed relational database — AWS handles backups, patching, failover, and scaling.

**Supported engines:** MySQL, PostgreSQL, MariaDB, Oracle, SQL Server, Amazon Aurora

**Key Features:**
- Automated backups and point-in-time recovery
- Multi-AZ deployments for high availability (automatic failover)
- Read replicas for read scaling
- Encryption at rest and in transit

**Connecting to RDS:**
Once created, RDS provides an **endpoint URL**. Use it in your app:

```javascript
// Node.js (with pg for PostgreSQL)
const { Pool } = require('pg');
const pool = new Pool({
  host: 'mydb.abcdef.us-east-1.rds.amazonaws.com',
  port: 5432,
  database: 'myapp',
  user: 'admin',
  password: process.env.DB_PASSWORD,
});
```

```python
# Python (with psycopg2)
import psycopg2
conn = psycopg2.connect(
    host="mydb.abcdef.us-east-1.rds.amazonaws.com",
    database="myapp",
    user="admin",
    password=os.environ['DB_PASSWORD']
)
```

**Troubleshooting connection issues:**
If your app can't connect, go to the RDS instance → **Security Group** → add an inbound rule allowing TCP on port 5432 (PostgreSQL) or 3306 (MySQL) from your EC2's security group or your IP.

**GUI tool:** Use [TablePlus](https://tableplus.com/) or DBeaver — connect with the RDS endpoint, username, and password to browse your database visually.

**Useful CLI commands:**
```bash
# List RDS instances
aws rds describe-db-instances \
  --query "DBInstances[*].[DBInstanceIdentifier,Endpoint.Address,DBInstanceStatus]" \
  --output table

# Create a snapshot (manual backup)
aws rds create-db-snapshot \
  --db-instance-identifier my-db \
  --db-snapshot-identifier my-db-snapshot-$(date +%Y%m%d)
```

---

## 2. Amazon Aurora

AWS's cloud-native relational database — compatible with MySQL and PostgreSQL but significantly faster and more scalable.

**Highlights vs standard RDS:**
- Up to 5x faster than MySQL, 3x faster than PostgreSQL
- Storage automatically grows in 10 GB increments, up to 128 TB
- Up to 15 read replicas (vs 5 for standard RDS)
- Aurora Serverless v2: automatically scales capacity up/down to zero based on load
- Global Database: replicate across multiple AWS regions with < 1 second replication lag

**Use Aurora when:** you need MySQL/PostgreSQL compatibility but with higher performance and better scalability than standard RDS.

---

## 3. Amazon DynamoDB

Fully managed, serverless NoSQL database. Supports key-value and document data models.

**Key Highlights:**
- **Serverless**: No capacity to manage in on-demand mode
- **Millisecond latency** at any scale — millions of requests/second
- **Flexible schema**: Each item can have different attributes
- Global Tables for multi-region, active-active replication
- DynamoDB Streams for change data capture (event-driven patterns)
- Point-in-time recovery (PITR) and encryption at rest

**Core Concepts:**

| Concept | Description |
|---------|-------------|
| **Table** | Top-level container (like a SQL table, but schema-less) |
| **Item** | A single record (like a SQL row) |
| **Attribute** | A field within an item (like a SQL column, but flexible) |
| **Partition Key** | Primary key used to distribute data across partitions |
| **Sort Key** | Optional secondary key for ordering items within a partition |
| **GSI** | Global Secondary Index — query on non-primary-key attributes |

**Node.js example with AWS SDK v3:**
```javascript
const { DynamoDBClient, PutItemCommand, GetItemCommand, QueryCommand } = require("@aws-sdk/client-dynamodb");
const { DynamoDBDocumentClient, PutCommand, GetCommand } = require("@aws-sdk/lib-dynamodb");

const client = new DynamoDBClient({ region: "us-east-1" });
const docClient = DynamoDBDocumentClient.from(client); // easier to use

// Put item
await docClient.send(new PutCommand({
  TableName: "Users",
  Item: { userId: "user123", name: "Alice", age: 30 }
}));

// Get item
const result = await docClient.send(new GetCommand({
  TableName: "Users",
  Key: { userId: "user123" }
}));
console.log(result.Item);
```

**CLI example:**
```bash
# Create a table
aws dynamodb create-table \
  --table-name Users \
  --attribute-definitions AttributeName=userId,AttributeType=S \
  --key-schema AttributeName=userId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Put an item
aws dynamodb put-item \
  --table-name Users \
  --item '{"userId": {"S": "user123"}, "name": {"S": "Alice"}}'

# Get an item
aws dynamodb get-item \
  --table-name Users \
  --key '{"userId": {"S": "user123"}}'
```

**Capacity Modes:**
- **On-demand**: Pay per request — great for unpredictable workloads
- **Provisioned**: Set read/write capacity units — cheaper at consistent, predictable load

---

## 4. Amazon Redshift

Managed **data warehouse** service optimized for running complex analytical SQL queries across massive datasets (terabytes to petabytes).

**Key Highlights:**
- Columnar storage — highly efficient for aggregation queries
- Massively parallel processing (MPP)
- Integrates with S3 (via Redshift Spectrum — query S3 data directly)
- Redshift Serverless option available
- Common with BI tools: Tableau, Looker, QuickSight

**Use Redshift when:** you need to run complex analytical queries across large historical datasets — NOT for transactional (OLTP) workloads.

---

## 5. Amazon ElastiCache

Fully managed **in-memory caching** service. Dramatically reduces database load and improves application response times.

**Supported engines:**
- **Redis**: Rich data structures, persistence, pub/sub, Lua scripting, clustering
- **Memcached**: Simple caching, multi-threaded, horizontally scalable

**Common use cases:**
- Session storage
- Database query result caching
- Rate limiting
- Real-time leaderboards (Redis sorted sets)
- Pub/sub messaging

**Node.js example (with ioredis):**
```javascript
const Redis = require('ioredis');
const redis = new Redis({
  host: 'my-cluster.abc.cache.amazonaws.com',
  port: 6379,
});

// Cache a DB result
const cached = await redis.get('user:123');
if (cached) return JSON.parse(cached);

const user = await db.findUser(123);
await redis.setex('user:123', 3600, JSON.stringify(user)); // cache for 1 hour
return user;
```

---

## 6. Amazon DocumentDB

Managed **document database** compatible with MongoDB APIs and drivers. Designed for JSON-like document storage at scale.

**Use DocumentDB when:** you're migrating a MongoDB workload to AWS and want a fully managed option without self-managing MongoDB on EC2.

> Note: DocumentDB is not 100% MongoDB-compatible — check your MongoDB version and features before migrating.

---

## 7. Amazon Neptune

Fully managed **graph database** supporting two query models:
- **Property Graph** with Gremlin or openCypher
- **RDF** with SPARQL

**Use cases:** social networks, recommendation engines, fraud detection, knowledge graphs, identity graphs.

---

## 8. Amazon Timestream

Fully managed **time-series database** optimized for storing and querying data that changes over time.

**Use cases:** IoT sensor data, application metrics, clickstream analysis, real-time monitoring.

Features automatic data tiering — recent data in memory, historical data in magnetic storage.

---

## 9. Amazon QLDB (Quantum Ledger Database)

Fully managed **ledger database** providing a cryptographically verifiable, immutable transaction log.

**Use cases:** financial transactions, supply chain tracking, compliance audit trails, healthcare records.

Every change is recorded and can be mathematically verified — no one (including AWS) can alter historical records.

---

## 10. Amazon Keyspaces

Managed **Apache Cassandra-compatible** database service. No Cassandra cluster to manage.

**Use cases:** large-scale applications with wide-column data models, high write throughput, multi-region workloads.

---

## Choosing the Right Database

```
Need SQL / transactions?
  → High performance needed?  → Yes: Aurora  |  No: RDS

Need NoSQL?
  → Document/key-value at scale?  → DynamoDB
  → MongoDB workloads?            → DocumentDB

Need analytics / warehousing?    → Redshift

Need to cache / speed up DB?     → ElastiCache (Redis or Memcached)

Need graph relationships?        → Neptune

Need time-series data?           → Timestream

Need immutable audit log?        → QLDB

Need Cassandra?                  → Keyspaces
```

---

## Security Best Practices

- **Never expose RDS/DynamoDB directly to the internet** — place them in private subnets.
- **Use IAM roles** (not hardcoded credentials) for application access to DynamoDB.
- **Store DB passwords in SSM Parameter Store or Secrets Manager**, not in environment variables or code.
- **Enable Multi-AZ** for RDS in production for automatic failover.
- **Enable encryption at rest** on all databases.
- **Enable automated backups** and test restoration periodically.