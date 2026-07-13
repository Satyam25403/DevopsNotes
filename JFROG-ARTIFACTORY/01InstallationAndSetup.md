# JFrog Artifactory - Installation & Setup

Getting Artifactory running, logging in for the first time, and creating your first repository.

## Table of Contents
- [Installation Options](#installation-options)
- [Docker Installation](#docker-installation)
- [Docker Compose with PostgreSQL](#docker-compose-with-postgresql)
- [First Login](#first-login)
- [Artifactory UI Layout](#artifactory-ui-layout)
- [Creating Your First Repository](#creating-your-first-repository)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installation Options

**Visual:**
```
┌──────────────────────────────────────────────────┐
│ Edition               Notes                            │
├──────────────────────────────────────────────────┤
│ JFrog Artifactory OSS     Free, basic Maven/Gradle/Ivy     │
│                        support only                          │
│ JFrog Artifactory Pro       Paid, ALL package types             │
│                        (npm, Docker, PyPI, Go, etc.)               │
│ JFrog Cloud (SaaS)             Hosted, no infra to manage             │
└──────────────────────────────────────────────────┘

Installation Methods:
┌─────────────┐   ┌─────────────┐   ┌──────────────────┐
│ Docker           │   │ RPM/DEB Package │   │ Kubernetes (Helm)     │
│ (quickest)         │   │ (traditional VM)   │   │ (production clusters)   │
└─────────────┘   └─────────────┘   └──────────────────┘
```

---

## Docker Installation

```bash
docker run -d --name artifactory \
  -p 8081:8081 -p 8082:8082 \
  releases-docker.jfrog.io/jfrog/artifactory-pro:latest
```

**Visual:**
```
Port Purposes:
┌──────────────────────────────────┐
│  8081  →  Artifactory UI + REST API    │
│  8082  →  Router (used for some            │
│            package-type-specific              │
│            protocols, e.g. Docker)                │
└──────────────────────────────────┘
```

⚠️ **Production note:** the default embedded Derby database is fine for evaluation only. Production deployments point at an external PostgreSQL database:

```bash
docker run -d --name artifactory \
  -p 8081:8081 -p 8082:8082 \
  -e DB_TYPE=postgresql \
  -e DB_URL=jdbc:postgresql://postgres-host:5432/artifactory \
  -e DB_USER=artifactory \
  -e DB_PASSWORD=artifactory_password \
  releases-docker.jfrog.io/jfrog/artifactory-pro:latest
```

---

## Docker Compose with PostgreSQL

```yaml
version: "3"
services:
  artifactory:
    image: releases-docker.jfrog.io/jfrog/artifactory-pro:latest
    ports:
      - "8081:8081"
      - "8082:8082"
    environment:
      DB_TYPE: postgresql
      DB_URL: jdbc:postgresql://postgres:5432/artifactory
      DB_USER: artifactory
      DB_PASSWORD: artifactory_password
    volumes:
      - artifactory_data:/var/opt/jfrog/artifactory
    depends_on:
      - postgres

  postgres:
    image: postgres:14
    environment:
      POSTGRES_DB: artifactory
      POSTGRES_USER: artifactory
      POSTGRES_PASSWORD: artifactory_password
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  artifactory_data:
  postgres_data:
```

```bash
docker-compose up -d
```

**Visual:**
```
Stack Relationship:
┌─────────────┐        ┌──────────────┐
│  Artifactory     │ ─────→ │  PostgreSQL        │
│  (port 8081)       │        │  (metadata/config)    │
└─────────────┘        └──────────────┘
        │
        ↓ actual binary storage
┌─────────────────────────────────┐
│  Filesystem (or S3/GCS in production) │
└─────────────────────────────────────┘

Note: binary CONTENT and METADATA are stored
separately — PostgreSQL holds config/metadata,
while actual artifact bytes live on a filesystem
or object storage backend (configurable, covered
further in file 05).
```

---

## First Login

Navigate to `http://localhost:8081`

**Default credentials:** `admin` / `password` (forced change on first login)

**Visual:**
```
Setup Wizard Flow (first login):
┌───────────────────────────────┐
│  1. Change admin password           │
│  2. Accept license (Pro/Enterprise)     │
│  3. Configure base URL                    │
│  4. Select initial repository               │
│     templates to create (optional)             │
└───────────────────────────────┘
         ↓
   Main Artifactory Dashboard
```

---

## Artifactory UI Layout

**Visual:**
```
┌────────────────────────────────────────────────────────┐
│  ☰   [Search artifacts...]                    🔔  ⚙️  👤   │
├──────────┬─────────────────────────────────────────────┤
│  Application  │                                                │
│  Artifacts       │             Main Content Area                     │
│  Builds            │        (repository browser, artifact              │
│  Xray                │         details, build info, etc. —                 │
│  Administration       │         changes based on left nav selection)          │
│  (Left Nav)               │                                                        │
└──────────┴─────────────────────────────────────────────┘
```

**Key sections:**
```
Artifacts        → browse repositories and their contents
Builds            → view Build Info for tracked CI builds
Xray                → security/license scanning results (if enabled)
Administration        → repositories, users, permissions, replication
```

---

## Creating Your First Repository

```
Administration → Repositories → Repositories → Add Repositories
```

**Creating a local repository (for your own npm packages):**
```
Package Type: npm
Repository Type: Local
Repository Key: npm-local
```

**Creating a remote repository (caching npmjs.com):**
```
Package Type: npm
Repository Type: Remote
Repository Key: npm-remote
URL: https://registry.npmjs.org
```

**Creating a virtual repository (unifying both):**
```
Package Type: npm
Repository Type: Virtual
Repository Key: npm-all
Included Repositories: npm-local, npm-remote
```

**Visual:**
```
Resulting Setup:
┌─────────────────────────────────┐
│         npm-all (virtual)              │  ← developers point npm HERE
│  ┌──────────┐  ┌──────────┐  │
│  │ npm-local     │  │ npm-remote     │  │
│  │ (your own       │  │ (caches            │  │
│  │  packages)         │  │  npmjs.com)          │  │
│  └──────────┘  └──────────┘  │
└─────────────────────────────────┘
```

**Configuring npm to use it:**
```bash
npm config set registry https://artifactory.internal:8081/artifactory/api/npm/npm-all/
```

**Visual:**
```
After this config change:
npm install express
        ↓
Request goes to npm-all (virtual) → checks npm-local
first (not found, it's a public package) → checks
npm-remote → not cached yet → fetches from npmjs.com,
CACHES it → serves to the developer

Next developer's npm install express:
        ↓
Request goes to npm-all → npm-remote already has it
CACHED → served instantly, no trip to the public
internet needed at all
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is onboarding Artifactory for a company that currently has all 15 development teams pulling packages directly from public registries with no central control.

**What they do:**
1. Deploys Artifactory via **Docker Compose backed by an external PostgreSQL database** (not the embedded default), ensuring metadata survives container restarts/redeployments independently of the container lifecycle.
2. Creates **remote repositories** for every package ecosystem in active use across the 15 teams: npm, Docker Hub, PyPI, Maven Central, and Go modules — covering the full range without needing per-team custom setups.
3. Creates matching **virtual repositories** for each, giving every team ONE consistent internal URL pattern (`artifactory.internal/artifactory/api/<type>/<type>-all/`) regardless of which specific remote/local repos are aggregated behind it.
4. Rolls out the registry URL change via a **shared configuration snippet** (`.npmrc`, `pip.conf`, `settings.xml` templates) distributed to all 15 teams, rather than leaving each team to configure it slightly differently and inconsistently.
5. Within the first week, confirms via Artifactory's cache hit metrics that the vast majority of package requests are now served from cache — meaningfully reducing both build time and exposure to public registry outages.

**Why this matters:** The biggest early win from adopting Artifactory is almost always this exact scenario — decoupling internal build reliability from the availability and rate limits of external public package registries, achieved with a relatively simple one-time configuration change per team.

---

Next: **02repository_types_and_concepts.md** — the theory behind local/remote/virtual repositories, repository layouts, and permission targets.