# JFrog Artifactory - Introduction & Architecture

Understanding what Artifactory is, the binary management problem it solves, and how it fits at the center of a modern software supply chain.

## Table of Contents
- [What is Artifactory](#what-is-artifactory)
- [The Binary Management Problem](#the-binary-management-problem)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [The Software Supply Chain View](#the-software-supply-chain-view)
- [Artifactory vs Alternatives](#artifactory-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is Artifactory

**JFrog Artifactory is a universal binary repository manager** — a central place to store, version, and distribute build artifacts, packages, and container images across every language and package format an organization uses.

**Visual:**
```
Without Artifactory                     With Artifactory
┌─────────────────┐                   ┌──────────────────────┐
│ npm packages           │                   │      JFrog Artifactory       │
│  → npmjs.com directly       │                   │  ┌──────────────────┐  │
│                             │   ───────────→    │  │  npm    Docker           │  │
│ Docker images                  │                   │  │  Maven  PyPI               │  │
│  → Docker Hub directly            │                   │  │  Go     NuGet                 │  │
│                                       │                   │  │  Helm   Generic files             │  │
│ Maven artifacts                          │                   │  └──────────────────┘  │
│  → Maven Central directly                   │                   │  (one central system for            │
│                                                 │                   │   every artifact type)                 │
│ Every team pulls DIRECTLY                          │                   └──────────────────────┘
│ from public sources, every time                        │
└─────────────────┘
```

---

## The Binary Management Problem

**Visual:**
```
Problems with pulling directly from public registries
(npm, Docker Hub, PyPI, Maven Central) every time:

┌─────────────────────────────────────────────┐
│  1. Public registry has an outage               │
│     → EVERY build across the company fails           │
│       simultaneously, even though nothing               │
│       about YOUR code changed                              │
│                                                    │
│  2. A package gets YANKED/deleted upstream            │
│     → builds that depended on it suddenly                │
│       break, with no local copy to fall back on              │
│                                                    │
│  3. No control over WHAT versions/packages              │
│     teams are allowed to pull                              │
│     → a developer accidentally pulls a                        │
│       compromised/malicious package version                     │
│                                                    │
│  4. No single place to store YOUR OWN                       │
│     build outputs (internal libraries, Docker                  │
│     images, release artifacts)                                    │
└─────────────────────────────────────────────┘
```

**Artifactory's answer:** sit BETWEEN your builds and the public internet, caching what you use, storing what you produce, and giving you control over both.

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                                How Artifactory Helps
──────────────────────────────────────────────────────────────────
Public registry outages break builds     Remote repos CACHE packages locally,
                                        builds keep working even if upstream is down
No central place for internal packages    Local repos store your own build
(internal libraries, Docker images)        outputs, versioned and organized
Different teams use different registry     Virtual repos give ONE unified URL
URLs, inconsistent configuration            regardless of underlying repo type
No traceability of what's actually          Build Info captures exactly which
in a deployed artifact                    dependencies/versions went into a build
Manual, error-prone promotion between       Promotion moves a SPECIFIC, already-
dev/staging/production artifact stores       tested artifact through stages, not
                                        a risky rebuild at each stage
```

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                   Artifactory Concepts                     │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Local Repository     →  Stores artifacts YOU produce           │
│                        (your own builds, releases)                 │
│                                                           │
│  Remote Repository       →  A CACHING PROXY in front of a             │
│                        public registry (npm, Docker Hub, etc.)          │
│                                                           │
│  Virtual Repository        →  A single URL that AGGREGATES                  │
│                        multiple local + remote repos together              │
│                                                           │
│  Build Info                  →  Metadata capturing exactly what                  │
│                        went into a specific build (dependencies,               │
│                        environment, source commit)                                │
│                                                           │
│  Promotion                     →  Moving an artifact between                        │
│                        repositories representing lifecycle                            │
│                        stages (dev → staging → prod)                                    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
┌──────────┐  requests package  ┌─────────────────────────┐
│  Build/CI     │ ─────────────────────→ │        Virtual Repository        │
│  (npm install,  │                         │    "npm-all" (one URL)              │
│   docker pull)    │                         └──────────────┬──────────────────┘
└──────────┘                                          │
                                    ┌───────────────────┼───────────────────┐
                                    ↓                    ↓                    ↓
                          ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
                          │  Local Repo         │    │  Remote Repo         │    │  Remote Repo         │
                          │  "npm-local"           │    │  "npm-remote"           │    │  "npm-other-org"        │
                          │  (your own                │    │  (caches npmjs.com)       │    │  (caches a partner's        │
                          │   published packages)        │    │                             │    │   private registry)             │
                          └──────────────┘    └──────┬───────┘    └──────────────┘
                                                       │  cache miss → fetch from
                                                       ↓  upstream, cache locally
                                             ┌──────────────────┐
                                             │   npmjs.com (public)    │
                                             └──────────────────┘
```

**Flow in words:**
1. A build tool (npm, Docker, Maven, pip) points at ONE **virtual repository** URL.
2. The virtual repo checks its aggregated **local** and **remote** repos in order for the requested package.
3. If found in a **local repo** (your own artifact), it's served directly.
4. If it needs to come from a **remote repo** (e.g., npmjs.com) and isn't already cached, Artifactory fetches it from upstream ONCE, caches it locally, and serves it — subsequent requests hit the cache, not the public internet.

---

## The Software Supply Chain View

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│              End-to-End Artifact Lifecycle                │
├─────────────────────────────────────────────────────┤
│                                                           │
│  Source Code (Git)                                            │
│         ↓ build                                                  │
│  Build produces an artifact (jar, Docker image, npm package)      │
│         ↓ publish                                                    │
│  Artifactory: local repo "myapp-dev-local"                             │
│         ↓ tested, passes CI                                              │
│  PROMOTED to "myapp-staging-local" (same artifact, no rebuild)              │
│         ↓ passes staging validation                                          │
│  PROMOTED to "myapp-release-local"                                              │
│         ↓ deployed                                                                │
│  Production                                                                          │
│                                                           │
└─────────────────────────────────────────────────────────┘

Critical principle: the artifact that reaches production
is the EXACT SAME BINARY that was tested in staging —
promotion moves metadata/location, it never rebuilds.
This eliminates "it worked in staging but broke in prod
because the build was slightly different" class of bugs.
```

---

## Artifactory vs Alternatives

**Visual:**
```
Tool                     Focus                          Notes
─────────────────────────────────────────────────────────────────────
JFrog Artifactory           Universal, all package types      Broadest format support,
                                                             mature promotion/enterprise features
Nexus Repository               Universal, similar scope             Strong open-source tier,
                                                             similar core capability set
GitHub Packages                  GitHub-native, multi-format            Simple, tightly tied to
                                                             GitHub ecosystem, less enterprise depth
Docker Hub / ECR / GCR              Container images ONLY                  Single-format, cloud-native
                                                             registries, not universal
npm registry / PyPI                   Single-format public registries         Not for private/internal
                                                             artifact management

Artifactory's niche: the broadest package format support
(30+ types), mature enterprise features (replication, HA,
fine-grained promotion), and deep integration with the
rest of the JFrog platform (Xray for security scanning,
Pipelines for CI/CD).
```

---

## Real-Life DevOps Use Case

**Scenario:** A company with 40 microservices in Java, Node.js, Python, and Go is experiencing frequent build failures whenever npmjs.com or Docker Hub has intermittent outages, plus has no consistent way to promote a tested build to production without rebuilding it.

**Without Artifactory:**
- Every one of 40 services' CI pipelines pulls dependencies directly from public registries — a single public registry hiccup causes company-wide build failures.
- "Promotion" means re-running the build in a different environment, occasionally producing a SLIGHTLY different artifact (different dependency resolution, different timestamp) than what was actually tested.
- No visibility into which specific package versions are used across the whole organization when a critical CVE is announced.

**What the DevOps engineer does:**
1. Deploys **Artifactory**, setting up remote repositories caching npm, Docker Hub, PyPI, and Maven Central.
2. Creates **virtual repositories** so each team's existing build tool configuration only needs a URL change — no other tooling changes required.
3. Sets up **local repositories** for each team's own build outputs (JARs, Docker images, npm packages), replacing ad-hoc internal file shares.
4. Implements a **promotion pipeline**: builds land in a `-dev-local` repo, and only after passing automated tests get promoted (not rebuilt) into `-staging-local`, then `-release-local`.
5. Result: public registry outages no longer break builds (cached packages keep working), and the exact same tested binary reaches production every time.

**Result:** Artifactory becomes the organization's single, resilient source of truth for every artifact type, decoupling internal build reliability from the availability of external public registries.

---

Next: **01installation_and_setup.md** — installing Artifactory, first login, and setting up your first repository.