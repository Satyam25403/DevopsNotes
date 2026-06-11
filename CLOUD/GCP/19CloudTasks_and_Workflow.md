# GCP Cloud Tasks & Cloud Workflows

---

# Part 1: Cloud Tasks (Async Job Queue)

Cloud Tasks is GCP's fully managed service for creating, dispatching, and managing asynchronous task queues. It decouples work from the services that generate it — the GCP equivalent of AWS SQS with delayed delivery + AWS Step Functions task dispatch.

---

## What Cloud Tasks Is For

- **Offload slow work**: A user uploads a file → your API enqueues a task to process it → returns immediately → worker processes asynchronously
- **Rate limiting**: Control how fast tasks are dispatched to workers (e.g., throttle 3rd-party API calls)
- **Retries with backoff**: Automatic retry with configurable exponential backoff on failures
- **Scheduled/delayed tasks**: Dispatch a task to run at a specific future time
- **Deduplication**: Named tasks prevent duplicate processing

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Queue** | A named container with delivery settings (rate, retry, backoff) |
| **Task** | A unit of work: an HTTP endpoint to call + optional payload |
| **Worker** | Any HTTPS endpoint (Cloud Run, App Engine, GCE) that processes tasks |
| **Schedule Time** | Optional future time when a task should first be dispatched |
| **Dispatch Rate** | Max tasks/second dispatched from a queue |

---

## CLI Commands

```bash
# Create a task queue
gcloud tasks queues create image-processing \
  --location=us-central1 \
  --max-dispatches-per-second=10 \
  --max-concurrent-dispatches=5 \
  --max-attempts=5 \
  --min-backoff=10s \
  --max-backoff=300s

# Create a task (dispatch to an HTTP worker)
gcloud tasks create-http-task \
  --queue=image-processing \
  --location=us-central1 \
  --url=https://my-worker.run.app/tasks/process-image \
  --method=POST \
  --body-content='{"imageId":"img-123","userId":"user-456"}' \
  --header="Content-Type:application/json"

# Schedule a task for the future (dispatch in 30 minutes)
gcloud tasks create-http-task \
  --queue=image-processing \
  --location=us-central1 \
  --url=https://my-worker.run.app/tasks/process-image \
  --method=POST \
  --schedule-time=$(date -d '+30 minutes' --utc +%Y-%m-%dT%H:%M:%SZ) \
  --body-content='{"imageId":"img-456"}'

# List queues
gcloud tasks queues list --location=us-central1

# List tasks in a queue
gcloud tasks list --queue=image-processing --location=us-central1

# Pause / resume a queue
gcloud tasks queues pause image-processing --location=us-central1
gcloud tasks queues resume image-processing --location=us-central1

# Purge all tasks from a queue
gcloud tasks queues purge image-processing --location=us-central1
```

---

## Creating Tasks from Code (Node.js)

```javascript
const { CloudTasksClient } = require('@google-cloud/tasks');
const client = new CloudTasksClient();

async function enqueueTask(imageId, userId) {
  const queuePath = client.queuePath('my-project', 'us-central1', 'image-processing');

  const task = {
    httpRequest: {
      httpMethod: 'POST',
      url: 'https://my-worker.run.app/tasks/process-image',
      headers: { 'Content-Type': 'application/json' },
      body: Buffer.from(JSON.stringify({ imageId, userId })).toString('base64'),
      oidcToken: {
        serviceAccountEmail: 'tasks-invoker@my-project.iam.gserviceaccount.com',
      },
    },
    // Optional: schedule for 5 minutes from now
    scheduleTime: {
      seconds: Math.floor(Date.now() / 1000) + 300,
    },
  };

  const [response] = await client.createTask({ parent: queuePath, task });
  console.log(`Task created: ${response.name}`);
}

// Worker (Cloud Run handler)
app.post('/tasks/process-image', async (req, res) => {
  const { imageId, userId } = req.body;
  try {
    await processImage(imageId, userId);
    res.status(204).send();     // 2xx = success (task removed from queue)
  } catch (err) {
    console.error(err);
    res.status(500).send();     // 5xx = failure (task retried)
  }
});
```

---

## Queue Configuration Options

```bash
gcloud tasks queues update image-processing \
  --location=us-central1 \
  --max-dispatches-per-second=50 \    # Rate limit to backend
  --max-concurrent-dispatches=20 \    # Max parallel in-flight tasks
  --max-attempts=10 \                 # Total attempts including first
  --min-backoff=5s \                  # Wait at least 5s before retry
  --max-backoff=600s \                # Max wait between retries
  --max-doublings=5                   # Number of times to double backoff
```

