# Google Cloud Pub/Sub

Cloud Pub/Sub is GCP's fully managed, real-time messaging service for decoupling services and building event-driven architectures. It combines the push/pull flexibility of SQS with the fan-out capabilities of SNS — in a single service. The GCP equivalent of AWS SQS + SNS.

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | A named channel where publishers send messages |
| **Subscription** | A named attachment to a topic — each subscription gets a copy of every message |
| **Message** | Data (up to 10 MB) + optional attributes (key-value pairs) + message ID + timestamp |
| **Publisher** | Any service or app that sends messages to a topic |
| **Subscriber** | A service that receives messages from a subscription |
| **Acknowledgment (ack)** | Subscriber confirms it processed a message — removes it from the subscription |
| **Ack Deadline** | Time allowed to ack before Pub/Sub re-delivers (default: 10s, max: 600s) |
| **Dead Letter Topic** | Messages that fail to be acked after N attempts are forwarded here |

---

## Delivery Models

### Pull (like SQS)
Subscribers poll for messages at their own pace — ideal for batch workers or services that control their own throughput.

### Push (like SNS)
Pub/Sub delivers messages to an HTTPS endpoint (your webhook or Cloud Run service) — ideal for real-time event-driven triggers.

### BigQuery Subscription
Pub/Sub writes messages directly to a BigQuery table — no consumer code needed.

### Cloud Storage Subscription
Pub/Sub writes messages in batches to Cloud Storage files.

---

## CLI Commands

### Topics
```bash
# Create a topic
gcloud pubsub topics create orders-topic

# List topics
gcloud pubsub topics list

# Publish a message
gcloud pubsub topics publish orders-topic \
  --message='{"orderId":"123","amount":99.99}' \
  --attribute="source=checkout,priority=high"

# Delete a topic
gcloud pubsub topics delete orders-topic
```

### Subscriptions
```bash
# Create a pull subscription
gcloud pubsub subscriptions create orders-worker-sub \
  --topic=orders-topic \
  --ack-deadline=60 \
  --message-retention-duration=7d

# Create a push subscription (delivers to a webhook)
gcloud pubsub subscriptions create orders-push-sub \
  --topic=orders-topic \
  --push-endpoint=https://myapp.run.app/webhooks/orders \
  --push-auth-service-account=pubsub-invoker@my-project.iam.gserviceaccount.com

# Create a BigQuery subscription (direct to table)
gcloud pubsub subscriptions create orders-bq-sub \
  --topic=orders-topic \
  --bigquery-table=my-project:my_dataset.orders_raw \
  --write-metadata

# Create a Dead Letter Topic subscription
gcloud pubsub subscriptions create orders-worker-sub \
  --topic=orders-topic \
  --dead-letter-topic=orders-deadletter-topic \
  --max-delivery-attempts=5

# List subscriptions
gcloud pubsub subscriptions list

# Pull messages manually (testing)
gcloud pubsub subscriptions pull orders-worker-sub \
  --limit=5 \
  --auto-ack          # Acks immediately after display
```

---

## Code Examples (Node.js)

### Publisher
```javascript
const { PubSub } = require('@google-cloud/pubsub');
const pubsub = new PubSub();

async function publishOrder(order) {
  const topic = pubsub.topic('orders-topic');
  const messageId = await topic.publishMessage({
    data: Buffer.from(JSON.stringify(order)),
    attributes: {
      source: 'checkout-service',
      version: '2',
    },
  });
  console.log(`Published message ${messageId}`);
}

// Batch publishing (more efficient for high volume)
const topic = pubsub.topic('events-topic', {
  batching: {
    maxMessages: 100,
    maxMilliseconds: 10,
  },
});
```

