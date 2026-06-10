# Google Cloud CLI (gcloud)

The `gcloud` CLI is the primary tool for interacting with Google Cloud services from your terminal — perfect for developers, DevOps engineers, and cloud architects who want to automate deployments, manage infrastructure, and script cloud operations.

---

## What You Can Do with gcloud CLI

- **Manage GCP resources**: Compute Engine VMs, Cloud Storage buckets, IAM, Cloud Run, GKE, and more
- **Automate workflows**: Use shell scripts or CI/CD pipelines to deploy infrastructure
- **Query and filter data**: Get JSON/YAML outputs and filter with `--format` and `--filter` flags
- **Switch projects and accounts**: Work across multiple GCP projects with ease

---

## 1. Installation

### Linux
```bash
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
```

### Windows
Download and run the installer from:
```
https://cloud.google.com/sdk/docs/install
```

### macOS (Homebrew)
```bash
brew install --cask google-cloud-sdk
```

### Verify Installation
```bash
gcloud --version
```

---

## 2. Configure Credentials

### Interactive Login
```bash
gcloud auth login
```
Opens a browser for Google account authentication.

### Application Default Credentials (recommended for apps/CI)
```bash
gcloud auth application-default login
```

### Using Service Accounts (recommended for CI/CD)
```bash
# Activate a service account with a key file
gcloud auth activate-service-account --key-file=sa-key.json

# Or set via environment variable
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/sa-key.json
```

### Set Default Project and Region
```bash
gcloud config set project my-project-id
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

### Using Named Configurations (for multiple projects)
```bash
gcloud config configurations create dev
gcloud config configurations create prod

# Switch between configurations
gcloud config configurations activate prod

# Or use --project flag per command
gcloud compute instances list --project=my-other-project
```

---

## 3. Common Commands

```bash
# Compute Engine (VMs)
gcloud compute instances list                          # List all VMs
gcloud compute instances create my-vm --zone=us-central1-a --machine-type=e2-medium
gcloud compute instances start my-vm --zone=us-central1-a
gcloud compute instances stop my-vm --zone=us-central1-a
gcloud compute ssh my-vm --zone=us-central1-a         # SSH into VM

# Cloud Storage
gcloud storage ls                                      # List all buckets
gcloud storage ls gs://my-bucket/                     # List bucket contents
gcloud storage cp myfile.txt gs://my-bucket/          # Upload file
gcloud storage cp gs://my-bucket/myfile.txt .         # Download file
gcloud storage rsync -r ./local-dir gs://my-bucket/   # Sync directory

# IAM
gcloud iam service-accounts list
gcloud projects get-iam-policy my-project-id
gcloud projects add-iam-policy-binding my-project-id \
  --member="user:alice@example.com" --role="roles/viewer"

# Cloud Run
gcloud run services list
gcloud run deploy my-service --image=gcr.io/my-project/my-image --region=us-central1

# GKE
gcloud container clusters list
gcloud container clusters get-credentials my-cluster --region=us-central1

# Cloud Functions
gcloud functions list
gcloud functions deploy my-function --runtime=nodejs20 --trigger-http
```

---

## 4. Output Formatting

```bash
# JSON output (for scripting)
gcloud compute instances list --format=json

# YAML output
gcloud compute instances list --format=yaml

# Table with custom columns
gcloud compute instances list --format="table(name, zone, status, machineType)"

# Filter results
gcloud compute instances list --filter="status=RUNNING"
gcloud compute instances list --filter="zone:us-central1-a AND status=RUNNING"
```

---

## 5. Key Flags

| Flag | Description |
|------|-------------|
| `--project` | Override the active project |
| `--region` | Specify region (for regional resources) |
| `--zone` | Specify zone (for zonal resources) |
| `--format` | Output format: `json`, `yaml`, `table`, `value` |
| `--filter` | Filter results using a condition |
| `--quiet` / `-q` | Suppress confirmation prompts |
| `--verbosity` | Set log verbosity: `debug`, `info`, `warning`, `error` |

---

## 6. Useful Utilities Bundled with Cloud SDK

| Tool | Purpose |
|------|---------|
| `gsutil` | Legacy Cloud Storage CLI (now superseded by `gcloud storage`) |
| `bq` | BigQuery CLI — run queries, manage datasets and tables |
| `kubectl` | Kubernetes CLI — automatically configured by `gcloud container get-credentials` |
| `gcloud beta` / `gcloud alpha` | Preview features not yet in GA |

---

## 7. CI/CD Usage

```bash
# Authenticate with a service account in a CI pipeline
echo "$GCP_SA_KEY" | base64 --decode > sa-key.json
gcloud auth activate-service-account --key-file=sa-key.json
gcloud config set project $GCP_PROJECT_ID

# Authenticate Docker to push to Artifact Registry
gcloud auth configure-docker us-central1-docker.pkg.dev

# Deploy to Cloud Run in CI
gcloud run deploy my-app \
  --image=us-central1-docker.pkg.dev/my-project/my-repo/my-app:$IMAGE_TAG \
  --region=us-central1 \
  --platform=managed \
  --quiet
```

---

## Quick Reference

```bash
gcloud help                        # General help
gcloud [command] --help            # Help for a specific command
gcloud info                        # Show active configuration
gcloud config list                 # Show all config values
gcloud auth list                   # List credentialed accounts
gcloud projects list               # List all accessible projects
gcloud services list --enabled     # List enabled APIs for current project
```