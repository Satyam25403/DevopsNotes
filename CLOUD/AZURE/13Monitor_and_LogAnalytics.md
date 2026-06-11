# Azure Monitor & Log Analytics
## (analogous to AWS CloudWatch)

Azure Monitor is the unified observability platform in Azure — it collects metrics, logs, traces, and alerts across all Azure resources, applications, and infrastructure. **Log Analytics** is the query engine that powers it, using **KQL (Kusto Query Language)**.

---

## Architecture Overview

```
Azure Resources → Diagnostic Settings → Log Analytics Workspace
                                      → Storage Account (archive)
                                      → Event Hubs (stream to SIEM)

Applications    → Application Insights → Log Analytics Workspace

Azure Monitor ──→ Metrics Explorer (real-time graphs)
             ──→ Alerts (metric / log / activity log based)
             ──→ Dashboards & Workbooks (visualization)
             ──→ Action Groups (email, SMS, webhook, Function)
```

---

## Core Components

| Component | Description | AWS Equivalent |
|-----------|-------------|----------------|
| **Metrics** | Numeric time-series data (CPU, requests/sec) | CloudWatch Metrics |
| **Logs** | Structured log records (queryable via KQL) | CloudWatch Logs Insights |
| **Log Analytics Workspace** | Central store for all log data | CloudWatch Log Groups |
| **Application Insights** | APM — traces, exceptions, dependencies, perf | X-Ray + CloudWatch Application Signals |
| **Alerts** | Notify or act when a condition is met | CloudWatch Alarms |
| **Action Groups** | Who/how to notify when alert fires | SNS Topics |
| **Workbooks** | Interactive dashboards using KQL | CloudWatch Dashboards |
| **Diagnostic Settings** | Route resource logs to workspace/storage/hub | CloudWatch Log Subscription Filters |

---

## Log Analytics Workspace

Everything flows into a workspace. Most Azure resources can send logs and metrics here via **Diagnostic Settings**.

```bash
# Create a Log Analytics Workspace
az monitor log-analytics workspace create \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --location eastus \
  --retention-time 90          # days (default 30, max 730)

# Get the workspace ID (needed for diagnostic settings)
az monitor log-analytics workspace show \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --query customerId \
  --output tsv
```

---

## Diagnostic Settings (route resource logs to workspace)

```bash
# Enable logs + metrics for a storage account
az monitor diagnostic-settings create \
  --name storageDiagnostics \
  --resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --workspace /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace \
  --logs '[{"category": "StorageRead", "enabled": true}, {"category": "StorageWrite", "enabled": true}]' \
  --metrics '[{"category": "Transaction", "enabled": true}]'

# Enable for an AKS cluster
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

---

## Application Insights (APM)

Application Insights provides end-to-end distributed tracing, exception tracking, performance monitoring, and custom telemetry for your applications.

```bash
# Create an Application Insights resource
az monitor app-insights component create \
  --resource-group myRG \
  --app myAppInsights \
  --location eastus \
  --workspace /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace

# Get the connection string (use this in your app, not the old instrumentation key)
az monitor app-insights component show \
  --resource-group myRG \
  --app myAppInsights \
  --query connectionString \
  --output tsv
```

### Instrument a Node.js app

```bash
npm install @azure/monitor-opentelemetry
```

```javascript
// instrument.js — import FIRST, before any other module
const { useAzureMonitor } = require("@azure/monitor-opentelemetry");

useAzureMonitor({
  azureMonitorExporterOptions: {
    connectionString: process.env.APPLICATIONINSIGHTS_CONNECTION_STRING,
  },
});

// The SDK auto-instruments:
// - HTTP/HTTPS requests (incoming and outgoing)
// - Express / Fastify / Koa routes
// - Database calls (pg, mysql2, mongodb, redis)
// - Azure SDK calls
```

```javascript
// Custom telemetry
const { trace, metrics } = require("@opentelemetry/api");

const tracer = trace.getTracer("my-service");
const meter = metrics.getMeter("my-service");

// Custom span (trace)
const span = tracer.startSpan("process-order");
span.setAttribute("order.id", orderId);
span.setAttribute("order.value", total);
try {
  // do work
  span.setStatus({ code: SpanStatusCode.OK });
} catch (err) {
  span.recordException(err);
  span.setStatus({ code: SpanStatusCode.ERROR });
} finally {
  span.end();
}

