# Trivy - IaC & Misconfiguration Scanning

Catching insecure infrastructure settings and hardcoded secrets BEFORE they're ever deployed — not just known CVEs in packages.

## Table of Contents
- [Misconfigurations vs Vulnerabilities](#misconfigurations-vs-vulnerabilities)
- [Scanning Dockerfiles](#scanning-dockerfiles)
- [Scanning Terraform](#scanning-terraform)
- [Scanning Kubernetes Manifests](#scanning-kubernetes-manifests)
- [Scanning a Live Kubernetes Cluster](#scanning-a-live-kubernetes-cluster)
- [Secret Detection](#secret-detection)
- [Custom Policies with Rego](#custom-policies-with-rego)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Misconfigurations vs Vulnerabilities

**Visual:**
```
┌─────────────────────────────────────────────────┐
│  Vulnerability (CVE)                                    │
│  "This SPECIFIC package version has a KNOWN flaw"          │
│  Example: openssl 3.0.1 has CVE-2024-1234                     │
│                                                        │
│  Misconfiguration                                          │
│  "This SETTING/pattern is generally considered              │
│   insecure practice, regardless of package versions"           │
│  Example: Dockerfile runs as root user (USER not set)             │
│  Example: Terraform S3 bucket has public-read ACL                    │
│  Example: Kubernetes pod has privileged: true                            │
└─────────────────────────────────────────────────┘

Vulnerabilities come from a DATABASE (NVD, GHSA, etc.)
Misconfigurations come from BUILT-IN POLICY RULES
(security best-practice checks, not tied to any specific CVE)
```

---

## Scanning Dockerfiles

```bash
trivy config Dockerfile
```

**Example Dockerfile with issues:**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get install -y curl
COPY . /app
WORKDIR /app
CMD ["./start.sh"]
```

**Visual:**
```
Terminal Output:
┌─────────────────────────────────────────────────┐
│ Dockerfile (dockerfile)                               │
│                                                        │
│ Tests: 3 (SUCCESSES: 0, FAILURES: 3)                    │
│                                                        │
│ ✗ AVD-DS-0002  MEDIUM                                    │
│   Image user should not be 'root'                          │
│   → No USER instruction found; container will run             │
│     as root by default                                           │
│                                                        │
│ ✗ AVD-DS-0026  LOW                                          │
│   No HEALTHCHECK defined                                        │
│                                                        │
│ ✗ AVD-DS-0001  HIGH                                            │
│   'apt-get upgrade' not run — image may contain                    │
│    outdated packages with known vulnerabilities                        │
└─────────────────────────────────────────────────┘
```

**Fixed Dockerfile:**
```dockerfile
FROM ubuntu:22.04
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y curl && \
    rm -rf /var/lib/apt/lists/*
RUN useradd -m appuser
COPY --chown=appuser:appuser . /app
WORKDIR /app
USER appuser
HEALTHCHECK CMD curl -f http://localhost/health || exit 1
CMD ["./start.sh"]
```

**Visual — why "runs as root" matters:**
```
Container running as root:
If an attacker exploits ANY vulnerability in the app,
they gain ROOT privileges INSIDE the container —
easier to escalate to host-level compromise via
container escape techniques.

Container running as a non-root user:
Same exploit gives the attacker only limited,
non-root privileges — a meaningfully smaller
blast radius even if something goes wrong.
```

---

## Scanning Terraform

```bash
trivy config ./terraform/
```

**Example Terraform with an issue:**
```hcl
resource "aws_s3_bucket" "data" {
  bucket = "company-data-bucket"
}

resource "aws_s3_bucket_acl" "data_acl" {
  bucket = aws_s3_bucket.data.id
  acl    = "public-read"
}
```

**Visual:**
```
Terminal Output:
┌─────────────────────────────────────────────────┐
│ terraform/main.tf                                     │
│                                                        │
│ ✗ AVD-AWS-0088  CRITICAL                                │
│   S3 Bucket has public read access                        │
│   → aws_s3_bucket_acl.data_acl sets acl = "public-read"       │
│   → Anyone on the internet can read every object                │
│     in this bucket                                                 │
└─────────────────────────────────────────────────┘
```

**Visual — why catching this in Terraform BEFORE apply matters:**
```
Without IaC scanning:
Terraform apply succeeds → S3 bucket goes live PUBLIC →
   sensitive data exposed → discovered weeks later by
   a security audit, or worse, by an attacker

With IaC scanning in the pipeline:
terraform plan → trivy config scan → CRITICAL finding →
   pipeline BLOCKS the apply → developer fixes acl
   BEFORE anything ever goes live
```

---

## Scanning Kubernetes Manifests

```bash
trivy config ./k8s-manifests/
```

**Example manifest with issues:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
    - name: myapp
      image: myapp:latest
      securityContext:
        privileged: true
```

**Visual:**
```
Terminal Output:
┌─────────────────────────────────────────────────┐
│ pod.yaml (kubernetes)                                 │
│                                                        │
│ ✗ AVD-KSV-0017  CRITICAL                                │
│   Container is running in privileged mode                    │
│   → privileged: true grants the container almost                │
│     all capabilities of the host machine itself                     │
│                                                        │
│ ✗ AVD-KSV-0013  MEDIUM                                      │
│   Image tag ':latest' used                                       │
│   → Non-deterministic; the same manifest could deploy                 │
│     a DIFFERENT image content over time                                │
└─────────────────────────────────────────────────┘
```

**Visual — why `privileged: true` is dangerous:**
```
Normal container:
Isolated from the host via Linux namespaces/cgroups —
even a compromised container is contained

Privileged container:
Nearly equivalent to running directly on the host —
a compromised privileged container can often access
host devices, other containers, and potentially the
underlying node itself
```

---

## Scanning a Live Kubernetes Cluster

**Beyond static manifest files, Trivy can scan resources ALREADY RUNNING in a cluster.**

```bash
trivy k8s --report summary cluster
```

**Visual:**
```
Terminal Output:
┌─────────────────────────────────────────────────┐
│  Kubernetes Cluster Security Report                     │
│                                                        │
│  Namespace: production                                    │
│    Deployment/payments-api                                   │
│      Vulnerabilities: 5 (2 CRITICAL)                              │
│      Misconfigurations: 2                                             │
│                                                        │
│  Namespace: staging                                          │
│    Deployment/test-service                                         │
│      Vulnerabilities: 12 (0 CRITICAL)                                  │
│      Misconfigurations: 4                                                    │
└─────────────────────────────────────────────────┘

This catches DRIFT — resources that were deployed
before scanning was enforced in the pipeline, or
manually kubectl-applied outside of CI/CD entirely.
```

---

## Secret Detection

```bash
trivy fs --scanners secret .
```

**Example file with a hardcoded secret:**
```python
# config.py
API_KEY = "sk_live_51H8xJ2K9nP3mQ7rT"
DATABASE_URL = "postgresql://admin:SuperSecret123@db.internal:5432/prod"
```

**Visual:**
```
Terminal Output:
┌─────────────────────────────────────────────────┐
│ config.py                                                │
│                                                        │
│ ✗ AWS Access Key ID (aws-access-key-id)                     │
│   Severity: CRITICAL                                             │
│   Line 2: API_KEY = "sk_live_51H8xJ2K9nP3mQ7rT"                     │
│                                                        │
│ ✗ Generic Password (generic-password)                             │
│   Severity: HIGH                                                       │
│   Line 3: DATABASE_URL = "postgresql://admin:Sup...                     │
└─────────────────────────────────────────────────┘
```

**Visual — secret scanning also applies to container image LAYERS:**
```
trivy image myapp:latest
→ ALSO scans each image layer's filesystem for secrets,
  catching cases where a secret was copied in during
  an earlier build stage and never fully removed —
  even if the FINAL layer doesn't show it directly,
  it can still be extracted from earlier layers.
```

---

## Custom Policies with Rego

**Trivy's built-in checks cover common cases, but organizations can add custom rules using Open Policy Agent's Rego language.**

```rego
# custom-policies/no-default-namespace.rego
package custom.kubernetes

deny[msg] {
  input.kind == "Deployment"
  input.metadata.namespace == "default"
  msg := "Deployments must not be deployed to the 'default' namespace"
}
```

```bash
trivy config --policy ./custom-policies/ ./k8s-manifests/
```

**Visual:**
```
Why custom policies matter:
Built-in Trivy checks cover industry-standard best
practices (no privileged containers, no public S3
buckets, etc.) — but every organization has its OWN
specific rules (e.g. "all deployments must have a
cost-center label", "no deployments to default
namespace") that only custom Rego policies can enforce.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is building out a "shift-left" security practice, wanting infrastructure misconfigurations caught during code review, not after deployment.

**What they implement:**
1. Adds `trivy config` as a **pre-commit hook** for Terraform and Kubernetes manifest changes, giving developers instant feedback in their own terminal before they even open a pull request.
2. Adds the same scan as a **required CI check** on every PR touching `terraform/` or `k8s-manifests/`, blocking merge on CRITICAL findings like public storage buckets or privileged containers — catching anyone who skipped or bypassed the local pre-commit hook.
3. Adds **secret scanning** (`trivy fs --scanners secret`) to the same PR check, catching a real incident-in-the-making where a developer had accidentally hardcoded a database password while debugging locally and almost committed it.
4. Writes a **custom Rego policy** enforcing the company's internal requirement that every Kubernetes Deployment must include a `cost-center` label for billing attribution — something no built-in Trivy check covers, since it's specific to their organization.
5. Runs a **weekly `trivy k8s` scan** against the live production cluster (not just manifest files in Git) to catch configuration drift — resources that were manually `kubectl edit`-ed outside the normal GitOps pipeline, bypassing the PR-based checks entirely.

**Why this matters:** Static manifest scanning alone doesn't catch drift from manual cluster changes, and vulnerability scanning alone doesn't catch insecure configuration patterns — combining IaC scanning, secret detection, and live-cluster scanning closes gaps that any single scan type would miss.

---

Next: **04cicd_integration.md** — wiring Trivy into Jenkins, GitHub Actions, and GitLab CI as an automated, enforced security gate.