# Helm - Introduction and Local Setup

Comprehensive guide to Helm, the package manager for Kubernetes, including installation and initial configuration.

## Table of Contents
- [What is Helm](#what-is-helm)
- [Why Use Helm](#why-use-helm)
- [Helm vs Kubectl](#helm-vs-kubectl)
- [Helm Chart Structure](#helm-chart-structure)
- [Local Kubernetes Setup](#local-kubernetes-setup)
- [Installing Helm](#installing-helm)
- [Quick Start](#quick-start)

---

## What is Helm

### Overview

**Helm** is the package manager for Kubernetes - like apt for Ubuntu, yum for RedHat, or npm for Node.js.

**Key capabilities:**
- **Packaging** - Bundle all YAMLs into one chart
- **Templating** - Avoid hardcoding repeated values
- **Sharing** - Distribute via ArtifactHub.io
- **Versioning** - Track and rollback releases

### The Problem Helm Solves

**Without Helm:**
```bash
# Multiple manual commands
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# To delete
kubectl delete -f deployment.yaml
kubectl delete -f service.yaml
kubectl delete -f ingress.yaml
# ... and so on
```

**With Helm:**
```bash
# Single command to deploy
helm install my-app ./my-chart

# Single command to remove
helm uninstall my-app
```

---

## Why Use Helm

### Benefits

**1. Simplification**
- One command vs many kubectl commands
- Manage complex applications easily
- Consistent deployment process

**2. Templating**
- Parameterize YAML files
- Reuse charts across environments
- No hardcoded values

**3. Versioning**
- Track release history
- Easy rollbacks
- Audit trail

**4. Sharing**
- Distribute charts via repositories
- Reuse community charts
- Package internal applications

### Use Cases

**Deploy complex applications:**
- WordPress (DB + web server + PHP)
- ArgoCD (multiple components)
- Prometheus + Grafana stack

**Multi-environment deployments:**
- Same chart for dev, staging, prod
- Different values per environment
- Consistent configuration

**Team collaboration:**
- Share charts across teams
- Standardize deployments
- Version control infrastructure

---

## Helm vs Kubectl

### Comparison

**Kubectl (Manual):**
```bash
# Create each resource manually
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl create configmap nginx-config --from-file=nginx.conf
kubectl create secret generic db-creds --from-literal=password=secret

# Update requires editing each file
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

**Helm (Automated):**
```bash
# Deploy everything at once
helm install nginx ./nginx-chart

# Update with one command
helm upgrade nginx ./nginx-chart

# Rollback easily
helm rollback nginx

# Clean up completely
helm uninstall nginx
```

### When to Use What

| Use Kubectl When | Use Helm When |
|------------------|---------------|
| Learning Kubernetes | Production deployments |
| Simple single resources | Complex multi-resource apps |
| Quick testing | Multi-environment setup |
| Debugging | Team collaboration |

---

## Helm Chart Structure

### Basic Chart Layout

```
my-chart/
├── Chart.yaml              # Chart metadata
├── values.yaml             # Default configuration values
├── charts/                 # Dependency charts
└── templates/              # Kubernetes manifest templates
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── _helpers.tpl        # Template helpers
```

### Key Files

**Chart.yaml:**
```yaml
apiVersion: v2
name: my-app
description: A Helm chart for my application
version: 1.0.0
appVersion: "1.0"
```

**values.yaml:**
```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.21"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80
```

**templates/deployment.yaml:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
      - name: {{ .Chart.Name }}
        image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
        ports:
        - containerPort: {{ .Values.service.port }}
```

### Templating Example

**Instead of hardcoding:**
```yaml
replicas: 3
image: nginx:1.21
```

**Use references:**
```yaml
replicas: {{ .Values.replicaCount }}
image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
```

**Benefits:**
- Change values in one place (values.yaml)
- Reuse chart for different environments
- Override values at install time

---

## Local Kubernetes Setup

### Prerequisites

You need a local Kubernetes cluster. Choose one:

### Option 1: Kind (Kubernetes in Docker)

**Advantages:**
- ✅ Lightweight
- ✅ Fast startup
- ✅ Multi-node clusters
- ✅ Works with Helm perfectly

**Installation:**
```bash
# Install Kind
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create cluster
kind create cluster --name helm-demo

# Verify
kubectl cluster-info
kubectl get nodes
```

**Helm compatibility:** ✅ Works out of the box!

### Option 2: Minikube

**Installation:**
```bash
# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start cluster
minikube start

# Verify
kubectl get nodes
```

### Option 3: MicroK8s

**Installation (Ubuntu):**
```bash
# Install
sudo snap install microk8s --classic

# Add user to group
sudo usermod -a -G microk8s $USER
sudo chown -f -R $USER ~/.kube
newgrp microk8s

# Enable Helm
microk8s enable helm

# Enable other add-ons (optional)
microk8s enable dns storage ingress dashboard
```

**Configure kubectl:**
```bash
mkdir -p $HOME/.kube
cd $HOME/.kube
microk8s config > config
export KUBECONFIG=$HOME/.kube/config
```

### Verify Cluster

```bash
# Check all resources
kubectl get all --all-namespaces

# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system
```

---

## Installing Helm

### Installation Methods

**Method 1: Script (Recommended)**
```bash
# Download and install specific version
curl -L https://git.io/get_helm.sh | bash -s -- --version v3.8.2

# Or latest version
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**Method 2: Package Manager**

**Debian/Ubuntu:**
```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

**RHEL/CentOS:**
```bash
sudo dnf install helm
```

**macOS:**
```bash
brew install helm
```

### Verify Installation

```bash
# Check version
helm version

# Output:
# version.BuildInfo{Version:"v3.8.2", GitCommit:"...", GitTreeState:"clean", GoVersion:"go1.17"}

# Check installation location
which helm
# /usr/local/bin/helm

# List files
ls -lart /usr/local/bin/ | grep helm
```

---

## Quick Start

### Create Your First Chart

```bash
# Create new chart
helm create helloworld

# View structure
tree helloworld

# Output:
# helloworld/
# ├── Chart.yaml
# ├── values.yaml
# ├── charts/
# └── templates/
#     ├── deployment.yaml
#     ├── service.yaml
#     ├── ingress.yaml
#     └── _helpers.tpl
```

### Deploy the Chart

**1. Modify values (optional):**
```bash
cd helloworld
vi values.yaml

# Change service type to NodePort
service:
  type: NodePort
  port: 80
```

**2. Install chart:**
```bash
cd ..
helm install myhelloworld ./helloworld

# Syntax:
# helm install <release-name> <chart-directory>
```

**3. Verify deployment:**
```bash
# List Helm releases
helm list -a

# Check Kubernetes resources
kubectl get all
kubectl get pods
kubectl get services
kubectl get deployments

# Detailed service info
kubectl describe service myhelloworld
```

### Access the Application

**Get NodePort:**
```bash
kubectl get service myhelloworld -o wide

# Output:
# NAME            TYPE       PORT(S)        NODE-PORT
# myhelloworld   NodePort   80:31234/TCP   ...
```

**Get Node IP:**

**For Minikube:**
```bash
minikube ip
# Example: 192.168.49.2
```

**For Kind:**
```bash
kubectl get nodes -o wide
# Use the INTERNAL-IP
```

**Test the service:**
```bash
# Replace with your node IP and port
curl http://192.168.49.2:31234

# Or use port forwarding
kubectl port-forward service/myhelloworld 8080:80

# Then access
curl http://localhost:8080
```

### Override NodePort

**In values.yaml:**
```yaml
service:
  type: NodePort
  port: 80
  nodePort: 31000  # Specify exact port
```

---

## Basic Helm Commands

### Essential Commands

```bash
# Create chart
helm create <chart-name>

# Install chart
helm install <release-name> <chart-path>

# List releases
helm list
helm list -a  # Include uninstalled

# Upgrade release
helm upgrade <release-name> <chart-path>

# Rollback release
helm rollback <release-name> <revision>

# Uninstall release
helm uninstall <release-name>

# Get release info
helm status <release-name>
helm get values <release-name>
helm get manifest <release-name>
```

---

## Summary

**Helm simplifies Kubernetes by:**
- ✅ Packaging multiple YAMLs into one chart
- ✅ Templating to avoid hardcoded values
- ✅ Version control for deployments
- ✅ Easy sharing via repositories

**Setup checklist:**
- ✅ Install Kubernetes (Kind/Minikube/MicroK8s)
- ✅ Install Helm
- ✅ Create and deploy first chart
- ✅ Verify with kubectl

**Next steps:**
- Explore Helm chart architecture
- Learn templating and values
- Use Helm repositories
- Deploy production applications

---

This guide covers Helm introduction, setup, and getting started with your first deployment.