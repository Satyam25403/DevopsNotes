# Azure App Service
## (analogous to AWS Elastic Beanstalk / App Runner)

Azure App Service is a fully managed PaaS for hosting web apps, REST APIs, and mobile backends. You bring the code or container — Azure handles the OS, runtime, patching, scaling, and load balancing.

Supported runtimes: Node.js, Python, .NET, Java, PHP, Ruby, Go, and custom containers.

---

## Core Concepts

### App Service Plan (analogous to Elastic Beanstalk environment tier / EC2 instance type)
The underlying compute that powers your app. Defines the region, number of VM instances, and pricing tier. Multiple apps can share one plan.

| Tier | Description | Use Case |
|------|-------------|----------|
| **Free / Shared** | Shared infrastructure, no custom domain | Dev/test only |
| **Basic** | Dedicated VMs, manual scale, custom domains | Non-production |
| **Standard** | Auto-scale, staging slots, daily backups | Production |
| **Premium** | More CPU/RAM, VNet integration, more slots | High-traffic production |
| **Isolated** | Dedicated environment (App Service Environment) | Enterprise/compliance |

### Deployment Slots (analogous to Elastic Beanstalk environments with swap)
Named environments (staging, canary, etc.) within the same app. You can **swap** slots with zero downtime — staging becomes production, production becomes staging. Slots have their own URLs and settings.

### Web App
The actual application resource that runs inside a plan. Has its own hostname, configuration, and deployment pipeline.

---

## Creating an App

```bash
# Create an App Service Plan
az appservice plan create \
  --name myAppPlan \
  --resource-group myRG \
  --location eastus \
  --sku B1 \
  --is-linux

# Create a web app (Node.js)
az webapp create \
  --resource-group myRG \
  --plan myAppPlan \
  --name my-unique-app-name \
  --runtime "NODE:20-lts"

# App is immediately accessible at:
# https://my-unique-app-name.azurewebsites.net
```

---

## Deploying Code

### Deploy from local Git
```bash
# Configure local git deployment
az webapp deployment source config-local-git \
  --name my-unique-app-name \
  --resource-group myRG

# Push code (Azure gives you a remote URL)
git remote add azure <deployment-url>
git push azure main
```

### Deploy a ZIP package
```bash
az webapp deploy \
  --resource-group myRG \
  --name my-unique-app-name \
  --src-path ./app.zip \
  --type zip
```

### Deploy a Docker container
```bash
# Create web app from a container image
az webapp create \
  --resource-group myRG \
  --plan myAppPlan \
  --name my-unique-app-name \
  --deployment-container-image-name myregistry.azurecr.io/my-app:latest

# Enable continuous deployment from ACR
az webapp config container set \
  --name my-unique-app-name \
  --resource-group myRG \
  --container-image-name myregistry.azurecr.io/my-app:latest \
  --container-registry-url https://myregistry.azurecr.io
```

---

## Configuration

```bash
# Set app settings (environment variables)
az webapp config appsettings set \
  --resource-group myRG \
  --name my-unique-app-name \
  --settings NODE_ENV=production PORT=8080 API_KEY=@Microsoft.KeyVault(SecretUri=https://myvault.vault.azure.net/secrets/api-key/)

# Set connection strings
az webapp config connection-string set \
  --resource-group myRG \
  --name my-unique-app-name \
  --connection-string-type SQLAzure \
  --settings DefaultConnection="Server=myserver.database.windows.net;..."

# View current settings
az webapp config appsettings list \
  --resource-group myRG \
  --name my-unique-app-name \
  --output table
```

> **Key Vault references**: Use `@Microsoft.KeyVault(SecretUri=...)` in app settings to pull secrets directly from Key Vault without hardcoding them. The app's managed identity must have `Key Vault Secrets User` role.

---

## Deployment Slots

```bash
# Create a staging slot
az webapp deployment slot create \
  --resource-group myRG \
  --name my-unique-app-name \
  --slot staging

# Deploy to staging (not production)
az webapp deploy \
  --resource-group myRG \
  --name my-unique-app-name \
  --slot staging \
  --src-path ./app.zip \
  --type zip

# Swap staging to production (zero downtime)
az webapp deployment slot swap \
  --resource-group myRG \
  --name my-unique-app-name \
  --slot staging \
  --target-slot production
```

---

## Scaling

```bash
# Scale up (larger VM size) — change the plan tier
az appservice plan update \
  --resource-group myRG \
  --name myAppPlan \
  --sku P1v3

# Scale out manually (add more instances)
az appservice plan update \
  --resource-group myRG \
  --name myAppPlan \
  --number-of-workers 3

# Configure autoscale
az monitor autoscale create \
  --resource-group myRG \
  --resource myAppPlan \
  --resource-type Microsoft.Web/serverfarms \
  --name autoscale-config \
  --min-count 1 \
  --max-count 5 \
  --count 1

# Add CPU-based autoscale rule
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name autoscale-config \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 2
```

---

## Managed Identity (no credentials in app settings)

```bash
# Enable system-assigned identity
az webapp identity assign \
  --resource-group myRG \
  --name my-unique-app-name

# Grant access to Key Vault
az role assignment create \
  --assignee $(az webapp identity show -g myRG -n my-unique-app-name --query principalId -o tsv) \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<sub-id>/resourceGroups/myRG/providers/Microsoft.KeyVault/vaults/myKeyVault
```

In your Node.js app, `DefaultAzureCredential` automatically picks up the managed identity — no changes needed.

---

## Custom Domains & TLS

```bash
# Add custom domain
az webapp config hostname add \
  --resource-group myRG \
  --webapp-name my-unique-app-name \
  --hostname www.mydomain.com

# Create a managed TLS certificate (free)
az webapp config ssl create \
  --resource-group myRG \
  --name my-unique-app-name \
  --hostname www.mydomain.com

# Bind the certificate
az webapp config ssl bind \
  --resource-group myRG \
  --name my-unique-app-name \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI
```

---

## Logs & Diagnostics

```bash
# Enable application logging
az webapp log config \
  --resource-group myRG \
  --name my-unique-app-name \
  --application-logging filesystem \
  --level information

# Stream live logs
az webapp log tail \
  --resource-group myRG \
  --name my-unique-app-name

# Download log files
az webapp log download \
  --resource-group myRG \
  --name my-unique-app-name \
  --log-file logs.zip
```

---

## VNet Integration (for private database access)

```bash
# Integrate the app with a VNet subnet
az webapp vnet-integration add \
  --resource-group myRG \
  --name my-unique-app-name \
  --vnet myVNet \
  --subnet appSubnet
```

Once integrated, outbound traffic from the app (e.g., to a private database) routes through the VNet.

---

## Key Differences from AWS

| Feature | AWS | Azure App Service |
|---------|-----|-------------------|
| Closest equivalent | Elastic Beanstalk / App Runner | App Service |
| Compute unit | Environment/Service | App Service Plan |
| Blue/green deploys | EB environment swap | Deployment Slots |
| Runtime versions | Platform versions | `--runtime` flag |
| Custom containers | App Runner / EB Docker | `--deployment-container-image-name` |
| Free TLS certs | ACM (requires ALB) | App Service Managed Certificate (free) |
| VNet access | VPC integration | VNet Integration |