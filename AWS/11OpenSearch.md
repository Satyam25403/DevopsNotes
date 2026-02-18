# Amazon OpenSearch Service

Amazon OpenSearch Service is a fully managed search and analytics engine built on the open-source **OpenSearch** project (a community-driven fork of Elasticsearch). It's designed for log analytics, real-time monitoring, full-text search, and clickstream analysis.

---

## What It's Used For

- **Centralized logging**: Aggregate logs from EC2, Lambda, ECS, and other services
- **Full-text search**: Power search functionality in your applications
- **Security analytics**: Audit trails, threat detection, SIEM workloads
- **Real-time monitoring**: Application performance and infrastructure metrics
- **Business analytics**: SQL-like queries and visualizations via OpenSearch Dashboards

---

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Domain** | A managed OpenSearch cluster (your deployment unit in AWS) |
| **Index** | Logical partition of data — like a database table |
| **Document** | A JSON object stored in an index — the basic data unit |
| **Shard** | A sub-unit of an index for distribution across nodes |
| **Mapping** | Schema definition for an index (field types, analyzers) |
| **Query DSL** | OpenSearch's JSON-based query language |
| **OpenSearch Dashboards** | Built-in UI for querying, visualizing, and managing data (similar to Kibana) |

---

## Deployment Options

| Option | Description |
|--------|-------------|
| **Provisioned** | You choose instance types, storage, and node count. More control, you pay for reserved capacity. |
| **Serverless** | AWS handles scaling — great for unpredictable or spiky workloads. No cluster management needed. |

---

## Creating a Domain

1. Go to **Amazon OpenSearch Service → Create domain**
2. Choose deployment type (Provisioned or Serverless)
3. Select instance type (e.g., `t3.small.search` for dev, `r6g.large.search` for production)
4. Configure storage (EBS volumes)
5. Set access control (IAM or fine-grained access control)
6. Enable encryption at rest and node-to-node encryption
7. Deploy

---

## Ingesting Data

### From CloudWatch Logs (most common for logging)
Set up a **CloudWatch Logs subscription filter** to stream logs to OpenSearch automatically:
1. Go to CloudWatch Logs → your log group → **Subscription filters**
2. Choose **Amazon OpenSearch Service** as the destination
3. Select your domain

### Via the REST API
```bash
# Index a document
curl -X POST "https://<domain-endpoint>/my-index/_doc" \
  -H "Content-Type: application/json" \
  -d '{"timestamp": "2024-01-01T10:00:00", "level": "ERROR", "message": "DB connection failed", "service": "api"}'

# Bulk index multiple documents
curl -X POST "https://<domain-endpoint>/_bulk" \
  -H "Content-Type: application/json" \
  -d '
{"index": {"_index": "logs"}}
{"level": "INFO", "message": "Request received"}
{"index": {"_index": "logs"}}
{"level": "ERROR", "message": "Timeout"}
'
```

### Common Ingestion Pipelines
- **Fluent Bit / Fluentd**: Lightweight log forwarders for containers (ECS, EKS)
- **Logstash**: Feature-rich pipeline for complex log transformations
- **AWS Kinesis Data Firehose**: Serverless data delivery to OpenSearch
- **AWS Lambda**: Custom log processing and forwarding

---

## Querying Data

### Basic Search via REST API
```bash
# Match all documents
curl -X GET "https://<domain-endpoint>/my-index/_search" \
  -H "Content-Type: application/json" \
  -d '{"query": {"match_all": {}}}'

# Full-text search
curl -X GET "https://<domain-endpoint>/logs/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "match": { "message": "connection failed" }
    }
  }'

# Filter by field value
curl -X GET "https://<domain-endpoint>/logs/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "term": { "level": "ERROR" }
    },
    "sort": [{ "timestamp": "desc" }],
    "size": 50
  }'

# Range query (last 24 hours)
curl -X GET "https://<domain-endpoint>/logs/_search" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {
      "range": {
        "timestamp": {
          "gte": "now-24h",
          "lt": "now"
        }
      }
    }
  }'
```

### Aggregations (analytics)
```json
{
  "aggs": {
    "errors_by_service": {
      "terms": { "field": "service.keyword" }
    },
    "avg_response_time": {
      "avg": { "field": "duration_ms" }
    }
  }
}
```

---

## OpenSearch Dashboards

Built-in visualization and exploration UI (accessible via the domain endpoint `/dashboards`).

**Common uses:**
- **Discover**: Explore raw documents and logs with filters
- **Visualize**: Create charts, pie graphs, histograms, maps
- **Dashboard**: Combine visualizations into operational dashboards
- **Dev Tools**: Run Query DSL queries interactively

---

## Security

- **IAM-based access control**: Control access at the domain level via resource-based policies
- **Fine-grained access control (FGAC)**: Role-based access at the index/document/field level
- **Encryption at rest** (KMS) and **node-to-node encryption** (TLS)
- **VPC support**: Place the domain inside a VPC for private access

---

## OpenSearch vs Other AWS Services

| Need | Service |
|------|---------|
| Log aggregation + search | OpenSearch |
| Application metrics & alarms | CloudWatch |
| Long-term log archiving | S3 + Athena |
| Full-text search for app data | OpenSearch |
| Analytics on structured data | Redshift |

---

## Cost Optimization Tips

- Use **UltraWarm** storage for older, infrequently accessed data (up to 90% cheaper than hot storage).
- Use **cold storage** for even older data you rarely need to query.
- Set **Index State Management (ISM) policies** to automatically move or delete old indices:
  ```json
  {
    "policy": {
      "states": [{
        "name": "delete_after_30_days",
        "transitions": [{
          "state_name": "deleted",
          "conditions": { "min_index_age": "30d" }
        }]
      }]
    }
  }
  ```
- Choose the right instance type — don't over-provision for dev/test environments.