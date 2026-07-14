# Helm Charts and Templating

Guide to Helm chart architecture, templating, and values management.

## Chart Architecture

### Chart Structure
```
my-chart/
├── Chart.yaml       # Metadata
├── values.yaml      # Default values
├── charts/          # Dependencies
└── templates/       # K8s manifests
    ├── deployment.yaml
    ├── service.yaml
    ├── _helpers.tpl
    └── NOTES.txt
```

## Templating System

### Reference Types

| Reference | Source | Example |
|-----------|--------|---------|
| `.Values.*` | values.yaml | `.Values.replicaCount` |
| `.Chart.*` | Chart.yaml | `.Chart.Name` |
| `.Release.*` | Runtime | `.Release.Name` |
| `{{ include }}` | _helpers.tpl | `{{ include "app.fullname" . }}` |

### Values.yaml
```yaml
replicaCount: 3
image:
  repository: nginx
  tag: "1.21"
service:
  type: NodePort
  port: 80
```

### Template Usage
```yaml
# deployment.yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

## Multi-Environment Deployment

### Environment-Specific Values

**values-dev.yaml:**
```yaml
replicaCount: 1
env: dev
service:
  port: 8080
```

**values-prod.yaml:**
```yaml
replicaCount: 5
env: prod
service:
  port: 80
```

### Deploy Different Environments
```bash
helm install dev-release ./chart -f values-dev.yaml
helm install prod-release ./chart -f values-prod.yaml
```

## Best Practices

1. Use meaningful variable names
2. Keep values.yaml simple
3. Use _helpers.tpl for reusable snippets
4. Document in NOTES.txt
5. Version your charts

---

This guide covers Helm charts, templating, and multi-environment deployments.