# HCL — DevOps Reference Notes

> **HashiCorp Configuration Language** — a declarative, human-readable configuration language used across the HashiCorp ecosystem: Terraform, Vault, Packer, Consul, and Nomad. HCL is designed to be written by humans and parsed by machines, striking a balance between JSON's machine-friendliness and full programming language expressiveness.

---

## Table of Contents

1. [Core Concepts](#1-core-concepts)
2. [HCL Syntax Fundamentals](#2-hcl-syntax-fundamentals)
3. [Expressions, Types & Functions](#3-expressions-types--functions)
4. [Terraform — Project Structure](#4-terraform--project-structure)
5. [Terraform — Providers & Resources](#5-terraform--providers--resources)
6. [Terraform — Variables & Outputs](#6-terraform--variables--outputs)
7. [Terraform — Data Sources & Locals](#7-terraform--data-sources--locals)
8. [Terraform — State Management](#8-terraform--state-management)
9. [Terraform — Modules](#9-terraform--modules)
10. [Terraform — Meta-Arguments](#10-terraform--meta-arguments)
11. [Terraform — Workspace & Environments](#11-terraform--workspace--environments)
12. [Terraform — Common AWS Patterns](#12-terraform--common-aws-patterns)
13. [Vault HCL — Policies & Config](#13-vault-hcl--policies--config)
14. [Packer HCL — Image Builds](#14-packer-hcl--image-builds)
15. [Common Gotchas & Best Practices](#15-common-gotchas--best-practices)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Core Concepts

```
HCL is used by:
  Terraform  → infrastructure provisioning (.tf files)
  Vault      → secrets engine policies, auth config
  Packer     → machine image builds (.pkr.hcl files)
  Consul     → service mesh config
  Nomad      → workload scheduling
```

HCL files are made up of **blocks**, **arguments**, and **expressions**:

```hcl
# Block: type "label" "label" { ... }
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # argument = expression
  instance_type = "t3.micro"
}

# Block type only (no labels)
terraform {
  required_version = ">= 1.5.0"
}
```

- File extension: `.tf` (Terraform), `.pkr.hcl` (Packer), `.hcl` (generic)
- Encoding: UTF-8
- Comments: `#` (single line), `//` (single line), `/* ... */` (multi-line)
- HCL is **not** a programming language — no loops as statements, no mutable state (Terraform adds `for_each`, `count`, `dynamic` as meta-constructs)

---

## 2. HCL Syntax Fundamentals

### Arguments

```hcl
# Simple assignments
name    = "web-server"
count   = 3
enabled = true
tags    = { env = "prod", team = "platform" }
ports   = [80, 443, 8080]
```

### Blocks

```hcl
# Block with no labels
terraform { }

# Block with one label
provider "aws" { }

# Block with two labels (type + name)
resource "aws_s3_bucket" "my_bucket" { }

# Nested blocks
resource "aws_security_group" "web" {
  name = "web-sg"

  ingress {              # nested block — no labels here
    from_port = 80
    to_port   = 80
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Comments

```hcl
# Hash comment — single line (preferred)
// Double-slash comment — single line

/*
  Multi-line block comment
  Used for larger explanations
*/

resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"   # Ubuntu 22.04 LTS
  instance_type = "t3.micro"               # Free tier eligible
}
```

### String literals

```hcl
# Double-quoted strings — support interpolation and escape sequences
name = "my-${var.environment}-server"

# Heredoc — multi-line string
user_data = <<-EOT
  #!/bin/bash
  apt-get update
  apt-get install -y nginx
  systemctl enable nginx
EOT

# Heredoc with indentation stripping (<<- strips leading whitespace)
policy = <<-POLICY
  {
    "Version": "2012-10-17",
    "Statement": []
  }
POLICY
```

---

## 3. Expressions, Types & Functions

### Primitive types

```hcl
# string
name = "production"

# number (integer or float)
replicas = 3
threshold = 0.75

# bool
enabled = true
```

### Collection types

```hcl
# list (ordered, same type)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# map (string keys, same-type values)
tags = {
  Environment = "production"
  Team        = "platform"
  ManagedBy   = "terraform"
}

# set (unordered, unique, same type)
# defined in variable type constraints, not as literals

# tuple (ordered, mixed types) — mostly used in type constraints
# object (named attributes, mixed types) — mostly used in type constraints
```

### String interpolation and templates

```hcl
# Basic interpolation
bucket_name = "myapp-${var.environment}-assets"

# Expression in interpolation
name = "replica-${count.index + 1}"

# Conditional in string
suffix = "app-${var.is_prod ? "prod" : "dev"}"

# Template directives (in strings only)
hosts = <<-EOT
  %{ for host in var.hosts ~}
  ${host}
  %{ endfor ~}
EOT
```

### Conditional expression

```hcl
# condition ? true_val : false_val
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"
min_size      = var.is_prod ? 3 : 1
tags          = var.enable_tagging ? { env = var.environment } : {}
```

### For expressions

```hcl
# List comprehension
upper_names = [for name in var.names : upper(name)]

# Filter
prod_buckets = [for b in var.buckets : b if b.env == "prod"]

# Map comprehension
name_to_id = {for s in var.services : s.name => s.id}

# Flip a map
inverted = {for k, v in var.mapping : v => k}
```

### Splat expressions

```hcl
# Legacy splat — works on list of objects
all_ids = aws_instance.web[*].id

# Full splat — same but clearer
all_arns = [for i in aws_iam_role.workers : i.arn]
```

### Built-in functions

```hcl
# String functions
upper("hello")                   # "HELLO"
lower("HELLO")                   # "hello"
title("hello world")             # "Hello World"
trimspace("  hello  ")           # "hello"
chomp("hello\n")                 # "hello"
format("%-10s %d", "id", 42)     # "id         42"
formatlist("Hello, %s!", names)  # list of formatted strings
replace("hello world", "world", "HCL")  # "hello HCL"
split(",", "a,b,c")              # ["a", "b", "c"]
join(", ", ["a", "b", "c"])      # "a, b, c"
startswith("hello", "hel")       # true
endswith("hello", "llo")         # true
substr("hello", 1, 3)            # "ell"
regex("(\\d+)", "abc123")        # ["123"]
regexall("\\d+", "a1b2c3")       # ["1", "2", "3"]

# Collection functions
length(var.list)                  # count items
contains(var.list, "item")        # true/false
index(var.list, "item")           # position
element(var.list, 2)              # item at index (wraps)
slice(var.list, 0, 3)             # sublist
flatten([["a","b"],["c"]])        # ["a","b","c"]
compact(["a","","b"])             # ["a","b"] (remove empty strings)
distinct(["a","a","b"])           # ["a","b"]
sort(var.list)                    # alphabetic sort
reverse(var.list)
concat(list1, list2)
setintersection(set1, set2)
setunion(set1, set2)
setsubtract(set1, set2)

# Map functions
keys(var.map)                     # list of keys
values(var.map)                   # list of values
lookup(var.map, "key", "default") # safe key access with default
merge(map1, map2)                 # merge maps (map2 wins on collision)
tomap({a = 1, b = 2})
zipmap(["a","b"], [1, 2])         # {a=1, b=2}

# Type conversion
tostring(42)                      # "42"
tonumber("42")                    # 42
tobool("true")                    # true
tolist(toset(["a","b"]))
toset(["a","a","b"])              # {"a","b"}

# Numeric
min(3, 1, 2)                      # 1
max(3, 1, 2)                      # 3
abs(-5)                           # 5
ceil(1.2)                         # 2
floor(1.8)                        # 1
pow(2, 8)                         # 256.0
log(8, 2)                         # 3.0

# Encoding
base64encode("hello")
base64decode("aGVsbG8=")
jsonencode({name = "priya"})      # → JSON string
jsondecode(file("policy.json"))   # parse JSON file → HCL object
yamlencode({name = "priya"})
yamldecode(file("values.yaml"))
urlencode("hello world")

# Filesystem (only during plan/apply — not runtime)
file("./scripts/init.sh")         # read file as string
filebase64("./certs/ca.crt")      # read file as base64
fileexists("./optional.conf")     # bool — file exists?
fileset(".", "*.tf")              # list matching files
templatefile("tpl.tftpl", {name = "priya"})  # render template

# Hash / crypto
md5("hello")
sha1("hello")
sha256("hello")
bcrypt("password")

# Date / time
timestamp()                       # "2024-01-15T10:30:00Z" (apply time)
formatdate("DD MMM YYYY", timestamp())

# IP / CIDR
cidrhost("10.0.0.0/16", 4)       # "10.0.0.4"
cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
cidrnetmask("10.0.0.0/24")       # "255.255.255.0"
```

---

## 4. Terraform — Project Structure

### Recommended layout

```
project/
├── main.tf           # primary resources
├── variables.tf      # input variable declarations
├── outputs.tf        # output value declarations
├── versions.tf       # terraform{} and required_providers blocks
├── providers.tf      # provider configuration
├── locals.tf         # local value definitions
├── data.tf           # data source lookups
│
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── eks/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
│
├── environments/
│   ├── dev.tfvars
│   ├── staging.tfvars
│   └── production.tfvars
│
└── terraform.tfstate   # local state (use remote in production)
```

### `versions.tf`

```hcl
terraform {
  required_version = ">= 1.5.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = ">= 2.20.0"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }

  # Remote state backend (S3 example)
  backend "s3" {
    bucket         = "my-tfstate-bucket"
    key            = "prod/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Version constraint operators

```hcl
version = "= 1.5.0"    # exact version only
version = ">= 1.5.0"   # minimum version
version = "~> 1.5"     # >= 1.5.0, < 2.0.0 (patch/minor updates only)
version = "~> 1.5.0"   # >= 1.5.0, < 1.6.0 (patch updates only)
version = ">= 1.4, < 2.0"  # range
```

---

## 5. Terraform — Providers & Resources

### Provider configuration

```hcl
# providers.tf
provider "aws" {
  region  = var.aws_region
  profile = var.aws_profile   # from ~/.aws/credentials

  default_tags {
    tags = {
      ManagedBy   = "terraform"
      Environment = var.environment
      Project     = var.project_name
    }
  }
}

# Multiple provider instances (alias)
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "ap_south"
  region = "ap-south-1"
}

# Use aliased provider in a resource
resource "aws_s3_bucket" "dr_bucket" {
  provider = aws.us_east
  bucket   = "my-dr-bucket"
}
```

### Resource syntax

```hcl
# resource "type" "name" { }
# Referenced as: type.name.attribute  e.g. aws_instance.web.id

resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.instance_type

  subnet_id              = aws_subnet.public.id     # reference another resource
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.web.name

  user_data = base64encode(templatefile("${path.module}/scripts/init.sh", {
    db_host = var.db_host
    app_env = var.environment
  }))

  root_block_device {
    volume_type = "gp3"
    volume_size = 20
    encrypted   = true
  }

  tags = merge(local.common_tags, {
    Name = "${local.prefix}-web"
    Role = "webserver"
  })

  lifecycle {
    create_before_destroy = true
    ignore_changes        = [user_data, ami]
    prevent_destroy       = true    # safety: refuse destroy
  }
}
```

### Resource references

```hcl
# Implicit dependency — Terraform builds the graph automatically
resource "aws_subnet" "public" {
  vpc_id = aws_vpc.main.id    # terraform knows this depends on aws_vpc.main
}

# Explicit dependency — when there's no visible reference
resource "aws_s3_object" "config" {
  depends_on = [aws_iam_role_policy.s3_access]
  bucket = aws_s3_bucket.configs.bucket
  key    = "app.conf"
  source = "app.conf"
}
```

---

## 6. Terraform — Variables & Outputs

### Variable declaration (`variables.tf`)

```hcl
# Simple string with default
variable "environment" {
  description = "Deployment environment (dev/staging/production)"
  type        = string
  default     = "dev"
}

# Constrained string (enum-like)
variable "instance_type" {
  type    = string
  default = "t3.micro"
  validation {
    condition     = contains(["t3.micro", "t3.small", "t3.medium", "t3.large"], var.instance_type)
    error_message = "instance_type must be t3.micro, t3.small, t3.medium, or t3.large."
  }
}

# Number with validation
variable "replica_count" {
  type    = number
  default = 2
  validation {
    condition     = var.replica_count >= 1 && var.replica_count <= 10
    error_message = "replica_count must be between 1 and 10."
  }
}

# Bool
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List of strings
variable "availability_zones" {
  type    = list(string)
  default = ["ap-south-1a", "ap-south-1b"]
}

# Map of strings
variable "tags" {
  type    = map(string)
  default = {}
}

# Object — structured config
variable "database" {
  type = object({
    engine         = string
    engine_version = string
    instance_class = string
    storage_gb     = number
    multi_az       = bool
  })
  default = {
    engine         = "postgres"
    engine_version = "15.4"
    instance_class = "db.t3.medium"
    storage_gb     = 100
    multi_az       = false
  }
}

# Sensitive — masked in plan output and logs
variable "db_password" {
  type      = string
  sensitive = true
}

# No default — must be provided
variable "project_name" {
  type        = string
  description = "Name of the project. Used as a prefix for all resources."
}
```

### Providing variable values

```bash
# 1. terraform.tfvars (auto-loaded)
# 2. *.auto.tfvars (auto-loaded, alphabetical order)
# 3. -var-file flag
# 4. -var flag
# 5. TF_VAR_name environment variable

terraform plan -var-file="production.tfvars"
terraform apply -var="replica_count=5" -var="environment=production"
export TF_VAR_db_password="supersecret"
```

```hcl
# production.tfvars
environment      = "production"
instance_type    = "t3.large"
replica_count    = 3
enable_monitoring = true

database = {
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = "db.t3.large"
  storage_gb     = 500
  multi_az       = true
}
```

### Output values (`outputs.tf`)

```hcl
output "instance_id" {
  description = "ID of the web server instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the web server"
  value       = aws_instance.web.public_ip
}

output "load_balancer_dns" {
  description = "DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}

# Sensitive output — masked in terminal, still in state
output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.username}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true
}

# Complex output
output "subnet_ids" {
  value = {
    public  = aws_subnet.public[*].id
    private = aws_subnet.private[*].id
  }
}
```

```bash
# Use outputs in CLI
terraform output instance_id
terraform output -json                     # all outputs as JSON
terraform output -raw load_balancer_dns   # no quotes (for scripting)

# Use in another root module via remote state
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "my-tfstate-bucket"
    key    = "network/terraform.tfstate"
    region = "ap-south-1"
  }
}

resource "aws_instance" "app" {
  subnet_id = data.terraform_remote_state.network.outputs.subnet_ids.private[0]
}
```

---

## 7. Terraform — Data Sources & Locals

### Data sources

```hcl
# data "type" "name" { }
# Referenced as: data.type.name.attribute

# Fetch latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]   # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Fetch current AWS account info
data "aws_caller_identity" "current" {}

# Fetch current region
data "aws_region" "current" {}

# Fetch existing VPC by tag
data "aws_vpc" "main" {
  tags = { Name = "main-vpc" }
}

# Fetch all availability zones in region
data "aws_availability_zones" "available" {
  state = "available"
}

# Fetch existing secret from Secrets Manager
data "aws_secretsmanager_secret_version" "db_pass" {
  secret_id = "prod/myapp/db-password"
}

# Fetch Kubernetes cluster info
data "aws_eks_cluster" "main" {
  name = var.cluster_name
}

data "aws_eks_cluster_auth" "main" {
  name = var.cluster_name
}
```

### Local values (`locals.tf`)

```hcl
locals {
  # Computed once, reused many times
  prefix       = "${var.project_name}-${var.environment}"
  account_id   = data.aws_caller_identity.current.account_id
  region       = data.aws_region.current.name

  # Common tags merged everywhere
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    AccountId   = local.account_id
  }

  # Conditional logic
  is_prod      = var.environment == "production"
  instance_type = local.is_prod ? "t3.large" : "t3.micro"
  min_replicas  = local.is_prod ? 3 : 1

  # Derived collections
  az_count = length(data.aws_availability_zones.available.names)

  # CIDR subnets per AZ
  public_subnets  = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 8, i)]
  private_subnets = [for i in range(local.az_count) : cidrsubnet(var.vpc_cidr, 8, i + 10)]

  # Parsed secret
  db_credentials = jsondecode(data.aws_secretsmanager_secret_version.db_pass.secret_string)
}
```

---

## 8. Terraform — State Management

### Remote backends

```hcl
# S3 backend (most common for AWS)
terraform {
  backend "s3" {
    bucket         = "mycompany-tfstate"
    key            = "services/myapp/production/terraform.tfstate"
    region         = "ap-south-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    profile        = "terraform-deployer"
  }
}

# GCS backend (Google Cloud)
terraform {
  backend "gcs" {
    bucket = "mycompany-tfstate"
    prefix = "terraform/state/production"
  }
}

# Terraform Cloud / HCP Terraform
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "my-app-production"
    }
  }
}
```

### State CLI commands

```bash
# List all resources in state
terraform state list

# Show a specific resource
terraform state show aws_instance.web

# Move a resource (rename without destroy/recreate)
terraform state mv aws_instance.web aws_instance.web_server

# Move into a module
terraform state mv aws_instance.web module.compute.aws_instance.web

# Remove from state (stops tracking, does NOT destroy)
terraform state rm aws_instance.orphaned

# Pull state to stdout
terraform state pull | jq '.resources[] | select(.type == "aws_instance")'

# Push local state to remote (dangerous — use with care)
terraform state push terraform.tfstate

# Import existing resource into state
terraform import aws_instance.web i-0123456789abcdef0

# Force-unlock a stuck state lock
terraform force-unlock LOCK_ID
```

### Partial configuration (backend secrets in CI)

```bash
# Don't hardcode secrets in backend config
# Use -backend-config flags or a separate file

terraform init \
  -backend-config="bucket=my-tfstate" \
  -backend-config="key=prod/terraform.tfstate" \
  -backend-config="region=ap-south-1"

# Or a backend.hcl file (not committed to git)
terraform init -backend-config=backend.hcl
```

---

## 9. Terraform — Modules

### Calling a module

```hcl
# Local module
module "vpc" {
  source = "./modules/vpc"

  # Input variables for the module
  project_name       = var.project_name
  environment        = var.environment
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = data.aws_availability_zones.available.names
}

# Registry module
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "${local.prefix}-eks"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnet_ids
}

# Git source
module "my_module" {
  source = "git::https://github.com/org/terraform-modules.git//modules/vpc?ref=v2.1.0"
}
```

### Module outputs

```hcl
# Reference module outputs
resource "aws_instance" "app" {
  subnet_id = module.vpc.private_subnet_ids[0]
  vpc_security_group_ids = [module.security.app_sg_id]
}

# Pass module output to another module
module "rds" {
  source    = "./modules/rds"
  vpc_id    = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
}
```

### Writing a module (`modules/vpc/`)

```hcl
# modules/vpc/variables.tf
variable "vpc_cidr"    { type = string }
variable "environment" { type = string }
variable "project_name" { type = string }
variable "availability_zones" { type = list(string) }

# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "${var.project_name}-${var.environment}-vpc" }
}

resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  tags = { Name = "${var.project_name}-public-${count.index + 1}" }
}

# modules/vpc/outputs.tf
output "vpc_id"             { value = aws_vpc.main.id }
output "public_subnet_ids"  { value = aws_subnet.public[*].id }
```

---

## 10. Terraform — Meta-Arguments

### `count`

```hcl
# Create N identical resources
resource "aws_instance" "worker" {
  count         = var.worker_count
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
  tags = { Name = "worker-${count.index + 1}" }
}

# Conditional resource (0 or 1)
resource "aws_cloudwatch_metric_alarm" "cpu" {
  count = var.enable_monitoring ? 1 : 0
  # ...
}

# Reference: aws_instance.worker[0].id
# All IDs: aws_instance.worker[*].id
```

### `for_each`

```hcl
# Preferred over count for non-identical resources
# Avoids index shifting when items are added/removed

resource "aws_s3_bucket" "env_buckets" {
  for_each = toset(["dev", "staging", "production"])
  bucket   = "${var.project_name}-${each.key}-assets"
  tags     = { Environment = each.key }
}

# for_each with a map
variable "services" {
  default = {
    web = { port = 80,  replicas = 3 }
    api = { port = 8080, replicas = 2 }
  }
}

resource "aws_lb_target_group" "services" {
  for_each = var.services
  name     = "${local.prefix}-${each.key}"
  port     = each.value.port
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
}

# Reference: aws_s3_bucket.env_buckets["production"].arn
# All IDs: values(aws_s3_bucket.env_buckets)[*].id
```

### `lifecycle`

```hcl
resource "aws_instance" "web" {
  # ...
  lifecycle {
    # Recreate new before destroying old (zero-downtime replacement)
    create_before_destroy = true

    # Ignore drift on these attributes (e.g. managed outside Terraform)
    ignore_changes = [
      ami,
      user_data,
      tags["LastUpdated"]
    ]

    # Refuse to destroy (safeguard for critical resources)
    prevent_destroy = true

    # Custom precondition — checked before apply
    precondition {
      condition     = var.environment != "production" || var.replica_count >= 3
      error_message = "Production requires at least 3 replicas."
    }

    # Custom postcondition — checked after apply
    postcondition {
      condition     = self.arn != ""
      error_message = "Resource must have a valid ARN after creation."
    }
  }
}
```

### `dynamic` blocks

```hcl
# Generate repeated nested blocks programmatically
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

---

## 11. Terraform — Workspace & Environments

### Workspaces

```bash
# Workspaces = isolated state files in the same backend
terraform workspace list
terraform workspace new staging
terraform workspace select production
terraform workspace show      # current workspace
terraform workspace delete staging
```

```hcl
# Use workspace name in config
locals {
  env_config = {
    default    = { instance_type = "t3.micro",  replica_count = 1 }
    staging    = { instance_type = "t3.small",  replica_count = 2 }
    production = { instance_type = "t3.large",  replica_count = 5 }
  }
  config = local.env_config[terraform.workspace]
}

resource "aws_instance" "app" {
  instance_type = local.config.instance_type
  count         = local.config.replica_count
}
```

### Environment separation strategies

```
Strategy 1: Workspaces (same root, different state)
  ✅ Simple, DRY
  ❌ One mistake can affect all envs

Strategy 2: Directories (separate root per env)
  environments/dev/        → terraform apply
  environments/staging/    → terraform apply
  environments/production/ → terraform apply
  ✅ Full isolation, separate state backends
  ❌ Code duplication (use modules to reduce)

Strategy 3: Terragrunt (wrapper tool)
  ✅ DRY across environments, hierarchical vars
  ✅ Remote state management, dependency graph
```

---

## 12. Terraform — Common AWS Patterns

### VPC with public/private subnets

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = merge(local.common_tags, { Name = "${local.prefix}-vpc" })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge(local.common_tags, { Name = "${local.prefix}-igw" })
}

resource "aws_subnet" "public" {
  count                   = length(local.public_subnets)
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.public_subnets[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  tags = merge(local.common_tags, {
    Name = "${local.prefix}-public-${count.index + 1}"
    "kubernetes.io/role/elb" = "1"   # required for EKS ALB controller
  })
}

resource "aws_subnet" "private" {
  count             = length(local.private_subnets)
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnets[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  tags = merge(local.common_tags, {
    Name = "${local.prefix}-private-${count.index + 1}"
    "kubernetes.io/role/internal-elb" = "1"
  })
}

resource "aws_eip" "nat" {
  count  = length(local.public_subnets)
  domain = "vpc"
  tags   = merge(local.common_tags, { Name = "${local.prefix}-nat-eip-${count.index + 1}" })
}

resource "aws_nat_gateway" "main" {
  count         = length(local.public_subnets)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.main]
  tags = merge(local.common_tags, { Name = "${local.prefix}-nat-${count.index + 1}" })
}
```

### EKS cluster

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name                   = "${local.prefix}-eks"
  cluster_version                = "1.29"
  cluster_endpoint_public_access = true

  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id

  eks_managed_node_groups = {
    general = {
      min_size       = 2
      max_size       = 10
      desired_size   = 3
      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"
      labels = { role = "general" }
    }
    spot = {
      min_size       = 0
      max_size       = 10
      desired_size   = 2
      instance_types = ["t3.medium", "t3.large"]
      capacity_type  = "SPOT"
      labels = { role = "spot" }
      taints = [{
        key    = "spot"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }
  }

  tags = local.common_tags
}
```

### RDS PostgreSQL

```hcl
resource "aws_db_subnet_group" "main" {
  name       = "${local.prefix}-db-subnet"
  subnet_ids = aws_subnet.private[*].id
  tags       = local.common_tags
}

resource "aws_db_instance" "postgres" {
  identifier        = "${local.prefix}-postgres"
  engine            = "postgres"
  engine_version    = "15.4"
  instance_class    = var.db_instance_class
  allocated_storage = 100
  storage_type      = "gp3"
  storage_encrypted = true

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az               = local.is_prod
  backup_retention_period = local.is_prod ? 7 : 1
  deletion_protection    = local.is_prod

  skip_final_snapshot = !local.is_prod
  final_snapshot_identifier = local.is_prod ? "${local.prefix}-final-snapshot" : null

  tags = merge(local.common_tags, { Name = "${local.prefix}-postgres" })
}
```

---

## 13. Vault HCL — Policies & Config

### Vault policy syntax

```hcl
# vault write sys/policy/myapp-policy policy=@myapp-policy.hcl

# Allow read of specific secrets
path "secret/data/myapp/*" {
  capabilities = ["read", "list"]
}

# Allow create and update of a specific path
path "secret/data/myapp/config" {
  capabilities = ["create", "read", "update"]
}

# Full access to a team's secrets
path "secret/data/team-platform/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Deny access (overrides any allow)
path "secret/data/team-platform/prod-keys" {
  capabilities = ["deny"]
}

# Allow token self-renewal
path "auth/token/renew-self" {
  capabilities = ["update"]
}

# Allow looking up own token
path "auth/token/lookup-self" {
  capabilities = ["read"]
}

# Allow PKI cert issuance
path "pki/issue/my-role" {
  capabilities = ["create", "update"]
}
```

### Capabilities reference

```
create  → POST/PUT to new paths
read    → GET
update  → POST/PUT to existing paths
delete  → DELETE
list    → LIST (show keys, not values)
sudo    → root-protected paths
deny    → explicitly block (overrides all)
```

### Vault provider in Terraform

```hcl
provider "vault" {
  address = "https://vault.example.com:8200"
  # Auth via VAULT_TOKEN env var, or:
  auth_login {
    path = "auth/aws/login"
    parameters = {
      role = "my-terraform-role"
    }
  }
}

# Read a secret
data "vault_generic_secret" "db" {
  path = "secret/data/myapp/database"
}

# Use the secret in a resource
resource "aws_db_instance" "main" {
  password = data.vault_generic_secret.db.data["password"]
}

# Write a secret to Vault
resource "vault_generic_secret" "app_config" {
  path = "secret/data/myapp/config"
  data_json = jsonencode({
    api_key  = var.api_key
    app_env  = var.environment
  })
}
```

---

## 14. Packer HCL — Image Builds

### Basic Packer template (`.pkr.hcl`)

```hcl
# versions.pkr.hcl
packer {
  required_version = ">= 1.9.0"
  required_plugins {
    amazon = {
      source  = "github.com/hashicorp/amazon"
      version = "~> 1.0"
    }
    ansible = {
      source  = "github.com/hashicorp/ansible"
      version = "~> 1.1"
    }
  }
}
```

```hcl
# variables.pkr.hcl
variable "aws_region"    { default = "ap-south-1" }
variable "instance_type" { default = "t3.micro" }
variable "app_version"   { default = "latest" }

variable "base_ami_filters" {
  default = {
    name                = "ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"
    virtualization-type = "hvm"
  }
}
```

```hcl
# main.pkr.hcl
source "amazon-ebs" "ubuntu" {
  region        = var.aws_region
  instance_type = var.instance_type

  source_ami_filter {
    filters = var.base_ami_filters
    owners        = ["099720109477"]   # Canonical
    most_recent   = true
  }

  ssh_username = "ubuntu"

  ami_name        = "myapp-${var.app_version}-{{timestamp}}"
  ami_description = "My App AMI built by Packer"

  ami_regions = ["ap-south-1", "us-east-1"]   # copy to multiple regions

  tags = {
    Name        = "myapp-${var.app_version}"
    BaseAMI     = "{{ .SourceAMIName }}"
    BuildTime   = "{{timestamp}}"
    AppVersion  = var.app_version
    ManagedBy   = "packer"
  }

  launch_block_device_mappings {
    device_name           = "/dev/sda1"
    volume_size           = 20
    volume_type           = "gp3"
    delete_on_termination = true
    encrypted             = true
  }
}

build {
  name    = "myapp-image"
  sources = ["source.amazon-ebs.ubuntu"]

  # Shell provisioner — inline commands
  provisioner "shell" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get upgrade -y",
      "sudo apt-get install -y curl git jq unzip"
    ]
  }

  # Shell provisioner — external script
  provisioner "shell" {
    script          = "scripts/install-docker.sh"
    execute_command = "sudo bash '{{.Path}}'"
  }

  # File upload
  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/tmp/app.conf"
  }

  # Ansible provisioner
  provisioner "ansible" {
    playbook_file = "ansible/site.yml"
    extra_arguments = [
      "--extra-vars", "app_version=${var.app_version}",
      "--tags", "install,configure"
    ]
  }

  # Shell for final cleanup / validation
  provisioner "shell" {
    inline = [
      "sudo cloud-init clean",
      "sudo rm -rf /tmp/* /var/tmp/*",
      "sudo apt-get clean"
    ]
  }

  post-processor "manifest" {
    output     = "manifest.json"
    strip_path = true
  }
}
```

```bash
# Packer CLI
packer init .          # install plugins
packer validate .      # validate template
packer fmt .           # format HCL files
packer build .         # build the image
packer build -var="app_version=v1.2.3" .
packer build -only="amazon-ebs.ubuntu" .   # specific source
```

---

## 15. Common Gotchas & Best Practices

### Plan before apply — always

```bash
# Save plan to file — apply exactly what was reviewed
terraform plan -out=tfplan
terraform apply tfplan

# In CI: generate and review plan, then apply in a separate step
terraform plan -out=tfplan -var-file=production.tfvars
terraform apply -auto-approve tfplan
```

### `count` vs `for_each` — the index trap

```hcl
# PROBLEM with count: removing an item shifts all indices
# If you remove "banana" from position [1], all items after it get new indices
# → Terraform destroys and recreates them!
resource "aws_s3_bucket" "bad" {
  count  = length(var.names)
  bucket = var.names[count.index]
}

# SOLUTION: for_each uses stable keys, not positions
resource "aws_s3_bucket" "good" {
  for_each = toset(var.names)
  bucket   = each.key
}
```

### Sensitive values in state

```hcl
# State file ALWAYS contains all values in plaintext (including sensitive = true)
# Always encrypt your backend storage
# In S3: enable SSE-KMS + bucket versioning + access logging

variable "db_password" {
  type      = string
  sensitive = true    # masked in plan/apply output only — still in state!
}
```

### Avoid hardcoding AMIs and region-specific IDs

```hcl
# WRONG — hardcoded AMI is region-specific and goes stale
resource "aws_instance" "web" {
  ami = "ami-0c55b159cbfafe1f0"   # ← will break in other regions
}

# CORRECT — always use a data source
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
resource "aws_instance" "web" {
  ami = data.aws_ami.ubuntu.id
}
```

### Terraform fmt and validate in CI

```bash
# Format check — non-zero exit if any file needs formatting
terraform fmt -check -recursive

# Validate — checks syntax and internal consistency
terraform validate

# Both in one CI step
terraform fmt -check -recursive && terraform validate
```

### Lock file — always commit

```bash
# .terraform.lock.hcl pins provider versions
# ALWAYS commit this file — ensures reproducible builds
git add .terraform.lock.hcl

# Upgrade providers explicitly
terraform init -upgrade
```

### Resource tagging strategy

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    Team        = var.team
    ManagedBy   = "terraform"
    Repository  = "github.com/org/infra"
    CostCenter  = var.cost_center
  }
}

# Merge common tags with resource-specific tags
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.prefix}-web"
    Role = "webserver"
  })
}
```

### Moved blocks — rename without destroying

```hcl
# When you rename a resource or move it into a module
# Use moved{} instead of deleting and recreating

moved {
  from = aws_instance.web
  to   = module.compute.aws_instance.web
}

moved {
  from = aws_s3_bucket.old_name
  to   = aws_s3_bucket.new_name
}
```

---

## 16. Quick Reference Cheat Sheet

```hcl
# ─── BLOCK TYPES ────────────────────────────────────────────────────────────
terraform  { }                           # settings, backend, required_providers
provider   "aws"          { }            # provider config
resource   "aws_instance" "web" { }      # managed resource
data       "aws_ami"      "ubuntu" { }   # read-only data source
variable   "name"         { }            # input variable
output     "name"         { }            # output value
locals     { }                           # local computed values
module     "vpc"          { }            # module call
moved      { from = ...; to = ... }      # rename without destroy

# ─── TYPES ──────────────────────────────────────────────────────────────────
string   number   bool   null
list(string)   set(string)   map(string)
object({ name = string, port = number })
tuple([string, number, bool])
any

# ─── EXPRESSIONS ────────────────────────────────────────────────────────────
"${var.name}-suffix"                     # string interpolation
var.env == "prod" ? "t3.large" : "t3.micro"    # conditional
[for item in var.list : upper(item)]     # list for
{for k, v in var.map : k => upper(v)}   # map for
[for item in var.list : item if item != "skip"]  # filtered for
aws_instance.web[*].id                  # splat

# ─── META-ARGUMENTS ─────────────────────────────────────────────────────────
count     = 3                            # N identical resources
for_each  = toset(["a","b","c"])         # one per key (preferred)
depends_on = [resource.other]            # explicit dependency
provider  = aws.alias                   # use aliased provider
lifecycle { create_before_destroy = true; prevent_destroy = true }

# ─── REFERENCES ─────────────────────────────────────────────────────────────
var.name                                 # input variable
local.prefix                             # local value
data.aws_ami.ubuntu.id                   # data source attribute
aws_instance.web.id                      # resource attribute
module.vpc.vpc_id                        # module output
count.index                              # current count index
each.key / each.value                    # current for_each item
self.id                                  # self-reference in lifecycle
path.module / path.root                  # filesystem paths
terraform.workspace                      # current workspace name

# ─── MOST-USED FUNCTIONS ────────────────────────────────────────────────────
merge(map1, map2)           toset(list)          length(coll)
flatten([list1, list2])     compact(list)        distinct(list)
lookup(map, key, default)   keys(map)            values(map)
format("%-10s", str)        join(", ", list)     split(",", str)
replace(str, old, new)      upper/lower(str)     trimspace(str)
cidrsubnet(cidr, 8, 1)      cidrhost(cidr, 4)
jsonencode(obj)             jsondecode(str)
templatefile(path, vars)    file(path)           filebase64(path)
tostring(num)               tonumber(str)        tobool(str)
```

### Terraform CLI workflow

```bash
terraform init              # initialise, download providers, configure backend
terraform fmt -recursive    # format all .tf files
terraform validate          # syntax and config check
terraform plan              # preview changes (always do this first)
terraform plan -out=tfplan  # save plan to file
terraform apply             # apply (asks for confirmation)
terraform apply tfplan      # apply saved plan (no confirmation needed)
terraform apply -auto-approve  # apply without confirmation (CI only)
terraform destroy           # destroy all managed resources
terraform output            # show output values
terraform state list        # list resources in state
terraform state show <addr> # inspect a resource in state
terraform taint <addr>      # force recreation on next apply (deprecated → use -replace)
terraform apply -replace="aws_instance.web"  # force replace a resource
terraform import <addr> <id>  # import existing resource into state
```

### Key rules at a glance

| Rule                             | Detail                                                          |
|----------------------------------|-----------------------------------------------------------------|
| `for_each` over `count`          | Stable keys, no index shifting when items are removed           |
| Always `plan -out`               | Save plan to file, apply exact same plan                        |
| Commit `.terraform.lock.hcl`     | Pins provider versions for reproducibility                      |
| Encrypt remote state             | State contains all values including secrets                     |
| `sensitive = true` ≠ encrypted   | Only masks terminal output — state file still plain text        |
| Use `data` for AMIs              | Never hardcode AMI IDs — they're region-specific and go stale   |
| Tag everything                   | Use `merge(local.common_tags, {...})` on every resource         |
| `moved {}` for renames           | Avoids destroy/recreate when renaming resources or moving to modules |
| `lifecycle.prevent_destroy`      | Guard critical resources (RDS, S3, stateful infra)              |
| One root module per environment  | Full state isolation between dev / staging / production         |

---

*Part of DevOpsNotes / LANGUAGES — see also `01_JSON.md`, `02_Groovy_Jenkins.md`, `04_Jinja2.md`*