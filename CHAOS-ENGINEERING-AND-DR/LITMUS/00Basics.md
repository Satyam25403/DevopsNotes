# LitmusChaos Basics - What It Is, Architecture, Install

Essential LitmusChaos concepts and setup commands for running chaos engineering experiments on Kubernetes, with visual diagrams.

## Table of Contents
- [What is LitmusChaos](#what-is-litmuschaos)
- [Litmus vs Chaos Mesh](#litmus-vs-chaos-mesh)
- [Architecture](#architecture)
- [Install ChaosCenter](#install-chaoscenter)
- [Connect a Cluster (Agent)](#connect-a-cluster-agent)
- [Install the litmusctl CLI](#install-the-litmusctl-cli)
- [Verify Installation](#verify-installation)

---

## What is LitmusChaos

**LitmusChaos is a CNCF chaos engineering framework for Kubernetes, built around a central management console (ChaosCenter) and a marketplace of pre-built, community-contributed experiments (ChaosHub).**

**Visual:**
```
Core Idea:
┌────────────────────────────────────────────────┐
│  ChaosHub                                        │
│  (100+ ready-made experiments: pod-delete,        │
│   node-drain, disk-fill, cpu-hog, aws-ssm-chaos,  │
│   http-latency, kafka-broker-pod-failure, ...)     │
└─────────────────────┬────────────────────────────┘
                       │ install/run via
┌─────────────────────▼────────────────────────────┐
│  ChaosCenter (web console)                         │
│  build workflows visually, schedule, observe        │
│  resilience score across the whole application       │
└─────────────────────┬────────────────────────────┘
                       │ orchestrates
┌─────────────────────▼────────────────────────────┐
│  Chaos Operator + Argo Workflows                    │
│  runs the actual experiments as Kubernetes jobs      │
└────────────────────────────────────────────────┘
```

**What it gives you:**
- ChaosHub - large catalog of pre-built experiments across K8s, cloud (AWS/GCP/Azure), and platforms (Kafka, Kafka, Spring Boot, etc.)
- ChaosCenter - centralized multi-cluster dashboard with resilience scoring
- Built on Argo Workflows - each chaos run is itself a Kubernetes-native workflow
- Probes - built-in way to validate steady-state (HTTP, command, k8s, Prometheus checks) during an experiment
- GitOps support - store and sync experiments from a Git repo

---

## Litmus vs Chaos Mesh

**Both are CNCF chaos projects - worth knowing the practical differences.**

**Visual:**
```
LitmusChaos                          Chaos Mesh
┌──────────────────────┐            ┌──────────────────────┐
│ ChaosHub - large        │            │ No built-in catalog -   │
│  experiment catalog      │            │  you author your own    │
│  (100+ ready experiments)│            │  CRDs from scratch       │
│                          │            │                          │
│ ChaosCenter - multi-      │            │ Chaos Dashboard -         │
│  cluster dashboard,        │            │  single-cluster focused    │
│  resilience score           │            │                            │
│                              │            │                            │
│ Built on Argo Workflows       │            │ Own Workflow/Schedule CRDs │
│                                │            │                            │
│ Probes for steady-state         │            │ No built-in probe concept -│
│  validation built in              │            │  you check externally       │
└──────────────────────────────┘            └──────────────────────────┘

Choose Litmus when: you want a large ready-made experiment
                     catalog and multi-cluster fleet management
Choose Chaos Mesh when: you want tighter, simpler native K8s
                     CRDs and don't need a marketplace
```

---

## Architecture

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                  ChaosCenter (control plane)               │
│         (can be self-hosted or Litmus Cloud SaaS)           │
│                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌──────────────┐ │
│  │ litmus-server   │  │ litmus-frontend │  │  MongoDB      │ │
│  │ (GraphQL API)   │  │ (web UI)        │  │  (state store)│ │
│  └────────────────┘  └────────────────┘  └──────────────┘ │
└───────────────────────────┬───────────────────────────────┘
                            │ connects to
              ┌─────────────┴─────────────┐
              ▼                           ▼
┌───────────────────────┐    ┌───────────────────────┐
│  Cluster: staging        │    │  Cluster: production    │
│  (Subscriber/Agent)      │    │  (Subscriber/Agent)      │
│                          │    │                          │
│  chaos-operator           │    │  chaos-operator           │
│  Argo Workflow controller │    │  Argo Workflow controller │
│  ChaosExperiment CRDs      │    │  ChaosExperiment CRDs      │
│  ChaosEngine CRDs           │    │  ChaosEngine CRDs           │
│  ChaosResult CRDs            │    │  ChaosResult CRDs            │
└───────────────────────┘    └───────────────────────┘
```

**Key components:**
| Component | Role |
|---|---|
| `ChaosCenter` | Central web console (server + frontend + DB), can manage multiple clusters |
| `Subscriber / Agent` | Lightweight component installed per-cluster that connects it to ChaosCenter |
| `chaos-operator` | Watches ChaosEngine CRDs, spins up the actual experiment pods |
| `Argo Workflows` | Underlying engine that runs multi-step chaos scenarios |
| `ChaosHub` | Git-backed catalog of experiment definitions |

---

## Install ChaosCenter

```bash
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-portal/2.0.0/litmus-portal-crds.yml
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-portal/2.0.0/litmus-portal-admin-rbac.yml
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-portal/2.0.0/litmus-portal-mongodb.yml
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-portal/2.0.0/litmus-portal-server.yml
kubectl apply -f https://litmuschaos.github.io/litmus/litmus-portal/2.0.0/litmus-portal-frontend.yml
```

**Or via Helm (recommended for production):**

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
kubectl create ns litmus
helm install chaos litmuschaos/litmus -n litmus
```

**Visual:**
```
Namespace: litmus
┌─────────────────────────────────┐
│ litmusportal-server (Deploy)     │
│ litmusportal-frontend (Deploy)   │
│ litmusportal-mongodb (StatefulSet)│
└─────────────────────────────────┘

Get the dashboard NodePort/LoadBalancer:
kubectl get svc -n litmus litmusportal-frontend-service
```

---

## Connect a Cluster (Agent)

**Every cluster you want to run chaos in must be "onboarded" as an Agent/Subscriber from the ChaosCenter UI or via litmusctl.**

**Visual:**
```
ChaosCenter UI Flow:
┌────────────────────────────────────────────┐
│ 1. Environments → New Environment            │
│ 2. Choose type: Non-Production / Production   │
│ 3. Choose install mode: Cluster-wide / Namespace│
│ 4. Copy generated kubectl apply command        │
│ 5. Run it against the TARGET cluster            │
└────────────────────────────────────────────┘

Result:
┌───────────────────────┐
│ Target Cluster           │
│  subscriber (Deploy)      │  ← connects back to ChaosCenter
│  chaos-operator (Deploy)  │  ← executes experiments
│  event-tracker (Deploy)   │  ← streams status back
└───────────────────────┘

Cluster-wide install → can run chaos in ANY namespace
Namespace install     → scoped to just one namespace (safer default)
```

---

## Install the litmusctl CLI

```bash
curl -O https://raw.githubusercontent.com/litmuschaos/litmusctl/master/install.sh
sh install.sh
```

**Visual:**
```
litmusctl lets you manage Litmus from the terminal instead
of only the web UI - useful for scripting/CI integration.

litmusctl config set-account --endpoint=<chaoscenter-url> ...
litmusctl create project --name my-project
litmusctl get agents --project-id <id>
```

---

## Verify Installation

```bash
kubectl get pods -n litmus
kubectl get pods -n <agent-namespace>
```

**Output Example:**
```
NAMESPACE: litmus
NAME                                    READY   STATUS
litmusportal-server-7f8...              3/3     Running
litmusportal-frontend-5c7...            1/1     Running
mongodb-0                                1/1     Running

NAMESPACE: my-app (agent namespace)
NAME                                    READY   STATUS
chaos-operator-ce-6f8...                1/1     Running
subscriber-7d9...                       1/1     Running
event-tracker-8x1...                    1/1     Running
```

**Visual:**
```
Green checkmark in ChaosCenter UI "Environments" tab
= agent successfully connected and reporting healthy
```

---

## Visual Summary

```
1. Install ChaosCenter (control plane)
   helm install chaos litmuschaos/litmus -n litmus

2. Open dashboard
   kubectl get svc -n litmus litmusportal-frontend-service

3. Onboard a cluster (Agent/Subscriber)
   Environments → New Environment → apply generated YAML

4. Browse ChaosHub
   Pick a pre-built experiment (e.g. pod-delete)

5. Run it
   Create ChaosEngine → chaos-operator executes it

6. Observe
   ChaosResult CRD + ChaosCenter dashboard show resilience score
```

**Core Idea:**
```
┌────────────────┐  browse   ┌────────────────┐   run    ┌────────────────┐
│   ChaosHub     │ ────────→ │  ChaosEngine   │ ───────→ │  ChaosResult   │
│ (experiment     │           │ (this run's     │           │ (pass/fail +    │
│  catalog)         │           │  config)          │           │  resilience score)│
└────────────────┘           └────────────────┘           └────────────────┘
```

---

This guide covers LitmusChaos basics: what it is, how it compares to Chaos Mesh, its architecture, and installing ChaosCenter with an onboarded cluster agent, with visual representations of each step.