# Azure Databases: SQL & Cosmos DB
## (analogous to AWS RDS & DynamoDB)

Azure offers managed relational and NoSQL database services. The two most widely used are **Azure SQL Database** (managed SQL Server) and **Azure Cosmos DB** (globally distributed multi-model NoSQL).

---

## Part 1: Azure SQL Database
## (analogous to AWS RDS for SQL Server / Aurora)

Azure SQL Database is a fully managed relational database built on SQL Server. Azure handles patching, backups, HA, and scaling — you just connect and query.

> Also available: **Azure Database for PostgreSQL** and **Azure Database for MySQL** — same managed model, different engines.

---

### Deployment Models

| Model | Description | AWS Equivalent |
|-------|-------------|----------------|
| **Single Database** | Isolated database with its own resources | RDS Single-AZ / Multi-AZ |
| **Elastic Pool** | Multiple databases sharing a resource pool | RDS Multi-tenant with shared storage |
| **Managed Instance** | Near 100% SQL Server compatibility, VNet-native | RDS Custom / SQL Server on EC2 |

---

### Creating a SQL Database

```bash
# Create a logical server (the container for databases)
az sql server create \
  --resource-group myRG \
  --name my-sql-server \
  --location eastus \
  --admin-user sqladmin \
  --admin-password "MyP@ssword123!"

# Create a database on that server
az sql db create \
  --resource-group myRG \
  --server my-sql-server \
  --name myDatabase \
  --service-objective S2 \
  --backup-storage-redundancy Geo

# Allow your client IP to connect (firewall rule)
az sql server firewall-rule create \
  --resource-group myRG \
  --server my-sql-server \
  --name AllowMyIP \
  --start-ip-address 203.0.113.5 \
  --end-ip-address 203.0.113.5

# Allow Azure services to connect
az sql server firewall-rule create \
  --resource-group myRG \
  --server my-sql-server \
  --name AllowAzureServices \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

---

### Service Tiers (Purchasing Models)

**DTU model** (simple, bundled compute + I/O + storage):

| Tier | DTUs | Use Case |
|------|------|----------|
| Basic | 5 | Dev/test |
| Standard S0–S12 | 10–3000 | General workloads |
| Premium P1–P15 | 125–4000 | OLTP, high I/O |

**vCore model** (more control, analogous to RDS instance types):

```bash
az sql db create \
  --resource-group myRG \
  --server my-sql-server \
  --name myDatabase \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 4  # 4 vCores
```

---

### Connecting (Node.js)

```bash
npm install mssql
```

```javascript
const sql = require("mssql");
const { DefaultAzureCredential } = require("@azure/identity");

// Option 1: With Entra ID (managed identity — no password)
const credential = new DefaultAzureCredential();
const token = await credential.getToken("https://database.windows.net/.default");

const pool = await sql.connect({
  server: "my-sql-server.database.windows.net",
  database: "myDatabase",
  authentication: {
    type: "azure-active-directory-access-token",
    options: { token: token.token },
  },
  options: { encrypt: true },
});

// Option 2: Username/password
const pool = await sql.connect({
  server: "my-sql-server.database.windows.net",
  database: "myDatabase",
  user: "sqladmin",
  password: process.env.SQL_PASSWORD,
  options: { encrypt: true },
});

// Query
const result = await pool.request()
  .input("userId", sql.Int, 42)
  .query("SELECT * FROM users WHERE id = @userId");

console.log(result.recordset);
```

---

### Backup & High Availability

```bash
# Point-in-time restore (automated backups kept 7–35 days)
az sql db restore \
  --resource-group myRG \
  --server my-sql-server \
  --name myDatabase-restored \
  --source-database myDatabase \
  --time "2025-01-15T12:00:00Z"

# Geo-restore (restore to a different region from geo-redundant backup)
az sql db restore \
  --resource-group myRG-westus \
  --server my-sql-server-westus \
  --name myDatabase-georestored \
  --source-database-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Sql/servers/my-sql-server/databases/myDatabase \
  --geo-backup-id <backup-id>
```

**Active Geo-Replication** (analogous to RDS Read Replicas with failover):

```bash
az sql db replica create \
  --resource-group myRG \
  --server my-sql-server \
  --name myDatabase \
  --partner-server my-sql-server-westus \
  --partner-resource-group myRG-westus
```

---

### Private Connectivity

```bash
# Create a private endpoint for SQL (no public internet traffic)
az network private-endpoint create \
  --resource-group myRG \
  --name mySQLPrivateEndpoint \
  --vnet-name myVNet \
  --subnet dataSubnet \
  --private-connection-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Sql/servers/my-sql-server \
  --group-id sqlServer \
  --connection-name mySQLConnection
