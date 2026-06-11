# GCP Databases

GCP offers purpose-built databases for every workload type — relational, NoSQL, in-memory, analytical, and more. Choosing the right one is key to building performant, cost-efficient applications.

---

## Quick Reference

| Database | Type | Best For |
|----------|------|----------|
| Cloud SQL | Relational (SQL) | Traditional apps with MySQL, PostgreSQL, SQL Server |
| Cloud Spanner | Relational (cloud-native) | Global, horizontally scalable relational DB |
| Firestore | NoSQL (document) | Real-time apps, mobile/web backends |
| Bigtable | NoSQL (wide-column) | High-throughput analytics, time-series, IoT |
| BigQuery | Data warehouse | Analytics, BI, large SQL queries |
| Memorystore | In-memory cache | Speed boost, session storage (Redis/Valkey) |
| AlloyDB | PostgreSQL-compatible | High-performance PostgreSQL workloads |
| Datastore | NoSQL (document, legacy) | Existing Datastore apps (Firestore successor) |

---

## 1. Cloud SQL (Relational — MySQL / PostgreSQL / SQL Server)

Fully managed relational database — GCP handles backups, patching, failover, and scaling.

**Supported engines:** MySQL 8.0, PostgreSQL 15/16, SQL Server 2019/2022

**Key Features:**
- Automatic backups and point-in-time recovery (PITR)
- High availability with automatic failover to a standby replica
- Read replicas for scaling reads
- Private IP via VPC (recommended) or Cloud SQL Auth Proxy for secure connections
- Automatic storage increase

```bash
# Create a Cloud SQL PostgreSQL instance
gcloud sql instances create my-postgres \
  --database-version=POSTGRES_16 \
  --tier=db-n1-standard-2 \
  --region=us-central1 \
  --storage-auto-increase \
  --backup-start-time=02:00 \
  --availability-type=REGIONAL    # HA with automatic failover

# Create a database
gcloud sql databases create mydb --instance=my-postgres

# Create a user
gcloud sql users create myuser \
  --instance=my-postgres \
  --password=supersecret

# Connect via Cloud SQL Auth Proxy (secure, no public IP needed)
cloud-sql-proxy my-project:us-central1:my-postgres &
psql -h 127.0.0.1 -U myuser -d mydb

# Create a read replica
gcloud sql instances create my-postgres-replica \
  --master-instance-name=my-postgres \
  --region=us-east1
```

### Cloud SQL Auth Proxy (for app connectivity)
```bash
# Download and run the proxy
curl -o cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.8.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy
./cloud-sql-proxy --port=5432 my-project:us-central1:my-postgres

# Application connects to 127.0.0.1:5432 as if it were a local database
```

---

## 2. Cloud Spanner (Global Relational — Horizontally Scalable)

Cloud Spanner is Google's globally distributed, strongly consistent relational database. It combines the familiar SQL interface of relational databases with the horizontal scalability of NoSQL. Zero downtime maintenance.

**When to use Spanner over Cloud SQL:**
- Need > 30,000 read/write QPS (beyond a single Cloud SQL instance)
- Need multi-region or global data distribution with strong consistency
- Cannot tolerate downtime for maintenance or failover

```bash
# Create a Spanner instance
gcloud spanner instances create my-spanner \
  --config=regional-us-central1 \
  --processing-units=100 \       # 100 PU = 0.1 node (minimum)
  --description="My Spanner instance"

# Create a database
gcloud spanner databases create mydb --instance=my-spanner

# Execute DDL
gcloud spanner databases ddl update mydb \
  --instance=my-spanner \
  --ddl="CREATE TABLE Users (
    UserId STRING(36) NOT NULL,
    Name STRING(256),
    Email STRING(256),
    CreatedAt TIMESTAMP
  ) PRIMARY KEY (UserId)"

# Query
gcloud spanner databases execute-sql mydb \
  --instance=my-spanner \
  --sql="SELECT * FROM Users LIMIT 10"
```

---

## 3. Firestore (NoSQL Document Database)

Firestore is GCP's fully managed, serverless NoSQL document database — great for real-time mobile/web apps. It scales automatically to zero and has no servers to manage.

**Key features:**
- Real-time listeners (push updates to clients automatically)
- Offline support for mobile apps
- ACID transactions across multiple documents
- Strong consistency
- Native mode (recommended) vs Datastore mode (backward-compatible)

