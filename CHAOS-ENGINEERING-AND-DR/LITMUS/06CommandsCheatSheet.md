# LitmusChaos Command Cheat Sheet

Quick reference for all LitmusChaos commands and CRD patterns covered across the guides.

---

## Install & Setup

```bash
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm repo update
kubectl create ns litmus
helm install chaos litmuschaos/litmus -n litmus
kubectl get svc -n litmus litmusportal-frontend-service
kubectl get pods -n litmus
```

## litmusctl CLI

```bash
curl -O https://raw.githubusercontent.com/litmuschaos/litmusctl/master/install.sh && sh install.sh
litmusctl config set-account --endpoint=<url> --username=<user> --password=<pass>
litmusctl create project --name my-project
litmusctl get projects
litmusctl get agents --project-id <id>
litmusctl create chaos-hub --name my-hub --repo-url <git-url> --repo-branch main
litmusctl create workflow --project-id <id> --cluster-id <agent-id> --workflow-manifest workflow.yaml
```

## Core CRDs

```bash
# Install a ChaosExperiment from ChaosHub
kubectl apply -f https://hub.litmuschaos.io/api/chaos/3.0.0?file=charts/generic/pod-delete/experiment.yaml -n my-app
kubectl get chaosexperiments -n my-app

# Run via ChaosEngine
kubectl apply -f chaosengine.yaml
kubectl get chaosengine <name> -n my-app
kubectl patch chaosengine <name> -n my-app --type merge --patch '{"spec":{"engineState":"stop"}}'
kubectl delete chaosengine <name> -n my-app

# Check the result
kubectl get chaosresult <name> -n my-app -o yaml
kubectl get chaosresult <name> -n my-app -o jsonpath='{.status.experimentStatus.verdict}'
```

## Common ChaosEngine Env Vars

```yaml
TOTAL_CHAOS_DURATION: "60"
CHAOS_INTERVAL: "10"
FORCE: "false"
PODS_AFFECTED_PERC: "10"
RAMP_TIME: "0"
```

## Probes (steady-state validation)

```yaml
probe:
  - name: "..."
    type: httpProbe | cmdProbe | k8sProbe | promProbe
    mode: SOT | EOT | Edge | Continuous | OnChaos
```

## Workflows (Argo-based)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Workflow
spec:
  entrypoint: custom-chaos
  templates:
    - name: custom-chaos
      steps:
        - - name: step-a   # sequential stage
        - - name: step-b
            # name: step-c  (same list = parallel with step-b)
```

## Troubleshooting

```bash
kubectl get pods -n litmus
kubectl get pods -n <agent-namespace>
kubectl describe chaosengine <name> -n my-app
kubectl logs -n my-app -l name=pod-delete
kubectl get chaosresult <name> -n my-app -o yaml
```

## Upgrade & Uninstall

```bash
helm upgrade chaos litmuschaos/litmus -n litmus --version 3.x.x
kubectl delete chaosengine --all --all-namespaces
kubectl delete chaosexperiment --all --all-namespaces
helm uninstall chaos -n litmus
kubectl delete ns litmus
```

---

## Command Quick Index

| Command | Purpose |
|---|---|
| `helm install chaos litmuschaos/litmus` | Install ChaosCenter control plane |
| `kubectl apply -f <chaosexperiment>.yaml` | Install an experiment template from ChaosHub |
| `kubectl apply -f <chaosengine>.yaml` | Trigger an actual experiment run |
| `kubectl get chaosresult <name> -o yaml` | Check pass/fail verdict + probe results |
| `kubectl patch chaosengine ... engineState:stop` | Abort a running experiment |
| `litmusctl create workflow` | Trigger a multi-step workflow from CI/scripts |
| `litmusctl get agents` | List connected clusters |

---

This cheat sheet summarizes all LitmusChaos commands from the basics, core CRDs/ChaosHub, probes, workflows/scheduling, production safety, and operations guides.