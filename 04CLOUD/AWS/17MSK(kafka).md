# Amazon MSK (Managed Streaming for Apache Kafka)

Amazon MSK is AWS's fully managed service for running **Apache Kafka** — a distributed event streaming platform for building real-time data pipelines and streaming applications.

---

## What Is Apache Kafka?

Apache Kafka is an open-source distributed streaming system built around a **publish-subscribe** model with persistent, ordered, replayable event logs.

**Core capabilities:**
- High-throughput ingestion of real-time data (millions of events/second)
- Durable, ordered event storage with configurable retention
- Pub/sub messaging with multiple independent consumers
- Stream processing with Kafka Streams or Apache Flink

**Common industries:** finance (fraud detection), e-commerce (order tracking), IoT (telemetry), gaming (activity tracking), adtech (clickstream).

---

## Kafka Core Concepts

| Concept | Description |
|---------|-------------|
| **Event / Message** | The unit of data — a key, value, timestamp, and optional headers |
| **Topic** | A named, ordered log of events (like a table in a database) |
| **Partition** | A topic is split into partitions for parallelism and scalability |
| **Offset** | The position of a message within a partition — consumers track their offset |
| **Producer** | Writes events to topics |
| **Consumer** | Reads events from topics at its own pace |
| **Consumer Group** | A set of consumers sharing the work of reading a topic — each partition goes to one consumer in the group |
| **Broker** | A Kafka server node that stores partitions |
| **Cluster** | A group of brokers working together |
| **Replication Factor** | How many brokers store a copy of each partition (for fault tolerance) |
| **Retention** | How long Kafka keeps messages (by time or size) — default: 7 days |

---

## Kafka vs SQS vs SNS

| Feature | Kafka (MSK) | SQS | SNS |
|---------|------------|-----|-----|
| Message persistence | Yes (configurable retention) | Until consumed (14 days max) | No |
| Message replay | Yes (seek to any offset) | No | No |
| Multiple consumers | Yes (each consumer group reads independently) | No (one consumer per message) | Yes (push to all subscribers) |
| Ordering | Per partition | FIFO queues only | No |
| Throughput | Extremely high | High | High |
| Complexity | High | Low | Low |
| Best for | Event sourcing, analytics pipelines, high-throughput streams | Task queues, decoupling | Fan-out notifications |

> **Rule of thumb**: Use SQS/SNS for standard microservice messaging. Use Kafka/MSK when you need message replay, extremely high throughput, multiple independent consumer groups, or a persistent event log.

---

## What MSK Adds

MSK removes operational burden while running fully open-source Kafka:

- **Managed brokers**: AWS handles provisioning, patching, and replacing failed brokers
- **Multi-AZ replication**: Built-in fault tolerance across Availability Zones
- **Monitoring**: CloudWatch metrics and open monitoring with Prometheus
- **Security**: IAM authentication, TLS encryption, VPC isolation, SASL/SCRAM
- **MSK Connect**: Run Kafka Connect connectors to stream data in/out of Kafka (S3, DynamoDB, RDS, etc.)
- **MSK Replicator**: Cross-cluster or cross-region replication
- **Serverless mode**: Auto-scaling with no capacity management

---

## Creating an MSK Cluster

### Via Console
1. Go to **Amazon MSK → Create cluster**
2. Choose **Quick create** or **Custom create**
3. Select: cluster type (Provisioned or Serverless), Kafka version, broker instance type
4. Configure networking (VPC, subnets — choose at least 2 AZs for HA)
5. Configure security (TLS, SASL, IAM)
6. Create

### Via CLI
```bash
# Create a serverless cluster (simplest)
aws kafka create-cluster-v2 \
  --cluster-name my-kafka-cluster \
  --serverless '{
    "VpcConfigs": [{
      "SubnetIds": ["subnet-abc123", "subnet-def456"],
      "SecurityGroupIds": ["sg-xyz789"]
    }]
  }'

# List clusters
aws kafka list-clusters-v2

# Get broker endpoints (needed to connect)
aws kafka get-bootstrap-brokers --cluster-arn <cluster-arn>
```

---

## Working with Kafka (Producer & Consumer)

### Node.js with `kafkajs`

```bash
npm install kafkajs
```

