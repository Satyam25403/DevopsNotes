# Google App Engine

App Engine is GCP's fully managed **Platform-as-a-Service (PaaS)** that abstracts away infrastructure management. You provide your code; Google provisions and manages everything needed to run it. It's the GCP equivalent of AWS Elastic Beanstalk.

---

## The Problem It Solves

Running a production app requires:
- A server (OS, CPU, memory)
- A runtime (Python, Node.js, Java…)
- A web server (Nginx, Gunicorn…)
- Networking (VPC, load balancer, firewall)
- Scaling, monitoring, and deployment pipelines

App Engine handles **all of this automatically**.

> If Compute Engine is renting a raw server and configuring everything yourself, App Engine is handing Google your app and saying: *"Just make it run."*

---

## Two Environments

| Feature | Standard Environment | Flexible Environment |
|---------|---------------------|---------------------|
| Runtimes | Fixed (Python, Node.js, Go, Java, PHP, Ruby) | Any via custom Docker image |
| Scaling | To zero, very fast cold starts | Min 1 instance always running |
| Pricing | Per instance-hour (cheap at low traffic) | Per vCPU/memory-hour |
| Infrastructure | Google-managed sandbox | Managed Compute Engine VMs |
| SSH access | ❌ | ✅ |
| Best for | Web apps with variable traffic | Long-running processes, custom runtimes |

---

## Deploying to App Engine

### Basic Deployment
```bash
# Enable the App Engine API
gcloud services enable appengine.googleapis.com

# Initialize App Engine (one-time per project)
gcloud app create --region=us-central

# Deploy your app (requires app.yaml in the current directory)
gcloud app deploy

# Deploy a specific app.yaml
gcloud app deploy app.yaml --quiet

# View the deployed app
gcloud app browse
```

### `app.yaml` — Standard Environment (Node.js)
```yaml
runtime: nodejs20

instance_class: F2

automatic_scaling:
  min_instances: 0
  max_instances: 10
  target_cpu_utilization: 0.65

env_variables:
  NODE_ENV: "production"
  PORT: "8080"

handlers:
  - url: /.*
    script: auto
    secure: always
```

### `app.yaml` — Standard Environment (Python)
```yaml
runtime: python312

instance_class: F2

automatic_scaling:
  min_instances: 1
  max_instances: 20

entrypoint: gunicorn -b :$PORT main:app

env_variables:
  FLASK_ENV: "production"
```

### `app.yaml` — Flexible Environment (Custom Docker)
```yaml
runtime: custom
env: flex

resources:
  cpu: 1
  memory_gb: 1.3
  disk_size_gb: 10

automatic_scaling:
  min_num_instances: 1
  max_num_instances: 10
  cpu_utilization:
    target_utilization: 0.65

env_variables:
  NODE_ENV: production
```

---

## CLI Commands

```bash
# Deploy
gcloud app deploy

# View logs
gcloud app logs tail -s default

# List services (microservices)
gcloud app services list

# List versions of a service
gcloud app versions list

# Migrate traffic to a new version
gcloud app services set-traffic default --splits=v2=1

# Canary: split traffic between versions
gcloud app services set-traffic default --splits=v1=0.8,v2=0.2

# Stop an old version (stops billing but keeps it available)
gcloud app versions stop v1

# Delete an old version
gcloud app versions delete v1
```

---

## Traffic Splitting (Blue/Green & Canary)

```bash
# Deploy new version without traffic
gcloud app deploy --no-promote --version=v2

# Gradually shift traffic
gcloud app services set-traffic default \
  --splits=v1=0.9,v2=0.1 \
  --split-by=random

# Full cutover
gcloud app services set-traffic default --splits=v2=1

# Rollback
gcloud app services set-traffic default --splits=v1=1
```

Split strategies:
- `--split-by=random` — random assignment per request
- `--split-by=ip` — sticky by client IP
- `--split-by=cookie` — sticky by session cookie (smoother UX)

---

## Multiple Services (Microservices)

App Engine supports multiple services in one project. Each service has its own `app.yaml` and scales independently.

```bash
# Deploy additional services
gcloud app deploy services/api/app.yaml
gcloud app deploy services/worker/app.yaml
gcloud app deploy services/admin/app.yaml
```

Routing:
- `https://my-project.uc.r.appspot.com` → `default` service
- `https://api-dot-my-project.uc.r.appspot.com` → `api` service
- `https://v2-dot-api-dot-my-project.uc.r.appspot.com` → version `v2` of `api` service

---

## Environment Variables and Secrets

```bash
# Set env vars in app.yaml
env_variables:
  API_KEY: "my-value"          # ⚠️ Not for secrets — visible in source control

# For secrets, use Secret Manager and load in code:
# const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');
```

---

## Cron Jobs (Scheduled Tasks)

Create a `cron.yaml` file:
```yaml
cron:
  - description: "Daily cleanup job"
    url: /tasks/cleanup
    schedule: every 24 hours

  - description: "Send weekly digest"
    url: /tasks/weekly-digest
    schedule: every monday 09:00
    timezone: America/New_York
```

```bash
gcloud app deploy cron.yaml
gcloud app describe --service=cron  # View cron config
```

---

## Dispatch Rules (URL Routing Across Services)

Create a `dispatch.yaml` to route URLs to different services:
```yaml
dispatch:
  - url: "*/api/*"
    service: api
  - url: "*/admin/*"
    service: admin
  - url: "*/*"
    service: default
```

```bash
gcloud app deploy dispatch.yaml
```

---

## App Engine vs Cloud Run vs GKE

| Feature | App Engine | Cloud Run | GKE |
|---------|------------|-----------|-----|
| Container required | ❌ (Standard) / ✅ (Flex) | ✅ | ✅ |
| Scale to zero | ✅ (Standard) | ✅ | ❌ |
| Traffic splitting | ✅ Built-in | ✅ Built-in | Via Istio/Gateway |
| Cron jobs | ✅ Built-in (`cron.yaml`) | Via Cloud Scheduler | Via CronJob |
| Complexity | Low | Low | High |
| Best for | Traditional web apps, APIs with variable traffic | Stateless containers | Complex microservices needing full K8s |