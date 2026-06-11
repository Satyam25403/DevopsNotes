# Google Cloud Monitoring (& Cloud Logging)

Cloud Monitoring and Cloud Logging are GCP's unified observability services — providing **metrics, logs, alerts, dashboards, and traces** for your entire GCP infrastructure and applications. Together they're the GCP equivalent of Amazon CloudWatch.

---

## What Cloud Monitoring Provides

| Feature | Description |
|---------|-------------|
| **Metrics** | Time-series data: CPU, memory, request count, error rate, latency |
| **Alerting** | Trigger notifications when metrics cross thresholds |
| **Dashboards** | Real-time visualization of metrics and logs |
| **Uptime Checks** | External probes that verify your endpoints are reachable |
| **SLOs** | Define and monitor Service Level Objectives |

## What Cloud Logging Provides

| Feature | Description |
|---------|-------------|
| **Log ingestion** | Collect logs from GCP services, Compute Engine, GKE, Cloud Run, etc. |
| **Log Explorer** | Search and analyze logs with an interactive query interface |
| **Log-based Metrics** | Extract metrics from log entries to power alerts and dashboards |
| **Log Sinks** | Export logs to Cloud Storage, BigQuery, or Pub/Sub |
| **Log Analytics** | SQL-based analysis of log data at scale |
| **Error Reporting** | Automatically groups and tracks application errors |

---

## Metrics

Cloud Monitoring collects metrics automatically for all GCP services — no agents required.

### Built-in Metrics (automatic)
- **Compute Engine**: `compute.googleapis.com/instance/cpu/utilization`, `disk/read_bytes_count`, `network/sent_bytes_count`
- **Cloud Run**: `run.googleapis.com/request_count`, `request_latencies`, `container/cpu/utilization`
- **Cloud SQL**: `cloudsql.googleapis.com/database/cpu/utilization`, `disk/quota`, `network/connections`
- **GKE**: `container.googleapis.com/container/cpu/request_utilization`, `memory/used_bytes`
- **Cloud Functions**: `cloudfunctions.googleapis.com/function/execution_count`, `execution_times`

### Custom Metrics (send from your app)
```bash
# Via the Monitoring API (Node.js SDK example)
const monitoring = require('@google-cloud/monitoring');
const client = new monitoring.MetricServiceClient();

await client.createTimeSeries({
  name: client.projectPath('my-project'),
  timeSeries: [{
    metric: {
      type: 'custom.googleapis.com/orders/processed',
      labels: { environment: 'production' },
    },
    resource: {
      type: 'global',
      labels: { project_id: 'my-project' },
    },
    points: [{
      interval: { endTime: { seconds: Date.now() / 1000 } },
      value: { int64Value: 42 },
    }],
  }],
});
```

---

## Alerting Policies

```bash
# Create an alert when Cloud Run error rate > 5%
gcloud monitoring policies create --policy-from-file=alert-policy.yaml
```

Example `alert-policy.yaml`:
```yaml
displayName: "Cloud Run High Error Rate"
conditions:
  - displayName: "Error rate > 5%"
    conditionThreshold:
      filter: |
        resource.type = "cloud_run_revision"
        AND metric.type = "run.googleapis.com/request_count"
        AND metric.labels.response_code_class = "5xx"
      comparison: COMPARISON_GT
      thresholdValue: 0.05
      duration: 60s
      aggregations:
        - alignmentPeriod: 60s
          perSeriesAligner: ALIGN_RATE
notificationChannels:
  - projects/my-project/notificationChannels/CHANNEL_ID
alertStrategy:
  autoClose: 1800s    # Auto-close if alert resolves after 30 min
```

### Create Notification Channels
```bash
# Email notification channel
gcloud monitoring channels create \
  --display-name="DevOps Team Email" \
  --type=email \
  --channel-labels=email_address=devops@example.com

# List channels to get the ID
gcloud monitoring channels list
```

---

## Uptime Checks

Verify your services are accessible from multiple global locations:

```bash
# Create an HTTPS uptime check
gcloud monitoring uptime create my-app-check \
  --resource-type=uptime-url \
  --hostname=myapp.com \
  --path=/health \
  --check-interval=60s \
  --timeout=10s \
  --port=443

# List uptime checks
gcloud monitoring uptime list
```

---

## Cloud Logging

### Viewing Logs