```javascript
const { Kafka } = require('kafkajs');

const kafka = new Kafka({
  clientId: 'my-app',
  brokers: ['broker1:9092', 'broker2:9092'], // from MSK bootstrap brokers
  ssl: true, // MSK requires TLS
});

// Producer
const producer = kafka.producer();

const sendEvent = async () => {
  await producer.connect();
  await producer.send({
    topic: 'user-events',
    messages: [
      {
        key: 'user123',
        value: JSON.stringify({ event: 'page_view', page: '/home', timestamp: Date.now() }),
      }
    ],
  });
  await producer.disconnect();
};

// Consumer
const consumer = kafka.consumer({ groupId: 'analytics-group' });

const runConsumer = async () => {
  await consumer.connect();
  await consumer.subscribe({ topic: 'user-events', fromBeginning: false });

  await consumer.run({
    eachMessage: async ({ topic, partition, message }) => {
      const event = JSON.parse(message.value.toString());
      console.log(`Received [partition ${partition} @ offset ${message.offset}]:`, event);
    },
  });
};
```

### Python with `confluent-kafka`

```bash
pip install confluent-kafka
```

```python
from confluent_kafka import Producer, Consumer, KafkaError
import json

# Producer
producer = Producer({
    'bootstrap.servers': 'broker1:9092,broker2:9092',
    'security.protocol': 'SSL',
})

producer.produce(
    'user-events',
    key='user123',
    value=json.dumps({'event': 'purchase', 'amount': 99.99}),
)
producer.flush()

# Consumer
consumer = Consumer({
    'bootstrap.servers': 'broker1:9092,broker2:9092',
    'group.id': 'payment-group',
    'auto.offset.reset': 'latest',
    'security.protocol': 'SSL',
})

consumer.subscribe(['user-events'])

while True:
    msg = consumer.poll(timeout=1.0)
    if msg and not msg.error():
        data = json.loads(msg.value())
        print(f"Offset {msg.offset()}: {data}")
```

---

## Kafka Topics via CLI (using Kafka tools)

```bash
# Create a topic (run from a client EC2 in the same VPC)
kafka-topics.sh --bootstrap-server <broker-endpoints> \
  --create \
  --topic user-events \
  --partitions 3 \
  --replication-factor 2

# List topics
kafka-topics.sh --bootstrap-server <broker-endpoints> --list

# Describe topic (see partition distribution)
kafka-topics.sh --bootstrap-server <broker-endpoints> \
  --describe --topic user-events

# Produce test messages
kafka-console-producer.sh \
  --bootstrap-server <broker-endpoints> \
  --topic user-events

# Consume messages (from beginning)
kafka-console-consumer.sh \
  --bootstrap-server <broker-endpoints> \
  --topic user-events \
  --from-beginning \
  --group test-group
```

---

## MSK Connect (Kafka Connect)

Stream data between Kafka and external systems without writing custom code:

**Source connectors** (external system → Kafka):
- Debezium (CDC from MySQL/PostgreSQL/MongoDB)
- S3 Source, DynamoDB Streams Source

**Sink connectors** (Kafka → external system):
- S3 Sink (archive events to S3)
- OpenSearch Sink (index events for search)
- DynamoDB Sink, Redshift Sink

---

## Common Architectures

### Event Sourcing Pipeline
```
Microservices → MSK Topic → Stream Processor (Flink/Lambda) → DynamoDB / OpenSearch / Redshift
```

### CDC (Change Data Capture)
```
RDS/MySQL → Debezium (MSK Connect) → MSK Topic → Multiple consumers
```
Capture every database change and propagate it to other systems in real time.

### Log Aggregation
```
EC2 / ECS / Lambda → Fluent Bit → MSK Topic → OpenSearch / S3
```

---

## Monitoring

Key CloudWatch metrics for MSK:

| Metric | What to Watch |
|--------|--------------|
| `KafkaDataLogsDiskUsed` | Disk usage per broker |
| `OffsetLag` | How far behind consumers are (consumer group lag) |
| `UnderReplicatedPartitions` | Should always be 0 — indicates replication issues |
| `ActiveControllerCount` | Should always be 1 |
| `BytesInPerSec` / `BytesOutPerSec` | Throughput per broker |

---

## Best Practices

- **Partitioning**: Choose the number of partitions based on expected throughput — more partitions = more parallelism (can't easily reduce later).
- **Replication factor**: Use at least 3 for production data durability.
- **Consumer group IDs**: Give each independent service its own group ID — they all read the full topic independently.
- **Message keys**: Use meaningful keys (e.g., `userId`) so related events always go to the same partition, preserving order.
- **Schema Registry**: Use AWS Glue Schema Registry or Confluent Schema Registry to manage Avro/JSON schemas and prevent incompatible producers/consumers.
- **Retention**: Set retention based on replay needs — longer retention = higher storage cost.
- **Monitor consumer lag**: An increasing `OffsetLag` means consumers are falling behind — scale up consumers.