# Azure SDK for JavaScript / Node.js
## (analogous to AWS SDK for JavaScript v3)

The Azure SDK for JavaScript is the official toolkit for integrating Azure services into your Node.js (or browser) applications. It follows the same modular pattern as AWS SDK v3 — install only what you need.

> All Azure SDK packages follow the naming pattern `@azure/<service-name>`.

---

## Getting Started

### 1. Install the SDK Packages

```bash
# Storage
npm install @azure/storage-blob

# Key Vault / Secrets (analogous to Secrets Manager + Parameter Store)
npm install @azure/keyvault-secrets

# Identity (authentication — always needed)
npm install @azure/identity

# Cosmos DB (analogous to DynamoDB)
npm install @azure/cosmos

# Service Bus (analogous to SQS/SNS)
npm install @azure/service-bus

# Event Hubs (analogous to MSK/Kafka)
npm install @azure/event-hubs

# App Configuration (analogous to Parameter Store)
npm install @azure/app-configuration

# Virtual Machines
npm install @azure/arm-compute

# Resource Manager (general Azure resource management)
npm install @azure/arm-resources
```

---

### 2. Authentication

The Azure SDK uses a unified `@azure/identity` package. The key class is `DefaultAzureCredential` — it tries multiple authentication methods in order:

1. Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`)
2. Workload Identity (Kubernetes)
3. Managed Identity (inside Azure VMs, App Services, Functions)
4. Azure CLI credentials (local dev)
5. Visual Studio Code credentials

```javascript
const { DefaultAzureCredential } = require("@azure/identity");

// Works in dev (uses az login) and production (uses Managed Identity)
const credential = new DefaultAzureCredential();
```

> This is similar to how AWS SDK v3 uses `fromIni()` locally and instance profile credentials in production — but Azure's `DefaultAzureCredential` handles both automatically.

---

### 3. Blob Storage (analogous to S3)

```javascript
const { BlobServiceClient } = require("@azure/storage-blob");
const { DefaultAzureCredential } = require("@azure/identity");

const accountUrl = "https://mystorageacct.blob.core.windows.net";
const credential = new DefaultAzureCredential();
const blobServiceClient = new BlobServiceClient(accountUrl, credential);

// Upload a blob
async function uploadBlob() {
  const containerClient = blobServiceClient.getContainerClient("mycontainer");
  const blockBlobClient = containerClient.getBlockBlobClient("hello.txt");

  await blockBlobClient.upload("Hello, Azure!", Buffer.byteLength("Hello, Azure!"));
  console.log("Blob uploaded");
}

// Download a blob
async function downloadBlob() {
  const containerClient = blobServiceClient.getContainerClient("mycontainer");
  const blobClient = containerClient.getBlobClient("hello.txt");

  const downloadResponse = await blobClient.download();
  const content = await streamToString(downloadResponse.readableStreamBody);
  console.log(content);
}

// List blobs
async function listBlobs() {
  const containerClient = blobServiceClient.getContainerClient("mycontainer");
  for await (const blob of containerClient.listBlobsFlat()) {
    console.log(blob.name);
  }
}
```

---

### 4. Key Vault Secrets (analogous to Secrets Manager / Parameter Store)

```javascript
const { SecretClient } = require("@azure/keyvault-secrets");
const { DefaultAzureCredential } = require("@azure/identity");

const vaultUrl = "https://mykeyvault.vault.azure.net";
const credential = new DefaultAzureCredential();
const client = new SecretClient(vaultUrl, credential);

// Store a secret
await client.setSecret("db-password", "super-secret-value");

// Retrieve a secret
const secret = await client.getSecret("db-password");
console.log(secret.value);

// Delete a secret
await client.beginDeleteSecret("db-password");
```

---

### 5. Cosmos DB (analogous to DynamoDB)

```javascript
const { CosmosClient } = require("@azure/cosmos");

const client = new CosmosClient({
  endpoint: "https://myaccount.documents.azure.com",
  key: process.env.COSMOS_KEY,
});

const { database } = await client.databases.createIfNotExists({ id: "mydb" });
const { container } = await database.containers.createIfNotExists({ id: "users" });

// Insert item
await container.items.create({ id: "user1", name: "Satyam", role: "engineer" });

// Query
const { resources } = await container.items
  .query("SELECT * FROM c WHERE c.role = 'engineer'")
  .fetchAll();
console.log(resources);
```

---

### 6. Service Bus (analogous to SQS/SNS)

```javascript
const { ServiceBusClient } = require("@azure/service-bus");

const client = new ServiceBusClient(process.env.SERVICEBUS_CONNECTION_STRING);

// Send a message (producer)
const sender = client.createSender("myqueue");
await sender.sendMessages({ body: { task: "process-image", id: "123" } });
await sender.close();

// Receive messages (consumer)
const receiver = client.createReceiver("myqueue");
const messages = await receiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });
for (const msg of messages) {
  console.log(msg.body);
  await receiver.completeMessage(msg);
}
await client.close();
```

---

### 7. App Configuration (analogous to Parameter Store)

```javascript
const { AppConfigurationClient } = require("@azure/app-configuration");
const { DefaultAzureCredential } = require("@azure/identity");

const client = new AppConfigurationClient(
  "https://myappconfig.azconfig.io",
  new DefaultAzureCredential()
);

// Get a setting
const setting = await client.getConfigurationSetting({ key: "myapp:db-url" });
console.log(setting.value);

// Set a setting
await client.setConfigurationSetting({ key: "myapp:feature-flag", value: "true" });
```

---

## Credential Chain Summary

| Environment | Credential Method |
|-------------|-----------------|
| Local dev (az login) | `AzureCliCredential` (auto via `DefaultAzureCredential`) |
| Azure VM / App Service | `ManagedIdentityCredential` (auto via `DefaultAzureCredential`) |
| CI/CD (GitHub Actions, etc.) | `EnvironmentCredential` via `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` |
| Kubernetes | `WorkloadIdentityCredential` |

---

## Package Reference

| Package | Purpose | AWS Equivalent |
|---------|---------|----------------|
| `@azure/identity` | Authentication | `@aws-sdk/credential-providers` |
| `@azure/storage-blob` | Object storage | `@aws-sdk/client-s3` |
| `@azure/keyvault-secrets` | Secrets management | `@aws-sdk/client-secrets-manager` |
| `@azure/cosmos` | NoSQL database | `@aws-sdk/client-dynamodb` |
| `@azure/service-bus` | Message queues/topics | `@aws-sdk/client-sqs` + `@aws-sdk/client-sns` |
| `@azure/event-hubs` | Event streaming | `@aws-sdk/client-kafka` (MSK) |
| `@azure/app-configuration` | Config/feature flags | `@aws-sdk/client-ssm` |
| `@azure/arm-compute` | VM management | `@aws-sdk/client-ec2` |