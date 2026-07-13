# GCP CI/CD: Cloud Build, Cloud Deploy & Artifact Registry

GCP provides a fully integrated CI/CD suite that takes code from commit to production. While AWS has CodeCommit/CodeBuild/CodeDeploy/CodePipeline, GCP's equivalent stack is **Cloud Source Repositories + Cloud Build + Cloud Deploy + Artifact Registry**.

```
Source (GitHub/GitLab/CSR) → Cloud Build (CI) → Artifact Registry (Images) → Cloud Deploy (CD)
```

---

## Overview

| Service | Role | AWS Analog |
|---------|------|-----------|
| **Cloud Source Repositories** | Git hosting | CodeCommit |
| **Cloud Build** | Build, test & CI automation | CodeBuild |
| **Artifact Registry** | Container & package registry | ECR |
| **Cloud Deploy** | Managed progressive delivery (CD) | CodeDeploy |

> **Note**: Most teams use **GitHub or GitLab** as their source control and trigger Cloud Build from there — Cloud Source Repositories is less commonly used for new projects.

---

## Cloud Build

Cloud Build is a fully serverless CI platform that executes your build steps in containers. Each step is a Docker container with a command — making builds completely reproducible and language-agnostic.

### `cloudbuild.yaml` — Build File

```yaml
# cloudbuild.yaml
steps:
  # Step 1: Install dependencies
  - name: 'node:20'
    entrypoint: npm
    args: ['ci']

  # Step 2: Run tests
  - name: 'node:20'
    entrypoint: npm
    args: ['test']

  # Step 3: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - build
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:latest
      - .

  # Step 4: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    args:
      - push
      - --all-tags
      - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app

  # Step 5: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - run
      - deploy
      - my-service
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA
      - --region=us-central1
      - --quiet

# Image to retain in Artifact Registry
images:
  - us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA

options:
  logging: CLOUD_LOGGING_ONLY
  machineType: 'E2_HIGHCPU_8'    # Faster build machine

timeout: '1200s'    # 20-minute build timeout
```

### Trigger a Build Manually

```bash
# Run a build from local source
gcloud builds submit \
  --config=cloudbuild.yaml \
  --region=us-central1 .

# Run a build with substitution variables
gcloud builds submit \
  --config=cloudbuild.yaml \
  --substitutions=_ENV=staging,_VERSION=1.2.3 .

# Quick build + push (no config file)
gcloud builds submit \
  --tag=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest .
```

---

## Cloud Build Triggers (CI automation)

Triggers automatically start builds on git events:

```bash
# Create a trigger on push to main branch (GitHub)
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-github-org \
  --branch-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --region=us-central1

# Create a trigger on any PR to main
gcloud builds triggers create github \
  --repo-name=my-repo \
  --repo-owner=my-github-org \
  --pull-request-pattern='^main$' \
  --build-config=cloudbuild.yaml \
  --region=us-central1

# List triggers
gcloud builds triggers list --region=us-central1

# Manually run a trigger
gcloud builds triggers run my-trigger \
  --branch=main \
  --region=us-central1
```

---

## Built-in Cloud Build Variables

| Variable | Value |
|----------|-------|
| `$PROJECT_ID` | GCP project ID |
| `$BUILD_ID` | Unique build ID |
| `$COMMIT_SHA` | Git commit hash |
| `$BRANCH_NAME` | Branch that triggered the build |
| `$TAG_NAME` | Git tag (if triggered by a tag) |
| `$SHORT_SHA` | First 7 chars of commit SHA |
| `$REPO_NAME` | Repository name |

---

## Multi-Environment Pipeline with Substitutions

```yaml
# cloudbuild.yaml
substitutions:
  _ENV: staging          # Default — override per trigger
  _REGION: us-central1

steps:
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA', '.']

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA']

  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - run
      - deploy
      - my-service-${_ENV}
      - --image=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$SHORT_SHA
      - --region=${_REGION}
      - --quiet
```

---

## Cloud Deploy (Managed Continuous Delivery)

