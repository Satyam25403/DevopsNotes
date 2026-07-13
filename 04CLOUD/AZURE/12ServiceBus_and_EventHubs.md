# Azure Messaging: Service Bus & Event Hubs
## (analogous to AWS SQS/SNS & Kinesis)

Azure has two primary messaging services for decoupling applications: **Service Bus** for reliable enterprise messaging and **Event Hubs** for high-throughput event streaming.

---

## Service Comparison

| Service | Pattern | Throughput | Retention | AWS Equivalent |
|---------|---------|-----------|-----------|----------------|
| **Service Bus Queues** | Point-to-point (one consumer) | Moderate | Up to 14 days | SQS Standard |
| **Service Bus Topics** | Publish/subscribe (many consumers) | Moderate | Up to 14 days | SNS + SQS |
| **Event Hubs** | Event stream (ordered, replay) | Millions/sec | Up to 90 days | Kinesis Data Streams |
| **Azure Queue Storage** | Simple queue, very cheap | Low | 7 days | SQS (basic) |
| **Event Grid** | Event routing (push, fan-out) | High | 24 hours | EventBridge |

---

## Part 1: Azure Service Bus
## (analogous to AWS SQS + SNS)

Service Bus is a fully managed enterprise message broker. Supports **queues** (point-to-point) and **topics with subscriptions** (pub/sub). Key features: FIFO, dead-lettering, sessions, message deferral, duplicate detection.

---

### Creating a Namespace and Queue

```bash
# Create a Service Bus namespace
az servicebus namespace create \
  --resource-group myRG \
  --name my-servicebus-ns \
  --location eastus \
  --sku Standard

# Create a queue
az servicebus queue create \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --name orders-queue \
  --max-delivery-count 10 \
  --lock-duration PT30S \
  --default-message-time-to-live P14D

# Get connection string
az servicebus namespace authorization-rule keys list \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString \
  --output tsv
```

---

### Topics and Subscriptions (pub/sub)

```bash
# Create a topic
az servicebus topic create \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --name order-events

# Create subscriptions (each gets its own copy of every message)
az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --topic-name order-events \
  --name inventory-service

az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --topic-name order-events \
  --name notification-service

# Add a filter to a subscription (only receive messages matching rule)
az servicebus topic subscription rule create \
  --resource-group myRG \
  --namespace-name my-servicebus-ns \
  --topic-name order-events \
  --subscription-name inventory-service \
  --name high-value-filter \
  --filter-sql-expression "orderValue > 1000"
```

---

### Node.js: Send and Receive

```bash
npm install @azure/service-bus
```

```javascript
const { ServiceBusClient } = require("@azure/service-bus");
const { DefaultAzureCredential } = require("@azure/identity");

const fullyQualifiedNamespace = "my-servicebus-ns.servicebus.windows.net";
const client = new ServiceBusClient(fullyQualifiedNamespace, new DefaultAzureCredential());

// --- Producer ---
async function sendOrder(order) {
  const sender = client.createSender("orders-queue");
  try {
    await sender.sendMessages({
      body: order,
      contentType: "application/json",
      subject: "new-order",
      messageId: order.id,                  // for duplicate detection
      sessionId: order.customerId,          // for FIFO sessions
    });
    console.log("Order sent:", order.id);
  } finally {
    await sender.close();
  }
}

// --- Consumer (process and complete) ---
async function startConsumer() {
  const receiver = client.createReceiver("orders-queue", {
    receiveMode: "peekLock",               // message locked, not deleted until completed
  });

  receiver.subscribe({
    processMessage: async (msg) => {
      console.log("Processing order:", msg.body);
      // do work...
      await receiver.completeMessage(msg); // remove from queue
    },
    processError: async (err) => {
      console.error("Receiver error:", err);
    },
  });
}

// --- Dead-letter queue (failed messages) ---
async function processDeadLetters() {
  const dlqReceiver = client.createReceiver(
    "orders-queue",
    { subQueueType: "deadLetter" }
  );
  const messages = await dlqReceiver.receiveMessages(10, { maxWaitTimeInMs: 5000 });
  for (const msg of messages) {
    console.log("Dead-lettered:", msg.body, "Reason:", msg.deadLetterReason);
    await dlqReceiver.completeMessage(msg);
  }
}
```

---

### Message Sessions (FIFO per entity, analogous to SQS FIFO)

Sessions guarantee ordered processing per `sessionId` (e.g., per customer, per order).

```javascript
// Enable sessions on queue first:
// az servicebus queue create ... --requires-session true

const sessionReceiver = await client.acceptNextSession("orders-queue");
console.log("Processing session:", sessionReceiver.sessionId);

const messages = await sessionReceiver.receiveMessages(10);
for (const msg of messages) {
  await sessionReceiver.completeMessage(msg);
}
await sessionReceiver.close();
```

---

## Part 2: Azure Event Hubs
## (analogous to AWS Kinesis Data Streams)

Event Hubs is a fully managed, real-time event ingestion service. Designed for high-throughput data pipelines — telemetry, logs, clickstreams, IoT data. Messages are retained and replayable by offset.

---

