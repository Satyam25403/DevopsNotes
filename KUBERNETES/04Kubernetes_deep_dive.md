# Advanced Kubernetes Operations

Complete guide to KIND cluster management, pods, deployments, services, environment variables, ConfigMaps, Secrets, resource management, and autoscaling.

## Table of Contents
- [KIND Cluster Management](#kind-cluster-management)
- [Working with Pods](#working-with-pods)
- [Deployments](#deployments)
- [Services Deep Dive](#services-deep-dive)
- [Environment Variables](#environment-variables)
- [ConfigMaps](#configmaps)
- [Secrets](#secrets)
- [Resource Requests and Limits](#resource-requests-and-limits)
- [Horizontal Pod Autoscaler (HPA)](#horizontal-pod-autoscaler-hpa)

---

## KIND Cluster Management

### Creating Clusters

**Basic single-node cluster:**
```bash
kind create cluster --name=first-cluster
```

**Multi-node cluster with config:**
```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: first-cluster
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

**Create from config:**
```bash
# Use config file
kind create cluster --config kind-config.yaml

# Name cluster at runtime
kind create cluster --name my-cluster --config kind-config.yaml
```

### Understanding kindest/node

**What you see with `docker ps`:**
```bash
docker ps
# CONTAINER ID   IMAGE                  
# abc123def456   kindest/node:v1.27.0
```

**What is kindest/node?**
- KIND's bootstrap container
- NOT a Kubernetes pod
- Docker container simulating Kubernetes node
- Contains: kubelet, container runtime, etc.
- Think of it as the VM that runs Kubernetes

**Analogy:**
- **kindest/node container** = Stage and lighting (enables the show)
- **Kubernetes pods** = Actors performing scenes
- When stage is built, no actors until you deploy pods

**Why it doesn't show in `kubectl get pods`:**
- kindest/node is OUTSIDE Kubernetes
- It hosts the cluster
- Kubernetes doesn't "see" it as a pod
- It's the infrastructure, not the workload

### Inspecting KIND Nodes

**Enter the node container:**
```bash
# Get container ID
docker ps

# Enter container
docker exec -it <container-id> sh

# Switch to bash
bash

# View Kubernetes components
cd /etc/kubernetes/manifests
ls
# Output: etcd.yaml, kube-apiserver.yaml, kube-controller-manager.yaml, kube-scheduler.yaml
```

### Cluster Operations (Windows)

```bash
# List all clusters
kind get clusters

# View current cluster
kubectl config current-context

# Switch between clusters
kubectl config use-context kind-<cluster-name>

# Example
kubectl config use-context kind-dev-cluster
```

### Deleting Clusters

```bash
# Delete specific cluster
kind delete cluster --name <cluster-name>

# Example
kind delete cluster --name dev-cluster
```

---

## Working with Pods

### Imperative Pod Management

**Create pod:**
```bash
# Basic pod creation
kubectl run nginx-pod --image=nginx

# With specific image version
kubectl run nginx-pod --image=nginx:1.21
```

**View pods:**
```bash
# List running pods
kubectl get pods

# Static logs
kubectl logs <pod-name>

# Real-time logs (follow)
kubectl logs -f <pod-name>

# Describe pod (metadata)
kubectl describe pod <pod-name>

# Get YAML/JSON
kubectl get pod <pod-name> -o yaml
kubectl get pod <pod-name> -o json
```

### Declarative Pod Management (YAML)

**Basic pod YAML:**
```yaml
# pod.yaml
# Four top-level fields: apiVersion, kind, metadata, spec
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod2
  labels:
    env: test
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

**Apply pod:**
```bash
# Create from YAML
kubectl create -f pod.yaml

# Apply (create or update)
kubectl apply -f pod.yaml
```

### Dry Run (Virtual Execution)

**Generate YAML without creating:**
```bash
# Dry run
kubectl run nginx --image=nginx --dry-run=client

# Extract YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Create from extracted YAML
kubectl apply -f pod.yaml
```

**Works for any Kubernetes object:**
```bash
# Deployment
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# Service
kubectl expose deployment nginx --port=80 --dry-run=client -o yaml > service.yaml
```

### Pod Operations

**Edit running pod:**
```bash
# Edit YAML in editor
kubectl edit pod <pod-name>

# Or edit file and apply
vim pod.yaml
kubectl apply -f pod.yaml
```

**View pod labels:**
```bash
kubectl get pods --show-labels

# Wide output (includes node, IP)
kubectl get pods -o wide
```

**Execute into pod:**
```bash
# Interactive shell
kubectl exec -it <pod-name> -- bash
kubectl exec -it <pod-name> -- sh

# Run single command
kubectl exec <pod-name> -- ls -la
kubectl exec <pod-name> -- env
```

**Delete pod:**
```bash
kubectl delete pod <pod-name>
```

---

## Deployments

### What is a Deployment?

**Deployment:**
- Replica placement strategy for pods
- Uses ReplicaSet internally
- Ensures desired number of pods always running
- Provides rolling updates (zero downtime)

**Rolling update:**
- Updates pods one at a time
- Ensures availability during updates
- Old pods remain until new ones ready
- Automatic rollback capability

### Imperative Deployment Management

**Create deployment:**
```bash
# Create with replicas
kubectl create deployment nginx-deploy --image=nginx --replicas=3

# View deployments
kubectl get deployment

# Describe deployment (metadata + history)
kubectl describe deployment nginx-deploy

# Delete deployment
kubectl delete deployment nginx-deploy
```

**Scale deployment:**
```bash
kubectl scale deployment nginx-deploy --replicas=5
```

**Edit deployment:**
```bash
# Opens YAML in editor
kubectl edit deployment nginx-deploy
# Save to apply changes immediately
```

**Generate YAML:**
```bash
kubectl create deployment nginx-deploy \
  --image=nginx \
  --replicas=3 \
  --dry-run=client -o yaml > deployment.yaml
```

### Understanding Deployment Fields

**Get field explanations:**
```bash
# Quick reference
kubectl explain deployment
kubectl explain deployment.spec
kubectl explain deployment.spec.template
```

**How new pods are created:**
- Template in `spec.template` field
- Pod goes down → new pod created from template
- Template defines: labels, containers, resources

### Rolling Updates and Rollbacks

**Update image:**
```bash
# Change image version
kubectl set image deployment/nginx-deploy nginx=nginx:1.22

# Check rollout status
kubectl rollout status deployment nginx-deploy

# View rollout history
kubectl rollout history deployment nginx-deploy
```

**Rollback:**
```bash
# Undo last rollout
kubectl rollout undo deployment nginx-deploy

# Verify rollback
kubectl get pods
kubectl describe pod <pod-name>
# Shows older image version (e.g., nginx:1.21)
```

**Example workflow:**
```bash
# 1. Initial deployment (nginx:1.21)
kubectl create deployment nginx-deploy --image=nginx:1.21 --replicas=3

# 2. Update to newer version
kubectl set image deployment/nginx-deploy nginx=nginx:1.22

# 3. Verify update
kubectl get pods
kubectl describe pod <pod-name>  # Shows nginx:1.22

# 4. Rollback if needed
kubectl rollout undo deployment nginx-deploy

# 5. Verify rollback
kubectl describe pod <pod-name>  # Shows nginx:1.21
```

**Restart deployment:**
```bash
# Rolling restart (zero downtime)
kubectl rollout restart deployment nginx-deploy
```

---

## Services Deep Dive

### Service Types

1. **ClusterIP** - Internal access only
2. **NodePort** - External access on specific port
3. **LoadBalancer** - Cloud load balancer endpoint
4. **ExternalName** - DNS mapping to external domain

### Port Terminology

**Understanding the port flow:**
```
User (Browser)
    ↓
NodePort (30080) - Where user accesses
    ↓
Service Port (80) - Service's virtual port
    ↓
TargetPort (80) - Pod/container port
```

**Definitions:**
- **NodePort:** External access point (30000-32767 range)
- **Service Port:** Internal cluster port (load balancer)
- **TargetPort:** Container's actual listening port

**Get service YAML:**
```bash
kubectl get service <service-name> -o yaml
```

### ClusterIP Service

**Characteristics:**
- Internal cluster access only
- Two ports: port and targetPort
- Not accessible from outside (unless port-forwarded)

**Example deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
```

**Apply and watch:**
```bash
kubectl apply -f deployment.yaml
kubectl get deploy --watch
kubectl get pods -o wide
```

**Test without service:**
```bash
curl localhost:80
# Won't work - no access to pod
```

**Create ClusterIP service:**
```bash
# Expose deployment
kubectl expose deployment nginx-deployment --port=80

# Or specify target port
kubectl expose deployment nginx-deployment --port=80 --target-port=80

# Default type is ClusterIP
# Explicit: --type=ClusterIP
```

**Service details:**
```bash
kubectl get service
kubectl describe service nginx-deployment
```

**Port forwarding for access:**
```bash
# Forward pod port
kubectl port-forward pod/<pod-name> 8080:80

# Forward service port (better)
kubectl port-forward svc/nginx-deployment 8080:80

# Access at http://localhost:8080
```

**Minikube access:**
```bash
minikube service list
minikube service nginx-deployment --url
```

### NodePort Service

**Expose with NodePort:**
```bash
kubectl expose deployment nginx-deployment --type=NodePort --port=30080
```

**NodePort YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80          # Service internal port
    nodePort: 30080   # External access port
    targetPort: 80    # Pod container port
```

**Traffic flow:**
```
[External User]
    ↓
http://<NodeIP>:30080    # NodePort (external)
    ↓
Service (port 80)        # Internal service port
    ↓
Pod container (port 80)  # TargetPort
```

**Apply service:**
```bash
kubectl apply -f service.yaml
kubectl get service
kubectl describe service nginx-nodeport
kubectl get ep  # Get endpoints
```

**Delete service:**
```bash
kubectl delete service nginx-nodeport
```

**How services find pods:**

**Service selector matches pod labels:**
```yaml
# Service YAML
spec:
  selector:
    app: sample1  # Checks this

# Pod YAML
metadata:
  labels:
    app: sample1  # Must match
```

### KIND Port Forwarding Issue

**Why port-forward is needed in KIND:**

**The problem:**
- KIND nodes are Docker containers
- Private Docker network (172.18.0.x)
- NodePorts exposed inside container, not on host
- `localhost` can't access NodePorts directly

**Solution 1: Port Forward (Simple)**
```bash
kubectl port-forward service/nginx-service 8080:80
```

**Traffic flow:**
```
localhost:8080
  → API Server
    → kube-proxy
      → ClusterIP/NodePort
        → Pod
```

**Key point:** Port-forward works for ALL service types (ClusterIP, NodePort, LoadBalancer)

**Solution 2: extraPortMappings (Better)**

**Why this works:**
- Maps container port to host port
- Makes NodePort accessible on localhost
- No port-forward needed

**Cluster config with port mapping:**
```yaml
# cluster-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: first-cluster

nodes:
  - role: control-plane
    extraPortMappings:
    - containerPort: 30080
      hostPort: 30080
      protocol: TCP
    - containerPort: 30081
      hostPort: 30081
      protocol: TCP
  
  - role: worker
    extraPortMappings:
    - containerPort: 30080
      hostPort: 30082  # Different host port
      protocol: TCP
```

**Important:**
- Each node needs unique hostPorts
- Can't have two services on same host port
- containerPort can repeat across nodes
- hostPort must be unique across ALL nodes

**Create cluster:**
```bash
kind create cluster --name my-cluster --config cluster-config.yaml
```

**Combined deployment and service:**
```yaml
# deployment-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**Apply and access:**
```bash
kubectl apply -f deployment-service.yaml
kubectl get pods
kubectl get service

# Access directly (no port-forward needed)
curl http://localhost:30080
```

### LoadBalancer Service

**Requires cloud provider:**
- AWS, GCP, Azure
- Cloud control manager needed
- Cannot test locally with KIND/Minikube

**YAML:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80  # Only port needed
```

**Why only port field:**
- LoadBalancer exposes on standard ports (80/443)
- HTTP (80) or HTTPS (443)
- Cloud provider handles external IP
- Recommended for production websites

---

## Environment Variables

### What are Environment Variables?

**Purpose:**
- Inject configuration into containers
- Not just code - inject into running containers
- Key-value pairs in manifest

### Basic Environment Variables

**Deployment with env vars:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginxdeploy
  name: nginxdeploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginxdeploy
  template:
    metadata:
      labels:
        app: nginxdeploy
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        env:
        - name: APP_NAME
          value: "kubernetes-101"
        - name: PORT
          value: "3000"
        ports:
        - containerPort: 3000
```

**Important:**
- Environment variables at container level
- Inside `containers` field
- Key-value pairs

**Application usage:**
```javascript
// Node.js example
const port = process.env.PORT || 3000;
const appName = process.env.APP_NAME || 'default';

console.log(`${appName} running on port ${port}`);
// Output: kubernetes-101 running on port 3000
```

### Verify Environment Variables

**Check in running pod:**
```bash
# Get pods
kubectl get pods

# Method 1: Interactive shell
kubectl exec -it <pod-name> -- sh
printenv

# Method 2: Direct command
kubectl exec -it <pod-name> -- printenv
```

**Output:**
```
APP_NAME=kubernetes-101
PORT=3000
...
```

---

## ConfigMaps

### Why ConfigMaps?

**Problem:**
- 100s of environment variables
- Shared across deployments (app, frontend, backend)
- Repeating in every YAML manifest

**Solution:**
- Create one ConfigMap with all env vars
- Reference ConfigMap in multiple deployments
- Centralized management

### Creating ConfigMaps

**Imperative:**
```bash
# Create with literals
kubectl create configmap app-config \
  --from-literal=FIRST_NAME=piyush \
  --from-literal=LAST_NAME=sachdeva

# View
kubectl get configmap
kubectl get cm  # Short form

# Describe
kubectl describe configmap app-config
```

**Declarative (YAML):**
```bash
# Generate YAML
kubectl create configmap app-config \
  --from-literal=FIRST_NAME=piyush \
  --from-literal=LAST_NAME=sachdeva \
  --dry-run=client -o yaml > configmap.yaml
```

**ConfigMap YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  FIRST_NAME: piyush
  LAST_NAME: sachdeva
  DATABASE_URL: mongodb://db:27017
  NODE_ENV: production
```

### Using ConfigMaps in Deployments

**Method 1: Reference each key (verbose):**
```yaml
spec:
  containers:
  - name: nginx
    image: nginx:latest
    env:
    - name: FIRST_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: FIRST_NAME
    - name: LAST_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: LAST_NAME
```

**Problem:**
- 4 lines per environment variable
- Not scalable for many variables

**Method 2: Reference entire ConfigMap (better):**
```yaml
spec:
  containers:
  - name: nginx
    image: nginx:latest
    envFrom:
    - configMapRef:
        name: app-config
```

**Advantages:**
- All ConfigMap keys become env vars
- Single reference
- Easy to add more variables

**Apply and verify:**
```bash
kubectl apply -f deployment.yaml
kubectl get pods
kubectl exec -it <pod-name> -- printenv
```

### Multiple ConfigMaps

**Reference multiple ConfigMaps:**
```yaml
spec:
  containers:
  - name: nginx
    envFrom:
    - configMapRef:
        name: app-config
    - configMapRef:
        name: db-config
```

**Or mix methods:**
```yaml
spec:
  containers:
  - name: nginx
    env:
    - name: FIRST_NAME
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: FIRST_NAME
    - name: SERVER_NAME
      valueFrom:
        configMapKeyRef:
          name: server-config
          key: SERVER_NAME
```

**Benefits:**
- Centralized configuration
- Reusable across deployments
- Easy updates (update ConfigMap, restart pods)

---

## Secrets

### ConfigMap vs Secret

**ConfigMap:**
- Visible to naked eye
- Plain text
- General configuration

**Secrets:**
- Encoded (base64)
- Sensitive data
- API keys, passwords, SSH keys, certificates

### Secret Types

- **Opaque** - Generic key-value (default)
- **kubernetes.io/basic-auth** - Username/password
- **kubernetes.io/ssh-auth** - SSH keys
- **kubernetes.io/tls** - TLS certificates
- **kubernetes.io/dockerconfigjson** - Docker registry credentials

**Important:**
- Secrets are **encoded**, not encrypted
- base64 encoding (reversible)
- Use 3rd party secret managers for encryption (Vault, AWS Secrets Manager)

### Generic Secrets (Opaque)

**Create secret:**
```bash
kubectl create secret generic db-secret \
  --from-literal=password=hellopass

# View secrets
kubectl get secret

# Describe (won't show value)
kubectl describe secret db-secret

# Get YAML (shows base64 encoded)
kubectl get secret db-secret -o yaml
```

**Encode/Decode:**
```bash
# Decode secret
echo "aGVsbG9wYXNz" | base64 --decode
# Output: hellopass

# Encode text
echo -n "hellopass" | base64
# Output: aGVsbG9wYXNz

# Note: Trailing "Cg==" indicates line break "\n"
echo "hellopass" | base64
# Output: aGVsbG9wYXNzCg==
```

### Docker Registry Secret

**Use case:** Pull images from private Docker repository

**Create secret:**
```bash
# Create access token at hub.docker.com
# Account → Settings → Personal Access Tokens

kubectl create secret docker-registry docker-secret \
  --docker-email=user@example.com \
  --docker-username=username \
  --docker-password=<PAT-token> \
  --docker-server=https://index.docker.io/v2/
```

**View secret:**
```bash
kubectl get secrets
kubectl describe secret docker-secret  # Won't show credentials
kubectl get secret docker-secret -o yaml  # Shows encoded dockerconfigjson
```

**Decode dockerconfigjson:**
```bash
# Copy .dockerconfigjson value
echo "<base64-string>" | base64 --decode
```

**Use in deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: private-app
  template:
    metadata:
      labels:
        app: private-app
    spec:
      containers:
      - name: app
        image: username/my-private-repo:latest
        ports:
        - containerPort: 3000
      imagePullSecrets:
      - name: docker-secret  # Reference secret here
```

**What happens without secret:**
```bash
kubectl apply -f deployment.yaml
kubectl get pods
# Status: ErrImagePull or ImagePullBackOff
```

**With secret:**
```bash
kubectl apply -f deployment.yaml
kubectl get pods
# Status: Running ✅
```

**Alternative:** Mount secret as volume
```yaml
volumeMounts:
- name: secret-volume
  mountPath: /etc/secrets
volumes:
- name: secret-volume
  secret:
    secretName: docker-secret
```

---

## Resource Requests and Limits

### Why Resource Management?

**OOM (Out of Memory):**
- Memory utilization exceeds available memory
- Can kill existing pods/workloads
- Can crash entire node

**CPU Issues:**
- CPU throttling
- Performance degradation
- Slow response times

### Defining Resources

**Pod with resources:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stress-test
spec:
  containers:
  - name: stress-test
    image: polinux/stress
    command: ["stress"]
    resources:
      requests:
        cpu: "1"           # 1 CPU core
        memory: "50Mi"     # 50 megabytes initially
      limits:
        cpu: "1"           # Maximum 1 CPU core
        memory: "300Mi"    # Maximum 300 megabytes
    args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "350M", "--timeout", "10s"]
```

**Understanding fields:**
- **requests:** Initial allocation (guaranteed)
- **limits:** Maximum allowed (hard cap)

**Test OOM Kill:**
```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl describe pod stress-test
```

**Output:**
```
Status: Failed
Reason: OOMKilled
Message: Container exceeded memory limit (350M > 300M)
```

**Fix:**
```yaml
args: ["--cpu", "1", "--vm", "1", "--vm-bytes", "250M", "--timeout", "10s"]
# 250M < 300M limit ✅
```

**Apply and verify:**
```bash
kubectl apply -f pod.yaml
kubectl get pods
# Status: Running or Completed ✅
```

### CPU Units

```
1 CPU = 1000m (millicores)
0.5 CPU = 500m
0.1 CPU = 100m
```

**Example:**
```yaml
resources:
  requests:
    cpu: "200m"  # 0.2 CPU core
  limits:
    cpu: "500m"  # 0.5 CPU core
```

---

## Horizontal Pod Autoscaler (HPA)

### Autoscaling Types

**HPA (Horizontal Pod Autoscaler):**
- Scales pods horizontally (more replicas)
- Existing pods kept running
- Non-disruptive (widely used)
- Based on workload metrics

**VPA (Vertical Pod Autoscaler):**
- Upgrades to bigger pods (more CPU/memory)
- Deletes previous pods
- Disruptive (traffic redirection needed)

**Cluster Autoscaler:**
- Scales infrastructure (nodes)
- Adds more worker nodes
- Cloud provider integration

**KEDA (Event-Driven Autoscaling):**
- Third-party tool
- Custom metrics (Kafka queue, DB connections)
- Event-based scaling

### Metrics Server

**What is Metrics Server:**
- Collects node-level metrics
- Pod metrics, container runtime metrics
- Exposes via Metrics API to API server
- Used for scaling decisions and monitoring

**Metrics collected:**
- CPU usage
- Memory usage
- Network I/O
- Disk I/O

**Commands enabled:**
```bash
kubectl top nodes
kubectl top pods
```

### Enable Metrics Server

**Minikube:**
```bash
# Enable addon
minikube addons enable metrics-server

# Verify
kubectl top nodes
kubectl top pods
```

**KIND:**
```bash
# Download from GitHub
# https://github.com/kubernetes-sigs/metrics-server

kubectl apply -f metrics-server.yaml

# Patch for insecure TLS (local dev)
kubectl edit deployment metrics-server -n kube-system
```

**Add flag to args:**
```yaml
spec:
  containers:
  - name: metrics-server
    args:
    - --cert-dir=/tmp
    - --secure-port=4443
    - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
    - --kubelet-insecure-tls  # Add this
```

**Why `--kubelet-insecure-tls`:**
- Skips TLS verification for Kubelet
- Local clusters (KIND/Minikube) use self-signed certs
- Not signed by trusted CA
- Required for local development

**Apply to correct namespace:**
```bash
kubectl apply -f metrics-server.yaml -n kube-system
```

**Restart deployment:**
```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

**Verify:**
```bash
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes
kubectl top pods
```

### HPA Demo

**Test pod with resources:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-app
  labels:
    app: deploy-nginx
spec:
  containers:
  - name: hello-app
    image: username/my-app:latest
    resources:
      requests:
        cpu: "200m"  # 0.2 core
      limits:
        cpu: "500m"  # 0.5 core
    ports:
    - containerPort: 3000
  imagePullSecrets:
  - name: docker-secret
```

**Create NodePort service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
spec:
  type: NodePort
  selector:
    app: deploy-nginx
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080
```

**Apply:**
```bash
kubectl apply -f pod.yaml
kubectl apply -f service.yaml
kubectl get service
kubectl get ep  # Endpoints
```

**Access service:**
```bash
# KIND (with extraPortMappings)
curl http://localhost:30080

# Minikube
minikube service svc-nginx

# Port forward
kubectl port-forward svc/svc-nginx 8080:80
curl http://localhost:8080
```

**Create deployment for autoscaling:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-nginx
  labels:
    app: deploy-nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: deploy-nginx
  template:
    metadata:
      labels:
        app: deploy-nginx
    spec:
      containers:
      - name: hello-app
        image: username/my-app:latest
        resources:
          requests:
            cpu: "200m"
          limits:
            cpu: "500m"
        ports:
        - containerPort: 3000
      imagePullSecrets:
      - name: docker-secret
```

**Deploy:**
```bash
# Delete test pod
kubectl delete pod hello-app

# Apply deployment
kubectl apply -f deployment.yaml
kubectl get pods
kubectl get service  # Service should attach to new pods
kubectl describe service svc-nginx
```

**Create HPA:**
```bash
# Auto-scale based on CPU
kubectl autoscale deployment deploy-nginx \
  --min=1 \
  --max=3 \
  --cpu-percent=15

# View HPA
kubectl get hpa

# Describe HPA
kubectl describe hpa deploy-nginx
```

**Monitor autoscaling (3 terminals):**

**Terminal 1:**
```bash
kubectl get pods -w
```

**Terminal 2:**
```bash
kubectl get hpa -w
```

**Terminal 3:**
```bash
kubectl get deployment -w
```

**Load test (Terminal 4):**
```bash
# Generate load
while sleep 0.5; do
  curl http://localhost:8080
done
```

**What happens:**
- CPU usage increases
- HPA detects threshold crossed (>15%)
- Scales deployment from 2 → 3 pods
- Load distributes across pods
- CPU usage drops
- After cooldown, scales down

**Inspect scaling:**
```bash
kubectl describe hpa deploy-nginx
kubectl describe deployment deploy-nginx

# Output shows:
# - Current CPU usage
# - Target CPU percentage
# - Current replicas
# - Desired replicas
# - Scaling events
```

---

## Quick Reference

### KIND Commands

```bash
kind create cluster
kind create cluster --config config.yaml
kind get clusters
kind delete cluster --name <n>
```

### Pod Commands

```bash
kubectl run <n> --image=<img>
kubectl get pods
kubectl logs <n>
kubectl exec -it <n> -- bash
kubectl delete pod <n>
```

### Deployment Commands

```bash
kubectl create deployment <n> --image=<img> --replicas=3
kubectl get deployment
kubectl scale deployment <n> --replicas=5
kubectl rollout status deployment <n>
kubectl rollout undo deployment <n>
```

### Service Commands

```bash
kubectl expose deployment <n> --port=80
kubectl get svc
kubectl port-forward svc/<n> 8080:80
```

### ConfigMap/Secret Commands

```bash
kubectl create configmap <n> --from-literal=KEY=value
kubectl create secret generic <n> --from-literal=KEY=value
kubectl get configmap
kubectl get secret
```

### HPA Commands

```bash
kubectl autoscale deployment <n> --min=1 --max=5 --cpu-percent=50
kubectl get hpa
kubectl top pods
kubectl top nodes
```

---

This comprehensive guide covers advanced Kubernetes operations including KIND cluster management, pods, deployments, services, environment variables, ConfigMaps, Secrets, resource management, and horizontal pod autoscaling.