# JFrog Artifactory - CI/CD Integration

Wiring Artifactory into real pipelines: capturing Build Info, publishing artifacts automatically, and building an automated promotion pipeline.

## Table of Contents
- [Build Info: The Key Differentiator](#build-info-the-key-differentiator)
- [JFrog CLI](#jfrog-cli)
- [Jenkins Integration](#jenkins-integration)
- [GitHub Actions Integration](#github-actions-integration)
- [GitLab CI Integration](#gitlab-ci-integration)
- [Automated Promotion Pipeline](#automated-promotion-pipeline)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Build Info: The Key Differentiator

**This is Artifactory's standout feature for CI/CD — capturing rich metadata about exactly what went into each build, not just the artifact itself.**

**Visual:**
```
Without Build Info:
Artifact "myapp-2.0.0.jar" exists in the repo...
but WHAT dependencies did it actually use? WHICH
git commit? WHAT CI job produced it? WHO triggered it?
→ All of this is LOST, unless manually tracked elsewhere

With Build Info:
┌─────────────────────────────────────────┐
│  Build: myapp #247                            │
│  Git commit: a1b2c3d                              │
│  Triggered by: jenkins-pipeline-payments             │
│  Dependencies used:                                     │
│    - commons-lang3:3.14.0                                  │
│    - jackson-databind:2.16.0                                  │
│    - internal-auth-lib:1.5.2                                     │
│  Artifacts produced:                                                 │
│    - myapp-2.0.0.jar                                                    │
│    - myapp-2.0.0-sources.jar                                              │
└─────────────────────────────────────────┘

This full picture is captured AUTOMATICALLY when
using the JFrog CLI or CI plugins, without manual effort.
```

**Visual — why this matters during an incident:**
```
Scenario: a critical CVE is announced in jackson-databind
2.16.0.

Without Build Info: "Do we use this anywhere? Which
   services? Which specific builds?" — requires manually
   checking every project's lockfiles/pom.xml across
   the whole organization.

With Build Info: search Artifactory's Build Info records
   directly for "jackson-databind:2.16.0" — instantly get
   a list of EVERY build across the whole company that
   used this exact dependency version, in seconds.
```

---

## JFrog CLI

**The standard tool for interacting with Artifactory from CI pipelines — wraps package manager commands while automatically capturing Build Info.**

```bash
# Install
curl -fL https://install-cli.jfrog.io | sh

# Configure connection
jf config add my-server --url=https://artifactory.internal --user=admin --password=$ARTIFACTORY_TOKEN

# Run a build wrapped with Build Info collection
jf npm install
jf npm publish

jf rt build-collect-env myapp-build 247
jf rt build-add-git myapp-build 247
jf rt build-publish myapp-build 247
```

**Visual:**
```
jf npm install
        ↓
JFrog CLI wraps the normal "npm install" command,
but ALSO records exactly which package versions
were resolved during this specific run
        ↓
jf rt build-publish
        ↓
Sends the full collected Build Info (dependencies,
environment variables, git commit, CI job metadata)
to Artifactory, associated with this build number
```

---

## Jenkins Integration

```groovy
pipeline {
    agent any
    stages {
        stage('Install & Build') {
            steps {
                sh 'jf npm install'
                sh 'jf npm run build'
            }
        }
        stage('Publish') {
            steps {
                sh 'jf npm publish'
            }
        }
        stage('Capture Build Info') {
            steps {
                sh 'jf rt build-collect-env myapp-build ${BUILD_NUMBER}'
                sh 'jf rt build-add-git myapp-build ${BUILD_NUMBER}'
                sh 'jf rt build-publish myapp-build ${BUILD_NUMBER}'
            }
        }
    }
}
```

**Visual:**
```
Jenkins Pipeline Flow:
Install & Build → Publish artifact → Capture & publish
   Build Info (linking THIS specific Jenkins build
   number to the exact dependencies/artifact produced)

The JFrog Jenkins Plugin can also handle much of
this automatically via pipeline steps, reducing the
amount of manual CLI scripting needed.
```

---

## GitHub Actions Integration

```yaml
name: Build and Publish
on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup JFrog CLI
        uses: jfrog/setup-jfrog-cli@v4
        env:
          JF_URL: ${{ secrets.ARTIFACTORY_URL }}
          JF_ACCESS_TOKEN: ${{ secrets.ARTIFACTORY_TOKEN }}

      - name: Install and Build
        run: |
          jf npm install
          jf npm run build

      - name: Publish
        run: jf npm publish

      - name: Publish Build Info
        run: |
          jf rt build-collect-env
          jf rt build-add-git
          jf rt build-publish
```

**Visual:**
```
Why the official setup-jfrog-cli action matters:
Handles authentication securely via GitHub Secrets,
installs the correct CLI version, and pre-configures
the connection — avoiding manual credential handling
in each individual workflow step.
```

---

## GitLab CI Integration

```yaml
stages:
  - build
  - publish

build-and-publish:
  stage: publish
  image: releases-docker.jfrog.io/jfrog/jfrog-cli:latest
  variables:
    JF_URL: $ARTIFACTORY_URL
    JF_ACCESS_TOKEN: $ARTIFACTORY_TOKEN
  script:
    - jf npm install
    - jf npm run build
    - jf npm publish
    - jf rt build-collect-env
    - jf rt build-add-git
    - jf rt build-publish
```

---

## Automated Promotion Pipeline

**Moving a build through lifecycle stages WITHOUT rebuilding — the core supply-chain-integrity benefit from file 00.**

```bash
jf rt build-promote myapp-build 247 myapp-release-local \
  --source-repo=myapp-staging-local \
  --status=released \
  --comment="Passed staging validation, approved for release"
```

**Visual:**
```
Promotion Flow:
┌─────────────────────┐
│  Build #247                │
│  Currently in:                 │
│  myapp-staging-local              │
└──────────┬──────────────┘
           │  jf rt build-promote
           ↓
┌─────────────────────┐
│  SAME artifact, now in:      │
│  myapp-release-local              │
│  Status: "released"                    │
│  Comment: audit trail preserved             │
└─────────────────────┘

Critical: the BYTES of the artifact never change.
Only its location/metadata/status change. This is
what guarantees "what we tested is EXACTLY what
we deployed."
```

**Automating promotion as a pipeline gate:**
```groovy
stage('Promote to Release') {
    when {
        expression { currentBuild.result == 'SUCCESS' }
    }
    steps {
        sh '''
          jf rt build-promote myapp-build ${BUILD_NUMBER} myapp-release-local \
            --source-repo=myapp-staging-local \
            --status=released
        '''
    }
}
```

**Visual:**
```
Full Pipeline with Promotion Gate:
Build → Publish to dev-local → Deploy to staging →
   Run automated tests against staging →
   ┌─────────────┐
   │  Tests PASS?      │
   └──────┬──────────┘
          │ Yes                    │ No
          ↓                          ↓
   Promote to                  Pipeline STOPS,
   release-local                 artifact stays in
          ↓                          staging-local,
   Deploy to production            never promoted
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer needs to answer, within an hour, "which of our 40 microservices use the vulnerable dependency version" after a critical CVE is announced in a widely-used JSON parsing library — a request from the security team during an active incident.

**What they do:**
1. Because every pipeline across all 40 services has been publishing **Build Info** via JFrog CLI as standard practice, runs a single Artifactory API query searching Build Info records across all builds for the specific vulnerable dependency name and version.
2. Gets back an exact list of affected builds/services within minutes — a task that would otherwise require manually checking 40 separate repos' lockfiles.
3. Cross-references this list against which specific builds have been **promoted to release-local** (i.e., actually deployed to production) versus those still sitting in dev/staging — prioritizing the incident response on production-affecting services first.
4. For each affected service, triggers a new build with the patched dependency version, and once it passes staging validation, uses `jf rt build-promote` to move it to release-local — with the promotion's audit comment explicitly referencing the CVE ticket, preserving a clear incident-response paper trail.
5. After the incident, proposes (and gets approved) a new automated **build-time gate**: any build with dependencies matching a known-critical CVE list gets blocked from promotion automatically, rather than relying on manual cross-referencing next time.

**Why this matters:** Build Info's value is most obvious exactly in this kind of incident-response scenario — the ability to instantly answer "where exactly is this dependency used, in production, right now" is the difference between a focused, hour-long remediation and a days-long manual audit across dozens of repositories.

---

Next: **05advanced_realworld_usecases.md** — replication, high availability, Xray security scanning integration, and mature real-world Artifactory operating practices.