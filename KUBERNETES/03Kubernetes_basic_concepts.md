# KIND and Kubernetes Fundamentals

Complete guide to KIND (Kubernetes in Docker), kubectl, and fundamental Kubernetes concepts including Pods, Deployments, Services, and Namespaces.

## Table of Contents
- [KIND - Kubernetes in Docker](#kind---kubernetes-in-docker)
- [kubectl Command-Line Tool](#kubectl-command-line-tool)
- [Understanding Pods, Deployments, and Services](#understanding-pods-deployments-and-services)
- [Namespaces](#namespaces)
- [YAML Configuration](#yaml-configuration)
- [Working with Deployments and Services](#working-with-deployments-and-services)
- [Database Services](#database-services)
- [Data Persistence](#data-persistence)
- [Ingress](#ingress)
- [Headless Services](#headless-services)
- [Production Deployment Example](#production-deployment-example)
- [Custom Domain Setup](#custom-domain-setup)

---

## KIND - Kubernetes in Docker

### What is KIND?

**KIND (Kubernetes IN Docker)** runs Kubernetes clusters inside Docker containers.

**Purpose:**
- Lightweight and fast local Kubernetes
- Ideal for CI/CD pipelines
- Quick testing environment
- Easy multi-node cluster setup

**Advantages:**
- ✅ Runs entirely in Docker containers
- ✅ Fast cluster creation/deletion
- ✅ Perfect for testing and development
- ✅ No VM overhead (unlike Minikube with VM drivers)

**When to use KIND:**
- CI/CD pipelines (automated testing)
- Quick local testing
- Multi-node cluster experiments
- Kubernetes learning

---

## kubectl Command-Line Tool

### What is kubectl?

Command-line tool for interacting with and controlling Kubernetes clusters.

### Core Operations

**Resource Management:**
- Create, update, delete resources
- `kubectl apply`, `kubectl delete`

**Cluster Inspection:**
- Inspect cluster state
- `kubectl get`, `kubectl describe`

**Debugging and Monitoring:**
- Debug and monitor applications
- `kubectl logs`, `kubectl exec`

### Works With

- **Minikube** - Local Kubernetes
- **KIND** - Kubernetes in Docker
- **Cloud Clusters** - EKS (AWS), AKS (Azure), GKE (Google Cloud)
- **kubeadm** - Custom cluster setups

### Complementary Tools

**kubens** - Switch namespaces easily
```bash
kubens                    # List namespaces
kubens <namespace>        # Switch to namespace
```

**kubectx** - Switch between clusters easily
```bash
kubectx                   # List clusters
kubectx <cluster-name>    # Switch to cluster
```

---

## Understanding Pods, Deployments, and Services

### Real-World Example: Full-Stack Application

**Scenario:**
- Backend: Node.js server (APIs, database logic)
- Frontend: React app (served via Nginx or Vite)
- Requirements: Communication, scaling, resilience

### Backend Components

**🧩 Backend (Node.js API)**

**Pod:**
- Runs your Node.js container
- Smallest deployable unit

**Deployment:**
- Ensures backend pod always running
- Horizontal scaling (replicas: 3)
- Self-healing (restarts failed pods)

**Service:**
- Exposes Node.js backend as `backend-service`
- Other pods access via DNS: `http://backend-service:3000`
- Internal load balancing

### Frontend Components

**🧩 Frontend (React App)**

**Pod:**
- Runs React container
- Served statically via Nginx

**Deployment:**
- Keeps UI pods running
- Safe updates with zero downtime

**Service:**
- Exposes React frontend to external users
- Type: LoadBalancer or NodePort

### Component Analogies

**Pod:**
- Analogy: One delivery shop or warehouse
- Contains: One or more containers
- Typically: One container per pod

**Deployment:**
- Analogy: "Always keep 3 shops active"
- Policy: Maintain desired pod count
- Manages: Pod lifecycle and scaling

**Node:**
- Analogy: A city or town hosting your shops
- Contains: Multiple pods
- Provides: CPU, memory, storage

**Cluster:**
- Analogy: Your entire delivery network
- Contains: Multiple nodes
- Provides: Orchestration and management

**Service:**
- Analogy: GPS routing system between stores
- Provides: Stable networking endpoint
- Handles: Load balancing and service discovery

### Setup Visualization

```
[Cluster]
  ├── Node A (Host Machine)
  │   ├── Frontend Pod 1
  │   └── Backend Pod 1
  ├── Node B
  │   ├── Frontend Pod 2
  │   └── Backend Pod 2
  └── Node C
      ├── Frontend Pod 3
      └── Backend Pod 3
```

**How it works:**
- Kubernetes decides pod placement across nodes
- Node overload triggers pod migration
- DNS resolution: `http://backend-service:3000` (no IPs needed)

### Service Exposure Strategy

**Frontend Service:**
- Exposes React to outside world (users)
- Type: LoadBalancer or NodePort
- Public access

**Backend Service:**
- Internal connection only
- Frontend calls backend APIs
- Type: ClusterIP
- No external exposure

**Security benefit:**
- Frontend: Publicly accessible
- Backend: Internal only (not exposed to internet)
- Database: Internal only (accessed by backend)

---

## Namespaces

### What are Namespaces?

**Namespaces** are virtual clusters within your physical cluster - they provide logical isolation and resource management.

**Definition:**
- Kubernetes object grouping other objects
- Groups: Pods, Services, Deployments, ConfigMaps, Secrets
- Common name for related resources

### Use Cases

#### 1. Environment Separation

**dev-namespace:**
- Fast iteration
- Debug logging enabled
- Exposed ports for testing

**staging-namespace:**
- Near-production replica
- Final testing environment
- Production-like configuration

**prod-namespace:**
- Hardened security
- Monitored closely
- Minimal exposure

**Same manifests, different configurations:**
- Each namespace runs same Deployments/Services
- Tuned for specific purpose
- Isolated from other environments

#### 2. Blue-Green Deployment Strategy

**Powerful deployment pattern using namespaces:**

**Setup:**
```
production-blue namespace:
  ├── React Deployment
  ├── Node.js Deployment
  ├── Services
  ├── ConfigMaps
  └── Secrets

production-green namespace:
  ├── React Deployment (new version)
  ├── Node.js Deployment (new version)
  ├── Services
  ├── ConfigMaps
  └── Secrets
```

**Deployment Flow:**

| Phase | Action |
|-------|--------|
| ✅ **Active** | production-blue serves all user traffic |
| 🧪 **Test** | Deploy updated code to production-green and validate |
| 🔄 **Switch** | Update external Service to point to production-green |
| 🧹 **Cleanup** | Tear down or repurpose production-blue |

**Why This is Powerful:**

**Zero Downtime:**
- Traffic switches instantly
- No user interruption
- Instant rollback capability

**Isolated Testing:**
- Crash-test without affecting live traffic
- Validate in production-like environment
- Full integration testing

**Advanced Capabilities:**
- Pod traffic sniffing between namespaces
- RBAC for access restriction
- Chaos engineering with tools like chaos-mesh
- Gradual traffic migration (canary deployments)

### Namespace Commands

```bash
# View all namespaces (requires kubens tool)
kubens

# Switch to namespace
kubens <namespace>

# Create namespace
kubectl create ns <namespace>

# Delete namespace (deletes all resources inside)
kubectl delete ns <namespace>

# Create namespace via YAML
kubectl apply -f namespace.yaml
```

**Example namespace YAML:**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production-blue
```

---

## YAML Configuration

### What is YAML?

**YAML** (YAML Ain't Markup Language) - Human-readable configuration language.

**Used in:**
- Kubernetes
- Docker Compose
- GitHub Actions
- Ansible
- Terraform
- CI/CD pipelines

### Basic Application Requirements

For a basic application, you need:
1. **Deployment** - Defines pods
2. **Service** - Exposes pods

**VS Code Extension:** Kubernetes - Provides template YAML files

### 1. Deployment YAML

**Deployment = Pod Blueprint**

A deployment is a collection of pods. The template section specifies pod configuration.

**What deployment YAML defines:**
- What to run (image)
- How many to run (replicas)
- Where to run it (selector)
- How to configure it (resources, ports, env vars)

**Example:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: web-app-ns
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend  # Must match template labels
  template:
    metadata:
      labels:
        app: backend  # Must match selector
    spec:
      containers:
      - name: backend
        image: docker.io/username/node-image
        resources:
          limits:
            memory: "128Mi"
            cpu: "500m"
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          value: "mongodb://database:27017"
```

**Key fields explained:**

**selector.matchLabels:**
- Selects pods with matching labels
- `app: backend` - Select all pods labeled as backend

**template.metadata.labels:**
- Labels assigned to new pods
- Must match selector labels
- New pods join deployment via label matching

**Apply deployment:**
```bash
# Apply with namespace in YAML
kubectl apply -f deployment.yaml

# Override namespace at runtime
kubectl apply -f deployment.yaml -n <namespace>
```

### 2. Service YAML

**Service = Stable Access Point**

**What service YAML provides:**
- Selects pods using selector
- Exposes via stable IP and port
- Handles load balancing across replicas

**Example:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: web-app-ns
spec:
  selector:
    app: backend  # Selects pods with this label
  ports:
  - port: 3000        # Service port (access point)
    targetPort: 3000  # Pod/container port (destination)
```

**How it works:**
- Service can select Deployment or StatefulSet pods
- Selector matches pod labels
- Requests to service distribute across 3 running pods
- Automatic load balancing

**Apply service:**
```bash
kubectl apply -f service.yaml
```

**Result:**
- Requests → Service → One of 3 backend pods
- Load balancing handled automatically

### 3. Port Forwarding for Local Access

Access service from local machine:

```bash
kubectl port-forward svc/backend-service 8080:3000
```

**Access:** `http://localhost:8080` reaches backend

---

## Working with Deployments and Services

### Essential kubectl Commands

```bash
# View all resources in namespace
kubectl get all -n <namespace>

# View specific resource types
kubectl get svc              # Services
kubectl get deploy           # Deployments
kubectl get pods             # Pods

# Describe resources
kubectl describe svc/<name>
kubectl describe deploy/<name>
kubectl describe pod/<name>

# Edit resources (opens in editor)
kubectl edit <resource-type>/<resource-name>
kubectl edit deployment backend-deploy

# Delete resources
kubectl delete <resource-type>/<resource-name>
kubectl delete deployment backend-deploy
kubectl delete svc backend-service

# View logs
kubectl logs <pod-name>
kubectl logs <pod-name> --follow      # Real-time
kubectl logs <pod-name> --tail=50     # Last 50 lines

# Delete all resources
kubectl delete all --all  # ⚠️ Deletes pods, services, deployments
```

---

## Database Services

### MongoDB Deployment Example

#### Local Docker Development

```bash
# Run MongoDB locally
docker run -p 27017:27017 mongo
# Accessible at localhost:27017
```

#### Kubernetes MongoDB Deployment

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb
spec:
  replicas: 1  # Single source of truth
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongodb-storage
          mountPath: /data/db
      volumes:
      - name: mongodb-storage
        persistentVolumeClaim:
          claimName: mongodb-pvc
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  selector:
    app: mongodb
  ports:
  - protocol: TCP
    port: 27017
    targetPort: 27017
```

**⚠️ Important Notes:**

**Why replicas: 1?**
- Using Deployment for database (not StatefulSet)
- Multiple replicas = different data per pod
- Single replica = single source of truth
- For production: Use StatefulSet with replication

### Connecting from Application

**Node.js connection:**
```javascript
// Flexible connection with environment variable
const DB_URI = process.env.DB_URI || 'mongodb://database:27017';
mongoose.connect(DB_URI, { 
  useNewUrlParser: true, 
  useUnifiedTopology: true 
});
```

**Why this works:**
- `process.env.DB_URI` - Different per environment (local, staging, prod)
- `database` - Internal service name (DNS resolution)
- Kubernetes DNS resolves to MongoDB service
- Scalability: Scale app and MongoDB independently

### Mongo Express - Database GUI

**Mongo Express** provides GUI for MongoDB (like Minikube Dashboard for clusters).

#### Docker Approach

```bash
# Find MongoDB container IP
docker ps
docker inspect <mongo-container-id>
# Find "IPAddress": "172.17.0.2"

# Check MongoDB running
curl http://172.17.0.2:27017

# Run Mongo Express
docker run \
  -e ME_CONFIG_MONGODB_SERVER=172.17.0.2:27017 \
  -p 8081:8081 \
  mongo-express

# Access GUI at http://localhost:8081
```

#### Kubernetes Approach

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        env:
        - name: ME_CONFIG_MONGODB_SERVER
          value: database  # Service name
          # Full DNS: database.<namespace>.svc.cluster.local:27017
        - name: ME_CONFIG_MONGODB_PORT
          value: "27017"
        ports:
        - containerPort: 8081
```

**Service (NodePort):**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongo-express
spec:
  type: NodePort
  selector:
    app: mongo-express
  ports:
  - protocol: TCP
    port: 8081
    targetPort: 8081
    nodePort: 31000  # Accessible at minikube-ip:31000
```

**Access methods:**

**Minikube:**
```bash
minikube service mongo-express
# Opens at http://<minikube-ip>:31000
```

**ClusterIP + Port Forward:**
```yaml
spec:
  type: ClusterIP  # Change from NodePort
  # Remove nodePort field
```

```bash
kubectl port-forward svc/mongo-express 8081:8081
# Access at http://localhost:8081
```

**Cloud (LoadBalancer):**
```yaml
spec:
  type: LoadBalancer
```

---

## Data Persistence

### The Problem: Ephemeral Containers

**By default, Kubernetes pods are ephemeral:**
- Pod termination → filesystem vanishes
- Pod rescheduling → data lost
- Container restart → data lost

**MongoDB example:**
- Data stored in `/data/db` inside container
- No persistent volume → data in writable layer only
- Pod deleted → all data lost

### Solution: Persistent Volumes

#### 1. Define PersistentVolumeClaim (PVC)

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongodb-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: standard
```

#### 2. Mount PVC in Deployment

```yaml
spec:
  containers:
  - name: mongodb
    image: mongo
    volumeMounts:
    - name: mongodb-storage
      mountPath: /data/db
  volumes:
  - name: mongodb-storage
    persistentVolumeClaim:
      claimName: mongodb-pvc
```

**Result:**
- MongoDB writes to persistent volume
- Data survives pod deletion/rescheduling
- Data intact across container restarts

**💡 Better Approach: StatefulSet**
- Designed for stateful applications
- Better pod identity control
- Persistent storage per pod
- Ideal for databases

### Why Persistent Volumes Matter

#### Docker Perspective

**🔁 Containers Are Temporary**
- Container stops → internal filesystem wiped
- Data lost: logs, uploads, database files
- Unless explicitly persisted

**💾 Docker Solution:**
```bash
# Named volume
docker run -v mydata:/app/data myimage

# Data persists even after container removal
```

#### Kubernetes Perspective

**☸️ Kubernetes Components:**

**PersistentVolume (PV):**
- Actual storage (disk, NFS, cloud volume)
- Provisioned by admin or dynamically

**PersistentVolumeClaim (PVC):**
- Request for storage by pod
- Specifies: size, access mode, StorageClass

**Pods mount PVCs:**
```yaml
volumeMounts:
- mountPath: "/data"
  name: my-storage

volumes:
- name: my-storage
  persistentVolumeClaim:
    claimName: my-pvc
```

**Result:**
- Data survives rescheduling
- Data survives crashes
- Data survives node migrations

### Storage Components

**StorageClass:**
- Template for dynamic provisioning
- Abstracts storage backend
- Examples: AWS EBS, GCE PD, NFS, Azure Disk

**PersistentVolume (PV):**
- Actual storage piece
- Disk or NFS share
- Provisioned manually (static) or dynamically

**PersistentVolumeClaim (PVC):**
- Storage request by pod
- Specifies: size, access mode, optionally StorageClass

### Hotel Booking Analogy

**🎓 Understanding the relationship:**

**StorageClass** = Hotel brand
- Budget, luxury, pet-friendly
- Defines how rooms are created

**PersistentVolume (PV)** = Actual room
- Room 101, Room 202
- Physical storage available

**PersistentVolumeClaim (PVC)** = Guest request
- "Need king bed + WiFi"
- Storage requirements

**How it works:**
1. PVC requests storage with features
2. If matching PV exists → bound
3. If no match → Kubernetes uses StorageClass to create new PV dynamically

### Lifecycle Summary

**1. Provisioning:**
- Static: Manual PV creation by admin
- Dynamic: Automatic via StorageClass

**2. Binding:**
- PVC binds to matching PV
- One-to-one relationship

**3. Usage:**
- Pods mount PVCs
- Access storage through mount path

**4. Reclaiming:**
- PVC deleted → reclaim policy applies
- Options: Retain, Recycle, Delete

### Complete Example

```yaml
# Deployment with Persistent Storage
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: docker.io/mongo
        ports:
        - containerPort: 27017
        volumeMounts:
        - mountPath: /data/db
          name: mongodb
      volumes:
      - name: mongodb
        persistentVolumeClaim:
          claimName: mongo-pvc

---

# Service
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  selector:
    app: mongo
  ports:
  - port: 27017
    targetPort: 27017

---

# PersistentVolume
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  hostPath:
    path: /data/mongo/
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: standard

---

# PersistentVolumeClaim
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
  storageClassName: standard
```

**Apply:**
```bash
kubectl apply -f mongodb-storage.yaml
kubectl get pv
kubectl get pvc
kubectl describe pv mongo-pv
kubectl describe pvc mongo-pvc
```

---

## Ingress

### What is Ingress?

**Ingress** routes external HTTP/HTTPS traffic to internal services without exposing each service directly.

**Benefits:**
- Single entry point
- Path-based routing
- Host-based routing
- TLS/SSL termination
- Reduced attack surface

### Without Ingress

```
Internet → Service A (exposed, NodePort 30001)
Internet → Service B (exposed, NodePort 30002)
Internet → Service C (exposed, NodePort 30003)
```

**Problems:**
- Multiple exposed ports
- Increased attack surface
- No centralized SSL
- Complex firewall rules

### With Ingress

```
Internet → Ingress (single entry, HTTPS)
              ↓
         ┌────┼────┐
         ↓    ↓    ↓
    Service A (ClusterIP, internal)
    Service B (ClusterIP, internal)
    Service C (ClusterIP, internal)
```

**Benefits:**
- ✅ Single exposure point
- ✅ All services internal (ClusterIP)
- ✅ Centralized TLS/SSL
- ✅ Path/host-based routing

### Ingress Resource

**Ingress is a Kubernetes resource** that defines routing rules for external traffic.

**Example:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
  - host: backend.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: flask-backend
            port:
              number: 5000
  - host: frontend.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: express-frontend
            port:
              number: 3000
```

### TLS/HTTPS Support

**Add TLS section:**
```yaml
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    # ... paths
```

**Create TLS secret:**
```bash
kubectl create secret tls example-tls \
  --cert=cert.pem \
  --key=key.pem
```

### Apply Ingress

```bash
kubectl apply -f ingress.yaml
```

**Important:**
- Use ClusterIP for all backend services
- Ingress handles external exposure
- Cloud provider assigns external IP to Ingress

### Routing Examples

**Real-world traffic flow:**

| Request URL | Routed To |
|-------------|-----------|
| `https://example.com/api` | Flask backend (port 5000) |
| `https://example.com/` | Express frontend (port 3000) |
| `https://backend.example.com/api` | Flask backend |
| `https://frontend.example.com/` | Express frontend |

---

## Headless Services

### What is a Headless Service?

**Headless service** doesn't assign ClusterIP - exposes individual pod IPs via DNS.

**Definition:**
```yaml
spec:
  clusterIP: None
```

**Kubernetes behavior:**
- "Don't give me virtual IP"
- "Let me discover pods directly"

### Classroom Analogy

**Normal Service:**
- Like calling "Class 10A"
- Teacher picks any student
- You don't know which one

**Headless Service:**
- Like calling each student by name
- You know exactly who you're talking to
- Direct communication

### Use Cases

**StatefulSets:**
- Each pod has stable identity (db-0, db-1, db-2)
- Direct pod access needed
- Persistent storage per pod

**Database Clustering:**
- Pods need direct communication
- Replication setup
- Master-slave configuration

**Custom Load Balancing:**
- Application chooses which pod
- Client-side load balancing
- Advanced routing logic

**Service Discovery:**
- DNS returns all pod IPs
- Not single ClusterIP
- Full pod list available

### DNS Behavior

**Normal Service DNS:**
```
my-service.default.svc.cluster.local → 10.96.0.1 (ClusterIP)
```

**Headless Service DNS:**
```
my-service.default.svc.cluster.local →
  - 10.244.1.5 (pod-0)
  - 10.244.2.3 (pod-1)
  - 10.244.3.7 (pod-2)
```

**Pod-specific DNS:**
```
pod-0.my-service.default.svc.cluster.local → 10.244.1.5
pod-1.my-service.default.svc.cluster.local → 10.244.2.3
pod-2.my-service.default.svc.cluster.local → 10.244.3.7
```

### Example YAML

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-headless-service
spec:
  clusterIP: None
  selector:
    app: my-app
  ports:
  - port: 80
```

**Use with StatefulSet:**
```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: my-headless-service  # References headless service
  replicas: 3
  # ... rest of spec
```

---

## Production Deployment Example

### Complete Full-Stack Application

#### Step 1: Define Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web-app-ns
```

**Purpose:**
- Logical isolation
- Can create blue-green namespaces later
- Resource quota control

#### Step 2: Backend Config and Secret

**ConfigMap:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
  namespace: web-app-ns
data:
  DATABASE_URL: mongodb://db:27017/myapp
  NODE_ENV: production
```

**Secret:**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: backend-secret
  namespace: web-app-ns
type: Opaque
data:
  DB_PASSWORD: bXlzZWNyZXRwYXNz  # Base64-encoded
```

**Encode secret:**
```bash
echo -n "mypassword" | base64
# bXlwYXNzd29yZA==
```

#### Step 3: Backend Deployment + Service

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-deploy
  namespace: web-app-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
      - name: backend
        image: username/node-app:latest
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: backend-config
              key: DATABASE_URL
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: backend-secret
              key: DB_PASSWORD
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: web-app-ns
spec:
  type: ClusterIP  # Internal only
  selector:
    app: backend
  ports:
  - port: 3000
    targetPort: 3000
```

#### Step 4: Frontend Deployment + Service

**Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
  namespace: web-app-ns
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: username/react-app:latest
        ports:
        - containerPort: 80
```

**Service:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: web-app-ns
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

**Frontend accessing backend:**
```javascript
// Full DNS name
fetch("http://backend-service.web-app-ns.svc.cluster.local:3000/api/users")

// Short form (same namespace)
fetch("http://backend-service:3000/api/users")
```

---

## Custom Domain Setup

### Deploy to Custom Domain (example: satyam.dev)

#### Prerequisites

- Kubernetes cluster (cloud or on-premise)
- Domain name (from registrar)
- Ingress Controller

#### Step 1: Deploy Ingress Controller

**Using NGINX Ingress:**
```bash
# Add Helm repo
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

# Install NGINX Ingress Controller
helm install nginx-ingress ingress-nginx/ingress-nginx

# Creates LoadBalancer service with external IP
```

**Verify:**
```bash
kubectl get svc -n ingress-nginx
# Look for EXTERNAL-IP
```

#### Step 2: Create Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: web-app-ns
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: satyam.dev
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 3000
```

**Apply:**
```bash
kubectl apply -f ingress.yaml
```

#### Step 3: Point Domain to Cluster

**Get external IP:**
```bash
kubectl get svc -n ingress-nginx
# Note EXTERNAL-IP of nginx-ingress-controller
```

**Configure DNS:**
1. Go to domain registrar (Namecheap, GoDaddy, etc.)
2. Add A record:
   - Host: `@` (or `satyam.dev`)
   - Points to: `<EXTERNAL-IP>`
   - TTL: 300 (or auto)

**Wait for DNS propagation:**
```bash
# Check DNS resolution
nslookup satyam.dev
dig satyam.dev
```

#### Step 4: Add HTTPS with cert-manager (Optional)

**Install cert-manager:**
```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```

**Create ClusterIssuer:**
```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx
```

**Update Ingress with TLS:**
```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - satyam.dev
    secretName: satyam-dev-tls
  rules:
  - host: satyam.dev
    # ... paths
```

**Apply:**
```bash
kubectl apply -f cluster-issuer.yaml
kubectl apply -f ingress.yaml
```

**Verify certificate:**
```bash
kubectl get certificate -n web-app-ns
kubectl describe certificate satyam-dev-tls -n web-app-ns
```

**Access:**
```
https://satyam.dev → Frontend
https://satyam.dev/api → Backend
```

---

## Quick Reference

### Essential Commands

```bash
# Namespaces
kubens
kubectl create ns <n>
kubectl delete ns <n>

# Resources
kubectl get all -n <namespace>
kubectl apply -f <file>
kubectl delete -f <file>

# Pods
kubectl get pods
kubectl logs <pod>
kubectl exec -it <pod> -- bash

# Services
kubectl get svc
kubectl port-forward svc/<n> 8080:80

# Ingress
kubectl get ingress
kubectl describe ingress <n>

# Storage
kubectl get pv
kubectl get pvc
```

---

This comprehensive guide covers KIND, kubectl, Kubernetes fundamentals, namespaces, YAML configuration, database services, persistence, ingress, and production deployments with custom domains.