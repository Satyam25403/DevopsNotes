# Azure Data Factory & Synapse Analytics
## (analogous to AWS Glue & Redshift / Athena)

---

## Part 1: Azure Data Factory (ADF)
## (analogous to AWS Glue + Step Functions for data)

Azure Data Factory is a fully managed ETL/ELT and data orchestration service. It moves and transforms data between 90+ connectors — databases, file stores, SaaS apps, and cloud services — without writing infrastructure code.

---

## Core Concepts

| Concept | Description | AWS Equivalent |
|---------|-------------|----------------|
| **Pipeline** | Workflow of activities (copy, transform, branch, loop) | Glue Workflow / Step Functions |
| **Activity** | A single step in a pipeline (Copy, Databricks, SQL Stored Proc) | Glue Job |
| **Dataset** | Named pointer to data (a table, file path, query) | Glue Table |
| **Linked Service** | Connection to an external service (credentials + endpoint) | Glue Connection |
| **Trigger** | What starts a pipeline (schedule, event, tumbling window) | EventBridge / CloudWatch Events |
| **Integration Runtime (IR)** | The compute that executes activities | Glue DPU |
| **Data Flow** | Visual drag-and-drop transformation (Spark under the hood) | Glue Visual ETL |

---

## Creating a Data Factory

```bash
az datafactory create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --location eastus
```

---

## Linked Services (connections)

Linked services store connection details for source and sink systems. Think of them as named credentials + endpoints.

```bash
# Create a Linked Service for Azure SQL Database
az datafactory linked-service create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --linked-service-name AzureSQLLinkedService \
  --properties '{
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=myserver.database.windows.net;Database=myDB;",
      "credential": {
        "type": "SystemAssignedManagedIdentity"
      }
    }
  }'

# Create a Linked Service for Blob Storage
az datafactory linked-service create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --linked-service-name BlobStorageLinkedService \
  --properties '{
    "type": "AzureBlobStorage",
    "typeProperties": {
      "serviceEndpoint": "https://mystorage.blob.core.windows.net",
      "credential": { "type": "SystemAssignedManagedIdentity" }
    }
  }'
```

> **Always use Managed Identity** for linked services — no connection strings or keys to rotate.

---

## Pipelines

Pipelines are defined as JSON and can be created via the ADF Studio UI (drag-and-drop) or CLI/ARM templates.

### Copy Activity (move data from A to B)

```json
{
  "name": "CopyBlobToSQL",
  "properties": {
    "activities": [
      {
        "name": "CopyFromBlobToSQL",
        "type": "Copy",
        "inputs": [
          {
            "referenceName": "BlobSourceDataset",
            "type": "DatasetReference"
          }
        ],
        "outputs": [
          {
            "referenceName": "SQLSinkDataset",
            "type": "DatasetReference"
          }
        ],
        "typeProperties": {
          "source": {
            "type": "DelimitedTextSource",
            "storeSettings": { "type": "AzureBlobStorageReadSettings", "recursive": true }
          },
          "sink": {
            "type": "AzureSqlSink",
            "writeBehavior": "upsert",
            "upsertSettings": { "keys": ["id"] }
          },
          "enableStaging": false,
          "parallelCopies": 4
        }
      }
    ]
  }
}
```

```bash
az datafactory pipeline create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --pipeline-name CopyBlobToSQL \
  --pipeline @pipeline.json
```

---

## Triggers

```bash
# Schedule trigger (run daily at 2 AM UTC)
az datafactory trigger create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --trigger-name DailyIngestion \
  --properties '{
    "type": "ScheduleTrigger",
    "typeProperties": {
      "recurrence": {
        "frequency": "Day",
        "interval": 1,
        "startTime": "2025-01-01T02:00:00Z",
        "timeZone": "UTC"
      }
    },
    "pipelines": [{ "pipelineReference": { "type": "PipelineReference", "referenceName": "CopyBlobToSQL" } }]
  }'

# Start the trigger
az datafactory trigger start \
  --resource-group myRG \
  --factory-name my-data-factory \
  --trigger-name DailyIngestion
```

