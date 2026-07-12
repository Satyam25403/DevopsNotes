# SonarQube - Running Code Analysis (Practical)

Hands-on configuration for scanning real projects across different languages and build tools.

## Table of Contents
- [sonar-project.properties Deep Dive](#sonar-projectproperties-deep-dive)
- [Scanning a Java (Maven) Project](#scanning-a-java-maven-project)
- [Scanning a Java (Gradle) Project](#scanning-a-java-gradle-project)
- [Scanning a JavaScript/TypeScript Project](#scanning-a-javascripttypescript-project)
- [Scanning a Python Project](#scanning-a-python-project)
- [Including Test Coverage](#including-test-coverage)
- [Excluding Files](#excluding-files)
- [Reading the Scan Report](#reading-the-scan-report)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## sonar-project.properties Deep Dive

**Visual:**
```
File Location:
my-project/
├── sonar-project.properties   ← lives at project root
├── src/
└── pom.xml (or package.json, requirements.txt, etc.)
```

**Common keys:**
```properties
# Identity
sonar.projectKey=payment-service
sonar.projectName=Payment Service
sonar.projectVersion=1.0

# What to scan
sonar.sources=src
sonar.tests=test
sonar.exclusions=**/node_modules/**,**/*.generated.js

# Server connection
sonar.host.url=http://sonarqube.internal.company.com
sonar.token=${env.SONAR_TOKEN}

# Language-specific
sonar.java.binaries=target/classes
sonar.python.coverage.reportPaths=coverage.xml
```

**Visual:**
```
Property Categories:
┌─────────────────┬───────────────────────────────┐
│ Category         │ Examples                        │
├─────────────────┼───────────────────────────────┤
│ Identity          │ projectKey, projectName          │
│ Scope             │ sources, tests, exclusions        │
│ Connection        │ host.url, token                   │
│ Language-specific │ java.binaries, python.coverage    │
└─────────────────┴───────────────────────────────┘
```

---

## Scanning a Java (Maven) Project

Maven has a dedicated SonarQube plugin — no separate scanner install needed.

```bash
mvn clean verify sonar:sonar \
  -Dsonar.host.url=http://localhost:9000 \
  -Dsonar.token=sqa_your_token_here
```

**Visual:**
```
Maven Build + Scan Flow:
┌──────────┐   ┌──────────┐   ┌───────────┐   ┌────────────────┐
│  clean    │ → │  verify   │ → │ sonar:sonar│ → │ Upload Report   │
│ (wipe old │   │ (compile, │   │ (analyze)  │   │ to SonarQube    │
│  builds)  │   │  test)    │   │            │   │ Server           │
└──────────┘   └──────────┘   └───────────┘   └────────────────┘

Why "verify" before "sonar:sonar"?
SonarQube needs compiled .class files AND test/coverage reports
to give a complete picture — running sonar:sonar alone on
un-compiled code gives incomplete results.
```

**pom.xml snippet (optional, for defaults):**
```xml
<properties>
  <sonar.projectKey>payment-service</sonar.projectKey>
  <sonar.host.url>http://localhost:9000</sonar.host.url>
</properties>
```

---

## Scanning a Java (Gradle) Project

```groovy
// build.gradle
plugins {
  id "org.sonarqube" version "4.4.1.3373"
}

sonar {
  properties {
    property "sonar.projectKey", "payment-service"
    property "sonar.host.url", "http://localhost:9000"
  }
}
```

```bash
gradle sonar -Dsonar.token=sqa_your_token_here
```

**Visual:**
```
Gradle Task Graph:
compileJava → test → jacocoTestReport → sonar
                          ↓
                  (coverage report generated
                   BEFORE sonar task runs)
```

---

## Scanning a JavaScript/TypeScript Project

```properties
# sonar-project.properties
sonar.projectKey=frontend-app
sonar.sources=src
sonar.tests=src
sonar.test.inclusions=**/*.test.js,**/*.spec.ts
sonar.javascript.lcov.reportPaths=coverage/lcov.info
```

```bash
# Generate coverage first (example with Jest)
npm run test -- --coverage

# Then scan
sonar-scanner
```

**Visual:**
```
Frontend Pipeline:
┌────────────┐   ┌──────────────┐   ┌────────────┐   ┌──────────┐
│ npm install │ → │ npm run test  │ → │ lcov.info   │ → │ sonar-    │
│             │   │ --coverage    │   │ generated   │   │ scanner   │
└────────────┘   └──────────────┘   └────────────┘   └──────────┘

Without coverage report generated FIRST,
SonarQube will show 0% coverage even if tests exist.
```

---

## Scanning a Python Project

```properties
# sonar-project.properties
sonar.projectKey=data-pipeline
sonar.sources=src
sonar.tests=tests
sonar.python.coverage.reportPaths=coverage.xml
sonar.python.version=3.11
```

```bash
# Generate coverage first
pytest --cov=src --cov-report=xml

# Then scan
sonar-scanner
```

**Visual:**
```
Python Coverage Flow:
pytest --cov  →  coverage.xml  →  sonar-scanner reads it  →  Dashboard shows %

Common mistake:
Running sonar-scanner WITHOUT generating coverage.xml first
→ Coverage always shows 0%, even with 500 passing tests
```

---

## Including Test Coverage

**Visual:**
```
General Rule Across ALL Languages:

Step 1: Run tests with coverage tool ENABLED
         (JaCoCo for Java, Istanbul/Jest for JS, coverage.py for Python)
Step 2: Coverage tool produces a report file (XML/LCOV format)
Step 3: sonar-project.properties points to that report file
Step 4: THEN run sonar-scanner

If you skip Step 1-2, Sonar has nothing to read → 0% coverage
```

---

## Excluding Files

Not everything should be analyzed — generated code, vendored libraries, and test fixtures create noise.

```properties
sonar.exclusions=**/node_modules/**,**/*.min.js,**/vendor/**,**/migrations/**
sonar.coverage.exclusions=**/*.dto.ts,**/config/**
sonar.test.exclusions=**/*.mock.js
```

**Visual:**
```
Without Exclusions:
node_modules/ (50,000 files) ──→ scanned ──→ 45-min scan, noisy results ❌

With Exclusions:
node_modules/ ──→ skipped ──→ src/ only scanned ──→ 2-min scan, clean results ✓
```

---

## Reading the Scan Report

**Visual:**
```
Dashboard After Scan:
┌───────────────────────────────────────────────────┐
│  payment-service          Quality Gate: ❌ FAILED   │
├───────────────────────────────────────────────────┤
│  New Code (since last version):                    │
│    Bugs: 1          ← BLOCKS the gate               │
│    Vulnerabilities: 0                                │
│    Coverage: 68%    ← below 80% threshold, BLOCKS   │
│    Duplication: 1.2%                                 │
│                                                       │
│  Click "1 Bug" to drill in:                          │
│  ┌─────────────────────────────────────────────┐   │
│  │ File: PaymentProcessor.java, line 45          │   │
│  │ "Null pointer might be thrown here"            │   │
│  │ Severity: CRITICAL                              │   │
│  │ [View Code] [Mark as Won't Fix] [Assign]       │   │
│  └─────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────┘
```

**Triage options for any issue:**
```
Confirm       → Yes, this is a real problem, needs fixing
False Positive → Sonar misunderstood the code, not a real issue
Won't Fix      → Real issue, but accepted risk / intentional
Assign         → Delegate to a specific developer
```

---

## Real-Life DevOps Use Case

**Scenario:** A platform team manages 30 microservices across Java, Node.js, and Python. Each has different build tools.

**What the DevOps engineer standardizes:**
1. Creates a **shared CI template** (Jenkins Shared Library or GitHub Actions reusable workflow) with language-specific scan logic branching on a `LANGUAGE` variable, so every team doesn't reinvent scanning config.
2. Enforces that **every repo must generate a coverage report before scanning** — added as a pipeline validation step that fails the build early with a clear error if coverage.xml/lcov.info is missing, rather than silently showing 0% coverage.
3. Adds a shared `sonar.exclusions` baseline (node_modules, build artifacts, generated protobuf code) across all repos via the reusable pipeline template, so individual teams don't have to remember to exclude noise.
4. Sets up a **Slack notification** integration so failed Quality Gates post directly to the team's channel with a link to the specific failing issue — reducing the "check the dashboard manually" friction.

**Why this matters:** Standardizing the *scanning process* itself (not just the rules) is what lets a platform team support 30+ repos without each team's pipeline drifting into inconsistent, half-working SonarQube setups.

---

Next: **04cicd_pipeline_integration.md** — wiring SonarQube into Jenkins, GitHub Actions, and GitLab CI so the Quality Gate actually blocks merges/deploys.