### Core Concepts

| Concept | Description | Kinesis Equivalent |
|---------|-------------|-------------------|
| **Namespace** | Container for event hubs | — |
| **Event Hub** | A named stream | Kinesis Stream |
| **Partition** | Ordered lane within a hub | Shard |
| **Consumer Group** | Independent cursor through the stream | Kinesis Consumer |
| **Offset** | Position within a partition | Sequence Number |
| **Throughput Units (TU)** | 1 TU = 1 MB/s in, 2 MB/s out | Shard (1 MB/s in) |

---

### Creating an Event Hub

```bash
# Create namespace
az eventhubs namespace create \
  --resource-group myRG \
  --name my-eventhubs-ns \
  --location eastus \
  --sku Standard \
  --capacity 2                  # 2 throughput units

# Create an event hub
az eventhubs eventhub create \
  --resource-group myRG \
  --namespace-name my-eventhubs-ns \
  --name telemetry \
  --partition-count 8 \
  --message-retention 7          # days

# Create a consumer group
az eventhubs eventhub consumer-group create \
  --resource-group myRG \
  --namespace-name my-eventhubs-ns \
  --eventhub-name telemetry \
  --name analytics-consumer
```

---

### Node.js: Produce and Consume Events

```bash
npm install @azure/event-hubs @azure/eventhubs-checkpointstore-blob
```

```javascript
const { EventHubProducerClient, EventHubConsumerClient } = require("@azure/event-hubs");
const { ContainerClient } = require("@azure/storage-blob");
const { BlobCheckpointStore } = require("@azure/eventhubs-checkpointstore-blob");
const { DefaultAzureCredential } = require("@azure/identity");

const credential = new DefaultAzureCredential();
const fullyQualifiedNamespace = "my-eventhubs-ns.servicebus.windows.net";
const eventHubName = "telemetry";

// --- Producer: send a batch of events ---
async function sendEvents(events) {
  const producer = new EventHubProducerClient(
    fullyQualifiedNamespace,
    eventHubName,
    credential
  );

  const batch = await producer.createBatch({
    partitionKey: events[0].deviceId,   // route related events to same partition
  });

  for (const event of events) {
    batch.tryAdd({ body: event });
  }

  await producer.sendBatch(batch);
  await producer.close();
}

// --- Consumer with checkpointing (resume from last position after restart) ---
async function startConsumer() {
  // Checkpoint store — tracks processed offsets in Blob Storage
  const containerClient = new ContainerClient(
    "https://mystorage.blob.core.windows.net/checkpoints",
    credential
  );
  await containerClient.createIfNotExists();

  const checkpointStore = new BlobCheckpointStore(containerClient);
  const consumer = new EventHubConsumerClient(
    "$Default",
    fullyQualifiedNamespace,
    eventHubName,
    credential,
    checkpointStore
  );

  consumer.subscribe({
    processEvents: async (events, context) => {
      for (const event of events) {
        console.log(`Partition ${context.partitionId}:`, event.body);
      }
      // Checkpoint after processing — won't reprocess on restart
      await context.updateCheckpoint(events[events.length - 1]);
    },
    processError: async (err, context) => {
      console.error(`Error on partition ${context.partitionId}:`, err);
    },
  });
}
```

---

## Part 3: Event Grid
## (analogous to AWS EventBridge)

Event Grid is Azure's event routing service — it receives events from Azure resources (blob uploaded, VM deleted, resource created) or custom sources and fans them out to subscribers (Functions, Logic Apps, webhooks, Service Bus).

```bash
# Subscribe a Function to blob upload events
az eventgrid event-subscription create \
  --name blobUploadSubscription \
  --source-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --endpoint-type azurefunction \
  --endpoint /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function-app/functions/processBlobUpload \
  --included-event-types Microsoft.Storage.BlobCreated

# Custom event topic (your app publishes events, others subscribe)
az eventgrid topic create \
  --resource-group myRG \
  --name myCustomTopic \
  --location eastus
```

```javascript
// Publish a custom event
const { EventGridPublisherClient, AzureKeyCredential } = require("@azure/eventgrid");

const client = new EventGridPublisherClient(
  "https://mycustomtopic.eastus-1.eventgrid.azure.net/api/events",
  "EventGrid",
  new DefaultAzureCredential()
);

await client.send([
  {
    eventType: "MyApp.OrderPlaced",
    subject: "orders/order-123",
    dataVersion: "1.0",
    data: { orderId: "order-123", customerId: "cust-456", total: 299.99 },
  },
]);
```

---

## When to Use What

| Scenario | Use |
|----------|-----|
| Decouple microservices, task queues | **Service Bus Queue** |
| Broadcast events to multiple consumers | **Service Bus Topic** |
| FIFO / ordered processing per entity | **Service Bus Queue with Sessions** |
| IoT telemetry, log ingestion, clickstreams | **Event Hubs** |
| Replay events, multiple independent readers | **Event Hubs** |
| React to Azure resource events (blob upload, VM start) | **Event Grid** |
| Orchestrate workflows across services | **Event Grid + Functions** |
| Simple low-volume queue | **Azure Queue Storage** |