```bash
# Firestore is provisioned per project — no "create instance" command
# Enable via:
gcloud services enable firestore.googleapis.com

# Firestore is best accessed via the Node.js/Python SDK (see SDK doc)
# CLI for data export/import:
gcloud firestore export gs://my-backup-bucket/firestore-export \
  --collection-ids=users,orders

gcloud firestore import gs://my-backup-bucket/firestore-export
```

---

## 4. BigQuery (Data Warehouse)

BigQuery is GCP's fully managed, serverless data warehouse for analytics. It can query terabytes in seconds using familiar SQL — no infrastructure to manage.

**Key features:**
- Serverless: no clusters to provision
- Columnar storage for fast analytical queries
- Streaming inserts + batch loads
- Built-in ML (BigQuery ML — train models with SQL)
- Integrates with Looker, Data Studio, and Vertex AI

```bash
# Create a dataset
bq mk --dataset my-project:my_dataset

# Load data from Cloud Storage
bq load \
  --source_format=CSV \
  --autodetect \
  my-project:my_dataset.my_table \
  gs://my-bucket/data.csv

# Run a query
bq query --use_legacy_sql=false \
  'SELECT user_id, COUNT(*) as orders
   FROM `my-project.my_dataset.orders`
   WHERE created_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 DAY)
   GROUP BY user_id
   ORDER BY orders DESC
   LIMIT 10'

# Export query results to Cloud Storage
bq extract \
  --destination_format=CSV \
  my-project:my_dataset.my_table \
  gs://my-bucket/export-*.csv
```

---

## 5. Memorystore (In-Memory Cache — Redis / Valkey)

Fully managed Redis or Valkey (open-source Redis fork) for caching, session storage, and leaderboards.

```bash
# Create a Redis instance
gcloud redis instances create my-redis \
  --size=2 \                     # 2 GB
  --region=us-central1 \
  --tier=STANDARD_HA \           # High availability with failover
  --redis-version=redis_7_0

# Get connection details
gcloud redis instances describe my-redis --region=us-central1

# Connect (must be in the same VPC)
redis-cli -h REDIS_HOST -p 6379

# Create a Valkey instance (Memorystore for Valkey)
gcloud memorystore instances create my-valkey \
  --location=us-central1 \
  --node-type=SHARED_CORE_NANO \
  --replica-count=1
```

---

## 6. AlloyDB (High-Performance PostgreSQL)

AlloyDB is Google's fully managed PostgreSQL-compatible database — built for demanding enterprise workloads with up to 4x faster transactional throughput than standard PostgreSQL.

**When to choose AlloyDB over Cloud SQL PostgreSQL:**
- Need higher performance from PostgreSQL
- Large-scale OLTP workloads
- Need built-in ML integration (AlloyDB AI)

```bash
# Create an AlloyDB cluster
gcloud alloydb clusters create my-cluster \
  --region=us-central1 \
  --password=supersecret

# Create a primary instance
gcloud alloydb instances create my-primary \
  --instance-type=PRIMARY \
  --cpu-count=2 \
  --cluster=my-cluster \
  --region=us-central1

# Create a read pool instance
gcloud alloydb instances create my-read-pool \
  --instance-type=READ_POOL \
  --cpu-count=2 \
  --read-pool-node-count=2 \
  --cluster=my-cluster \
  --region=us-central1
```

---

## Database Selection Guide

| Requirement | Recommended DB |
|-------------|---------------|
| MySQL/PostgreSQL app migration | Cloud SQL |
| Global relational, massive scale | Cloud Spanner |
| Real-time mobile/web app | Firestore |
| Analytics / data warehouse | BigQuery |
| Caching / sessions / leaderboards | Memorystore (Redis) |
| High-perf PostgreSQL | AlloyDB |
| High-throughput time-series/IoT | Bigtable |

---

## Best Practices

- **Use Cloud SQL Auth Proxy** or **Private IP** — never expose Cloud SQL to the public internet
- **Enable automated backups and PITR** on Cloud SQL for all production instances
- **Use read replicas** to offload analytics and reporting queries from primary
- **Store connection strings and passwords** in Secret Manager, not environment variables
- **Choose REGIONAL availability type** for Cloud SQL in production (automatic HA failover)
- **Use Firestore** for new NoSQL projects — it supersedes Datastore
- **Partition BigQuery tables** by date and cluster by common query columns for cost and performance