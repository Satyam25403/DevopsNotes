# Azure Container Registry (ACR) & Container Instances (ACI)
## (analogous to Amazon ECR & ECS)

---

## Key Concepts

| Term | Definition |
|------|------------|
| **Docker Image** | A packaged application with all its dependencies, ready to run anywhere |
| **Repository** | A collection of related Docker images, versioned by tags |
| **Registry** | A centralized service that hosts multiple repositories |
| **Container** | A running instance of a Docker image |

---

## Azure Container Registry (ACR)
## (analogous to Amazon ECR)

ACR is Azure's private Docker image registry — fully managed, secure, and integrated with AKS, App Service, and Azure Pipelines.

**Features:**
- Private image storage with Azure RBAC access control
- Vulnerability scanning via **Microsoft Defender for Containers**
- Geo-replication for global availability (Premium tier)
- Automated build tasks (ACR Tasks) to build/push on code commits
- Supports Docker and OCI-compatible images

---

### Creating and Using ACR

```bash
# Create a container registry
az acr create \
  --name myRegistry \
  --resource-group myRG \
  --sku Basic \
  --admin-enabled true

# Login to ACR
az acr login --name myRegistry

# Get login server URL
az acr show --name myRegistry --query loginServer --output tsv
# → myregistry.azurecr.io
```

---

### Pushing Images to ACR

```bash
# Tag your image for ACR
docker tag my-app:latest myregistry.azurecr.io/my-app:latest

# Push the image
docker push myregistry.azurecr.io/my-app:latest

# List repositories
az acr repository list --name myRegistry --output table

# List tags for a repo
az acr repository show-tags \
  --name myRegistry \
  --repository my-app \
  --output table
```

---

### ACR Tasks (build in the cloud, analogous to CodeBuild for images)

```bash
# Build and push directly in Azure (no local Docker needed)
az acr build \
  --registry myRegistry \
  --image my-app:latest \
  .

# Create a recurring build task triggered on git commit
az acr task create \
  --name buildTask \
  --registry myRegistry \
  --image my-app:{{.Run.ID}} \
  --context https://github.com/myorg/myrepo.git \
  --branch main \
  --file Dockerfile \
  --git-access-token <PAT>
```

---

## Azure Container Instances (ACI)
## (analogous to ECS Fargate — serverless containers)

ACI lets you run containers directly in Azure without managing any infrastructure. No clusters, no orchestration — just spin up a container and pay per second.

**Best for:**
- Simple single-container workloads
- Batch jobs and event-driven tasks
- Development and testing
- Quick container deployments without Kubernetes complexity

---

### Running a Container

```bash
# Run a public image
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image nginx:latest \
  --ports 80 \
  --dns-name-label myapp-dns \
  --os-type Linux

# Show the FQDN (public URL)
az container show \
  --name mycontainer \
  --resource-group myRG \
  --query ipAddress.fqdn \
  --output tsv

# View logs
az container logs --name mycontainer --resource-group myRG

# Stream logs
az container attach --name mycontainer --resource-group myRG
```

---

### Running a Private ACR Image

```bash
# Grant ACI access to ACR via managed identity (recommended)
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myregistry.azurecr.io/my-app:latest \
  --acr-identity <managed-identity-resource-id> \
  --assign-identity <managed-identity-resource-id> \
  --ports 3000 \
  --dns-name-label myapp-dns \
  --cpu 1 \
  --memory 1.5

# Alternative: use admin credentials (less secure)
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myregistry.azurecr.io/my-app:latest \
  --registry-login-server myregistry.azurecr.io \
  --registry-username $(az acr credential show -n myRegistry --query username -o tsv) \
  --registry-password $(az acr credential show -n myRegistry --query passwords[0].value -o tsv) \
  --ports 3000
```

---

### Environment Variables & Secrets

```bash
az container create \
  --name mycontainer \
  --resource-group myRG \
  --image myregistry.azurecr.io/my-app:latest \
  --environment-variables NODE_ENV=production PORT=3000 \
  --secure-environment-variables DB_PASSWORD=supersecret \
  --ports 3000
```

---

### Container Lifecycle

```bash
az container start --name mycontainer --resource-group myRG
az container stop --name mycontainer --resource-group myRG
az container restart --name mycontainer --resource-group myRG
az container delete --name mycontainer --resource-group myRG --yes
az container list --output table
```

---

### Multi-Container Groups (analogous to ECS Task Definition with multiple containers)

Deploy multiple containers in the same group — they share the same host, network, and storage:

```yaml
# container-group.yaml
apiVersion: 2021-10-01
location: eastus
name: myContainerGroup
properties:
  containers:
  - name: app
    properties:
      image: myregistry.azurecr.io/my-app:latest
      ports:
      - port: 3000
      resources:
        requests:
          cpu: 1
          memoryInGb: 1
  - name: sidecar
    properties:
      image: nginx:alpine
      ports:
      - port: 80
      resources:
        requests:
          cpu: 0.5
          memoryInGb: 0.5
  osType: Linux
  restartPolicy: Always
  ipAddress:
    type: Public
    ports:
    - protocol: tcp
      port: 3000
```

```bash
az container create --resource-group myRG --file container-group.yaml
```

---

## ACR vs ACI vs AKS — When to Use What

| Scenario | Service |
|----------|---------|
| Store and manage container images | **ACR** |
| Quick single container, no orchestration needed | **ACI** |
| Production workloads, microservices, auto-scaling | **AKS** |
| Simple app deployment with managed infrastructure | **Azure App Service (containers)** |

---

## Key Differences from AWS

| Feature | AWS | Azure |
|---------|-----|-------|
| Container registry | ECR | ACR |
| Serverless containers | ECS Fargate | ACI |
| Managed Kubernetes | EKS | AKS |
| Registry URL format | `<account>.dkr.ecr.<region>.amazonaws.com` | `<registry>.azurecr.io` |
| Image vulnerability scan | Amazon Inspector | Microsoft Defender for Containers |
| Cloud builds | CodeBuild | ACR Tasks |