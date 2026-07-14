# Chaos Mesh Command Cheat Sheet

Quick reference for all Chaos Mesh commands and CRD patterns covered across the guides.

---

## Install & Setup

```bash
curl -sSL https://mirrors.chaos-mesh.org/latest/install.sh | bash
kubectl create ns chaos-mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm repo update
helm install chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock \
  --version 2.6.3
kubectl get pods -n chaos-mesh
kubectl get crds | grep chaos-mesh.org
kubectl port-forward -n chaos-mesh svc/chaos-dashboard 2333:2333
```

## Core Experiment Types (kubectl apply -f <file>.yaml)

```yaml
# PodChaos
action: pod-kill | pod-failure | container-kill

# NetworkChaos
action: delay | loss | duplicate | corrupt | bandwidth | partition

# StressChaos
stressors: { cpu: {workers, load}, memory: {workers, size} }

# IOChaos
action: latency | fault | attrOverride

# TimeChaos
timeOffset: "-1h"

# DNSChaos
action: error | random

# HTTPChaos
target: Request | Response, abort: true

# KernelChaos
failKernRequest: { callchain, failtype, probability }
```

## Targeting

```yaml
selector:
  namespaces: ["my-app"]
  labelSelectors:
    app: backend
mode: one | all | fixed | fixed-percent | random-max-percent
value: "30"      # used with fixed / fixed-percent
duration: "60s"  # always set this
```

## Workflows & Schedules

```bash
kubectl apply -f workflow.yaml       # Workflow CRD (Serial/Parallel/Suspend)
kubectl apply -f schedule.yaml       # Schedule CRD (cron-based recurring chaos)
kubectl get schedule <name> -n my-app
kubectl get podchaos -n my-app --selector=managed-by=<schedule-name>
```

## Monitoring & Debugging

```bash
kubectl get events -n my-app --field-selector involvedObject.kind=PodChaos
chaosctl logs
chaosctl debug <podchaos-name> -n my-app
kubectl describe podchaos <name> -n my-app
kubectl logs -n chaos-mesh deploy/chaos-controller-manager
```

## Safety / Abort

```bash
kubectl annotate podchaos <name> -n my-app experiment.chaos-mesh.org/pause=true
kubectl delete podchaos <name> -n my-app        # immediate abort
kubectl delete networkchaos <name> -n my-app
kubectl delete podchaos,networkchaos,stresschaos,iochaos,timechaos,dnschaos,httpchaos,kernelchaos,workflow,schedule --all --all-namespaces
```

## RBAC

```bash
chaosctl dashboard rbac-token -n my-app --role=viewer
chaosctl dashboard rbac-token -n my-app --role=manager
```

## Upgrade & Uninstall

```bash
helm repo update
helm upgrade chaos-mesh chaos-mesh/chaos-mesh -n chaos-mesh --version 2.7.0
helm uninstall chaos-mesh -n chaos-mesh
kubectl delete ns chaos-mesh
```

---

## Command Quick Index

| Command | Purpose |
|---|---|
| `helm install chaos-mesh ...` | Install Chaos Mesh control plane + daemon |
| `kubectl apply -f *.yaml` | Create any Chaos CRD (PodChaos, NetworkChaos, etc.) |
| `kubectl get <chaos-kind> -n ns` | Check status of a running experiment |
| `kubectl delete <chaos-kind> <name> -n ns` | Abort an experiment immediately |
| `chaosctl debug <name> -n ns` | Inspect raw injected fault (tc/iptables/eBPF) |
| `chaosctl dashboard rbac-token` | Generate scoped dashboard access token |
| `kubectl port-forward svc/chaos-dashboard` | Access the web dashboard |

---

This cheat sheet summarizes all Chaos Mesh commands from the basics, experiment types, workflows/scheduling, dashboard/observability, production safety, and operations guides.