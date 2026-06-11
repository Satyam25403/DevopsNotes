# Google Kubernetes Engine (GKE)

Google Kubernetes Engine is GCP's fully managed Kubernetes service. Google invented Kubernetes internally (as Borg), so GKE benefits from first-party expertise. GKE handles the Kubernetes **control plane** for you — you focus on deploying workloads, not managing cluster infrastructure. The GCP equivalent of AWS EKS.

---

## Why Kubernetes? Why GKE?

**Kubernetes** is the industry-standard container orchestration platform. It handles:
- Scheduling containers across nodes
- Self-healing (restarting failed containers)
- Horizontal scaling
- Service discovery and load balancing
- Rolling updates and rollbacks
- Configuration and secret management

**GKE** gives you Kubernetes without managing the control plane:
- Google runs, patches, and scales the Kubernetes API server and etcd
- Multi-zone and multi-region clusters out of the box
- Deep integration with GCP services (IAM, VPC, Cloud Load Balancing, Artifact Registry, Cloud Monitoring)
- Fully compatible with standard Kubernetes tooling (`kubectl`, Helm, ArgoCD, Flux)
- **Autopilot mode**: Google manages nodes too — you only define workloads

---

## GKE Modes

| Feature | Standard Mode | Autopilot Mode |
|---------|--------------|----------------|
| Node management | You manage node pools | Google manages all nodes |
| Pricing | Per node (VM) | Per pod (resource requests) |
| Flexibility | Full K8s control | Opinionated, hardened |
| Cost at low load | Higher (idle nodes) | Lower (pay per pod) |
| Best for | Complex workloads, GPU, custom configs | Most production workloads |

---

## Setting Up a GKE Cluster

```bash
# Enable APIs
gcloud services enable container.googleapis.com

# Create an Autopilot cluster (recommended)
gcloud container clusters create-auto my-cluster \
  --region=us-central1

# Create a Standard cluster
gcloud container clusters create my-cluster \
  --zone=us-central1-a \
  --num-nodes=3 \
  --machine-type=e2-standard-4 \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10 \
  --workload-pool=my-project.svc.id.goog    # Enable Workload Identity

# Configure kubectl to connect to the cluster
gcloud container clusters get-credentials my-cluster --region=us-central1

# Verify connection
kubectl get nodes
kubectl cluster-info
```

---

## Node Pools

```bash
# Add a node pool (e.g., for GPU workloads)
gcloud container node-pools create gpu-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --num-nodes=2 \
  --machine-type=a2-highgpu-1g \
  --accelerator=type=nvidia-tesla-a100,count=1

# Add a Spot (preemptible) node pool for batch workloads
gcloud container node-pools create spot-pool \
  --cluster=my-cluster \
  --zone=us-central1-a \
  --num-nodes=0 \
  --enable-autoscaling \
  --min-nodes=0 \
  --max-nodes=20 \
  --machine-type=n2-standard-4 \
  --spot

# List node pools
gcloud container node-pools list --cluster=my-cluster --zone=us-central1-a

# Upgrade a node pool
gcloud container node-pools upgrade spot-pool \
  --cluster=my-cluster --zone=us-central1-a
```

---

## Deploying Workloads

### Deployment (stateless apps)
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-app-ksa      # Workload Identity KSA
      containers:
        - name: my-app
          image: us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          env:
            - name: NODE_ENV
              value: production
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

```bash
kubectl apply -f deployment.yaml
kubectl rollout status deployment/my-app
kubectl get pods -l app=my-app
```

### Service + Ingress (expose to internet)
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
---
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: "gce"    # GCP Cloud Load Balancer
    kubernetes.io/ingress.global-static-ip-name: "my-static-ip"
    networking.gke.io/managed-certificates: "my-cert"
spec:
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: my-app-service
                port:
                  number: 80
```

### Managed TLS Certificate for GKE Ingress
```yaml
apiVersion: networking.gke.io/v1
kind: ManagedCertificate
metadata:
  name: my-cert
spec:
  domains:
    - myapp.example.com
```

---

## Workload Identity (GKE → GCP Services without key files)

Workload Identity is the recommended way to let GKE pods access GCP APIs securely — no key files needed.

```bash
# 1. Create a GCP Service Account (GSA)
gcloud iam service-accounts create my-app-gsa \
  --display-name="My App GCP Service Account"

# 2. Grant the GSA needed permissions (e.g., Cloud Storage)
gcloud projects add-iam-policy-binding my-project \
  --member="serviceAccount:my-app-gsa@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"

# 3. Create a Kubernetes Service Account (KSA)
kubectl create serviceaccount my-app-ksa --namespace=default

# 4. Bind KSA → GSA via Workload Identity
gcloud iam service-accounts add-iam-policy-binding my-app-gsa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:my-project.svc.id.goog[default/my-app-ksa]"

# 5. Annotate the KSA
kubectl annotate serviceaccount my-app-ksa \
  --namespace=default \
  iam.gke.io/gcp-service-account=my-app-gsa@my-project.iam.gserviceaccount.com

# 6. Reference KSA in your Deployment (serviceAccountName: my-app-ksa)
# The pod will now authenticate as my-app-gsa automatically
```

---

## Horizontal Pod Autoscaler (HPA)

```bash
# Scale based on CPU
kubectl autoscale deployment my-app --cpu-percent=60 --min=2 --max=20

# Or via YAML
kubectl apply -f - <<EOF
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
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
EOF
```

---

## Rolling Updates and Rollbacks

```bash
# Update image (triggers rolling update)
kubectl set image deployment/my-app my-app=us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.3.0

# Watch rollout progress
kubectl rollout status deployment/my-app

# Rollback to previous version
kubectl rollout undo deployment/my-app

# Rollback to specific revision
kubectl rollout history deployment/my-app
kubectl rollout undo deployment/my-app --to-revision=3
```

---

## Cluster Operations

```bash
# List clusters
gcloud container clusters list

# Upgrade cluster Kubernetes version
gcloud container clusters upgrade my-cluster \
  --master --cluster-version=1.30 \
  --zone=us-central1-a

# Resize a node pool
gcloud container clusters resize my-cluster \
  --node-pool=default-pool \
  --num-nodes=5 \
  --zone=us-central1-a

# Delete cluster
gcloud container clusters delete my-cluster --zone=us-central1-a
```

---

## GKE vs Cloud Run

| Feature | GKE | Cloud Run |
|---------|-----|-----------|
| Kubernetes API | ✅ Full | ❌ |
| Scale to zero | ❌ (Standard) / ✅ (Autopilot) | ✅ |
| Stateful workloads | ✅ | Limited |
| GPU support | ✅ | ✅ (limited) |
| Custom networking (Istio, NetworkPolicy) | ✅ | ❌ |
| Operational complexity | High | Low |
| Best for | Complex microservices, ML, stateful apps | Stateless HTTP services and event-driven functions |