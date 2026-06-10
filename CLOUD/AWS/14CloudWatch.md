# Amazon CloudWatch

Amazon CloudWatch is AWS's unified observability service — providing **metrics, logs, alarms, dashboards, and events** for your entire AWS infrastructure and applications. It's the primary tool for monitoring, debugging, and automating responses to operational changes.

---

## What CloudWatch Provides

| Feature | Description |
|---------|-------------|
| **Metrics** | Time-series data points: CPU usage, memory, request count, error rate, latency |
| **Logs** | Collect, store, and search log output from any AWS service or custom app |
| **Alarms** | Trigger notifications or automated actions when metrics cross thresholds |
| **Dashboards** | Real-time visualization of metrics and logs |
| **Events / EventBridge** | React to changes in AWS resources or run scheduled tasks |
| **Logs Insights** | Interactive query language for analyzing log data at scale |
| **Container Insights** | Enhanced monitoring for ECS, EKS, and Kubernetes |
| **Application Insights** | Automated monitoring and anomaly detection for applications |
| **Synthetics** | Canary scripts that simulate user interactions to catch issues proactively |

---

## Metrics

CloudWatch collects **metrics** — numerical data points associated with your resources over time.

### Built-in Metrics (automatic, no setup needed)
- **EC2**: CPUUtilization, NetworkIn, NetworkOut, DiskReadOps, StatusCheckFailed
- **RDS**: DatabaseConnections, FreeStorageSpace, ReadLatency, WriteLatency
- **Lambda**: Invocations, Errors, Duration, Throttles, ConcurrentExecutions
- **ALB**: RequestCount, TargetResponseTime, HTTPCode_Target_5XX_Count
- **SQS**: NumberOfMessagesSent, ApproximateAgeOfOldestMessage
- **DynamoDB**: SuccessfulRequestLatency, ConsumedReadCapacityUnits

### Custom Metrics
Publish your own metrics from any application:

```bash
# Via CLI
aws cloudwatch put-metric-data \
  --namespace "MyApp" \
  --metric-name "ActiveUsers" \
  --value 452 \
  --unit Count
```

```javascript
// Via AWS SDK (Node.js)
const { CloudWatchClient, PutMetricDataCommand } = require("@aws-sdk/client-cloudwatch");

const client = new CloudWatchClient({ region: "us-east-1" });

await client.send(new PutMetricDataCommand({
  Namespace: "MyApp",
  MetricData: [{
    MetricName: "OrdersProcessed",
    Value: 1,
    Unit: "Count",
    Dimensions: [{ Name: "Environment", Value: "production" }],
    Timestamp: new Date(),
  }]
}));
```

---

## Logs

CloudWatch Logs stores log output from AWS services and custom applications.

### Automatic Log Sources
- **Lambda**: `/aws/lambda/<function-name>` — enabled by default
- **ECS**: configured via `awslogs` log driver in task definition
- **API Gateway**: enable access logging in stage settings
- **RDS**: enable via parameter groups
- **VPC Flow Logs**: enable per VPC

### Sending Custom App Logs to CloudWatch

**From EC2 (CloudWatch Agent):**
```bash
# Install agent
sudo apt install amazon-cloudwatch-agent

# Configure (minimal example)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Or manually write config
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
{
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/home/ubuntu/app/output.log",
          "log_group_name": "/myapp/production",
          "log_stream_name": "{instance_id}"
        }]
      }
    }
  }
}
EOF

sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

**Via SDK (Node.js):**
```javascript
const { CloudWatchLogsClient, PutLogEventsCommand } = require("@aws-sdk/client-cloudwatch-logs");

const client = new CloudWatchLogsClient({ region: "us-east-1" });

await client.send(new PutLogEventsCommand({
  logGroupName: "/myapp/production",
  logStreamName: "instance-1",
  logEvents: [{
    timestamp: Date.now(),
    message: JSON.stringify({ level: "ERROR", message: "DB connection failed" })
  }]
}));
```

### Viewing Logs via CLI
```bash
# List log groups
aws logs describe-log-groups

# Tail live logs (like tail -f)
aws logs tail /aws/lambda/my-function --follow

# Filter logs by pattern
aws logs filter-log-events \
  --log-group-name /aws/lambda/my-function \
  --filter-pattern "ERROR" \
  --start-time $(date -d "1 hour ago" +%s000)
```

---

## CloudWatch Logs Insights

A powerful query language for analyzing log data across log groups:

```sql
-- Find top 10 error types in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| stats count(*) as errorCount by errorType
| sort errorCount desc
| limit 10