// Custom metric (counter)
const orderCounter = meter.createCounter("orders.processed");
orderCounter.add(1, { region: "eastus", tier: "premium" });
```

---

## KQL (Kusto Query Language)

KQL is the query language for Log Analytics. Syntax is a pipeline of transformations — similar to Unix pipes or Splunk SPL.

### Basic KQL Patterns

```kusto
// Most recent 100 exceptions from Application Insights
exceptions
| where timestamp > ago(1h)
| order by timestamp desc
| take 100

// Request failure rate by operation
requests
| where timestamp > ago(24h)
| summarize
    total = count(),
    failed = countif(success == false)
    by name
| extend failureRate = round(100.0 * failed / total, 2)
| order by failureRate desc

// Slow dependencies (external calls > 500ms)
dependencies
| where timestamp > ago(1h)
| where duration > 500
| project timestamp, name, type, target, duration, success
| order by duration desc

// Container logs from AKS
ContainerLogV2
| where TimeGenerated > ago(30m)
| where ContainerName == "my-app"
| where LogMessage contains "ERROR"
| project TimeGenerated, PodName, LogMessage
| order by TimeGenerated desc

// VM CPU over 80%
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where CounterValue > 80
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart

// Join two tables (requests with their exceptions)
requests
| where timestamp > ago(1h)
| where success == false
| join kind=leftouter (
    exceptions
    | where timestamp > ago(1h)
) on operation_Id
| project timestamp, name, resultCode, outerMessage
```

---

## Alerts

### Metric Alert (fire when a metric crosses a threshold)

```bash
az monitor metrics alert create \
  --resource-group myRG \
  --name "High CPU Alert" \
  --scopes /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --condition "avg Percentage CPU > 85" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action myActionGroup
```

### Log Alert (fire when a KQL query returns results)

```bash
az monitor scheduled-query create \
  --resource-group myRG \
  --name "App Error Rate Alert" \
  --scopes /subscriptions/<sub-id>/resourceGroups/myRG/providers/microsoft.insights/components/myAppInsights \
  --condition-query "requests | where timestamp > ago(5m) | summarize failRate = countif(success==false) * 100.0 / count() | where failRate > 5" \
  --condition-time-aggregation Count \
  --condition-operator GreaterThan \
  --condition-threshold 0 \
  --evaluation-frequency 5m \
  --window-size 5m \
  --severity 1 \
  --action-groups myActionGroup
```

### Activity Log Alert (fire on Azure resource changes)

```bash
# Alert when any VM is deleted in the subscription
az monitor activity-log alert create \
  --resource-group myRG \
  --name "VM Deleted Alert" \
  --scope /subscriptions/<sub-id> \
  --condition category=Administrative and operationName=Microsoft.Compute/virtualMachines/delete \
  --action-groups myActionGroup
```

---

## Action Groups (who to notify)

```bash
# Create an action group
az monitor action-group create \
  --resource-group myRG \
  --name myActionGroup \
  --short-name myAG \
  --action email devteam devteam@example.com \
  --action sms oncall +919876543210

# Add a webhook (e.g., PagerDuty, Slack, Teams)
az monitor action-group update \
  --resource-group myRG \
  --name myActionGroup \
  --add-action webhook pagerduty https://events.pagerduty.com/integration/<key>/enqueue

# Add an Azure Function as an action
az monitor action-group update \
  --resource-group myRG \
  --name myActionGroup \
  --add-action azurefunction alertHandler \
    /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function-app \
    /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function-app/functions/alertHandler \
    https://my-function-app.azurewebsites.net/api/alertHandler
```

---

## Metrics Explorer (quick CLI queries)

```bash
# Get CPU metrics for a VM (last hour, 5 minute bins)
az monitor metrics list \
  --resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Compute/virtualMachines/myVM \
  --metric "Percentage CPU" \
  --interval PT5M \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --output table

# List available metrics for a resource type
az monitor metrics list-definitions \
  --resource /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorage \
  --output table
```

---

## Key Differences from AWS CloudWatch

| Feature | AWS CloudWatch | Azure Monitor |
|---------|---------------|---------------|
| Log query language | CloudWatch Logs Insights (custom) | KQL (Kusto) |
| Log storage | Log Groups | Log Analytics Workspace |
| APM / tracing | X-Ray + Container Insights | Application Insights |
| Dashboards | CloudWatch Dashboards | Workbooks + Azure Dashboards |
| Notifications | SNS | Action Groups |
| Metric alarms | CloudWatch Alarms | Metric Alerts |
| Log-based alerts | CloudWatch Metric Filters + Alarms | Scheduled Query Alerts |
| Resource event alerts | CloudTrail + EventBridge | Activity Log Alerts |
| Agent (on VMs) | CloudWatch Agent | Azure Monitor Agent (AMA) |