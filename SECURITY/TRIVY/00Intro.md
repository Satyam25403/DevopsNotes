# Trivy - Introduction & Architecture

Understanding what Trivy is, the range of security problems it catches, and how it fits into a DevSecOps pipeline.

## Table of Contents
- [What is Trivy](#what-is-trivy)
- [The Scope Problem Trivy Solves](#the-scope-problem-trivy-solves)
- [Why DevOps Teams Use It](#why-devops-teams-use-it)
- [What Trivy Can Scan](#what-trivy-can-scan)
- [Core Concepts Overview](#core-concepts-overview)
- [Architecture](#architecture)
- [Trivy vs Alternatives](#trivy-vs-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## What is Trivy

**Trivy is an open-source, all-in-one security scanner built by Aqua Security, covering vulnerabilities, misconfigurations, secrets, and licenses across containers, code, and infrastructure.**

**Visual:**
```
Input                              Trivy                          Output
┌──────────────────┐             ┌──────────────┐          ┌─────────────────┐
│ Docker image           │             │                    │          │  Vulnerabilities: 12   │
│ Dockerfile               │  ────────→  │  Trivy Scanner       │  ───────→│  Misconfigs: 3           │
│ Terraform files             │             │  (single binary,     │          │  Secrets found: 1           │
│ Kubernetes manifests           │             │   no server needed     │          │  License issues: 0            │
│ Git repository                    │             │   for basic scans)       │          └─────────────────┘
│ SBOM file                            │             └──────────────┘
└──────────────────┘
```

---

## The Scope Problem Trivy Solves

**Visual:**
```
Before an all-in-one scanner, DevOps/security teams typically needed:
┌──────────────────────────────────────────────┐
│  Tool for container image CVEs      (e.g. Clair)  │
│  Tool for IaC misconfigurations        (e.g. tfsec)  │
│  Tool for secret detection                (e.g. gitleaks)│
│  Tool for license compliance                 (custom scripts)│
│  Tool for SBOM generation                       (yet another tool)│
└──────────────────────────────────────────────┘
→ 5 different tools, 5 different configs, 5 different
  outputs to correlate manually.

Trivy's Pitch:
ONE binary, ONE command, covers ALL of the above.
```

---

## Why DevOps Teams Use It

**Visual:**
```
Problem                                How Trivy Helps
──────────────────────────────────────────────────────────────────
Vulnerable base images shipped to prod    Scans OS packages + language
                                        dependencies for known CVEs
Misconfigured Terraform/K8s YAML             Catches insecure defaults before
(e.g. public S3 buckets, privileged pods)    they're ever applied
Hardcoded secrets committed to Git             Scans for API keys, passwords,
                                        tokens in code and image layers
No visibility into what's actually IN         Generates SBOMs (Software Bill
a container image                          of Materials) for compliance/audit
Manual, inconsistent security review           Single command integrates into
before every release                        CI/CD as an automated gate
```

---

## What Trivy Can Scan

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│               Trivy Scan Targets                         │
├─────────────────────────────────────────────────────┤
│                                                           │
│  Container Image     →  OS packages + app dependencies       │
│                        (npm, pip, maven, go modules, etc.)     │
│                                                           │
│  Filesystem            →  Local directory/project source code   │
│                                                           │
│  Git Repository          →  Remote repo, without cloning manually   │
│                                                           │
│  Kubernetes                →  Live cluster resources                    │
│                                                           │
│  IaC (Terraform, CloudFormation, →  Misconfigurations before deploy       │
│   Kubernetes YAML, Dockerfile)                                              │
│                                                           │
│  SBOM                       →  Scan an existing Software Bill of              │
│                            Materials file for vulnerabilities                     │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Core Concepts Overview

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                     Trivy Concepts                         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Vulnerability (CVE)  →  A known security flaw in a            │
│                        specific package/version                   │
│                                                           │
│  Severity              →  CRITICAL / HIGH / MEDIUM / LOW /       │
│                        UNKNOWN — how serious the issue is           │
│                                                           │
│  Misconfiguration        →  An insecure setting in IaC/config          │
│                        files (not a CVE, a policy violation)              │
│                                                           │
│  Secret                    →  Hardcoded credentials/keys found              │
│                        in code or image layers                                 │
│                                                           │
│  SBOM                        →  Software Bill of Materials — a full            │
│                        inventory of every package/library                          │
│                        in an image or project                                        │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Architecture

**Visual:**
```
                     ┌───────────────────────────┐
                     │         Trivy CLI               │
                     │  (single Go binary, no          │
                     │   runtime dependencies)           │
                     └─────────────┬─────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ↓                    ↓                     ↓
     ┌─────────────┐    ┌──────────────────┐   ┌──────────────┐
     │  Vulnerability   │    │  Misconfiguration      │   │  Secret          │
     │  DB (downloaded    │    │  Policies (Rego/OPA-       │   │  Detection          │
     │  periodically,       │    │  based rules, built-in)      │   │  (regex patterns)     │
     │  cached locally)        │    └──────────────────┘   └──────────────┘
     └─────────────┘
              ↓
     ┌─────────────────────────────────────┐
     │       Scan Target (image/repo/IaC/etc.)  │
     └─────────────────────────────────────┘
              ↓
     ┌─────────────────────────────────────┐
     │      Report (table, JSON, SARIF, etc.)   │
     └─────────────────────────────────────┘
```

**Flow in words:**
1. Trivy downloads and caches a **vulnerability database** (aggregated from NVD, vendor advisories, GitHub Security Advisories, and more).
2. When scanning, it inventories every package/dependency in the target (image layers, lockfiles, etc.).
3. It cross-references that inventory against the vulnerability database for known CVEs.
4. Separately, it runs built-in **policy rules** against IaC files for misconfigurations, and **regex-based detection** for secrets.
5. Results are output in your chosen format — human-readable table, JSON, or SARIF (for GitHub code scanning integration).

---

## Trivy vs Alternatives

**Visual:**
```
Tool                Focus                          Notes
─────────────────────────────────────────────────────────────────────
Trivy                 All-in-one (vuln + IaC +        Free, fast, single binary,
                     secrets + SBOM + license)         broadest scope
Clair                  Container image CVEs only         Needs a running server,
                                                        narrower scope
tfsec/Checkov            IaC misconfiguration only          Deep Terraform-specific
                                                        checks, narrower scope
Snyk                     Vuln + IaC + code (SaaS)            Commercial, strong SaaS UX,
                                                        deeper remediation guidance
Grype                     Container image CVEs only            Similar to Trivy's image
                                                        scanning, narrower scope

Trivy's niche: broadest single-tool coverage,
completely free, and a single binary that works
identically in a laptop terminal, CI pipeline, or
Kubernetes admission controller.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is asked to add basic security scanning to a company's CI/CD pipelines for the first time, across 20 microservices with a mix of Node.js, Python, and Go, each with their own Dockerfile and Kubernetes manifests.

**Without Trivy:**
- No consistent way to know if a shipped container image has known critical vulnerabilities.
- Terraform changes occasionally introduce publicly-accessible storage buckets that nobody catches until a security audit finds them months later.
- Developers occasionally commit API keys to Git by accident, discovered only when a service starts behaving oddly.

**What the DevOps engineer does:**
1. Adds a **Trivy image scan step** to every pipeline immediately after the Docker build stage, failing the build on any CRITICAL or HIGH severity vulnerability.
2. Adds a **Trivy config scan step** against each repo's Terraform and Kubernetes YAML files, catching issues like overly permissive IAM policies or containers running as root before they're ever applied.
3. Adds a **Trivy secret scan** as a pre-commit hook AND a CI step (defense in depth), catching accidentally committed credentials before they reach a shared branch.
4. Generates and stores an **SBOM** for every release build, satisfying an upcoming customer security questionnaire asking "what's actually inside your container images."

**Result:** One tool, one consistent command syntax, covers vulnerability scanning, IaC security, secret detection, and compliance documentation — across all 20 services, regardless of language or IaC tool used.

---

Next: **01installation_and_setup.md** — installing Trivy and running your first vulnerability scan.