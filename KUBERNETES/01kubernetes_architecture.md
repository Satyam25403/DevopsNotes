# Kubernetes Introduction and Architecture

Comprehensive guide to Kubernetes fundamentals, architecture, components, and core concepts.

## Table of Contents
- [What is Kubernetes](#what-is-kubernetes)
- [Kubernetes Architecture](#kubernetes-architecture)
- [Core Components](#core-components)
- [Kubernetes Objects](#kubernetes-objects)
- [Service Types](#service-types)
- [Real-World Analogies](#real-world-analogies)

---

## What is Kubernetes?

**Kubernetes (K8s)** is a container orchestration tool for managing containers efficiently.

**Container orchestration includes:**
- Create containers
- Delete containers
- Maintain containers
- Configure containers
- Scale containers
- Load balance containers
- Monitor health

### Why Kubernetes?

**Without Kubernetes:**
```bash
# Manual container management
docker run app1
docker run app2
docker run app3
# What if app2 crashes?
# How do you scale?
# How do you load balance?
```

**With Kubernetes:**
- ✅ Automatic scaling
- ✅ Self-healing (restarts failed containers)
- ✅ Load balancing
- ✅ Service discovery
- ✅ Rolling updates
- ✅ Resource optimization

---

## Kubernetes Architecture

### Cluster Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Master Node (Control Plane)          │   │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │   │
│  │  │   API    │  │Scheduler │  │ Controller   │   │   │
│  │  │  Server  │  │          │  │  Manager     │   │   │
│  │  └──────────┘  └──────────┘  └──────────────┘   │   │
│  │  ┌──────────┐                                    │   │
│  │  │   etcd   │  Key-value store                  │   │
│  │  └──────────┘                                    │   │
│  └──────────────────────────────────────────────────┘   │
│                          │                               │
│         ┌────────────────┼────────────────┐             │
│         ▼                ▼                ▼              │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐        │
│  │Worker Node │  │Worker Node │  │Worker Node │        │
│  │            │  │            │  │            │        │
│  │  kubelet   │  │  kubelet   │  │  kubelet   │        │
│  │  kube-proxy│  │  kube-proxy│  │  kube-proxy│        │
│  │  Container │  │  Container │  │  Container │        │
│  │  Runtime   │  │  Runtime   │  │  Runtime   │        │
│  │            │  │            │  │            │        │
│  │  [Pods]    │  │  [Pods]    │  │  [Pods]    │        │
│  └────────────┘  └────────────┘  └────────────┘        │
└─────────────────────────────────────────────────────────┘
```

### Master Node (Control Plane)

**At least 1 master node required** (can have multiple for high availability)

**Components:**

**1. API Server**
- Entry point for all commands and communication
- Allows use of `kubectl` CLI
- Receives HTTP requests (RESTful API)
- Validates and processes commands
- Central gateway to the cluster

```bash
kubectl → HTTP Request → API Server → Cluster Action
```

**2. Scheduler**
- Assigns pods to nodes
- Based on resource availability (CPU, memory)
- Considers constraints and policies
- Optimizes resource utilization

**Example:**
```
New Pod requested
  ↓
Scheduler checks nodes
  ↓
Node 1: CPU 80% used ❌
Node 2: CPU 40% used ✅
  ↓
Pod assigned to Node 2
```

**3. Controller Manager**
- Watches cluster state
- Triggers changes to reach desired state
- Manages controllers (Deployments, ReplicaSets, etc.)

**Controllers include:**
- ReplicaSet Controller (maintains pod replicas)
- Deployment Controller (manages deployments)
- Node Controller (monitors node health)
- Service Controller (manages services)

**4. etcd**
- Key-value store
- Stores cluster state and configuration
- Single source of truth
- Distributed and consistent

**What etcd stores:**
- Pod configurations
- Service endpoints
- Node status
- Secrets and ConfigMaps
- Network policies

---

### Worker Nodes

**0 or more worker nodes** (where actual workloads run)

**Components:**

**1. kubelet**
- Agent running on each node
- Talks to container runtime
- Manages pods and containers
- Reports node status to master
- Executes pod specifications

**2. kube-proxy**
- Handles networking
- Manages IP addresses and ports
- Load balancing between pods
- Routes requests to correct pods

**3. Container Runtime**
- Runs containers
- Container Runtime Interface (CRI) compatible
- Examples: Docker, containerd, CRI-O

**Pod lifecycle on worker node:**
```
API Server → kubelet
  ↓
kubelet → Container Runtime
  ↓
Container Runtime creates containers
  ↓
kube-proxy configures networking
```

---

## Core Components

### Developer Perspective

| Component | Description | Real-World Analogy |
|-----------|-------------|-------------------|
| **Pod** | Smallest deployable unit (1+ containers) | Shipping box containing items |
| **Node** | VM/machine in cluster (has CPU/memory) | Worker in warehouse to run boxes |
| **Cluster** | Collection of nodes | Entire warehouse system |
| **Deployment** | Blueprint for managing pods (stateless) | Plan: "Keep 3 boxes running always" |
| **StatefulSet** | Manages pods with persistent identity | Named boxes: box-0, box-1, box-2 |
| **Service** | Exposes pods to network | Receptionist routing requests |
| **Namespace** | Logical partitioning | Labeled rooms (dev, prod, test) |
| **ReplicaSet** | Ensures specified number of pod replicas | Maintains exact box count |
| **DaemonSet** | Ensures one pod per node | One security guard per floor |
| **ConfigMap** | Stores configuration as key-value | Configuration clipboard |
| **Ingress** | Manages external HTTP/HTTPS access | Single entrance gate |
| **Volume** | Persistent storage for pods | Storage locker for boxes |

---

## Kubernetes Objects

### Essential Objects

**1. Pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.21
```

**Purpose:** Runs your container(s)

---

**2. Deployment**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
```

**Purpose:** Manages and scales pods (stateless applications)

---

**3. Service**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: myapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
```

**Purpose:** Connects and exposes pods

---

**4. Namespace**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
```

**Purpose:** Logical grouping of resources

**Use cases:**
- Separate environments (dev, staging, prod)
- Team isolation
- Resource quotas

---

**5. ConfigMap**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  API_URL: "https://api.example.com"
  PORT: "8080"
  FEATURE_FLAG: "true"
```

**Purpose:** Environment configs (API URLs, feature flags, ports)

---

**6. Secret**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: cGFzc3dvcmQxMjM=  # base64 encoded
```

**Purpose:** Store sensitive data securely (DB passwords, API keys)

---

**7. PersistentVolumeClaim (PVC)**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**Purpose:** Request storage from cluster

**Analogy:** Asking IT for a permanent hard drive

---

**8. StatefulSet**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: "db"
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      containers:
      - name: postgres
        image: postgres:13
```

**Purpose:** Stateful apps like databases (fixed names: db-0, db-1, db-2)

---

**9. DaemonSet**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  selector:
    matchLabels:
      app: monitoring
  template:
    metadata:
      labels:
        app: monitoring
    spec:
      containers:
      - name: node-exporter
        image: prom/node-exporter
```

**Purpose:** Runs one pod per node (monitoring, logging)

**Analogy:** Antivirus software on every machine

---

**10. Job / CronJob**
```yaml
# Job (one-time)
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool
      restartPolicy: Never

# CronJob (scheduled)
apiVersion: batch/v1
kind: CronJob
metadata:
  name: weekly-backup
spec:
  schedule: "0 0 * * 0"  # Every Sunday at midnight
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: backup-tool
          restartPolicy: Never
```

**Purpose:** One-time or scheduled tasks (backups, cleanups)

---

**11. Ingress**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend
            port:
              number: 80
```

**Purpose:** HTTP routing from outside to services

**Benefit:** Single point of entry (reduces attack surface)

---

## Service Types

### Overview

| Type | Visibility | Use Case | Access Method |
|------|-----------|----------|---------------|
| **ClusterIP** | Internal only | Inter-pod communication | `<service-name>:<port>` |
| **NodePort** | External | Quick testing/local setup | `<NodeIP>:<NodePort>` |
| **LoadBalancer** | External (cloud) | Production public access | Cloud-provided IP |
| **ExternalName** | DNS redirect | External service mapping | DNS resolution |

---

### 1. ClusterIP (Default)

**Internal communication only.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 5000
    targetPort: 5000
```

**When to use:**
- ✅ Frontend calling backend
- ✅ Internal microservices communication
- ✅ Don't want external access

**Access:**
```bash
# From another pod
curl http://backend:5000
```

---

### 2. NodePort

**Exposes service on each node's IP at a static port.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30001  # 30000-32767 range
```

**When to use:**
- ✅ Quick exposure for testing
- ✅ Local development (Minikube)
- ✅ Non-cloud environments

**Access:**
```bash
# From outside cluster
http://<NodeIP>:30001
```

---

### 3. LoadBalancer

**Cloud provider creates external load balancer.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
```

**When to use:**
- ✅ Production on cloud (AWS, GCP, Azure)
- ✅ Need public IP for real traffic
- ✅ Automatic load balancing

**Access:**
```bash
# Cloud provides external IP
http://<External-IP>
```

**Cloud provider provisions:**
- AWS: ELB (Elastic Load Balancer)
- GCP: Cloud Load Balancer
- Azure: Azure Load Balancer

---

### 4. ExternalName

**Maps service to external DNS name.**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.google.com
```

**When to use:**
- ✅ Third-party API or database
- ✅ Service not hosted in Kubernetes
- ✅ DNS resolution inside cluster

**Access:**
```bash
# Inside cluster
curl http://external-db
# Resolves to db.google.com
```

---

## Real-World Analogies

### Warehouse System

**Cluster = Entire Warehouse**
- Nodes = Workers in warehouse
- Pods = Shipping boxes (containing products)
- Deployment = Instruction: "Always keep 3 boxes ready"
- Service = Receptionist routing customers
- Namespace = Labeled rooms (Room A: Development, Room B: Production)

### Office Building

**Cluster = Office Building**
- Nodes = Floors
- Pods = Office rooms
- Deployment = Office allocation plan
- Service = Reception desk
- DaemonSet = Security guard on each floor
- Ingress = Main entrance gate

### Restaurant

**Cluster = Restaurant**
- Nodes = Kitchen stations
- Pods = Dishes being prepared
- Deployment = Recipe for consistent dishes
- Service = Waiter serving customers
- ConfigMap = Recipe book
- Secret = Secret sauce recipe (locked away)

---

## Quick Reference

### Object Summary

```
Pod         → Runs containers
Deployment  → Manages pods (stateless)
StatefulSet → Manages pods (stateful)
Service     → Exposes pods
Namespace   → Groups resources
ConfigMap   → Configuration data
Secret      → Sensitive data
PVC         → Storage request
DaemonSet   → One pod per node
Job         → One-time task
CronJob     → Scheduled task
Ingress     → HTTP routing
```

### Service Types

```
ClusterIP    → Internal only
NodePort     → External (testing)
LoadBalancer → External (production/cloud)
ExternalName → DNS mapping
```

---

This comprehensive guide covers Kubernetes fundamentals, architecture, components, and core concepts for container orchestration.