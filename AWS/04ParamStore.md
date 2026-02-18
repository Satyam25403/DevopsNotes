# AWS Systems Manager (SSM) Parameter Store

AWS Systems Manager Parameter Store is a secure, scalable, and centralized service for managing configuration data and secrets. Think of it as a cloud-native `.env` file — but with encryption, access control, versioning, and integration with other AWS services built in.

It's part of **AWS Systems Manager (SSM)**, which is the parent service for a suite of operational tools.

---

## What You Can Store

- **Plain text values**: `NODE_ENV=production`, feature flags, config values
- **Encrypted secrets**: Passwords, API keys, tokens (encrypted via AWS KMS)
- **Hierarchical paths**: Organize by app and environment, e.g. `/myapp/prod/db-url`

---

## Parameter Types

| Type | Description |
|------|-------------|
| `String` | Plain text value |
| `StringList` | Comma-separated list of strings |
| `SecureString` | Encrypted with AWS KMS |

---

## CLI Usage

### Store a Parameter
```bash
# Plain text
aws ssm put-parameter \
  --name "/myapp/dev/db-url" \
  --value "mongodb://localhost:27017" \
  --type "String"

# Encrypted (SecureString)
aws ssm put-parameter \
  --name "/myapp/dev/db-password" \
  --value "supersecret" \
  --type "SecureString" \
  --key-id "alias/aws/ssm"

# Overwrite an existing parameter
aws ssm put-parameter \
  --name "/myapp/dev/db-url" \
  --value "mongodb://newhost:27017" \
  --type "String" \
  --overwrite
```

### Retrieve a Parameter
```bash
# Get a single parameter
aws ssm get-parameter \
  --name "/myapp/dev/db-url"

# Get with decryption (required for SecureString)
aws ssm get-parameter \
  --name "/myapp/dev/db-password" \
  --with-decryption

# Get multiple parameters by path (all params under /myapp/dev/)
aws ssm get-parameters-by-path \
  --path "/myapp/dev/" \
  --with-decryption \
  --recursive
```

### Delete a Parameter
```bash
aws ssm delete-parameter --name "/myapp/dev/db-url"
```

---

## Using Parameter Store in Node.js (AWS SDK v3)

```javascript
const { SSMClient, GetParameterCommand, GetParametersByPathCommand } = require("@aws-sdk/client-ssm");

const ssm = new SSMClient({ region: "us-east-1" });

// Get a single parameter
const getParam = async (name) => {
  const command = new GetParameterCommand({
    Name: name,
    WithDecryption: true,
  });
  const response = await ssm.send(command);
  return response.Parameter.Value;
};

// Get all parameters under a path (useful for loading all env vars at startup)
const getParamsByPath = async (path) => {
  const command = new GetParametersByPathCommand({
    Path: path,
    WithDecryption: true,
    Recursive: true,
  });
  const response = await ssm.send(command);
  return response.Parameters.reduce((acc, param) => {
    const key = param.Name.split("/").pop(); // get last segment as key
    acc[key] = param.Value;
    return acc;
  }, {});
};

// Usage
const config = await getParamsByPath("/myapp/prod/");
// config = { db-url: "...", db-password: "...", api-key: "..." }
```

---

## Versioning

Parameter Store automatically keeps a version history. You can retrieve a specific version:
```bash
aws ssm get-parameter --name "/myapp/dev/db-url:3"  # version 3
```

---

## Why Use Parameter Store?

- **Security**: Encrypt secrets with KMS; control access with IAM policies
- **Centralized**: One place for all config — no scattered `.env` files across servers
- **No hardcoded secrets**: Apps pull config at runtime — rotating secrets doesn't require redeployment
- **Integrated**: Works natively with EC2, Lambda, ECS, CodeBuild, Fargate, and more
- **Audit trail**: CloudTrail logs every access and change

---

## Parameter Store vs Secrets Manager

| Feature | Parameter Store | Secrets Manager |
|---------|----------------|-----------------|
| Cost | Free (Standard tier) | ~$0.40/secret/month |
| Automatic rotation | No (manual) | Yes (built-in for RDS, etc.) |
| Cross-account | Limited | Yes |
| Best for | Config + simple secrets | Credentials needing auto-rotation |

> **Rule of thumb**: Use Parameter Store for config and non-rotating secrets. Use Secrets Manager for database passwords and API keys that need automatic rotation.

---

## IAM Permissions Required

To read parameters, the calling IAM user/role needs:
```json
{
  "Effect": "Allow",
  "Action": [
    "ssm:GetParameter",
    "ssm:GetParameters",
    "ssm:GetParametersByPath"
  ],
  "Resource": "arn:aws:ssm:us-east-1:123456789012:parameter/myapp/*"
}
```

For `SecureString`, also add:
```json
{ "Effect": "Allow", "Action": "kms:Decrypt", "Resource": "<kms-key-arn>" }
```