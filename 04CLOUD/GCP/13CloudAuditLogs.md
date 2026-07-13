# Google Cloud Audit Logs

Google Cloud Audit Logs provides a complete **audit trail of all API activity** across your GCP project, folder, or organization. Every action taken — via the Console, CLI, SDK, or any GCP service — is recorded. It's your primary tool for security auditing, compliance, and incident investigation. The GCP equivalent of AWS CloudTrail.

> Think of Audit Logs as the CCTV of your GCP account — it records who did what, when, and from where.

---

## What Audit Logs Track

For every recorded event, Audit Logs captures:

| Field | Example |
|-------|---------| 
| **Who** | IAM principal (user, service account) that made the request |
| **What** | API method called (e.g., `storage.objects.delete`, `compute.instances.insert`) |
| **When** | Timestamp (UTC) |
| **Where** | Source IP, GCP region |
| **Result** | Success or failure (with error code and message) |
| **Resource** | Which specific resource was affected |
| **Request/Response** | Optionally logged request and response payloads |

---

## Audit Log Types

| Log Type | What It Contains | Default State |
|----------|-----------------|---------------|
| **Admin Activity** | Writes to resource metadata (create, delete, update VM, change IAM policy) | Always enabled, free |
| **Data Access** | Reads/writes to resource data (read a Cloud Storage object, query BigQuery) | Disabled by default (high volume) |
| **System Event** | GCP system actions (auto-scaling, live migration) | Always enabled, free |
| **Policy Denied** | Requests blocked by IAM or VPC Service Controls | Always enabled, free |

> **Admin Activity logs are always on and free** — you can't disable them. Data Access logs must be explicitly enabled and generate higher log volumes (and cost).

---

## Viewing Audit Logs

### In the Console
GCP Console → Logging → Log Explorer → filter by `logName:"cloudaudit.googleapis.com"`

### Via CLI
```bash
# View recent Admin Activity logs
gcloud logging read \
  'logName="projects/my-project/logs/cloudaudit.googleapis.com%2Factivity"' \
  --limit=50 \
  --format="table(timestamp, protoPayload.authenticationInfo.principalEmail, protoPayload.methodName, protoPayload.resourceName)"

# View Data Access logs
gcloud logging read \
  'logName="projects/my-project/logs/cloudaudit.googleapis.com%2Fdata_access"' \
  --limit=50

# Filter: Who deleted something?
gcloud logging read \
  'logName:"cloudaudit.googleapis.com/activity"
   AND protoPayload.methodName=~"delete"' \
  --limit=20

# Filter: What did a specific user do?
gcloud logging read \
  'logName:"cloudaudit.googleapis.com/activity"
   AND protoPayload.authenticationInfo.principalEmail="alice@example.com"' \
  --limit=50

# Filter: All IAM policy changes
gcloud logging read \
  'logName:"cloudaudit.googleapis.com/activity"
   AND protoPayload.methodName="SetIamPolicy"' \
  --limit=20

# Filter: Failed/denied requests
gcloud logging read \
  'logName:"cloudaudit.googleapis.com/activity"
   AND protoPayload.status.code!=0' \
  --limit=20
```

---

## Enabling Data Access Logs

Data Access logs are off by default. Enable them per service:

```bash
# Get current audit config
gcloud projects get-iam-policy my-project \
  --format=json | jq '.auditConfigs'

# Enable Data Access logs for Cloud Storage (all types)
gcloud projects set-iam-policy my-project policy.json
```

Example `policy.json` (add to existing IAM policy):
```json
{
  "auditConfigs": [
    {
      "service": "storage.googleapis.com",
      "auditLogConfigs": [
        { "logType": "DATA_READ" },
        { "logType": "DATA_WRITE" }
      ]
    },
    {
      "service": "cloudsql.googleapis.com",
      "auditLogConfigs": [
        { "logType": "DATA_READ" },
        { "logType": "DATA_WRITE" }
      ]
    },
    {
      "service": "allServices",
      "auditLogConfigs": [
        { "logType": "ADMIN_READ" }
      ]
    }
  ]
}
```

---

## Exporting Audit Logs (Log Sinks)

Audit Logs are retained for 400 days (Admin Activity) and 30 days (Data Access) by default. Export for longer retention or analysis:

