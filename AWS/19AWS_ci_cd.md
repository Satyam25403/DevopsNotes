# AWS CI/CD: CodeCommit, CodeBuild, CodeDeploy & CodePipeline

AWS provides a fully integrated CI/CD suite that takes code from commit to production without leaving the AWS ecosystem.

```
Code Commit → Code Build → Code Deploy
        ↑________________________________|
                 CodePipeline (orchestrates all three)
```

---

## Overview

| Service | Role | Analog |
|---------|------|--------|
| **CodeCommit** | Source control (Git hosting) | GitHub / GitLab |
| **CodeBuild** | Build & test automation | GitHub Actions / Jenkins |
| **CodeDeploy** | Deployment automation | Spinnaker / Octopus Deploy |
| **CodePipeline** | Pipeline orchestration | GitHub Actions workflow / Jenkins Pipeline |

---

## 1. AWS CodeCommit

A fully managed **Git repository** service — private, secure, and integrated with IAM.

> **Note**: AWS announced CodeCommit is no longer accepting new customers (July 2024). Existing users continue to have access. Consider **GitHub, GitLab, or Bitbucket** as alternatives — CodePipeline integrates with all of them.

### Connecting via SSH

```bash
# 1. Generate an SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/codecommit_rsa

# 2. Upload the public key in IAM
# IAM → Users → [your user] → Security credentials → SSH public keys for CodeCommit
# Note the SSH key ID shown after upload

# 3. Create ~/.ssh/config
cat >> ~/.ssh/config << 'EOF'
Host git-codecommit.*.amazonaws.com
  User <IAM-SSH-KEY-ID>
  IdentityFile ~/.ssh/codecommit_rsa
EOF

chmod 600 ~/.ssh/config
```

### Basic Workflow

```bash
# Clone a repo
git clone ssh://git-codecommit.<region>.amazonaws.com/v1/repos/<repo-name>

# Or add as remote to an existing repo
git init
git add .
git commit -m "Initial commit"
git remote add origin ssh://git-codecommit.<region>.amazonaws.com/v1/repos/<repo-name>
git push origin main

# Standard git workflow applies
git pull
git checkout -b feature/my-feature
git push origin feature/my-feature
```

### Create a Repository via CLI

```bash
aws codecommit create-repository \
  --repository-name my-app \
  --repository-description "My application source"

aws codecommit list-repositories
aws codecommit get-repository --repository-name my-app
```

---

## 2. AWS CodeBuild

A fully managed **continuous integration** service. It compiles code, runs tests, and produces build artifacts — no build servers to manage.

### How It Works

1. CodeBuild pulls your source code (from CodeCommit, GitHub, S3, etc.)
2. It spins up a temporary build container
3. It runs the commands in your `buildspec.yml`
4. It stores artifacts in S3
5. The container is destroyed

### buildspec.yml

The build specification file must be at the **root of your repository**:

```yaml
version: 0.2

env:
  variables:
    NODE_ENV: production
  parameter-store:
    DB_URL: /myapp/prod/db-url          # pull from SSM Parameter Store

phases:
  install:
    runtime-versions:
      nodejs: 18
    commands:
      - echo "Installing dependencies..."
      - npm ci                           # use ci for reproducible installs

  pre_build:
    commands:
      - echo "Running tests..."
      - npm test
      - echo "Logging in to ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - echo "Building application..."
      - npm run build
      - echo "Building Docker image..."
      - docker build -t $ECR_REGISTRY/$ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $ECR_REGISTRY/$ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_REGISTRY/$ECR_REPO:latest

  post_build:
    commands:
      - echo "Pushing Docker image to ECR..."
      - docker push $ECR_REGISTRY/$ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker push $ECR_REGISTRY/$ECR_REPO:latest
      - printf '[{"name":"my-container","imageUri":"%s"}]' $ECR_REGISTRY/$ECR_REPO:$CODEBUILD_RESOLVED_SOURCE_VERSION > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json             # passed to CodeDeploy/ECS
    - appspec.yml                       # CodeDeploy config
    - scripts/**/*                      # deployment scripts

cache:
  paths:
    - node_modules/**/*                 # cache dependencies between builds
```

### Create a Build Project via CLI

```bash
aws codebuild create-project \
  --name my-build-project \
  --source '{"type": "CODECOMMIT", "location": "https://git-codecommit.us-east-1.amazonaws.com/v1/repos/my-app"}' \
  --artifacts '{"type": "S3", "location": "my-artifacts-bucket", "packaging": "ZIP"}' \
  --environment '{
    "type": "LINUX_CONTAINER",
    "image": "aws/codebuild/standard:7.0",
    "computeType": "BUILD_GENERAL1_SMALL",
    "privilegedMode": true
  }' \
  --service-role arn:aws:iam::123456789012:role/CodeBuildServiceRole

# Trigger a manual build
aws codebuild start-build --project-name my-build-project

# Check build status
aws codebuild list-builds-for-project --project-name my-build-project
aws codebuild batch-get-builds --ids <build-id>
```