-- Lambda cold start analysis
fields @timestamp, @type, @duration, @billedDuration, @initDuration
| filter @type = "REPORT"
| stats avg(@initDuration), max(@initDuration), count(*) by bin(5m)

-- API response time percentiles
fields @timestamp, @message
| filter @message like /duration/
| parse @message /duration: (?<duration>\d+)/
| stats avg(duration), p90(duration), p99(duration) by bin(1h)
```

Access via: **CloudWatch → Logs Insights**

---

## Alarms

Alarms trigger notifications or automated actions when a metric crosses a threshold.

### Creating an Alarm via Console
1. Go to **CloudWatch → Alarms → Create alarm**
2. Select a metric (e.g., EC2 `CPUUtilization`)
3. Set threshold (e.g., > 80% for 5 minutes)
4. Choose action: **SNS notification**, **Auto Scaling**, **EC2 action**, or **Lambda**

### Creating an Alarm via CLI
```bash
# Alert if EC2 CPU > 80% for 2 consecutive 5-minute periods
aws cloudwatch put-metric-alarm \
  --alarm-name "HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:my-alert-topic \
  --ok-actions arn:aws:sns:us-east-1:123456789012:my-alert-topic

# Check alarm states
aws cloudwatch describe-alarms --state-value ALARM
```

### Composite Alarms
Combine multiple alarms with AND/OR logic to reduce alert noise:
```bash
aws cloudwatch put-composite-alarm \
  --alarm-name "AppDegraded" \
  --alarm-rule "ALARM(HighCPU) AND ALARM(HighLatency)"
```

---

## Dashboards

Create real-time dashboards combining metrics and log widgets:

1. Go to **CloudWatch → Dashboards → Create dashboard**
2. Add widgets: Line chart, Number, Bar chart, Log table, Alarm status
3. Select metrics or log queries for each widget
4. Share dashboards with the team

**Useful dashboard widgets:**
- EC2 CPU across all instances
- Lambda error rate over time
- RDS connection count
- ALB 5xx error rate
- Custom business metrics (orders/minute, active users)

---

## EventBridge (formerly CloudWatch Events)

Respond to AWS resource changes or run scheduled tasks:

### Scheduled Rule (cron job)
```bash
# Trigger Lambda every day at 9 AM UTC
aws events put-rule \
  --name "DailyReport" \
  --schedule-expression "cron(0 9 * * ? *)" \
  --state ENABLED

aws events put-targets \
  --rule "DailyReport" \
  --targets "Id=1,Arn=arn:aws:lambda:us-east-1:123456789012:function:generateReport"
```

### Event Pattern (react to AWS changes)
```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```
Trigger cleanup actions whenever an EC2 instance is terminated.

---

## CloudWatch Agent (EC2 Enhanced Monitoring)

The default EC2 metrics don't include **memory or disk usage** — you need the CloudWatch Agent for those:

```bash
# Install
sudo apt install amazon-cloudwatch-agent

# After configuring, the agent publishes:
# - mem_used_percent
# - disk_used_percent
# - swap_used_percent
# - net_bytes_sent / net_bytes_recv
```

---

## Common Monitoring Patterns

### Lambda Monitoring
Key metrics to alarm on:
- `Errors` > 0 → something is failing
- `Throttles` > 0 → hitting concurrency limits
- `Duration` > 80% of timeout → risk of timeouts

### API / Web App Monitoring
- ALB `HTTPCode_Target_5XX_Count` > threshold → alarm
- ALB `TargetResponseTime` p99 > 2s → latency alarm
- Custom metric: requests per second, error rate per endpoint

### Database Monitoring
- RDS `FreeStorageSpace` < 20% → disk almost full
- RDS `DatabaseConnections` approaching max → connection pool issue
- DynamoDB `ThrottledRequests` > 0 → capacity too low

---

## Costs

- **Metrics**: First 10 custom metrics free; then $0.30/metric/month
- **Alarms**: $0.10/alarm/month (standard resolution)
- **Logs**: $0.50/GB ingested; $0.03/GB stored per month
- **Dashboards**: $3/dashboard/month (first 3 free)
- **Logs Insights**: $0.005 per GB scanned

> **Tip**: Use log retention policies to avoid paying indefinitely for old logs. Set 30–90 day retention for most log groups.

```bash
# Set 30-day retention on a log group
aws logs put-retention-policy \
  --log-group-name /myapp/production \
  --retention-in-days 30
```