```bash
# Storage event trigger (fire when a blob lands in a container)
az datafactory trigger create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --trigger-name BlobArrivalTrigger \
  --properties '{
    "type": "BlobEventsTrigger",
    "typeProperties": {
      "blobPathBeginsWith": "/raw-data/blobs/",
      "blobPathEndsWith": ".csv",
      "events": ["Microsoft.Storage.BlobCreated"],
      "scope": "/subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage"
    },
    "pipelines": [{ "pipelineReference": { "type": "PipelineReference", "referenceName": "CopyBlobToSQL" } }]
  }'
```

---

## Run and Monitor

```bash
# Run a pipeline immediately
az datafactory pipeline create-run \
  --resource-group myRG \
  --factory-name my-data-factory \
  --pipeline-name CopyBlobToSQL

# List recent pipeline runs
az datafactory pipeline-run query-by-factory \
  --resource-group myRG \
  --factory-name my-data-factory \
  --last-updated-after 2025-01-01T00:00:00Z \
  --last-updated-before 2025-12-31T00:00:00Z

# Get activity runs for a pipeline run
az datafactory activity-run query-by-pipeline-run \
  --resource-group myRG \
  --factory-name my-data-factory \
  --run-id <pipeline-run-id> \
  --last-updated-after 2025-01-01T00:00:00Z \
  --last-updated-before 2025-12-31T00:00:00Z
```

---

## Self-Hosted Integration Runtime (access on-prem data)

When your data source is on-premises or in another cloud, you install the Self-Hosted IR on a local VM. It creates an outbound connection to ADF — no firewall inbound rules needed.

```bash
# Create the IR resource in ADF
az datafactory integration-runtime create \
  --resource-group myRG \
  --factory-name my-data-factory \
  --integration-runtime-name OnPremIR \
  --properties '{ "type": "SelfHosted" }'

# Get the auth key to install on the local machine
az datafactory integration-runtime list-auth-key \
  --resource-group myRG \
  --factory-name my-data-factory \
  --integration-runtime-name OnPremIR
```

Then download and install the Integration Runtime on the on-premises machine and register it using the auth key.

---

## Part 2: Azure Synapse Analytics
## (analogous to AWS Redshift + Athena + Glue + Lake Formation)

Azure Synapse Analytics is a unified analytics platform that brings together data warehousing, big data, and data integration in a single service. It combines:

| Capability | Description | AWS Equivalent |
|------------|-------------|----------------|
| **Dedicated SQL Pool** | Provisioned columnar data warehouse | Redshift |
| **Serverless SQL Pool** | Query files in-place with SQL, no provisioning | Athena |
| **Apache Spark Pool** | Managed Spark clusters for big data processing | Glue / EMR |
| **Synapse Pipelines** | ADF-compatible ETL pipelines (built-in) | Glue Workflows |
| **Synapse Link** | Near-real-time sync from Cosmos DB / SQL / Dataverse | Zero-ETL |

---

## Creating a Synapse Workspace

```bash
# Create a Synapse workspace
az synapse workspace create \
  --resource-group myRG \
  --name my-synapse-workspace \
  --location eastus \
  --storage-account mystorage \
  --file-system synapse-fs \
  --sql-admin-login-user synapseadmin \
  --sql-admin-login-password "MyP@ssword123!"
```

---

## Serverless SQL Pool (query files without a warehouse)

No setup needed — query CSV, Parquet, JSON, and Delta files directly from Azure Data Lake Storage using standard T-SQL. Pay per query.

