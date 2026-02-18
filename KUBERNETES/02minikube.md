# Minikube - Local Kubernetes

Comprehensive guide to Minikube for running Kubernetes clusters locally.

## Table of Contents
- [What is Minikube](#what-is-minikube)
- [Installation](#installation)
- [Cluster Management](#cluster-management)
- [Minikube Commands](#minikube-commands)
- [Kubernetes Dashboard](#kubernetes-dashboard)
- [Accessing Services](#accessing-services)
- [Best Practices](#best-practices)

---

## What is Minikube?

**Minikube** allows you to create Kubernetes clusters locally for development and testing.

### Minikube vs KIND

**Minikube:**
- ✅ Full Kubernetes cluster locally
- ✅ Multiple drivers (Docker, VirtualBox, Hyper-V)
- ✅ Kubernetes Dashboard included
- ✅ Ingress and LoadBalancer emulation
- ⚠️ Can be resource-intensive (might slow down machine)

**KIND (Kubernetes in Docker):**
- ✅ Lightweight
- ✅ Faster startup
- ❌ Fewer features than Minikube

**Recommendation:** Use Minikube for full-featured local development.

---

## Installation

### Prerequisites

- Docker installed
- OR VirtualBox/Hyper-V (if not using Docker driver)
- kubectl installed

### Install Minikube

**Linux:**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
```

**macOS:**
```bash
brew install minikube
```

**Windows:**
```powershell
# Using Chocolatey
choco install minikube

# Or download installer from:
# https://minikube.sigs.k8s.io/docs/start/
```

### Verify Installation

```bash
minikube version
# Output: minikube version: v1.32.0
```

---

## Cluster Management

### Create a Cluster

**Basic cluster (single node):**
```bash
minikube start
```

**Multi-node cluster:**
```bash
# Create cluster with specific number of nodes
minikube start --nodes=3 -p dev-cluster

# --nodes: Number of worker nodes
# -p: Profile/cluster name
```

**With specific driver:**
```bash
# Docker driver (default)
minikube start --driver=docker

# VirtualBox
minikube start --driver=virtualbox

# Hyper-V (Windows)
minikube start --driver=hyperv
```

**With resource limits:**
```bash
minikube start \
  --cpus=4 \
  --memory=8192 \
  --disk-size=20g
```

### Cluster Profiles

**Why use profiles?**
- Multiple isolated clusters
- Different configurations (dev, staging, test)
- Separate environments

**Create named cluster:**
```bash
# Development cluster
minikube start --nodes=2 -p dev-cluster

# Production-like cluster
minikube start --nodes=3 -p prod-cluster

# Testing cluster
minikube start --nodes=1 -p test-cluster
```

### View All Clusters

```bash
# List all profiles/clusters
minikube profile list

# Output:
# |-------------|-----------|---------|
# |   Profile   |  Status   |  Nodes  |
# |-------------|-----------|---------|
# | dev-cluster | Running   |    2    |
# | prod-cluster| Stopped   |    3    |
# | test-cluster| Running   |    1    |
# |-------------|-----------|---------|
```

### Switch Between Clusters

```bash
# Switch to dev cluster
minikube profile dev-cluster

# Verify current profile
minikube profile

# All kubectl commands now use dev-cluster
kubectl get nodes
```

### Stop and Delete Clusters

```bash
# Stop cluster (pause)
minikube stop -p dev-cluster

# Start stopped cluster
minikube start -p dev-cluster

# Delete cluster (wipe completely)
minikube delete -p dev-cluster

# Delete all clusters
minikube delete --all
```

---

## Minikube Commands

### Essential Commands

```bash
# Start default cluster
minikube start

# Stop cluster (pause)
minikube stop

# Delete cluster (wipe)
minikube delete

# View cluster status
minikube status

# Get cluster IP
minikube ip

# SSH into node
minikube ssh

# View logs
minikube logs
```

### Cluster Information

```bash
# Get Minikube IP
minikube ip
# Output: 192.168.49.2

# View cluster config
kubectl config view

# Get node info
kubectl get nodes

# Check cluster status
minikube status
# Output:
# minikube
# type: Control Plane
# host: Running
# kubelet: Running
# apiserver: Running
# kubeconfig: Configured
```

### Addons

**Enable/disable features:**

```bash
# List available addons
minikube addons list

# Enable addon
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard

# Disable addon
minikube addons disable ingress

# Common addons:
# - ingress (HTTP routing)
# - metrics-server (resource metrics)
# - dashboard (Web UI)
# - registry (Local Docker registry)
# - storage-provisioner (Dynamic storage)
```

---

## Kubernetes Dashboard

### Open Dashboard

**Automatic (opens in browser):**
```bash
minikube dashboard
```

**This command:**
- Opens official Kubernetes Dashboard
- Visual interface for managing resources
- Shows pods, deployments, services, etc.
- Sets up secure proxy automatically

### Dashboard URL (Headless/Remote)

**For remote servers or WSL2:**
```bash
minikube dashboard --url

# Output:
# http://127.0.0.1:37561/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/
```

**Then:**
- Copy URL
- Open in browser
- Or set up SSH tunnel if on remote server

### Dashboard Features

**What you can do:**
- View all resources (pods, services, deployments)
- Check resource usage (CPU, memory)
- View logs
- Edit YAML configurations
- Create/delete resources
- Monitor events
- Manage namespaces

---

## Accessing Services

### Service Access Methods

**1. Using minikube service command:**
```bash
# Automatically opens service in browser
minikube service frontend

# Get service URL without opening browser
minikube service frontend --url

# Output:
# http://192.168.49.2:30080
```

**2. Using kubectl port-forward:**
```bash
kubectl port-forward service/frontend 8080:80
# Access at http://localhost:8080
```

**3. Using NodePort:**
```bash
# Get Minikube IP
minikube ip
# Output: 192.168.49.2

# Access service
http://192.168.49.2:<NodePort>
```

### Example: Expose Application

**Create deployment:**
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Create service:**
```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
```

**Apply and access:**
```bash
# Deploy
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# Access via minikube
minikube service webapp

# Or get URL
minikube service webapp --url
# http://192.168.49.2:30080
```

---

## Best Practices

### 1. Resource Allocation

```bash
# Allocate appropriate resources
minikube start \
  --cpus=2 \
  --memory=4096 \
  --disk-size=20g

# Don't over-allocate (leaves resources for host OS)
```

### 2. Use Profiles

```bash
# Separate environments
minikube start -p dev
minikube start -p staging
minikube start -p testing

# Easy switching
minikube profile dev
```

### 3. Enable Useful Addons

```bash
# Common addons for development
minikube addons enable ingress
minikube addons enable metrics-server
minikube addons enable dashboard
```

### 4. Clean Up Regularly

```bash
# Stop when not in use
minikube stop

# Delete old clusters
minikube delete -p old-cluster

# Clear Docker cache
minikube ssh
docker system prune -a
```

### 5. Use LoadBalancer Emulation

```bash
# Start Minikube tunnel (in separate terminal)
minikube tunnel

# Now LoadBalancer services get external IPs
```

---

## Troubleshooting

### Common Issues

**Cluster won't start:**
```bash
# Check driver issues
minikube start --driver=docker

# View logs
minikube logs

# Delete and recreate
minikube delete
minikube start
```

**Can't access service:**
```bash
# Verify service exists
kubectl get services

# Get service URL
minikube service <service-name> --url

# Check pods are running
kubectl get pods
```

**Out of resources:**
```bash
# Stop cluster
minikube stop

# Start with fewer resources
minikube start --cpus=2 --memory=2048
```

**Dashboard not opening:**
```bash
# Enable dashboard addon
minikube addons enable dashboard

# Get URL manually
minikube dashboard --url
```

---

## Complete Workflow Example

### Development Setup

```bash
# 1. Start cluster
minikube start --nodes=2 -p myproject

# 2. Enable addons
minikube addons enable ingress
minikube addons enable metrics-server

# 3. Deploy application
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml

# 4. Access application
minikube service myapp

# 5. View dashboard
minikube dashboard

# 6. When done
minikube stop -p myproject
```

---

## Quick Reference

### Cluster Management

```bash
# Create
minikube start
minikube start --nodes=3 -p dev

# Status
minikube status
minikube profile list

# Switch
minikube profile dev

# Stop/Delete
minikube stop -p dev
minikube delete -p dev
```

### Access

```bash
# Service
minikube service <name>
minikube service <name> --url

# Dashboard
minikube dashboard
minikube dashboard --url

# IP
minikube ip
```

### Addons

```bash
# List
minikube addons list

# Enable/Disable
minikube addons enable ingress
minikube addons disable ingress
```

---

This comprehensive guide covers Minikube for local Kubernetes development and testing.