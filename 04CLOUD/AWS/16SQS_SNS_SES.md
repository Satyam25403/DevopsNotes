# AWS Messaging Services: SQS, SNS & SES

AWS provides three purpose-built messaging services. Understanding when to use each is key to building decoupled, scalable architectures.

| Service | Model | Best For |
|---------|-------|----------|
| **SQS** | Pull-based queue | Decoupling services, async task processing |
| **SNS** | Push-based pub/sub | Fan-out, notifications to multiple subscribers |
| **SES** | Email API/SMTP | Transactional and marketing emails |

---

## Amazon SQS (Simple Queue Service)

SQS is a fully managed **message queue** for decoupling components of distributed systems. Producers send messages to a queue; consumers **poll** and process them at their own pace.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Standard Queue** | Best-effort ordering, at-least-once delivery, nearly unlimited throughput |
| **FIFO Queue** | Exactly-once processing, strict ordering, up to 3,000 msg/sec |
| **Visibility Timeout** | Time a message is hidden from other consumers while being processed (default: 30s) |
| **Dead-Letter Queue (DLQ)** | Receives messages that failed to be processed after N retries |
| **Message Retention** | How long unprocessed messages are kept (1 minute – 14 days, default: 4 days) |
| **Long Polling** | Consumer waits up to 20s for messages — reduces empty responses and cost |

### CLI Commands

```bash
# Create a standard queue
aws sqs create-queue --queue-name my-queue

# Create a FIFO queue
aws sqs create-queue \
  --queue-name my-queue.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=true

# Send a message
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --message-body '{"userId": "123", "action": "process_order"}'

# Receive messages (long polling)
aws sqs receive-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --wait-time-seconds 20 \
  --max-number-of-messages 10

# Delete a message after processing
aws sqs delete-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --receipt-handle <receipt-handle-from-receive>

# Get queue attributes (message count, etc.)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --attribute-names All
```

### Node.js Example

```javascript
const { SQSClient, SendMessageCommand, ReceiveMessageCommand, DeleteMessageCommand } = require("@aws-sdk/client-sqs");

const sqs = new SQSClient({ region: "us-east-1" });
const QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789012/my-queue";

// Producer: send a message
const sendMessage = async (payload) => {
  await sqs.send(new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(payload),
    MessageAttributes: {
      EventType: { DataType: "String", StringValue: "order.created" }
    }
  }));
};

// Consumer: poll and process
const processMessages = async () => {
  const response = await sqs.send(new ReceiveMessageCommand({
    QueueUrl: QUEUE_URL,
    MaxNumberOfMessages: 10,
    WaitTimeSeconds: 20, // long polling
  }));

  for (const message of response.Messages || []) {
    const body = JSON.parse(message.Body);
    console.log("Processing:", body);
    
    // Delete after successful processing
    await sqs.send(new DeleteMessageCommand({
      QueueUrl: QUEUE_URL,
      ReceiptHandle: message.ReceiptHandle,
    }));
  }
};
```

### Dead-Letter Queue Setup

```bash
# Create DLQ first
aws sqs create-queue --queue-name my-queue-dlq

# Get DLQ ARN
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue-dlq \
  --attribute-names QueueArn

# Attach DLQ to main queue (messages retry 3 times, then go to DLQ)
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/my-queue \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789012:my-queue-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

### Common SQS Patterns

**Decoupled microservices:**
```
API → SQS Queue → Worker Lambda/EC2 → Database
```
The API responds immediately; the worker processes asynchronously.

**Lambda as SQS consumer (event source mapping):**
```bash
aws lambda create-event-source-mapping \
  --function-name my-processor \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:my-queue \
  --batch-size 10
```
Lambda polls the queue automatically — no consumer loop needed.

---

## Amazon SNS (Simple Notification Service)

SNS is a fully managed **pub/sub messaging service**. Publishers send a message to a **Topic**; SNS immediately **pushes** it to all subscribers.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Topic** | The channel to publish to and subscribe from |
| **Publisher** | Any service or app that sends messages to a topic |
| **Subscriber** | Endpoint that receives messages (Lambda, SQS, HTTP, email, SMS, mobile push) |
| **Fan-out** | One message → delivered to multiple subscribers simultaneously |
| **Message Filtering** | Subscribers receive only messages matching their filter policy |
| **FIFO Topics** | Ordered, exactly-once delivery to FIFO SQS queues |

### CLI Commands

```bash
# Create a topic
aws sns create-topic --name my-topic

# Subscribe Lambda to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --protocol lambda \
  --notification-endpoint arn:aws:lambda:us-east-1:123456789012:function:my-function

# Subscribe SQS to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:123456789012:my-queue

# Subscribe an email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --protocol email \
  --notification-endpoint user@example.com

# Publish a message
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --subject "Alert" \
  --message "CPU usage exceeded 90% on prod server"

