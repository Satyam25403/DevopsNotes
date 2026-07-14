# Service Mesh (Istio) - Installation & Setup

Getting Istio installed on Kubernetes, enabling sidecar injection, and deploying your first meshed application.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installing Istio with istioctl](#installing-istio-with-istioctl)
- [Installation Profiles](#installation-profiles)
- [Verifying the Installation](#verifying-the-installation)
- [Enabling Sidecar Injection](#enabling-sidecar-injection)
- [Deploying Your First Meshed Application](#deploying-your-first-meshed-application)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Prerequisites

**Visual:**
```
┌──────────────────────────────────────────┐
│  Requirement          Why                     │
├──────────────────────────────────────────┤
│  A running Kubernetes cluster    Istio installs INTO an     │
│                                existing cluster, not standalone│
│  kubectl configured               Needed for verification and    │
│                                troubleshooting                     │
│  Sufficient cluster resources        Sidecars add CPU/memory           │
│                                overhead per pod                          │
└──────────────────────────────────────────┘
```

---

## Installing Istio with istioctl

```bash
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

istioctl install --set profile=demo -y
```

**Visual:**
```
Installation Flow:
┌────────────┐  ┌──────────────┐  ┌───────────────┐
│ Download          │→ │ istioctl installs   │→ │ istiod deployed        │
│ istioctl CLI          │  │ CRDs + control plane   │  │ into istio-system         │
└────────────┘  └──────────────┘  │ namespace                    │
                                    └───────────────┘

What gets installed:
- Custom Resource Definitions (VirtualService,
  DestinationRule, Gateway, PeerAuthentication, etc.)
- The istiod control plane deployment
- (Optionally) ingress/egress gateway deployments
```

---

## Installation Profiles

**Visual:**
```
┌───────────────────────────────────────────────┐
│ Profile          Use Case                              │
├───────────────────────────────────────────────┤
│ default             Production-oriented baseline,          │
│                   ingress gateway included                    │
│ demo                  All features enabled, tracing/Kiali        │
│                   ready — best for LEARNING/evaluation,           │
│                   not tuned for production resource usage            │
│ minimal                 Just the control plane, no gateways,           │
│                   smallest footprint                                     │
│ ambient                   Newer "sidecar-less" mode (see file 05)          │
└───────────────────────────────────────────────┘
```

```bash
# Production-oriented starting point
istioctl install --set profile=default -y
```

**Visual:**
```
Why start with "demo" for learning, "default" for real use:
demo enables extra components (tracing sample apps,
telemetry add-ons) useful for exploring Istio's
capabilities firsthand, but isn't resource-tuned —
production deployments should start from "default"
and customize from there.
```

---

## Verifying the Installation

```bash
kubectl get pods -n istio-system
```

**Visual:**
```
Expected Output:
NAME                     READY   STATUS    
istiod-7d8f9c6b5-xk2p9     1/1       Running     
istio-ingressgateway-...     1/1       Running     

istiod = the control plane (Pilot, Citadel, Galley
         functionality all consolidated into one binary
         since Istio 1.5+)
istio-ingressgateway = handles traffic entering the mesh
                       from outside the cluster
```

```bash
istioctl verify-install
```

**Visual:**
```
This command cross-checks that everything istioctl
tried to install actually landed correctly in the
cluster — catches partial/failed installations before
you spend time debugging a mesh that isn't actually
fully configured.
```

---

## Enabling Sidecar Injection

**Sidecar injection can happen automatically (namespace-wide) or manually (per-deployment) — automatic is the standard approach.**

```bash
kubectl label namespace default istio-injection=enabled
```

**Visual:**
```
Before labeling:
┌────────────────┐
│  Pod: myapp            │
│  ┌──────────────┐  │
│  │  App Container       │  │  ← ONLY the app container,
│  └──────────────┘  │     no sidecar
└────────────────┘

After labeling + new pod deployment:
┌────────────────┐
│  Pod: myapp            │
│  ┌──────────┐┌──────────┐│
│  │  App           ││  Envoy         ││  ← sidecar automatically
│  │  Container       ││  Sidecar        ││     injected on pod creation
│  └──────────┘└──────────┘│
└────────────────┘

Critical detail: labeling a namespace does NOT retroactively
inject sidecars into ALREADY-RUNNING pods — only pods
created AFTER labeling get the sidecar. Existing deployments
need a rollout restart:
kubectl rollout restart deployment -n default
```

---

## Deploying Your First Meshed Application

```yaml
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpbin
spec:
  replicas: 1
  selector:
    matchLabels:
      app: httpbin
  template:
    metadata:
      labels:
        app: httpbin
    spec:
      containers:
        - name: httpbin
          image: docker.io/kong/httpbin
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: httpbin
spec:
  ports:
    - port: 8000
      targetPort: 80
  selector:
    app: httpbin
```

```bash
kubectl apply -f sample-app.yaml
kubectl get pods
```

**Visual:**
```
Expected Output:
NAME                       READY   STATUS
httpbin-7d4b8f9c6-xk2p9      2/2       Running

"2/2" is the key detail — TWO containers are running
in this pod (the app container AND the Envoy sidecar),
confirming injection worked correctly. If you see "1/2"
or "1/1", the sidecar was NOT injected — check that the
namespace label was applied BEFORE this pod was created.
```

**Confirming the sidecar is present:**
```bash
kubectl describe pod httpbin-7d4b8f9c6-xk2p9 | grep -A 2 "istio-proxy"
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is rolling out Istio to a cluster running 15 existing microservices, and needs to do so without causing a production outage.

**What they do:**
1. Installs Istio using the **default profile** rather than demo, since this is heading toward production use, sizing the control plane resources appropriately for their cluster scale.
2. **Does NOT immediately label all namespaces** — instead, labels only a single low-traffic, non-critical namespace first (`internal-tools`) as a pilot, confirming sidecar injection and basic traffic flow work correctly before touching anything customer-facing.
3. After validating the pilot namespace for a few days with no issues, labels the next namespace, and performs a **controlled rollout restart** (`kubectl rollout restart deployment`) rather than deleting all pods simultaneously — ensuring Kubernetes' normal rolling update behavior keeps the service available throughout the sidecar injection process.
4. Uses `istioctl verify-install` and pod "2/2" ready-count checks as an explicit **verification gate** in their rollout runbook, catching any injection failures immediately rather than discovering them later during an incident.
5. Only after all 15 services are successfully running with sidecars does the engineer move on to configuring mTLS enforcement and traffic management policies (covered in later files) — deliberately separating "get the mesh installed safely" from "start using its advanced features" as two distinct phases.

**Why this matters:** Istio adoption failures most commonly happen from rolling out sidecar injection too aggressively across an entire cluster at once — a namespace-by-namespace, verified rollout is what prevents a service mesh installation from becoming an unplanned outage.

---

Next: **02traffic_management.md** — VirtualServices, DestinationRules, and controlling exactly how traffic flows between services.