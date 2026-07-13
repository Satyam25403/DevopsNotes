# Trivy - Advanced Features & Real-World Use Cases

Going beyond ad-hoc scanning: SBOM generation, client/server mode for scaling, the Kubernetes Operator, registry-wide scanning, and how mature organizations run Trivy continuously.

## Table of Contents
- [SBOM Generation](#sbom-generation)
- [Scanning an Existing SBOM](#scanning-an-existing-sbom)
- [Trivy Server/Client Mode](#trivy-serverclient-mode)
- [Trivy Operator for Kubernetes](#trivy-operator-for-kubernetes)
- [Registry-Wide Scanning](#registry-wide-scanning)
- [VEX (Vulnerability Exploitability eXchange)](#vex-vulnerability-exploitability-exchange)
- [Common Pitfalls & War Stories](#common-pitfalls--war-stories)
- [Real-Life DevOps Use Case (End-to-End)](#real-life-devops-use-case-end-to-end)

---

## SBOM Generation

**An SBOM (Software Bill of Materials) is a complete inventory of every package/library in an image or project — increasingly required for compliance and supply-chain security.**

```bash
trivy image --format cyclonedx --output sbom.json myapp:latest
```

**Visual:**
```
Why SBOMs matter now more than ever:
┌─────────────────────────────────────────────┐
│  Scenario: A new critical CVE is announced       │
│  in a popular library (e.g. another Log4Shell)      │
│                                                    │
│  Without an SBOM:                                     │
│  "Do we even USE this library anywhere?              │
│   Which of our 200 services? We don't know               │
│   without manually checking every single repo."             │
│                                                    │
│  With SBOMs generated for every release:                       │
│  Search all stored SBOMs for the library name →                   │
│  Instantly identify EXACTLY which services are                       │
│  affected, in minutes instead of days                                   │
└─────────────────────────────────────────────┘
```

**Supported SBOM formats:**
```
┌──────────────┬───────────────────────────────┐
│ CycloneDX        │ Widely adopted, good tooling         │
│                  │ ecosystem support                     │
│ SPDX               │ Linux Foundation standard, common       │
│                  │ in compliance/legal contexts               │
└──────────────┴───────────────────────────────┘
```

```bash
trivy image --format spdx-json --output sbom.spdx.json myapp:latest
```

---

## Scanning an Existing SBOM

**Once you have an SBOM, you can re-scan it later without needing the original image — useful for periodic re-checks as the vulnerability database updates.**

```bash
trivy sbom sbom.json
```

**Visual:**
```
Why this is powerful:
Image built 6 months ago → SBOM generated and archived
   at build time → new CVE disclosed today affecting a
   library from 6 months ago →
   trivy sbom old-sbom.json
   → Instantly know if THAT specific old release is
     affected, without needing to still have the
     original image available to re-scan directly.
```

---

## Trivy Server/Client Mode

**For large organizations running many scans, a shared server avoids every single CI job re-downloading and maintaining its own vulnerability database copy.**

**Visual:**
```
Without Server Mode:
┌──────────┐  ┌──────────┐  ┌──────────┐
│ CI Job 1     │  │ CI Job 2     │  │ CI Job 3     │
│ own DB copy   │  │ own DB copy   │  │ own DB copy   │
└──────────┘  └──────────┘  └──────────┘
→ Redundant downloads, redundant storage,
  DB might be slightly out of sync between jobs

With Server Mode:
        ┌─────────────────┐
        │   Trivy Server        │  ← ONE shared vulnerability DB,
        │  (central instance)     │     always up to date
        └────────┬────────────┘
                  │
    ┌─────────────┼─────────────┐
    ↓             ↓              ↓
┌────────┐   ┌────────┐    ┌────────┐
│ CI Job 1   │   │ CI Job 2   │    │ CI Job 3   │
│ (client)     │   │ (client)     │    │ (client)     │
└────────┘   └────────┘    └────────┘
```

**Starting the server:**
```bash
trivy server --listen 0.0.0.0:4954
```

**Client usage (in CI):**
```bash
trivy image --server http://trivy-server.internal:4954 myapp:latest
```

**Visual — benefits at scale:**
```
- Faster individual scans (no per-job DB download)
- Guaranteed consistency (every job scans against
  the EXACT same DB version at any given moment)
- Lower total bandwidth/storage cost across the org
```

---

## Trivy Operator for Kubernetes

**Instead of manually running `trivy k8s` scans, the Operator continuously and automatically scans everything running in the cluster.**

**Visual:**
```
┌─────────────────────────────────────────────┐
│              Kubernetes Cluster                    │
│  ┌───────────────────────────────────────┐  │
│  │      Trivy Operator (runs as a Deployment) │  │
│  │  Watches for new/changed Pods,                │  │
│  │  Deployments, ConfigMaps, etc.                     │  │
│  └──────────────────┬─────────────────────┘  │
│                     │ automatically triggers          │
│                     ↓ scans on changes                  │
│  ┌───────────────────────────────────────┐  │
│  │  VulnerabilityReport (Custom Resource)      │  │
│  │  ConfigAuditReport (Custom Resource)             │  │
│  │  ExposedSecretReport (Custom Resource)                │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
```

```bash
kubectl get vulnerabilityreports -A
```

**Visual:**
```
Why this matters:
CI/CD-only scanning misses anything deployed OUTSIDE
the pipeline (manual kubectl apply, Helm installs from
a laptop, etc.) — the Operator catches EVERYTHING
actually running in the cluster continuously, not just
what went through your specific pipeline.

Reports are queryable natively via kubectl, and can
feed into Grafana dashboards (querying the Kubernetes
API) for an always-current security posture view.
```

---

## Registry-Wide Scanning

**Beyond individual images, scan an entire container registry to audit everything stored there.**

```bash
trivy image --list-all-pkgs registry.company.com/myapp:latest
```

For scanning many images across a registry, this is typically scripted:
```bash
#!/bin/bash
for image in $(curl -s https://registry.company.com/v2/_catalog | jq -r '.repositories[]'); do
  trivy image --severity CRITICAL --format json --output "reports/${image//\//_}.json" "registry.company.com/${image}:latest"
done
```

**Visual:**
```
Why this matters:
Individual CI pipelines only scan images AS they're built —
but registries accumulate OLDER images too (previous
releases still deployed somewhere, images built before
scanning was even introduced). A periodic registry-wide
sweep catches vulnerabilities discovered in ALREADY-STORED
images that CI-time scanning alone would never re-check.
```

---

## VEX (Vulnerability Exploitability eXchange)

**A newer standard for expressing "yes, this CVE technically applies to a package we use, but it's NOT actually exploitable in our specific context" — more structured than a plain `.trivyignore`.**

**Visual:**
```
Problem VEX solves:
CVE-2024-9999 exists in a logging library your app uses,
BUT the vulnerable CODE PATH is never actually called
by your application's usage pattern — it's a REAL CVE,
but NOT exploitable for you specifically.

Without VEX: still shows as a blocking finding,
             requiring a blunt .trivyignore suppression

With VEX: a structured, standardized document explains
          WHY it's not exploitable, machine-readable,
          shareable with auditors/customers as evidence
          of a considered risk decision, not just silence
```

```bash
trivy image --vex my-vex-document.json myapp:latest
```

**Visual:**
```
VEX is increasingly requested by enterprise customers
and regulators as evidence of a MATURE vulnerability
management process — showing not just "we ignored this"
but "we formally assessed this and documented why it's
not exploitable in our context."
```

---

## Common Pitfalls & War Stories

**Visual:**
```
Pitfall 1: "Scans take forever in CI"
Cause: Re-downloading the vulnerability DB on every run
Fix: Cache the DB (file 04) or use server/client mode

Pitfall 2: "Team ignores scan failures entirely"
Cause: Scanning rolled out too strictly on day one,
       blocking on years of legacy vulnerability debt
Fix: Tiered rollout — block only on CRITICAL initially,
     tighten gradually as backlog is addressed

Pitfall 3: ".trivyignore has 200 entries, nobody remembers why"
Cause: Ignoring findings without comments/tickets/expiry
Fix: Mandate comments + expiry dates + PR review for
     every ignore entry

Pitfall 4: "CI scan passes, but the running cluster has
           known-vulnerable images deployed"
Cause: Only scanning at build time, nothing scans what's
       ACTUALLY running (manual deploys, drift, old images
       predating scanning adoption)
Fix: Deploy the Trivy Operator for continuous, live
     cluster scanning independent of the CI pipeline

Pitfall 5: "Can't answer 'are we affected by this new CVE'
           quickly during a fire-drill"
Cause: No SBOMs generated/archived historically
Fix: Generate and store an SBOM for every release,
     enabling fast trivy sbom re-checks later
```

---

## Real-Life DevOps Use Case (End-to-End)

**Scenario:** A company handling regulated financial data needs a mature, auditable vulnerability management program — not just "we run a scanner sometimes."

**Full workflow the DevOps/security team builds:**

1. **Shift-left scanning:** Every PR runs `trivy fs`, `trivy config`, and secret scanning via a shared reusable CI workflow, blocking on CRITICAL/HIGH findings with a tiered, gradually-tightening policy for legacy repos.
2. **Build-time image scanning:** Every container build runs `trivy image` against a **centralized Trivy server**, ensuring consistent vulnerability DB versions across all 50+ pipelines and avoiding redundant per-job downloads.
3. **SBOM archival:** Every release generates a **CycloneDX SBOM**, stored alongside build artifacts — when a major CVE hits the news, the security team searches archived SBOMs across all releases to answer "are we affected, and where" within minutes.
4. **Continuous cluster monitoring:** The **Trivy Operator** runs in every Kubernetes cluster, continuously scanning live workloads and catching configuration drift or manually-deployed images that bypassed CI entirely.
5. **Periodic registry sweep:** A weekly scheduled job scans the **entire container registry**, catching newly-disclosed vulnerabilities in older, already-deployed image versions that wouldn't trigger a new CI run on their own.
6. **VEX documentation:** For CVEs judged not exploitable in their specific context, the team produces a **VEX document** rather than a silent `.trivyignore` entry — giving auditors and enterprise customers structured, defensible evidence of the risk assessment process.
7. **Governed suppression process:** Any `.trivyignore` entry requires security team PR approval, a linked ticket, and an expiry date — preventing an ever-growing, unreviewed pile of forgotten risk acceptances.

**Why this is "real DevOps," not just running a tool:** Trivy here isn't just "a scanner in the pipeline" — it's woven into source control (PR gates), build systems (server mode), runtime (Operator), registry hygiene (periodic sweeps), and compliance documentation (SBOM/VEX). This is the difference between "we added a security scan" and "we have a continuously verified, auditable vulnerability management program."

---

This completes the Trivy note series: **Introduction → Setup → Vulnerability Scanning → IaC/Misconfig Scanning → CI/CD Integration → Advanced/Real-World Usage.**