```sql
-- Query a Parquet file in ADLS directly (no loading needed)
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://mystorage.dfs.core.windows.net/data/sales/year=2024/**',
    FORMAT = 'PARQUET'
) AS [result];

-- Query with schema inference and filtering
SELECT
    year(OrderDate) AS OrderYear,
    SUM(TotalAmount) AS Revenue
FROM OPENROWSET(
    BULK 'https://mystorage.dfs.core.windows.net/data/orders/*.parquet',
    FORMAT = 'PARQUET'
) WITH (
    OrderDate   DATE,
    TotalAmount DECIMAL(18,2),
    Region      NVARCHAR(50)
) AS orders
WHERE Region = 'APAC'
GROUP BY year(OrderDate)
ORDER BY OrderYear;

-- Create an external table (reusable, no data movement)
CREATE EXTERNAL TABLE dbo.SalesExternal (
    OrderId     INT,
    CustomerId  INT,
    OrderDate   DATE,
    Amount      DECIMAL(18,2)
)
WITH (
    LOCATION = 'sales/year=2024/',
    DATA_SOURCE = MyDataLake,
    FILE_FORMAT = ParquetFormat
);
```

---

## Dedicated SQL Pool (traditional data warehouse)

```bash
# Create a dedicated SQL pool (DW500c = 500 DWU)
az synapse sql pool create \
  --resource-group myRG \
  --workspace-name my-synapse-workspace \
  --name myDWH \
  --performance-level DW500c

# Pause (stop billing for compute — storage still billed)
az synapse sql pool pause \
  --resource-group myRG \
  --workspace-name my-synapse-workspace \
  --name myDWH

# Resume
az synapse sql pool resume \
  --resource-group myRG \
  --workspace-name my-synapse-workspace \
  --name myDWH
```

```sql
-- Distributed table (hash-distributed on CustomerId — good for joins)
CREATE TABLE dbo.Orders (
    OrderId     INT          NOT NULL,
    CustomerId  INT          NOT NULL,
    OrderDate   DATE,
    TotalAmount DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = HASH(CustomerId),
    CLUSTERED COLUMNSTORE INDEX
);

-- Replicated table (small dimension table copied to every node)
CREATE TABLE dbo.Products (
    ProductId   INT         NOT NULL,
    ProductName NVARCHAR(200),
    Category    NVARCHAR(100)
)
WITH (
    DISTRIBUTION = REPLICATE,
    CLUSTERED COLUMNSTORE INDEX
);

-- Load data from ADLS (PolyBase / COPY INTO)
COPY INTO dbo.Orders
FROM 'https://mystorage.dfs.core.windows.net/data/orders/*.parquet'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (IDENTITY = 'Managed Identity')
);
```

---

## Apache Spark Pool

```bash
# Create a Spark pool
az synapse spark pool create \
  --resource-group myRG \
  --workspace-name my-synapse-workspace \
  --name mySparkPool \
  --spark-version 3.4 \
  --node-count 3 \
  --node-size Medium \
  --enable-auto-scale true \
  --min-node-count 3 \
  --max-node-count 10 \
  --enable-auto-pause true \
  --delay 15
```

```python
# PySpark notebook in Synapse Studio
from pyspark.sql import functions as F

# Read from ADLS
df = spark.read.parquet("abfss://data@mystorage.dfs.core.windows.net/raw/events/")

# Transform
result = (df
  .filter(F.col("event_type") == "purchase")
  .groupBy("user_id", F.date_trunc("month", F.col("timestamp")).alias("month"))
  .agg(
      F.count("*").alias("purchase_count"),
      F.sum("amount").alias("total_spend"),
  )
  .orderBy("month", "total_spend", ascending=[True, False])
)

# Write back to ADLS as Delta
result.write.format("delta").mode("overwrite").save(
  "abfss://data@mystorage.dfs.core.windows.net/curated/user-monthly-spend/"
)
```

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Managed ETL/orchestration | Glue + Step Functions | Azure Data Factory / Synapse Pipelines |
| Data warehouse | Redshift | Synapse Dedicated SQL Pool |
| Serverless SQL on files | Athena | Synapse Serverless SQL Pool |
| Managed Spark | Glue / EMR | Synapse Spark Pool / Azure Databricks |
| Unified analytics platform | No single equivalent | Azure Synapse Analytics |
| On-prem connector | Glue JDBC | Self-Hosted Integration Runtime |
| Pause/resume warehouse | Redshift pause/resume | Synapse SQL Pool pause/resume |
| Zero-ETL operational sync | Aurora Zero-ETL to Redshift | Synapse Link |