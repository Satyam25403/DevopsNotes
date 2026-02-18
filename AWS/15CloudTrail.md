# AWS CloudTrail

AWS CloudTrail provides a complete **audit log of all API activity** across your AWS account. Every action taken — via the Console, CLI, SDK, or AWS service — is recorded. It's your primary tool for security auditing, compliance, and incident investigation.

> Think of CloudTrail as the CCTV of your AWS account — it records who did what, when, and from where.

---

## What CloudTrail Tracks

For every recorded event, CloudTrail captures:

| Field | Example |
|-------|---------|
| **Who** | IAM user, role, or AWS service that made the request |
| **What** | API action (e.g., `RunInstances`, `PutObject`, `DeleteUser`) |
| **When** | Timestamp (UTC) |
| **Where** | Source IP address, AWS region |
| **Result** | Success or failure (`errorCode`, `errorMessage`) |
| **Resource** | Which specific resource was affected |

**Sources tracked:**
- AWS Management Console
- AWS CLI
- AWS SDKs and APIs
- AWS services acting on your behalf (e.g., Auto Scaling launching instances)

---

## Key Components

### Event History (Free)
- Available by default — no setup needed
- Viewable in the CloudTrail console
- Covers the **last 90 days** of management events
- Searchable by user, resource, event type, or time range

### Trails (Long-term storage)
A Trail delivers logs continuously to an **S3 bucket** (and optionally to CloudWatch Logs or EventBridge).

```bash
# Create a trail delivering logs to S3
aws cloudtrail create-trail \
  --name my-audit-trail \
  --s3-bucket-name my-cloudtrail-logs \
  --is-multi-region-trail \
  --enable-log-file-validation

# Start logging
aws cloudtrail start-logging --name my-audit-trail

# Verify trail status
aws cloudtrail get-trail-status --name my-audit-trail
```

### CloudTrail Lake
A managed event data lake for running **SQL queries** directly on CloudTrail events — no ETL or S3 setup needed. Useful for large-scale audit analytics.

---

## Event Types

| Type | Description | Default |
|------|-------------|---------|
| **Management Events** | Control plane operations (create, delete, modify resources) | Enabled |
| **Data Events** | Data plane operations (S3 `GetObject`/`PutObject`, Lambda invocations) | Disabled (high volume) |
| **Insight Events** | Detects unusual API activity patterns (e.g., sudden spike in `TerminateInstances`) | Optional |

---

## Querying Events via CLI

```bash
# Look up events in the last 24 hours
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances \
  --start-time $(date -u -d '24 hours ago' +%Y-%m-%dT%H:%M:%SZ)

# Find all actions by a specific user
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=Username,AttributeValue=alice

# Find events affecting a specific resource
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-s3-bucket

# Output as table for readability
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteBucket \
  --query "Events[*].[EventTime,Username,EventName,Resources[0].ResourceName]" \
  --output table
```

---

## CloudTrail Log Format

Logs are stored in S3 as gzipped JSON files. Example event:
```json
{
  "eventVersion": "1.08",
  "userIdentity": {
    "type": "IAMUser",
    "userName": "alice",
    "arn": "arn:aws:iam::123456789012:user/alice"
  },
  "eventTime": "2024-01-15T10:23:45Z",
  "eventSource": "s3.amazonaws.com",
  "eventName": "DeleteBucket",
  "sourceIPAddress": "203.0.113.42",
  "requestParameters": {
    "bucketName": "my-important-bucket"
  },
  "responseElements": null,
  "errorCode": null
}
```

---

## Integrations

### CloudWatch Logs (real-time alerting)
Stream CloudTrail events to CloudWatch Logs to create **metric filters and alarms** on specific activity:

```bash
# Example: alarm when root account is used
# 1. Create metric filter on the log group
aws logs put-metric-filter \
  --log-group-name CloudTrail/MyTrailLogs \
  --filter-name RootAccountUsage \
  --filter-pattern '{ $.userIdentity.type = "Root" }' \
  --metric-transformations metricName=RootAccountUsageCount,metricNamespace=CloudTrailMetrics,metricValue=1

# 2. Create alarm on that metric
aws cloudwatch put-metric-alarm \
  --alarm-name RootAccountUsed \
  --metric-name RootAccountUsageCount \
  --namespace CloudTrailMetrics \
  --statistic Sum \
  --period 300 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:SecurityAlerts
```

### EventBridge (automated response)
Trigger Lambda or other services automatically in response to specific API calls:
```json
{
  "source": ["aws.iam"],
  "detail-type": ["AWS API Call via CloudTrail"],
  "detail": {
    "eventName": ["CreateUser", "AttachUserPolicy", "DeleteTrail"]
  }
}
```

### Amazon Athena (historical analysis)
Query CloudTrail logs stored in S3 using SQL:
```sql
-- Find all failed API calls in the last 7 days
SELECT eventtime, useridentity.username, eventname, errormessage
FROM cloudtrail_logs
WHERE errorcode IS NOT NULL
  AND eventtime > date_add('day', -7, now())
ORDER BY eventtime DESC
LIMIT 100;
```

---

## Common Use Cases

**Security Incident Investigation:**
"Who deleted that S3 bucket at 2 AM?" → Search CloudTrail by event name and time.

**Compliance Reporting:**
Demonstrate to auditors that IAM policies were never modified without approval.

**Detecting Credential Compromise:**
Alert when an IAM user authenticates from an unusual IP or geographic location.

**Unused Resource Cleanup:**
Find resources that haven't been accessed in 90 days using data events.

---

## Security Best Practices

- **Enable CloudTrail in all regions** with a multi-region trail — attackers may switch regions.
- **Enable log file validation** to detect tampering (CloudTrail generates hash digests to verify log integrity).
- **Protect the S3 bucket**: Enable S3 Object Lock, restrict access, and enable versioning.
- **Never disable or delete trails** — alert on `DeleteTrail` and `StopLogging` events.
- **Enable CloudTrail Insights** to automatically detect anomalous API activity.
- **Centralize logs** across all AWS accounts using AWS Organizations trails.

```bash
# Verify log file integrity
aws cloudtrail validate-logs \
  --trail-arn arn:aws:cloudtrail:us-east-1:123456789012:trail/my-trail \
  --start-time 2024-01-01T00:00:00Z
```

---

## CloudTrail vs CloudWatch

| | CloudTrail | CloudWatch |
|--|------------|------------|
| **What it tracks** | API calls and user activity | Metrics, logs, and performance data |
| **Focus** | Security, auditing, compliance | Monitoring, alerting, operational |
| **Who used it** | Security/compliance teams | DevOps/SRE teams |
| **Retention** | 90 days free; indefinite in S3 | Configurable (1 day – forever) |