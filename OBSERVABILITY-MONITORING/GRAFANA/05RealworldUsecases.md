# Grafana - Advanced Features & Real-World Use Cases

Going beyond manual dashboard building: dashboards-as-code, plugins, permissions/RBAC, and how mature organizations operate Grafana at scale.

## Table of Contents
- [Dashboards as Code (Provisioning)](#dashboards-as-code-provisioning)
- [Data Source Provisioning](#data-source-provisioning)
- [The Grafana Plugin Ecosystem](#the-grafana-plugin-ecosystem)
- [Organizations, Teams & Permissions](#organizations-teams--permissions)
- [Explore Mode for Incident Response](#explore-mode-for-incident-response)
- [Grafana API for Automation](#grafana-api-for-automation)
- [Common Pitfalls & War Stories](#common-pitfalls--war-stories)
- [Real-Life DevOps Use Case (End-to-End)](#real-life-devops-use-case-end-to-end)

---

## Dashboards as Code (Provisioning)

**Instead of manually clicking through the UI, dashboards can be defined as JSON files and version-controlled in Git, then automatically loaded on startup.**

**Visual:**
```
Manual UI Approach:
Build dashboard in browser → lives ONLY in Grafana's database →
   no history, no code review, easy to accidentally break,
   hard to replicate across environments

Provisioning-as-Code Approach:
dashboards/api-health.json (in Git) → committed, PR-reviewed →
   Grafana automatically loads it on startup from a config'd folder
```

**Provisioning config:**
```yaml
# /etc/grafana/provisioning/dashboards/dashboards.yaml
apiVersion: 1
providers:
  - name: "default"
    folder: "Platform Dashboards"
    type: file
    options:
      path: /var/lib/grafana/dashboards
```

**Visual:**
```
Directory Structure:
/var/lib/grafana/dashboards/
├── api-health.json
├── database-overview.json
└── kubernetes-cluster.json

On Grafana startup:
Reads dashboards.yaml config → finds the folder path →
   loads every .json file as a dashboard automatically →
   dashboards appear under "Platform Dashboards" folder
```

**Exporting an existing dashboard to JSON:**
```
Dashboard Settings → JSON Model → Copy → save as .json file
```

**Visual — why this matters for DevOps:**
```
Git repo becomes the SOURCE OF TRUTH for dashboards:
- Code review before a dashboard change goes live
- Full history of who changed what and why
- Same dashboards automatically deployed to dev/staging/prod
- Disaster recovery: rebuild Grafana from scratch, dashboards
  reappear automatically from the provisioning folder
```

---

## Data Source Provisioning

Data sources themselves can also be defined as code, avoiding manual UI setup per environment.

```yaml
# /etc/grafana/provisioning/datasources/datasources.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus.monitoring.svc:9090
    isDefault: true

  - name: Loki
    type: loki
    access: proxy
    url: http://loki.monitoring.svc:3100
```

**Visual:**
```
Without provisioning:
New Grafana instance → someone manually clicks through
"Add data source" for each environment → easy to typo a
URL, forget a data source, or configure inconsistently

With provisioning:
New Grafana instance starts → reads datasources.yaml →
   Prometheus and Loki connections configured identically,
   automatically, every single time
```

---

## The Grafana Plugin Ecosystem

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Plugin Type          Examples                             │
├─────────────────────────────────────────────────────┤
│ Data Source Plugins     Splunk, Datadog, MongoDB, Zabbix,      │
│                       and dozens more non-default sources        │
│                                                        │
│ Panel Plugins            Custom visualizations (e.g. flowcharts,   │
│                       specialized business-metric widgets)            │
│                                                        │
│ App Plugins                Full mini-applications embedded            │
│                       within Grafana (e.g. Kubernetes app,               │
│                       incident management app)                             │
└─────────────────────────────────────────────────────┘
```

```bash
grafana-cli plugins install grafana-piechart-panel
systemctl restart grafana-server
```

**Visual:**
```
Why plugins matter for DevOps:
Instead of building a custom integration for every
tool the company uses, check the plugin catalog first —
most major monitoring/logging tools already have an
official or community-maintained Grafana plugin.
```

---

## Organizations, Teams & Permissions

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│                    Grafana Instance                       │
│  ┌─────────────────────────────────────────────┐   │
│  │              Organization: "Acme Corp"            │   │
│  │  ┌──────────────┐  ┌──────────────┐          │   │
│  │  │  Team: Payments  │  │  Team: Platform    │          │   │
│  │  │  - Alice (Editor) │  │  - Dave (Admin)     │          │   │
│  │  │  - Bob (Viewer)     │  │  - Eve (Editor)       │          │   │
│  │  └──────────────┘  └──────────────┘          │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘

Folder-level Permissions:
"Payments Dashboards" folder → only Payments team can Edit,
                                Platform team can only View
"Platform Dashboards" folder  → only Platform team can Edit,
                                everyone else can View
```

**Role types:**
```
Viewer  → can view dashboards, cannot edit or create
Editor  → can create/edit dashboards and alert rules
Admin   → can manage data sources, users, permissions
```

**Visual — why folder permissions matter:**
```
Without scoped permissions:
Any Editor can accidentally modify ANY team's dashboard →
   payments team's carefully-tuned alert thresholds get
   accidentally changed by someone from a different team

With folder-level permissions:
Each team's dashboards/alerts are protected from accidental
cross-team edits, while still being VIEWABLE org-wide for
transparency.
```

---

## Explore Mode for Incident Response

**Explore is Grafana's ad-hoc query interface — no dashboard needed, just investigate directly.**

**Visual:**
```
Incident Response Flow Using Explore:
1. Alert fires: "Payments error rate high"
2. Engineer opens Explore, selects Prometheus
3. Runs ad-hoc query: rate(http_requests_total{service="payments",status=~"5.."}[1m])
4. Sees the spike started at 14:32
5. SWITCHES data source (same Explore view) to Loki
6. Queries: {service="payments"} |= "ERROR" (time range auto-syncs to 14:30-14:35)
7. Finds the exact stack trace causing the errors

All without ever having to build or navigate to a
pre-made dashboard — pure ad-hoc investigation.
```

**Split view (compare two queries side-by-side):**
```
┌─────────────────┬─────────────────┐
│  Prometheus query    │  Loki query            │
│  (metric spike)         │  (correlated logs)        │
└─────────────────┴─────────────────┘
Both panes share the same time range —
extremely useful for correlating "what does the
metric say" against "what do the logs say" at
the exact same moment.
```

---

## Grafana API for Automation

```bash
# Create a dashboard programmatically
curl -X POST http://grafana:3000/api/dashboards/db \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json" \
  -d @dashboard.json

# Get all dashboards in a folder
curl -H "Authorization: Bearer $GRAFANA_API_KEY" \
  http://grafana:3000/api/search?folderIds=5

# Push a deployment annotation (seen in file 03)
curl -X POST http://grafana:3000/api/annotations \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -d '{"text":"Deployed v2.1.0","tags":["deploy"]}'
```

**Visual:**
```
Common Automation Patterns:
┌──────────────────────────────────────────────────┐
│  Task                        API Use                    │
├──────────────────────────────────────────────────┤
│  New service onboarding         Auto-generate a standard    │
│                                dashboard from a template        │
│  CI/CD deployment tracking       Push annotations automatically  │
│  Bulk permission updates          Script folder permission changes │
│  Dashboard backup/migration        Export all dashboards via API    │
│                                for disaster recovery                  │
└──────────────────────────────────────────────────┘
```

---

## Common Pitfalls & War Stories

**Visual:**
```
Pitfall 1: "Dashboard looked fine in staging, empty in production"
Cause: Hardcoded data source UID differs between environments
Fix: Use provisioned data sources with consistent names,
     or template the data source itself as a variable

Pitfall 2: "Someone accidentally deleted a critical dashboard"
Cause: No version control, dashboard only existed in Grafana's DB
Fix: Provisioning-as-code with dashboards in Git — restore
     by just re-provisioning, no data loss

Pitfall 3: "Alert fired 200 times in 10 minutes, paging fatigue"
Cause: No "For" duration set, alert fires on every single
       evaluation cycle while the condition remains true
Fix: Grafana's alerting groups repeated firings and only
     re-notifies on state CHANGE by default — but double-check
     notification policy "repeat interval" isn't set too aggressively

Pitfall 4: "Dashboard queries are extremely slow"
Cause: Overly broad time range + high cardinality query
       (e.g. grouping by a label with 100,000 unique values)
Fix: Add recording rules in Prometheus for expensive queries,
     narrow default time ranges, avoid grouping by high-cardinality labels

Pitfall 5: "Everyone has Admin access, changes happen with no accountability"
Cause: Team defaulted to giving all users Admin for convenience
Fix: Scope roles properly (Viewer/Editor/Admin) and use folder
     permissions, reserving Admin for platform team only
```

---

## Real-Life DevOps Use Case (End-to-End)

**Scenario:** A mid-size company's platform team wants Grafana to be a mature, self-service observability platform for 15 engineering teams — not something the platform team manually configures for every request.

**Full workflow the team builds:**

1. **Everything as code:** All dashboards and data sources live in a `grafana-config` Git repo, provisioned automatically on every Grafana deployment — no manual UI configuration for anything that matters long-term.
2. **Standard dashboard template:** New services get a starter dashboard (request rate, error rate, latency, resource usage) automatically generated via a script calling the Grafana API whenever a new service is registered in the company's internal service catalog.
3. **Team-scoped permissions:** Each team has its own folder with Editor rights scoped to their team, while the whole org has Viewer access to everything — balancing team autonomy with organization-wide transparency.
4. **CI/CD-driven annotations:** Every deployment across all 15 teams automatically pushes an annotation via the Grafana API, so any dashboard viewer can immediately correlate a metric change with a recent deploy.
5. **Incident response workflow:** On-call engineers are trained to use **Explore's split view** to correlate Prometheus metrics against Loki logs side-by-side as their first investigative step, rather than building a new dashboard mid-incident.
6. **Alert governance:** A shared alerting library of common patterns (error rate, latency, saturation) lives in the same Git repo, so teams adopt proven alert rule templates rather than each reinventing (and potentially misconfiguring) their own from scratch.
7. **Disaster recovery tested:** The team periodically tears down and rebuilds a test Grafana instance purely from the provisioning repo, verifying that zero manual steps are needed to restore full dashboard/data-source functionality.

**Why this is "real DevOps," not just running a tool:** Grafana here isn't just "a pretty graphs website" — it's a governed, version-controlled, self-service platform integrated into the deployment pipeline (annotations), service catalog (auto-provisioned dashboards), and incident response process (Explore-first investigation). This is the difference between "we have Grafana" and "Grafana is how our organization sees and understands its own systems."

---

This completes the Grafana note series: **Introduction → Setup → Dashboards & Panels → Practical Dashboard Building → Alerting → Advanced/Real-World Usage.**