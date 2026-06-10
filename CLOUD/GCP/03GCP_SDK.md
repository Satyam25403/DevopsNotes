# GCP Client Libraries (Node.js / JavaScript)

The GCP Client Libraries for Node.js are the official toolkit for integrating Google Cloud services into your applications. They're modular, TypeScript-friendly, and support services from Cloud Storage to Pub/Sub to BigQuery.

> **Recommended over REST calls**: Client libraries handle authentication, retries, pagination, and serialization automatically. Each GCP service has its own npm package.

---

## Getting Started

### 1. Install the Libraries

```bash
# Install only the service(s) you need
npm install @google-cloud/storage
npm install @google-cloud/pubsub
npm install @google-cloud/bigquery
npm install @google-cloud/secret-manager
npm install @google-cloud/tasks
npm install @google-cloud/logging
npm install @google-cloud/firestore
npm install @google-cloud/spanner
```

### 2. Configure Credentials

The SDK automatically reads credentials from (in order of priority):
1. `GOOGLE_APPLICATION_CREDENTIALS` environment variable (path to a service account JSON key)
2. Application Default Credentials (`gcloud auth application-default login`)
3. Attached service account (when running on Compute Engine, GKE, Cloud Run, or Cloud Functions)

```bash
# For local development
gcloud auth application-default login

# For CI/CD or production (service account key)
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json
```

> **Best Practice**: On GCP infrastructure (Cloud Run, GKE, Compute Engine), attach a service account to the resource — no key file needed.

---

## Common Usage Examples

### Cloud Storage
```javascript
const { Storage } = require('@google-cloud/storage');
const storage = new Storage();

// Upload a file
await storage.bucket('my-bucket').upload('./local-file.txt', {
  destination: 'remote-file.txt',
});

// Download a file
await storage.bucket('my-bucket').file('remote-file.txt').download({
  destination: './local-download.txt',
});

// Generate a signed URL (temporary access)
const [url] = await storage.bucket('my-bucket').file('private.pdf').getSignedUrl({
  action: 'read',
  expires: Date.now() + 60 * 60 * 1000, // 1 hour
});

// List objects in a bucket
const [files] = await storage.bucket('my-bucket').getFiles({ prefix: 'images/' });
files.forEach(file => console.log(file.name));
```

### Secret Manager
```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
const client = new SecretManagerServiceClient();

// Access the latest version of a secret
async function getSecret(secretName) {
  const [version] = await client.accessSecretVersion({
    name: `projects/my-project/secrets/${secretName}/versions/latest`,
  });
  return version.payload.data.toString('utf8');
}

const dbPassword = await getSecret('db-password');
```

### Pub/Sub (Publish)
```javascript
const { PubSub } = require('@google-cloud/pubsub');
const pubsub = new PubSub();

// Publish a message
const topic = pubsub.topic('my-topic');
const messageId = await topic.publishMessage({
  data: Buffer.from(JSON.stringify({ event: 'order.created', orderId: '123' })),
  attributes: { source: 'orders-service' },
});
console.log(`Message published: ${messageId}`);
```

### Pub/Sub (Subscribe)
```javascript
const subscription = pubsub.subscription('my-subscription');

subscription.on('message', message => {
  const data = JSON.parse(message.data.toString());
  console.log('Received:', data);
  message.ack(); // Acknowledge to remove from queue
});

subscription.on('error', err => console.error('Error:', err));
```

### BigQuery
```javascript
const { BigQuery } = require('@google-cloud/bigquery');
const bigquery = new BigQuery();

// Run a query
const query = `
  SELECT name, COUNT(*) as count
  FROM \`my-project.my-dataset.my-table\`
  WHERE created_at > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  GROUP BY name
  ORDER BY count DESC
  LIMIT 10
`;

const [rows] = await bigquery.query({ query });
rows.forEach(row => console.log(row));
```

### Firestore
```javascript
const { Firestore } = require('@google-cloud/firestore');
const db = new Firestore();

// Write a document
await db.collection('users').doc('alice').set({
  name: 'Alice',
  email: 'alice@example.com',
  createdAt: Firestore.Timestamp.now(),
});

// Read a document
const doc = await db.collection('users').doc('alice').get();
if (doc.exists) {
  console.log(doc.data());
}

// Query
const snapshot = await db.collection('users')
  .where('role', '==', 'admin')
  .orderBy('createdAt', 'desc')
  .limit(10)
  .get();

snapshot.forEach(doc => console.log(doc.id, doc.data()));
```

---

## Using with TypeScript

All GCP Node.js client libraries include TypeScript type definitions out of the box:

```typescript
import { Storage, Bucket, File } from '@google-cloud/storage';
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';

const storage = new Storage({ projectId: process.env.GCP_PROJECT_ID });

async function uploadFile(bucketName: string, localPath: string, remotePath: string): Promise<void> {
  const bucket: Bucket = storage.bucket(bucketName);
  await bucket.upload(localPath, { destination: remotePath });
}
```

---

## Explicit Project / Credentials

```javascript
// Explicit configuration
const storage = new Storage({
  projectId: 'my-project-id',
  keyFilename: '/path/to/sa-key.json',   // Only needed outside GCP
});

// Or using credentials object directly
const storage = new Storage({
  projectId: 'my-project-id',
  credentials: {
    client_email: process.env.GCP_CLIENT_EMAIL,
    private_key: process.env.GCP_PRIVATE_KEY,
  },
});
```

---

## GCP SDK vs Direct REST API

| Approach | When to Use |
|----------|-------------|
| **Client Library** (recommended) | All application code — handles auth, retries, types |
| **`gcloud` CLI** | Manual operations, scripts, CI/CD pipelines |
| **REST API** | When no SDK exists for your language, or in edge cases |
| **Terraform / Deployment Manager** | Infrastructure provisioning (not application code) |

---

## Useful Packages Reference

| Package | Service |
|---------|---------|
| `@google-cloud/storage` | Cloud Storage |
| `@google-cloud/pubsub` | Pub/Sub messaging |
| `@google-cloud/bigquery` | BigQuery data warehouse |
| `@google-cloud/firestore` | Firestore NoSQL database |
| `@google-cloud/spanner` | Cloud Spanner |
| `@google-cloud/secret-manager` | Secret Manager |
| `@google-cloud/tasks` | Cloud Tasks (async job queue) |
| `@google-cloud/logging` | Cloud Logging |
| `@google-cloud/monitoring` | Cloud Monitoring metrics |
| `@google-cloud/translate` | Cloud Translation API |
| `@google-cloud/vision` | Cloud Vision AI |
| `@google-cloud/aiplatform` | Vertex AI |