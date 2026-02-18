# AWS SDK for JavaScript (v3)

The AWS SDK for JavaScript (v3) is the official toolkit for integrating AWS services into your Node.js (or browser) applications. It's modular, TypeScript-friendly, and supports over 300 AWS services.

> **v3 vs v2**: v3 is modular — install only what you need, resulting in smaller bundle sizes. It also has first-class TypeScript support and uses a middleware-based client architecture.

---

## Getting Started

### 1. Install the SDK

```bash
# Install only the service(s) you need
npm install @aws-sdk/client-s3
npm install @aws-sdk/client-dynamodb
npm install @aws-sdk/client-lambda
npm install @aws-sdk/client-ec2
npm install @aws-sdk/client-ssm         # Parameter Store
npm install @aws-sdk/client-sqs
npm install @aws-sdk/client-sns
```

### 2. Configure Credentials

The SDK automatically reads credentials from (in order of priority):
1. Environment variables
2. `~/.aws/credentials` file (set via `aws configure`)
3. IAM Role attached to the EC2/Lambda (best for production)

**Environment variables (recommended for local dev):**
```bash
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_REGION=us-east-1
```

**Or pass region directly in code:**
```javascript
const client = new S3Client({ region: "us-east-1" });
```

---

## S3 Examples

### Upload a File
```javascript
const { S3Client, PutObjectCommand } = require("@aws-sdk/client-s3");
const fs = require("fs");

const s3 = new S3Client({ region: "us-east-1" });

const uploadFile = async () => {
  const command = new PutObjectCommand({
    Bucket: "your-bucket-name",
    Key: "example.txt",
    Body: fs.createReadStream("example.txt"),
  });
  const response = await s3.send(command);
  console.log("Upload success:", response);
};

uploadFile().catch(console.error);
```

### Download / Get an Object
```javascript
const { GetObjectCommand } = require("@aws-sdk/client-s3");

const getFile = async () => {
  const command = new GetObjectCommand({
    Bucket: "your-bucket-name",
    Key: "example.txt",
  });
  const response = await s3.send(command);
  const body = await response.Body.transformToString();
  console.log("File contents:", body);
};
```

### Generate a Pre-signed URL
```javascript
const { getSignedUrl } = require("@aws-sdk/s3-request-presigner");

const url = await getSignedUrl(s3, new GetObjectCommand({
  Bucket: "your-bucket-name",
  Key: "private-file.pdf",
}), { expiresIn: 3600 }); // valid for 1 hour

console.log("Signed URL:", url);
```

---

## DynamoDB Example

```javascript
const { DynamoDBClient, PutItemCommand, GetItemCommand } = require("@aws-sdk/client-dynamodb");

const dynamo = new DynamoDBClient({ region: "us-east-1" });

// Put item
await dynamo.send(new PutItemCommand({
  TableName: "Users",
  Item: {
    userId: { S: "user123" },
    name: { S: "Alice" },
    age: { N: "30" },
  }
}));

// Get item
const result = await dynamo.send(new GetItemCommand({
  TableName: "Users",
  Key: { userId: { S: "user123" } }
}));
console.log(result.Item);
```

---

## SSM Parameter Store Example

```javascript
const { SSMClient, GetParameterCommand, PutParameterCommand } = require("@aws-sdk/client-ssm");

const ssm = new SSMClient({ region: "us-east-1" });

// Get a parameter
const getParam = async () => {
  const command = new GetParameterCommand({
    Name: "/myapp/dev/db-url",
    WithDecryption: true,
  });
  const response = await ssm.send(command);
  console.log("Value:", response.Parameter.Value);
};
```

---

## SQS Example

```javascript
const { SQSClient, SendMessageCommand, ReceiveMessageCommand } = require("@aws-sdk/client-sqs");

const sqs = new SQSClient({ region: "us-east-1" });

// Send a message
await sqs.send(new SendMessageCommand({
  QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456789012/my-queue",
  MessageBody: JSON.stringify({ event: "user_signup", userId: "123" }),
}));

// Receive messages
const response = await sqs.send(new ReceiveMessageCommand({
  QueueUrl: "https://sqs.us-east-1.amazonaws.com/123456789012/my-queue",
  MaxNumberOfMessages: 10,
  WaitTimeSeconds: 20, // long polling
}));
console.log(response.Messages);
```

---

## Error Handling

```javascript
const { S3ServiceException } = require("@aws-sdk/client-s3");

try {
  await s3.send(command);
} catch (err) {
  if (err instanceof S3ServiceException) {
    console.error("S3 error:", err.name, err.message);
  } else {
    throw err;
  }
}
```

---

## TypeScript Support

The SDK is fully typed out of the box:
```typescript
import { S3Client, PutObjectCommand, PutObjectCommandInput } from "@aws-sdk/client-s3";

const params: PutObjectCommandInput = {
  Bucket: "my-bucket",
  Key: "file.txt",
  Body: "Hello, world!",
};

const client = new S3Client({ region: "us-east-1" });
await client.send(new PutObjectCommand(params));
```

---

## Further Reference

- [AWS SDK v3 GitHub Examples](https://github.com/awsdocs/aws-doc-sdk-examples/tree/main/javascriptv3/example_code)
- [AWS SDK v3 Documentation](https://docs.aws.amazon.com/AWSJavaScriptSDK/v3/latest/)
- For other languages (Python/boto3, Java, Go), check the [AWS SDK documentation](https://aws.amazon.com/developer/tools/).