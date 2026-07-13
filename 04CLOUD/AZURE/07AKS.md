# Azure Kubernetes Service (AKS)
## (analogous to Amazon EKS)

AKS is Azure's fully managed Kubernetes service. Azure handles the control plane (API server, etcd, scheduler) for free — you only pay for the worker nodes (VMs). It integrates natively with ACR, Entra ID, Azure Monitor, and Azure networking.

---

## Core Concepts

### Cluster
A Kubernetes cluster consisting of a managed **control plane** (free) and one or more **node pools** (billed as VMs).

### Node Pool
A group of VMs with the same size and configuration. A cluster has at least one **system node pool** (for Kubernetes system pods) and can have multiple **user node pools** for workloads.

### Workload Identity (analogous to IRSA in EKS)
Pods get their own Azure managed identity — no credentials in environment variables. Uses OIDC federation between AKS and Entra ID.

---

## Creating a Cluster

```bash
# Create resource group
az group create --name myRG --location eastus

# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-count 2 \
  --node-vm-size Standard_D2s_v5 \
  --enable-managed-identity \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --attach-acr myRegistry \
  --generate-ssh-keys

# Get kubectl credentials
az aks get-credentials --resource-group myRG --name myAKSCluster

# Verify cluster access
kubectl get nodes
```

---

## Node Pools

```bash
# Add a user node pool (e.g., for GPU workloads)
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC6s_v3 \
  --node-taints sku=gpu:NoSchedule

# Scale a node pool
az aks nodepool scale \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --node-count 5

# Enable autoscaler on a node pool
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 10
```

---

## Workload Identity (no secrets in pods)

```bash
# Create a user-assigned managed identity
az identity create --name myAppIdentity --resource-group myRG

# Get the OIDC issuer URL
export OIDC_ISSUER=$(az aks show \
  --resource-group myRG \
  --name myAKSCluster \
  --query "oidcIssuerProfile.issuerUrl" -o tsv)

# Create federated credential (links Kubernetes SA to Azure identity)
az identity federated-credential create \
  --name myFederatedCredential \
  --identity-name myAppIdentity \
  --resource-group myRG \
  --issuer $OIDC_ISSUER \
  --subject system:serviceaccount:default:my-service-account \
  --audience api://AzureADTokenExchange

# Grant the identity access to a resource (e.g., Key Vault)
az role assignment create \
  --assignee $(az identity show -n myAppIdentity -g myRG --query principalId -o tsv) \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKeyVault
```

```yaml
# Kubernetes ServiceAccount annotated for workload identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: my-service-account
  namespace: default
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
---
# Pod using the annotated SA
apiVersion: v1
kind: Pod
metadata:
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: my-service-account
  containers:
  - name: app
    image: myregistry.azurecr.io/my-app:latest
```

---

## Common kubectl Commands (same as any Kubernetes)

```bash
# Deploy from manifest
kubectl apply -f deployment.yaml

# Check pod status
kubectl get pods -o wide

# View logs
kubectl logs <pod-name> --follow

# Execute into a pod
kubectl exec -it <pod-name> -- /bin/bash

# Port-forward for local testing
kubectl port-forward svc/my-service 8080:80

# View events (useful for debugging)
kubectl get events --sort-by='.lastTimestamp'
```

---

## Ingress (analogous to ALB Ingress Controller in EKS)

AKS supports multiple ingress options:

| Option | Description |
|--------|-------------|
| **Application Gateway Ingress Controller (AGIC)** | Azure-native L7 load balancer, WAF support |
| **NGINX Ingress Controller** | OSS option, install via Helm |
| **Azure Load Balancer** | L4, auto-created when `type: LoadBalancer` |

```bash
# Enable the Application Gateway add-on
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons ingress-appgw \
  --appgw-name myAppGateway \
  --appgw-subnet-cidr "10.225.0.0/16"
```

---

## Cluster Upgrades

```bash
# Check available Kubernetes versions
az aks get-upgrades --resource-group myRG --name myAKSCluster --output table

# Upgrade the cluster
az aks upgrade \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.30.0
```

---

## Monitoring

```bash
# Enable Azure Monitor / Container Insights
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

---

## Key Differences from AWS EKS

| Feature | AWS EKS | Azure AKS |
|---------|---------|-----------|
| Control plane cost | ~$0.10/hr | Free |
| Node identity | IRSA (IAM Roles for SA) | Workload Identity (OIDC) |
| Registry integration | ECR | ACR (`--attach-acr`) |
| Ingress | ALB Ingress Controller | AGIC / NGINX |
| Monitoring | CloudWatch Container Insights | Azure Monitor / Container Insights |
| Network plugin | VPC CNI | Azure CNI / Kubenet |
| Node OS upgrades | Managed node groups | Node pool OS image upgrades |