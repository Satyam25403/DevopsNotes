# GCP IAM (Identity and Access Management)

GCP IAM is Google Cloud's centralized service for controlling **who** can access your GCP resources and **what** they can do. It manages authentication (who you are) and authorization (what you're allowed to do) across your entire Google Cloud organization.

---

## Core Concepts

### Resource Hierarchy
GCP organizes resources in a parent-child hierarchy. IAM policies set at a higher level are inherited by all resources below it:

```
Organization
  └── Folders (optional grouping, e.g., teams or environments)
        └── Projects
              └── Resources (VMs, buckets, databases, etc.)
```

### Principals (Who)
A **principal** is any identity that can be granted access:

| Principal Type | Description |
|----------------|-------------|
| `user:` | A Google account (person) |
| `group:` | A Google Group (collection of users) |
| `serviceAccount:` | A non-human identity for apps/services |
| `domain:` | All users in a Google Workspace domain |
| `allAuthenticatedUsers` | Any authenticated Google account |
| `allUsers` | Anyone, including unauthenticated (public) |

### Roles (What)
Roles are collections of **permissions**. GCP has three types:

| Role Type | Example | Description |
|-----------|---------|-------------|
| **Basic** | `roles/owner`, `roles/editor`, `roles/viewer` | Broad, legacy roles — avoid in production |
| **Predefined** | `roles/storage.objectViewer` | Fine-grained, service-specific roles managed by Google |
| **Custom** | `projects/my-proj/roles/MyRole` | You define exactly which permissions to include |

> **Best Practice**: Always use **Predefined** or **Custom** roles. Basic roles grant far more permissions than most workloads need.

### Service Accounts
Service accounts are identities for applications, VMs, or services — not people.

**Key properties:**
- Has an email address: `my-sa@my-project.iam.gserviceaccount.com`
- Can be granted roles on resources
- Applications authenticate as service accounts via key files or Workload Identity
- A VM running with an attached service account uses it automatically (no key files needed)

> **Best Practice**: Attach service accounts to Compute Engine VMs and GKE workloads instead of distributing key files. This is the GCP equivalent of AWS EC2 Instance Profiles.

---

## IAM Policy

An **IAM Policy** is a list of **bindings** — each binding maps a role to a set of principals.

```json
{
  "bindings": [
    {
      "role": "roles/storage.objectViewer",
      "members": [
        "user:alice@example.com",
        "serviceAccount:my-app@my-project.iam.gserviceaccount.com"
      ]
    },
    {
      "role": "roles/compute.instanceAdmin",
      "members": [
        "group:devops@example.com"
      ]
    }
  ]
}
```

---

## CLI Usage

### View IAM Policy
```bash
# Project-level policy
gcloud projects get-iam-policy my-project-id

# Resource-level policy (e.g., a bucket)
gcloud storage buckets get-iam-policy gs://my-bucket
```

### Grant a Role
```bash
# Grant a user viewer access on a project
gcloud projects add-iam-policy-binding my-project-id \
  --member="user:alice@example.com" \
  --role="roles/viewer"

# Grant a service account access to a specific bucket
gcloud storage buckets add-iam-policy-binding gs://my-bucket \
  --member="serviceAccount:my-app@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectAdmin"
```

### Revoke a Role
```bash
gcloud projects remove-iam-policy-binding my-project-id \
  --member="user:alice@example.com" \
  --role="roles/viewer"
```

---

## Service Accounts

### Create a Service Account
```bash
gcloud iam service-accounts create my-service-account \
  --display-name="My App Service Account" \
  --description="Used by my-app to access Cloud Storage"
```

### List Service Accounts
```bash
gcloud iam service-accounts list
```

### Grant a Role to a Service Account
```bash
gcloud projects add-iam-policy-binding my-project-id \
  --member="serviceAccount:my-service-account@my-project.iam.gserviceaccount.com" \
  --role="roles/storage.objectViewer"
```

### Create and Download a Key File (use sparingly)
```bash
gcloud iam service-accounts keys create sa-key.json \
  --iam-account=my-service-account@my-project.iam.gserviceaccount.com
```

> **Warning**: Key files are long-lived credentials. Prefer **Workload Identity** for GKE or **attached service accounts** for Compute Engine instead of distributing key files.

### Impersonate a Service Account (for testing)
```bash
gcloud compute instances list \
  --impersonate-service-account=my-sa@my-project.iam.gserviceaccount.com
```

---

## Workload Identity Federation

**Workload Identity Federation** lets external identities (GitHub Actions, GitLab CI, on-prem services) authenticate to GCP **without service account key files**.

```bash
# Create a Workload Identity Pool
gcloud iam workload-identity-pools create "github-pool" \
  --location="global" \
  --display-name="GitHub Actions Pool"

# Create an OIDC Provider (e.g., for GitHub Actions)
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --issuer-uri="https://token.actions.githubusercontent.com" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository"

# Grant the external identity access to a service account
gcloud iam service-accounts add-iam-policy-binding my-sa@my-project.iam.gserviceaccount.com \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/github-pool/attribute.repository/my-org/my-repo"
```

---

## Custom Roles

```bash
# Create a custom role from a YAML definition
gcloud iam roles create myCustomRole \
  --project=my-project-id \
  --file=role-definition.yaml
```

Example `role-definition.yaml`:
```yaml
title: "My Custom Role"
description: "Read-only access to Cloud Storage and Cloud Run"
stage: "GA"
includedPermissions:
  - storage.objects.get
  - storage.objects.list
  - run.services.get
  - run.services.list
```

---

## Key Predefined Roles (DevOps Common)

| Role | What It Allows |
|------|---------------|
| `roles/owner` | Full control of all resources (avoid!) |
| `roles/editor` | Edit all resources (avoid!) |
| `roles/viewer` | Read-only on all resources |
| `roles/iam.securityAdmin` | Manage IAM policies |
| `roles/storage.admin` | Full Cloud Storage control |
| `roles/storage.objectViewer` | Read objects in buckets |
| `roles/compute.instanceAdmin` | Create/manage VMs |
| `roles/container.admin` | Full GKE control |
| `roles/run.admin` | Manage Cloud Run services |
| `roles/cloudfunctions.developer` | Deploy and manage Cloud Functions |
| `roles/logging.viewer` | View logs in Cloud Logging |

---

## Best Practices

- **Least privilege**: Grant only the permissions needed for the task
- **Use groups**: Assign roles to Google Groups, not individual users — easier to manage at scale
- **Avoid basic roles** (`owner`, `editor`, `viewer`) in production — they're too broad
- **Prefer attached service accounts** over key files for Compute Engine and GKE
- **Use Workload Identity Federation** for CI/CD systems instead of creating and distributing key files
- **Audit regularly**: Use IAM Recommender to identify over-privileged accounts
- **Condition-based access**: Use IAM Conditions to restrict access by time, IP range, or resource tags