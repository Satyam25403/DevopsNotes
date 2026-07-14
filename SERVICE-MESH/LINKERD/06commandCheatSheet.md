# Linkerd Command Cheat Sheet

Quick reference for all Linkerd commands covered across the guides.

---

## Install & Setup

```bash
curl --proto '=https' --tlsv1.2 -sSf https://run.linkerd.io/install | sh
linkerd version
linkerd check --pre
linkerd install --crds | kubectl apply -f -
linkerd install | kubectl apply -f -
linkerd install --ha | kubectl apply -f -
linkerd check
linkerd viz install | kubectl apply -f -
linkerd viz dashboard &
```

## Injection

```bash
kubectl annotate namespace my-app linkerd.io/inject=enabled
kubectl rollout restart deployment -n my-app
kubectl get deploy backend -o yaml | linkerd inject - | kubectl apply -f -
linkerd inject deployment.yaml | kubectl apply -f -
linkerd inject deployment.yaml --manual > injected.yaml
kubectl get deploy backend -o yaml | linkerd uninject - | kubectl apply -f -
linkerd check --proxy
```

## Traffic Management

```bash
linkerd profile --open-api swagger.json backend > backend-profile.yaml
linkerd profile --proto backend.proto backend > backend-profile.yaml
kubectl apply -f backend-canary.yaml   # TrafficSplit / HTTPRoute
linkerd viz stat ts/backend-canary -n my-app
```

## Observability

```bash
linkerd viz stat deploy -n my-app
linkerd viz stat pods -n my-app
linkerd viz stat ns
linkerd viz routes deploy/backend -n my-app
linkerd viz top deploy/backend -n my-app
linkerd viz tap deploy/backend -n my-app
linkerd viz tap deploy/backend --path /api/orders --method POST -n my-app
linkerd viz edges deploy -n my-app
linkerd viz install --set grafana.enabled=true | kubectl apply -f -
```

## Security & Policy

```bash
linkerd install --identity-trust-anchors-file ca.crt \
  --identity-issuer-certificate-file issuer.crt \
  --identity-issuer-key-file issuer.key | kubectl apply -f -
linkerd install --default-inbound-policy deny | kubectl apply -f -
linkerd check --proxy
linkerd viz edges deploy -n my-app
```

## Upgrades & Multicluster

```bash
linkerd upgrade --crds | kubectl apply -f -
linkerd upgrade | kubectl apply -f -
linkerd multicluster install | kubectl apply -f -
linkerd --context=west multicluster link --cluster-name west | kubectl --context=east apply -f -
linkerd multicluster check
linkerd multicluster gateways
```

## Diagnostics & Uninstall

```bash
linkerd diagnostics proxy-metrics deploy/backend -n my-app
linkerd diagnostics endpoints backend.my-app.svc.cluster.local:8080
linkerd diagnostics controller-metrics
kubectl logs backend-xyz -c linkerd-proxy -n my-app

linkerd viz uninstall | kubectl delete -f -
linkerd multicluster uninstall | kubectl delete -f -
linkerd uninstall | kubectl delete -f -
```

---

## Command Quick Index

| Command | Purpose |
|---|---|
| `linkerd check` | Verify control plane health |
| `linkerd check --proxy` | Verify data plane / proxy health |
| `linkerd check --pre` | Pre-install cluster validation |
| `linkerd inject` | Manually inject proxy sidecar into manifest |
| `linkerd uninject` | Remove proxy sidecar from manifest |
| `linkerd profile` | Generate a ServiceProfile from OpenAPI/proto |
| `linkerd viz stat` | Aggregate golden metrics |
| `linkerd viz routes` | Per-route metrics |
| `linkerd viz top` | Live traffic view |
| `linkerd viz tap` | Live request/response capture |
| `linkerd viz edges` | mTLS status between services |
| `linkerd viz dashboard` | Launch web dashboard |
| `linkerd upgrade` | Upgrade control plane |
| `linkerd multicluster` | Cross-cluster mesh federation |
| `linkerd diagnostics` | Raw/low-level debugging data |

---

This cheat sheet summarizes all Linkerd commands from the basics, injection, traffic management, observability, security, and production operations guides.