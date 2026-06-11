# GCP Infrastructure as Code: Deployment Manager & Terraform

Infrastructure as Code (IaC) lets you define and provision your entire GCP infrastructure using configuration files. The GCP-native equivalent of AWS CloudFormation is **Deployment Manager**, but in practice, **Terraform** is the most widely used IaC tool for GCP — and even Google recommends it.

---

## IaC Options for GCP

| Tool | Type | Best For |
|------|------|---------|
| **Terraform** (recommended) | Multi-cloud IaC | All GCP workloads; industry standard |
| **Deployment Manager** | GCP-native IaC | GCP-only; legacy projects |
| **Config Connector** | Kubernetes-native | Managing GCP resources via `kubectl` |
| **Pulumi** | Code-based IaC (Python/TypeScript) | Developers who prefer real code over HCL |

> For new projects, **use Terraform**. Deployment Manager is functionally equivalent to CloudFormation but Terraform has a far larger ecosystem, better community support, and is the GCP team's own recommendation for IaC.

---

## Part 1: Terraform for GCP

Terraform by HashiCorp is to multi-cloud what CloudFormation is to AWS — but it works across GCP, AWS, Azure, and hundreds of other providers.

### Installation

```bash
# macOS
brew tap hashicorp/tap && brew install hashicorp/tap/terraform

# Linux
wget https://releases.hashicorp.com/terraform/1.8.0/terraform_1.8.0_linux_amd64.zip
unzip terraform_1.8.0_linux_amd64.zip && sudo mv terraform /usr/local/bin/

# Verify
terraform --version
```

### Authentication

```bash
# For local development
gcloud auth application-default login

# For CI/CD — set GOOGLE_APPLICATION_CREDENTIALS
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json
```

---

### Basic Terraform Structure

```
my-infra/
├── main.tf          # Resource definitions
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── terraform.tfvars # Variable values (gitignored if secrets)
└── backend.tf       # Remote state config (GCS bucket)
```

### `main.tf` — Example GCP Infrastructure

```hcl
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 5.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
}

# Cloud Storage bucket
resource "google_storage_bucket" "app_assets" {
  name          = "${var.project_id}-app-assets"
  location      = "US"
  force_destroy = false

  uniform_bucket_level_access = true

  lifecycle_rule {
    action { type = "SetStorageClass"; storage_class = "NEARLINE" }
    condition { age = 30 }
  }
}

# Cloud SQL (PostgreSQL)
resource "google_sql_database_instance" "main" {
  name             = "my-postgres"
  database_version = "POSTGRES_16"
  region           = var.region

  settings {
    tier              = "db-n1-standard-2"
    availability_type = "REGIONAL"    # HA

    backup_configuration {
      enabled    = true
      start_time = "02:00"
      point_in_time_recovery_enabled = true
    }

    ip_configuration {
      ipv4_enabled    = false
      private_network = google_compute_network.vpc.id
    }
  }
}

resource "google_sql_database" "app_db" {
  name     = "appdb"
  instance = google_sql_database_instance.main.name
}

# Cloud Run service
resource "google_cloud_run_v2_service" "app" {
  name     = "my-app"
  location = var.region

  template {
    service_account = google_service_account.app_sa.email

    containers {
      image = "us-central1-docker.pkg.dev/${var.project_id}/my-repo/my-app:latest"

      resources {
        limits = {
          cpu    = "1"
          memory = "512Mi"
        }
      }

      env {
        name  = "NODE_ENV"
        value = "production"
      }

      env {
        name = "DB_PASSWORD"
        value_source {
          secret_key_ref {
            secret  = google_secret_manager_secret.db_password.secret_id
            version = "latest"
          }
        }
      }
    }

    scaling {
      min_instance_count = 1
      max_instance_count = 100
    }
  }
}

# Allow public access to Cloud Run
resource "google_cloud_run_service_iam_member" "public" {
  location = google_cloud_run_v2_service.app.location
  service  = google_cloud_run_v2_service.app.name
  role     = "roles/run.invoker"
  member   = "allUsers"
}

# Service Account for the app
resource "google_service_account" "app_sa" {
  account_id   = "my-app-sa"
  display_name = "My App Service Account"
}

# Grant service account access to storage bucket
resource "google_storage_bucket_iam_member" "app_bucket_access" {
  bucket = google_storage_bucket.app_assets.name
  role   = "roles/storage.objectAdmin"
  member = "serviceAccount:${google_service_account.app_sa.email}"
}

# Secret Manager secret
resource "google_secret_manager_secret" "db_password" {
  secret_id = "db-password"
  replication {
    auto {}
  }
}
```

