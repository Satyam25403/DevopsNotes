# Linkerd Basics - What It Is, Architecture, Install

Essential Linkerd concepts and setup commands for running a service mesh in Kubernetes, with visual diagrams.

## Table of Contents
- [What is Linkerd](#what-is-linkerd)
- [Service Mesh Concept](#service-mesh-concept)
- [Control Plane vs Data Plane](#control-plane-vs-data-plane)
- [Install the CLI](#install-the-cli)
- [Pre-Install Checks](#pre-install-checks)
- [Install the Control Plane](#install-the-control-plane)
- [Verify Installation](#verify-installation)
- [Install the Viz Extension](#install-the-viz-extension)

---

## What is Linkerd

**Linkerd is an ultralight, security-first service mesh for Kubernetes.** It adds observability, reliability, and mTLS security to services without requiring application code changes.

**Visual:**
```
Without Linkerd:
┌──────────┐   plain HTTP/gRPC   ┌──────────┐
│ Service A│ ──────────────────→│ Service B│
└──────────┘                     └──────────┘
  No encryption, no retries, no metrics

With Linkerd:
┌──────────┐   ┌───────┐  mTLS  ┌───────┐   ┌──────────┐
│ Service A│──→│ proxy │═══════→│ proxy │──→│ Service B│
└──────────┘   └───────┘        └───────┘   └──────────┘
                Transparent, automatic, no code change
```

**What it gives you for free:**
- Automatic mutual TLS (mTLS) between meshed pods
- Golden metrics: success rate, request volume, latency (p50/p95/p99)
- Automatic retries and timeouts
- Load balancing (EWMA - exponentially weighted moving average)
- Traffic splitting for canary/blue-green releases
- Zero code changes - works at the network layer

---

## Service Mesh Concept

**A service mesh is a dedicated infrastructure layer for service-to-service communication.**

**Visual:**
```
Kubernetes Cluster
┌───────────────────────────────────────────────────────┐
│  Namespace: default                                    │
│                                                         │
│  Pod: frontend            Pod: backend                 │
│  ┌─────────────────┐      ┌─────────────────┐          │
│  │ App Container   │      │ App Container   │          │
│  │      ↓↑         │      │      ↓↑         │          │
│  │ linkerd-proxy   │═════→│ linkerd-proxy   │          │
│  │ (sidecar)       │ mTLS │ (sidecar)       │          │
│  └─────────────────┘      └─────────────────┘          │
│         ↑                         ↑                    │
│         └─── metrics/policy ──────┘                     │
│                    │                                     │
│         ┌──────────┴──────────┐                          │
│         │   Control Plane      │                         │
│         │ (destination, proxy- │                         │
│         │  injector, identity) │                         │
│         └──────────────────────┘                         │
└───────────────────────────────────────────────────────┘

Every network call goes through a local proxy (sidecar)
The proxy handles TLS, retries, load balancing, metrics
```

---

## Control Plane vs Data Plane

**Two logical halves of Linkerd.**

**Visual:**
```
┌───────────────────────────────────────────────────────┐
│                    CONTROL PLANE                       │
│                 (namespace: linkerd)                    │
│                                                         │
│  ┌───────────┐  ┌────────────┐  ┌──────────────────┐   │
│  │identity   │  │destination │  │ proxy-injector    │   │
│  │(issues    │  │(service    │  │ (webhook, adds    │   │
│  │ mTLS certs│  │ discovery, │  │  sidecar to pods) │   │
│  │ )         │  │ policy)    │  │                   │   │
│  └───────────┘  └────────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────┘
                        │
                configures/certifies
                        ↓
┌───────────────────────────────────────────────────────┐
│                     DATA PLANE                         │
│           (linkerd-proxy sidecar in every pod)          │
│                                                         │
│   [proxy] ←→ [proxy] ←→ [proxy] ←→ [proxy] ...          │
│                                                         │
│   Written in Rust, ~10-20MB memory, sub-ms latency      │
└───────────────────────────────────────────────────────┘
```

**Key components:**
| Component | Role |
|---|---|
| `identity` | Issues short-lived TLS certs to proxies (mTLS) |
| `destination` | Service discovery, routing, traffic policy |
| `proxy-injector` | Mutating webhook that injects `linkerd-proxy` sidecar |
| `linkerd-proxy` | The actual data-plane sidecar (micro-proxy, Rust-based) |

---

## Install the CLI

### curl install script

**Installs the `linkerd` CLI binary locally.**

```bash
curl --proto '=https' --tlsv1.2 -sSf https://run.linkerd.io/install | sh
```

**Add to PATH:**
```bash
export PATH=$PATH:$HOME/.linkerd2/bin
```

**Visual:**
```
Your Machine
┌─────────────────────┐
│ ~/.linkerd2/bin/     │
│   └── linkerd        │  ← CLI binary
└─────────────────────┘

CLI talks to:
linkerd CLI ──kubectl config──→ Kubernetes API ──→ Cluster
```

### Verify CLI version

```bash
linkerd version
```

**Output Example:**
```
Client version: stable-2.14.x
Server version: unavailable   ← control plane not installed yet
```

---

## Pre-Install Checks

### linkerd check --pre

**Validates the cluster is ready for installation (RBAC, kubelet version, existing resources).**

```bash
linkerd check --pre
```

**Visual:**
```
Checks performed:
┌─────────────────────────────────────┐
│ ✓ can create Namespaces              │
│ ✓ can create ClusterRoles            │
│ ✓ can create CustomResourceDefinitions│
│ ✓ 'linkerd' namespace does not exist │
│ ✓ no clock skew detected             │
└─────────────────────────────────────┘

If all pass → safe to install
If any fail → fix cluster permissions first
```

---

## Install the Control Plane

### Generate and install CRDs first

```bash
linkerd install --crds | kubectl apply -f -
```

### Install the control plane

```bash
linkerd install | kubectl apply -f -
```

**Visual:**
```
Step 1: CRDs
linkerd install --crds  ──→  kubectl apply  ──→  CRDs registered
  (ServiceProfile, Server, AuthorizationPolicy, HTTPRoute, ...)

Step 2: Control Plane
linkerd install  ──→  kubectl apply  ──→  Namespace: linkerd created
                                          ┌──────────────────────┐
                                          │ linkerd-identity      │
                                          │ linkerd-destination   │
                                          │ linkerd-proxy-injector│
                                          └──────────────────────┘
```

### Production alternative: Helm install

```bash
helm repo add linkerd-edge https://helm.linkerd.io/edge
helm install linkerd-crds linkerd-edge/linkerd-crds -n linkerd --create-namespace
helm install linkerd-control-plane linkerd-edge/linkerd-control-plane \
  -n linkerd \
  --set-file identityTrustAnchorsPEM=ca.crt \
  --set-file identity.issuer.tls.crtPEM=issuer.crt \
  --set-file identity.issuer.tls.keyPEM=issuer.key
```

**Visual:**
```
Why Helm in production:
- Reproducible via values.yaml (GitOps friendly)
- Easier upgrades and rollbacks
- Supports custom trust anchors (own PKI, not linkerd's auto-generated one)

CLI install  → good for demos/dev
Helm install → recommended for production pipelines
```

---

## Verify Installation

### linkerd check

**Confirms control plane is healthy.**

```bash
linkerd check
```

**Visual:**
```
Checks:
┌───────────────────────────────────────┐
│ kubernetes-api                        │
│ ✓ can initialize the client            │
│                                        │
│ linkerd-existence                     │
│ ✓ control plane namespace exists       │
│ ✓ control plane pods are ready         │
│                                        │
│ linkerd-identity                      │
│ ✓ certificate config is valid          │
│ ✓ trust anchors are within expiry      │
│                                        │
│ linkerd-webhooks-and-apisvc-tls        │
│ ✓ proxy-injector webhook has valid cert│
└───────────────────────────────────────┘

Status: Linkerd is healthy
```

### Check control plane pods

```bash
kubectl get pods -n linkerd
```

**Output Example:**
```
NAME                                      READY   STATUS    RESTARTS
linkerd-destination-6f8...                4/4     Running   0
linkerd-identity-7d9...                   2/2     Running   0
linkerd-proxy-injector-5c7...             2/2     Running   0
```

---

## Install the Viz Extension

**Adds dashboard, Prometheus, and `linkerd viz` metrics commands (optional but almost always used in practice).**

```bash
linkerd viz install | kubectl apply -f -
linkerd check
```

**Visual:**
```
Namespace: linkerd-viz
┌─────────────────────────────────┐
│ metrics-api                     │
│ prometheus                      │
│ web (dashboard)                 │
│ tap                             │
│ grafana (optional)              │
└─────────────────────────────────┘

linkerd viz dashboard   → opens local browser dashboard
```

### Open the dashboard

```bash
linkerd viz dashboard &
```

**Visual:**
```
Browser Dashboard
┌────────────────────────────────────┐
│ Namespaces  Deployments  Meshed %  │
│ default     frontend      3/3      │
│ default     backend       2/2      │
└────────────────────────────────────┘
Shows: success rate, RPS, p50/p95/p99 latency per resource
```

---

## Visual Summary

**Complete Install Flow:**

```
1. Install CLI
   curl ... | sh
   ┌─────────────┐
   │ linkerd CLI │
   └─────────────┘

2. Pre-flight Check
   linkerd check --pre
   ┌─────────────┐
   │ Cluster OK  │
   └─────────────┘

3. Install CRDs
   linkerd install --crds | kubectl apply -f -

4. Install Control Plane
   linkerd install | kubectl apply -f -
   ┌──────────────────┐
   │ namespace:linkerd│
   └──────────────────┘

5. Verify
   linkerd check
   ┌─────────────┐
   │ Healthy ✓   │
   └─────────────┘

6. Install Viz (optional but common)
   linkerd viz install | kubectl apply -f -
   linkerd viz dashboard
```

**Three Core Ideas:**
```
┌────────────────┐   inject    ┌────────────────┐   secure    ┌────────────────┐
│  Kubernetes    │  ────────→  │   Data Plane   │  ────────→  │   mTLS + obs.  │
│    Pod         │             │  (proxy sidecar│             │   automatic    │
│                │             │   injected)    │             │                │
└────────────────┘             └────────────────┘             └────────────────┘
```

---

This guide covers Linkerd basics: what a service mesh is, control vs data plane, and installing Linkerd with visual representations of each step.