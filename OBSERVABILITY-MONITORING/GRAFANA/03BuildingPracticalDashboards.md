# Grafana - Building Practical Dashboards

Hands-on: template variables, building one reusable dashboard across environments, and practical query patterns that come up constantly in real DevOps dashboards.

## Table of Contents
- [The Reusability Problem](#the-reusability-problem)
- [Template Variables](#template-variables)
- [Variable Types](#variable-types)
- [Using Variables in Queries](#using-variables-in-queries)
- [Chained Variables](#chained-variables)
- [Practical PromQL Patterns](#practical-promql-patterns)
- [Practical LogQL Patterns](#practical-logql-patterns)
- [Annotations](#annotations)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Reusability Problem

**Visual:**
```
Without Variables:
"Production API Dashboard"    (hardcoded to prod)
"Staging API Dashboard"        (hardcoded to staging)
"Dev API Dashboard"             (hardcoded to dev)
→ 3 separate dashboards to maintain, update, keep in sync

With Variables:
"API Dashboard" + [Environment: prod ▼]
→ ONE dashboard, switch environment via dropdown,
   all panels automatically re-query for the selected value
```

---

## Template Variables

**A Variable is a placeholder that becomes a dropdown at the top of the dashboard.**

**Visual:**
```
Dashboard Header:
┌─────────────────────────────────────────────┐
│ [Environment: production ▼]  [Service: api ▼]   │
└─────────────────────────────────────────────┘

Changing "Environment" from production → staging
instantly re-runs EVERY panel's query with the
new value substituted in — no manual editing needed.
```

**Creating a variable:**
```
Dashboard Settings → Variables → Add variable

Name: environment
Type: Query
Data source: Prometheus
Query: label_values(up, env)
```

---

## Variable Types

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Type              Behavior                              │
├───────────────────────────────────────────────────┤
│ Query               Populated dynamically from a data     │
│                    source query (e.g. list of hostnames)   │
│                                                        │
│ Custom               Manually defined static list             │
│                    (e.g. "prod,staging,dev")                     │
│                                                        │
│ Constant              A fixed, hidden value (rarely shown       │
│                    to viewers, used for dashboard defaults)        │
│                                                        │
│ Data source            Lets the viewer switch which entire          │
│                    data source the dashboard queries against         │
│                                                        │
│ Interval               Time-based (e.g. "5m, 1h, 1d") for              │
│                    controlling aggregation granularity                   │
└───────────────────────────────────────────────────┘
```

**Custom variable example:**
```
Name: environment
Type: Custom
Values: production,staging,development
```

**Visual:**
```
Renders as:
[Environment: production ▼]
              staging
              development
```

---

## Using Variables in Queries

Once defined, reference a variable with `$variablename` or `${variablename}`.

**PromQL example:**
```
rate(http_requests_total{env="$environment", service="$service"}[5m])
```

**Visual:**
```
If dropdown selects: environment=staging, service=payments

Query sent to Prometheus becomes:
rate(http_requests_total{env="staging", service="payments"}[5m])

The panel title, legend, and query all update automatically —
this is what makes ONE dashboard work across every environment.
```

**Multi-value variables (select more than one at once):**
```
Variable settings: ☑ Multi-value  ☑ Include "All" option

Query becomes:
rate(http_requests_total{service=~"$service"}[5m])
                              ↑
                    note: =~ (regex match) instead of =
                    required when a variable can hold
                    multiple comma-separated values
```

---

## Chained Variables

**One variable's options can depend on another variable's selection.**

**Visual:**
```
Variable 1: $environment  →  [production ▼]
Variable 2: $service       →  depends on $environment's value

Query for $service:
label_values(up{env="$environment"}, service)

Result:
If $environment = production → $service shows: api, payments, auth
If $environment = staging     → $service shows: api-staging, test-svc

This prevents selecting an invalid/nonexistent combination
(e.g. a service that doesn't exist in that environment).
```

---

## Practical PromQL Patterns

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Goal                        PromQL Pattern                │
├─────────────────────────────────────────────────────┤
│ Requests per second           rate(http_requests_total[5m])│
│                                                        │
│ Error rate percentage           sum(rate(http_requests_total │
│                                {status=~"5.."}[5m]))            │
│                                / sum(rate(http_requests_total[5m]))│
│                                * 100                                │
│                                                        │
│ P99 latency                     histogram_quantile(0.99,             │
│                                rate(http_request_duration_seconds_bucket[5m]))│
│                                                        │
│ Memory usage %                    container_memory_usage_bytes /       │
│                                container_spec_memory_limit_bytes * 100     │
│                                                        │
│ Is service up (1 or 0)              up{job="api"}                            │
└─────────────────────────────────────────────────────┘
```

**Visual — building the error rate ratio query step by step:**
```
Step 1: sum(rate(http_requests_total{status=~"5.."}[5m]))
        → total 5xx errors per second, summed across all instances

Step 2: sum(rate(http_requests_total[5m]))
        → total requests per second (all statuses), summed

Step 3: (Step 1) / (Step 2) * 100
        → percentage of requests that are errors

This 3-step decomposition is the standard pattern for
ANY "rate of X out of total" dashboard panel.
```

---

## Practical LogQL Patterns

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Goal                        LogQL Pattern                   │
├─────────────────────────────────────────────────────┤
│ All logs from a service        {app="payments"}                  │
│                                                        │
│ Only error logs                  {app="payments"} |= "ERROR"        │
│                                                        │
│ Count errors per minute            sum(count_over_time(                 │
│                                {app="payments"} |= "ERROR" [1m]))          │
│                                                        │
│ Parse JSON and filter a field        {app="payments"} | json                │
│                                | status_code="500"                             │
│                                                        │
│ Extract and use a specific field       {app="payments"} | json                    │
│                                | line_format "{{.message}}"                          │
└─────────────────────────────────────────────────────┘
```

**Visual — turning logs into a metric (LogQL's powerful feature):**
```
{app="payments"} |= "ERROR"     ← raw log lines matching "ERROR"
        ↓
count_over_time(...[1m])         ← count of matching lines per minute
        ↓
sum(...)                         ← total across all instances/pods
        ↓
Result: a NUMBER over time, usable in a Time Series panel,
        exactly like a Prometheus metric — even though the
        underlying data source is just log lines.
```

---

## Annotations

**Annotations mark specific points in time on a graph — extremely useful for correlating deployments with metric changes.**

**Visual:**
```
Time Series Panel with Annotation:
Requests/sec
   │
   │        📍 Deploy v2.3.1
   │           │
   │      ╱‾‾‾╲│___
   │  ╱‾‾╱      ╲___╲___
   └──────────────────────→ time

The vertical marker instantly shows "this dip/spike
happened right after a deployment" — without annotations,
you'd have to separately cross-reference deployment
timestamps against the graph manually.
```

**Setting up automatic deployment annotations (via API):**
```bash
curl -X POST http://grafana:3000/api/annotations \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "text": "Deployed v2.3.1",
    "tags": ["deployment", "payments-service"]
  }'
```

**Visual:**
```
CI/CD Pipeline Integration:
Deploy stage completes → pipeline calls Grafana annotation API →
   Annotation appears automatically on ALL relevant dashboards →
   Team can immediately correlate "did this deploy cause the spike?"
```

---

## Real-Life DevOps Use Case

**Scenario:** A platform team supports 15 microservices across 3 environments (dev, staging, production) and wants ONE dashboard instead of 45 separate ones.

**What they build:**
1. A single **"Service Overview" dashboard** with two chained variables: `$environment` (Custom: dev, staging, production) and `$service` (Query, chained to environment, using `label_values(up{env="$environment"}, service)`).
2. All panels use `$environment` and `$service` in their PromQL queries, so switching either dropdown instantly re-scopes every panel — request rate, error rate, latency, and resource usage — to the selected combination.
3. Adds **deployment annotations** automatically pushed from the CI/CD pipeline via the Grafana API, so whenever any team deploys any service, a marker appears on the dashboard — letting engineers immediately correlate "did the last deploy cause this latency increase?"
4. Uses the **multi-value "All" option** on `$service` so a team lead can view aggregate metrics across all 15 services at once when doing a weekly health review, then narrow to one specific service when investigating an issue.
5. Sets a **default value** for `$environment` to "production" (since that's checked most often) but makes switching to staging trivial for pre-release validation.

**Why this matters:** Before variables, the platform team maintained 45 nearly-identical dashboards (15 services × 3 environments) that constantly drifted out of sync whenever someone updated one but forgot the other 44. One well-templated dashboard eliminates that maintenance burden entirely.

---

Next: **04alerting.md** — turning dashboard queries into actual alert rules that notify the team via Slack, PagerDuty, or email before customers notice a problem.