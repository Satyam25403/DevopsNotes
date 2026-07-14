# LitmusChaos Core CRDs & ChaosHub - Visual Guide

The building blocks of every Litmus experiment: ChaosExperiment, ChaosEngine, ChaosResult, and the ChaosHub catalog they come from.

## Table of Contents
- [The Three Core CRDs](#the-three-core-crds)
- [ChaosHub](#chaoshub)
- [ChaosExperiment](#chaosexperiment)
- [ChaosEngine](#chaosengine)
- [ChaosResult and Resilience Score](#chaosresult-and-resilience-score)
- [Common Experiments Catalog](#common-experiments-catalog)
- [Environment Variables (Tunables)](#environment-variables-tunables)

---

## The Three Core CRDs

**Every Litmus experiment run involves the same three Kubernetes objects working together.**

**Visual:**
```
ChaosHub (Git repo)
      │ provides the template
      ▼
┌──────────────────┐
│ ChaosExperiment   │  ← "what CAN be run" (the definition/template)
│ (pod-delete)       │     installed once per cluster from ChaosHub
└─────────┬─────────┘
          │ referenced by
          ▼
┌──────────────────┐
│ ChaosEngine        │  ← "run THIS experiment on THIS target, NOW"
│ (my-pod-delete-run)│     you create one of these per actual run
└─────────┬─────────┘
          │ chaos-operator executes it, produces
          ▼
┌──────────────────┐
│ ChaosResult         │  ← "what HAPPENED" (pass/fail + verdict)
│ (my-pod-delete-run- │     auto-created, holds the resilience outcome
│  pod-delete)        │
└──────────────────┘
```

---

## ChaosHub

**A Git-backed repository of ready-to-use ChaosExperiment definitions - install experiments into your cluster from here rather than writing them by hand.**

**Visual:**
```
ChaosCenter UI:
┌────────────────────────────────────────────────┐
│ ChaosHubs                                        │
│  ▸ Litmus ChaosHub (official, default)            │
│  ▸ My Custom Hub (private Git repo, optional)      │
├────────────────────────────────────────────────┤
│ Categories:                                       │
│  Kubernetes: pod-delete, pod-cpu-hog, node-drain,  │
│              container-kill, disk-fill, pod-network- │
│              latency, pod-network-loss, pod-io-stress│
│  Cloud:      aws-ssm-chaos, ec2-terminate-by-tag,     │
│              gcp-vm-instance-stop, azure-instance-stop │
│  Platform:   kafka-broker-pod-failure, spring-boot-     │
│              app-chaos, cassandra-pod-delete             │
└────────────────────────────────────────────────┘
```

### Connect a private/custom ChaosHub

```bash
litmusctl create chaos-hub \
  --name my-custom-hub \
  --repo-url https://github.com/my-org/my-chaos-experiments \
  --repo-branch main
```

**Visual:**
```
Use case: your team maintains its OWN experiment definitions
(e.g. a custom "kafka-partition-leader-kill" experiment) and
wants it available in the same catalog UI as official ones,
version-controlled in your own Git repo.
```

---

## ChaosExperiment

**Installed from ChaosHub into your cluster - this is the reusable template. You rarely write these from scratch; you install one and reference it.**

```bash
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.0.0?file=charts/generic/pod-delete/experiment.yaml -n my-app
```

**Visual:**
```
kubectl get chaosexperiments -n my-app

NAME          AGE
pod-delete    2d
disk-fill     2d
cpu-hog       2d

Think of ChaosExperiment like a Helm chart's "template" -
it defines the container image, default parameters, and
permissions needed, but doesn't DO anything until run
via a ChaosEngine.
```

---

## ChaosEngine

**The object you actually create to trigger a run - links a target application to a ChaosExperiment with your specific parameters.**

```yaml
apiVersion: litmuschaos.io/v1alpha1
kind: ChaosEngine
metadata:
  name: backend-pod-delete
  namespace: my-app
spec:
  appinfo:
    appns: my-app
    applabel: "app=backend"
    appkind: deployment
  engineState: active
  chaosServiceAccount: litmus-admin
  experiments:
    - name: pod-delete
      spec:
        components:
          env:
            - name: TOTAL_CHAOS_DURATION
              value: "60"
            - name: CHAOS_INTERVAL
              value: "10"
            - name: FORCE
              value: "false"
```

```bash
kubectl apply -f backend-pod-delete.yaml
kubectl get chaosengine backend-pod-delete -n my-app
```

**Visual:**
```
ChaosEngine ties together:
appinfo         → WHICH application (namespace + label + kind)
experiments     → WHICH ChaosExperiment(s) to run, with what env vars
chaosServiceAccount → WHAT permissions the experiment runs with
engineState     → active (run now) | stop (halt immediately)

Setting engineState: stop on an existing ChaosEngine is the
abort/kill-switch equivalent for Litmus.
```

---

## ChaosResult and Resilience Score

**Auto-created by the operator after (and during) a run - the pass/fail verdict plus a quantified score.**

```bash
kubectl get chaosresult backend-pod-delete-pod-delete -n my-app -o yaml
```

**Output Example:**
```yaml
status:
  experimentStatus:
    phase: Completed
    verdict: Pass
    probeSuccessPercentage: "100"
```

**Visual:**
```
Verdict: Pass   → steady-state (probes) held throughout the fault
Verdict: Fail   → a probe failed, or the app didn't recover in time
Verdict: Awaited→ still running

Resilience Score (ChaosCenter dashboard):
┌────────────────────────────────────────┐
│ Application: backend                     │
│  Total experiments run: 12                │
│  Passed: 10   Failed: 2                    │
│  Resilience Score: 83%                     │
└────────────────────────────────────────┘

Score is tracked over time per application - a rising
score across releases is a concrete resilience metric
you can report to stakeholders.
```

---

## Common Experiments Catalog

**A sample of the most frequently used ChaosHub experiments in real DevOps pipelines.**

| Experiment | Simulates |
|---|---|
| `pod-delete` | Kill a random pod (like Chaos Mesh PodChaos pod-kill) |
| `pod-cpu-hog` | CPU stress on a pod |
| `pod-memory-hog` | Memory stress on a pod |
| `pod-network-latency` | Add latency to pod network traffic |
| `pod-network-loss` | Packet loss on pod network traffic |
| `pod-network-partition` | Full network isolation of a pod |
| `disk-fill` | Fill up disk space inside a pod |
| `pod-io-stress` | Disk I/O stress |
| `node-drain` | Drain a Kubernetes node |
| `node-cpu-hog` | CPU stress at the node level |
| `container-kill` | Kill a specific container inside a pod |
| `kafka-broker-pod-failure` | Kill a Kafka broker pod specifically |
| `aws-ssm-chaos` | Run SSM-based chaos on an EC2 instance |
| `ec2-terminate-by-tag` | Terminate EC2 instances matching a tag |

**Visual:**
```
Kubernetes-level          Cloud-level              Platform-specific
┌──────────────┐         ┌──────────────┐        ┌──────────────────┐
│ pod-delete     │         │ ec2-terminate  │        │ kafka-broker-      │
│ node-drain      │         │ gcp-vm-stop     │        │ pod-failure          │
│ disk-fill        │         │ azure-instance- │        │ cassandra-pod-       │
│ cpu/memory-hog    │         │ stop             │        │ delete                 │
└──────────────┘         └──────────────┘        └──────────────────┘
```

---

## Environment Variables (Tunables)

**Every experiment is configured via env vars in the ChaosEngine - the same handful of tunables show up across most experiments.**

**Visual:**
```
Common across most experiments:
TOTAL_CHAOS_DURATION  → how long the experiment runs (seconds)
CHAOS_INTERVAL         → time between iterations (for repeated faults)
FORCE                  → true = SIGKILL immediately, false = graceful
RAMP_TIME               → delay before injection starts

Experiment-specific example (pod-network-latency):
NETWORK_LATENCY         → delay in ms to inject
JITTER                   → variability in ms
TARGET_CONTAINER          → which container in a multi-container pod
```

---

## Visual Summary

```
1. ChaosHub          → catalog of pre-built experiment definitions
2. ChaosExperiment    → installed template (from ChaosHub) in your cluster
3. ChaosEngine         → your specific run: target app + experiment + params
4. ChaosResult          → the verdict (Pass/Fail) + probe success %
5. Resilience Score      → aggregated trend across all runs, per application
```

---

This guide covers LitmusChaos's core CRDs - ChaosExperiment, ChaosEngine, and ChaosResult - plus the ChaosHub catalog they're built from, with visual representations of how they connect.