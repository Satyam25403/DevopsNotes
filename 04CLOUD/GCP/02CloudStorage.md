# Google Cloud Storage (GCS)

Google Cloud Storage is GCP's flagship **object storage** service, designed for storing and retrieving any amount of data from anywhere on the internet. Like AWS S3, it stores data as **objects** (file + metadata) inside **buckets** — with globally unique names and flexible access control.

---

## Key Features

- **Unlimited Scalability**: Store petabytes of data with no provisioning.
- **High Durability**: 99.999999999% (11 nines) durability — data is replicated across multiple zones or regions.
- **Strong Consistency**: All reads (including list operations) are strongly consistent — no eventual consistency surprises.
- **Fine-Grained Permissions**: IAM policies, ACLs, signed URLs, and HMAC keys.
- **Versioning**: Retain multiple versions of objects for recovery or auditing.
- **Lifecycle Policies**: Automatically transition or delete objects based on age or version count.
- **Pub/Sub Notifications**: Trigger Pub/Sub messages on object events (e.g., upload, delete).
- **Uniform Bucket-Level Access**: Recommended setting that disables ACLs in favor of IAM-only access control.

---

## Common Use Cases

- **Backup & Restore**: Snapshots, logs, and disaster recovery data.
- **Static Website Hosting**: Serve HTML/CSS/JS directly from Cloud Storage via a load balancer.
- **Data Lakes**: Centralize structured and unstructured data for BigQuery analytics.
- **Machine Learning**: Store training datasets and model artifacts for Vertex AI.
- **Archiving**: Use Nearline/Coldline/Archive storage classes for long-term, low-cost storage.
- **Application Assets**: Images, videos, user uploads for web/mobile apps.
- **CI/CD Artifacts**: Store build artifacts and deployment packages.

---

## Storage Classes

| Class | Access Pattern | Min Storage Duration | Use Case |
|-------|---------------|---------------------|----------|
| **Standard** | Frequent | None | Active data, websites, apps |
| **Nearline** | ~Once/month | 30 days | Backups, infrequent data |
| **Coldline** | ~Once/quarter | 90 days | Disaster recovery, archives |
| **Archive** | ~Once/year | 365 days | Long-term cold archives |

> Retrieval costs increase as you go down the list. Use lifecycle policies to automatically move objects between classes.

---

## CLI Commands

### Bucket Operations
```bash
# List all buckets
gcloud storage ls

# Create a bucket (name must be globally unique)
gcloud storage buckets create gs://my-unique-bucket-name \
  --location=US-CENTRAL1 \
  --default-storage-class=STANDARD \
  --uniform-bucket-level-access

# Delete an empty bucket
gcloud storage buckets delete gs://my-bucket

# Enable versioning
gcloud storage buckets update gs://my-bucket --versioning
```

### Object Operations
```bash
# List objects in a bucket
gcloud storage ls gs://my-bucket/
gcloud storage ls -l gs://my-bucket/             # with size and timestamps

# Upload
gcloud storage cp myfile.txt gs://my-bucket/
gcloud storage cp -r ./local-dir gs://my-bucket/mydir/   # recursive

# Download
gcloud storage cp gs://my-bucket/myfile.txt .
gcloud storage cp -r gs://my-bucket/mydir/ ./local-dir/

# Move / Rename
gcloud storage mv gs://my-bucket/old.txt gs://my-bucket/new.txt

# Delete
gcloud storage rm gs://my-bucket/myfile.txt
gcloud storage rm -r gs://my-bucket/mydir/          # recursive

# Sync a local directory with a bucket (like rsync)
gcloud storage rsync -r ./build gs://my-bucket/
gcloud storage rsync -r -d ./build gs://my-bucket/  # --delete: remove extra remote files
```

### Signed URLs (temporary public access)
```bash
# Generate a signed URL valid for 1 hour (requires a service account key)
gcloud storage sign-url gs://my-bucket/private-file.pdf \
  --duration=1h \
  --private-key-file=sa-key.json
```

---

## Access Control

### IAM (recommended — use Uniform Bucket-Level Access)
```bash
# Grant a user read access to a bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="user:alice@example.com" \
  --role="roles/storage.objectViewer"

# Grant a service account write access
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# View bucket IAM policy
gcloud storage buckets get-iam-policy gs://my-bucket

# Make a bucket publicly readable (for static websites)
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="allUsers" \
  --role="roles/storage.objectViewer"
```

---

## Lifecycle Policies

Define rules to automatically manage object transitions and deletions.

Example `lifecycle.json`:
```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": { "age": 30 }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "COLDLINE" },
        "condition": { "age": 90 }
      },
      {
        "action": { "type": "Delete" },
        "condition": { "age": 365 }
      },
      {
        "action": { "type": "Delete" },
        "condition": { "numNewerVersions": 3, "isLive": false }
      }
    ]
  }
}
```

```bash
gcloud storage buckets update gs://my-bucket --lifecycle-file=lifecycle.json
```

---

## Bucket Notifications (Pub/Sub)

Trigger a Pub/Sub message when objects are created, deleted, or updated:

```bash
# Create a notification for all object changes
gcloud storage buckets notifications create gs://my-bucket \
  --topic=my-pubsub-topic \
  --event-types=OBJECT_FINALIZE,OBJECT_DELETE
```

Common `--event-types`:
- `OBJECT_FINALIZE` — new object created or overwritten
- `OBJECT_DELETE` — object deleted
- `OBJECT_METADATA_UPDATE` — metadata changed
- `OBJECT_ARCHIVE` — live object becomes noncurrent (versioning)

---

## Static Website Hosting

```bash
# 1. Create a bucket matching your domain
gcloud storage buckets create gs://www.mysite.com \
  --uniform-bucket-level-access

# 2. Make it publicly readable
gcloud storage buckets add-iam-policy-binding gs://www.mysite.com \
  --member="allUsers" --role="roles/storage.objectViewer"

# 3. Upload your site
gcloud storage rsync -r ./dist gs://www.mysite.com/

# 4. Set main page and error page
gcloud storage buckets update gs://www.mysite.com \
  --web-main-page-suffix=index.html \
  --web-error-page=404.html
```

> For HTTPS and a custom domain, front the bucket with a **Cloud Load Balancer** + **Cloud CDN** (see Cloud CDN doc).

---

## Key IAM Roles for Cloud Storage

| Role | Permissions |
|------|------------|
| `roles/storage.admin` | Full control: buckets + objects |
| `roles/storage.objectAdmin` | Full control of objects only |
| `roles/storage.objectCreator` | Upload objects (no read/delete) |
| `roles/storage.objectViewer` | Read and list objects |
| `roles/storage.legacyBucketReader` | List bucket and read objects (legacy) |

---

## Best Practices

- **Enable Uniform Bucket-Level Access** on all new buckets — disables legacy ACLs and enforces IAM-only control
- **Enable versioning** on critical buckets to protect against accidental deletions
- **Use lifecycle policies** to automatically move objects to cheaper storage classes
- **Use signed URLs** for temporary, controlled access to private objects
- **Set retention policies** on compliance-sensitive buckets to prevent early deletion
- **Enable audit logging** via Cloud Audit Logs to track all bucket and object operations