# GCP Secret Manager

GCP Secret Manager is a secure, centralized service for managing API keys, passwords, certificates, and other sensitive configuration. Think of it as a cloud-native `.env` file — but with encryption, IAM access control, versioning, and audit logging built in.

---

## What You Can Store

- **Database passwords**: Connection strings, credentials
- **API keys**: Third-party service tokens, webhook secrets
- **TLS/SSL certificates and private keys**
- **Service account key files** (though Workload Identity is preferred)
- **Any sensitive string** up to 64 KiB

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Secret** | A named container for sensitive data (e.g., `db-password`) |
| **Version** | A specific value of a secret. Secrets are immutable per version. |
| **`latest`** | Always points to the most recently added version |
| **State** | Each version can be `ENABLED`, `DISABLED`, or `DESTROYED` |
| **Replication** | `AUTOMATIC` (Google-managed) or `USER_MANAGED` (specific regions) |

---

## CLI Usage

### Create a Secret
```bash
# Create a secret (empty — no value yet)
gcloud secrets create db-password \
  --replication-policy=automatic

# Create a secret with an initial value
echo -n "my-super-secret-password" | gcloud secrets create db-password \
  --replication-policy=automatic \
  --data-file=-
```

### Add a New Version (rotate a secret)
```bash
echo -n "new-rotated-password" | gcloud secrets versions add db-password \
  --data-file=-

# From a file
gcloud secrets versions add ssl-cert --data-file=./cert.pem
```

### Access a Secret Value
```bash
# Access the latest version
gcloud secrets versions access latest --secret=db-password

# Access a specific version
gcloud secrets versions access 3 --secret=db-password
```

### List Secrets and Versions
```bash
# List all secrets in the project
gcloud secrets list

# List all versions of a secret
gcloud secrets versions list db-password
```

### Disable / Enable / Destroy a Version
```bash
gcloud secrets versions disable 2 --secret=db-password
gcloud secrets versions enable 2 --secret=db-password
gcloud secrets versions destroy 1 --secret=db-password  # Permanent — cannot be undone
```

### Delete a Secret
```bash
gcloud secrets delete db-password
```

---

## Access Control (IAM)

Secrets are secured via IAM. Grant access only to the service accounts or users that need it.

```bash
# Grant a service account access to read a specific secret
gcloud secrets add-iam-policy-binding db-password \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"

# Grant a user the ability to manage secrets (not read values)
gcloud secrets add-iam-policy-binding db-password \
  --member="user:devops@example.com" \
  --role="roles/secretmanager.admin"
```

Key IAM roles:

| Role | Capabilities |
|------|-------------|
| `roles/secretmanager.admin` | Full control: create, update, delete, access |
| `roles/secretmanager.secretVersionManager` | Add and manage versions |
| `roles/secretmanager.secretAccessor` | **Read secret values** — grant to apps |
| `roles/secretmanager.viewer` | View metadata (not values) |

---

## Accessing Secrets in Code (Node.js)

```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

async function getSecret(secretName, version = 'latest') {
  const name = `projects/${process.env.GCP_PROJECT_ID}/secrets/${secretName}/versions/${version}`;
  const [response] = await client.accessSecretVersion({ name });
  return response.payload.data.toString('utf8');
}

// Usage
const dbPassword = await getSecret('db-password');
const apiKey = await getSecret('stripe-api-key');
```

### Caching Secret Values (recommended for performance)
```javascript
const cache = new Map();

async function getCachedSecret(secretName) {
  if (cache.has(secretName)) return cache.get(secretName);
  const value = await getSecret(secretName);
  cache.set(secretName, value);
  return value;
}
```

---

## Accessing Secrets in Cloud Run / GKE

### Cloud Run (mount as environment variable or volume)
```bash
# Deploy Cloud Run service with a secret as an env var
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-image \
  --region=us-central1 \
  --set-secrets="DB_PASSWORD=db-password:latest"

# Mount as a file volume
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-image \
  --region=us-central1 \
  --set-secrets="/secrets/db-password=db-password:latest"
```

### Kubernetes / GKE (via External Secrets Operator or direct API)
```yaml
# Using the Secrets Store CSI Driver with Secret Manager provider
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-secret-provider
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/my-project/secrets/db-password/versions/latest"
        fileName: "db-password"
```

---

## Secret Rotation

Secret Manager supports automatic rotation via Pub/Sub notifications:

```bash
# Add a rotation schedule to a secret
gcloud secrets update db-password \
  --rotation-period=2592000s \   # 30 days in seconds
  --next-rotation-time="2025-01-01T00:00:00Z"

# Configure a Pub/Sub topic to receive rotation notifications
gcloud secrets update db-password \
  --topics=projects/my-project/topics/secret-rotation
```

Your application subscribes to the Pub/Sub topic and updates the secret when triggered.

---

## Organizing Secrets

Use labels and naming conventions to organize secrets at scale:

```bash
# Add labels to a secret
gcloud secrets update db-password \
  --update-labels="env=prod,team=backend,app=orders"

# Filter secrets by label
gcloud secrets list --filter="labels.env=prod"
```

Recommended naming convention: `{app}-{env}-{resource}` — e.g.:
- `orders-prod-db-password`
- `payments-staging-stripe-key`
- `shared-prod-jwt-secret`

---

## Best Practices

- **Least privilege**: Use `roles/secretmanager.secretAccessor` for apps — never `admin`
- **Grant per-secret, not per-project**: Scope IAM to individual secrets, not all secrets in the project
- **Never hardcode secrets**: Replace `.env` files and hardcoded strings with Secret Manager calls
- **Audit access**: Enable Cloud Audit Logs to track who accessed which secret and when
- **Rotate regularly**: Use rotation schedules and Pub/Sub notifications to automate rotation
- **Destroy old versions**: Destroy (not just disable) old versions after rotation to reduce exposure
- **Use `latest` carefully**: Pin to a specific version in production if you need rollback safety