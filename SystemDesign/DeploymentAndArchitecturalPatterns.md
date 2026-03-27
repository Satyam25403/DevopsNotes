# 🚀 Deployment & Architectural Patterns

> **Series:** System Design Notes  
> **Module:** 13 — Deployment & Architectural Patterns  
> **Prerequisites:** `11_system_design_patterns.md`, `12_security.md`, `10_observability.md`  
> **Already covered:** Blue-Green, Canary, Shadow deployments → see `11_system_design_patterns.md §14`

---

## 📌 Table of Contents

**Part A — Deployment Patterns**
1. [CI/CD Pipeline Architecture](#1-cicd-pipeline-architecture)
2. [GitOps](#2-gitops)
3. [Feature Flags](#3-feature-flags)
4. [Infrastructure as Code (IaC)](#4-infrastructure-as-code-iac)
5. [Container Orchestration (Kubernetes)](#5-container-orchestration-kubernetes)
6. [Rollback Strategies](#6-rollback-strategies)
7. [Database Migrations in CI/CD](#7-database-migrations-in-cicd)

**Part B — Architectural Patterns**
8. [Monolith vs Microservices vs Modular Monolith](#8-monolith-vs-microservices-vs-modular-monolith)
9. [Hexagonal Architecture (Ports & Adapters)](#9-hexagonal-architecture-ports--adapters)
10. [Event-Driven Architecture (EDA)](#10-event-driven-architecture-eda)
11. [Cell-Based Architecture](#11-cell-based-architecture)
12. [Multi-Region Architecture](#12-multi-region-architecture)
13. [Serverless Architecture](#13-serverless-architecture)
14. [Pattern Decision Matrix](#14-pattern-decision-matrix)
15. [Interview Cheatsheet](#15-interview-cheatsheet)

---

# PART A — DEPLOYMENT PATTERNS

---

## 1. CI/CD Pipeline Architecture

> **CI (Continuous Integration):** Automatically build, test, and validate every code change as soon as it's pushed.  
> **CD (Continuous Delivery):** Automatically deliver validated code to a staging environment — one click to production.  
> **CD (Continuous Deployment):** Automatically deploy to production without human approval.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     COMPLETE CI/CD PIPELINE                             │
│                                                                         │
│  Developer pushes code                                                  │
│       ↓                                                                 │
│  ┌─────────────── CONTINUOUS INTEGRATION ───────────────────┐          │
│  │  Pre-commit hooks:  lint, secret scan, format check      │          │
│  │       ↓                                                   │          │
│  │  Build:             docker build, compile, package        │          │
│  │       ↓                                                   │          │
│  │  Unit Tests:        fast, isolated, < 5 minutes          │          │
│  │       ↓                                                   │          │
│  │  SAST + Dep scan:   security checks (Semgrep, Snyk)      │          │
│  │       ↓                                                   │          │
│  │  Integration Tests: DB, cache, queue interactions        │          │
│  │       ↓                                                   │          │
│  │  Container Scan:    Trivy scans image for CVEs           │          │
│  │       ↓                                                   │          │
│  │  Push artifact:     tagged image to registry (ECR)       │          │
│  └───────────────────────────────────────────────────────────┘          │
│       ↓                                                                 │
│  ┌─────────────── CONTINUOUS DELIVERY ──────────────────────┐          │
│  │  Deploy to Staging:  update K8s deployment               │          │
│  │       ↓                                                   │          │
│  │  E2E Tests:          Cypress / Playwright against staging │          │
│  │       ↓                                                   │          │
│  │  Performance tests:  k6 / Locust — check no regression   │          │
│  │       ↓                                                   │          │
│  │  DAST:               OWASP ZAP against staging           │          │
│  │       ↓                                                   │          │
│  │  Manual approval gate (for CD → Continuous Delivery)     │          │
│  └───────────────────────────────────────────────────────────┘          │
│       ↓                                                                 │
│  ┌─────────────── CONTINUOUS DEPLOYMENT ────────────────────┐          │
│  │  Deploy to Production (canary → full rollout)            │          │
│  │       ↓                                                   │          │
│  │  Smoke tests against production                          │          │
│  │       ↓                                                   │          │
│  │  Monitor golden signals (latency, errors, saturation)    │          │
│  │       ↓                                                   │          │
│  │  Auto-rollback if error rate spikes                      │          │
│  └───────────────────────────────────────────────────────────┘          │
└─────────────────────────────────────────────────────────────────────────┘
```

### Pipeline as Code (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main]

jobs:
  build-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run unit tests
        run: |
          docker compose -f docker-compose.test.yml up --exit-code-from tests

      - name: SAST scan
        uses: semgrep/semgrep-action@v1

      - name: Build image
        run: |
          docker build -t $ECR_REGISTRY/payment-service:$GITHUB_SHA .

      - name: Scan image
        run: |
          trivy image --exit-code 1 --severity CRITICAL \
            $ECR_REGISTRY/payment-service:$GITHUB_SHA

      - name: Push image
        run: |
          docker push $ECR_REGISTRY/payment-service:$GITHUB_SHA

  deploy-staging:
    needs: build-test
    steps:
      - name: Deploy to staging
        run: |
          kubectl set image deployment/payment-service \
            payment-service=$ECR_REGISTRY/payment-service:$GITHUB_SHA \
            --namespace staging

      - name: Run E2E tests
        run: npx playwright test --project=staging

  deploy-production:
    needs: deploy-staging
    environment: production     # requires manual approval in GitHub
    steps:
      - name: Canary deploy (10%)
        run: |
          kubectl apply -f k8s/payment-canary.yaml
          # route 10% traffic to canary

      - name: Monitor canary for 10 min
        run: |
          ./scripts/wait-and-check-error-rate.sh 10m 1%

      - name: Full rollout
        run: |
          kubectl set image deployment/payment-service \
            payment-service=$ECR_REGISTRY/payment-service:$GITHUB_SHA
```

---

## 2. GitOps

> **GitOps** is a deployment model where **Git is the single source of truth** for both application code AND infrastructure/deployment configuration. All changes to production happen via Git commits — never via manual `kubectl apply` or AWS console clicks.

```
TRADITIONAL DEPLOY (push-based):
  Developer → runs kubectl apply / helm upgrade → directly changes cluster
  Problems:
    - No audit trail of who changed what
    - Configuration drift (cluster state ≠ what's in Git)
    - Manual, error-prone

GITOPS (pull-based):
  Developer → opens PR → merges to main → Git repo updated
  GitOps agent (ArgoCD/Flux) watches Git repo
  Agent detects diff between Git state and cluster state
  Agent pulls and applies changes to cluster
  
  Git history = complete audit log of all changes ✅
  Cluster always converges to what Git says ✅
  Rollback = revert the Git commit ✅
```

### ArgoCD Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         GITOPS WITH ARGOCD                           │
│                                                                      │
│  Developer                                                           │
│    │ git push                                                         │
│    ↓                                                                 │
│  Git Repo (GitHub/GitLab)                                            │
│    │ contains: k8s manifests / Helm charts / Kustomize overlays      │
│    │                                                                  │
│    │ ArgoCD watches repo (poll every 3s or webhook)                   │
│    ↓                                                                 │
│  ArgoCD Controller (running in cluster)                              │
│    │ compares: desired state (Git) vs actual state (cluster)         │
│    │ detects drift → syncs automatically (or manual approve)         │
│    ↓                                                                 │
│  Kubernetes Cluster                                                  │
│    └─ always converges to what Git says                              │
│                                                                      │
│  ArgoCD UI/CLI:                                                      │
│    - Visual diff of what changed                                     │
│    - Sync status per app                                             │
│    - One-click rollback to any previous commit                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Benefits of GitOps

| Benefit | Detail |
|---|---|
| **Audit trail** | Every change is a git commit — who, what, when, why (PR description) |
| **Rollback** | `git revert` → cluster converges back to previous state |
| **Drift detection** | ArgoCD alerts when cluster diverges from Git (someone ran kubectl manually) |
| **Multi-env promotion** | PR from `staging` branch → `production` branch = environment promotion |
| **Disaster recovery** | New cluster? Point ArgoCD at Git repo → everything restored automatically |

---

## 3. Feature Flags

> **Feature flags** (also: feature toggles) let you deploy code to production but control who sees the new feature — independent of deployment. Decouple deploy from release.

```
WITHOUT FEATURE FLAGS:
  Feature complete → merge → deploy → feature live for everyone
  Risk: if feature is buggy → affects all users → full rollback needed

WITH FEATURE FLAGS:
  Feature complete → merge → deploy → feature OFF by default
  Gradually enable:
    1% internal users → dogfood
    10% beta users    → early access
    50% random users  → A/B test
    100%              → full release
  
  If bug discovered at 10% → turn flag OFF → instant "rollback" with no redeploy ✅
```

### Flag Types

```
RELEASE FLAG (temporary):
  Controls rollout of a new feature.
  Removed from code once feature is at 100%.
  
  if feature_flag("new-checkout-flow", user_id=user.id):
      return new_checkout()
  else:
      return old_checkout()

EXPERIMENT FLAG (A/B test):
  Routes users to variant A or B.
  Measures impact on metrics.
  Winner gets 100%. Loser removed.

OPERATIONAL FLAG (circuit breaker / kill switch):
  Long-lived. Disables expensive feature under load.
  
  if feature_flag("recommendations-enabled"):
      recs = recommendations_service.get(user_id)
  else:
      recs = []  # graceful degradation

PERMISSION FLAG:
  Feature only visible to certain roles/orgs.
  "Enterprise plan" features gated behind plan tier.
```

### Targeting Rules

```
Rules evaluated in order:

1. User is in beta testers list           → return TRUE
2. User's org is "Anthropic"              → return TRUE
3. User's region is "US"                  → return TRUE for 20% (hash rollout)
4. Default                                → return FALSE

Consistent hashing ensures:
  Same user always gets same variant (no flickering)
  hash(user_id + flag_name) % 100 < rollout_percentage
```

### Tools

| Tool | Type | Notes |
|---|---|---|
| **LaunchDarkly** | Commercial | Industry leader, streaming, real-time |
| **Unleash** | Open-source | Self-hosted, Kubernetes-native |
| **AWS AppConfig** | Managed | Integrated with CloudWatch, gradual deploys |
| **Flagsmith** | Open-source / Cloud | API-first, simple |
| **Split.io** | Commercial | Analytics-focused, A/B testing built-in |

---

## 4. Infrastructure as Code (IaC)

> **IaC** means defining cloud infrastructure (servers, networks, DBs, IAM) in code files, checked into Git, applied automatically. Infrastructure changes go through the same review process as application code.

### Tools Comparison

| Tool | Scope | Language | State |
|---|---|---|---|
| **Terraform** | Multi-cloud | HCL | Remote state (S3 + DynamoDB lock) |
| **AWS CDK** | AWS-native | Python/TypeScript/Java | CloudFormation under the hood |
| **Pulumi** | Multi-cloud | Any language (Python, Go, TS) | Pulumi Cloud or self-hosted |
| **Ansible** | Config management | YAML (agentless) | No state — idempotent playbooks |
| **Helm** | Kubernetes only | YAML templates | Release state in cluster |
| **Kustomize** | Kubernetes only | YAML overlays | No state — pure overlays |

### Terraform Workflow

```bash
# Standard workflow
terraform init      # download providers
terraform plan      # show what will change (diff)
terraform apply     # apply changes (prompts for confirmation)
terraform destroy   # destroy all resources

# In CI/CD:
terraform plan  → post diff as PR comment
terraform apply → auto on merge to main (after human approval)
```

```hcl
# Example: ECS service with auto-scaling
resource "aws_ecs_service" "payment_service" {
  name            = "payment-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.payment.arn
  desired_count   = var.desired_count

  deployment_circuit_breaker {
    enable   = true
    rollback = true     # auto-rollback failed deployments ✅
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.payment.arn
    container_name   = "payment-service"
    container_port   = 8080
  }
}

resource "aws_appautoscaling_policy" "payment_scale" {
  name               = "payment-cpu-scale"
  policy_type        = "TargetTrackingScaling"
  
  target_tracking_scaling_policy_configuration {
    target_value = 70.0   # scale at 70% CPU
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
  }
}
```

---

## 5. Container Orchestration (Kubernetes)

> Kubernetes manages containerised workloads at scale — scheduling, scaling, healing, rolling updates. Essential context for all modern deployment patterns.

### Core Concepts for Deployment Patterns

```
Deployment:
  Manages N replicas of a pod.
  Rolling update: replace pods one by one (configurable max unavailable + max surge)
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0     # never go below desired count
      maxSurge: 1           # allow 1 extra pod during rollout

Readiness Probe:
  LB only sends traffic to pod AFTER readiness probe passes.
  New pod starts → not ready → LB skips it → probe passes → LB adds it.
  Prevents traffic to half-started pods.

  readinessProbe:
    httpGet:
      path: /health/ready
      port: 8080
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 3

Liveness Probe:
  Restarts pod if it becomes unresponsive.
  livenessProbe:
    httpGet:
      path: /health/live
      port: 8080
    periodSeconds: 10
    failureThreshold: 3

PodDisruptionBudget (PDB):
  Ensures minimum replicas stay up during voluntary disruptions (node drain, upgrade).
  minAvailable: 2  (always keep 2 pods running)
```

### Horizontal Pod Autoscaler (HPA)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70     # scale at 70% CPU
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"       # or scale on custom metric
```

---

## 6. Rollback Strategies

> Deploying confidently requires knowing you can undo quickly.

```
STRATEGY 1: Kubernetes Rolling Rollback
  kubectl rollout undo deployment/payment-service
  → immediately rolls back to previous ReplicaSet
  → uses Kubernetes revision history
  
  kubectl rollout history deployment/payment-service
  → lists all revisions with change-cause annotation
  
  kubectl rollout undo deployment/payment-service --to-revision=5
  → rollback to specific revision

STRATEGY 2: GitOps Rollback
  git revert HEAD~1    → creates revert commit
  git push origin main → ArgoCD detects change → rolls back cluster
  Full audit trail preserved ✅

STRATEGY 3: Blue-Green Rollback (instant)
  See 11_system_design_patterns.md §14
  Switch ALB/DNS back to Blue → instant, zero redeploy

STRATEGY 4: Feature Flag Rollback (no deploy needed)
  Turn flag OFF in LaunchDarkly → feature disabled in seconds
  No kubectl, no git, no redeploy ✅

STRATEGY 5: Database Rollback (the hard one)
  See §7 below — this is the complex case.
```

### Rollback Decision Matrix

| Situation | Fastest Rollback |
|---|---|
| Bad code, no DB change | `kubectl rollout undo` or GitOps revert |
| Bad feature, not all users | Feature flag OFF |
| Bad deploy, separate env | Blue-Green switch |
| Bad DB migration (additive) | Rollback app code, leave DB (backward compatible) |
| Bad DB migration (destructive) | Emergency: restore from snapshot (data loss risk) |

---

## 7. Database Migrations in CI/CD

> The hardest part of zero-downtime deployment. Application code and database schema must stay compatible across the rollout window when old and new code run simultaneously.

### Expand-Contract Pattern (Strangler Fig for Schema)

```
GOAL: Rename column "user_name" to "full_name" with zero downtime.

NAIVE (causes downtime):
  Step 1: ALTER TABLE users RENAME COLUMN user_name TO full_name
  Step 2: Deploy new code reading full_name
  
  Problem: Between Step 1 and Step 2 → old code is running, new column name
           → old code crashes → outage 🔴

EXPAND-CONTRACT (zero downtime):
  Phase 1 — EXPAND (backward compatible):
    ALTER TABLE users ADD COLUMN full_name VARCHAR(255);
    Deploy code that writes to BOTH user_name AND full_name
    Background job: backfill full_name from user_name
    
  Phase 2 — MIGRATE read path:
    Deploy code that reads full_name (primary), falls back to user_name
    
  Phase 3 — CONTRACT (cleanup):
    Verify full_name is populated for all rows
    Deploy code that only uses full_name
    DROP COLUMN user_name
    
  Timeline: 3 separate deploys, days/weeks apart
  Zero downtime at any point ✅
```

### Migration Tools

| Tool | Language | Notes |
|---|---|---|
| **Flyway** | Java (any via CLI) | Version-numbered SQL files, checksum validation |
| **Liquibase** | Any | XML/YAML/SQL changelogs, rollback support |
| **Alembic** | Python (SQLAlchemy) | `upgrade` / `downgrade` commands |
| **Prisma Migrate** | Node.js | Auto-generates migrations from schema diff |
| **golang-migrate** | Go | Simple, CLI + library |

```bash
# Flyway — migrations in CI pipeline
flyway -url=jdbc:postgresql://db:5432/mydb migrate

# Migration file naming:
V1__create_users_table.sql
V2__add_email_index.sql
V3__add_full_name_column.sql   ← expand phase
V4__drop_user_name_column.sql  ← contract phase (later deploy)
```

---

# PART B — ARCHITECTURAL PATTERNS

---

## 8. Monolith vs Microservices vs Modular Monolith

### Monolith

> All business logic, data access, and UI in a single deployable unit.

```
┌──────────────────────────────────┐
│          MONOLITH                │
│  ┌──────────────────────────┐   │
│  │  Order Module            │   │
│  │  Payment Module          │   │
│  │  User Module             │   │
│  │  Notification Module     │   │
│  │  Inventory Module        │   │
│  │         │                │   │
│  │  Single Database         │   │
│  └──────────────────────────┘   │
│  One build → One deploy          │
└──────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to develop early on | Scaling: scale entire app for one hot module |
| No network calls between modules (in-process) | Single point of failure |
| Easy to run locally | Deployment: any change = full deploy |
| Simple transactions (one DB) | Tech stack locked in |
| Easy to debug (one process) | Codebase grows unwieldy over time |

**Best for:** Early-stage startups, teams < 10 engineers, unknown domain boundaries

---

### Microservices

> Each business capability is a separate, independently deployable service with its own database.

```
API Gateway
     │
     ├──► Order Service    → Orders DB (PostgreSQL)
     ├──► Payment Service  → Payments DB (PostgreSQL)
     ├──► User Service     → Users DB (PostgreSQL)
     ├──► Inventory Service→ Inventory DB (PostgreSQL)
     └──► Notification Service → (stateless, uses SQS)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Scale services independently | Distributed systems complexity |
| Independent deployments | Network latency between services |
| Technology diversity per service | Distributed transactions (SAGA) |
| Isolated failures (blast radius contained) | Service discovery, LB overhead |
| Separate teams can own separate services | Local dev is complex |
| | Observability harder (need tracing) |

**Best for:** Large teams, well-understood domain, scale requirements per service differ

---

### Modular Monolith ⭐ Often the Best Choice

> Single deployable unit BUT internally structured with strict module boundaries. Each module has its own domain model, data layer, and public API. Modules communicate only via defined interfaces — no direct DB cross-access.

```
┌──────────────────────────────────────────────────────────┐
│                    MODULAR MONOLITH                      │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │ Order Module │  │Payment Module│  │  User Module  │  │
│  │   own schema │  │  own schema  │  │   own schema  │  │
│  │   own repos  │  │  own repos   │  │   own repos   │  │
│  │  public API  │  │  public API  │  │   public API  │  │
│  └──────┬───────┘  └──────┬───────┘  └───────┬───────┘  │
│         │ only via interface │ only via interface │       │
│  ┌──────────────────────────────────────────────────┐   │
│  │           Shared Database                        │   │
│  │  (but each module uses its own schema prefix)    │   │
│  └──────────────────────────────────────────────────┘   │
│                                                          │
│  Single build + deploy (for now)                         │
│  If a module needs to be extracted → already has         │
│  clean boundaries → extract to microservice easily       │
└──────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to deploy (one unit) | Still limited to one tech stack |
| Clean domain boundaries enforced by code | One DB can become bottleneck |
| Easy local dev | Still scale-together |
| Ready to extract microservices when needed | |
| Good developer experience | |

**Best for:** Growing teams, pre-product-market-fit phase, when domain isn't fully understood yet

### The Right Progression

```
Startup phase:     Monolith (move fast, no distributed systems overhead)
Growth phase:      Modular Monolith (enforce boundaries, prepare for extraction)
Scale phase:       Microservices (extract high-traffic or high-change modules)
                   via Strangler Fig (see 11_system_design_patterns.md §13)
```

---

## 9. Hexagonal Architecture (Ports & Adapters)

> **Core idea:** Business logic (the domain) knows nothing about the outside world — no HTTP, no SQL, no Kafka. External concerns connect to the domain through well-defined **ports** (interfaces). **Adapters** implement those ports for specific technologies.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    HEXAGONAL ARCHITECTURE                           │
│                                                                     │
│         DRIVING SIDE (input)    DOMAIN     DRIVEN SIDE (output)    │
│                                                                     │
│  HTTP Controller ─────► ┌──────────────────┐ ◄──── Repository      │
│  gRPC Handler  ────────►│                  │      (DB Adapter)     │
│  CLI Command   ────────►│  Business Logic  │                       │
│  Event Consumer ───────►│  (pure domain)   │◄──── Message Publisher│
│  Test (direct) ────────►│                  │      (Kafka Adapter)  │
│                         └──────────────────┘                       │
│                              │          │                           │
│                              │ uses only│                           │
│                              │interfaces│                           │
│                              ▼          ▼                           │
│                         Port: IRepository  Port: IEventPublisher    │
│                                                                     │
│  Domain doesn't import:  Flask, SQLAlchemy, boto3, pika            │
│  Domain only imports:    its own domain interfaces                  │
└─────────────────────────────────────────────────────────────────────┘
```

### Example Structure

```python
# Domain — pure business logic, no framework imports
class OrderService:
    def __init__(self, repo: IOrderRepository, events: IEventPublisher):
        self._repo = repo
        self._events = events

    def place_order(self, user_id: str, items: list) -> Order:
        order = Order(user_id=user_id, items=items)
        order.validate()                     # pure domain logic
        self._repo.save(order)               # calls interface
        self._events.publish("order.placed", order)  # calls interface
        return order

# Adapter: REST controller (driving side)
@app.post("/orders")
def create_order(body: OrderRequest, order_service: OrderService = Depends()):
    return order_service.place_order(body.user_id, body.items)

# Adapter: DB (driven side)
class PostgresOrderRepository(IOrderRepository):
    def save(self, order: Order) -> None:
        db.execute("INSERT INTO orders ...", order.to_dict())

# Adapter: Test (swap real DB with fake for unit tests)
class InMemoryOrderRepository(IOrderRepository):
    def save(self, order: Order) -> None:
        self._store[order.id] = order
```

### Benefits

| Benefit | Detail |
|---|---|
| **Testable** | Unit test business logic with in-memory adapters (no DB needed) |
| **Swappable** | Swap PostgreSQL for DynamoDB — only adapter changes, domain untouched |
| **Framework-agnostic** | Switch from Flask to FastAPI — only HTTP adapter changes |
| **Isolated domain** | Business rules never contaminated by infra concerns |

**Best for:** Long-lived services, complex domain logic, teams that prioritize testability

---

## 10. Event-Driven Architecture (EDA)

> Already covered deeply in `07_message_queues.md §12`. Key architectural summary here.

```
REQUEST-DRIVEN (synchronous):
  Service A ──HTTP──► Service B ──HTTP──► Service C
  Tight coupling. B must be up. C must be up.

EVENT-DRIVEN (asynchronous):
  Service A ──event──► [Broker] ──event──► Service B
                                ──event──► Service C
  
  A doesn't know B or C exist.
  B and C can fail and recover independently.
  New services subscribe without changing A.
```

### When EDA is the Right Architecture

```
✅ USE EDA when:
  - Multiple services react to the same business event
  - Processing can be async (notifications, analytics, recommendations)
  - Need to decouple services for independent scaling
  - Need an audit trail / event log (Event Sourcing)
  - Handling traffic spikes via queue buffer

❌ DON'T USE EDA when:
  - You need an immediate response (user login, payment confirmation)
  - The system is simple — over-engineering risk
  - Team is small — operational overhead not worth it
  - Debugging/tracing is already stretched thin
```

---

## 11. Cell-Based Architecture

> **Cell-based architecture** partitions the system into small, independent, fully-functional cells. Each cell is a complete replica of the system serving a subset of users. Cells share no state with each other.

```
WITHOUT CELLS:
  All users → one global system → one bug → everyone affected

WITH CELLS (AWS Availability Zone model, Slack cell model):
  User group A → Cell 1 (full stack: API + DB + cache + queue)
  User group B → Cell 2 (full stack: API + DB + cache + queue)
  User group C → Cell 3 (full stack: API + DB + cache + queue)
  
  Cell 2 has an outage → only User group B affected → other cells unaffected
  Blast radius: 1/N of users ✅
```

```
Cell topology:
  ┌──────────────────────────────────────────────┐
  │                  CELL 1                      │
  │   ┌──────────┐  ┌──────────┐  ┌──────────┐  │
  │   │ API Tier │  │  Cache   │  │ DB (RDS) │  │
  │   └──────────┘  └──────────┘  └──────────┘  │
  │   ┌──────────┐  ┌──────────┐                 │
  │   │  Queue   │  │ Workers  │                 │
  │   └──────────┘  └──────────┘                 │
  │   Serves: users with ID hash % N == 0         │
  └──────────────────────────────────────────────┘
  
  ┌──────────────────────────────────────────────┐
  │                  CELL 2                      │
  │   (identical stack, different user partition) │
  └──────────────────────────────────────────────┘
```

### Cell Routing

```
Global Router (stateless):
  hash(user_id) % num_cells = cell_index
  → route to cell

Cell Assignment DB (lightweight):
  user_id → cell_id  (stored at edge, replicated globally)
  
  On cell expansion: reassign users, migrate data
```

**Used by:** AWS (AZ model), Slack (1000s of cells per datacenter), DoorDash, Shopify

---

## 12. Multi-Region Architecture

> Deploy to multiple geographic regions to achieve low latency globally and survive regional failures.

### Active-Passive (Single Active Region)

```
   Primary Region (us-east-1)          Standby Region (eu-west-1)
   ┌─────────────────────────┐         ┌─────────────────────────┐
   │  API → DB (read+write)  │──sync──►│  DB replica (read only) │
   └─────────────────────────┘         └─────────────────────────┘
   
   All traffic → us-east-1
   If us-east-1 fails → promote eu-west-1 replica to primary → DNS failover
   
   RTO: 5–30 minutes (time to promote + DNS propagation)
   RPO: seconds (replication lag)
```

### Active-Active (Multiple Active Regions)

```
   Region: us-east-1                   Region: ap-south-1
   ┌─────────────────────────┐         ┌─────────────────────────┐
   │  API → DB (read+write)  │◄──sync──►│  DB (read+write)        │
   └─────────────────────────┘         └─────────────────────────┘
         ↑                                       ↑
   US users routed here              India users routed here
   
   Global LB (Route53 latency-based / Cloudflare) routes to nearest
   Both regions are primary — no failover wait time
   
   Challenge: concurrent writes to same record → conflict resolution needed
   Solutions: CRDTs, vector clocks, last-write-wins, ownership sharding
```

### Data Residency Considerations

```
GDPR, HIPAA, data sovereignty laws require data to stay in specific regions.

Pattern: Data Sovereignty Sharding
  EU users → data stored ONLY in eu-west-1 → compliant with GDPR ✅
  US users → data stored ONLY in us-east-1
  IN users → data stored ONLY in ap-south-1
  
  Global identity service knows which region a user's data lives in
  API Gateway routes to correct region
  Cross-region data transfers prohibited by policy
```

---

## 13. Serverless Architecture

> Serverless doesn't mean "no servers" — it means you don't manage servers. Cloud provider handles provisioning, scaling, patching. You deploy functions or containers. Pay per invocation/execution time, not per server-hour.

### When Serverless is Right

```
✅ GOOD FIT:
  - Irregular, bursty workloads (event-driven processing)
  - Short-lived functions (< 15 min for Lambda)
  - API backends with variable traffic
  - Data processing pipelines (S3 trigger → Lambda → process)
  - Scheduled jobs (cron)
  - Cost-sensitive: you pay only when code runs

❌ BAD FIT:
  - Long-running processes
  - Low-latency requirements (cold start = 100ms–3s)
  - Stateful workloads (Lambda is stateless)
  - Predictable, high-constant-throughput (containers cheaper)
  - Complex dependencies (package size limits)
```

### Cold Start Problem

```
Cold start: first invocation (or after idle period) → runtime must initialise
  Node.js Lambda: ~100ms cold start
  Python Lambda:  ~200ms cold start
  Java Lambda:    ~2000ms cold start (JVM startup!)
  
Mitigations:
  1. Provisioned Concurrency: keep N instances warm (costs money)
  2. Smaller packages: less code = faster init
  3. Use Lambda SnapStart (Java): snapshot after init, restore from snapshot (~90% faster)
  4. Choose runtime: Node.js/Python/Go > Java/C# for cold start
  5. Avoid: huge package sizes, global DB connections at module level
```

### Serverless Patterns

```
API Gateway + Lambda (most common):
  Client → API Gateway → Lambda → DynamoDB

  Lambda function per endpoint or per service.
  Scales to 0 (no traffic = no cost).
  Scales to 1000s of concurrent invocations.

Event Processing Pipeline:
  S3 PUT event → Lambda (transform) → DynamoDB or SQS

  Image uploaded to S3 → Lambda generates thumbnails → store back to S3

Scheduled (cron):
  EventBridge rule (cron schedule) → Lambda (cleanup job)

  Serverless equivalent of cron jobs.
```

---

## 14. Pattern Decision Matrix

### Deployment Pattern Selection

| Need | Pattern |
|---|---|
| Consistent deployment process | CI/CD Pipeline |
| Git as single source of truth | GitOps (ArgoCD/Flux) |
| Decouple deploy from release | Feature Flags |
| Zero-downtime deploy | Rolling update / Blue-Green / Canary |
| Instant rollback | Feature flag OFF or Blue-Green switch |
| Infrastructure repeatability | IaC (Terraform/CDK) |
| Gradual rollout with metrics | Canary deployment |
| Safe schema changes | Expand-Contract pattern |

### Architectural Pattern Selection

| Scenario | Pattern |
|---|---|
| Early-stage startup, small team | Monolith |
| Growing team, domain not fully understood | Modular Monolith |
| Large team, well-defined domain | Microservices |
| Isolating blast radius per user group | Cell-Based |
| Global low-latency + HA | Multi-Region Active-Active |
| Complex domain logic, high testability | Hexagonal (Ports & Adapters) |
| Multiple services react to business events | Event-Driven (Kafka/SNS+SQS) |
| Irregular, event-driven workloads | Serverless (Lambda) |
| Migrating from monolith | Strangler Fig + Modular Monolith bridge |

---

## 15. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **CI/CD** | Automated build-test-deploy pipeline triggered by code push |
| **GitOps** | Git is the single source of truth; agent pulls and applies changes |
| **Feature Flag** | Toggle features on/off independently of deployment |
| **IaC** | Infrastructure defined in code; versioned in Git |
| **Canary** | Route % of traffic to new version; roll back on signal *(see module 11)* |
| **Blue-Green** | Two environments; atomic traffic switch *(see module 11)* |
| **Expand-Contract** | Zero-downtime DB schema migration via additive intermediate steps |
| **Modular Monolith** | Single deploy, strict internal module boundaries |
| **Hexagonal** | Domain isolated from infrastructure via ports and adapters |
| **Cell-Based** | Independent full-stack replicas serving partitioned user groups |
| **Multi-Region** | Deploy in multiple geos for latency and HA |
| **Serverless** | Cloud manages servers; pay per execution; event-driven |
| **Cold Start** | First Lambda invocation latency due to runtime initialisation |
| **HPA** | Kubernetes Horizontal Pod Autoscaler — scales pods on metrics |

### Must-Know Interview Points

- ☑ **CI/CD pipeline = build → test → security scan → deploy → monitor.** Know each stage.
- ☑ **GitOps** = Git is the source of truth. ArgoCD pulls from Git → cluster converges.
- ☑ **Feature flags** decouple deploy from release. Critical for dark launches, A/B, kill switches.
- ☑ **Modular Monolith** is often better than microservices for growing teams. Know why.
- ☑ **Expand-Contract** is the answer to "how do you do zero-downtime DB migrations?"
- ☑ **Blue-Green and Canary** already covered in `11_system_design_patterns.md` — don't repeat, reference.
- ☑ **Cell-Based architecture** = blast radius isolation by user partition (AWS AZs, Slack).
- ☑ **Hexagonal** = domain knows nothing about HTTP/DB — ports and adapters mediate.
- ☑ **Cold start** = serverless tradeoff. Mitigate with provisioned concurrency or smaller packages.
- ☑ **IaC = infrastructure in Git** — same PR review, rollback, audit trail as code.

### The Interview Answer Template

```
When asked "How would you deploy a new feature safely?":

"I'd use a combination of:
  1. CI/CD pipeline — automated build, test, security scan on every commit.
  2. Canary deploy — route 5% of traffic to new version first,
     monitor golden signals for 10 minutes.
  3. Feature flag — new feature behind a flag so I can kill it
     without a redeploy if metrics degrade.
  4. Automated rollback — if error rate exceeds threshold,
     kubectl rollout undo triggers automatically.
  5. GitOps — all config in Git; rollback = git revert."

When asked "Monolith or Microservices?":

"It depends on the team size and domain maturity.
  Early stage (<10 engineers, undefined domain) → Modular Monolith.
  It gives you clean boundaries without distributed systems overhead.
  As the team scales and domain stabilises, use Strangler Fig to
  extract the most high-change or high-traffic modules into services.
  Don't go full microservices day one — the operational overhead is real."
```

---

*Sources: GitOps Working Group (OpenGitOps Principles), ArgoCD Documentation, LaunchDarkly Feature Flag Taxonomy, Terraform Documentation, AWS re:Invent (Cell-Based Architecture — Slack, DoorDash), AWS Multi-Region Architecture Patterns, Serverless Land (AWS Lambda Patterns), DORA (DevOps Research — Deployment Frequency, Change Failure Rate), Google SRE Book (Canarying, Progressive Delivery), Kubernetes Documentation (HPA, PDB, Rolling Updates), Hexagonal Architecture (Alistair Cockburn Original), Martin Fowler (Modular Monolith, Strangler Fig, Feature Toggles) — combined with first-principles system design knowledge.*