# Google Cloud Shell

Google Cloud Shell is a browser-based shell environment pre-authenticated with your Google Cloud Console credentials. It gives you instant access to the `gcloud` CLI, Cloud SDK tools, and your GCP resources — without installing anything locally.

---

## What Is Cloud Shell?

- **Browser-based**: Launch directly from the GCP Console (terminal icon in the top navigation bar).
- **Pre-authenticated**: Uses your Console session credentials — no `gcloud auth login` needed.
- **Debian Linux**: Comes with `gcloud` CLI, `kubectl`, `terraform`, `git`, `docker`, `python3`, `node`, `npm`, `go`, `java`, `vim`, `nano`, and more preinstalled.
- **5 GB persistent home directory** (`$HOME`): Scripts, SSH keys, and configs persist between sessions.
- **Free**: No cost to use Cloud Shell — you only pay for GCP resources you create.
- **Supports bash and tmux**.

---

## How to Launch

1. Log into the [GCP Console](https://console.cloud.google.com).
2. Click the **Cloud Shell icon** (terminal icon `>_`) in the top-right navigation bar.
3. A terminal opens at the bottom of the browser — you're ready to run `gcloud` commands immediately.

### Open in Full Screen
Click **"Open in new window"** (pop-out icon) for a full-tab terminal experience.

---

## Key Features

- **No setup**: Works from any device with a browser — no local install required.
- **IAM-scoped**: All commands run with the permissions of your currently logged-in Google account.
- **File upload/download**: Use the three-dot menu (⋮) to upload files to Cloud Shell or download files back to your machine.
- **Multiple tabs**: Click **"+"** to open multiple terminal sessions.
- **Web Preview**: Run a local web server and preview it in your browser via a forwarded port (default: 8080).
- **Code Editor**: Click **"Open Editor"** to launch a VS Code-like web editor backed by your Cloud Shell environment.
- **Boost mode**: Temporarily upgrade to a more powerful VM (e2-medium) for heavier tasks.

---

## Pre-installed Tools

| Tool | Purpose |
|------|---------|
| `gcloud` | Google Cloud CLI |
| `gsutil` | Cloud Storage CLI (legacy) |
| `bq` | BigQuery CLI |
| `kubectl` | Kubernetes CLI |
| `terraform` | Infrastructure as Code |
| `docker` | Container builds (limited) |
| `git` | Version control |
| `python3` / `pip3` | Python runtime |
| `node` / `npm` | Node.js runtime |
| `go` | Go runtime |
| `java` / `mvn` | Java runtime and Maven |
| `jq` | JSON processor |
| `vim`, `nano`, `emacs` | Text editors |

---

## Useful Cloud Shell Tricks

```bash
# Your active project
echo $DEVSHELL_PROJECT_ID
gcloud config get-value project

# Set the active project
gcloud config set project my-project-id

# Web preview on a custom port (opens browser tab)
# Start your app on port 8080, then click "Web Preview" > "Preview on port 8080"
python3 -m http.server 8080

# Download a file from Cloud Shell to your local machine
# Use the three-dot menu → "Download file" → enter the path
# Or via cloudshell URI
cloudshell download ./output.csv

# Upload local file to Cloud Shell
# Use the three-dot menu → "Upload file"

# Open Cloud Shell Editor
cloudshell edit .
```

---

## Persistent Storage

Your `$HOME` directory (`/home/<username>`) persists across sessions — everything else resets.

```bash
# Store scripts, configs, and SSH keys here
ls $HOME           # /home/your-google-account

# Cloud Shell machine resets between inactive sessions, but $HOME persists
# Example: save your commonly-used aliases
echo 'alias k=kubectl' >> ~/.bashrc
source ~/.bashrc
```

---

## Cloud Shell + Terraform

Cloud Shell comes with Terraform preinstalled — great for quick IaC experiments:

```bash
# Verify
terraform --version

# Initialize and apply a configuration
mkdir my-infra && cd my-infra
cat > main.tf << 'EOF'
provider "google" {
  project = var.project_id
  region  = "us-central1"
}

resource "google_storage_bucket" "demo" {
  name     = "my-demo-bucket-${var.project_id}"
  location = "US"
}
EOF

terraform init
terraform plan -var="project_id=$DEVSHELL_PROJECT_ID"
terraform apply -var="project_id=$DEVSHELL_PROJECT_ID" -auto-approve
```

---

## Cloud Shell Limitations

| Limitation | Detail |
|-----------|--------|
| **Ephemeral VM** | The VM resets after ~20 min of inactivity — only `$HOME` persists |
| **Not for production** | Intended for interactive use and development, not long-running workloads |
| **5 GB home storage** | Not suitable for large data processing |
| **CPU/RAM limits** | Standard VM is modest (e2-small); Boost mode available temporarily |
| **No root access** | `sudo` is available for package installs but not persistent |
| **Weekly usage limits** | If you exceed free tier, Cloud Shell is throttled |

---

## Cloud Shell vs Local gcloud

| Scenario | Use Cloud Shell | Use Local gcloud |
|----------|----------------|-----------------|
| Quick one-off commands | ✅ | ✅ |
| Working from a locked-down machine | ✅ | ❌ |
| Local file operations | ❌ | ✅ |
| Long-running scripts | ❌ | ✅ |
| Consistent environment across team | ✅ (same tools preinstalled) | ❌ (version drift) |