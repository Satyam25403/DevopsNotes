# Amazon ECR & ECS (Container Registry & Orchestration)

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Docker Image** | A packaged application with all its dependencies, ready to run anywhere |
| **Repository** | A collection of related Docker images, versioned by tags (e.g., `v1`, `latest`) |
| **Registry** | A centralized service that hosts multiple repositories |
| **Container** | A running instance of a Docker image |

---

## Amazon ECR (Elastic Container Registry)

ECR is your private Docker Hub inside AWS — a fully managed container image registry.

**Features:**
- Secure image storage with IAM-based access control
- Vulnerability scanning on push
- Lifecycle policies to automatically clean up old/untagged images
- Cross-region replication for global availability
- Supports Docker and OCI-compatible images

---

## Pushing Images to ECR

### 1. Prerequisites
```bash
aws --version    # AWS CLI must be configured
docker --version # Docker must be running locally
```

### 2. Create a Repository
Create an empty repository in the AWS Console (ECR → Repositories → Create) or via CLI:
```bash
aws ecr create-repository --repository-name my-app --region us-east-1
```

### 3. Authenticate Docker to ECR
```bash
aws ecr get-login-password --region <region> | \
  docker login --username AWS --password-stdin \
  <aws-account-id>.dkr.ecr.<region>.amazonaws.com
```

### 4. Tag Your Local Image
```bash
docker tag <local-image-name>:<tag> \
  <aws-account-id>.dkr.ecr.<region>.amazonaws.com/<ecr-repo-name>:<tag>
```

### 5. Push to ECR
```bash
docker push <aws-account-id>.dkr.ecr.<region>.amazonaws.com/<ecr-repo-name>:<tag>
```

---

## Pulling Images from ECR

```bash
# Authenticate (same as push)
aws ecr get-login-password --region <region> | \
  docker login --username AWS --password-stdin \
  <aws-account-id>.dkr.ecr.<region>.amazonaws.com

# Pull image
docker pull <aws-account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>

# List local images
docker images
```

---

## ECR Lifecycle Policies

Automatically clean up old images to control costs:
```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep last 10 images",
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": { "type": "expire" }
  }]
}
```

---

## Amazon ECS (Elastic Container Service)

ECS is AWS's container orchestration platform — it runs, scales, and manages containers across EC2 instances or serverless Fargate.

### ECS vs Kubernetes
| | ECS | Kubernetes (EKS) |
|--|-----|------------------|
| Complexity | Simpler, AWS-native | More powerful, more complex |
| Control | Less — AWS manages more | More — you control the cluster |
| Portability | AWS-specific | Multi-cloud |
| Best for | Teams wanting simplicity | Teams needing full K8s features |

---

## Core ECS Components

### Cluster
The top-level grouping of compute resources (EC2 instances or Fargate capacity). Everything in ECS lives inside a cluster.

### Task Definition
A blueprint that describes how your container(s) should run. Analogous to a Kubernetes Deployment spec or a `docker-compose.yml`.

Defines:
- Container image(s) to use (from ECR or public registries)
- CPU and memory allocation
- Port mappings
- Environment variables
- IAM roles for permissions
- Logging configuration (CloudWatch)
- Volume mounts

### Task
A running instance of a Task Definition. One task = one or more containers running together (like a Kubernetes Pod).

### Service
Manages the lifecycle of tasks. Keeps a specified number of tasks running, handles rolling updates, and integrates with load balancers. Analogous to a Kubernetes Deployment + Service combined.

---

## Launch Types

| Type | Description |
|------|-------------|
| **Fargate** | Serverless — AWS manages the underlying infrastructure. You only define CPU/memory. |
| **EC2** | You manage EC2 instances that form the cluster. More control, more responsibility. |

> **Recommendation**: Use Fargate for simplicity unless you have specific requirements (GPU, custom AMIs, cost optimization at scale).

---

## ECS Workflow

```
Build Docker Image → Push to ECR → Create Task Definition → Create Service → Deploy to Cluster
```

---

## Creating a Task Definition

1. Go to **ECS → Task Definitions → Create new**
2. Choose launch type: **Fargate** or **EC2**
3. Add container(s):
   - Name
   - ECR image URI
   - Port mappings
   - Environment variables
   - Log configuration (CloudWatch)
4. Set CPU and memory
5. Attach IAM execution role with **`AmazonECSTaskExecutionRolePolicy`**
   - This allows ECS to pull images from ECR and write logs to CloudWatch

> **Note**: Use `AmazonECSTaskExecutionRolePolicy` for Fargate (not `AWSEC2ContainerRegistryFullAccess` — that's for EC2 instances directly).

---

## Creating a Service

1. Go to **ECS → Clusters → [your cluster] → Services → Create**
2. Select:
   - Launch type (Fargate or EC2)
   - Task definition
   - Desired task count
3. Configure networking (VPC, subnets, security groups)
4. (Optional) Attach an **Application Load Balancer** for traffic routing
5. (Optional) Enable **Auto Scaling** based on CPU/memory metrics

---

## Useful CLI Commands

```bash
# List clusters
aws ecs list-clusters

# List services in a cluster
aws ecs list-services --cluster my-cluster

# Describe running tasks
aws ecs list-tasks --cluster my-cluster --service-name my-service
aws ecs describe-tasks --cluster my-cluster --tasks <task-id>

# Force a new deployment (redeploy with latest image)
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment

# View service events (for debugging)
aws ecs describe-services \
  --cluster my-cluster \
  --services my-service \
  --query "services[0].events[:5]"
```

---

## Logging

ECS integrates with CloudWatch Logs. In your Task Definition, configure:
```json
"logConfiguration": {
  "logDriver": "awslogs",
  "options": {
    "awslogs-group": "/ecs/my-app",
    "awslogs-region": "us-east-1",
    "awslogs-stream-prefix": "ecs"
  }
}
```

Logs appear in CloudWatch under the specified log group.

---

## IAM Roles for ECS

| Role | Purpose |
|------|---------|
| **Task Execution Role** | Allows ECS agent to pull images from ECR and write logs. Attach `AmazonECSTaskExecutionRolePolicy`. |
| **Task Role** | Permissions your *application* needs (e.g., access S3, DynamoDB). Attach service-specific policies. |

---

## ECS Best Practices

- **Use Fargate** for new projects to avoid managing EC2 capacity.
- **Pin image tags** in Task Definitions (avoid `latest` in production — use specific version tags).
- **Use ECR lifecycle policies** to keep registry costs down.
- **Enable Container Insights** in CloudWatch for enhanced monitoring.
- **Use ALB + ECS Service** for production traffic — it handles health checks and rolling updates.
- **Store secrets in SSM Parameter Store or Secrets Manager** and inject via Task Definition environment — never hardcode credentials.
- **Use Task Roles** (not execution roles) for application-level AWS access.