# Publish with message attributes (used for filtering)
aws sns publish \
  --topic-arn arn:aws:sns:us-east-1:123456789012:my-topic \
  --message "Order placed" \
  --message-attributes '{"eventType": {"DataType": "String", "StringValue": "order.created"}}'
```

### Node.js Example

```javascript
const { SNSClient, PublishCommand } = require("@aws-sdk/client-sns");

const sns = new SNSClient({ region: "us-east-1" });

const notify = async (message, subject) => {
  await sns.send(new PublishCommand({
    TopicArn: "arn:aws:sns:us-east-1:123456789012:my-topic",
    Message: message,
    Subject: subject,
    MessageAttributes: {
      eventType: { DataType: "String", StringValue: "order.created" }
    }
  }));
};
```

### Message Filtering

Subscribers can filter which messages they receive:
```json
{
  "eventType": ["order.created", "order.updated"]
}
```
A subscriber with this filter only receives messages where `eventType` is `order.created` or `order.updated`.

### Fan-out Pattern (SNS + SQS)

The most common production pattern — SNS pushes to multiple SQS queues for independent processing:

```
Order Event → SNS Topic
                ├── SQS Queue → Email Service Lambda
                ├── SQS Queue → Inventory Service Lambda
                └── SQS Queue → Analytics Service Lambda
```

Benefits: each service processes independently, at its own pace, with its own DLQ.

---

## Amazon SES (Simple Email Service)

SES is a cloud-based email platform for sending transactional and marketing emails at scale with high deliverability.

### Key Features

- **High deliverability**: Dedicated IPs, shared IPs, or bring your own IP
- **Authentication**: DKIM, SPF, DMARC support to prevent spoofing
- **Email receiving**: Receive inbound email and trigger Lambda or S3
- **Bounce and complaint tracking**: Automatically manage suppression lists
- **Templates**: Store and reuse email templates with variable substitution
- **Configuration Sets**: Track email events (opens, clicks, bounces, complaints)

### Setup

1. **Verify a domain or email address** (required before sending)
2. **Exit sandbox mode** (by default SES is sandboxed — you can only send to verified addresses; request production access to send to anyone)
3. Set up **DKIM** records in Route 53 for better deliverability

### CLI Commands

```bash
# Verify an email address (sandbox)
aws ses verify-email-identity --email-address sender@example.com

# Send a simple email
aws ses send-email \
  --from "sender@yourdomain.com" \
  --to "recipient@example.com" \
  --subject "Welcome!" \
  --text "Thanks for signing up."

# Send HTML email
aws ses send-email \
  --from "sender@yourdomain.com" \
  --destination '{"ToAddresses":["recipient@example.com"]}' \
  --message '{
    "Subject": {"Data": "Welcome!"},
    "Body": {
      "Html": {"Data": "<h1>Welcome to our app!</h1>"},
      "Text": {"Data": "Welcome to our app!"}
    }
  }'

# Create an email template
aws ses create-template \
  --template '{
    "TemplateName": "WelcomeEmail",
    "SubjectPart": "Welcome, {{name}}!",
    "HtmlPart": "<h1>Hi {{name}}, welcome to {{appName}}!</h1>",
    "TextPart": "Hi {{name}}, welcome to {{appName}}!"
  }'

# Send using a template
aws ses send-templated-email \
  --source "noreply@yourdomain.com" \
  --destination '{"ToAddresses":["user@example.com"]}' \
  --template "WelcomeEmail" \
  --template-data '{"name":"Alice","appName":"MyApp"}'
```

### Node.js Example

```javascript
const { SESClient, SendEmailCommand } = require("@aws-sdk/client-ses");

const ses = new SESClient({ region: "us-east-1" });

const sendWelcomeEmail = async (toEmail, userName) => {
  await ses.send(new SendEmailCommand({
    Source: "noreply@yourdomain.com",
    Destination: { ToAddresses: [toEmail] },
    Message: {
      Subject: { Data: `Welcome, ${userName}!` },
      Body: {
        Html: { Data: `<h1>Hi ${userName}, thanks for joining!</h1>` },
        Text: { Data: `Hi ${userName}, thanks for joining!` },
      },
    },
  }));
};
```

---

## Choosing Between SQS, SNS, and SES

| Scenario | Use |
|----------|-----|
| Process tasks asynchronously (one consumer) | SQS |
| Notify multiple services of an event simultaneously | SNS |
| Send welcome/reset/receipt emails | SES |
| Rate-limit processing of high-volume events | SQS |
| Mobile push notifications | SNS |
| Marketing email campaigns | SES |
| One event → trigger multiple independent systems | SNS → multiple SQS queues |
| Ordered, exactly-once message processing | SQS FIFO or SNS FIFO |

---

## SQS vs SNS: Key Difference

| | SQS | SNS |
|--|-----|-----|
| Delivery model | Pull (consumer polls) | Push (SNS pushes to subscribers) |
| Consumers | One consumer per message | Many subscribers, all get the message |
| Persistence | Messages persist until deleted | No persistence — deliver or drop |
| Best for | Task queues, async processing | Fan-out, event notification |