```bash
# Export all audit logs to Cloud Storage (for long-term archiving)
gcloud logging sinks create audit-archive \
  storage.googleapis.com/my-audit-logs-bucket \
  --log-filter='logName:"cloudaudit.googleapis.com"'

# Export to BigQuery for SQL-based security analysis
gcloud logging sinks create audit-bq \
  bigquery.googleapis.com/projects/my-project/datasets/audit_logs \
  --log-filter='logName:"cloudaudit.googleapis.com"'

# Grant the sink service account write access to the destination
# (shown in the sink creation output)
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:SINK_SA@gcp-sa-logging.iam.gserviceaccount.com" \
  --role="roles/storage.objectCreator"
```

---

## BigQuery Analysis of Audit Logs

Once exported to BigQuery, run security analytics with SQL:

```sql
-- Top 10 most active users in the last 7 days
SELECT
  protopayload_auditlog.authenticationinfo.principalemail AS user,
  COUNT(*) AS action_count
FROM `my-project.audit_logs.cloudaudit_googleapis_com_activity_*`
WHERE _TABLE_SUFFIX >= FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY))
GROUP BY user
ORDER BY action_count DESC
LIMIT 10;

-- All resource deletions in the last 24 hours
SELECT
  timestamp,
  protopayload_auditlog.authenticationinfo.principalemail AS deleted_by,
  protopayload_auditlog.methodname AS action,
  protopayload_auditlog.resourcename AS resource
FROM `my-project.audit_logs.cloudaudit_googleapis_com_activity_*`
WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE())
  AND LOWER(protopayload_auditlog.methodname) LIKE '%delete%'
ORDER BY timestamp DESC;

-- IAM policy changes
SELECT
  timestamp,
  protopayload_auditlog.authenticationinfo.principalemail AS changed_by,
  protopayload_auditlog.resourcename AS resource
FROM `my-project.audit_logs.cloudaudit_googleapis_com_activity_*`
WHERE _TABLE_SUFFIX >= FORMAT_DATE('%Y%m%d', DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY))
  AND protopayload_auditlog.methodname = 'SetIamPolicy'
ORDER BY timestamp DESC;
```

---

## Alert on Suspicious Activity

Set up alerts using log-based metrics triggered from audit logs:

```bash
# Create a metric that counts IAM policy changes
gcloud logging metrics create iam-policy-changes \
  --description="Count of IAM policy change events" \
  --log-filter='logName:"cloudaudit.googleapis.com/activity"
               AND protoPayload.methodName="SetIamPolicy"'

# Create an alert policy for this metric (via Console or YAML)
gcloud monitoring policies create --policy-from-file=iam-alert.yaml
```

Example `iam-alert.yaml`:
```yaml
displayName: "IAM Policy Change Detected"
conditions:
  - displayName: "IAM policy change count > 0"
    conditionThreshold:
      filter: 'metric.type="logging.googleapis.com/user/iam-policy-changes"'
      comparison: COMPARISON_GT
      thresholdValue: 0
      duration: 0s
notificationChannels:
  - projects/my-project/notificationChannels/CHANNEL_ID
```

---

## Security Command Center Integration

Audit Logs feed directly into **Security Command Center (SCC)** — GCP's centralized security findings platform:

```bash
# Enable Security Command Center
gcloud services enable securitycenter.googleapis.com

# List security findings
gcloud scc findings list organizations/ORG_ID \
  --filter="state=ACTIVE AND category=AUDIT_LOGGING_DISABLED"
```

---

## Compliance Use Cases

| Compliance Need | Audit Log Solution |
|-----------------|-------------------|
| SOC 2 / ISO 27001 — access logging | Admin Activity + Data Access logs enabled |
| PCI-DSS — cardholder data access | Data Access logs for Cloud SQL / Spanner |
| HIPAA — PHI access audit | Data Access logs for all relevant services |
| Long-term retention (7 years) | Log sink to Cloud Storage with locked retention policy |
| Immutable audit trail | Log sink to a locked bucket (bucket lock feature) |

```bash
# Lock a bucket to prevent log deletion (compliance)
gcloud storage buckets update gs://my-audit-logs-bucket \
  --retention-period=2557d    # 7 years
  --lock-retention-policy     # Irreversible — cannot be unlocked
```

---

## Best Practices

- **Enable Data Access logs for critical services**: Cloud SQL, Cloud Storage, Secret Manager, IAM
- **Export to Cloud Storage** with a retention lock for compliance and long-term archiving
- **Export to BigQuery** for incident investigation and security analytics
- **Alert on IAM changes**: Any `SetIamPolicy` call should notify your security team
- **Alert on failed requests**: A spike in `PERMISSION_DENIED` may indicate an attack or misconfiguration
- **Restrict who can disable audit logs**: Only organization admins should have `logging.privateLogEntries.list`
- **Audit log access**: Monitor who is reading audit logs — reading audit logs is itself auditable