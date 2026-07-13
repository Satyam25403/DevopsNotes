# Trivy - Installation & Setup

Getting Trivy installed and running your very first vulnerability scan.

## Table of Contents
- [Installation Options](#installation-options)
- [Verifying Installation](#verifying-installation)
- [The Vulnerability Database](#the-vulnerability-database)
- [Your First Image Scan](#your-first-image-scan)
- [Understanding the Output](#understanding-the-output)
- [Basic Output Formats](#basic-output-formats)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Installation Options

### Linux (apt/dnf via install script)

```bash
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
```

### Mac (Homebrew)

```bash
brew install trivy
```

### Docker (no local install needed)

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image nginx:latest
```

**Visual:**
```
┌──────────────────────────────────────────────┐
│ Method                Best For                     │
├──────────────────────────────────────────────┤
│ Install script          Local dev machines             │
│ Homebrew                  Mac local dev                  │
│ Docker                      CI/CD pipelines, zero local     │
│                            dependency management               │
│ Binary download                Air-gapped/offline environments   │
└──────────────────────────────────────────────┘
```

---

## Verifying Installation

```bash
trivy --version
```

**Visual:**
```
Output:
Version: 0.50.1
Vulnerability DB:
  Version: 2
  UpdatedAt: 2026-07-12T06:00:00Z
```

---

## The Vulnerability Database

**Before its first scan, Trivy downloads a vulnerability database — this happens automatically.**

**Visual:**
```
First Run:
┌─────────────────────────────────────────┐
│  Trivy checks: is the local DB present       │
│  and up to date?                                │
│         ↓ No                                       │
│  Downloads DB from GitHub Container Registry     │
│  (aggregated from NVD, GHSA, vendor advisories)     │
│         ↓                                              │
│  Caches locally at ~/.cache/trivy/               │
│         ↓                                              │
│  Proceeds with the actual scan                       │
└─────────────────────────────────────────┘

Subsequent runs: only re-downloads if the cached
DB is older than the configured refresh interval
(default: checks for updates on every run, but
only downloads if stale).
```

**Manually updating the DB:**
```bash
trivy image --download-db-only
```

**Skipping the DB update (useful for air-gapped/offline scans, or faster repeated local runs):**
```bash
trivy image --skip-db-update nginx:latest
```

---

## Your First Image Scan

```bash
trivy image nginx:latest
```

**Visual:**
```
Scan Flow:
┌────────────┐   ┌──────────────┐   ┌───────────────┐
│ Pull/inspect  │ → │ Extract package │ → │ Cross-reference │
│ image layers    │   │ inventory (OS +  │   │ against          │
│                  │   │ language deps)     │   │ vulnerability DB   │
└────────────┘   └──────────────┘   └───────────────┘
                                              ↓
                                    Table of findings printed
```

---

## Understanding the Output

**Visual:**
```
Terminal Output:
┌────────────────────────────────────────────────────┐
│ nginx:latest (debian 12.4)                              │
│                                                          │
│ Total: 15 (UNKNOWN: 0, LOW: 6, MEDIUM: 5, HIGH: 3,        │
│            CRITICAL: 1)                                     │
│                                                          │
│ ┌──────────┬───────────────┬──────────┬─────────┐  │
│ │ Library     │ Vulnerability     │ Severity   │ Fixed     │  │
│ ├──────────┼───────────────┼──────────┼─────────┤  │
│ │ openssl       │ CVE-2024-1234       │ CRITICAL     │ 3.0.13-1   │  │
│ │ libxml2         │ CVE-2024-5678         │ HIGH           │ 2.9.14-1     │  │
│ │ curl              │ CVE-2024-9012           │ MEDIUM           │ 8.5.0-1        │  │
│ └──────────┴───────────────┴──────────┴─────────┘  │
└────────────────────────────────────────────────────┘
```

**Reading this table:**
```
Library       → the specific package containing the vulnerability
Vulnerability → the CVE identifier (searchable for full details)
Severity       → CRITICAL/HIGH/MEDIUM/LOW/UNKNOWN
Fixed           → the version that RESOLVES this vulnerability
                  (if blank, no fix is available yet)
```

**Visual — the "Fixed" column drives action:**
```
If "Fixed" version exists:
→ Actionable: update the base image or dependency to that version

If "Fixed" column is empty:
→ No patch available yet from the upstream maintainer;
  track it, but blocking a pipeline entirely on this
  may not be productive — assess actual exploitability
```

---

## Basic Output Formats

```bash
# Human-readable table (default)
trivy image nginx:latest

# JSON (for programmatic processing)
trivy image -f json -o results.json nginx:latest

# SARIF (for GitHub code scanning integration)
trivy image -f sarif -o results.sarif nginx:latest
```

**Visual:**
```
┌──────────────┬───────────────────────────────────┐
│ Format          │ Use Case                                │
├──────────────┼───────────────────────────────────┤
│ table (default)   │ Human reading in a terminal                │
│ json                │ Parsing in scripts, feeding dashboards       │
│ sarif                │ GitHub Security tab integration                  │
│ cyclonedx/spdx          │ SBOM formats (covered in file 05)                  │
└──────────────┴───────────────────────────────────┘
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is manually validating a new base image candidate before recommending the team standardize on it.

**What they do:**
1. Runs `trivy image --severity CRITICAL,HIGH candidate-base:latest` to quickly see only the vulnerabilities that would actually matter for a go/no-go decision, filtering out LOW/MEDIUM noise for this initial pass.
2. Compares the result against the **current base image** the team already uses, running the same command against it — establishing a baseline for whether the new candidate is actually an improvement, not just "has some vulnerabilities" (nearly every image does).
3. Exports the comparison as **JSON** for both images and writes a short script to diff the CVE lists, presenting the team with "here are 3 new critical CVEs introduced, and 8 resolved" rather than two overwhelming raw tables.
4. Confirms via the **Fixed** column that the 3 new critical CVEs actually have available fixes by bumping one further dependency version, and validates the fix works before finalizing the recommendation.

**Why this matters:** A raw "here are 50 vulnerabilities" scan result is overwhelming and not actionable on its own — filtering by severity, comparing against a baseline, and checking fix availability is what turns a scan into an actual decision.

---

Next: **02vulnerability_scanning_practical.md** — deep practical usage: severity filtering, ignoring false positives, exit codes for pipeline gating, and scanning filesystems/repos.