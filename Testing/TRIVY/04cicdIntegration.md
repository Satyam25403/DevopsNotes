# Trivy - CI/CD Integration

Wiring Trivy into real pipelines so security scanning is automated and enforced, not a manual step someone forgets to run.

## Table of Contents
- [Where Trivy Fits in a Pipeline](#where-trivy-fits-in-a-pipeline)
- [Jenkins Integration](#jenkins-integration)
- [GitHub Actions Integration](#github-actions-integration)
- [GitLab CI Integration](#gitlab-ci-integration)
- [Uploading Results to GitHub Security Tab (SARIF)](#uploading-results-to-github-security-tab-sarif)
- [Caching the Vulnerability Database in CI](#caching-the-vulnerability-database-in-ci)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Where Trivy Fits in a Pipeline

**Visual:**
```
Full Pipeline with Multiple Trivy Checkpoints:

Code Commit → PR Opened
      │
      ├─→ trivy fs .            (scan source dependencies)
      ├─→ trivy config .          (scan Terraform/K8s/Dockerfile)
      └─→ trivy fs --scanners secret .  (scan for hardcoded secrets)
      │
      ↓ (all pass)
Build Docker Image
      │
      └─→ trivy image myapp:$TAG   (scan final built artifact)
      │
      ↓ (passes)
Deploy to Staging
      │
      └─→ trivy k8s (periodic, not per-deploy)  (catch live drift)
      │
      ↓
Deploy to Production
```

**Why multiple checkpoints, not just one:** each catches a different class of problem, and catching issues EARLIER (source code stage) is cheaper to fix than catching them LATER (running cluster stage).

---

## Jenkins Integration

```groovy
pipeline {
    agent any
    stages {
        stage('Dependency Scan') {
            steps {
                sh 'trivy fs --severity CRITICAL,HIGH --exit-code 1 .'
            }
        }
        stage('IaC Scan') {
            steps {
                sh 'trivy config --severity CRITICAL,HIGH --exit-code 1 ./terraform ./k8s'
            }
        }
        stage('Build Image') {
            steps {
                sh 'docker build -t myapp:${BUILD_NUMBER} .'
            }
        }
        stage('Image Scan') {
            steps {
                sh 'trivy image --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed myapp:${BUILD_NUMBER}'
            }
        }
        stage('Deploy') {
            steps {
                sh './deploy.sh'
            }
        }
    }
}
```

**Visual:**
```
Jenkins Stage Flow:
Dependency Scan → IaC Scan → Build → Image Scan → Deploy
       ↓ FAIL          ↓ FAIL              ↓ FAIL
   🛑 Pipeline stops immediately at whichever stage fails —
      Deploy stage NEVER runs if any earlier stage failed
```

---

## GitHub Actions Integration

```yaml
name: Security Scan
on:
  pull_request:
    branches: [main]

jobs:
  trivy-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy Filesystem Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Trivy Config Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'config'
          scan-ref: '.'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'

      - name: Build Image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Trivy Image Scan
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'image'
          image-ref: 'myapp:${{ github.sha }}'
          severity: 'CRITICAL,HIGH'
          exit-code: '1'
          ignore-unfixed: true
```

**Visual:**
```
GitHub Actions PR Flow:
PR opened → checkout → fs scan → config scan → build image →
   image scan → (if all pass) ✓ Required check PASSES →
   merge button enabled

If ANY scan step fails → ✗ Required check FAILS →
   merge button disabled (if branch protection requires this check)
```

---

## GitLab CI Integration

```yaml
stages:
  - scan
  - build
  - deploy

fs-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy fs --severity CRITICAL,HIGH --exit-code 1 .

config-scan:
  stage: scan
  image: aquasec/trivy:latest
  script:
    - trivy config --severity CRITICAL,HIGH --exit-code 1 ./terraform ./k8s

build:
  stage: build
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .

image-scan:
  stage: build
  image: aquasec/trivy:latest
  needs: ["build"]
  script:
    - trivy image --severity CRITICAL,HIGH --exit-code 1 --ignore-unfixed myapp:$CI_COMMIT_SHA

deploy:
  stage: deploy
  needs: ["fs-scan", "config-scan", "image-scan"]
  script:
    - ./deploy.sh
```

**Visual:**
```
GitLab Pipeline DAG:
fs-scan ──┐
config-scan ─┼──→ (all must pass) ──→ deploy
image-scan ──┘

The "needs" keyword creates an explicit dependency —
deploy literally cannot start until all three scan
jobs have completed successfully.
```

---

## Uploading Results to GitHub Security Tab (SARIF)

**Instead of just failing the build, surface findings directly in GitHub's native Security tab for better visibility and tracking.**

```yaml
- name: Trivy Scan (SARIF output)
  uses: aquasecurity/trivy-action@master
  with:
    scan-type: 'fs'
    format: 'sarif'
    output: 'trivy-results.sarif'

- name: Upload to GitHub Security Tab
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

**Visual:**
```
Without SARIF upload:
Findings only visible in CI job logs — easy to miss,
no historical tracking, no easy way to mark something
as "acknowledged" vs "needs fixing" at the platform level

With SARIF upload:
Findings appear in GitHub's Security tab, alongside
CodeQL and other security tooling — trackable over time,
dismissible with a documented reason, visible to the
whole team without digging through CI logs
```

---

## Caching the Vulnerability Database in CI

**Without caching, every single CI run re-downloads the vulnerability database — slow and wasteful.**

```yaml
- name: Cache Trivy DB
  uses: actions/cache@v4
  with:
    path: ~/.cache/trivy
    key: trivy-db-${{ github.run_id }}
    restore-keys: |
      trivy-db-
```

**Visual:**
```
Without caching:
Every CI run → downloads ~150MB vulnerability DB →
   adds 30-60 seconds to EVERY pipeline run,
   across potentially hundreds of runs per day

With caching:
First run downloads DB → cached → subsequent runs
   reuse cached DB (refreshing only if stale) →
   saves significant CI time and bandwidth at scale
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is standardizing Trivy scanning across 20 GitHub repositories and wants consistent enforcement without each team configuring it slightly differently.

**What they build:**
1. A **shared reusable GitHub Actions workflow** (`trivy-security-scan.yml`) containing the fs/config/image scan sequence with sensible defaults, which each of the 20 repos simply calls via `workflow_call` — avoiding 20 slightly-different, independently-maintained copies of the same logic.
2. Configures the shared workflow to **upload SARIF results** to each repo's Security tab, giving security team members a single place to review findings across all 20 repos without needing direct access to each team's CI logs.
3. Adds **DB caching** to the shared workflow, cutting a noticeable amount of time off every single pipeline run across the whole organization, at a meaningful aggregate CI cost savings given how many times per day these pipelines run.
4. Sets **branch protection rules** on `main` across all 20 repos requiring the shared workflow's status check to pass — enforced centrally via a script using the GitHub API, rather than trusting each team to configure it correctly themselves.
5. Establishes that any team needing a `.trivyignore` entry must get it **approved via PR review** by a member of the security team, preventing teams from silently suppressing findings they simply don't want to deal with.

**Why this matters:** Building one well-tested, centrally-maintained reusable workflow — rather than asking 20 teams to each copy-paste and maintain their own Trivy configuration — is what makes security scanning consistent and genuinely enforced across a growing organization, instead of slowly drifting into 20 different half-working setups.

---

Next: **05advanced_realworld_usecases.md** — SBOM generation, Trivy server/client mode, Kubernetes Operator, registry scanning, and mature real-world DevSecOps practices.