---

# Part 2: Cloud Workflows (Orchestration)

Cloud Workflows is GCP's fully managed service for orchestrating sequences of HTTP-based API calls and GCP service calls — the GCP equivalent of AWS Step Functions.

---

## What Cloud Workflows Is For

- **Orchestrate multi-step processes**: Call Cloud Run → write to Firestore → call external API → send Pub/Sub message
- **Error handling and retries**: Built-in try/catch and retry policies per step
- **Long-running processes**: Workflows can pause and wait (callbacks, delays)
- **Replace complex glue code**: No more chaining Lambda functions or Cloud Functions with ad-hoc queues

---

## Workflow Definition (YAML)

```yaml
# order-fulfillment.yaml
main:
  params: [args]
  steps:
    - validate_order:
        call: http.post
        args:
          url: https://validation-service.run.app/validate
          body:
            orderId: ${args.orderId}
          auth:
            type: OIDC
        result: validation_response

    - check_validation:
        switch:
          - condition: ${validation_response.body.valid == false}
            next: reject_order

    - process_payment:
        call: http.post
        args:
          url: https://payment-service.run.app/charge
          body:
            orderId: ${args.orderId}
            amount: ${args.amount}
          auth:
            type: OIDC
        result: payment_response

    - notify_warehouse:
        call: googleapis.pubsub.v1.projects.topics.publish
        args:
          topic: ${"projects/" + sys.get_env("GOOGLE_CLOUD_PROJECT_ID") + "/topics/new-orders"}
          body:
            messages:
              - data: ${base64.encode(json.encode(args))}
        result: pubsub_result

    - update_database:
        call: http.patch
        args:
          url: ${"https://api.run.app/orders/" + args.orderId}
          body:
            status: "confirmed"
            paymentId: ${payment_response.body.paymentId}
          auth:
            type: OIDC

    - return_result:
        return:
          status: "success"
          orderId: ${args.orderId}
          paymentId: ${payment_response.body.paymentId}

    - reject_order:
        return:
          status: "rejected"
          orderId: ${args.orderId}
          reason: ${validation_response.body.reason}
```

---

## CLI Commands

```bash
# Deploy a workflow
gcloud workflows deploy order-fulfillment \
  --source=order-fulfillment.yaml \
  --location=us-central1 \
  --service-account=workflow-sa@my-project.iam.gserviceaccount.com

# Execute a workflow
gcloud workflows run order-fulfillment \
  --location=us-central1 \
  --data='{"orderId":"123","amount":99.99}'

# List executions
gcloud workflows executions list order-fulfillment \
  --location=us-central1

# Describe execution (view result)
gcloud workflows executions describe EXECUTION_ID \
  --workflow=order-fulfillment \
  --location=us-central1

# Cancel a running execution
gcloud workflows executions cancel EXECUTION_ID \
  --workflow=order-fulfillment \
  --location=us-central1
```

---

## Triggering Workflows

```javascript
// Trigger from code (Node.js)
const { ExecutionsClient } = require('@google-cloud/workflows');
const client = new ExecutionsClient();

const [execution] = await client.createExecution({
  parent: client.workflowPath('my-project', 'us-central1', 'order-fulfillment'),
  execution: {
    argument: JSON.stringify({ orderId: '123', amount: 99.99 }),
  },
});
console.log(`Execution started: ${execution.name}`);
```

```bash
# Schedule a workflow via Cloud Scheduler
gcloud scheduler jobs create http daily-cleanup-job \
  --location=us-central1 \
  --schedule="0 2 * * *" \
  --uri="https://workflowexecutions.googleapis.com/v1/projects/my-project/locations/us-central1/workflows/daily-cleanup/executions" \
  --message-body='{}' \
  --oauth-service-account-email=scheduler-sa@my-project.iam.gserviceaccount.com
```

---

## Cloud Tasks vs Pub/Sub vs Workflows

| Feature | Cloud Tasks | Pub/Sub | Cloud Workflows |
|---------|-------------|---------|----------------|
| Use case | Async jobs, rate-limited dispatch | Event fan-out, streaming | Multi-step orchestration |
| Delivery | Exactly-once to one worker | At-least-once to N subscribers | Single, ordered execution |
| Scheduling | ✅ Future time | ❌ | ✅ Delays/waits |
| Retries | ✅ Configurable | ✅ Nack-based | ✅ Per-step policies |
| Visibility | Per-task status | Limited | Full execution history |
| Long-running | ❌ (max 30 min) | ❌ | ✅ (up to 1 year) |
| Best for | Worker queues, webhooks, rate control | Decoupling services, event-driven | Business process orchestration |