### IAM Role for CodeBuild

The CodeBuild service role needs permissions for:
- `codecommit:GitPull` — pull source code
- `s3:GetObject`, `s3:PutObject` — read/write artifacts
- `ecr:*` — push Docker images
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` — write build logs to CloudWatch
- `ssm:GetParameters` — read Parameter Store values

---

## 3. AWS CodeDeploy

A deployment service that automates application deployments to **EC2**, **Lambda**, or **on-premises servers**.

### Deployment Types

| Type | Description |
|------|-------------|
| **In-place** | Stop app → deploy new version → restart (brief downtime) |
| **Blue/Green** | Deploy to new instances → shift traffic → terminate old |

### Setup for EC2 Deployment

#### Step 1: Prepare the EC2 Instance

```bash
# Attach IAM role with AmazonEC2RoleforAWSCodeDeploy to the instance

# Install CodeDeploy agent (Amazon Linux 2)
sudo yum update -y
sudo yum install -y ruby wget
cd /home/ec2-user
wget https://aws-codedeploy-us-east-1.s3.us-east-1.amazonaws.com/latest/install
chmod +x ./install
sudo ./install auto

# Check agent status
sudo service codedeploy-agent status
sudo systemctl enable codedeploy-agent  # start on boot
```

#### Step 2: Add Deployment Files to Your Repo

**`appspec.yml`** (tells CodeDeploy what to deploy and where):
```yaml
version: 0.0
os: linux
files:
  - source: build/
    destination: /var/www/html/
  - source: config/nginx.conf
    destination: /etc/nginx/sites-available/
hooks:
  BeforeInstall:
    - location: scripts/before_install.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/after_install.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
      timeout: 300
      runas: root
  ValidateService:
    - location: scripts/validate.sh
      timeout: 300
      runas: root
```

**`scripts/start_server.sh`**:
```bash
#!/bin/bash
systemctl restart nginx
systemctl restart myapp
```

**`scripts/validate.sh`**:
```bash
#!/bin/bash
curl -f http://localhost/health || exit 1
```

#### Step 3: Create CodeDeploy Application

```bash
# Create application
aws deploy create-application \
  --application-name my-app \
  --compute-platform Server

# Create deployment group (targets EC2 instances by tag)
aws deploy create-deployment-group \
  --application-name my-app \
  --deployment-group-name production \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployServiceRole \
  --ec2-tag-filters Key=Environment,Type=KEY_AND_VALUE,Value=production \
  --deployment-config-name CodeDeployDefault.OneAtATime

# Trigger a manual deployment
aws deploy create-deployment \
  --application-name my-app \
  --deployment-group-name production \
  --s3-location bucket=my-artifacts,bundleType=zip,key=my-app.zip

# Check deployment status
aws deploy get-deployment --deployment-id d-ABCDEF123
```

#### EC2 Tagging for CodeDeploy

Tag your EC2 instances so CodeDeploy knows which ones to target:
```bash
aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key=Environment,Value=production
```

### appspec.yml Lifecycle Hooks Order

```
BeforeBlockTraffic → BlockTraffic → AfterBlockTraffic
→ ApplicationStop
→ BeforeInstall → Install → AfterInstall
→ ApplicationStart
→ ValidateService
→ BeforeAllowTraffic → AllowTraffic → AfterAllowTraffic
```

---

## 4. AWS CodePipeline

CodePipeline orchestrates the entire CI/CD flow — connecting source, build, test, and deploy stages into an automated pipeline triggered on every commit.

### Pipeline Stages

```
[Source]          [Build]         [Test]       [Deploy]
CodeCommit    →  CodeBuild    →  CodeBuild  →  CodeDeploy / ECS / Lambda
(or GitHub,      (build &        (run tests)   (deploy to production)
 S3, ECR)        create artifact)
