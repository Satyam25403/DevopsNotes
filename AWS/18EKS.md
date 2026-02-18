# Amazon EKS (Elastic Kubernetes Service)

Amazon EKS is AWS's fully managed Kubernetes service. It handles the Kubernetes **control plane** (master nodes) for you — you focus on deploying workloads, not managing the cluster infrastructure.

---

## Why Kubernetes? Why EKS?

**Kubernetes** is the industry-standard container orchestration platform. It handles:
- Scheduling containers across nodes
- Self-healing (restarting failed containers)
- Horizontal scaling
- Service discovery and load balancing
- Rolling updates and rollbacks
- Configuration and secret management

**EKS** gives you Kubernetes without managing the control plane:
- AWS runs, patches, and scales the Kubernetes API server and etcd
- Multi-AZ control plane out of the box
- Deep integration with AWS services (IAM, VPC, ELB, ECR, CloudWatch)
- Fully compatible with standard Kubernetes tooling (`kubectl`, Helm, ArgoCD)

---

## EKS vs ECS

| Feature | EKS | ECS |
|---------|-----|-----|
| Orchestration | Kubernetes | AWS-native |
| Portability | Multi-cloud (standard K8s) | AWS-specific |
| Learning curve | Steeper | Easier |
| Ecosystem | Massive (Helm, ArgoCD, Istio…) | Smaller |
| Control | More | Less |
| Best for | Teams who know K8s or need multi-cloud | Teams wanting simplicity on AWS |

---

## Core Kubernetes Concepts (with EKS context)

| Concept | Description |
|---------|-------------|
| **Node** | A worker machine (EC2 instance or Fargate) that runs your containers |
| **Pod** | The smallest deployable unit — one or more containers sharing network/storage |
| **Deployment** | Manages a set of identical Pods, handles rolling updates |
| **Service** | Stable network endpoint for a set of Pods (ClusterIP, NodePort, LoadBalancer) |
| **Ingress** | HTTP routing rules — maps hostnames/paths to Services |
| **Namespace** | Logical isolation within a cluster (e.g., `dev`, `staging`, `prod`) |
| **ConfigMap** | Store non-sensitive configuration as key-value pairs |
| **Secret** | Store sensitive data (passwords, tokens) — base64 encoded |
| **HPA** | Horizontal Pod Autoscaler — scales pods based on CPU/memory/custom metrics |
| **PersistentVolume** | Storage abstraction backed by EBS, EFS, etc. |

---

## Node Types

### Managed Node Groups (recommended)
AWS provisions, manages, and auto-updates EC2 worker nodes. You choose instance type and scaling settings.

### Self-Managed Nodes
You provision and manage EC2 instances yourself. More control, more work.

### Fargate
Serverless pods — no nodes to manage. AWS provisions compute per pod. Great for variable workloads.

---

## Getting Started

### 1. Install Tools

```bash
# kubectl — Kubernetes CLI
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && sudo mv kubectl /usr/local/bin/

# eksctl — EKS cluster management CLI (simplest way to create clusters)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin/

# Helm — Kubernetes package manager
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify
kubectl version --client
eksctl version
helm version
```

### 2. Create a Cluster

```bash
# Create cluster with managed node group (takes ~15 minutes)
eksctl create cluster \
  --name my-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed

# eksctl also configures kubectl automatically
kubectl get nodes
```

**Or with a config file (recommended for reproducibility):**
```yaml
# cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-cluster
  region: us-east-1

managedNodeGroups:
  - name: workers
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 1
    maxSize: 5
    privateNetworking: true  # nodes in private subnets
    iam:
      withAddonPolicies:
        cloudWatch: true
        albIngress: true
```

```bash
eksctl create cluster -f cluster.yaml
```

### 3. Configure kubectl

```bash
# Update kubeconfig (if not done automatically by eksctl)
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Verify connection
kubectl cluster-info
kubectl get nodes
```

---

## Deploying an Application

### Deployment + Service

```yaml
# app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: my-app
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:v1.0
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: production
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          readinessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer  # creates an AWS ALB/NLB automatically
```

```bash
kubectl apply -f app.yaml

# Watch deployment
kubectl get pods -w
kubectl get service my-app-service  # get the external LoadBalancer URL
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

## Working with ECR in EKS

EKS nodes need permission to pull images from ECR. Grant this by attaching the `AmazonEC2ContainerRegistryReadOnly` policy to the node IAM role (eksctl does this automatically if you use `--managed`).

```bash
# Authenticate kubectl to pull from ECR (handled automatically by EKS)
# Just use the full ECR image URI in your pod spec:
# image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

---

## Ingress with AWS Load Balancer Controller

For HTTP routing with a single ALB across multiple services:

```bash
# Install AWS Load Balancer Controller via Helm
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

---

## IAM Roles for Service Accounts (IRSA)

Give pods fine-grained AWS permissions without using instance-level IAM roles:

```bash
# 1. Create OIDC provider for your cluster
eksctl utils associate-iam-oidc-provider \
  --cluster my-cluster \
  --approve

# 2. Create IAM role linked to the Kubernetes service account
eksctl create iamserviceaccount \
  --name my-app-sa \
  --namespace default \
  --cluster my-cluster \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess \
  --approve

# 3. Reference the service account in your Deployment
# spec.template.spec.serviceAccountName: my-app-sa
```

---

## Common kubectl Commands

```bash
# Cluster info
kubectl cluster-info
kubectl get nodes
kubectl top nodes  # CPU/memory usage

# Pods
kubectl get pods -A  # all namespaces
kubectl get pods -n my-namespace
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f  # stream logs
kubectl logs <pod-name> --previous  # logs from crashed container
kubectl exec -it <pod-name> -- /bin/sh  # shell into container

# Deployments
kubectl get deployments
kubectl rollout status deployment/my-app
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app  # roll back

# Scale
kubectl scale deployment my-app --replicas=5

# Apply / delete
kubectl apply -f manifest.yaml
kubectl delete -f manifest.yaml
kubectl delete pod <pod-name>  # pod will be recreated by deployment

# Port forward for local testing
kubectl port-forward service/my-app-service 8080:80
```

---

## Cluster Management

```bash
# List clusters
eksctl get cluster

# Add a node group
eksctl create nodegroup \
  --cluster my-cluster \
  --name gpu-workers \
  --node-type p3.2xlarge \
  --nodes 2

# Scale node group
eksctl scale nodegroup \
  --cluster my-cluster \
  --name standard-workers \
  --nodes 4

# Update cluster version
eksctl upgrade cluster --name my-cluster --approve

# Delete cluster (tears down everything)
eksctl delete cluster --name my-cluster
```

---

## EKS Best Practices

- **Use private node groups** — nodes in private subnets, only the ALB is public.
- **Enable IRSA** instead of instance IAM roles for pod-level AWS permissions.
- **Use namespaces + RBAC** to isolate teams and environments.
- **Set resource requests and limits** on all containers — prevents resource starvation.
- **Use `readinessProbe` and `livenessProbe`** to let Kubernetes manage unhealthy pods automatically.
- **Enable Container Insights** via CloudWatch for enhanced pod and node monitoring.
- **Use Cluster Autoscaler or Karpenter** to automatically add/remove nodes based on pending pods.
- **Store secrets in AWS Secrets Manager** and sync to Kubernetes using External Secrets Operator — don't store plain secrets in etcd.
- **Pin image versions** — never use `latest` in production.