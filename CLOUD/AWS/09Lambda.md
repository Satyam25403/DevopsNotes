# AWS Lambda (Serverless Functions)

AWS Lambda lets you run code without provisioning or managing servers. You upload your function, define a trigger, and AWS handles everything else — scaling, availability, and infrastructure.

---

## Core Concepts

- **Serverless**: No servers to manage. AWS provisions compute on demand.
- **Event-driven**: Lambda runs in response to events — HTTP requests, S3 uploads, queue messages, scheduled cron jobs, etc.
- **Pay-per-use**: Billed per request and per millisecond of execution time. Zero cost when idle.
- **Automatic scaling**: Scales from zero to thousands of concurrent executions without any configuration.

---

## What You Provide

- Your function code
- A runtime (Node.js, Python, Java, Go, Ruby, .NET, or custom)
- A trigger (what causes the function to run)
- IAM permissions (what AWS resources the function can access)
- Optional environment variables, memory allocation (128 MB – 10 GB), and timeout (max 15 min)

---

## Supported Runtimes

- Node.js (14, 16, 18, 20)
- Python (3.9, 3.10, 3.11, 3.12)
- Java (8, 11, 17, 21)
- Go
- Ruby
- .NET (6, 8)
- Custom runtimes (via Lambda Layers)

---

## Common Triggers

| Trigger | Use Case |
|---------|----------|
| **API Gateway** | REST/HTTP API endpoints |
| **S3** | Process files on upload |
| **DynamoDB Streams** | React to table changes |
| **SQS** | Process messages from a queue |
| **SNS** | React to notifications |
| **EventBridge (cron)** | Scheduled jobs |
| **Kinesis** | Real-time stream processing |
| **ALB** | HTTP traffic without API Gateway |
| **CloudWatch Logs** | Process log events |

---

## Sample Node.js Lambda Function

```javascript
exports.handler = async (event, context) => {
  console.log("Received event:", JSON.stringify(event));

  // event: data from the trigger (e.g., API Gateway request, S3 event)
  // context: metadata about the invocation (function name, memory limit, etc.)

  return {
    statusCode: 200,
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ message: "Hello from Lambda!" })
  };
};
```

### Sample Python Function
```python
import json

def lambda_handler(event, context):
    print("Received event:", json.dumps(event))
    
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'message': 'Hello from Lambda!'})
    }
```

---

## Sample Event Payloads

### API Gateway Event
```json
{
  "httpMethod": "GET",
  "path": "/hello",
  "queryStringParameters": {
    "name": "Altair"
  },
  "headers": {
    "Content-Type": "application/json"
  }
}
```

### S3 Event
```json
{
  "Records": [{
    "s3": {
      "bucket": { "name": "my-bucket" },
      "object": { "key": "uploads/photo.jpg" }
    }
  }]
}
```

### SQS Event
```json
{
  "Records": [{
    "body": "{\"userId\": \"123\", \"action\": \"signup\"}",
    "messageId": "abc123"
  }]
}
```

---

## Deploying a Lambda Function

### Via Console
1. Go to **Lambda → Create function**
2. Choose runtime
3. Write or upload your code
4. Set the handler (e.g., `index.handler` for Node.js)
5. Configure memory, timeout, and environment variables
6. Add a trigger
7. Use the **Test** tab with a sample event to verify

### Via AWS CLI
```bash
# Package and create a function
zip function.zip index.js

aws lambda create-function \
  --function-name my-function \
  --runtime nodejs18.x \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::<account-id>:role/lambda-execution-role

# Update function code
aws lambda update-function-code \
  --function-name my-function \
  --zip-file fileb://function.zip

# Invoke a function manually
aws lambda invoke \
  --function-name my-function \
  --payload '{"httpMethod":"GET","path":"/hello"}' \
  response.json

cat response.json
```

---

## Lambda + API Gateway

The most common pattern for building REST APIs without a server:

```
Client → API Gateway → Lambda → DynamoDB / RDS / S3
```

1. Create a Lambda function
2. Go to **API Gateway → Create API → REST API or HTTP API**
3. Create routes (e.g., `GET /hello`) and integrate each with your Lambda
4. Deploy the API — get a public URL

Or use the **Function URL** feature for a simpler HTTPS endpoint directly on Lambda (no API Gateway needed for basic cases).

---

## Lambda Layers

Layers let you share libraries, dependencies, or custom runtimes across multiple functions without bundling them into every deployment package.

```bash
# Create a layer from a zip
aws lambda publish-layer-version \
  --layer-name my-deps \
  --zip-file fileb://layer.zip \
  --compatible-runtimes nodejs18.x
```

Common use: shared `node_modules`, Python packages, binary dependencies (e.g., ffmpeg).

---

## Environment Variables & Secrets

```bash
# Set environment variables
aws lambda update-function-configuration \
  --function-name my-function \
  --environment "Variables={DB_URL=mongodb://...,NODE_ENV=production}"
```

For sensitive values, store in **SSM Parameter Store** or **Secrets Manager** and read at runtime — don't hardcode in environment variables if possible.

---

## IAM Permissions

Lambda needs two types of IAM roles:

| Role | Purpose |
|------|---------|
| **Execution Role** | What the function is allowed to do (e.g., read from S3, write to DynamoDB) |
| **Resource-based Policy** | What can *invoke* the function (e.g., API Gateway, S3) |

Minimum execution role policy:
```json
{
  "Effect": "Allow",
  "Action": [
    "logs:CreateLogGroup",
    "logs:CreateLogStream",
    "logs:PutLogEvents"
  ],
  "Resource": "*"
}
```

---

## Cold Starts

When Lambda hasn't been invoked recently, it needs to spin up a new execution environment — this is a **cold start** and adds latency (typically 100ms–1s depending on runtime and package size).

**Ways to reduce cold starts:**
- Use **Provisioned Concurrency** (keeps instances warm — adds cost)
- Keep deployment packages small
- Use lighter runtimes (Python/Node.js are faster than Java/.NET to initialize)
- Avoid large imports at the top level

---

## Monitoring & Logs

All Lambda logs go to **CloudWatch Logs** automatically. Each function gets a log group: `/aws/lambda/<function-name>`.

```bash
# Tail logs
aws logs tail /aws/lambda/my-function --follow
```

Key metrics in CloudWatch:
- **Invocations**: Total calls
- **Errors**: Failed executions
- **Duration**: Execution time
- **Throttles**: Requests rejected due to concurrency limits
- **ConcurrentExecutions**: Active instances

---

## Lambda Limits

| Limit | Value |
|-------|-------|
| Max execution timeout | 15 minutes |
| Max memory | 10 GB |
| Max deployment package (zipped) | 50 MB (250 MB unzipped) |
| Max ephemeral storage (`/tmp`) | 10 GB |
| Default concurrent executions | 1,000 per region (can be increased) |

---

## When to Use Lambda vs Other Compute

| | Lambda | EC2 | ECS/Fargate | Beanstalk |
|--|--------|-----|-------------|-----------|
| Use case | Event-driven, short tasks | Long-running, custom infra | Containerized apps | Traditional web apps |
| Max runtime | 15 min | Unlimited | Unlimited | Unlimited |
| Idle cost | $0 | Ongoing | Minimal (Fargate) | Ongoing |
| Scaling | Instant, automatic | Manual / ASG | Automatic | Automatic |