Cloud Deploy provides **progressive delivery** to a sequence of targets (dev → staging → prod) with built-in approval gates, rollbacks, and deployment history.

### Delivery Pipeline Definition

```yaml
# clouddeploy.yaml
apiVersion: deploy.cloud.google.com/v1
kind: DeliveryPipeline
metadata:
  name: my-app-pipeline
  location: us-central1
description: Deploy my-app through dev → staging → prod
serialPipeline:
  stages:
    - targetId: dev
      profiles: []
    - targetId: staging
      profiles: [staging]
    - targetId: prod
      profiles: [prod]
      strategy:
        canary:
          runtimeConfig:
            cloudRun:
              automaticTrafficControl: true
          canaryDeployment:
            percentages: [25, 50]
            verify: true
```

```yaml
# targets.yaml
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: dev
  location: us-central1
run:
  location: projects/my-project/locations/us-central1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: staging
  location: us-central1
requireApproval: false
run:
  location: projects/my-project/locations/us-central1
---
apiVersion: deploy.cloud.google.com/v1
kind: Target
metadata:
  name: prod
  location: us-central1
requireApproval: true   # Manual approval gate before prod deployment
run:
  location: projects/my-project/locations/us-central1
```

```bash
# Apply pipeline and targets
gcloud deploy apply --file=clouddeploy.yaml --region=us-central1
gcloud deploy apply --file=targets.yaml --region=us-central1

# Create a release (starts deployment to first stage)
gcloud deploy releases create release-$(date +%Y%m%d-%H%M%S) \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --images=my-app=us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA

# Promote to the next stage
gcloud deploy releases promote \
  --release=RELEASE_NAME \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --to-target=staging

# Approve a deployment (for prod)
gcloud deploy rollouts approve ROLLOUT_NAME \
  --release=RELEASE_NAME \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1

# Rollback a deployment
gcloud deploy rollouts rollback \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1 \
  --release=RELEASE_NAME
```

---

## End-to-End CI/CD Flow

```yaml
# cloudbuild.yaml — Full CI/CD pipeline
steps:
  # CI: Install, Test, Build, Push
  - name: 'node:20'
    entrypoint: npm
    args: ['ci']
    id: install

  - name: 'node:20'
    entrypoint: npm
    args: ['test']
    id: test
    waitFor: [install]

  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA', '.']
    id: build
    waitFor: [test]

  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA']
    id: push
    waitFor: [build]

  # CD: Create a Cloud Deploy release
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - deploy
      - releases
      - create
      - release-$SHORT_SHA
      - --delivery-pipeline=my-app-pipeline
      - --region=us-central1
      - --images=my-app=us-central1-docker.pkg.dev/$PROJECT_ID/my-repo/my-app:$COMMIT_SHA
    waitFor: [push]
```

---

## Viewing Build History & Logs

```bash
# List recent builds
gcloud builds list --region=us-central1 --limit=10

# Describe a specific build
gcloud builds describe BUILD_ID --region=us-central1

# Stream logs for a running build
gcloud builds log BUILD_ID --region=us-central1 --stream

# List Cloud Deploy pipelines
gcloud deploy delivery-pipelines list --region=us-central1

# List releases for a pipeline
gcloud deploy releases list \
  --delivery-pipeline=my-app-pipeline \
  --region=us-central1
```

---

## GitHub Actions → Cloud Build Integration

Many teams use GitHub Actions for CI and Cloud Build/Deploy for CD:

```yaml
# .github/workflows/deploy.yml
name: Deploy to GCP
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write    # For Workload Identity Federation

    steps:
      - uses: actions/checkout@v4

      - id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/providers/github-provider'
          service_account: 'github-actions@my-project.iam.gserviceaccount.com'

      - uses: google-github-actions/setup-gcloud@v2

      - name: Build and push
        run: |
          gcloud auth configure-docker us-central1-docker.pkg.dev --quiet
          docker build -t us-central1-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }} .
          docker push us-central1-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }}

      - name: Deploy to Cloud Run
        run: |
          gcloud run deploy my-service \
            --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:${{ github.sha }} \
            --region=us-central1 --quiet
```