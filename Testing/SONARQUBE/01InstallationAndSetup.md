# SonarQube - Installation & Setup

Getting a SonarQube server running and executing your first scan, with visual walkthroughs.

## Table of Contents
- [Installation Options](#installation-options)
- [Docker Installation (Recommended for Local/DevOps)](#docker-installation-recommended-for-localdevops)
- [Accessing the Dashboard](#accessing-the-dashboard)
- [Generating a Token](#generating-a-token)
- [Installing SonarScanner CLI](#installing-sonarscanner-cli)
- [Running Your First Scan](#running-your-first-scan)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installation Options

**Visual:**
```
┌──────────────────────────────────────────────────────────┐
│                  SonarQube Editions                       │
├──────────────────────────────────────────────────────────┤
│ Community Edition   → Free, core static analysis           │
│ Developer Edition   → + branch analysis, PR decoration      │
│ Enterprise Edition  → + portfolio mgmt, more security rules │
│ Data Center Edition → + high availability, scaling          │
└──────────────────────────────────────────────────────────┘

Installation Methods:
┌─────────────┐   ┌─────────────┐   ┌──────────────────┐   ┌───────────────┐
│ Docker       │   │ ZIP/Binary  │   │ Kubernetes (Helm) │   │ SonarCloud    │
│ (quickest)   │   │ (manual)    │   │ (production)      │   │ (SaaS, hosted)│
└─────────────┘   └─────────────┘   └──────────────────┘   └───────────────┘
```

For DevOps engineers, **Docker** is the fastest way to try it locally, and **Helm on Kubernetes** is the common production setup.

---

## Docker Installation (Recommended for Local/DevOps)

### Step 1: Run SonarQube Server

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  sonarqube:community
```

**Visual:**
```
Before:
Your Machine
┌─────────────────┐
│  (nothing running)│
└─────────────────┘

After docker run:
Your Machine
┌─────────────────────────────────┐
│  Container: sonarqube            │
│  ┌──────────────────────────┐   │
│  │ Web Server  : port 9000  │   │
│  │ Compute Eng.              │   │
│  │ Embedded DB (H2, temp)    │   │  ← use Postgres for real setups
│  └──────────────────────────┘   │
└─────────────────────────────────┘
         ↓ exposed on
   http://localhost:9000
```

⚠️ **Production note:** The default embedded database (H2) is for evaluation only. Real deployments use PostgreSQL:

```bash
docker run -d --name sonarqube \
  -p 9000:9000 \
  -e SONAR_JDBC_URL=jdbc:postgresql://postgres-host:5432/sonar \
  -e SONAR_JDBC_USERNAME=sonar \
  -e SONAR_JDBC_PASSWORD=sonar_password \
  sonarqube:community
```

**Visual:**
```
Production Setup:
┌────────────────┐      ┌──────────────────┐
│  SonarQube      │ ───→ │   PostgreSQL DB   │
│  Container      │      │   (persistent)    │
└────────────────┘      └──────────────────┘

Why? Container restarts lose H2 data.
Postgres survives restarts/redeploys.
```

### Step 2 (Alternative): docker-compose for Full Stack

```yaml
version: "3"
services:
  sonarqube:
    image: sonarqube:community
    depends_on:
      - db
    ports:
      - "9000:9000"
    environment:
      SONAR_JDBC_URL: jdbc:postgresql://db:5432/sonar
      SONAR_JDBC_USERNAME: sonar
      SONAR_JDBC_PASSWORD: sonar
    volumes:
      - sonarqube_data:/opt/sonarqube/data
      - sonarqube_extensions:/opt/sonarqube/extensions

  db:
    image: postgres:13
    environment:
      POSTGRES_USER: sonar
      POSTGRES_PASSWORD: sonar
      POSTGRES_DB: sonar
    volumes:
      - postgresql_data:/var/lib/postgresql/data

volumes:
  sonarqube_data:
  sonarqube_extensions:
  postgresql_data:
```

```bash
docker-compose up -d
```

**Visual:**
```
docker-compose stack:
┌─────────────────────────────────────────────┐
│  Network: default                            │
│  ┌───────────────┐      ┌─────────────────┐  │
│  │  sonarqube      │ ←──→ │  postgres        │  │
│  │  (port 9000)    │      │  (internal only) │  │
│  └───────────────┘      └─────────────────┘  │
│  Volumes persist data across restarts        │
└─────────────────────────────────────────────┘
```

---

## Accessing the Dashboard

Navigate to `http://localhost:9000`

**Default credentials:** `admin` / `admin` (you'll be forced to change this on first login)

**Visual:**
```
Login Screen                  Dashboard (after login)
┌───────────────┐            ┌─────────────────────────────┐
│ Username: admin│            │  Projects   Quality Gates    │
│ Password: admin│  ───────→  │  Quality Profiles  Rules     │
│  [Log In]      │            │  Administration              │
└───────────────┘            └─────────────────────────────┘
```

---

## Generating a Token

Scans authenticate using a token, not username/password. This is critical for CI/CD pipelines.

**Steps:**
```
My Account → Security → Generate Tokens
```

**Visual:**
```
Token Generation UI:
┌─────────────────────────────────────────┐
│ Name: jenkins-ci-token                    │
│ Type: Global Analysis Token              │
│ Expires: 90 days                          │
│  [Generate]                               │
├─────────────────────────────────────────┤
│ Token: sqa_a1b2c3d4e5f6...  ⚠️ shown once │
└─────────────────────────────────────────┘

Store this token as a CI/CD secret:
- Jenkins Credentials Store
- GitHub Actions Secrets
- GitLab CI/CD Variables

NEVER commit it to source code.
```

---

## Installing SonarScanner CLI

The scanner is the tool that actually walks your source code and sends results to the server.

### Linux/Mac

```bash
# Download
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip

# Unzip
unzip sonar-scanner-cli-5.0.1.3006-linux.zip

# Add to PATH
export PATH=$PATH:/path/to/sonar-scanner-5.0.1.3006-linux/bin
```

### Verify Installation

```bash
sonar-scanner -v
```

**Visual:**
```
Output:
INFO: Scanner configuration file: ...
INFO: SonarScanner 5.0.1.3006
INFO: Java 17.0.2 Eclipse Adoptium ...
INFO: Linux 5.15 amd64
```

### Alternative: Use Docker Scanner (no local install needed)

```bash
docker run --rm \
  -e SONAR_HOST_URL="http://localhost:9000" \
  -e SONAR_TOKEN="sqa_your_token_here" \
  -v "$(pwd):/usr/src" \
  sonarsource/sonar-scanner-cli
```

**Visual:**
```
Why DevOps engineers prefer the Docker scanner:
- No local Java/scanner install needed on CI runners
- Same image works across all pipelines (consistency)
- Easy to pin exact scanner version
```

---

## Running Your First Scan

### Step 1: Create `sonar-project.properties` in project root

```properties
sonar.projectKey=my-first-project
sonar.projectName=My First Project
sonar.sources=.
sonar.host.url=http://localhost:9000
sonar.token=sqa_your_token_here
```

### Step 2: Run the Scanner

```bash
sonar-scanner
```

**Visual:**
```
Terminal Output Flow:
┌─────────────────────────────────────┐
│ INFO: Load global settings           │
│ INFO: Load project repositories      │
│ INFO: Analyzing 42 files             │
│ INFO: Sensor JavaSensor [java]       │
│ INFO: Sensor PythonSensor [python]   │
│ INFO: ANALYSIS SUCCESSFUL             │
│ INFO: More about the report          │
│   processing at http://localhost:9000│
│   /dashboard?id=my-first-project     │
└─────────────────────────────────────┘

Scan Pipeline:
Source Code → Scanner (local analysis) → Report → Uploaded to Server
                                                         ↓
                                              Compute Engine processes
                                                         ↓
                                              Dashboard updates
```

### Step 3: View Results

Navigate to the dashboard URL printed in the terminal output.

**Visual:**
```
Project Dashboard:
┌────────────────────────────────────────────┐
│ my-first-project                            │
├────────────────────────────────────────────┤
│  Quality Gate: ✓ PASSED                     │
│                                              │
│  Bugs: 2        Vulnerabilities: 0           │
│  Code Smells: 15   Security Hotspots: 1      │
│  Coverage: 0%   Duplication: 3.2%            │
└────────────────────────────────────────────┘
```

---

## Real-Life DevOps Use Case

**Scenario:** You're onboarding a new microservice repo into your organization's SonarQube instance.

**Typical DevOps steps:**
1. Provision a **project key** in SonarQube (often automated via SonarQube's Web API instead of manual UI clicks, so new repos get scanned automatically).
2. Generate a **project-specific token** (not a personal token) so the CI pipeline authenticates independently of any individual's account.
3. Store the token in the CI/CD secret manager (Jenkins Credentials, GitHub Secrets, Vault, etc.).
4. Add a `sonar-project.properties` file to the repo, or pass equivalent flags directly in the CI pipeline (common in monorepos with many services).
5. Add the scan step to the pipeline **before the deploy stage** so bad code never reaches production.

```bash
# Example: automating project creation via API instead of UI clicks
curl -u admin:admin -X POST \
  "http://localhost:9000/api/projects/create?project=payment-service&name=Payment+Service"
```

**Why automate this?** In organizations with 50+ microservices, manually creating each project and token in the UI doesn't scale. DevOps engineers script this as part of a "new service scaffolding" pipeline.

---

Next: **02quality_gates_and_profiles.md** — the theory behind Quality Gates, Quality Profiles, and Rules, and how to customize them for your organization.