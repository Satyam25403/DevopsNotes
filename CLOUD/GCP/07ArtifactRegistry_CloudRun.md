# Google Artifact Registry & Cloud Run

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Docker Image** | A packaged application with all its dependencies, ready to run anywhere |
| **Repository** | A collection of related artifacts (Docker images, npm packages, Maven jars) versioned by tags |
| **Registry** | A centralized service that hosts multiple repositories |
| **Container** | A running instance of a Docker image |

---

## Google Artifact Registry

Artifact Registry is GCP's fully managed private registry for Docker images, Helm charts, npm packages, Maven artifacts, and more — the successor to Google Container Registry (GCR).

**Features:**
- IAM-based access control (no separate credentials)
- Vulnerability scanning on push via Container Analysis
- Lifecycle policies to clean up old/untagged images
- Multi-region support for global availability
- Supports Docker, OCI, Helm, npm, Maven, Python, and Go modules
- Integrated with Cloud Build, Cloud Run, and GKE

---

## Setting Up Artifact Registry

```bash
# Enable the API
gcloud services enable artifactregistry.googleapis.com

# Create a Docker repository
gcloud artifacts repositories create my-repo \
  --repository-format=docker \
  --location=us-central1 \
  --description="My app Docker images"

# List repositories
gcloud artifacts repositories list

# Configure Docker authentication
gcloud auth configure-docker us-central1-docker.pkg.dev
```

---

## Pushing Images to Artifact Registry

```bash
# 1. Build your Docker image
docker build -t my-app:latest .

# 2. Tag for Artifact Registry
# Format: REGION-docker.pkg.dev/PROJECT/REPO/IMAGE:TAG
docker tag my-app:latest us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest
docker tag my-app:latest us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.2.3

# 3. Push
docker push us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest
docker push us-central1-docker.pkg.dev/my-project/my-repo/my-app:v1.2.3
```

### Pulling Images
```bash
docker pull us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest
```

---

## Vulnerability Scanning

```bash
# List vulnerabilities found in an image
gcloud artifacts docker images list us-central1-docker.pkg.dev/my-project/my-repo/my-app

# Get detailed scan results
gcloud artifacts docker images describe \
  us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --show-package-vulnerability
```

---

## Cleanup Policies (Lifecycle Management)

```bash
# Delete untagged images older than 30 days
gcloud artifacts repositories set-cleanup-policies my-repo \
  --location=us-central1 \
  --policy='[
    {
      "name": "delete-untagged",
      "action": {"type": "Delete"},
      "condition": {
        "tagState": "untagged",
        "olderThan": "2592000s"
      }
    }
  ]'
```

---

## Cloud Run

Cloud Run is GCP's fully managed serverless container platform. You provide a Docker image; Google handles servers, scaling (including scale-to-zero), load balancing, and TLS. It's the GCP equivalent of AWS Fargate + App Runner.

**Key properties:**
- **Serverless containers**: No cluster to manage
- **Scale to zero**: Pay nothing when idle
- **Auto-scaling**: From 0 to thousands of instances in seconds
- **HTTPS by default**: Automatic TLS certificate provisioning
- **Any language/framework**: If it runs in a container, it runs on Cloud Run

---

## Deploying to Cloud Run

```bash
# Deploy directly from an image
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --platform=managed \
  --allow-unauthenticated       # Makes the service publicly accessible

# Deploy with environment variables and secrets
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --set-env-vars="NODE_ENV=production,PORT=8080" \
  --set-secrets="DB_PASSWORD=db-password:latest" \
  --service-account=my-app@my-project.iam.gserviceaccount.com

# Deploy with scaling and resource limits
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:latest \
  --region=us-central1 \
  --memory=512Mi \
  --cpu=1 \
  --min-instances=1 \
  --max-instances=100 \
  --concurrency=80             # Max concurrent requests per instance
```

### Build and Deploy in One Command (Cloud Build)
```bash
gcloud run deploy my-service \
  --source=. \                 # Builds from local Dockerfile automatically
  --region=us-central1 \
  --allow-unauthenticated
```

---

## Manage Cloud Run Services

```bash
# List services
gcloud run services list --region=us-central1

# Describe a service (URL, config, latest revision)
gcloud run services describe my-service --region=us-central1

# View logs
gcloud run services logs read my-service --region=us-central1

# Delete a service
gcloud run services delete my-service --region=us-central1
```

---

## Traffic Splitting (Blue/Green, Canary)

```bash
# Deploy a new revision without sending it traffic
gcloud run deploy my-service \
  --image=gcr.io/my-project/my-app:v2 \
  --region=us-central1 \
  --no-traffic                # Deploy but send 0% traffic

# Gradually shift traffic (canary release)
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00023-abc=10,LATEST=90

# Full cutover to latest
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-latest

# Rollback to a specific revision
gcloud run services update-traffic my-service \
  --region=us-central1 \
  --to-revisions=my-service-00021-xyz=100
```

---

## Cloud Run + Artifact Registry CI/CD Pattern

```bash
# Typical CI/CD pipeline flow:
# 1. Build image with Cloud Build
gcloud builds submit \
  --tag=us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA

# 2. Deploy new revision to Cloud Run
gcloud run deploy my-service \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:$COMMIT_SHA \
  --region=us-central1 \
  --quiet
```

---

## Cloud Run Access Control

```bash
# Allow unauthenticated (public) access
gcloud run services add-iam-policy-binding my-service \
  --region=us-central1 \
  --member="allUsers" \
  --role="roles/run.invoker"

# Restrict to a specific service account (service-to-service auth)
gcloud run services add-iam-policy-binding my-service \
  --region=us-central1 \
  --member="serviceAccount:caller-sa@my-project.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```

---

## Cloud Run vs GKE vs App Engine

| Feature | Cloud Run | GKE | App Engine |
|---------|-----------|-----|-----------|
| Server management | None | You manage nodes | None |
| Scaling | Auto (0 → ∞) | Manual/HPA | Auto |
| Container support | ✅ Any container | ✅ Full Kubernetes | ✅ (Flex) / ❌ (Standard) |
| Cold starts | Yes (configurable min) | No | Yes |
| Complexity | Low | High | Low |
| Best for | Stateless HTTP services | Complex microservices | Simple web apps |