```bash
# View recent logs for a Cloud Run service
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"' \
  --limit=50 \
  --format="table(timestamp, severity, textPayload)"

# View logs for a specific Compute Engine VM
gcloud logging read \
  'resource.type="gce_instance" AND resource.labels.instance_id="INSTANCE_ID"' \
  --limit=100

# Stream logs in real-time (tail)
gcloud alpha logging tail \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="my-service"'

# Filter by severity
gcloud logging read \
  'severity>=ERROR AND resource.type="cloud_run_revision"' \
  --limit=50

# Search for a specific string
gcloud logging read \
  'resource.type="cloud_run_revision" AND textPayload:"NullPointerException"' \
  --limit=20
```

### Structured Logging (from your app)

Cloud Logging automatically parses JSON logs from stdout:

```javascript
// Node.js — write structured logs to stdout
// Cloud Run, GKE, and Cloud Functions capture stdout automatically
console.log(JSON.stringify({
  severity: 'ERROR',
  message: 'Database connection failed',
  error: err.message,
  requestId: req.headers['x-request-id'],
  userId: req.user?.id,
}));

// Severity levels: DEFAULT, DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY
```

```python
# Python — structured logging
import json
import sys

def log(severity, message, **kwargs):
    entry = {"severity": severity, "message": message, **kwargs}
    print(json.dumps(entry), flush=True)

log("ERROR", "Payment failed", order_id="123", amount=99.99)
```

---

## Log Sinks (Export Logs)

Route logs to Cloud Storage, BigQuery, or Pub/Sub for archiving or analysis:

```bash
# Export all ERROR logs to Cloud Storage
gcloud logging sinks create error-logs-archive \
  storage.googleapis.com/my-logs-bucket \
  --log-filter='severity>=ERROR'

# Export all logs to BigQuery for SQL analysis
gcloud logging sinks create all-logs-bq \
  bigquery.googleapis.com/projects/my-project/datasets/logs_dataset \
  --log-filter='resource.type="cloud_run_revision"'

# Stream logs to Pub/Sub for real-time processing
gcloud logging sinks create realtime-sink \
  pubsub.googleapis.com/projects/my-project/topics/log-stream \
  --log-filter='severity>=WARNING'

# List sinks
gcloud logging sinks list

# Grant the sink's service account write access to the destination
# (gcloud shows the SA email after sink creation)
```

---

## Log-based Metrics

Extract numerical metrics from log data to drive alerts and dashboards:

```bash
# Count occurrences of "payment failed" in logs
gcloud logging metrics create payment-failures \
  --description="Count of payment failure log entries" \
  --log-filter='resource.type="cloud_run_revision" AND textPayload:"payment failed"'

# Now usable in alerting policies and dashboards like any other metric
```

---

## Error Reporting

Cloud Error Reporting automatically groups unhandled exceptions and tracks their frequency:

```bash
# Enable Error Reporting API
gcloud services enable clouderrorreporting.googleapis.com

# View error groups
gcloud beta error-reporting events list --service=my-service
```

From code (errors written to stderr or structured logs are auto-captured):
```javascript
// Unhandled exceptions in Cloud Run are automatically captured
// For manual reporting:
const { ErrorReporting } = require('@google-cloud/error-reporting');
const errors = new ErrorReporting();

try {
  // risky operation
} catch (err) {
  errors.report(err);
}
```

---

## Dashboards

```bash
# Create a dashboard from a JSON definition
gcloud monitoring dashboards create --config-from-file=dashboard.json

# List dashboards
gcloud monitoring dashboards list
```

Dashboards are best created via the GCP Console UI (Monitoring → Dashboards → Create Dashboard) using the drag-and-drop chart editor, then exported as JSON for version control.

---

## Key Differences: GCP vs AWS Monitoring

| Feature | GCP | AWS Equivalent |
|---------|-----|---------------|
| Metrics | Cloud Monitoring | CloudWatch Metrics |
| Logs | Cloud Logging | CloudWatch Logs |
| Log search | Log Explorer (UI) / `gcloud logging read` | CloudWatch Logs Insights |
| Alerts | Alerting Policies | CloudWatch Alarms |
| Dashboards | Cloud Monitoring Dashboards | CloudWatch Dashboards |
| Uptime checks | Cloud Monitoring Uptime | CloudWatch Synthetics |
| Error grouping | Error Reporting | (no direct equivalent) |
| Log export | Log Sinks | CloudWatch Subscriptions |
| Tracing | Cloud Trace | AWS X-Ray |