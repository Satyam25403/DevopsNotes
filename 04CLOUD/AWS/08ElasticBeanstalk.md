# AWS Elastic Beanstalk

Elastic Beanstalk is a fully managed **Platform-as-a-Service (PaaS)** that abstracts away infrastructure management. You provide your code; AWS provisions and manages everything needed to run it.

---

## The Problem It Solves

Running a production app requires:
- A server (OS, CPU, memory)
- A runtime (Python, Node.js, Java…)
- A web server (Nginx, Apache…)
- Networking (VPC, security groups, load balancer)
- Scaling, monitoring, and deployment pipelines

Elastic Beanstalk handles **all of this automatically**.

> If EC2 is renting a raw server and configuring everything yourself, Elastic Beanstalk is handing AWS your app and saying: *"Just make it run."*

---

## What Beanstalk Provisions Automatically

| Layer | What AWS Creates |
|-------|-----------------|
| **Compute** | EC2 instances with Amazon Linux or Windows |
| **Runtime** | Node.js, Python, Java, .NET, PHP, Ruby, Go, or Docker |
| **Web Server** | Nginx, Apache, IIS, or Passenger |
| **Networking** | VPC, subnets, security groups |
| **Load Balancing** | Application Load Balancer |
| **DNS** | Route 53 alias record |
| **Scaling** | Auto Scaling Group |
| **Monitoring** | CloudWatch metrics, health checks, logs |

---

## Supported Platforms

- Node.js
- Python
- Java (Tomcat)
- .NET (Windows / Linux)
- PHP
- Ruby
- Go
- Docker (single container or multi-container)
- Custom platforms

---

## How to Deploy

### Option 1: Via the AWS Console

1. Go to **Elastic Beanstalk → Create Application**
2. Choose your platform (e.g., Node.js)
3. Upload your code as a **ZIP file** containing:
   - Your source code
   - Dependencies file (`package.json`, `requirements.txt`, etc.)
   - **Do NOT include `node_modules`** or other build artifacts
4. Click **Create environment** and wait for provisioning
5. Access your app at the provided Beanstalk URL (e.g., `myapp.us-east-1.elasticbeanstalk.com`)

### Option 2: Via EB CLI

```bash
# Install EB CLI
pip install awsebcli

# Initialize in your project directory
eb init

# Create an environment and deploy
eb create my-env

# Deploy updates
eb deploy

# Open the deployed app in browser
eb open

# Check environment status
eb status

# View logs
eb logs
```

### Deploying Updates
To update your app, simply zip your code and upload it in the Console, or run `eb deploy` via CLI.

---

## Application vs Environment

- **Application**: A logical container grouping your app's versions and environments.
- **Environment**: A running deployment of a specific app version (e.g., `my-app-prod`, `my-app-staging`).

You can run multiple environments per application — handy for staging/production separation.

---

## Configuration

You can customize Beanstalk behavior using `.ebextensions/` — config files in YAML/JSON placed in your project root:

```yaml
# .ebextensions/nodecommand.config
option_settings:
  aws:elasticbeanstalk:container:nodejs:
    NodeCommand: "npm start"
```

Common configuration options:
- Instance type
- Environment variables
- Auto Scaling min/max
- Health check paths
- Load balancer settings

---

## Environment Variables

Set environment variables in **Configuration → Software → Environment properties** in the Console, or via `.ebextensions`:
```yaml
option_settings:
  aws:elasticbeanstalk:application:environment:
    NODE_ENV: production
    DB_URL: your-rds-endpoint
```

---

## Deployment Policies

| Policy | Description |
|--------|-------------|
| **All at once** | Fastest, but causes brief downtime |
| **Rolling** | Updates instances in batches — reduced capacity during deploy |
| **Rolling with additional batch** | Maintains full capacity during deploy |
| **Immutable** | Launches new instances with new version — safest, slowest |
| **Blue/Green** | Deploy to a separate environment, then swap URLs |

---

## Verifying What Beanstalk Created

After deployment, you can manually verify in the AWS Console:
- **EC2**: Your app instances
- **EC2 → Load Balancers**: The ALB Beanstalk created
- **EC2 → Auto Scaling Groups**: Scaling configuration
- **CloudWatch**: Logs and metrics
- **RDS** (if configured): Your managed database

---

## Beanstalk vs Other AWS Compute Options

| | Elastic Beanstalk | EC2 (raw) | ECS/Fargate | Lambda |
|--|------------------|-----------|-------------|--------|
| Control | Medium | Full | High | Minimal |
| Management overhead | Low | High | Medium | Minimal |
| Best for | Traditional web apps | Custom infra | Containerized apps | Event-driven functions |
| Scaling | Auto (managed) | Manual / ASG | Auto | Auto |

---

## Limitations & When Not to Use It

- **Not ideal for microservices** — ECS or EKS is better for containerized, multi-service architectures.
- **Less control** over the underlying infrastructure compared to raw EC2.
- **Costs can be opaque** — you pay for the underlying resources (EC2, ALB, etc.) but Beanstalk itself is free.
- For serverless or event-driven workloads, Lambda is more appropriate.