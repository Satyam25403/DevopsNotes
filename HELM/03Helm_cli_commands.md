# Helm CLI Commands

Comprehensive guide to Helm command-line interface.

## Essential Commands

### Create and Install
```bash
# Create new chart
helm create <chart-name>

# Install chart
helm install <release-name> <chart-path>
helm install myapp ./my-chart

# Install with custom values
helm install myapp ./my-chart -f custom-values.yaml
```

### List and Status
```bash
# List releases
helm list
helm list -a         # All (including deleted)
helm list --all-namespaces

# Release status
helm status <release-name>
helm get values <release-name>
helm get manifest <release-name>
```

### Upgrade and Rollback
```bash
# Upgrade release
helm upgrade <release-name> <chart-path>
helm upgrade myapp ./my-chart

# Rollback to previous revision
helm rollback <release-name>
helm rollback <release-name> <revision-number>

# Check revision history
helm history <release-name>
```

### Testing and Debugging
```bash
# Dry run (don't actually install)
helm install myapp --dry-run --debug ./my-chart

# Template rendering (local, no K8s connection)
helm template ./my-chart

# Lint chart for errors
helm lint ./my-chart
```

### Uninstall
```bash
# Uninstall release
helm uninstall <release-name>

# Keep history
helm uninstall <release-name> --keep-history
```

## Repository Management

```bash
# Add repository
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repositories
helm repo update

# List repositories
helm repo list

# Search hub
helm search hub wordpress

# Search repo
helm search repo bitnami/

# Remove repository
helm repo remove bitnami
```

## Package Management

```bash
# Package chart
helm package ./my-chart

# Download chart
helm pull bitnami/wordpress
helm pull bitnami/wordpress --untar

# Show chart info
helm show chart bitnami/wordpress
helm show values bitnami/wordpress
helm show readme bitnami/wordpress
```

## Advanced Options

```bash
# Install specific version
helm install myapp bitnami/wordpress --version 15.0.0

# Set values from command line
helm install myapp ./chart --set replicaCount=3,image.tag=v2.0

# Create namespace
helm install myapp ./chart --create-namespace -n dev

# Wait for resources
helm install myapp ./chart --wait --timeout 5m
```

## Quick Reference

| Command | Purpose |
|---------|---------|
| `helm create` | Create new chart |
| `helm install` | Deploy chart |
| `helm upgrade` | Update release |
| `helm rollback` | Revert to previous version |
| `helm list` | Show releases |
| `helm uninstall` | Remove release |
| `helm lint` | Validate chart |

---

This guide covers all essential Helm CLI commands for chart management.