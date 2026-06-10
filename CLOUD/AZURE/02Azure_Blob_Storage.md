# Azure Blob Storage
## (analogous to Amazon S3)

Azure Blob Storage is Microsoft's massively scalable **object storage** service for unstructured data — files, images, videos, logs, backups, and more. Like S3, it stores data as **blobs (objects)** inside **containers** inside **storage accounts**.

---

## Key Features

- **Unlimited Scalability**: Store petabytes of unstructured data — no provisioning needed.
- **High Durability**: LRS = 3 copies in one datacenter; GRS = 6 copies across 2 regions.
- **Tiered Storage**: Hot, Cool, Cold, Archive — optimize cost vs. access speed.
- **Versioning**: Keep multiple versions of blobs for recovery or auditing.
- **Lifecycle Management**: Automatically transition or delete blobs based on age.
- **Event Notifications**: Trigger Azure Functions, Event Grid on blob events (upload, delete).
- **Shared Access Signatures (SAS)**: Scoped, time-limited URLs (analogous to S3 pre-signed URLs).

---

## Hierarchy

```
Storage Account
  └── Container (analogous to S3 bucket)
        └── Blob (analogous to S3 object)
```

> In S3, bucket names are globally unique. In Azure, **storage account names** are globally unique.

---

## Common Use Cases

- **Backup & Restore**: VM disk snapshots, database backups, disaster recovery.
- **Static Website Hosting**: Serve HTML/CSS/JS directly from Blob Storage.
- **Data Lakes**: Use with Azure Data Lake Storage Gen2 for analytics (analogous to S3 + Athena).
- **Media Storage**: Images, videos, documents for web/mobile apps.
- **Archive**: Blob Archive tier for long-term, ultra-low-cost storage (analogous to S3 Glacier).
- **CI/CD Artifacts**: Build outputs, deployment packages.

---

## Blob Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Block Blob** | Default. Made of blocks; optimized for sequential reads | Files, images, videos, backups |
| **Append Blob** | Optimized for append-only writes | Log files, event streams |
| **Page Blob** | Random read/write in 512-byte pages | VM disk images (VHDs) |

---

## Access Tiers

| Tier | Access Frequency | Latency | Storage Cost | Access Cost |
|------|-----------------|---------|-------------|-------------|
| **Hot** | Frequent | ms | Highest | Lowest |
| **Cool** | Infrequent (monthly) | ms | Lower | Higher |
| **Cold** | Rare (quarterly) | ms | Lower | Higher |
| **Archive** | Rarely (yearly) | Hours (rehydrate first) | Lowest | Highest |

> Archive tier requires **rehydration** (changing to Hot/Cool) before you can read data — similar to restoring from S3 Glacier.

---

## Redundancy Options (analogous to S3 storage classes for durability)

| Option | Description |
|--------|-------------|
| **LRS** (Locally Redundant) | 3 copies in one datacenter |
| **ZRS** (Zone Redundant) | 3 copies across 3 AZs in one region |
| **GRS** (Geo-Redundant) | 6 copies — 3 local + 3 in paired region |
| **GZRS** (Geo-Zone Redundant) | ZRS in primary + LRS in paired region |

---

## CLI Usage

### Create Storage Account
```bash
az storage account create \
  --name mystorageacct \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2
```

### Create Container
```bash
az storage container create \
  --name mycontainer \
  --account-name mystorageacct \
  --public-access off
```

### Upload / Download Blobs
```bash
# Upload
az storage blob upload \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./myfile.txt

# Download
az storage blob download \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --file ./downloaded.txt

# List blobs
az storage blob list \
  --account-name mystorageacct \
  --container-name mycontainer \
  --output table
```

### Generate SAS URL (analogous to S3 Pre-signed URL)
```bash
az storage blob generate-sas \
  --account-name mystorageacct \
  --container-name mycontainer \
  --name myfile.txt \
  --permissions r \
  --expiry 2025-12-31 \
  --output tsv
```

---

## Lifecycle Management Policy

```json
{
  "rules": [
    {
      "name": "move-to-cool",
      "enabled": true,
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 90 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

---

## Access Control

| Method | Description |
|--------|-------------|
| **Storage Account Keys** | Master keys (like AWS root credentials — avoid using directly) |
| **SAS Tokens** | Scoped, time-limited, permission-limited URLs |
| **Azure RBAC** | Assign roles (e.g., Storage Blob Data Reader) to users/identities |
| **Entra ID (OAuth)** | Best practice for application access — use managed identities |
| **Public Access** | Enable anonymous read for public blobs/containers (off by default) |

---

## Key Differences from AWS S3

| Feature | AWS S3 | Azure Blob Storage |
|---------|--------|--------------------|
| Namespace | Buckets (globally unique) | Storage Account + Container |
| Pre-signed URLs | Pre-signed URLs | SAS Tokens |
| Static website | S3 website endpoint | Static website endpoint on storage account |
| Event triggers | S3 Event Notifications | Azure Event Grid |
| Access logging | S3 Server Access Logs | Azure Storage Diagnostics Logs |
| Versioning | Bucket-level | Container-level |
| Infrequent access tier | S3 Standard-IA | Cool / Cold tier |
| Long-term archival | S3 Glacier | Archive tier |