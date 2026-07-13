# Azure Functions
## (analogous to AWS Lambda)

Azure Functions is Azure's serverless compute service. You write a function, Azure runs it in response to a trigger — HTTP request, queue message, timer, blob upload, and more. You pay only for execution time.

---

## Core Concepts

### Function App
The deployment and management unit — analogous to a Lambda "application" grouping. Contains one or more functions. Shares configuration, networking, and an App Service Plan (or Consumption Plan).

### Trigger
What causes a function to run. Each function has exactly one trigger.

### Bindings
Declarative connections to other services — input (read data) or output (write data) — without writing integration code. e.g., read from Blob Storage, write to Service Bus.

### Hosting Plans

| Plan | Description | AWS Equivalent |
|------|-------------|----------------|
| **Consumption** | Scale to zero, pay per execution | Lambda default |
| **Flex Consumption** | Faster cold starts, more control | Lambda with provisioned |
| **Premium** | Always warm instances, VNet support | Lambda with VPC |
| **Dedicated (App Service)** | Run on existing App Service Plan | Lambda on dedicated |

---

## Triggers Reference

| Trigger | Fires When | AWS Equivalent |
|---------|-----------|----------------|
| **HTTP** | HTTP request received | API Gateway → Lambda |
| **Timer** | Cron schedule | EventBridge Scheduler → Lambda |
| **Blob Storage** | Blob uploaded/modified | S3 Event → Lambda |
| **Queue Storage** | Message in Azure Queue | SQS → Lambda |
| **Service Bus** | Message in Service Bus queue/topic | SQS/SNS → Lambda |
| **Event Hub** | Event stream record | Kinesis → Lambda |
| **Cosmos DB** | Document change (change feed) | DynamoDB Streams → Lambda |
| **Event Grid** | Any Azure resource event | EventBridge → Lambda |

---

## Creating a Function App

```bash
# Create a storage account (required by Functions)
az storage account create \
  --name myfuncstorage \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

# Create a Function App (Consumption Plan, Node.js 20)
az functionapp create \
  --resource-group myRG \
  --consumption-plan-location eastus \
  --runtime node \
  --runtime-version 20 \
  --functions-version 4 \
  --name my-function-app \
  --storage-account myfuncstore
```

---

## Writing Functions (Node.js v4 programming model)

### HTTP Trigger

```javascript
// src/functions/hello.js
const { app } = require("@azure/functions");

app.http("hello", {
  methods: ["GET", "POST"],
  authLevel: "anonymous",
  handler: async (request, context) => {
    const name = request.query.get("name") || "World";
    return {
      status: 200,
      body: `Hello, ${name}!`,
    };
  },
});
```

### Timer Trigger (cron)

```javascript
const { app } = require("@azure/functions");

app.timer("scheduledTask", {
  schedule: "0 */5 * * * *", // every 5 minutes
  handler: async (myTimer, context) => {
    context.log("Timer fired at", new Date().toISOString());
    // do work
  },
});
```

### Blob Storage Trigger

```javascript
const { app } = require("@azure/functions");

app.storageBlob("processBlobUpload", {
  path: "uploads/{name}",
  connection: "AzureWebJobsStorage",
  handler: async (blob, context) => {
    context.log(`Blob ${context.triggerMetadata.name} uploaded, size: ${blob.length}`);
    // process blob contents
  },
});
```

### Service Bus Trigger (with output binding)

```javascript
const { app, output } = require("@azure/functions");

const sbOutput = output.serviceBus({
  queueName: "results-queue",
  connection: "ServiceBusConnection",
});

app.serviceBusQueue("processOrder", {
  queueName: "orders-queue",
  connection: "ServiceBusConnection",
  return: sbOutput,
  handler: async (message, context) => {
    context.log("Processing order:", message);
    return { orderId: message.id, status: "processed" }; // written to output queue
  },
});
```

---

## Deploying Functions

### Via Azure Functions Core Tools (local dev + deploy)

```bash
# Install Functions Core Tools
npm install -g azure-functions-core-tools@4

# Create a new function project
func init my-func-project --javascript
cd my-func-project

# Create a new function
func new --name hello --template "HTTP trigger"

# Run locally
func start

# Deploy to Azure
func azure functionapp publish my-function-app
```

### Via CLI (ZIP deploy)

```bash
az functionapp deployment source config-zip \
  --resource-group myRG \
  --name my-function-app \
  --src ./function.zip
```

---

## Configuration & Secrets

```bash
# Set app settings (environment variables)
az functionapp config appsettings set \
  --resource-group myRG \
  --name my-function-app \
  --settings NODE_ENV=production \
    DB_CONNECTION_STRING="@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/db-conn/)"
```

> Same Key Vault reference syntax as App Service — assign `Key Vault Secrets User` role to the function app's managed identity.

---

## Managed Identity

```bash
# Assign system-managed identity
az functionapp identity assign \
  --resource-group myRG \
  --name my-function-app

# Grant access to Blob Storage
az role assignment create \
  --assignee $(az functionapp identity show -g myRG -n my-function-app --query principalId -o tsv) \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage
```

Inside the function, `DefaultAzureCredential` works automatically.

---

## Durable Functions (analogous to AWS Step Functions)

Durable Functions extends Azure Functions with stateful workflows — chaining, fan-out/fan-in, human approval, and long-running processes.

```javascript
const { app } = require("@azure/functions");
const df = require("durable-functions");

// Orchestrator — coordinates the workflow
df.app.orchestration("orderWorkflow", function* (context) {
  const validated = yield context.df.callActivity("ValidateOrder", context.df.getInput());
  const charged = yield context.df.callActivity("ChargePayment", validated);
  yield context.df.callActivity("SendConfirmation", charged);
  return "Order complete";
});

// Activity — individual unit of work
df.app.activity("ValidateOrder", {
  handler: async (order) => {
    // validate and return
    return { ...order, valid: true };
  },
});

// HTTP starter — kicks off the orchestration
app.http("startOrder", {
  route: "orders/start",
  handler: df.createHttpStart("orderWorkflow"),
});
```

---

## Monitoring

```bash
# View function invocation logs
az monitor app-insights component show \
  --app my-function-app \
  --resource-group myRG

# Stream live logs
az webapp log tail \
  --resource-group myRG \
  --name my-function-app
```

All function executions are logged to **Application Insights** by default. Query with KQL in the Azure Portal.

---

## Key Differences from AWS Lambda

| Feature | AWS Lambda | Azure Functions |
|---------|-----------|----------------|
| Deployment unit | Function (individual) | Function App (group) |
| Config file | No config file | `host.json`, `local.settings.json` |
| Trigger config | Event source mapping (console/CLI) | Declared in code (v4 model) |
| Stateful workflows | Step Functions | Durable Functions |
| Local dev tool | SAM CLI / Lambda Runtime Emulator | Azure Functions Core Tools (`func`) |
| Max timeout (Consumption) | 15 min | 10 min (configurable to unlimited on Premium) |
| Cold start mitigation | Provisioned Concurrency | Premium plan (always warm) |
| Layers | Lambda Layers | Built with `npm install` in package |