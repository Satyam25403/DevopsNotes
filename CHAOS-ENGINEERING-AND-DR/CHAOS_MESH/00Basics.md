# Chaos Mesh Basics - What It Is, Architecture, Install

Essential Chaos Mesh concepts and setup commands for running chaos engineering experiments on Kubernetes, with visual diagrams.

## Table of Contents
- [What is Chaos Engineering](#what-is-chaos-engineering)
- [What is Chaos Mesh](#what-is-chaos-mesh)
- [Architecture](#architecture)
- [Install the CLI (chaosctl)](#install-the-cli-chaosctl)
- [Install Chaos Mesh via Helm](#install-chaos-mesh-via-helm)
- [Verify Installation](#verify-installation)
- [Your First Experiment](#your-first-experiment)

---

## What is Chaos Engineering

**Chaos engineering is the practice of deliberately injecting failure into a system to verify it survives real-world turbulence, before that turbulence happens in production unplanned.**

**Visual:**
```
Traditional Testing                    Chaos Engineering
┌──────────────────────┐              ┌──────────────────────┐
│ Does the code work    │              │ Does the SYSTEM       │
│ under normal          │              │ survive when a node    │
│ conditions?           │              │ dies, network drops,   │
│                       │              │ or a pod gets killed?  │
└──────────────────────┘              └──────────────────────┘

Question answered:                     Question answered:
"Is the code correct?"                 "Is production resilient?"
```

**The core loop:**
```
1. Define steady state   (e.g. success rate > 99%, p99 < 200ms)
2. Form a hypothesis      ("if pod X dies, traffic fails over cleanly")
3. Inject real failure    (kill pod, add latency, drop network)
4. Observe                (did steady state hold?)
5. Fix the weakness       (retries, timeouts, more replicas, alerts)
6. Automate & repeat      (run continuously, not just once)
```

---

## What is Chaos Mesh

**Chaos Mesh is a CNCF chaos engineering platform built specifically for Kubernetes. It manages fault injection declaratively via CRDs (Custom Resource Definitions) - you write YAML, Chaos Mesh does the injection.**

**Visual:**
```
Without Chaos Mesh:
Engineer manually SSHs into a node,
runs `kill -9`, `tc`, `stress-ng` by hand
┌─────────────────────────────┐
│ Manual, inconsistent,        │
│ hard to repeat, no audit log │
└─────────────────────────────┘

With Chaos Mesh:
kubectl apply -f pod-kill-experiment.yaml
┌─────────────────────────────┐
│ Declarative, repeatable,     │
│ scheduled, versioned in Git, │
│ automatically reverted        │
└─────────────────────────────┘
```

**What it can inject:**
- Pod failures (kill, container kill, pod failure)
- Network faults (latency, packet loss, partition, corruption, bandwidth limits)
- Stress (CPU, memory pressure)
- I/O faults (disk latency, read/write errors)
- Time skew (clock drift)
- DNS faults (resolution errors, random DNS responses)
- HTTP faults (abort, delay, replace requests/responses)
- Kernel faults (via eBPF - injecting syscall errors)

---

## Architecture

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                 Namespace: chaos-mesh                     │
│                                                            │
│  ┌───────────────────┐   ┌────────────────────────────┐  │
│  │ chaos-controller-  │   │ chaos-dashboard             │  │
│  │ manager            │   │ (web UI, RBAC, event log)  │  │
│  │ (watches CRDs,     │   │                            │  │
│  │  schedules faults) │   └────────────────────────────┘  │
│  └─────────┬──────────┘                                   │
│            │ instructs                                     │
│  ┌─────────▼──────────┐                                    │
│  │ chaos-daemon        │  ← DaemonSet, one per node          │
│  │ (privileged, does   │                                    │
│  │  the actual fault    │                                    │
│  │  injection via       │                                    │
│  │  iptables/tc/eBPF)   │                                    │
│  └─────────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
            │ acts on
            ▼
┌─────────────────────────────────────────────────────────┐
│  Target Pods (in any application namespace)               │
│  frontend, backend, database ...                           │
└─────────────────────────────────────────────────────────┘
```

**Key components:**
| Component | Role |
|---|---|
| `chaos-controller-manager` | Watches Chaos CRDs, decides what/when/where to inject |
| `chaos-daemon` | Privileged DaemonSet on every node, executes the actual fault (iptables, tc, syscalls) |
| `chaos-dashboard` | Web UI to create, monitor, and audit experiments |
| `chaosctl` | CLI for debugging running experiments |

---

## Install the CLI (chaosctl)

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash
```

**Visual:**
```
Your Machine
┌─────────────────────┐
│ chaosctl             │  ← CLI binary, mainly used for debugging
└─────────────────────┘

chaosctl talks to the cluster the same way kubectl does
(uses your current kubeconfig context)
```

---

## Install Chaos Mesh via Helm

```bash
kubectl create ns chaos-mesh

helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update

helm install chaos-mesh chaos-mesh/chaos-mesh \
  -n chaos-mesh \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --version 2.6.3
```

**Visual:**
```
Step 1: Namespace
kubectl create ns chaos-mesh

Step 2: Helm install
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh
             │
             ▼
┌─────────────────────────────────┐
│ chaos-controller-manager (Deploy)│
│ chaos-daemon (DaemonSet)         │
│ chaos-dashboard (Deploy)         │
│ CRDs: PodChaos, NetworkChaos,    │
│  StressChaos, IOChaos, DNSChaos, │
│  HTTPChaos, TimeChaos, Workflow, │
│  Schedule ...                    │
└─────────────────────────────────┘

⚠️ runtime/socketPath MUST match your cluster's container
   runtime (containerd, docker, crio) or fault injection fails silently
```

---

## Verify Installation

```bash
kubectl get pods -n chaos-mesh
```

**Output Example:**
```
NAME                                       READY   STATUS    RESTARTS
chaos-controller-manager-6f8...            1/1     Running   0
chaos-daemon-4x9j2                         1/1     Running   0   ← one per node
chaos-daemon-7k2m1                         1/1     Running   0
chaos-dashboard-5c7...                     1/1     Running   0
```

```bash
kubectl get crds | grep chaos-mesh.org
```

**Visual:**
```
Expect to see CRDs like:
podchaos.chaos-mesh.org
networkchaos.chaos-mesh.org
stresschaos.chaos-mesh.org
iochaos.chaos-mesh.org
timechaos.chaos-mesh.org
dnschaos.chaos-mesh.org
httpchaos.chaos-mesh.org
kernelchaos.chaos-mesh.org
workflows.chaos-mesh.org
schedules.chaos-mesh.org

If these exist → Chaos Mesh is ready to accept experiments
```

### Access the dashboard

```bash
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

**Visual:**
```
Browser: http://localhost:2333
┌────────────────────────────────────┐
│ Chaos Mesh Dashboard                │
│  Experiments │ Workflows │ Events   │
│  Archives    │ Schedules            │
└────────────────────────────────────┘
```

---

## Your First Experiment

**Kill a random pod matching a label - the "hello world" of chaos engineering.**

```yaml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: kill-backend-pod
  namespace: my-app
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - my-app
    labelSelectors:
      app: backend
```

```bash
kubectl apply -f kill-backend-pod.yaml
kubectl get podchaos -n my-app
```

**Visual:**
```
Before:
backend-pod-1  Running
backend-pod-2  Running
backend-pod-3  Running

After PodChaos (action: pod-kill, mode: one):
backend-pod-1  Terminating  ← randomly chosen victim
backend-pod-2  Running
backend-pod-3  Running

Kubernetes reschedules backend-pod-1 automatically (if Deployment)
Question to answer: did traffic fail over without errors?
```

---

## Visual Summary

```
1. Install
   kubectl create ns chaos-mesh
   helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh

2. Verify
   kubectl get pods -n chaos-mesh
   kubectl get crds | grep chaos-mesh.org

3. Access dashboard
   kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333

4. Run first experiment
   kubectl apply -f kill-backend-pod.yaml

5. Observe
   Did the system stay healthy? (metrics, alerts, user impact)
```

**Core Idea:**
```
┌────────────────┐   inject    ┌────────────────┐   observe    ┌────────────────┐
│  Steady State  │  ────────→  │  Real Failure  │  ─────────→  │  Confidence or │
│  (assumed)     │             │  (controlled)  │              │  Weakness Found│
└────────────────┘             └────────────────┘             └────────────────┘
```

---

This guide covers Chaos Mesh basics: what chaos engineering is, Chaos Mesh's architecture, and getting your first fault-injection experiment running, with visual representations of each step.