# SonarQube - CI/CD Pipeline Integration

Wiring SonarQube into real pipelines so the Quality Gate becomes an enforced checkpoint, not just a dashboard nobody checks.

## Table of Contents
- [The Webhook Problem](#the-webhook-problem)
- [Jenkins Integration](#jenkins-integration)
- [GitHub Actions Integration](#github-actions-integration)
- [GitLab CI Integration](#gitlab-ci-integration)
- [Quality Gate as a Pipeline Blocker](#quality-gate-as-a-pipeline-blocker)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Webhook Problem

**Critical concept:** The scanner uploads the report asynchronously — analysis happens on the SonarQube server AFTER the scanner finishes, not instantly.

**Visual:**
```
Naive (WRONG) approach:
┌────────────┐   ┌───────────────┐   ┌────────────────────┐
│ sonar-scanner│ → │ scan completes │ → │ Pipeline immediately│
│              │   │ "SUCCESS"       │   │ checks Quality Gate │
└────────────┘   └───────────────┘   └────────────────────┘
                                              ↓
                                    ❌ Gate not computed yet!
                                    (Compute Engine still processing)

Correct approach — use a Quality Gate WEBHOOK or polling wrapper:
┌────────────┐   ┌───────────────┐   ┌─────────────┐   ┌──────────────┐
│ sonar-scanner│ → │ Report uploaded │ → │ Compute Engine│ → │ Webhook fires │
│              │   │                  │   │ processes it  │   │ pipeline      │
└────────────┘   └───────────────┘   └─────────────┘   │ continues      │
                                                          └──────────────┘

Most CI plugins/wrappers (e.g. "waitForQualityGate" in Jenkins)
handle this polling/webhook logic automatically.
```

---

## Jenkins Integration

### Step 1: Install the SonarQube Scanner Plugin
```
Manage Jenkins → Plugins → Install "SonarQube Scanner"
```

### Step 2: Configure Server Connection
```
Manage Jenkins → System → SonarQube servers
  Name: MySonarQube
  Server URL: http://sonarqube.internal:9000
  Server authentication token: (Jenkins Credential)
```

### Step 3: Declarative Pipeline

```groovy
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
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
Jenkins Pipeline Flow:
Build → SonarQube Analysis → [WAIT for webhook] → Quality Gate → Deploy
                                     ↓
                            timeout box: 5 minutes
                                     ↓
                          Gate FAILED → abortPipeline: true
                                     ↓
                          🛑 Deploy stage NEVER runs
```

**Why `abortPipeline: true` matters:** without it, Jenkins will just log the failed gate but continue to the Deploy stage anyway — silently defeating the whole purpose.

---

## GitHub Actions Integration

```yaml
name: CI with SonarQube
on:
  pull_request:
    branches: [main]

jobs:
  sonarqube:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # SonarQube needs full git history for blame data

      - name: Build and Test
        run: mvn clean verify

      - name: SonarQube Scan
        uses: SonarSource/sonarqube-scan-action@v2
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

      - name: Quality Gate Check
        uses: SonarSource/sonarqube-quality-gate-action@v1
        timeout-minutes: 5
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

**Visual:**
```
GitHub Actions Flow:
PR opened → Checkout (full history) → Build/Test → Scan → Quality Gate Action
                                                                ↓
                                                    ❌ FAILED → Action fails
                                                                ↓
                                              PR shows "Required check failed"
                                                                ↓
                                              Merge button DISABLED (if branch
                                              protection requires this check)
```

⚠️ **`fetch-depth: 0` is easy to forget** — without full git history, SonarQube can't correctly attribute "new code" vs old code, and New Code metrics become unreliable.

---

## GitLab CI Integration

```yaml
stages:
  - build
  - sonarqube
  - deploy

sonarqube-check:
  stage: sonarqube
  image: sonarsource/sonar-scanner-cli:latest
  variables:
    SONAR_TOKEN: "$SONAR_TOKEN"
    SONAR_HOST_URL: "$SONAR_HOST_URL"
    GIT_DEPTH: "0"
  script:
    - sonar-scanner
  allow_failure: false
  only:
    - merge_requests

deploy:
  stage: deploy
  script:
    - ./deploy.sh
  needs: ["sonarqube-check"]
```

**Visual:**
```
GitLab MR Pipeline:
build → sonarqube-check → deploy
              ↓
     allow_failure: false
              ↓
     Gate fails → pipeline stage fails → deploy never triggers
              (needs: dependency blocks it)
```

---

## Quality Gate as a Pipeline Blocker

**Visual — comparing enforcement levels:**
```
Level 1: Scan only, no gate check (weak)
Build → Scan → Deploy
        (report exists, but nobody has to look at it)

Level 2: Gate checked, but pipeline continues anyway (weak)
Build → Scan → Check Gate → Deploy
                  ❌ FAILED → (logged but ignored) → Deploy anyway 😬

Level 3: Gate blocks pipeline (correct)
Build → Scan → Wait for Gate → 🛑 STOP if FAILED → Deploy only if PASSED

Level 4: Gate blocks PR merge at the SCM level (strongest)
PR opened → Required Status Check: "SonarQube Quality Gate"
          → GitHub/GitLab branch protection rule requires it to pass
          → Merge button physically disabled until it passes
```

**Level 4 is the gold standard** — it means even a developer who tries to bypass the CI pipeline manually still can't merge, because the merge action itself is gated at the platform level (GitHub/GitLab branch protection), not just the pipeline.

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer notices developers are merging PRs even when the SonarQube scan fails, because the pipeline logs the failure but doesn't actually block anything (Level 2 above).

**What they do to fix it:**
1. Upgrade the pipeline to explicitly **wait for the Quality Gate webhook** (`waitForQualityGate` in Jenkins, or the dedicated quality-gate-action in GitHub Actions) instead of just running the scan and moving on.
2. Configure **branch protection rules** on `main` requiring the "SonarQube Quality Gate" status check to pass before the merge button is enabled.
3. Set a **reasonable timeout** (e.g., 5 minutes) on the gate-wait step, with an alert if it times out — this catches cases where the webhook from SonarQube to CI fails silently (a common real-world networking issue between self-hosted CI runners and a firewalled SonarQube server).
4. Adds a **PR comment bot** (via the scan action) that posts a summary comment with a direct link to failing issues, so developers don't have to hunt through the SonarQube UI.

**Common pitfall this solves:** In self-hosted setups, the SonarQube server often can't reach the CI server directly to fire the webhook (firewall/network segmentation). The DevOps engineer has to explicitly allow that traffic, or configure the CI-side plugin to poll the SonarQube API instead of waiting passively for a webhook that will never arrive.

---

Next: **05advanced_realworld_usecases.md** — PR decoration, branch analysis, Security Hotspot review workflows, and broader war-stories from real DevOps practice.