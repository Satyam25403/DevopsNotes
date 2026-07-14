# SonarQube - Quality Gates, Quality Profiles & Rules

The core theory that makes SonarQube useful as an automated gatekeeper, not just a reporting dashboard.

## Table of Contents
- [Issue Types](#issue-types)
- [Severity Levels](#severity-levels)
- [Quality Profiles](#quality-profiles)
- [Rules](#rules)
- [Quality Gates](#quality-gates)
- [New Code vs Overall Code](#new-code-vs-overall-code)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Issue Types

SonarQube classifies every problem it finds into one of these categories:

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│                     Issue Types                          │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  🐛 Bug              → Code that is demonstrably wrong    │
│                        (e.g. null pointer dereference)     │
│                                                           │
│  🔓 Vulnerability     → Exploitable security weakness      │
│                        (e.g. SQL injection risk)            │
│                                                           │
│  🛡️ Security Hotspot  → Security-sensitive code needing    │
│                        human review (not auto-flagged bad) │
│                                                           │
│  💩 Code Smell        → Maintainability issue               │
│                        (e.g. duplicated code, long method) │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**Key distinction:** A Vulnerability is something SonarQube is confident is exploitable. A Security Hotspot is "this pattern *could* be dangerous — a human needs to look at it" (e.g., use of `MD5`, which is fine for a checksum but bad for password hashing).

---

## Severity Levels

**Visual:**
```
Severity Scale:
BLOCKER   ████████████████████  Must fix immediately (crash, security hole)
CRITICAL  ███████████████       Serious bug/vulnerability
MAJOR     ██████████            Quality issue, real impact
MINOR     █████                 Style/small maintainability issue
INFO      ██                    Informational only

Example mapping:
BLOCKER  → Hardcoded database password
CRITICAL → Unclosed database connection (resource leak)
MAJOR    → Method with cyclomatic complexity of 25
MINOR    → Unused import statement
INFO     → TODO comment left in code
```

---

## Quality Profiles

**A Quality Profile is a named collection of rules, activated per language.**

**Visual:**
```
Quality Profile: "Sonar way" (Java)
┌──────────────────────────────────────────┐
│  Rule: Avoid empty catch blocks    [ON]    │
│  Rule: Methods should not be too long [ON] │
│  Rule: Avoid hardcoded credentials  [ON]   │
│  Rule: Use == for String compare  [OFF]    │
│  ... (400+ more rules)                     │
└──────────────────────────────────────────┘

Every project is assigned ONE Quality Profile per language:
┌─────────────────┐        ┌─────────────────────┐
│  payment-service │ ────→  │  Java: "Company Java"│
│  (Java + Python)  │        │  Python: "Sonar way" │
└─────────────────┘        └─────────────────────┘
```

**Built-in vs Custom:**
- SonarQube ships a default profile called **"Sonar way"** per language — a curated, reasonable baseline.
- Organizations often **copy and customize** it (e.g., disable a noisy rule, add stricter security rules) to create something like `"Company Java Standard"`.

### Creating a Custom Profile

```
Quality Profiles → Create → "Company Java Standard" → based on "Sonar way"
→ Activate/Deactivate specific rules → Set as Default for Java
```

**Visual:**
```
Inheritance:
Sonar way (built-in)
     │
     ├── inherited by ──→ Company Java Standard
     │                     (adds 5 custom rules,
     │                      disables 2 noisy ones)
     │
     └── inherited by ──→ Company Java Strict
                           (used only for payment/security-critical repos)
```

---

## Rules

**A Rule is a single, specific check** — the atomic unit of analysis.

**Visual:**
```
Rule Detail Example:
┌────────────────────────────────────────────────────┐
│ Rule: "Credentials should not be hard-coded"        │
│ Key: java:S2068                                      │
│ Type: Vulnerability                                  │
│ Severity: BLOCKER                                    │
│ Tags: cwe, security, owasp-a3                        │
│                                                       │
│ Noncompliant Code:                                   │
│   String password = "admin123";  ❌                  │
│                                                       │
│ Compliant Code:                                      │
│   String password = System.getenv("DB_PASSWORD"); ✓  │
└────────────────────────────────────────────────────┘
```

Rules come from:
- **Built-in language analyzers** (Java, Python, JS/TS, Go, C#, etc.)
- **Community/paid plugins** for extra frameworks
- **Security rule sets** mapped to OWASP Top 10, CWE, SANS Top 25

---

## Quality Gates

**A Quality Gate is a set of pass/fail conditions a project's analysis must meet.** This is what actually blocks or allows a pipeline to proceed.

**Visual:**
```
Default Quality Gate: "Sonar way"
┌──────────────────────────────────────────────────┐
│  Condition                        Threshold        │
├──────────────────────────────────────────────────┤
│  New Coverage                     ≥ 80%            │
│  New Duplicated Lines             ≤ 3%             │
│  New Maintainability Rating       = A               │
│  New Reliability Rating          = A               │
│  New Security Rating              = A               │
│  New Security Hotspots Reviewed  = 100%            │
└──────────────────────────────────────────────────┘

Analysis Result:
┌──────────────────┐        ┌──────────────────┐
│ New Coverage: 65% │  ────→ │  ❌ CONDITION FAILED│
└──────────────────┘        └──────────────────┘
         ↓
   Overall Quality Gate: ❌ FAILED
         ↓
   CI/CD Pipeline: BUILD FAILS / PR BLOCKED
```

### Creating a Custom Quality Gate

```
Quality Gates → Create → "Production Services Gate"
→ Add Condition: New Vulnerabilities = 0
→ Add Condition: New Bugs = 0
→ Add Condition: New Coverage ≥ 70%
→ Assign to specific projects
```

**Visual:**
```
Different gates for different risk levels:
┌────────────────────┐   ┌──────────────────────┐   ┌────────────────────┐
│ internal-tools-gate │   │ standard-services-gate │   │ payment-critical-gate│
│ (lenient)            │   │ (default)               │   │ (very strict)        │
│ Coverage ≥ 50%       │   │ Coverage ≥ 80%          │   │ Coverage ≥ 90%        │
│ 0 Blockers           │   │ 0 Blockers/Criticals    │   │ 0 Blockers/Criticals  │
│                       │   │                         │   │ 0 Security Hotspots   │
└────────────────────┘   └──────────────────────┘   └────────────────────┘
```

---

## New Code vs Overall Code

This is one of the most important — and most misunderstood — concepts in SonarQube.

**Visual:**
```
Legacy Codebase Problem:
┌────────────────────────────────────────────────┐
│  Existing code: 500,000 lines, 2,000 old issues  │
│  (written over 5 years, before SonarQube existed) │
└────────────────────────────────────────────────┘

If Quality Gate checked "Overall Code":
→ Gate FAILS forever, blocking every single PR
→ Team gets frustrated, disables SonarQube entirely ❌

If Quality Gate checks "New Code" only:
→ Only code changed in the last 30 days / since last version
  must meet the standard
→ Old issues remain visible but don't block new work
→ Codebase improves gradually as new code is written cleanly ✓
```

**"New Code" is defined by a New Code Period**, configurable per project:
```
Options:
- Previous version (compare against last release tag)
- Number of days (e.g. last 30 days)
- Specific date
- Reference branch (e.g. compare feature branch against main)
```

**This is the mechanism that makes adopting SonarQube on legacy codebases realistic** — you enforce quality on what's new, not what's historical.

---

## Real-Life DevOps Use Case

**Scenario:** A company has a 6-year-old monolith with thousands of pre-existing code smells. Leadership wants to introduce SonarQube without stalling all development.

**What the DevOps engineer configures:**
1. Sets the **New Code Period** to "Previous version" so only code changed since the last release is measured.
2. Creates a **custom Quality Gate** requiring:
   - Zero new Bugs/Vulnerabilities
   - New Code Coverage ≥ 75%
   - All new Security Hotspots reviewed
3. Leaves the **Overall Code** metrics visible on the dashboard for visibility/reporting to management, but **does not** gate the pipeline on them.
4. Over 6 months, tracks the **Technical Debt Ratio** trending down as legacy code gets touched and cleaned up incrementally.
5. For a newly-started microservice with no legacy baggage, assigns a **stricter Quality Gate** (e.g., 90% coverage) since there's no excuse for a fresh codebase to have low standards.

**Why this matters in practice:** Applying one universal, strict gate across both a brand-new service and a legacy monolith is the #1 reason teams abandon SonarQube. Segmenting gates by project maturity is standard DevOps practice.

---

Next: **03running_code_analysis.md** — practical scanning configuration for Java, Python, and JavaScript projects, plus reading and acting on scan reports.