```

### Create a Pipeline via Console

1. Go to **CodePipeline → Create pipeline**
2. **Source stage**: Choose CodeCommit (or GitHub via OAuth) → select repo and branch
3. **Build stage**: Choose CodeBuild project
4. **Deploy stage**: Choose CodeDeploy application, or ECS service for container deployments
5. **Artifact store**: Create or select an S3 bucket (CodePipeline uses this to pass artifacts between stages)
6. Create pipeline — it runs immediately and on every new commit

### Create via CLI

```bash
aws codepipeline create-pipeline --cli-input-json file://pipeline.json
```

**`pipeline.json`** structure:
```json
{
  "pipeline": {
    "name": "my-app-pipeline",
    "roleArn": "arn:aws:iam::123456789012:role/CodePipelineServiceRole",
    "artifactStore": {
      "type": "S3",
      "location": "my-codepipeline-artifacts"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [{
          "name": "SourceAction",
          "actionTypeId": {
            "category": "Source",
            "owner": "AWS",
            "provider": "CodeCommit",
            "version": "1"
          },
          "configuration": {
            "RepositoryName": "my-app",
            "BranchName": "main"
          },
          "outputArtifacts": [{"name": "SourceOutput"}]
        }]
      },
      {
        "name": "Build",
        "actions": [{
          "name": "BuildAction",
          "actionTypeId": {
            "category": "Build",
            "owner": "AWS",
            "provider": "CodeBuild",
            "version": "1"
          },
          "configuration": {
            "ProjectName": "my-build-project"
          },
          "inputArtifacts": [{"name": "SourceOutput"}],
          "outputArtifacts": [{"name": "BuildOutput"}]
        }]
      },
      {
        "name": "Deploy",
        "actions": [{
          "name": "DeployAction",
          "actionTypeId": {
            "category": "Deploy",
            "owner": "AWS",
            "provider": "CodeDeploy",
            "version": "1"
          },
          "configuration": {
            "ApplicationName": "my-app",
            "DeploymentGroupName": "production"
          },
          "inputArtifacts": [{"name": "BuildOutput"}]
        }]
      }
    ]
  }
}
```

### Pipeline for ECS (Container Deployment)

For containerized apps, the deploy stage uses ECS instead of CodeDeploy:
```json
{
  "name": "Deploy",
  "actions": [{
    "name": "ECSDeployAction",
    "actionTypeId": {
      "category": "Deploy",
      "owner": "AWS",
      "provider": "ECS",
      "version": "1"
    },
    "configuration": {
      "ClusterName": "my-cluster",
      "ServiceName": "my-service",
      "FileName": "imagedefinitions.json"
    },
    "inputArtifacts": [{"name": "BuildOutput"}]
  }]
}
```

The `imagedefinitions.json` (produced by CodeBuild) tells ECS which new image to deploy:
```json
[{"name": "my-container", "imageUri": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:abc1234"}]
```

### Manual Approval Stage

Add a human gate before production deployment:
```json
{
  "name": "Approval",
  "actions": [{
    "name": "ManualApproval",
    "actionTypeId": {
      "category": "Approval",
      "owner": "AWS",
      "provider": "Manual",
      "version": "1"
    },
    "configuration": {
      "NotificationArn": "arn:aws:sns:us-east-1:123456789012:pipeline-approvals",
      "CustomData": "Please review the staging deployment before approving production."
    }
  }]
}
```

---

## Full Pipeline: Code to Production

```
Developer pushes to main branch
    ↓
CodePipeline triggers automatically
    ↓
CodeBuild: npm install → npm test → docker build → push to ECR
    ↓
[Optional] Manual Approval
    ↓
CodeDeploy / ECS: pull new image → rolling deployment → health check
    ↓
App is live in production ✅
```

---

## IAM Roles Required

| Service | Role | Key Permissions |
|---------|------|-----------------|
| CodeBuild | `CodeBuildServiceRole` | CodeCommit read, S3 read/write, ECR push, CloudWatch Logs, SSM |
| CodeDeploy | `CodeDeployServiceRole` | EC2 describe/tag, Auto Scaling, ELB, S3 |
| CodePipeline | `CodePipelineServiceRole` | CodeCommit, CodeBuild, CodeDeploy, S3, SNS |
| EC2 (target) | Instance profile | S3 read (for artifacts), CloudWatch Logs |

---

## Alternative: GitHub Actions + AWS

Many teams prefer GitHub Actions over the AWS CI/CD suite. You can mix and match — e.g., use GitHub Actions for CI and CodeDeploy for deployment:

```yaml
# .github/workflows/deploy.yml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v2
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- name: Login to ECR
  uses: aws-actions/amazon-ecr-login@v1

- name: Build and push image
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPO:$GITHUB_SHA .
    docker push $ECR_REGISTRY/$ECR_REPO:$GITHUB_SHA

- name: Deploy to ECS
  uses: aws-actions/amazon-ecs-deploy-task-definition@v1
  with:
    task-definition: task-definition.json
    service: my-service
    cluster: my-cluster
```