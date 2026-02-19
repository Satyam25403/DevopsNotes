# Terraform Production Best Practices

Production-ready Terraform workflows, modules, state management, CI/CD integration, and enterprise patterns.

## Table of Contents
- [Module Development](#module-development)
- [Remote State Management](#remote-state-management)
- [Workspaces](#workspaces)
- [CI/CD Integration](#cicd-integration)
- [Security Best Practices](#security-best-practices)
- [Testing](#testing)
- [Import Existing Resources](#import-existing-resources)
- [Drift Detection](#drift-detection)

---

## Module Development

### Creating Reusable Modules

**Module structure:**
```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

**modules/vpc/main.tf:**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    var.tags,
    {
      Name = var.vpc_name
    }
  )
}

resource "aws_subnet" "public" {
  count                   = length(var.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnets[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-public-${count.index + 1}"
    }
  )
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = "${var.vpc_name}-igw"
    }
  )
}
```

**modules/vpc/variables.tf:**
```hcl
variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}

variable "public_subnets" {
  description = "List of public subnet CIDR blocks"
  type        = list(string)
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/outputs.tf:**
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "vpc_cidr_block" {
  description = "CIDR block of VPC"
  value       = aws_vpc.main.cidr_block
}
```

### Using Modules

**main.tf:**
```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_name           = "production-vpc"
  cidr_block         = "10.0.0.0/16"
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones = ["us-east-1a", "us-east-1b"]

  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_ids[0]
  # ... rest of config
}
```

### Public Module Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "3.19.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Terraform   = "true"
    Environment = "dev"
  }
}
```

---

## Remote State Management

### S3 Backend (AWS)

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**Create S3 bucket and DynamoDB table:**
```hcl
# backend-setup.tf (run separately first)
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### GCS Backend (GCP)

```hcl
terraform {
  backend "gcs" {
    bucket = "my-terraform-state"
    prefix = "production"
  }
}
```

### Azure Backend

```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "terraformstate"
    container_name       = "tfstate"
    key                  = "production.terraform.tfstate"
  }
}
```

### State Locking

**Why locking matters:**
- Prevents concurrent modifications
- Avoids state corruption
- Ensures consistency

**DynamoDB for AWS:**
- Automatically enabled with table
- Lock acquired before operations
- Released after completion

---

## Workspaces

### Managing Multiple Environments

```bash
# List workspaces
terraform workspace list

# Create new workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select dev

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete dev
```

### Workspace-Aware Configuration

```hcl
locals {
  environment = terraform.workspace

  instance_type = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.large"
  }

  replica_count = {
    dev     = 1
    staging = 2
    prod    = 3
  }
}

resource "aws_instance" "app" {
  instance_type = local.instance_type[local.environment]
  
  tags = {
    Environment = local.environment
    Name        = "${local.environment}-app"
  }
}
```

---

## CI/CD Integration

### GitHub Actions

**.github/workflows/terraform.yml:**
```yaml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-east-1

jobs:
  terraform:
    name: Terraform
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.6.0

    - name: Terraform Format
      run: terraform fmt -check

    - name: Terraform Init
      run: terraform init
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    - name: Terraform Validate
      run: terraform validate

    - name: Terraform Plan
      run: terraform plan -out=tfplan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && github.event_name == 'push'
      run: terraform apply -auto-approve tfplan
      env:
        AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
        AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

### GitLab CI

**.gitlab-ci.yml:**
```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

stages:
  - validate
  - plan
  - apply

before_script:
  - terraform init

validate:
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - tfplan

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  only:
    - main
  when: manual
```

### Jenkins

```groovy
pipeline {
    agent any

    parameters {
        choice(name: 'ACTION', choices: ['plan', 'apply', 'destroy'], description: 'Terraform action')
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Environment')
    }

    environment {
        AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Terraform Init') {
            steps {
                sh 'terraform init'
            }
        }

        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'plan' || params.ACTION == 'apply' }
            }
            steps {
                sh 'terraform plan -out=tfplan'
            }
        }

        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                input message: 'Approve apply?'
                sh 'terraform apply tfplan'
            }
        }

        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                input message: 'Approve destroy?'
                sh 'terraform destroy -auto-approve'
            }
        }
    }
}
```

---

## Security Best Practices

### Never Commit Secrets

**.gitignore:**
```
# Local .terraform directories
**/.terraform/*

# .tfstate files
*.tfstate
*.tfstate.*

# Crash log files
crash.log

# Exclude all .tfvars files
*.tfvars
*.tfvars.json

# Ignore override files
override.tf
override.tf.json

# SSH keys
*.pem
*.key

# Backup files
*.backup
```

### Use Environment Variables

```bash
# AWS
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"

# Terraform variables
export TF_VAR_db_password="secret"
export TF_VAR_api_key="secret"
```

### Sensitive Variables

```hcl
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = false
}

output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true
}
```

### Secret Management

**AWS Secrets Manager:**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  # ... rest of config
}
```

**HashiCorp Vault:**
```hcl
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_generic_secret" "db_creds" {
  path = "secret/database/creds"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db_creds.data["username"]
  password = data.vault_generic_secret.db_creds.data["password"]
  # ... rest of config
}
```

---

## Testing

### Terratest (Go)

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformVPC(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/vpc",
        Vars: map[string]interface{}{
            "vpc_name": "test-vpc",
            "cidr_block": "10.0.0.0/16",
        },
    }

    defer terraform.Destroy(t, terraformOptions)

    terraform.InitAndApply(t, terraformOptions)

    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

### Terraform Validate

```bash
# Syntax check
terraform validate

# Format check
terraform fmt -check -recursive

# Security scan with tfsec
tfsec .

# Cost estimation with infracost
infracost breakdown --path .
```

---

## Import Existing Resources

```bash
# Import EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Import S3 bucket
terraform import aws_s3_bucket.data my-bucket-name

# Import RDS instance
terraform import aws_db_instance.main my-db-instance
```

**Write configuration first:**
```hcl
resource "aws_instance" "web" {
  # Configuration matching existing instance
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t2.micro"
  
  # Run terraform plan to verify
}
```

---

## Drift Detection

### Detect Changes

```bash
# Refresh state
terraform refresh

# Show differences
terraform plan

# Generate plan to fix drift
terraform plan -out=fix-drift.plan

# Apply fixes
terraform apply fix-drift.plan
```

### Automated Drift Detection

**GitHub Action:**
```yaml
name: Drift Detection

on:
  schedule:
    - cron: '0 0 * * *'  # Daily

jobs:
  detect-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: hashicorp/setup-terraform@v2
      
      - name: Terraform Init
        run: terraform init
        
      - name: Terraform Plan
        run: terraform plan -detailed-exitcode
        continue-on-error: true
        id: plan
        
      - name: Notify on Drift
        if: steps.plan.outcome == 'failure'
        run: |
          echo "Drift detected!"
          # Send notification (Slack, email, etc.)
```

---

This guide covers production Terraform workflows including modules, remote state, workspaces, CI/CD, security, testing, and drift detection.