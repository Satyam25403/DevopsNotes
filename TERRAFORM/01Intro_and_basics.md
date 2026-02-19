# Terraform - Infrastructure as Code

Complete guide to Terraform for infrastructure provisioning and management across multiple cloud providers.

## Table of Contents
- [What is Terraform](#what-is-terraform)
- [Installation and Setup](#installation-and-setup)
- [Core Concepts](#core-concepts)
- [AWS Provider Configuration](#aws-provider-configuration)
- [Basic Resources](#basic-resources)
- [Variables](#variables)
- [Outputs](#outputs)
- [State Management](#state-management)
- [Essential Commands](#essential-commands)

---

## What is Terraform?

**Terraform** is an open-source Infrastructure as Code (IaC) tool by HashiCorp.

**How it works:**
- Configuration files written in HCL (HashiCorp Configuration Language)
- Declarative syntax (describe desired state)
- Cloud-agnostic (works across multiple providers)

**What it does:**
- Provisions infrastructure
- Updates existing resources
- Manages infrastructure lifecycle
- Tracks infrastructure state

### Multi-Cloud Power

**Cloud-agnostic capabilities:**

**AWS:**
- EC2, S3, IAM, VPC, RDS, Lambda
- ELB, Route53, CloudFront

**Azure:**
- VMs, Resource Groups
- Networking, Storage
- AKS, App Services

**Google Cloud:**
- GKE, Compute Engine
- Cloud Storage, Cloud SQL
- Load Balancers

**Others:**
- Kubernetes
- VMware
- OpenStack
- On-premises infrastructure

### Why Teams Love Terraform

**✅ Version Control:**
- Track infrastructure changes in Git
- Review infrastructure code
- Rollback capabilities

**✅ Reusable Modules:**
- DRY principle for infrastructure
- Shared modules across teams
- Community modules

**✅ Declarative Syntax:**
- Describe desired end state
- Terraform figures out how to get there
- No imperative scripting

**✅ State Tracking:**
- Knows what's deployed
- Detects drift
- Plans changes intelligently

**✅ Automation-Friendly:**
- CI/CD integration
- Automated deployments
- Infrastructure pipelines

---

## Installation and Setup

### Install Terraform

**Linux:**
```bash
# Download
wget https://releases.hashicorp.com/terraform/1.6.0/terraform_1.6.0_linux_amd64.zip

# Unzip
unzip terraform_1.6.0_linux_amd64.zip

# Move to PATH
sudo mv terraform /usr/local/bin/

# Verify
terraform --version
```

**macOS:**
```bash
brew install terraform
terraform --version
```

**Windows:**
```powershell
choco install terraform
terraform --version
```

### Cloud Provider Setup

**AWS Setup:**
```bash
# Create IAM user with programmatic access
# Get access key and secret key
# Attach policies: EC2FullAccess, S3FullAccess, etc.
```

**GCP Setup:**
```bash
# Create service account
# Download JSON key file
# Set GOOGLE_APPLICATION_CREDENTIALS
```

**Azure Setup:**
```bash
# Create service principal
# Get credentials (client_id, client_secret, tenant_id)
```

---

## Core Concepts

### Terraform File Structure

```
terraform-project/
├── main.tf              # Main configuration
├── variables.tf         # Input variables
├── outputs.tf           # Output values
├── terraform.tfvars     # Variable values
├── .terraform/          # Provider plugins (auto-generated)
├── terraform.tfstate    # State file (auto-generated)
└── .terraform.lock.hcl  # Dependency lock (auto-generated)
```

**⚠️ Never manually edit:**
- `.terraform/` directory
- `terraform.tfstate` file
- `.terraform.lock.hcl` file

### Three Parts of Terraform Scripts

**1. Provider:**
- Connects to backend (cloud provider)
- Multiple providers supported
- Required for resource creation

**2. Resources:**
- Infrastructure components
- Multiple resources per configuration
- Dependencies managed automatically

**3. Miscellaneous:**
- Variables
- Modules
- Functions
- Data sources

---

## AWS Provider Configuration

### Basic Provider Setup

**main.tf:**
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
  region     = "ap-south-1"
  access_key = "YOUR_ACCESS_KEY"
  secret_key = "YOUR_SECRET_KEY"
}
```

**⚠️ Better: Use environment variables:**
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
```

**Then:**
```hcl
provider "aws" {
  region = "ap-south-1"
}
```

### Initialize Terraform

```bash
# Create directory
mkdir terraform-aws
cd terraform-aws

# Create main.tf with provider config

# Initialize
terraform init

# Creates:
# - .terraform/ directory
# - .terraform.lock.hcl file
```

---

## Basic Resources

### S3 Bucket

```hcl
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name-12345"
  
  tags = {
    Name        = "My Bucket"
    Environment = "Dev"
  }
}

resource "aws_s3_bucket_acl" "bucket_acl" {
  bucket = aws_s3_bucket.my_bucket.id
  acl    = "private"
}
```

**Note:** `my_bucket` is Terraform's resource name (for reference), not the actual bucket name.

### EC2 Instance

```hcl
resource "aws_instance" "web_server" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  key_name      = "my-key-pair"
  
  tags = {
    Name = "Web Server"
    Env  = "Production"
  }
}
```

### Complete Example

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
  region = "us-east-1"
}

resource "aws_instance" "web" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = "t2.micro"
  
  tags = {
    Name = "HelloWorld"
  }
}
```

**Apply:**
```bash
terraform init
terraform apply

# Creates terraform.tfstate file
```

**Destroy:**
```bash
terraform destroy
```

---

## Variables

### Input Variables

**variables.tf:**
```hcl
variable "instance_type" {
  description = "Type of EC2 instance"
  type        = string
  default     = "t2.micro"
}

variable "num_var" {
  description = "Number variable"
  type        = number
  default     = 27
}

variable "access_key" {
  description = "AWS Access Key"
  type        = string
  sensitive   = true
}

variable "secret_key" {
  description = "AWS Secret Key"
  type        = string
  sensitive   = true
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### Using Variables

**main.tf:**
```hcl
provider "aws" {
  region     = "ap-south-1"
  access_key = var.access_key
  secret_key = var.secret_key
}

resource "aws_instance" "example" {
  ami           = "ami-0c94855ba95c71c99"
  instance_type = var.instance_type
  
  count = var.num_var > 100 ? 2 : 1
  
  tags = {
    Environment = var.environment
  }
}
```

### Variable Values File

**variables.tfvars:**
```hcl
num_var       = 300
access_key    = "AKIAIOSFODNN7EXAMPLE"
secret_key    = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
instance_type = "t2.small"
environment   = "prod"
```

**Apply with variables:**
```bash
terraform plan -var-file="variables.tfvars"
terraform apply -var-file="variables.tfvars"
```

### Override Variables

```bash
# Command line
terraform apply -var="instance_type=t2.small"

# Environment variable
export TF_VAR_instance_type="t2.small"
terraform apply

# Multiple variables
terraform apply \
  -var="instance_type=t2.small" \
  -var="environment=staging"
```

### Variable Types

```hcl
# String
variable "region" {
  type    = string
  default = "us-east-1"
}

# Number
variable "port" {
  type    = number
  default = 8080
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t2.large"
  }
}

# Object
variable "instance_config" {
  type = object({
    ami           = string
    instance_type = string
  })
  default = {
    ami           = "ami-123"
    instance_type = "t2.micro"
  }
}
```

---

## Outputs

### Output Values

**outputs.tf:**
```hcl
output "instance_ip" {
  description = "Public IP of EC2 instance"
  value       = aws_instance.web_server.public_ip
}

output "instance_id" {
  description = "ID of EC2 instance"
  value       = aws_instance.web_server.id
}

output "s3_bucket_name" {
  description = "Name of S3 bucket"
  value       = aws_s3_bucket.my_bucket.id
}

output "s3_bucket_arn" {
  description = "ARN of S3 bucket"
  value       = aws_s3_bucket.my_bucket.arn
}
```

### View Outputs

```bash
# After apply, outputs shown automatically

# View specific output
terraform output instance_ip

# View all outputs
terraform output

# JSON format
terraform output -json
```

### Sensitive Outputs

```hcl
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true
}
```

---

## State Management

### Terraform State

**terraform.tfstate:**
- Tracks current infrastructure state
- Maps resources to real-world objects
- Metadata and resource dependencies

**⚠️ Important:**
- Never manually edit
- Contains sensitive data
- Should be stored remotely in production

### Remote State

**AWS S3 Backend:**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
    
    # State locking
    dynamodb_table = "terraform-locks"
  }
}
```

### State Commands

```bash
# List resources
terraform state list

# Show resource details
terraform state show aws_instance.web

# Remove resource from state
terraform state rm aws_instance.web

# Move resource
terraform state mv aws_instance.web aws_instance.new_web

# Pull remote state
terraform state pull

# Push local state
terraform state push
```

---

## Essential Commands

### Basic Workflow

```bash
# Initialize
terraform init

# Format code
terraform fmt

# Validate configuration
terraform validate

# Plan changes
terraform plan

# Apply changes
terraform apply

# Destroy everything
terraform destroy
```

### Plan and Apply

```bash
# Show plan
terraform plan

# Save plan
terraform plan -out=tfplan

# Apply saved plan
terraform apply tfplan

# Auto-approve
terraform apply -auto-approve
```

### Validation

```bash
# Validate syntax
terraform validate

# Format files
terraform fmt

# Format recursively
terraform fmt -recursive
```

### Targeting

```bash
# Plan specific resource
terraform plan -target=aws_instance.web

# Apply specific resource
terraform apply -target=aws_instance.web

# Destroy specific resource
terraform destroy -target=aws_instance.web
```

### Refresh and Import

```bash
# Refresh state
terraform refresh

# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

---

## Complete Example

**Project structure:**
```
aws-infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
└── terraform.tfvars
```

**main.tf:**
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
  region = var.aws_region
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name        = "WebServer"
    Environment = var.environment
  }
}

resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name
  
  tags = {
    Name        = "DataBucket"
    Environment = var.environment
  }
}
```

**variables.tf:**
```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "ami_id" {
  description = "AMI ID"
  type        = string
}

variable "instance_type" {
  description = "Instance type"
  type        = string
  default     = "t2.micro"
}

variable "bucket_name" {
  description = "S3 bucket name"
  type        = string
}

variable "environment" {
  description = "Environment"
  type        = string
}
```

**outputs.tf:**
```hcl
output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "s3_bucket_arn" {
  value = aws_s3_bucket.data.arn
}
```

**terraform.tfvars:**
```hcl
ami_id        = "ami-0c94855ba95c71c99"
instance_type = "t2.small"
bucket_name   = "my-data-bucket-12345"
environment   = "production"
```

**Deploy:**
```bash
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars"
```

---

This guide covers Terraform fundamentals for infrastructure provisioning across cloud providers.