### `variables.tf`
```hcl
variable "project_id" {
  description = "GCP Project ID"
  type        = string
}

variable "region" {
  description = "GCP region"
  type        = string
  default     = "us-central1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "staging"
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### `outputs.tf`
```hcl
output "cloud_run_url" {
  description = "Cloud Run service URL"
  value       = google_cloud_run_v2_service.app.uri
}

output "bucket_name" {
  description = "App assets bucket name"
  value       = google_storage_bucket.app_assets.name
}
```

### `backend.tf` — Store State in GCS

```hcl
terraform {
  backend "gcs" {
    bucket = "my-project-terraform-state"
    prefix = "my-app/prod"           # Separate state per env
  }
}
```

```bash
# Create the state bucket first (once)
gcloud storage buckets create gs://my-project-terraform-state \
  --location=us-central1 \
  --uniform-bucket-level-access
```

---

### Terraform Workflow

```bash
# Initialize (download providers, set up backend)
terraform init

# Preview changes (dry run)
terraform plan -var="project_id=my-project-id"

# Apply changes
terraform apply -var="project_id=my-project-id"

# Apply without confirmation prompt (CI/CD)
terraform apply -auto-approve -var="project_id=my-project-id"

# Destroy all resources
terraform destroy -var="project_id=my-project-id"

# Show current state
terraform show

# List all managed resources
terraform state list

# Import an existing resource into Terraform state
terraform import google_storage_bucket.app_assets my-project-id-app-assets
```

---

## Part 2: GCP Deployment Manager (Native IaC)

Deployment Manager is GCP's native IaC tool. Like CloudFormation, you define resources in YAML templates and Google manages the stack lifecycle.

### Basic Template
```yaml
# infra.yaml
resources:
  - name: my-storage-bucket
    type: storage.v1.bucket
    properties:
      location: US
      storageClass: STANDARD
      iamConfiguration:
        uniformBucketLevelAccess:
          enabled: true

  - name: my-pubsub-topic
    type: pubsub.v1.topic
    properties:
      topic: my-topic

  - name: my-pubsub-subscription
    type: pubsub.v1.subscription
    properties:
      subscription: my-subscription
      topic: $(ref.my-pubsub-topic.name)
      ackDeadlineSeconds: 60
```

### CLI Commands
```bash
# Preview a deployment
gcloud deployment-manager deployments create my-deployment \
  --config=infra.yaml \
  --preview

# Create/apply
gcloud deployment-manager deployments create my-deployment \
  --config=infra.yaml

# Update
gcloud deployment-manager deployments update my-deployment \
  --config=infra.yaml

# List deployments
gcloud deployment-manager deployments list

# Describe a deployment (see all resources)
gcloud deployment-manager deployments describe my-deployment

# Delete a deployment (deletes all resources in it)
gcloud deployment-manager deployments delete my-deployment
```

---

## Terraform vs Deployment Manager

| Feature | Terraform | Deployment Manager |
|---------|-----------|-------------------|
| Multi-cloud | ✅ | ❌ GCP only |
| Community/ecosystem | Huge | Small |
| State management | Local or remote (GCS) | Google-managed |
| Language | HCL | YAML / Python / Jinja2 |
| Drift detection | ✅ | Limited |
| Google recommendation | ✅ Preferred | Legacy |
| Modules/reuse | ✅ Excellent (Terraform Registry) | Limited |

> **Bottom line**: Use **Terraform** for all new GCP infrastructure. Deployment Manager is the CloudFormation equivalent but Terraform is what the industry and Google themselves recommend.