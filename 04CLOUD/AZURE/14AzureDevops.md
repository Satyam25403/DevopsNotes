# Azure DevOps & Pipelines
## (analogous to AWS CodePipeline / CodeBuild / CodeDeploy + GitHub)

Azure DevOps is a suite of developer tools covering the full software delivery lifecycle — source control, CI/CD pipelines, work tracking, artifact management, and test plans. It can work standalone or alongside GitHub.

---

## Azure DevOps Services

| Service | Description | AWS / GitHub Equivalent |
|---------|-------------|------------------------|
| **Azure Repos** | Git repositories (private, unlimited) | CodeCommit / GitHub |
| **Azure Pipelines** | CI/CD — build, test, deploy | CodeBuild + CodePipeline |
| **Azure Boards** | Work items, sprints, Kanban | Jira / GitHub Issues |
| **Azure Artifacts** | Package registry (npm, NuGet, Maven, PyPI) | CodeArtifact |
| **Azure Test Plans** | Manual and automated test management | — |

> Azure Pipelines also works with GitHub repos — many teams use GitHub for code and Azure Pipelines for CI/CD.

---

## Azure Pipelines: Core Concepts

| Concept | Description | AWS Equivalent |
|---------|-------------|----------------|
| **Pipeline** | YAML file defining the full CI/CD process | CodePipeline |
| **Stage** | Logical phase (Build, Test, Deploy) | CodePipeline Stage |
| **Job** | Group of steps running on one agent | CodeBuild Project |
| **Step / Task** | Individual action (run script, deploy to AKS) | CodeBuild build step |
| **Agent** | Machine that runs jobs | CodeBuild compute |
| **Service Connection** | Stored credential for external services | CodePipeline IAM roles |
| **Variable Group** | Shared variables across pipelines | Parameter Store |
| **Artifact** | Build output passed between stages | CodePipeline Artifact |
| **Environment** | Deployment target with approvals and history | CodeDeploy Deployment Group |

---

## Pipeline YAML: Full Example (Node.js → AKS)

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
      - release/*

variables:
  imageRepository: "my-app"
  containerRegistry: "myregistry.azurecr.io"
  dockerfilePath: "Dockerfile"
  tag: "$(Build.BuildId)"
  k8sNamespace: "production"

stages:

# ── Stage 1: Build & Test ──────────────────────────────────────────
- stage: Build
  displayName: Build and Test
  jobs:
  - job: BuildJob
    displayName: Build, test, push image
    pool:
      vmImage: ubuntu-latest
    steps:

    - task: NodeTool@0
      inputs:
        versionSpec: "20.x"
      displayName: Install Node.js

    - script: |
        npm ci
        npm run lint
        npm test -- --coverage
      displayName: Install, lint, test

    - task: PublishTestResults@2
      inputs:
        testResultsFormat: JUnit
        testResultsFiles: "**/test-results.xml"
      displayName: Publish test results

    - task: PublishCodeCoverageResults@2
      inputs:
        summaryFileLocation: "coverage/cobertura-coverage.xml"
      displayName: Publish coverage

    - task: Docker@2
      displayName: Build and push Docker image
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: myACRServiceConnection
        tags: |
          $(tag)
          latest

    - publish: k8s/
      artifact: manifests
      displayName: Publish K8s manifests

