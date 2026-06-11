# JSON — DevOps Reference Notes

> **JavaScript Object Notation** — a lightweight, text-based data interchange format. While YAML dominates config files, JSON is the lingua franca of APIs, container registries, cloud provider CLIs, CI artifact metadata, and observability tooling.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [Data Types](#2-data-types)
3. [Collections — Objects & Arrays](#3-collections--objects--arrays)
4. [JSON vs YAML — When to Use Which](#4-json-vs-yaml--when-to-use-which)
5. [JSON Schema — Validation](#5-json-schema--validation)
6. [JSON Path — Querying](#6-json-path--querying)
7. [JSON in Docker & Container Registries](#7-json-in-docker--container-registries)
8. [JSON in Kubernetes](#8-json-in-kubernetes)
9. [JSON in Terraform](#9-json-in-terraform)
10. [JSON in AWS CLI & Cloud Providers](#10-json-in-aws-cli--cloud-providers)
11. [JSON in GitHub Actions & CI Pipelines](#11-json-in-github-actions--ci-pipelines)
12. [JSON Logging (Structured Logs)](#12-json-logging-structured-logs)
13. [Common Gotchas & Bugs](#13-common-gotchas--bugs)
14. [Validation & Querying Tools](#14-validation--querying-tools)
15. [Quick Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. Core Concepts

- JSON is a **strict subset of JavaScript** object/array literals
- **All valid JSON is valid YAML** — JSON is a subset of YAML 1.2
- **No comments** — JSON has zero comment syntax (use `//` in JSONC only)
- **Double quotes only** — single quotes are invalid in JSON
- **No trailing commas** — a comma after the last item is a parse error
- File extensions: `.json` (universal), `.jsonc` (JSON with comments — VS Code, some tools)
- Encoding: UTF-8 (default and strongly recommended), UTF-16, UTF-32

### Why JSON matters in DevOps

```
REST APIs          → request/response bodies are JSON
Docker             → image manifests, inspect output, daemon config
Kubernetes         → kubectl -o json, API responses, admission webhooks
Terraform          → variable files (.tfvars.json), state file, provider outputs
AWS / GCP / Azure  → CLI output, IAM policies, resource configs
Observability      → structured log lines, alert payloads, trace metadata
GitHub Actions     → job outputs, fromJSON(), context objects
Package managers   → package.json, package-lock.json, composer.json
```

---

## 2. Data Types

JSON has exactly **six data types** — no more, no less.

```json
{
  "string":   "hello world",
  "number":   42,
  "float":    3.14,
  "boolean":  true,
  "null":     null,
  "object":   { "key": "value" },
  "array":    [1, 2, 3]
}
```

### Strings

```json
{
  "plain":       "hello",
  "with_escape": "line1\nline2",
  "tab":         "col1\tcol2",
  "unicode":     "\u0041",
  "path":        "C:\\Users\\priya\\Desktop",
  "url":         "https://example.com/api/v1",
  "empty":       ""
}
```

**Supported escape sequences:**

| Escape | Character         |
|--------|-------------------|
| `\"`   | double quote      |
| `\\`   | backslash         |
| `\/`   | forward slash     |
| `\n`   | newline           |
| `\r`   | carriage return   |
| `\t`   | tab               |
| `\uXXXX` | Unicode code point |

### Numbers

```json
{
  "integer":    42,
  "negative":   -7,
  "float":      3.14159,
  "scientific": 1.5e10,
  "neg_sci":    2.5E-4
}
```

> **No octal, hex, binary literals** — JSON numbers are always decimal.
> No `Infinity`, `-Infinity`, or `NaN` — these are JavaScript-only.

### Booleans and Null

```json
{
  "enabled":  true,
  "disabled": false,
  "nothing":  null
}
```

> **Only lowercase** — `True`, `False`, `NULL`, `Null`, `None` are all parse errors.

---

## 3. Collections — Objects & Arrays

### Objects (key-value maps)

```json
{
  "server": {
    "host": "localhost",
    "port": 8080,
    "debug": true
  }
}
```

Rules:
- Keys **must** be strings in double quotes
- Keys should be unique (duplicates are technically allowed by spec but behaviour is parser-dependent — last value usually wins)
- Order is not guaranteed (treat objects as unordered)

### Arrays

```json
{
  "fruits":     ["apple", "banana", "cherry"],
  "ports":      [80, 443, 8080],
  "mixed":      [1, "two", true, null],
  "empty":      []
}
```

### Array of objects — the most common DevOps pattern

```json
{
  "containers": [
    { "name": "web",  "image": "nginx:latest",   "port": 80 },
    { "name": "db",   "image": "postgres:15",    "port": 5432 }
  ]
}
```

### Deeply nested structures

```json
{
  "application": {
    "name": "my-app",
    "version": "2.1.0",
    "replicas": 3,
    "resources": {
      "requests": { "cpu": "250m", "memory": "128Mi" },
      "limits":   { "cpu": "500m", "memory": "256Mi" }
    },
    "env": [
      { "name": "DB_HOST",  "value": "postgres-service" },
      { "name": "DB_PORT",  "value": "5432" }
    ],
    "labels": {
      "app": "my-app",
      "tier": "backend",
      "environment": "production"
    }
  }
}
```

---

## 4. JSON vs YAML — When to Use Which

| Concern              | JSON                                | YAML                              |
|----------------------|-------------------------------------|-----------------------------------|
| Human readability    | Verbose, noisy braces               | Clean, minimal syntax             |
| Comments             | ❌ Not supported                    | ✅ `#` comments                   |
| Multi-line strings   | `\n` escapes only                   | `|` and `>` block scalars         |
| Trailing commas      | ❌ Parse error                      | N/A                               |
| API payloads         | ✅ Universal standard               | Rarely used                       |
| Config files         | Acceptable but noisy                | ✅ Preferred (K8s, Ansible, etc.) |
| CLI output parsing   | ✅ `jq` is powerful                 | `yq` (newer ecosystem)            |
| Schema validation    | ✅ JSON Schema is mature            | Partial (via JSON Schema tools)   |
| Streaming/large data | ✅ Simple tokenization              | More complex                      |
| Terraform            | ✅ `.tfvars.json` for automation    | ✅ HCL preferred for humans       |
| Docker daemon config | ✅ `daemon.json` is JSON            | N/A                               |
| IAM policies (AWS)   | ✅ JSON only                        | N/A                               |

**Rule of thumb:** Use JSON when a machine generates or consumes the file. Use YAML when a human writes or reads it.

---

## 5. JSON Schema — Validation

JSON Schema lets you define and validate the structure of JSON documents.

### Basic schema example

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://example.com/schemas/deployment-config.json",
  "title": "Deployment Config",
  "type": "object",
  "required": ["name", "image", "replicas"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 63,
      "pattern": "^[a-z0-9-]+$"
    },
    "image": {
      "type": "string",
      "description": "Docker image reference"
    },
    "replicas": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100,
      "default": 1
    },
    "port": {
      "type": "integer",
      "minimum": 1,
      "maximum": 65535
    },
    "env": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["name", "value"],
        "properties": {
          "name":  { "type": "string" },
          "value": { "type": "string" }
        },
        "additionalProperties": false
      }
    },
    "labels": {
      "type": "object",
      "additionalProperties": { "type": "string" }
    }
  },
  "additionalProperties": false
}
```

### Common schema keywords

```json
{
  "type":                 "string | number | integer | boolean | null | object | array",
  "enum":                 ["dev", "staging", "production"],
  "const":                "production",
  "default":              42,
  "description":          "Human-readable field description",

  "minLength":            1,
  "maxLength":            255,
  "pattern":              "^[a-zA-Z0-9_-]+$",
  "format":               "date-time | uri | email | ipv4 | hostname",

  "minimum":              0,
  "maximum":              65535,
  "exclusiveMinimum":     0,
  "multipleOf":           2,

  "minItems":             1,
  "maxItems":             10,
  "uniqueItems":          true,

  "required":             ["field1", "field2"],
  "additionalProperties": false,
  "minProperties":        1,
  "maxProperties":        20
}
```

### Schema composition

```json
{
  "oneOf": [
    { "$ref": "#/$defs/typeA" },
    { "$ref": "#/$defs/typeB" }
  ],
  "anyOf": [
    { "type": "string" },
    { "type": "number" }
  ],
  "allOf": [
    { "$ref": "#/$defs/base" },
    { "required": ["extra_field"] }
  ],
  "not": { "type": "null" },
  "$defs": {
    "base": { "type": "object" }
  }
}
```

### Validate with CLI tools

```bash
# ajv-cli (Node.js)
npm install -g ajv-cli
ajv validate -s schema.json -d data.json

# check-jsonschema (Python)
pip install check-jsonschema
check-jsonschema --schemafile schema.json data.json

# Using in CI (GitHub Actions)
- name: Validate config
  run: check-jsonschema --schemafile .schemas/config.schema.json config.json
```

---

## 6. JSON Path — Querying

JSONPath is an expression language for navigating JSON, analogous to XPath for XML.

### Syntax reference

| Expression     | Meaning                                  |
|----------------|------------------------------------------|
| `$`            | Root element                             |
| `.key`         | Child key                                |
| `['key']`      | Child key (bracket notation)             |
| `..key`        | Recursive descent — find key anywhere    |
| `[*]`          | All array elements                       |
| `[0]`          | First element (zero-indexed)             |
| `[-1]`         | Last element                             |
| `[0:3]`        | Slice — elements 0, 1, 2                 |
| `[?(@.age>18)]`| Filter — elements where age > 18         |
| `length(@)`    | Length function                          |

### Examples

```bash
# Given deployment.json:
# { "spec": { "replicas": 3, "containers": [{"name":"web","image":"nginx:1.25"}] } }

# Get replicas
jq '.spec.replicas' deployment.json                          # → 3

# Get first container name
jq '.spec.containers[0].name' deployment.json               # → "web"

# Get all container images
jq '.spec.containers[].image' deployment.json               # → "nginx:1.25"

# Filter: containers with port 80
jq '.spec.containers[] | select(.port == 80)' deployment.json

# Recursive: find all "image" values anywhere in the doc
jq '.. | objects | .image? // empty' deployment.json
```

---

## 7. JSON in Docker & Container Registries

### Docker daemon config (`/etc/docker/daemon.json`)

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "registry-mirrors": [
    "https://mirror.example.com"
  ],
  "insecure-registries": [
    "registry.internal:5000"
  ],
  "default-address-pools": [
    { "base": "172.30.0.0/16", "size": 24 }
  ],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "storage-driver": "overlay2",
  "live-restore": true,
  "debug": false
}
```

### `docker inspect` — parsing output with `jq`

```bash
# docker inspect returns a JSON array

# Get container IP address
docker inspect my-container | jq '.[0].NetworkSettings.IPAddress'

# Get all environment variables
docker inspect my-container | jq '.[0].Config.Env[]'

# Get mounted volumes
docker inspect my-container | jq '.[0].Mounts'

# Get image ID from a running container
docker inspect my-container | jq -r '.[0].Image'

# Check health status
docker inspect my-container | jq '.[0].State.Health.Status'

# Get port bindings
docker inspect my-container | jq '.[0].NetworkSettings.Ports'

# Filter: all containers using a specific image
docker inspect $(docker ps -q) | jq '.[] | select(.Config.Image | startswith("nginx")) | .Name'
```

### OCI Image Manifest (registry API)

```json
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    }
  ],
  "annotations": {
    "org.opencontainers.image.created": "2024-01-15T10:30:00Z",
    "org.opencontainers.image.revision": "abc1234"
  }
}
```

### `docker build` metadata with `--metadata-file`

```bash
# Capture build metadata as JSON
docker buildx build \
  --metadata-file build-metadata.json \
  --push \
  -t registry.example.com/myapp:latest .

# Parse the digest for use in downstream steps
IMAGE_DIGEST=$(jq -r '."containerimage.digest"' build-metadata.json)
```

---

## 8. JSON in Kubernetes

### `kubectl` JSON output and querying

```bash
# Output any resource as JSON
kubectl get pod my-pod -o json
kubectl get deployment my-deploy -o json

# Extract specific fields with jsonpath
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod my-pod -o jsonpath='{.status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}'

# Pretty-print with newlines
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'

# Combined jq approach (more powerful)
kubectl get pods -o json | jq '.items[] | {name: .metadata.name, status: .status.phase}'

# Get all container images across all pods
kubectl get pods -A -o json | jq '.items[].spec.containers[].image'

# Find pods not in Running state
kubectl get pods -o json | jq '.items[] | select(.status.phase != "Running") | .metadata.name'

# Get resource limits for all containers
kubectl get pods -o json | jq '
  .items[] | {
    pod: .metadata.name,
    containers: [.spec.containers[] | {
      name: .name,
      cpu_limit: .resources.limits.cpu,
      mem_limit: .resources.limits.memory
    }]
  }
'
```

### JSON patch — `kubectl patch`

```bash
# Strategic merge patch (default)
kubectl patch deployment my-deploy -p '{"spec":{"replicas":5}}'

# JSON Patch (RFC 6902) — precise surgical edits
kubectl patch deployment my-deploy --type=json -p='[
  {"op": "replace", "path": "/spec/replicas", "value": 5},
  {"op": "add",     "path": "/metadata/labels/version", "value": "v2"},
  {"op": "remove",  "path": "/metadata/annotations/deprecated"}
]'

# JSON Merge Patch (RFC 7396)
kubectl patch deployment my-deploy --type=merge -p '
{
  "spec": {
    "template": {
      "spec": {
        "containers": [{"name": "web", "image": "nginx:1.26"}]
      }
    }
  }
}'
```

### JSON Patch operations (RFC 6902)

```json
[
  { "op": "add",     "path": "/a/b",     "value": "new_value"   },
  { "op": "remove",  "path": "/a/b"                              },
  { "op": "replace", "path": "/a/b",     "value": "new_value"   },
  { "op": "move",    "from": "/a/b",     "path": "/a/c"         },
  { "op": "copy",    "from": "/a/b",     "path": "/a/d"         },
  { "op": "test",    "path": "/a/b",     "value": "expected"    }
]
```

### Kubernetes Admission Webhook (JSON payload)

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "request": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "kind": { "group": "apps", "version": "v1", "kind": "Deployment" },
    "resource": { "group": "apps", "version": "v1", "resource": "deployments" },
    "operation": "CREATE",
    "object": {
      "metadata": { "name": "my-deploy", "namespace": "default" },
      "spec": { "replicas": 3 }
    }
  }
}
```

### Admission Webhook response

```json
{
  "apiVersion": "admission.k8s.io/v1",
  "kind": "AdmissionReview",
  "response": {
    "uid": "705ab4f5-6393-11e8-b7cc-42010a800002",
    "allowed": true,
    "patchType": "JSONPatch",
    "patch": "W3sib3AiOiJhZGQiLCJwYXRoIjoiL3NwZWMvcmVwbGljYXMiLCJ2YWx1ZSI6M31d"
  }
}
```

> The `patch` value is a base64-encoded JSON Patch array.

---

## 9. JSON in Terraform

### Variable definition files (`.tfvars.json`)

```json
{
  "region":        "ap-south-1",
  "environment":   "production",
  "instance_type": "t3.medium",
  "replica_count": 3,
  "tags": {
    "Project":     "myapp",
    "ManagedBy":   "terraform",
    "Environment": "production"
  },
  "allowed_cidrs": [
    "10.0.0.0/8",
    "172.16.0.0/12"
  ]
}
```

```bash
# Use with terraform plan/apply
terraform plan -var-file="prod.tfvars.json"
```

### Terraform state — key JSON paths

```bash
# Parse terraform.tfstate with jq

# List all resource addresses
jq '.resources[].instances[].attributes.id' terraform.tfstate

# Find a specific resource
jq '.resources[] | select(.name == "web_server")' terraform.tfstate

# Get all instance IPs
jq '.resources[] | select(.type == "aws_instance") | .instances[].attributes.public_ip' terraform.tfstate

# Count resources by type
jq '[.resources[].type] | group_by(.) | map({type: .[0], count: length})' terraform.tfstate
```

### Terraform output as JSON

```bash
# Get all outputs as JSON
terraform output -json

# Get a specific output
terraform output -json db_endpoint

# Parse in a script
DB_HOST=$(terraform output -json db_endpoint | jq -r '.value')
```

### `terraform show -json` plan file

```bash
terraform plan -out=tfplan
terraform show -json tfplan | jq '.resource_changes[] | {
  address: .address,
  action: .change.actions[]
}'
```

---

## 10. JSON in AWS CLI & Cloud Providers

### AWS CLI output and filtering

```bash
# Default output is JSON
aws ec2 describe-instances

# Filter with --query (JMESPath)
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress]' \
  --output table

# Get running instance IDs
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output json

# Get ECR image tags
aws ecr describe-images \
  --repository-name my-repo \
  --query 'imageDetails[*].imageTags' \
  --output json

# Pipe to jq for more power
aws ecs list-tasks --cluster my-cluster | \
  jq -r '.taskArns[]' | \
  xargs -I{} aws ecs describe-tasks --cluster my-cluster --tasks {}
```

### AWS IAM Policy — JSON document

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3ReadAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "StringEquals": {
          "s3:prefix": ["uploads/", "public/"]
        }
      }
    },
    {
      "Sid": "DenyDeleteObjects",
      "Effect": "Deny",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### AWS S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/my-app-role"
      },
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### AWS CloudWatch Alarm — JSON config

```json
{
  "AlarmName": "HighCPUUtilization",
  "AlarmDescription": "CPU > 80% for 5 minutes",
  "MetricName": "CPUUtilization",
  "Namespace": "AWS/EC2",
  "Statistic": "Average",
  "Period": 300,
  "EvaluationPeriods": 2,
  "Threshold": 80.0,
  "ComparisonOperator": "GreaterThanThreshold",
  "Dimensions": [
    { "Name": "InstanceId", "Value": "i-0123456789abcdef0" }
  ],
  "AlarmActions": [
    "arn:aws:sns:ap-south-1:123456789012:ops-alerts"
  ],
  "TreatMissingData": "notBreaching"
}
```

---

## 11. JSON in GitHub Actions & CI Pipelines

### `fromJSON()` and `toJSON()` in expressions

```yaml
# fromJSON — parse a JSON string into an object
jobs:
  deploy:
    strategy:
      matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}

# toJSON — serialize context to JSON (useful for debugging)
- name: Dump context
  run: echo '${{ toJSON(github) }}'

# Parse job output JSON
- name: Use output
  run: |
    IMAGE="${{ fromJSON(steps.meta.outputs.json).tags[0] }}"
    echo "Deploying $IMAGE"
```

### Dynamic matrix from JSON output

```yaml
jobs:
  setup:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          MATRIX=$(jq -cn '{
            include: [
              {"env": "staging",    "region": "us-east-1"},
              {"env": "production", "region": "ap-south-1"}
            ]
          }')
          echo "matrix=$MATRIX" >> $GITHUB_OUTPUT

  deploy:
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      matrix: ${{ fromJSON(needs.setup.outputs.matrix) }}
    steps:
      - run: echo "Deploying to ${{ matrix.env }} in ${{ matrix.region }}"
```

### Parsing CI artifact metadata

```bash
# Parse build metadata produced by docker/metadata-action
TAGS=$(cat build-metadata.json | jq -r '."containerimage.tags" | split(",")')
DIGEST=$(cat build-metadata.json | jq -r '."containerimage.digest"')

# Parse test result JSON (Jest, pytest-json, etc.)
FAILURES=$(cat test-results.json | jq '.numFailedTests')
if [ "$FAILURES" -gt 0 ]; then
  echo "Tests failed: $FAILURES failures"
  exit 1
fi
```

### Slack notification via webhook (JSON payload)

```bash
curl -X POST "$SLACK_WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d "$(jq -cn \
    --arg env "$ENVIRONMENT" \
    --arg tag "$IMAGE_TAG" \
    --arg status "$DEPLOY_STATUS" \
    '{
      text: ("Deploy to " + $env + " — " + $status),
      attachments: [{
        color: (if $status == "success" then "good" else "danger" end),
        fields: [
          {title: "Image Tag", value: $tag, short: true},
          {title: "Environment", value: $env, short: true}
        ]
      }]
    }'
  )"
```

### package.json — CI-relevant fields

```json
{
  "name": "my-app",
  "version": "2.1.0",
  "scripts": {
    "build":    "webpack --mode production",
    "test":     "jest --coverage",
    "lint":     "eslint src/",
    "docker":   "docker build -t my-app:$npm_package_version ."
  },
  "engines": {
    "node": ">=20.0.0",
    "npm":  ">=10.0.0"
  },
  "dependencies": { },
  "devDependencies": { }
}
```

```bash
# Read version in CI without jq
VERSION=$(node -p "require('./package.json').version")

# Or with jq
VERSION=$(jq -r '.version' package.json)
```

---

## 12. JSON Logging (Structured Logs)

Structured JSON logs are the standard for containerised workloads — they're machine-parseable by log aggregators (Loki, CloudWatch, Datadog, Splunk).

### Standard JSON log line

```json
{
  "timestamp": "2024-01-15T10:30:00.123Z",
  "level":     "error",
  "message":   "Database connection failed",
  "service":   "api-gateway",
  "version":   "2.1.0",
  "trace_id":  "abc123def456",
  "span_id":   "789xyz",
  "pod":       "api-gateway-7d9f8b-xk2p9",
  "namespace": "production",
  "error": {
    "type":    "ConnectionRefusedError",
    "message": "connect ECONNREFUSED 10.0.0.5:5432",
    "stack":   "Error: connect ECONNREFUSED...\n    at ..."
  },
  "context": {
    "user_id":    "u-12345",
    "request_id": "req-67890",
    "endpoint":   "/api/v1/orders",
    "method":     "POST",
    "duration_ms": 1523
  }
}
```

### Querying logs with `jq`

```bash
# Filter logs from a file or stream
cat app.log | jq 'select(.level == "error")'

# Filter by time range
cat app.log | jq 'select(.timestamp > "2024-01-15T10:00:00Z")'

# Extract specific fields
cat app.log | jq '{time: .timestamp, msg: .message, svc: .service}'

# Count errors by service
cat app.log | jq -s '[.[] | select(.level == "error")] | group_by(.service) | map({service: .[0].service, count: length})'

# Get slow requests (>500ms)
cat app.log | jq 'select(.context.duration_ms > 500) | {endpoint: .context.endpoint, ms: .context.duration_ms}'

# Last N errors
cat app.log | jq -s '[.[] | select(.level == "error")] | .[-10:]'
```

### Kubernetes log parsing

```bash
# Stream structured logs from pod and parse
kubectl logs -f my-pod | jq 'select(.level == "error") | .message'

# Get logs with timestamps
kubectl logs my-pod --timestamps | grep ERROR | head -20

# Parse multi-container pod logs
kubectl logs my-pod -c web | jq '{ts: .timestamp, msg: .message}'
```

### Popular JSON logging libraries

| Language   | Library              | Config example                     |
|------------|----------------------|------------------------------------|
| Node.js    | `pino`, `winston`    | `pino()` outputs JSON by default   |
| Python     | `structlog`, `loguru`| `structlog.get_logger()`           |
| Go         | `zap`, `zerolog`     | `zap.NewProductionConfig()`        |
| Java       | `logstash-logback`   | `LogstashEncoder` in logback.xml   |
| Ruby       | `semantic_logger`    | `format: :json`                    |

---

## 13. Common Gotchas & Bugs

### No comments

```jsonc
// This is JSONC — valid only in tools that explicitly support it (VS Code settings, tsconfig.json)
// Standard JSON parsers WILL REJECT this

{
  // "name": "priya",   // ← parse error in standard JSON
  "name": "priya"
}
```

**Workaround:** Use a `"_comment"` key (non-standard but widely tolerated):
```json
{
  "_comment": "This config is for production only",
  "name": "priya"
}
```

### Trailing commas

```json
{
  "name": "priya",
  "port": 8080,        ← PARSE ERROR — trailing comma
}

[
  "apple",
  "banana",            ← PARSE ERROR — trailing comma
]
```

### Numbers lose precision — use strings for large integers

```json
{
  "safe":     9007199254740991,
  "unsafe":   9007199254740993,
  "workaround": "9007199254740993"
}
```

> JavaScript's `Number.MAX_SAFE_INTEGER` is `2^53 - 1`. Snowflake IDs, Unix timestamps in nanoseconds, and AWS account IDs can exceed this — always use strings for IDs.

### All keys must be double-quoted

```json
// WRONG
{ name: "priya" }
{ 'name': "priya" }

// CORRECT
{ "name": "priya" }
```

### Strings must use double quotes

```json
// WRONG
{ "name": 'priya' }

// CORRECT
{ "name": "priya" }
```

### No undefined, no functions

```json
// WRONG — these are JavaScript, not JSON
{ "value": undefined }
{ "fn": function() {} }
{ "date": new Date() }

// CORRECT
{ "value": null }
{ "date": "2024-01-15T10:30:00Z" }
```

### Dates have no native type — use ISO 8601 strings

```json
{
  "created_at": "2024-01-15T10:30:00Z",
  "updated_at": "2024-01-15T10:30:00+05:30",
  "date_only":  "2024-01-15",
  "unix_epoch": 1705312200
}
```

### Duplicate keys — silent data loss

```json
{
  "port": 8080,
  "port": 9090
}
// Most parsers accept this — last value wins silently
// port → 9090, the 8080 entry is gone
// Always lint your JSON to catch this
```

### Deeply nested null check fails

```bash
# jq returns null for missing paths — not an error
echo '{"a": {}}' | jq '.a.b.c'      # → null (not an error)

# Use // to provide a default
echo '{"a": {}}' | jq '.a.b.c // "default"'   # → "default"

# Use has() to check existence
echo '{"a": {}}' | jq '.a | has("b")'          # → false
```

---

## 14. Validation & Querying Tools

### `jq` — the JSON Swiss Army knife

```bash
# Install
brew install jq          # macOS
apt install jq           # Ubuntu/Debian
dnf install jq           # Fedora/RHEL

# Basic usage
cat data.json | jq '.'                         # pretty-print
cat data.json | jq -c '.'                      # compact output
cat data.json | jq -r '.name'                  # raw string (no quotes)
cat data.json | jq -e '.status == "ok"'        # exit 1 if false (useful in CI)

# Build new JSON
jq -n --arg name "priya" --argjson port 8080 \
  '{name: $name, port: $port}'

# Slurp multiple files into array
jq -s '.' file1.json file2.json

# Update value
jq '.spec.replicas = 5' deployment.json

# Add key
jq '.metadata.labels.version = "v2"' deployment.json

# Delete key
jq 'del(.metadata.annotations.deprecated)' deployment.json

# Map over array
jq '.items | map({name: .metadata.name, ns: .metadata.namespace})' k8s-list.json

# Select/filter
jq '.items[] | select(.status.phase == "Running")' pods.json

# Reduce / aggregate
jq '[.items[].spec.replicas] | add' deployments.json   # sum all replicas

# Group and count
jq '[.items[].status.phase] | group_by(.) | map({phase: .[0], count: length})' pods.json

# Conditional
jq '.replicas | if . > 3 then "scaled" else "single" end' config.json
```

### `jq` in CI scripts

```bash
# Safe variable injection (prevents injection attacks)
jq -n \
  --arg image    "$IMAGE_TAG" \
  --arg env      "$ENVIRONMENT" \
  --argjson port "$PORT" \
  '{image: $image, env: $env, port: $port}' > deploy-config.json

# Check a value and exit non-zero on failure (-e flag)
jq -e '.status == "healthy"' health.json || { echo "Unhealthy!"; exit 1; }
```

### Validation tools

```bash
# python — zero dependency JSON lint
python3 -m json.tool input.json                # pretty-print + validate
python3 -m json.tool input.json > /dev/null    # validate only (check exit code)

# jsonlint (npm)
npm install -g jsonlint
jsonlint config.json

# fx — interactive JSON viewer (TUI)
npm install -g fx
fx data.json

# gron — make JSON greppable
npm install -g gron
gron data.json | grep "\.name"
gron data.json | grep "\.name" | gron --ungron   # back to JSON

# jless — pager for JSON (like less but for JSON)
brew install jless
kubectl get pods -o json | jless
```

### Conversion tools

```bash
# JSON ↔ YAML
yq -o=json config.yaml         # YAML → JSON
yq -P policy.json              # JSON → YAML (pretty)

# JSON → CSV
jq -r '.[] | [.name, .port, .env] | @csv' data.json

# JSON → TSV
jq -r '.[] | [.name, .port] | @tsv' data.json

# CSV → JSON (using miller)
mlr --c2j cat data.csv

# Flatten deeply nested JSON
jq '[paths(scalars) as $p | {key: ($p | join(".")), value: getpath($p)}] | from_entries' data.json
```

### Online tools

| Tool                    | Purpose                                 |
|-------------------------|-----------------------------------------|
| `jsonlint.com`          | JSON linting and formatting             |
| `jqplay.org`            | Interactive jq playground               |
| `jsonschema.dev`        | JSON Schema editor and validator        |
| `json-schema.org`       | Official JSON Schema specification      |
| `json.parser.online`    | JSON viewer and tree explorer           |

---

## 15. Quick Reference Cheat Sheet

```json
{
  "_comment_types": "The 6 JSON types",
  "string":  "double quotes only",
  "number":  42,
  "float":   3.14,
  "boolean": true,
  "null":    null,
  "object":  { "key": "value" },
  "array":   [1, 2, 3],

  "_comment_strings": "Escape sequences in strings",
  "newline":    "line1\nline2",
  "tab":        "col1\tcol2",
  "quote":      "say \"hello\"",
  "backslash":  "C:\\path\\file",
  "unicode":    "\u2665",

  "_comment_gotchas": "Common mistakes",
  "no_trailing_comma_here": "last item",
  "no_single_quotes":       "always double",
  "no_comments_in_json":    "use _comment keys as workaround",
  "no_undefined":           null,
  "large_id_as_string":     "9007199254740993"
}
```

```bash
# ─── jq QUICK REFERENCE ────────────────────────────────────────

# Read
jq '.key'                         # string/number value
jq '.nested.key'                  # nested value
jq '.[0]'                         # first array element
jq '.array[]'                     # all array elements
jq '.array[1:3]'                  # slice

# Filter
jq 'select(.status == "ok")'      # filter items
jq 'select(.port > 1024)'         # numeric comparison
jq 'select(.name | startswith("web"))' # string test

# Transform
jq '{name: .name, port: .port}'   # project fields
jq '.items | map({n: .name})'     # map array
jq 'del(.sensitive_key)'          # delete key
jq '.count += 1'                  # arithmetic
jq '.tags + ["new-tag"]'          # append to array

# Aggregate
jq '[.[].port] | length'          # count
jq '[.[].port] | add'             # sum
jq '[.[].port] | min'             # min
jq '[.[].port] | max'             # max
jq '[.[].port] | unique'          # deduplicate

# Output
jq -r '.name'                     # raw (no quotes)
jq -c '.'                         # compact
jq -e '.ok == true'               # exit 1 if false
jq -n '{}'                        # null input (build JSON)
```

### Key rules at a glance

| Rule                      | Detail                                               |
|---------------------------|------------------------------------------------------|
| Keys                      | Double-quoted strings only                           |
| String values             | Double quotes only — single quotes invalid           |
| Trailing commas           | Not allowed — parse error                            |
| Comments                  | Not allowed — use `"_comment"` key or JSONC          |
| Booleans                  | `true` / `false` lowercase only                      |
| Null                      | `null` lowercase only                                |
| Large integers            | Use strings — safe integer limit is 2^53 - 1         |
| Dates                     | No native type — use ISO 8601 strings                |
| Duplicate keys            | Technically invalid — last value silently wins       |
| Numbers                   | Decimal only — no hex, octal, binary literals        |
| `Infinity` / `NaN`        | Not supported — use `null` or omit                   |
| Encoding                  | UTF-8 preferred                                      |
| Comments in config files  | Use JSONC (`.jsonc`) or switch to YAML/TOML          |

---

*Part of DevOpsNotes / LANGUAGES — see also `02_YAML.md`, `03_Groovy_Jenkins.md`*