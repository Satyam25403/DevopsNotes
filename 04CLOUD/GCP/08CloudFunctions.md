# Google Cloud Functions (Serverless Functions)

Cloud Functions lets you run code without provisioning or managing servers. You write a function, define a trigger, and GCP handles everything else — scaling, availability, and infrastructure. It's the GCP equivalent of AWS Lambda.

---

## Core Concepts

- **Serverless**: No servers to manage. GCP provisions compute on demand.
- **Event-driven**: Functions run in response to events — HTTP requests, Pub/Sub messages, Cloud Storage changes, Firestore writes, scheduled cron jobs, etc.
- **Pay-per-use**: Billed per invocation and per 100ms of execution time. Zero cost when idle.
- **Automatic scaling**: Scales from zero to thousands of concurrent instances without any configuration.
- **Two generations**: Gen 1 (legacy) and **Gen 2** (recommended — powered by Cloud Run, longer timeouts, more power).

---

## Supported Runtimes

- Node.js (18, 20, 22)
- Python (3.10, 3.11, 3.12)
- Go (1.21, 1.22)
- Java (11, 17, 21)
- Ruby (3.2, 3.3)
- PHP (8.2, 8.3)
- .NET (6, 8)

---

## Deploying a Cloud Function

### HTTP Trigger (Gen 2)
```bash
# Node.js example — index.js
# exports.helloWorld = (req, res) => { res.send('Hello!'); };

gcloud functions deploy hello-world \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=helloWorld \
  --trigger-http \
  --allow-unauthenticated

# The output includes the function URL
```

### Pub/Sub Trigger
```bash
gcloud functions deploy process-order \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=process_order \
  --trigger-topic=orders-topic
```

### Cloud Storage Trigger
```bash
gcloud functions deploy on-file-upload \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=onFileUpload \
  --trigger-bucket=my-upload-bucket
```

### Scheduled (Cron) via Cloud Scheduler
```bash
# Deploy the function with HTTP trigger
gcloud functions deploy daily-report \
  --gen2 \
  --runtime=python312 \
  --region=us-central1 \
  --source=. \
  --entry-point=generate_daily_report \
  --trigger-http \
  --no-allow-unauthenticated      # Only allow authenticated callers

# Create a Cloud Scheduler job to call it
gcloud scheduler jobs create http daily-report-job \
  --location=us-central1 \
  --schedule="0 9 * * 1-5" \    # 9am Mon-Fri
  --uri=FUNCTION_URL \
  --oidc-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com
```

---

## Function Code Examples

### Node.js — HTTP Function
```javascript
// index.js
const functions = require('@google-cloud/functions-framework');

functions.http('helloWorld', (req, res) => {
  const name = req.query.name || req.body.name || 'World';
  res.send(`Hello, ${name}!`);
});
```

### Node.js — Pub/Sub Trigger
```javascript
const functions = require('@google-cloud/functions-framework');

functions.cloudEvent('processOrder', async (cloudEvent) => {
  const message = cloudEvent.data.message;
  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());
  console.log('Processing order:', data.orderId);
  // ... business logic
});
```

### Python — HTTP Function
```python
# main.py
import functions_framework

@functions_framework.http
def hello_world(request):
    name = request.args.get('name', 'World')
    return f'Hello, {name}!'
```

### Python — Cloud Storage Trigger
```python
import functions_framework
from google.cloud import storage

@functions_framework.cloud_event
def on_file_upload(cloud_event):
    data = cloud_event.data
    bucket_name = data["bucket"]
    file_name = data["name"]
    print(f"New file: gs://{bucket_name}/{file_name}")
    # Process the file...
```

---

## Environment Variables and Secrets

```bash
# Set environment variables at deploy time
gcloud functions deploy my-function \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --set-env-vars="NODE_ENV=production,LOG_LEVEL=info"

# Mount a secret as an environment variable
gcloud functions deploy my-function \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --set-secrets="DB_PASSWORD=db-password:latest"
```

---

## Managing Functions

```bash
# List functions
gcloud functions list --region=us-central1

# Describe a function (URL, trigger, runtime, status)
gcloud functions describe my-function --region=us-central1

# View logs
gcloud functions logs read my-function --region=us-central1
gcloud functions logs read my-function --region=us-central1 --limit=50

# Delete a function
gcloud functions delete my-function --region=us-central1

# Update env vars without redeploying code
gcloud functions deploy my-function \
  --region=us-central1 \
  --update-env-vars="NEW_VAR=value"
```

---

## Configuration Options

```bash
gcloud functions deploy my-function \
  --gen2 \
  --runtime=nodejs20 \
  --region=us-central1 \
  --source=. \
  --entry-point=handler \
  --trigger-http \
  --memory=512MB \             # 128MB – 32GiB (Gen 2)
  --cpu=1 \                    # Gen 2 only
  --timeout=300s \             # Max 3600s (1 hour) for Gen 2
  --min-instances=1 \          # Avoid cold starts
  --max-instances=100 \
  --concurrency=10 \           # Gen 2: requests per instance (like Cloud Run)
  --service-account=my-function-sa@my-project.iam.gserviceaccount.com
```

---

## Testing Locally

```bash
# Install the Functions Framework
npm install @google-cloud/functions-framework

# Run locally
npx @google-cloud/functions-framework --target=helloWorld --port=8080

# Test with curl
curl http://localhost:8080

# Python
pip install functions-framework
functions-framework --target=hello_world --port=8080
```

---

## Gen 1 vs Gen 2

| Feature | Gen 1 | Gen 2 (recommended) |
|---------|-------|---------------------|
| Max timeout | 9 minutes | 60 minutes |
| Max memory | 8 GiB | 32 GiB |
| Concurrency | 1 per instance | Up to 1000 (like Cloud Run) |
| Min instances | ✅ | ✅ |
| Source | Zip upload or GCS | Same + Docker image |
| Underlying platform | Functions-specific | Cloud Run |

---

## Common Use Cases

| Trigger | Use Case |
|---------|----------|
| HTTP | REST APIs, webhooks, form handlers |
| Pub/Sub | Async event processing, data pipelines |
| Cloud Storage | Image resizing on upload, ETL on file arrival |
| Firestore | Denormalization, notifications on data changes |
| Cloud Scheduler | Cron jobs, daily reports, cache warmup |
| Eventarc | React to any GCP service event (audit logs, etc.) |

---

## Best Practices

- **Use Gen 2** for all new functions — more powerful, better concurrency
- **Keep functions small and focused**: One function, one responsibility
- **Set min-instances=1** for latency-sensitive HTTP functions to eliminate cold starts
- **Attach a service account** with least-privilege IAM roles
- **Use Secret Manager** for secrets — never hardcode or use env vars for sensitive values
- **Set timeouts appropriately**: Don't use the maximum if your function runs in seconds
- **Idempotent design**: Pub/Sub delivers at-least-once — your function must handle duplicates safely