```

---

## Part 2: Azure Cosmos DB
## (analogous to AWS DynamoDB)

Cosmos DB is a globally distributed, multi-model NoSQL database with single-digit millisecond reads. You choose an API at creation time:

| API | Model | AWS Equivalent |
|-----|-------|----------------|
| **NoSQL (Core)** | JSON documents with SQL-like query | DynamoDB |
| **MongoDB** | MongoDB wire protocol compatible | DocumentDB |
| **PostgreSQL** | Distributed PostgreSQL (via Citus) | Aurora PostgreSQL |
| **Cassandra** | CQL-compatible column store | Keyspaces |
| **Gremlin** | Graph database | Neptune |
| **Table** | Key-value, Azure Table Storage compatible | DynamoDB simple key-value |

---

### Creating a Cosmos DB Account

```bash
# Create Cosmos DB account (NoSQL API)
az cosmosdb create \
  --resource-group myRG \
  --name mycosmosaccount \
  --kind GlobalDocumentDB \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false

# Create a database
az cosmosdb sql database create \
  --resource-group myRG \
  --account-name mycosmosaccount \
  --name myDatabase

# Create a container (partition key is REQUIRED)
az cosmosdb sql container create \
  --resource-group myRG \
  --account-name mycosmosaccount \
  --database-name myDatabase \
  --name users \
  --partition-key-path "/tenantId" \
  --throughput 400
```

---

### Partition Key Design

The partition key is the most important design decision — it determines how data is distributed across physical partitions. A good key:

- Has high cardinality (many distinct values)
- Distributes reads and writes evenly
- Is present in most queries (avoids cross-partition fan-out)

| Pattern | Good Key Example | Bad Key Example |
|---------|-----------------|-----------------|
| Multi-tenant app | `/tenantId` | `/country` (few values) |
| IoT device data | `/deviceId` | `/sensorType` |
| E-commerce orders | `/customerId` | `/orderStatus` |

---

### Node.js Usage

```javascript
const { CosmosClient } = require("@azure/cosmos");
const { DefaultAzureCredential } = require("@azure/identity");

// Using managed identity (preferred)
const client = new CosmosClient({
  endpoint: "https://mycosmosaccount.documents.azure.com",
  aadCredentials: new DefaultAzureCredential(),
});

const container = client.database("myDatabase").container("users");

// Create/upsert an item
await container.items.upsert({
  id: "user-123",          // required
  tenantId: "acme",        // partition key — required
  name: "Priya Sharma",
  email: "priya@acme.com",
  createdAt: new Date().toISOString(),
});

// Read a single item (fast — point read, 1 RU)
const { resource } = await container.item("user-123", "acme").read();
console.log(resource.name);

// Query (cross-partition unless filter includes partition key)
const { resources } = await container.items
  .query({
    query: "SELECT * FROM c WHERE c.tenantId = @tenant AND c.role = @role",
    parameters: [
      { name: "@tenant", value: "acme" },
      { name: "@role", value: "admin" },
    ],
  })
  .fetchAll();

// Delete an item
await container.item("user-123", "acme").delete();
```

---

### Throughput: RU/s (Request Units)

Cosmos DB bills on **Request Units (RU/s)** — a normalized measure of compute, I/O, and memory per operation.

| Operation | Approximate RU Cost |
|-----------|-------------------|
| Point read (1 KB item) | 1 RU |
| Write (1 KB item) | ~5 RUs |
| Query (per 1 KB result) | ~2.5 RUs |
| Cross-partition query | Higher (fan-out) |

```bash
# Autoscale (0–4000 RU/s, charged for max reached in each hour)
az cosmosdb sql container create \
  --resource-group myRG \
  --account-name mycosmosaccount \
  --database-name myDatabase \
  --name events \
  --partition-key-path "/deviceId" \
  --max-throughput 4000  # autoscale up to 4000 RU/s
```

---

### Global Distribution

```bash
# Add a read region (replicate data globally, low-latency reads)
az cosmosdb update \
  --resource-group myRG \
  --name mycosmosaccount \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=false \
                regionName=westeurope failoverPriority=1 isZoneRedundant=false \
                regionName=southeastasia failoverPriority=2 isZoneRedundant=false
```

---

### Change Feed (analogous to DynamoDB Streams)

Change feed captures all inserts and updates (not deletes by default) in order per partition key. Ideal for event-driven architectures.

```javascript
const { ChangeFeedStartFrom } = require("@azure/cosmos");

const iterator = container.items.getChangeFeedIterator({
  changeFeedStartFrom: ChangeFeedStartFrom.Beginning(),
});

for await (const { result } of iterator) {
  for (const item of result) {
    console.log("Changed item:", item.id);
  }
}
```

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Managed relational DB | RDS | Azure SQL Database |
| PostgreSQL managed | RDS PostgreSQL / Aurora | Azure Database for PostgreSQL |
| NoSQL document DB | DynamoDB | Cosmos DB (NoSQL API) |
| Global distribution | DynamoDB Global Tables | Cosmos DB multi-region (built-in) |
| NoSQL capacity unit | RCU/WCU | RU/s (unified read+write) |
| NoSQL streams | DynamoDB Streams | Cosmos DB Change Feed |
| Multi-model NoSQL | Multiple services | Cosmos DB (one account, multiple APIs) |
| Serverless NoSQL | DynamoDB on-demand | Cosmos DB serverless |