# Deployment Strategies

> **Series:** System Design Notes
> **Module:** 05 — Deployment Strategies
> **Prerequisites:** load_balancing, CDN, Basic DNS, containerization basics

---

## 📌 Table of Contents

1. [What is a Deployment Strategy?](#1-what-is-a-deployment-strategy)
2. [Core Terminology](#2-core-terminology)
3. [The Deployment Risk Spectrum](#3-the-deployment-risk-spectrum)
4. [Big Bang / Recreate Deployment](#4-big-bang--recreate-deployment)
5. [Rolling Deployment](#5-rolling-deployment)
6. [Blue-Green Deployment](#6-blue-green-deployment)
7. [Canary Deployment](#7-canary-deployment)
8. [A/B Deployment](#8-ab-deployment)
9. [Shadow Deployment](#9-shadow-deployment)
10. [Feature Flags (Feature Toggles)](#10-feature-flags-feature-toggles)
11. [Immutable Infrastructure Deployment](#11-immutable-infrastructure-deployment)
12. [GitOps Deployment](#12-gitops-deployment)
13. [Multi-Region Deployment](#13-multi-region-deployment)
14. [Database Deployment Strategies](#14-database-deployment-strategies)
15. [Strategy Comparison Matrix](#15-strategy-comparison-matrix)
16. [Real-World Architectures](#16-real-world-architectures)
17. [Common Mistakes](#17-common-mistakes)
18. [Interview Cheatsheet](#18-interview-cheatsheet)

---

## 1. What is a Deployment Strategy?

> **Definition:** A deployment strategy is the method by which new software versions are released to production — controlling how traffic shifts from old to new code, how rollback is handled, and how risk is distributed across users and time.

The key insight: **how you deploy is as important as what you deploy. The same buggy release can be catastrophic or containable depending entirely on your strategy.**

```
BAD DEPLOYMENT (no strategy):
  Ship everything → replace all servers → hope it works
  Bug found → all users affected → manual rollback → 2h downtime

GOOD DEPLOYMENT (canary):
  Ship to 1% of users → monitor error rates → auto-promote or auto-rollback
  Bug found → 1% of users affected → rollback in 30 seconds → no downtime
```

> ⚠️ **Common misconception:** Many teams treat deployment strategy as a DevOps concern separate from system design. In reality, your architecture dictates which strategies are even possible — stateless services, load balancer control, feature flag infrastructure, and database migration patterns are all design decisions that determine your deployment options.

---

## 2. Core Terminology

| Term | Definition |
|---|---|
| **Deployment** | The act of releasing a new version of software to an environment |
| **Release** | Making a feature available to users (can be decoupled from deployment via feature flags) |
| **Rollback** | Reverting to a previous known-good version |
| **Rollforward** | Fixing a bad deployment by shipping a corrected version forward instead of reverting |
| **Zero-downtime deployment** | Deploying without any interruption to users |
| **Blast radius** | The fraction of users/systems affected if a deployment goes wrong |
| **Traffic shifting** | Routing a percentage of requests to a new version |
| **Health check** | Automated test confirming a new instance is ready to serve traffic |
| **Readiness probe** | Kubernetes term — checks if a pod should receive traffic |
| **Liveness probe** | Kubernetes term — checks if a pod is alive and should be restarted |
| **SLO (Service Level Objective)** | Target reliability metric (e.g. 99.9% uptime, p99 latency < 200ms) |
| **Error budget** | How much unreliability you can burn before violating SLO |
| **Idempotency** | Operations that produce the same result whether run once or many times — critical for safe deploys |
| **Dark launch** | Deploying code that runs but produces no user-visible output yet |
| **Flighting** | Microsoft term for progressive exposure — same concept as canary |
| **Bake time** | The period you observe a new deployment before fully promoting it |

---

## 3. The Deployment Risk Spectrum

```
HIGH RISK ◄────────────────────────────────────────────► LOW RISK

Big Bang    Rolling    Blue-Green    Canary    Shadow    Feature Flags
  │            │           │           │          │            │
  │            │           │           │          │            │
All users    Sequential  Instant     Small %    No users    Any %
at once      batch swap  cutover     first      exposed     anytime
  │            │           │           │          │            │
Rollback:   Rollback:   Rollback:  Rollback:  Rollback:   Toggle
  Hard        Medium      Instant    Instant    N/A         off
```

**The central tradeoff in every deployment strategy:**

```
Speed of delivery  ←────────────────────────────────►  Safety of delivery
(ship fast,                                             (small blast radius,
 big bang)                                               easy rollback)
```

---

## 4. Big Bang / Recreate Deployment

> **Definition:** Stop the old version completely. Deploy the new version. Restart. All users experience the transition simultaneously.

```
TIMELINE:
  t=0:   Old version v1 serving 100% traffic
  t=1:   Shutdown all v1 instances  ← DOWNTIME BEGINS
  t=2:   Deploy v2 instances
  t=3:   v2 health checks pass
  t=4:   v2 serving 100% traffic    ← DOWNTIME ENDS

DOWNTIME WINDOW: t=1 to t=4 (minutes to hours depending on startup time)
```

```
┌─────────────────────────────────────────────────────────┐
│                  RECREATE DEPLOYMENT                    │
│                                                         │
│  Phase 1:  [v1][v1][v1][v1]  ← all v1                 │
│               ↓                                         │
│  Phase 2:  [ down ][ down ]  ← downtime                │
│               ↓                                         │
│  Phase 3:  [v2][v2][v2][v2]  ← all v2                 │
└─────────────────────────────────────────────────────────┘
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simplest possible implementation | Guaranteed downtime |
| No compatibility concerns between v1/v2 | All users affected if v2 has bugs |
| Clean state — no mixed version traffic | Rollback is another full redeploy |
| Works when v1 and v2 are incompatible | Not acceptable for production SLOs |

**When to use:**
- Development / staging environments only
- Batch processing systems where downtime is scheduled
- Internal tools where planned maintenance is acceptable
- When v1 and v2 are fundamentally incompatible (DB schema breaking changes)

---

## 5. Rolling Deployment

> **Definition:** Replace old instances with new ones gradually, one batch at a time. At no point is the system fully down — old and new versions run simultaneously during the transition.

```
TIMELINE (4 instances, batch size 1):

  t=0:  [v1][v1][v1][v1]   100% v1
  t=1:  [v2][v1][v1][v1]   25% v2  ← replace instance 1
  t=2:  [v2][v2][v1][v1]   50% v2  ← replace instance 2
  t=3:  [v2][v2][v2][v1]   75% v2  ← replace instance 3
  t=4:  [v2][v2][v2][v2]   100% v2 ← replace instance 4
```

```
┌─────────────────────────────────────────────────────────┐
│                   ROLLING DEPLOYMENT                    │
│                                                         │
│  LB → [v1][v1][v1][v1]                                 │
│     ↓                                                   │
│  LB → [v2][v1][v1][v1]  (health check passes → next)  │
│     ↓                                                   │
│  LB → [v2][v2][v1][v1]                                 │
│     ↓                                                   │
│  LB → [v2][v2][v2][v1]                                 │
│     ↓                                                   │
│  LB → [v2][v2][v2][v2]                                 │
└─────────────────────────────────────────────────────────┘
```

### Rolling Deployment Parameters

```
Batch size:     How many instances to replace at once
                Small batch  → safer, slower
                Large batch  → faster, more blast radius per step

Max unavailable: How many instances can be down simultaneously
                 (Kubernetes default: 25%)

Max surge:       How many extra instances to spin up during rollout
                 (run 5 instances temporarily while replacing 4)
                 Costs more but maintains full capacity throughout
```

### The Mixed Version Problem

```
During rolling deploy, v1 and v2 serve traffic simultaneously.
This breaks when:

  v1 writes data in format A
  v2 reads data expecting format B
  → Runtime errors for users who hit v1 then v2

Requirements for safe rolling deploys:
  ✅ API must be backwards compatible (v2 accepts v1 requests)
  ✅ Data formats must be backwards compatible
  ✅ Database schema must support both versions simultaneously
  ✅ No session affinity assumptions (user might hit v1 or v2)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Zero downtime | Mixed version traffic during transition |
| No extra infrastructure needed | Harder to rollback (must roll forward through all instances) |
| Gradual exposure | Both versions must be compatible simultaneously |
| Kubernetes native (`RollingUpdate` strategy) | Slow for large fleets |

**Used by:** Kubernetes default deploy strategy, AWS ECS rolling update, Heroku

---

## 6. Blue-Green Deployment

> **Definition:** Maintain two identical production environments — Blue (current live) and Green (new version). Switch all traffic from Blue to Green atomically at the load balancer or DNS level. Rollback is instant — switch back.

```
BEFORE DEPLOYMENT:
  Users → LB → [Blue: v1][Blue: v1][Blue: v1]
                [Green: idle]

DEPLOY GREEN:
  Users → LB → [Blue: v1][Blue: v1]   ← still live
                [Green: v2][Green: v2] ← deploy & test here

SWITCH TRAFFIC:
  Users → LB → [Green: v2][Green: v2]  ← instant cutover
                [Blue: v1][Blue: v1]    ← standby (keep for rollback)

ROLLBACK (if needed):
  Users → LB → [Blue: v1][Blue: v1]    ← instant, one config change
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN DEPLOYMENT                        │
│                                                                 │
│         ┌─────────────────────────────────────┐                │
│         │           Load Balancer             │                │
│         └──────┬──────────────────────┬───────┘                │
│                │ 100%                  │ 0%                     │
│         ┌──────▼──────┐        ┌──────▼──────┐                │
│         │  BLUE (v1)  │        │ GREEN (v2)  │                │
│         │  LIVE       │        │  STANDBY    │                │
│         └─────────────┘        └─────────────┘                │
│                                                                 │
│         After switch:                                           │
│                │ 0%                    │ 100%                   │
│         ┌──────▼──────┐        ┌──────▼──────┐                │
│         │  BLUE (v1)  │        │ GREEN (v2)  │                │
│         │  STANDBY    │        │  LIVE       │                │
│         └─────────────┘        └─────────────┘                │
└─────────────────────────────────────────────────────────────────┘
```

### Traffic Switch Mechanisms

```
Option 1: Load Balancer weight change
  → Instant (milliseconds)
  → Best: ALB target group swap, NGINX upstream switch

Option 2: DNS cutover
  → Takes minutes to hours (DNS propagation + TTL)
  → Use low TTL (60s) before switch to minimize lag
  → Worst for instant rollback

Option 3: Feature flag / service mesh
  → Instant, granular, no infra change
  → Best for microservices
```

### Blue-Green Database Problem

```
The hardest part of blue-green is the database.

WRONG approach:
  Blue and Green each have their own DB
  → Data written to Blue's DB during switch is lost
  → Never do this for stateful services

CORRECT approach:
  Blue and Green SHARE the same database
  → DB schema must be compatible with BOTH v1 and v2 simultaneously
  → Requires Expand-Contract pattern (see Section 14)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Instant, atomic traffic switch | Double the infrastructure cost |
| Instant rollback (switch back) | DB schema compatibility required |
| Green is fully tested before going live | Warming up caches on Green costs time |
| Zero mixed-version traffic | Not practical for very large fleets |
| Clean environment for each release | |

**When to use:** Systems where rollback speed is critical, compliance environments, stateless services

**Used by:** AWS CodeDeploy, Spinnaker, most enterprise CI/CD pipelines

---

## 7. Canary Deployment

> **Definition:** Route a small percentage of real production traffic to the new version while the majority stays on old. Monitor error rates, latency, and business metrics. Gradually increase traffic to new version if healthy; roll back instantly if not.

The name comes from "canary in a coal mine" — the canary signals danger before it reaches everyone.

```
TIMELINE:
  t=0:   v1=100%  v2=0%     ← baseline
  t=1:   v1=99%   v2=1%     ← canary phase (watch metrics)
  t=2:   v1=90%   v2=10%    ← expand if healthy
  t=3:   v1=50%   v2=50%    ← halfway
  t=4:   v1=0%    v2=100%   ← complete

  OR:
  t=1:   v1=99%   v2=1%     ← metrics degraded → ROLLBACK INSTANTLY
         v1=100%  v2=0%
```

```
┌──────────────────────────────────────────────────────────────────┐
│                      CANARY DEPLOYMENT                          │
│                                                                  │
│  Users ─────────────► Load Balancer                             │
│                             │                                    │
│                   ┌─────────┴──────────┐                        │
│                   │ 99%                │ 1%                     │
│              ┌────▼────┐         ┌─────▼────┐                  │
│              │  v1     │         │  v2      │ ← canary          │
│              │  stable │         │  new     │                   │
│              └─────────┘         └─────┬────┘                  │
│                                        │                        │
│                                  Metrics collected:             │
│                                  - Error rate                   │
│                                  - p99 latency                  │
│                                  - Business metrics             │
│                                  → Auto-promote or rollback     │
└──────────────────────────────────────────────────────────────────┘
```

### Canary Analysis — Automated Gate

```
For each canary stage, an automated system compares:

  v2 error rate       vs  v1 error rate      → must be within threshold
  v2 p99 latency      vs  v1 p99 latency     → must be within threshold
  v2 conversion rate  vs  v1 conversion rate → must not degrade

Tools: Spinnaker Canary Analysis, Argo Rollouts, Flagger

Decision:
  PASS  → automatically promote to next stage (1% → 5% → 25% → 100%)
  FAIL  → automatically rollback (traffic returns to v1 in seconds)
  PAUSE → human review required
```

### Canary vs Rolling

```
Rolling deploy: replaces instances — traffic distribution follows instance count
Canary deploy:  routes by percentage — instances may be unequal, traffic is explicit

Rolling: you control WHICH instances run v2
Canary:  you control HOW MUCH TRAFFIC hits v2
```

| ✅ Pros | ❌ Cons |
|---|---|
| Tiny blast radius on failure | Some real users hit the buggy version |
| Instant rollback | Requires sophisticated traffic splitting (service mesh or smart LB) |
| Real production data validates v2 | Mixed versions complicate debugging |
| Automated promotion gates | Metrics analysis adds pipeline complexity |
| Error budget is preserved | |

**When to use:** Any change with meaningful risk, all production services at scale

**Used by:** Google (all deploys), Netflix, Facebook, Uber, Amazon

---

## 8. A/B Deployment

> **Definition:** Route different user segments to different versions based on deterministic attributes (user ID, geography, device, account tier) rather than random percentage. Used to measure the effect of a change on a specific population.

```
CANARY:   random 1% of traffic → v2 (risk reduction)
A/B TEST: specific users → v2   (hypothesis testing)

The goal is different:
  Canary   → detect bugs safely before full rollout
  A/B test → measure business impact of a change
```

```
┌──────────────────────────────────────────────────────────────────┐
│                      A/B DEPLOYMENT                             │
│                                                                  │
│  User request arrives with attributes:                           │
│    - user_id, country, device, account_type                      │
│                                                                  │
│  Routing rules:                                                  │
│    user_id % 2 == 0  → Group A (v1 — control)                   │
│    user_id % 2 == 1  → Group B (v2 — variant)                   │
│                                                                  │
│  OR:                                                             │
│    country == "US"   → Group A                                   │
│    country == "IN"   → Group B                                   │
│                                                                  │
│  Metrics split by group:                                         │
│    Group A: conversion 3.2%, p99 latency 180ms                  │
│    Group B: conversion 4.1%, p99 latency 175ms → B wins         │
└──────────────────────────────────────────────────────────────────┘
```

### A/B vs Canary Key Differences

| Dimension | Canary | A/B |
|---|---|---|
| Purpose | Reduce deployment risk | Measure business impact |
| User assignment | Random % | Deterministic attribute |
| Duration | Hours (until promoted) | Days/weeks (statistical significance) |
| Metric focus | Error rate, latency | Conversion, engagement, revenue |
| Success condition | No degradation | Statistically significant improvement |
| Infrastructure | Traffic splitting | Traffic splitting + analytics pipeline |

### Statistical Significance Requirement

```
A/B test is NOT done when you "feel like" the numbers look good.

Required: p-value < 0.05 (95% confidence)
  → You need enough traffic to reach significance
  → Small sites may need weeks for a meaningful A/B test

Tools: Optimizely, LaunchDarkly, Statsig, Split.io, homegrown
```

| ✅ Pros | ❌ Cons |
|---|---|
| Data-driven feature decisions | Requires analytics infrastructure |
| Controlled user segments | Statistical analysis complexity |
| Can run multiple variants simultaneously | User confusion if they experience inconsistency |
| Decoupled from deployment risk | Long duration for low-traffic features |

**Used by:** Every major consumer product company. Amazon tests 1,000+ A/B experiments simultaneously.

---

## 9. Shadow Deployment

> **Definition:** Run the new version in parallel with the old, feeding it a copy of all real production traffic, but discarding its responses. Users never see v2 output. Lets you test v2 under real load with zero user impact.

```
WITHOUT SHADOW:
  Test with synthetic load → ship → real traffic reveals edge cases → incident

WITH SHADOW:
  Mirror real traffic to v2 → v2 processes all production load patterns
  → measure latency, errors, resource usage → fix issues → ship confidently
```

```
┌──────────────────────────────────────────────────────────────────┐
│                     SHADOW DEPLOYMENT                           │
│                                                                  │
│  User Request                                                    │
│       │                                                          │
│       ▼                                                          │
│  ┌─────────┐    ┌──────────────────┐                            │
│  │  Proxy  │───►│   v1 (live)      │──► Response to user        │
│  └────┬────┘    └──────────────────┘                            │
│       │                                                          │
│       │ mirror (async copy)                                      │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────┐                                            │
│  │   v2 (shadow)    │──► Response DISCARDED                     │
│  └────────┬─────────┘                                            │
│           │                                                      │
│           ▼                                                      │
│     Metrics collected:                                           │
│     - Does v2 return errors?                                     │
│     - Is v2 latency acceptable?                                  │
│     - Does v2 consume expected resources?                        │
│     - Do v2 outputs match v1? (response comparison)             │
└──────────────────────────────────────────────────────────────────┘
```

### Response Comparison (Diffy Testing)

```
Advanced shadow: compare v1 and v2 responses automatically

  v1 returns: {"price": 49.99, "currency": "USD"}
  v2 returns: {"price": 49.99, "currency": "USD"} ← matches ✅

  v1 returns: {"price": 49.99}
  v2 returns: {"price": 50.00}                    ← divergence ⚠️ bug found

Tools: Diffy (Twitter OSS), custom proxy middleware
```

### Shadow Deployment Caveats

```
CRITICAL: Side effects must be suppressed in v2.
  If your request creates a DB write, sends an email, charges a card:
    → v2 must NOT execute these side effects on shadow traffic
    → Use environment flags to suppress writes: if shadow_mode: skip_db_write()
    → This is the hardest part of shadow deployment
```

| ✅ Pros | ❌ Cons |
|---|---|
| Zero user impact | Double the infrastructure cost |
| Real production traffic patterns | Side effects must be explicitly suppressed |
| Catch performance regressions before launch | Complex infrastructure (traffic mirroring proxy) |
| Response comparison catches logic bugs | Cannot test user-facing UI changes |
| Validates capacity planning | |

**When to use:** Critical infrastructure changes, major refactors, payment systems, ML model replacements

**Used by:** Twitter (Diffy), Google, Netflix

---

## 10. Feature Flags (Feature Toggles)

> **Definition:** Decouple deployment (shipping code) from release (exposing features). Code ships to production with features disabled. Flags are flipped in a control plane to expose features to any subset of users at any time — with no redeployment.

```
WITHOUT FEATURE FLAGS:
  Code is done → deploy → feature is live → problem found → redeploy to revert

WITH FEATURE FLAGS:
  Code ships with flag OFF → flag flipped ON → feature live → problem found
  → flag flipped OFF in 5 seconds → feature hidden → no redeploy needed
```

```
Code:
  if feature_flag("new_checkout_flow", user):
      return new_checkout()
  else:
      return old_checkout()

Flag configuration (in control plane, no deploy needed):
  new_checkout_flow:
    enabled: true
    rules:
      - user.country == "US" AND user.account_type == "premium"  → 100%
      - user.id % 100 < 5                                         → 5% rollout
      - default                                                    → 0%
```

### Flag Taxonomy

```
┌───────────────────────────────────────────────────────────────────┐
│                     FEATURE FLAG TYPES                           │
├────────────────┬──────────────────────────────────────────────── │
│ Release toggle │ Ship dark, release later. Short-lived.          │
│                │ Example: new homepage — ship, enable on launch  │
├────────────────┼──────────────────────────────────────────────── │
│ Experiment     │ A/B test. Assigns users to variants.            │
│ toggle         │ Example: two checkout flows, measure conversion  │
├────────────────┼──────────────────────────────────────────────── │
│ Ops toggle     │ Kill switch for system behaviour.               │
│                │ Example: disable expensive feature under load   │
├────────────────┼──────────────────────────────────────────────── │
│ Permission     │ Controls access by user tier.                   │
│ toggle         │ Example: premium feature for paid users only    │
└────────────────┴──────────────────────────────────────────────── │
```

### Feature Flags + Deployment Strategies

```
Feature flags and deployment strategies are complementary:

  Deploy strategy  → controls WHICH CODE runs on servers
  Feature flag     → controls WHICH FEATURES users see

Combined:
  Canary deploy v2 to 10% of servers (deployment)
  Within that 10%, show new feature to 50% of users (flag)
  = 5% of total users see new feature under real load
```

### Flag Debt — The Silent Killer

```
Flags accumulate over time. Stale flags become:
  - Dead code paths that nobody tests
  - Cognitive overhead for every engineer
  - Security risk (old disabled feature paths with vulnerabilities)

Rule: Every flag gets a removal ticket on creation.
      Release flags: remove within 2 sprints of full rollout.
      Experiment flags: remove after experiment concludes.
```

| ✅ Pros | ❌ Cons |
|---|---|
| Instant rollback without redeployment | Flag debt if not cleaned up |
| Decouple deployment from release | Code complexity (every flagged feature = two paths) |
| Granular user targeting | Testing complexity (must test both flag states) |
| Ops kill switches for load shedding | Race conditions if flag state changes mid-request |
| Dark launching | Latency cost of flag evaluation (use local cache) |

**Tools:** LaunchDarkly, Unleash (open source), Statsig, Flagsmith, Split.io, AWS AppConfig, homegrown Redis-based

**Used by:** Every major tech company. Netflix calls them "feature switches." Facebook calls them "gating."

---

## 11. Immutable Infrastructure Deployment

> **Definition:** Servers are never modified after creation. Every deployment creates new server images (AMIs, container images) and replaces old instances entirely. No SSH, no in-place patches, no configuration drift.

```
MUTABLE INFRASTRUCTURE (traditional):
  Server exists → SSH in → patch → update config → restart
  Problem: servers diverge over time (snowflake servers)
           "works on my server but not that one"
           rollback = re-patch in reverse (terrifying)

IMMUTABLE INFRASTRUCTURE:
  New version → build new AMI/image → launch new instances
              → route traffic → terminate old instances
  Server is treated like a disposable artifact, not a pet
```

```
PIPELINE:
  Code commit
      ↓
  CI: build artifact (JAR, binary, wheel)
      ↓
  Packer/Dockerfile: bake artifact into AMI/image
      ↓
  Terraform/CDK: provision new instances from new image
      ↓
  LB: health check passes → shift traffic
      ↓
  Terraform: terminate old instances
  (old AMI archived for rollback — just re-provision from it)
```

### Immutable vs Mutable Comparison

```
MUTABLE:                          IMMUTABLE:
  Server lifecycle: months/years    Server lifecycle: hours/days
  Rollback: reverse the patches     Rollback: re-provision old AMI
  Config: applied post-boot         Config: baked into image
  Drift: guaranteed over time       Drift: impossible (servers replaced)
  Debugging: SSH in                 Debugging: logs + observability only
  State: servers accumulate state   State: servers are stateless
```

| ✅ Pros | ❌ Cons |
|---|---|
| No configuration drift | Slower deployments (image bake time) |
| Reliable rollback (just re-provision) | Larger image sizes = longer cold starts |
| Identical environments (dev/staging/prod) | No SSH access — observability must be excellent |
| Audit trail — every image is versioned | Stateful services are still hard |
| Security — attack surface can't persist | Image registry storage costs |

**Tools:** Packer (image baking), Terraform, Pulumi, AWS CloudFormation, Docker + Kubernetes (inherently immutable)

**Used by:** Netflix (Baked AMIs), HashiCorp toolchain ecosystem, most Kubernetes workloads

---

## 12. GitOps Deployment

> **Definition:** The entire desired state of your production system is declared in Git. A continuous reconciliation agent watches the Git repo and automatically applies any diff between desired state (Git) and actual state (cluster). Deployment = merging a PR.

```
TRADITIONAL CI/CD:
  Push code → CI builds → CI pushes to cluster (CI has cluster credentials)
  Problem: What IS deployed? You have to ask the CI/CD system.
           CI credentials = massive attack surface.

GITOPS:
  Push code → CI builds image → update image tag in Git (PR)
  PR merged → GitOps agent (ArgoCD) detects diff → applies to cluster
  Source of truth: Git repo. What's in Git IS what's deployed.
```

```
┌──────────────────────────────────────────────────────────────────┐
│                      GITOPS FLOW                                │
│                                                                  │
│  Developer                                                       │
│      │ git push feature branch                                   │
│      ↓                                                           │
│  CI Pipeline                                                     │
│      │ build image → push to registry → open PR                 │
│      │ (updates image tag in infra repo)                         │
│      ↓                                                           │
│  PR Review & Merge (to infra/config repo)                        │
│      │                                                           │
│      ↓ (GitOps agent polls or receives webhook)                  │
│  ArgoCD / Flux                                                   │
│      │ detects drift: desired (Git) ≠ actual (cluster)          │
│      │ applies manifests                                         │
│      ↓                                                           │
│  Kubernetes Cluster                                              │
│      │ new pods running ✅                                       │
│      ↓                                                           │
│  Git is the only source of truth                                 │
└──────────────────────────────────────────────────────────────────┘
```

### GitOps Rollback

```
Traditional rollback: trigger a CI job, hope it works
GitOps rollback:      git revert the PR + merge → cluster reconciles to previous state
                      Rollback = 30 seconds + a PR merge

History: every change is a commit with author, timestamp, diff
Audit:   who deployed what and when is always answerable from git log
```

| ✅ Pros | ❌ Cons |
|---|---|
| Git is complete audit trail | Secret management complexity (don't put secrets in Git) |
| Rollback is `git revert` | Repo sprawl with many services |
| No cluster credentials in CI | Drift detection lag (polling interval) |
| Self-healing — cluster always converges to Git state | Learning curve for teams new to Kubernetes |
| PRs for all production changes | Not all infra fits declarative model |

**Tools:** ArgoCD, Flux, Weave GitOps

**Used by:** Most modern Kubernetes-native teams, Intuit, Adobe, Mercedes-Benz

---

## 13. Multi-Region Deployment

> **Definition:** Running your service in multiple geographic regions simultaneously, with strategies to propagate code changes across regions safely and maintain data consistency.

```
SINGLE REGION:
  All traffic → us-east-1 → 200ms for users in Asia
  us-east-1 outage → 100% of users affected

MULTI-REGION:
  Users → nearest region → low latency globally
  us-east-1 outage → traffic fails over to us-west-2 → partial degradation only
```

### Region Rollout Strategies

```
Strategy 1: All regions simultaneously
  → Fastest rollout
  → Bug affects all regions simultaneously
  → Only for low-risk / well-tested changes

Strategy 2: Sequential region rollout (region-as-canary)
  t=0: deploy to ap-southeast-1 (smallest traffic)
  t=1: observe metrics — pass?
  t=2: deploy to eu-west-1
  t=3: observe metrics — pass?
  t=4: deploy to us-east-1 (largest traffic, deploy last)

  Benefits: us-east-1 gets the most battle-tested version
            bug found in ap-southeast-1 → never reaches us-east-1

Strategy 3: Active-Active with DNS failover
  All regions serve traffic simultaneously
  Route 53 / Cloudflare LB routes to nearest healthy region
  Failover: health check fails → DNS re-routes within 30s

Strategy 4: Active-Passive (Pilot Light)
  Primary region serves all traffic
  Secondary region has minimal infra (DB replica, no compute)
  Failover: scale up secondary, update DNS
  RTO: 5–15 minutes
  Cost: lower (secondary runs minimal infra)
```

### Multi-Region Data Challenges

```
The hardest part: data consistency across regions

Options:
  ┌─────────────────┬──────────────────────────────────────────┐
  │ Single master   │ All writes to us-east-1, replicate out   │
  │                 │ Simple. Cross-region write latency high   │
  ├─────────────────┼──────────────────────────────────────────┤
  │ Multi-master    │ Writes accepted in any region             │
  │                 │ Conflict resolution required              │
  │                 │ DynamoDB Global Tables, CockroachDB       │
  ├─────────────────┼──────────────────────────────────────────┤
  │ CRDT-based      │ Conflict-free data types — no conflicts   │
  │                 │ Limited to certain data models            │
  └─────────────────┴──────────────────────────────────────────┘
```

---

## 14. Database Deployment Strategies

> **The hardest part of any deployment.** Application code can be deployed and rolled back in seconds. Database schema changes are often irreversible and affect all versions of your code simultaneously.

### Core Constraint

```
During any deployment (except big bang), multiple app versions
run simultaneously and ALL query the SAME database.

  v1 app running on server A
  v2 app running on server B
  Both → same Postgres instance

  Schema change: add NOT NULL column
    v1 doesn't know about new column → INSERT fails → 💥
```

### Expand-Contract Pattern (Parallel Change) ⭐

> The only safe way to make breaking schema changes while maintaining zero downtime.

```
WRONG: Add NOT NULL column and deploy app that uses it simultaneously
RIGHT: Three-phase migration across multiple deploys

PHASE 1 — EXPAND (backwards compatible schema change):
  Add new column as NULLABLE (no default required, old app ignores it)
  Deploy: schema migration → both v1 and v2 compatible
  
  ALTER TABLE users ADD COLUMN phone VARCHAR(20) NULL;

PHASE 2 — MIGRATE (backfill + dual write):
  Deploy v2: writes to BOTH old and new column
  Backfill job: populate new column for all existing rows
  Verify: new column is non-null for all rows
  
  UPDATE users SET phone = '' WHERE phone IS NULL;

PHASE 3 — CONTRACT (remove old, enforce constraint):
  Deploy v3: reads only from new column
  Drop old column / add NOT NULL constraint
  Old code is gone → safe to enforce
  
  ALTER TABLE users ALTER COLUMN phone SET NOT NULL;
```

```
Timeline view:

  Deploy 1:   [schema: add nullable column]
  Deploy 2:   [app: dual write old+new]  [backfill job runs]
  Deploy 3:   [app: read new column only]  [drop old / add constraint]

  Each deploy is independently safe to roll back.
  No deploy breaks old or new code.
```

### Other Database Migration Patterns

```
Non-destructive changes (safe, no coordination needed):
  ✅ Add nullable column
  ✅ Add new table
  ✅ Add index (use CONCURRENTLY in Postgres)
  ✅ Widen a VARCHAR

Destructive changes (require expand-contract):
  ⚠️ Rename column        → add new, dual-write, drop old
  ⚠️ Change column type   → add new typed col, migrate, drop old
  ⚠️ Add NOT NULL         → add nullable, backfill, add constraint
  ⚠️ Split a table        → add new tables, dual-write, migrate reads
  ⚠️ Drop a column        → deprecate in app first, then drop
```

### Schema Migration Tools

| Tool | Language | Notes |
|---|---|---|
| Flyway | Java/JVM | SQL-based, sequential versioned migrations |
| Liquibase | Java/JVM | XML/YAML/SQL, rollback support |
| Alembic | Python | SQLAlchemy native |
| Prisma Migrate | Node.js | Declarative schema with migration generation |
| golang-migrate | Go | Simple, CLI + library |
| gh-ost | MySQL | Online schema change with zero locking |
| pt-online-schema-change | MySQL | Percona toolkit, shadow table approach |

### Ghost Table / Online Schema Change

```
For large tables (100M+ rows), ALTER TABLE locks the table for minutes/hours.

Solution: gh-ost (GitHub Online Schema Tool)

How it works:
  1. Create shadow table with new schema
  2. Replay binlog writes to shadow table (tracks all ongoing changes)
  3. Chunk-copy existing rows to shadow table
  4. Atomic table swap when shadow is caught up
  5. Original table renamed as backup

Result: schema migration with zero table lock on production traffic.
```

---

## 15. Strategy Comparison Matrix

| Strategy | Downtime | Rollback Speed | Blast Radius | Infra Cost | Complexity | Best For |
|---|---|---|---|---|---|---|
| **Big Bang** | Yes | Slow (redeploy) | 100% | Low | Low | Dev/staging only |
| **Rolling** | None | Medium (roll fwd) | Partial | None extra | Low | General use, Kubernetes default |
| **Blue-Green** | None | Instant (flip LB) | 0% (before switch) | 2× | Medium | Critical services, compliance |
| **Canary** | None | Instant | 1–10% | Slight | High | All production services at scale |
| **A/B** | None | Instant | Controlled segment | Slight | High | Business experimentation |
| **Shadow** | None | N/A | 0% (users) | 2× | High | Critical rewrites, ML models |
| **Feature Flags** | None | Instant (toggle) | Configurable | Low | Medium | Any feature with risk |
| **Immutable Infra** | None | Re-provision | Depends on strategy | Medium | Medium | All cloud workloads |
| **GitOps** | None | `git revert` | Depends | Low extra | High | Kubernetes-native teams |

---

## 16. Real-World Architectures

### Google — Canary Everything

```
Google's deployment philosophy:
  No change ships directly to 100% of users. Ever.

Process (simplified):
  1. Dev → unit tests → code review → submit
  2. Auto-deploy to canary (1% of traffic for that service)
  3. Automated analysis: error rates, latency, SLO burn rate
  4. If pass → staged rollout: 1% → 10% → 50% → 100%
  5. Each stage: 30-minute minimum bake time + automated analysis
  6. Any stage fails → automatic rollback to previous version

Infrastructure:
  - Borg (internal Kubernetes predecessor) manages rollout
  - Monarch (internal monitoring) feeds canary analysis
  - Every service has SLOs — canary analysis checks SLO burn rate
  
Result: Google ships code to millions of users thousands of times per day
        with a rollback rate that makes each individual deploy low-risk.
```

### Netflix — Automated Canary Analysis (ACA)

```
Netflix's Kayenta system:

  1. New version deployed as canary (small % of traffic)
  2. Kayenta pulls metrics for canary and baseline from Atlas (metrics platform)
  3. Statistical comparison: Mann-Whitney U test on metric distributions
  4. Score assigned: 0 (terrible) → 100 (identical to baseline)
  5. Score > 75: auto-promote
     Score < 75: auto-rollback
     Human review: 60–75

  Metrics analyzed:
    - Request error rate
    - p50, p95, p99 latency
    - JVM heap usage
    - Custom business metrics (stream starts, auth failures)

Netflix also uses:
  - Chaos Monkey: randomly terminates instances in production
  - Chaos Kong: simulates entire AWS region failure
  → Forces deployment to be resilient to partial failures by design
```

### Amazon — Deployment Cells

```
Amazon's approach for large services (S3, DynamoDB):

  Infrastructure divided into "cells" — isolated deployments
  Each cell serves a subset of customers

  Deployment order:
    1. One-box: single host in one cell (1 server out of thousands)
       → run for 1 hour → check metrics
    2. Deploy one cell fully
       → run for 24 hours → check metrics
    3. Deploy all cells in one region
    4. Deploy remaining regions sequentially

  If anything fails at any stage:
    → rollback that stage only
    → rest of fleet unaffected

This gives S3/DynamoDB the ability to deploy with essentially zero blast radius
at each progressive stage.
```

### Facebook / Meta — Dark Launching at Scale

```
Facebook ships code continuously but controls exposure via:

  1. Internal employees first ("dogfooding")
  2. Beta users (opted-in power users)
  3. 1% of users
  4. Progressive % rollout with automatic metric monitoring

Gating infrastructure (Gatekeeper):
  - Every feature has a "gate" that controls exposure
  - Gates can be flipped instantly for any user segment
  - Rollback = flip gate, no redeployment

Database changes:
  - Never break backwards compatibility in a single migration
  - Expand-contract pattern across multiple deploy cycles
  - Read traffic shifted to new column only after confirmed safe
```

---

## 17. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **Deploying DB schema and app code simultaneously** | Old app hits new schema → runtime error | Expand-contract: schema deploy first, backwards compatible, then app |
| **No health checks before routing traffic** | Traffic hits instances still starting up → errors | Readiness probe: only route traffic after health check passes |
| **Canary with too short a bake time** | Slow bugs (memory leaks, eventual consistency) not caught | Minimum 30-minute bake per stage; 24h for stateful services |
| **Blue-green with separate databases** | Data written to blue's DB lost on cutover | Both environments share the same database always |
| **Feature flags never cleaned up** | Dead code accumulates, cognitive overhead, security risk | Flag removal ticket at creation; max 2-sprint lifetime for release flags |
| **Rolling deploy with incompatible API changes** | v1 and v2 call each other — protocol mismatch | Ensure all versions can communicate during transition |
| **Rollback without testing it** | Rollback procedure fails during incident (worst time to discover this) | Test rollback in staging regularly; game-day exercises |
| **Skipping staging environment** | Bugs only found in production | Staging must mirror production deployment pipeline and strategy |
| **Manual deployments in production** | Inconsistent, unaudited, undocumented | All production changes via automated pipeline with Git as source of truth |
| **No deployment metrics** | You don't know if a deploy was safe | Alert on error rate, latency, and business metrics for every deploy |
| **Deploying on Friday afternoon** | Bug found → on-call on weekend → long MTTR | Deployment freeze: Fri afternoon to Mon morning for non-critical services |

---

## 18. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Big Bang** | Replace everything at once — downtime guaranteed |
| **Rolling** | Replace instances in batches — zero downtime, mixed versions briefly |
| **Blue-Green** | Two identical environments — instant atomic cutover, instant rollback |
| **Canary** | Route small % to new version — blast radius contained, auto rollback |
| **A/B deployment** | Route specific user segments to different versions — measure business impact |
| **Shadow** | Mirror traffic to new version — users see nothing, you see everything |
| **Feature flag** | Decouple deployment from release — toggle features without redeployment |
| **Immutable infra** | Never modify servers — bake new image, replace instances |
| **GitOps** | Git is source of truth — agent reconciles cluster to match Git state |
| **Expand-Contract** | Three-phase DB migration for zero-downtime schema changes |
| **Blast radius** | Fraction of users affected if this deployment fails |
| **Bake time** | Observation period after each canary stage before promoting |

### When to Use Which Strategy

| Scenario | Recommendation |
|---|---|
| Internal tool, acceptable downtime window | Big Bang (simplest) |
| General web service, Kubernetes, low risk change | Rolling |
| Compliance requirement, instant rollback mandatory | Blue-Green |
| Any risky production change | Canary with automated analysis |
| Measuring business impact of product change | A/B with statistical analysis |
| Critical rewrite, ML model swap, payment system | Shadow first, then canary |
| New feature with uncertain risk | Feature flag (dark launch → progressive rollout) |
| Cloud infra, want config drift eliminated | Immutable infrastructure |
| Kubernetes-native team, auditability required | GitOps (ArgoCD/Flux) |
| Breaking DB schema change, zero downtime | Expand-Contract over multiple deploys |

### The Interview Answer Template

When deployment strategy comes up:

```
1. RISK PROFILE:
   "For this change I'd use canary because X. If it were higher risk I'd
    shadow it first. If it were purely a business question I'd A/B test it."

2. ROLLBACK PLAN:
   "Rollback: flip traffic back at LB in seconds / toggle feature flag off /
    git revert + merge. For DB changes: schema is backwards compatible so
    app rollback works without schema rollback."

3. BLAST RADIUS:
   "At most N% of users hit the bug. Once error rate exceeds threshold,
    automated system rolls back without human intervention."

4. DATABASE:
   "DB change uses expand-contract: ship backwards-compatible schema first,
    then shift app, then drop old. Three deploys, each independently safe."

5. METRICS GATE:
   "Promote only if: error rate ≤ baseline + 0.1%, p99 latency ≤ baseline + 20ms,
    business metric not degraded. Automated canary analysis makes the call."
```

### Must-Know Interview Points

- ☑ **Deployment ≠ Release.** Feature flags decouple them — code ships dark, feature goes live when ready.
- ☑ **Database is the hard part.** App rollback is easy. Schema rollback is dangerous. Use expand-contract.
- ☑ **Blue-green shares a DB.** Separate DBs per environment means data loss on cutover.
- ☑ **Canary = risk reduction. A/B = measurement.** Different goals, similar mechanism.
- ☑ **Shadow requires suppressing side effects.** v2 must not write to DB, send emails, charge cards on mirrored traffic.
- ☑ **Rollback should be boring.** Untested rollback procedures fail exactly when you need them most.
- ☑ **Immutable infra eliminates drift.** Servers are cattle not pets — replace, never patch.
- ☑ **GitOps = Git is the source of truth.** Agent reconciles. Rollback is `git revert`.
- ☑ **Feature flag debt is technical debt.** Every flag needs a removal date.
- ☑ **Friday deploys are a risk decision.** Not a rule — depends on your rollback speed and on-call maturity.
- ☑ **Multi-region = region-as-canary.** Deploy to smallest region first, largest region last.

---

*Sources: Google SRE Book, Netflix Tech Blog, Amazon Builder's Library, Martin Fowler (BlueGreenDeployment, FeatureToggle, BranchByAbstraction), HashiCorp Immutable Infrastructure, Weaveworks GitOps, Database Refactoring (Ambler & Sadalage) — combined with first-principles system design knowledge.*