### Pull Subscriber
```javascript
const subscription = pubsub.subscription('orders-worker-sub', {
  flowControl: {
    maxMessages: 10,    // Process at most 10 messages concurrently
  },
});

subscription.on('message', async (message) => {
  try {
    const order = JSON.parse(message.data.toString());
    console.log('Processing order:', order.orderId);

    await processOrder(order);

    message.ack(); // Success — remove from queue
  } catch (err) {
    console.error('Processing failed:', err);
    message.nack(); // Re-deliver after ack deadline
  }
});

subscription.on('error', (err) => {
  console.error('Subscription error:', err);
});
```

### One-time Pull (polling pattern)
```javascript
const [messages] = await pubsub
  .subscription('orders-worker-sub')
  .pull({ maxMessages: 10 });

for (const message of messages) {
  const data = JSON.parse(message.message.data.toString());
  console.log(data);
  // Acknowledge
  await pubsub.subscription('orders-worker-sub').ack([message.ackId]);
}
```

### Push Subscriber (Cloud Run / Express webhook)
```javascript
const express = require('express');
const app = express();
app.use(express.json());

app.post('/webhooks/orders', (req, res) => {
  // Pub/Sub push delivers: { message: { data: base64, attributes: {}, messageId: '' } }
  const message = req.body.message;
  if (!message) return res.status(400).send('No message');

  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());
  console.log('Received order:', data);

  // Return 2xx to acknowledge — anything else = retry
  res.status(204).send();
});
```

---

## Fan-Out Pattern (One Topic → Multiple Subscribers)

```bash
# Create subscriptions for different consumers of the same topic
gcloud pubsub subscriptions create orders-email-sub --topic=orders-topic
gcloud pubsub subscriptions create orders-inventory-sub --topic=orders-topic
gcloud pubsub subscriptions create orders-analytics-sub --topic=orders-topic

# Each subscription independently receives every message
# Email service, inventory service, and analytics service all get notified
```

---

## Filtering Messages

Subscribers can filter which messages they receive without additional code:

```bash
# Only receive high-priority orders
gcloud pubsub subscriptions create orders-urgent-sub \
  --topic=orders-topic \
  --message-filter='attributes.priority = "high"'

# Only receive orders from a specific region
gcloud pubsub subscriptions create orders-eu-sub \
  --topic=orders-topic \
  --message-filter='attributes.region = "EU"'
```

---

## Dead Letter Handling

```bash
# 1. Create a dead letter topic
gcloud pubsub topics create orders-deadletter

# 2. Create a subscription on the original topic with a DLT
gcloud pubsub subscriptions create orders-worker-sub \
  --topic=orders-topic \
  --dead-letter-topic=orders-deadletter \
  --max-delivery-attempts=5 \
  --ack-deadline=60

# 3. Subscribe to the dead letter topic to handle failures
gcloud pubsub subscriptions create orders-deadletter-sub \
  --topic=orders-deadletter
```

---

## Ordering Guarantees

```bash
# Enable message ordering (messages with the same ordering key are delivered in order)
gcloud pubsub topics create ordered-events-topic \
  --message-retention-duration=1d

gcloud pubsub subscriptions create ordered-sub \
  --topic=ordered-events-topic \
  --enable-message-ordering
```

In code (publisher must set an ordering key):
```javascript
await topic.publishMessage({
  data: Buffer.from(JSON.stringify(event)),
  orderingKey: `user-${userId}`,   // All events for same user arrive in order
});
```

---

## Pub/Sub vs SQS/SNS Comparison

| Feature | Pub/Sub | SQS | SNS |
|---------|---------|-----|-----|
| Pull-based delivery | ✅ | ✅ | ❌ |
| Push/webhook delivery | ✅ | ❌ | ✅ |
| Fan-out (1 topic → N subscribers) | ✅ | ❌ | ✅ |
| Message filtering | ✅ | ❌ | ✅ |
| FIFO / ordering | ✅ (with ordering key) | ✅ (FIFO queue) | ❌ |
| Dead letter support | ✅ | ✅ | ❌ |
| Max message size | 10 MB | 256 KB | 256 KB |
| Retention | 31 days | 14 days | N/A |
| BigQuery direct write | ✅ | ❌ | ❌ |