# ── Stage 2: Deploy to Staging ────────────────────────────────────
- stage: DeployStaging
  displayName: Deploy to Staging
  dependsOn: Build
  condition: succeeded()
  jobs:
  - deployment: DeployToStaging
    displayName: Deploy staging
    environment: staging                   # tracks deployment history in Azure DevOps
    pool:
      vmImage: ubuntu-latest
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: manifests

          - task: KubernetesManifest@1
            displayName: Deploy to AKS (staging)
            inputs:
              action: deploy
              connectionType: azureResourceManager
              azureSubscriptionConnection: myAzureServiceConnection
              azureResourceGroup: myRG
              kubernetesCluster: myAKSCluster
              namespace: staging
              manifests: $(Pipeline.Workspace)/manifests/*.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)

# ── Stage 3: Deploy to Production (with approval gate) ────────────
- stage: DeployProduction
  displayName: Deploy to Production
  dependsOn: DeployStaging
  condition: succeeded()
  jobs:
  - deployment: DeployToProduction
    displayName: Deploy production
    environment: production                # approval gate configured in Azure DevOps UI
    pool:
      vmImage: ubuntu-latest
    strategy:
      runOnce:
        deploy:
          steps:
          - download: current
            artifact: manifests

          - task: KubernetesManifest@1
            displayName: Deploy to AKS (production)
            inputs:
              action: deploy
              connectionType: azureResourceManager
              azureSubscriptionConnection: myAzureServiceConnection
              azureResourceGroup: myRG
              kubernetesCluster: myAKSCluster
              namespace: production
              manifests: $(Pipeline.Workspace)/manifests/*.yaml
              containers: $(containerRegistry)/$(imageRepository):$(tag)
```

---

## Common Pipeline Tasks

### Docker Build & Push

```yaml
- task: Docker@2
  inputs:
    command: buildAndPush
    repository: my-app
    dockerfile: Dockerfile
    containerRegistry: myACRServiceConnection
    tags: $(Build.BuildId)
```

### Azure CLI in a pipeline step

```yaml
- task: AzureCLI@2
  inputs:
    azureSubscription: myAzureServiceConnection
    scriptType: bash
    scriptLocation: inlineScript
    inlineScript: |
      az webapp deploy \
        --resource-group myRG \
        --name my-app \
        --src-path $(Build.ArtifactStagingDirectory)/app.zip \
        --type zip
```

### Deploy ARM / Bicep template

```yaml
- task: AzureResourceManagerTemplateDeployment@3
  inputs:
    deploymentScope: Resource Group
    azureResourceManagerConnection: myAzureServiceConnection
    subscriptionId: $(subscriptionId)
    action: Create Or Update Resource Group
    resourceGroupName: myRG
    location: eastus
    templateLocation: Linked artifact
    csmFile: infra/main.bicep
    overrideParameters: -environment production -appName my-app
```

### Run tests and publish results

```yaml
- script: npm test -- --reporters=jest-junit
  displayName: Run tests

- task: PublishTestResults@2
  inputs:
    testResultsFormat: JUnit
    testResultsFiles: junit.xml
```

---

## Variables and Secrets

### Inline variables

```yaml
variables:
  environment: production
  imageTag: $(Build.BuildId)
```

### Variable Groups (shared across pipelines)

```bash
# Create a variable group via CLI
az pipelines variable-group create \
  --name "production-secrets" \
  --variables NODE_ENV=production PORT=8080 \
  --authorize true

# Add a secret variable (value hidden in logs)
az pipelines variable-group variable create \
  --group-id <group-id> \
  --name DB_PASSWORD \
  --value "supersecret" \
  --secret true
```

```yaml
# Reference in pipeline
variables:
- group: production-secrets
```

### Key Vault integration in Variable Groups

```bash
# Link a variable group to Key Vault — secrets auto-sync at pipeline start
az pipelines variable-group create \
  --name "keyvault-secrets" \
  --authorize true \
  --variables "dummy=placeholder" \
  # Then link to Key Vault in the Azure DevOps UI under Pipelines → Library
```

---

## Environments & Approval Gates

Environments track deployment history and let you gate deployments with manual approvals.

```bash
# Create environments via CLI
az devops environment create --name staging
az devops environment create --name production
```

Then in the Azure DevOps Portal: **Pipelines → Environments → production → Approvals and Checks → Add → Approvals** — pick reviewers who must approve before the production stage runs.

---

## Service Connections

Service connections store credentials used by pipeline tasks to talk to Azure, Docker registries, Kubernetes clusters, etc.

```bash
# Create an Azure Resource Manager service connection (using managed identity)
az devops service-endpoint azurerm create \
  --azure-rm-service-principal-id <client-id> \
  --azure-rm-subscription-id <sub-id> \
  --azure-rm-subscription-name "My Subscription" \
  --azure-rm-tenant-id <tenant-id> \
  --name myAzureServiceConnection
```

---

## Triggers

```yaml
# Branch trigger
trigger:
  branches:
    include: [main, release/*]
    exclude: [feature/*]
  paths:
    include: [src/]            # only trigger if src/ changes

# PR trigger (validate before merge)
pr:
  branches:
    include: [main]
  drafts: false

# Schedule trigger
schedules:
- cron: "0 2 * * *"           # 2 AM UTC daily
  displayName: Nightly build
  branches:
    include: [main]
  always: true                 # run even if no code changes
```

---

## Azure Artifacts (package registry)

```bash
# Create a feed
az artifacts universal publish \
  --organization https://dev.azure.com/myorg \
  --project myProject \
  --scope project \
  --feed myFeed \
  --name my-package \
  --version 1.0.0 \
  --description "My shared package" \
  --path ./dist
```

```bash
# .npmrc — use Azure Artifacts as npm registry
registry=https://pkgs.dev.azure.com/myorg/myProject/_packaging/myFeed/npm/registry/
always-auth=true
```

---

## Bicep & Infrastructure as Code (analogous to CloudFormation / CDK)

Bicep is Azure's native IaC language — cleaner syntax than ARM JSON, compiles to ARM templates.

```bicep
// main.bicep
param location string = resourceGroup().location
param appName string

resource appServicePlan 'Microsoft.Web/serverfarms@2022-03-01' = {
  name: '${appName}-plan'
  location: location
  sku: {
    name: 'B1'
    tier: 'Basic'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource webApp 'Microsoft.Web/sites@2022-03-01' = {
  name: appName
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'NODE|20-lts'
      appSettings: [
        { name: 'NODE_ENV', value: 'production' }
      ]
    }
  }
}

output appUrl string = 'https://${webApp.properties.defaultHostName}'
```

```bash
# Deploy Bicep directly
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters appName=my-app

# Preview changes before deploying (like terraform plan)
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters appName=my-app
```

---

## Key Differences from AWS DevOps Tools

| Feature | AWS | Azure DevOps |
|---------|-----|-------------|
| CI/CD pipelines | CodeBuild + CodePipeline | Azure Pipelines |
| Source control | CodeCommit / GitHub | Azure Repos / GitHub |
| Package registry | CodeArtifact | Azure Artifacts |
| IaC language | CloudFormation / CDK | Bicep / ARM Templates |
| Pipeline as code | buildspec.yml + pipeline JSON | azure-pipelines.yml |
| Approval gates | CodePipeline manual approvals | Environment approvals |
| Secrets in pipeline | Parameter Store / Secrets Manager | Variable Groups + Key Vault |
| Agent/runner | CodeBuild managed environment | Microsoft-hosted agent / self-hosted |