# Terraform Advanced Providers - AWS, Kubernetes, GCP, Helm

Complete guide to Terraform providers for multi-cloud infrastructure provisioning with AWS, Kubernetes, GCP, and Helm.

## Table of Contents
- [AWS Provider Examples](#aws-provider-examples)
- [Kubernetes Provider](#kubernetes-provider)
- [GCP Provider](#gcp-provider)
- [Helm Provider](#helm-provider)
- [Multi-Provider Architecture](#multi-provider-architecture)

---

## AWS Provider Examples

### Complete EC2 and S3 Setup

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
  # Better: Use environment variables
  # AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY
}

# S3 Bucket
resource "aws_s3_bucket" "data_bucket" {
  bucket = "my-data-bucket-12345"
  
  tags = {
    Name        = "Data Bucket"
    Environment = "Production"
  }
}

resource "aws_s3_bucket_acl" "bucket_acl" {
  bucket = aws_s3_bucket.data_bucket.id
  acl    = "private"
}

# EC2 Instance
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = "my-key-pair"  # IAM user with EC2 permissions
  
  tags = {
    Name = "Web Server"
    Num  = var.num_var
  }
  
  # Security group
  vpc_security_group_ids = [aws_security_group.web_sg.id]
}

# Security Group
resource "aws_security_group" "web_sg" {
  name        = "web-server-sg"
  description = "Security group for web server"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Outputs for AWS Resources

**outputs.tf:**
```hcl
output "s3_hosted_zone_id" {
  description = "S3 bucket hosted zone ID"
  value       = aws_s3_bucket.data_bucket.hosted_zone_id
}

output "s3_bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.data_bucket.arn
}

output "instance_public_ip" {
  description = "EC2 instance public IP"
  value       = aws_instance.web_server.public_ip
}

output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web_server.id
}
```

**Access outputs:**
```bash
terraform apply
terraform output instance_public_ip
terraform output -json
```

---

## Kubernetes Provider

### Kubernetes Configuration

```hcl
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "minikube"  # or "kind-cluster", "gke_project_zone_cluster"
}
```

### Namespace

```hcl
resource "kubernetes_namespace" "app_namespace" {
  metadata {
    name = "production"
    
    labels = {
      environment = "production"
      managed_by  = "terraform"
    }
  }
}
```

### Deployment

```hcl
resource "kubernetes_deployment" "web_app" {
  metadata {
    name      = "web-deployment"
    namespace = kubernetes_namespace.app_namespace.metadata[0].name
    
    labels = {
      app = "web"
    }
  }

  spec {
    replicas = 3
    
    selector {
      match_labels = {
        app = "web"
      }
    }

    template {
      metadata {
        labels = {
          app = "web"
        }
      }

      spec {
        container {
          name  = "nginx"
          image = "nginx:1.21"
          
          port {
            container_port = 80
          }
          
          resources {
            limits = {
              cpu    = "500m"
              memory = "512Mi"
            }
            requests = {
              cpu    = "250m"
              memory = "256Mi"
            }
          }
        }
      }
    }
  }
}
```

### Service

```hcl
resource "kubernetes_service" "web_service" {
  metadata {
    name      = "web-service"
    namespace = kubernetes_namespace.app_namespace.metadata[0].name
  }

  spec {
    selector = {
      app = "web"  # Matches deployment labels
    }

    port {
      port        = 80
      target_port = 80
    }

    type = "LoadBalancer"
  }
}
```

### Complete Kubernetes Stack

```hcl
# Namespace
resource "kubernetes_namespace" "myapp" {
  metadata {
    name = "myapp-prod"
  }
}

# ConfigMap
resource "kubernetes_config_map" "app_config" {
  metadata {
    name      = "app-config"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }

  data = {
    DATABASE_URL = "postgres://db:5432"
    API_URL      = "https://api.example.com"
  }
}

# Secret
resource "kubernetes_secret" "app_secret" {
  metadata {
    name      = "app-secret"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }

  data = {
    db_password = base64encode("supersecret")
  }
}

# Deployment
resource "kubernetes_deployment" "backend" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
    labels = {
      app = "backend"
    }
  }

  spec {
    replicas = 2

    selector {
      match_labels = {
        app = "backend"
      }
    }

    template {
      metadata {
        labels = {
          app = "backend"
        }
      }

      spec {
        container {
          name  = "api"
          image = "myapp/backend:v1.0"

          env_from {
            config_map_ref {
              name = kubernetes_config_map.app_config.metadata[0].name
            }
          }

          env {
            name = "DB_PASSWORD"
            value_from {
              secret_key_ref {
                name = kubernetes_secret.app_secret.metadata[0].name
                key  = "db_password"
              }
            }
          }

          port {
            container_port = 5000
          }
        }
      }
    }
  }
}

# Service
resource "kubernetes_service" "backend_service" {
  metadata {
    name      = "backend"
    namespace = kubernetes_namespace.myapp.metadata[0].name
  }

  spec {
    selector = {
      app = "backend"
    }

    port {
      port        = 5000
      target_port = 5000
    }

    type = "ClusterIP"
  }
}
```

**Apply:**
```bash
terraform init
terraform plan
terraform apply -auto-approve
```

---

## GCP Provider

### GCP Configuration

```hcl
provider "google" {
  project     = "my-project-id"
  region      = "us-central1"
  credentials = file("~/.gcp/service-account-key.json")
}
```

### Static IP for LoadBalancer

```hcl
resource "google_compute_address" "lb_ip" {
  name   = "web-lb-ip"
  region = "us-central1"
}

resource "kubernetes_service" "web_lb" {
  metadata {
    name      = "web-loadbalancer"
    namespace = kubernetes_namespace.app_namespace.metadata[0].name
  }

  spec {
    selector = {
      app = "web"
    }

    port {
      port        = 80
      target_port = 80
    }

    type             = "LoadBalancer"
    load_balancer_ip = google_compute_address.lb_ip.address
  }
}

output "loadbalancer_ip" {
  description = "LoadBalancer IP address"
  value       = google_compute_address.lb_ip.address
}
```

### GKE Cluster

```hcl
resource "google_container_cluster" "primary" {
  name     = "my-gke-cluster"
  location = "us-central1-a"

  # Node pool configuration
  initial_node_count = 3

  node_config {
    machine_type = "e2-medium"
    disk_size_gb = 20

    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}

# Configure kubectl after cluster creation
output "cluster_endpoint" {
  value = google_container_cluster.primary.endpoint
}

output "cluster_ca_certificate" {
  value     = google_container_cluster.primary.master_auth[0].cluster_ca_certificate
  sensitive = true
}
```

---

## Helm Provider

### Helm Configuration

```hcl
provider "helm" {
  kubernetes {
    config_path    = "~/.kube/config"
    config_context = "minikube"
  }
}
```

### Deploy Chart from Repository

```hcl
# Deploy NGINX from Bitnami
resource "helm_release" "nginx" {
  name       = "nginx"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "nginx"
  version    = "13.2.24"
  namespace  = "default"

  set {
    name  = "service.type"
    value = "LoadBalancer"
  }

  set {
    name  = "replicaCount"
    value = "3"
  }
}

# Deploy Prometheus
resource "helm_release" "prometheus" {
  name       = "prometheus"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "prometheus"
  namespace  = "monitoring"
  
  create_namespace = true

  values = [
    file("${path.module}/prometheus-values.yaml")
  ]
}
```

### Deploy Local Chart

**Project structure:**
```
terraform/
├── main.tf
└── charts/
    └── mychart/
        ├── Chart.yaml
        ├── templates/
        │   ├── deployment.yaml
        │   └── service.yaml
        └── values.yaml
```

**main.tf:**
```hcl
provider "helm" {
  kubernetes {
    config_path = "~/.kube/config"
  }
}

resource "helm_release" "local_app" {
  name      = "my-local-app"
  chart     = "./charts/mychart"
  namespace = "default"

  values = [
    file("./charts/mychart/values.yaml")
  ]
  
  set {
    name  = "image.tag"
    value = "v1.2.3"
  }
}
```

### Helm with Custom Values

```hcl
resource "helm_release" "app" {
  name       = "myapp"
  repository = "https://charts.example.com"
  chart      = "myapp"
  namespace  = "production"
  
  create_namespace = true

  # Load values file
  values = [
    file("${path.module}/values/production.yaml")
  ]

  # Override specific values
  set {
    name  = "image.repository"
    value = "myregistry/myapp"
  }

  set {
    name  = "image.tag"
    value = var.app_version
  }

  set {
    name  = "ingress.enabled"
    value = "true"
  }

  set {
    name  = "ingress.hosts[0]"
    value = "app.example.com"
  }
}
```

### Artifact Hub Charts

```hcl
# MySQL from Bitnami
resource "helm_release" "mysql" {
  name       = "mysql"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "mysql"
  namespace  = "database"
  
  create_namespace = true

  set {
    name  = "auth.rootPassword"
    value = var.mysql_root_password
  }

  set {
    name  = "primary.persistence.size"
    value = "10Gi"
  }
}

# Redis
resource "helm_release" "redis" {
  name       = "redis"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "redis"
  namespace  = "cache"
  
  create_namespace = true

  set {
    name  = "replica.replicaCount"
    value = "2"
  }
}
```

---

## Multi-Provider Architecture

### Complete Stack Example

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.0"
    }
    google = {
      source  = "hashicorp/google"
      version = "~> 4.0"
    }
  }
}

# AWS Provider
provider "aws" {
  region = var.aws_region
}

# GCP Provider
provider "google" {
  project = var.gcp_project
  region  = var.gcp_region
}

# Kubernetes Provider (points to GKE)
provider "kubernetes" {
  host                   = google_container_cluster.primary.endpoint
  cluster_ca_certificate = base64decode(google_container_cluster.primary.master_auth[0].cluster_ca_certificate)
  
  exec {
    api_version = "client.authentication.k8s.io/v1beta1"
    command     = "gke-gcloud-auth-plugin"
  }
}

# Helm Provider
provider "helm" {
  kubernetes {
    host                   = google_container_cluster.primary.endpoint
    cluster_ca_certificate = base64decode(google_container_cluster.primary.master_auth[0].cluster_ca_certificate)
    
    exec {
      api_version = "client.authentication.k8s.io/v1beta1"
      command     = "gke-gcloud-auth-plugin"
    }
  }
}

# Create GKE Cluster
resource "google_container_cluster" "primary" {
  name     = "production-cluster"
  location = "us-central1-a"
  
  initial_node_count = 3
  
  node_config {
    machine_type = "e2-standard-2"
  }
}

# Deploy application with Helm
resource "helm_release" "app" {
  name      = "myapp"
  chart     = "./charts/myapp"
  namespace = "production"
  
  create_namespace = true
  
  depends_on = [google_container_cluster.primary]
}

# AWS S3 for backups
resource "aws_s3_bucket" "backups" {
  bucket = "myapp-backups-${var.environment}"
  
  tags = {
    Environment = var.environment
  }
}
```

### Environment-Specific Deployments

**variables.tf:**
```hcl
variable "environment" {
  type = string
}

variable "kubernetes_context" {
  type = string
}

variable "app_replicas" {
  type = map(number)
  default = {
    dev     = 1
    staging = 2
    prod    = 3
  }
}
```

**main.tf:**
```hcl
provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = var.kubernetes_context
}

resource "kubernetes_namespace" "env" {
  metadata {
    name = var.environment
  }
}

resource "kubernetes_deployment" "app" {
  metadata {
    name      = "app"
    namespace = kubernetes_namespace.env.metadata[0].name
  }
  
  spec {
    replicas = var.app_replicas[var.environment]
    # ... rest of spec
  }
}
```

**Deploy to different environments:**
```bash
# Development
terraform workspace new dev
terraform apply -var="environment=dev" -var="kubernetes_context=dev-cluster"

# Staging
terraform workspace new staging
terraform apply -var="environment=staging" -var="kubernetes_context=staging-cluster"

# Production
terraform workspace new prod
terraform apply -var="environment=prod" -var="kubernetes_context=prod-cluster"
```

---

## Quick Reference

### AWS Resources
```bash
terraform apply                    # Create
terraform output instance_public_ip # Get IP
terraform destroy                   # Cleanup
```

### Kubernetes Resources
```bash
terraform apply -auto-approve
kubectl get all -n production     # Verify
terraform destroy
```

### Helm Charts
```bash
terraform apply
helm list                          # Verify
terraform destroy
```

---

This guide covers advanced Terraform provider configurations for AWS, Kubernetes, GCP, and Helm deployments.