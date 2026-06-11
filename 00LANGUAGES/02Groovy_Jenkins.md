# Groovy & Jenkins — DevOps Reference Notes

> **Groovy** is a JVM-based dynamic language used as the scripting engine for Jenkins pipelines. Understanding Groovy is essential for writing robust, reusable Jenkins pipelines, shared libraries, and DSL extensions.

---

## Table of Contents

1. [Groovy Core Concepts](#1-groovy-core-concepts)
2. [Groovy Data Types & Collections](#2-groovy-data-types--collections)
3. [Groovy Closures](#3-groovy-closures)
4. [Groovy String Interpolation & Multi-line Strings](#4-groovy-string-interpolation--multi-line-strings)
5. [Declarative Pipeline — Structure & Syntax](#5-declarative-pipeline--structure--syntax)
6. [Scripted Pipeline — Structure & Syntax](#6-scripted-pipeline--structure--syntax)
7. [Declarative vs Scripted — When to Use Which](#7-declarative-vs-scripted--when-to-use-which)
8. [Pipeline Steps Reference](#8-pipeline-steps-reference)
9. [Stages, Parallel, and Matrix](#9-stages-parallel-and-matrix)
10. [Environment Variables & Credentials](#10-environment-variables--credentials)
11. [Post Actions & Notifications](#11-post-actions--notifications)
12. [Shared Libraries](#12-shared-libraries)
13. [Docker in Jenkins Pipelines](#13-docker-in-jenkins-pipelines)
14. [Common Patterns & Best Practices](#14-common-patterns--best-practices)
15. [Gotchas, Bugs & Debugging](#15-gotchas-bugs--debugging)
16. [Quick Reference Cheat Sheet](#16-quick-reference-cheat-sheet)

---

## 1. Groovy Core Concepts

Groovy compiles to JVM bytecode and is fully interoperable with Java. In Jenkins, all pipeline logic runs inside a Groovy sandbox.

### Variables and typing

```groovy
// Dynamic typing — type inferred at runtime
def name = "priya"
def count = 42
def flag = true

// Optional explicit typing (Java-style)
String name = "priya"
int count = 42
boolean flag = true

// Constants
final String ENV = "production"

// Multiple assignment
def (a, b, c) = [1, 2, 3]
```

### Operators

```groovy
// Arithmetic
def x = 10 + 3   // 13
def y = 10 / 3   // 3.3333... (Groovy uses BigDecimal for division)
def z = 10.intdiv(3)  // 3 (integer division)

// Safe navigation — avoids NullPointerException
def city = user?.address?.city   // null if any part is null

// Elvis operator — default value shorthand
def name = username ?: "anonymous"   // if username is falsy, use "anonymous"

// Spaceship (comparison)
1 <=> 2    // -1
2 <=> 2    //  0
3 <=> 2    //  1

// in operator
"hello" in ["hello", "world"]   // true
```

### Control flow

```groovy
// if / else
if (env == "production") {
    deploy()
} else if (env == "staging") {
    deploySandbox()
} else {
    echo "Unknown environment"
}

// switch — Groovy switch supports any type
switch (status) {
    case "success": echo "OK"; break
    case ["warn", "unstable"]: echo "Warning"; break
    case ~/^err.*/: echo "Error pattern matched"; break
    default: echo "Unknown"
}

// for loops
for (int i = 0; i < 5; i++) { echo "$i" }
for (item in ["a", "b", "c"]) { echo item }
(1..5).each { echo it }

// while
while (retries < 3) {
    try { deploy(); break }
    catch (e) { retries++ }
}

// try / catch / finally
try {
    sh "kubectl apply -f deploy.yaml"
} catch (Exception e) {
    echo "Deploy failed: ${e.message}"
    currentBuild.result = "FAILURE"
} finally {
    echo "Cleanup always runs"
}
```

---

## 2. Groovy Data Types & Collections

### Lists

```groovy
def fruits = ["apple", "banana", "cherry"]

// Access
fruits[0]          // "apple"
fruits[-1]         // "cherry" (negative index)
fruits[1..2]       // ["banana", "cherry"] (range slice)

// Modify
fruits << "mango"               // append (same as fruits.add("mango"))
fruits.add(0, "avocado")        // insert at index
fruits.remove("banana")

// Functional operations
fruits.each { println it }
fruits.collect { it.toUpperCase() }    // → ["APPLE", "BANANA", ...]  (map)
fruits.findAll { it.startsWith("a") }  // → ["apple", "avocado"]      (filter)
fruits.find    { it.startsWith("b") }  // → "banana"                  (first match)
fruits.any     { it == "mango" }       // → true
fruits.every   { it.length() > 3 }     // → true/false
fruits.sort()
fruits.unique()
fruits.flatten()
fruits.join(", ")
fruits.size()
```

### Maps

```groovy
def config = [
    host: "localhost",
    port: 8080,
    debug: true
]

// Access
config.host           // "localhost"  (dot notation)
config["port"]        // 8080         (bracket notation)
config.get("missing", "default")  // "default" if key absent

// Modify
config.timeout = 30           // add or update
config.remove("debug")

// Iteration
config.each { key, val -> echo "$key = $val" }
config.collect { k, v -> "$k=$v" }.join("&")   // → "host=localhost&port=8080"
config.findAll { k, v -> v instanceof Integer } // filter by value type

// Check
config.containsKey("host")    // true
config.containsValue(8080)    // true
```

### Ranges

```groovy
def r = 1..5             // inclusive: [1, 2, 3, 4, 5]
def ex = 1..<5           // exclusive end: [1, 2, 3, 4]

r.each { echo "$it" }
r.contains(3)            // true
r.size()                 // 5
r.toList()               // [1, 2, 3, 4, 5]
```

---

## 3. Groovy Closures

Closures are the backbone of Jenkins pipeline DSL — every `stage { }`, `steps { }`, and `post { }` block is a closure.

```groovy
// Basic closure
def greet = { name -> "Hello, $name!" }
greet("priya")    // "Hello, priya!"

// Implicit parameter: 'it'
def double = { it * 2 }
double(5)    // 10

// Closure as method parameter
["a", "b"].each { item -> echo item }
["a", "b"].collect { it.toUpperCase() }

// Closure with multiple params
def multiply = { x, y -> x * y }

// Returning from a closure (use 'return')
def findFirst = { list, predicate ->
    list.find { predicate(it) }
}

// Closure delegates — how Jenkins DSL works
// When you write: stage("Build") { ... }
// Jenkins resolves 'stage' via the pipeline binding/delegate
```

---

## 4. Groovy String Interpolation & Multi-line Strings

```groovy
// GString — double quotes allow interpolation
def name = "priya"
def greeting = "Hello, ${name}!"         // "Hello, priya!"
def cmd = "kubectl get pods -n ${namespace} | grep ${appName}"

// Single quotes — no interpolation (literal)
def literal = 'Hello, ${name}!'         // "Hello, ${name}!" (not interpolated)

// Multi-line GString (triple double quotes)
def script = """
    #!/bin/bash
    echo "Deploying ${appName} to ${env}"
    kubectl set image deployment/${appName} \\
        ${appName}=${imageTag}
"""

// Multi-line literal (triple single quotes)
def raw = '''
    No ${interpolation} here
    Literal backslash: \n
'''

// Slashy strings — great for regex (no escaping needed)
def regex = /^[a-z0-9-]+$/
def path  = /C:\Users\priya\Desktop/   // no need to escape backslashes
```

---

## 5. Declarative Pipeline — Structure & Syntax

Declarative is the **recommended approach** for new pipelines. It enforces a strict structure and is easier to read and lint.

```groovy
pipeline {
    // ── Agent ─────────────────────────────────────────────────
    agent {
        label "linux"           // run on node with this label
        // OR:
        // agent any            // any available agent
        // agent none           // define per-stage (see below)
        // agent { docker { image "node:20" } }
    }

    // ── Triggers ──────────────────────────────────────────────
    triggers {
        cron("H 2 * * 1")                // every Monday ~2 AM
        pollSCM("H/5 * * * *")           // poll SCM every 5 min
        upstream(upstreamProjects: "other-job", threshold: hudson.model.Result.SUCCESS)
    }

    // ── Parameters ────────────────────────────────────────────
    parameters {
        string(name: "IMAGE_TAG",    defaultValue: "latest",     description: "Docker image tag")
        choice(name: "ENVIRONMENT",  choices: ["dev","staging","production"], description: "Target env")
        booleanParam(name: "SKIP_TESTS", defaultValue: false,   description: "Skip test stage")
        password(name: "API_KEY",    defaultValue: "",           description: "API key")
    }

    // ── Environment ───────────────────────────────────────────
    environment {
        APP_NAME   = "my-app"
        REGISTRY   = "registry.example.com"
        IMAGE      = "${REGISTRY}/${APP_NAME}:${params.IMAGE_TAG}"
        // Pull from Jenkins credentials store
        DB_PASS    = credentials("db-password")               // string credential
        DOCKER_REG = credentials("docker-registry-creds")    // username:password → DOCKER_REG_USR, DOCKER_REG_PSW
    }

    // ── Options ───────────────────────────────────────────────
    options {
        timeout(time: 30, unit: "MINUTES")
        buildDiscarder(logRotator(numToKeepStr: "10"))
        disableConcurrentBuilds()
        skipStagesAfterUnstable()
        timestamps()
        ansiColor("xterm")
    }

    // ── Stages ────────────────────────────────────────────────
    stages {
        stage("Checkout") {
            steps {
                checkout scm
                sh "git log --oneline -5"
            }
        }

        stage("Test") {
            when {
                not { expression { params.SKIP_TESTS } }
            }
            steps {
                sh "npm ci && npm test"
            }
            post {
                always {
                    junit "test-results/**/*.xml"
                    publishHTML(target: [reportDir: "coverage", reportFiles: "index.html", reportName: "Coverage"])
                }
            }
        }

        stage("Build Image") {
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }

        stage("Push Image") {
            when {
                branch "main"
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "docker-registry-creds",
                    usernameVariable: "DOCKER_USER",
                    passwordVariable: "DOCKER_PASS"
                )]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login ${REGISTRY} -u "\$DOCKER_USER" --password-stdin
                        docker push ${IMAGE}
                    """
                }
            }
        }

        stage("Deploy") {
            when {
                allOf {
                    branch "main"
                    environment name: "ENVIRONMENT", value: "production"
                }
            }
            steps {
                sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${IMAGE}"
                sh "kubectl rollout status deployment/${APP_NAME} --timeout=5m"
            }
        }
    }

    // ── Post ──────────────────────────────────────────────────
    post {
        success {
            slackSend(channel: "#deployments", color: "good", message: "✅ ${APP_NAME} deployed: ${IMAGE}")
        }
        failure {
            slackSend(channel: "#deployments", color: "danger", message: "❌ Pipeline failed: ${env.BUILD_URL}")
            emailext(subject: "Build failed: ${env.JOB_NAME}", body: "${env.BUILD_URL}", to: "ops@example.com")
        }
        always {
            cleanWs()
        }
        unstable {
            echo "Build is unstable — test failures detected"
        }
        changed {
            echo "Build result changed from previous run"
        }
    }
}
```

---

## 6. Scripted Pipeline — Structure & Syntax

Scripted pipelines are pure Groovy — full language power, but more verbose and harder to lint.

```groovy
// Scripted pipelines start with: node { }

node("linux") {
    def appName  = "my-app"
    def registry = "registry.example.com"
    def imageTag = ""

    try {
        stage("Checkout") {
            checkout scm
            imageTag = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
            echo "Building commit: ${imageTag}"
        }

        stage("Test") {
            sh "npm ci && npm test"
        }

        stage("Build") {
            sh "docker build -t ${registry}/${appName}:${imageTag} ."
        }

        stage("Push") {
            if (env.BRANCH_NAME == "main") {
                withCredentials([usernamePassword(
                    credentialsId: "docker-registry-creds",
                    usernameVariable: "USER",
                    passwordVariable: "PASS"
                )]) {
                    sh """
                        echo "\$PASS" | docker login ${registry} -u "\$USER" --password-stdin
                        docker push ${registry}/${appName}:${imageTag}
                    """
                }
            } else {
                echo "Skipping push — not on main branch"
            }
        }

        stage("Deploy") {
            timeout(time: 5, unit: "MINUTES") {
                sh "kubectl set image deployment/${appName} ${appName}=${registry}/${appName}:${imageTag}"
                sh "kubectl rollout status deployment/${appName}"
            }
        }

        currentBuild.result = "SUCCESS"

    } catch (Exception e) {
        currentBuild.result = "FAILURE"
        echo "Pipeline failed: ${e.message}"
        throw e     // rethrow so Jenkins marks the build red

    } finally {
        // Always clean up
        stage("Cleanup") {
            deleteDir()
        }
        // Notify
        def color = currentBuild.result == "SUCCESS" ? "good" : "danger"
        slackSend(channel: "#deployments", color: color, message: "${currentBuild.result}: ${appName}:${imageTag}")
    }
}
```

---

## 7. Declarative vs Scripted — When to Use Which

| Feature                     | Declarative                       | Scripted                          |
|-----------------------------|-----------------------------------|-----------------------------------|
| Syntax                      | Structured DSL blocks             | Full Groovy code                  |
| Readability                 | ✅ Easy to scan                   | Can get complex                   |
| Linting / validation        | ✅ `pipeline {}` linted by Jenkins| Harder to validate                |
| Groovy logic in steps       | Limited — use `script {}` block   | ✅ Full Groovy anywhere           |
| `when` conditions           | ✅ Built-in `when` directive      | Manual `if/else`                  |
| `post` hooks                | ✅ Built-in `post {}` block       | Manual `try/finally`              |
| Matrix builds               | ✅ `matrix {}` directive          | Manual loops                      |
| Error handling              | `catchError`, `post { failure }`  | `try/catch` anywhere              |
| Parallel stages             | ✅ `parallel {}` directive        | `parallel()` method               |
| Restart from stage          | ✅ Supported                      | ❌ Not supported                  |
| Blue Ocean UI compatibility | ✅ Full support                   | Partial                           |
| **Best for**                | Standard CI/CD, most pipelines    | Complex logic, loops, dynamic stages|

**Rule of thumb:** Start with Declarative. Drop into `script {}` blocks for Groovy logic. Only use fully Scripted when you need dynamic stage generation or complex flow control that Declarative can't express.

```groovy
// Mixing: Declarative with a script {} block
stage("Dynamic Deploy") {
    steps {
        script {
            // Full Groovy available inside script {}
            def targets = ["us-east-1", "ap-south-1"]
            targets.each { region ->
                sh "helm upgrade --install myapp ./chart --set region=${region}"
            }
        }
    }
}
```

---

## 8. Pipeline Steps Reference

### SCM

```groovy
// Checkout from configured SCM
checkout scm

// Explicit Git checkout
checkout([
    $class: "GitSCM",
    branches: [[name: "*/main"]],
    userRemoteConfigs: [[
        url: "https://github.com/org/repo.git",
        credentialsId: "github-creds"
    ]],
    extensions: [
        [$class: "CloneOption", shallow: true, depth: 1],
        [$class: "CleanBeforeCheckout"]
    ]
])

// Get current commit hash
def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
def branch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
```

### Shell steps

```groovy
// Basic shell command — fails build on non-zero exit
sh "kubectl apply -f deploy.yaml"

// Multi-line shell
sh """
    set -euo pipefail
    echo "Deploying ${appName}"
    kubectl apply -f deploy.yaml
    kubectl rollout status deployment/${appName} --timeout=5m
"""

// Capture output
def tag    = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
def status = sh(script: "curl -s -o /dev/null -w '%{http_code}' https://example.com", returnStdout: true).trim()

// Capture exit code without failing
def rc = sh(script: "grep -r 'TODO' src/", returnStatus: true)
if (rc != 0) { echo "No TODOs found" }

// Windows
bat "mvn clean install"
powershell "Get-Process"
```

### File operations

```groovy
// Write file
writeFile(file: "version.txt", text: "${imageTag}\n")

// Read file
def version = readFile("version.txt").trim()

// Read as JSON
def config = readJSON(file: "config.json")
echo config.port

// Read as YAML
def values = readYaml(file: "values.yaml")

// Write JSON
writeJSON(file: "output.json", json: [status: "ok", tag: imageTag], pretty: 2)

// Check existence
if (fileExists("build/output.jar")) {
    archiveArtifacts "build/output.jar"
}
```

### Artifacts and stashing

```groovy
// Archive build artifacts (persisted to Jenkins)
archiveArtifacts(artifacts: "dist/**", fingerprint: true)
archiveArtifacts(artifacts: "**/*.jar", excludes: "**/*-sources.jar")

// Stash — pass files between stages/nodes
stash(name: "built-app", includes: "dist/**,package.json")

// Unstash on another node
node("deploy-node") {
    unstash "built-app"
    sh "scp -r dist/* server:/var/www/"
}
```

### Build flow control

```groovy
// Fail the build
error("Deployment failed — rolling back")

// Mark unstable (yellow) without failing
unstable("Test warnings found")
currentBuild.result = "UNSTABLE"

// Abort current build
currentBuild.rawBuild.executor.interrupt(Result.ABORTED)

// Input — pause and wait for manual approval
input(
    message: "Deploy to production?",
    ok: "Deploy",
    submitter: "ops-team",
    parameters: [
        string(name: "REASON", description: "Reason for deployment")
    ]
)

// Timeout
timeout(time: 10, unit: "MINUTES") {
    input "Approve deployment?"
}

// Retry
retry(3) {
    sh "curl --fail https://example.com/health"
}

// Sleep
sleep(time: 30, unit: "SECONDS")

// waitUntil
waitUntil {
    def status = sh(script: "kubectl get pod my-pod -o jsonpath='{.status.phase}'", returnStdout: true).trim()
    return status == "Running"
}
```

---

## 9. Stages, Parallel, and Matrix

### Parallel stages

```groovy
// Declarative — parallel inside stage
stage("Test") {
    parallel {
        stage("Unit Tests") {
            agent { label "linux" }
            steps { sh "npm run test:unit" }
        }
        stage("Integration Tests") {
            agent { label "linux" }
            steps { sh "npm run test:integration" }
        }
        stage("Lint") {
            steps { sh "npm run lint" }
        }
    }
}

// Scripted — parallel() method
def tests = [:]
tests["unit"]        = { sh "npm run test:unit" }
tests["integration"] = { sh "npm run test:integration" }
tests["lint"]        = { sh "npm run lint" }
parallel tests

// Fail fast — stop all branches if one fails
parallel(
    failFast: true,
    "unit":   { sh "npm test" },
    "lint":   { sh "npm run lint" }
)
```

### Matrix builds (Declarative only)

```groovy
stage("Cross-Platform Build") {
    matrix {
        axes {
            axis {
                name "OS"
                values "linux", "windows", "macos"
            }
            axis {
                name "NODE_VERSION"
                values "18", "20", "22"
            }
        }
        excludes {
            exclude {
                axis { name "OS";           values "windows" }
                axis { name "NODE_VERSION"; values "18" }
            }
        }
        stages {
            stage("Build") {
                steps {
                    echo "Building on ${OS} with Node ${NODE_VERSION}"
                    sh "node --version"
                }
            }
        }
    }
}
```

### Dynamic parallel from a list (Scripted)

```groovy
def deployRegions(regions) {
    def jobs = [:]
    regions.each { region ->
        def r = region   // capture for closure
        jobs[r] = {
            node("deploy") {
                sh "helm upgrade --install myapp ./chart --set region=${r}"
            }
        }
    }
    parallel jobs
}

node {
    stage("Deploy") {
        deployRegions(["us-east-1", "eu-west-1", "ap-south-1"])
    }
}
```

---

## 10. Environment Variables & Credentials

### Built-in Jenkins environment variables

```groovy
echo env.BUILD_NUMBER          // "42"
echo env.BUILD_ID              // "42"
echo env.JOB_NAME              // "my-pipeline"
echo env.JOB_BASE_NAME         // "my-pipeline" (without folder path)
echo env.BUILD_URL             // "https://jenkins.example.com/job/my-pipeline/42/"
echo env.WORKSPACE             // "/var/jenkins/workspace/my-pipeline"
echo env.NODE_NAME             // "linux-agent-1"
echo env.BRANCH_NAME           // "main" (Multibranch Pipeline)
echo env.GIT_COMMIT            // full commit SHA
echo env.GIT_BRANCH            // "origin/main"
echo env.CHANGE_ID             // PR number (if PR build)
echo env.CHANGE_AUTHOR         // PR author (if PR build)
```

### Setting environment variables

```groovy
// Declarative — environment block
environment {
    APP_ENV = "production"
    IMAGE   = "registry.example.com/myapp:${params.TAG}"
}

// Scripted — env global
env.APP_VERSION = sh(script: "cat version.txt", returnStdout: true).trim()

// Scope to a stage using withEnv
withEnv(["NODE_ENV=test", "PORT=9999"]) {
    sh "npm test"
}
```

### Credentials

```groovy
// String credential
withCredentials([string(credentialsId: "api-token", variable: "API_TOKEN")]) {
    sh 'curl -H "Authorization: Bearer $API_TOKEN" https://api.example.com'
}

// Username + password
withCredentials([usernamePassword(
    credentialsId: "docker-registry",
    usernameVariable: "DOCKER_USER",
    passwordVariable: "DOCKER_PASS"
)]) {
    sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
}

// SSH key
withCredentials([sshUserPrivateKey(
    credentialsId: "deploy-key",
    keyFileVariable: "SSH_KEY",
    usernameVariable: "SSH_USER"
)]) {
    sh "ssh -i $SSH_KEY $SSH_USER@server 'ls -la'"
}

// File credential
withCredentials([file(credentialsId: "kubeconfig", variable: "KUBECONFIG_FILE")]) {
    sh "kubectl --kubeconfig=$KUBECONFIG_FILE get pods"
}

// Secret file via environment (Declarative)
environment {
    KUBECONFIG = credentials("kubeconfig-secret")
}

// NEVER echo credentials — Jenkins auto-masks them
// sh "echo $DOCKER_PASS"  ← prints "****" in logs
```

---

## 11. Post Actions & Notifications

### Post conditions

```groovy
post {
    always    { /* runs regardless of result */         cleanWs() }
    success   { /* only on success */                   }
    failure   { /* only on failure */                   }
    unstable  { /* only when result = UNSTABLE */       }
    aborted   { /* only when manually aborted */        }
    changed   { /* when result differs from last run */ }
    fixed     { /* was failing, now success */          }
    regression { /* was success, now failing */         }
    cleanup   { /* last post condition to run */        }
}
```

### Slack notifications

```groovy
// Requires: Slack Notification Plugin
def notifySlack(String result) {
    def colorMap = [SUCCESS: "good", FAILURE: "danger", UNSTABLE: "warning"]
    def emoji    = [SUCCESS: "✅",   FAILURE: "❌",      UNSTABLE: "⚠️"]
    def color    = colorMap[result] ?: "#808080"
    def icon     = emoji[result]    ?: "🔵"

    slackSend(
        channel:     "#deployments",
        color:       color,
        message:     "${icon} *${result}* | ${env.JOB_NAME} #${env.BUILD_NUMBER}\n<${env.BUILD_URL}|View Build>"
    )
}

post {
    success { notifySlack("SUCCESS") }
    failure { notifySlack("FAILURE") }
    unstable { notifySlack("UNSTABLE") }
}
```

### Email notifications

```groovy
// Requires: Email Extension Plugin (emailext)
post {
    failure {
        emailext(
            subject:   "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
            body:      """
                <p>Build failed: <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                <p>Check console output for details.</p>
            """,
            mimeType:  "text/html",
            to:        "ops@example.com",
            replyTo:   "no-reply@example.com",
            attachLog: true
        )
    }
}
```

### Build status badge and description

```groovy
// Set a short build description (visible in build history)
currentBuild.description = "Image: ${imageTag} → ${params.ENVIRONMENT}"

// Set display name
currentBuild.displayName = "#${env.BUILD_NUMBER} - ${imageTag}"

// Access previous build
def lastBuild = currentBuild.previousBuild
if (lastBuild?.result == "FAILURE") {
    echo "Previous build failed — this is a recovery build"
}
```

---

## 12. Shared Libraries

Shared libraries let you extract reusable pipeline code across multiple repos and pipelines.

### Directory structure

```
(repo root)
├── vars/                      # Global variables / DSL steps
│   ├── deployApp.groovy       # callable as deployApp(...) in pipelines
│   ├── buildImage.groovy
│   └── notifySlack.groovy
├── src/                       # Groovy classes (full OOP)
│   └── com/example/
│       ├── Docker.groovy
│       └── K8s.groovy
└── resources/                 # Static files (scripts, templates)
    └── scripts/
        └── health-check.sh
```

### `vars/deployApp.groovy`

```groovy
// vars/deployApp.groovy
// Called as: deployApp(app: "my-app", tag: "v1.2.3", env: "production")

def call(Map config = [:]) {
    def app = config.app ?: error("deployApp: 'app' is required")
    def tag = config.tag ?: "latest"
    def environment = config.get("env", "staging")

    echo "Deploying ${app}:${tag} to ${environment}"

    withCredentials([file(credentialsId: "${environment}-kubeconfig", variable: "KUBECONFIG")]) {
        sh """
            kubectl set image deployment/${app} ${app}=${app}:${tag}
            kubectl rollout status deployment/${app} --timeout=5m
        """
    }
}
```

### `src/com/example/Docker.groovy`

```groovy
// src/com/example/Docker.groovy
package com.example

class Docker implements Serializable {
    private def steps   // reference to pipeline steps

    Docker(steps) {
        this.steps = steps
    }

    def build(String image, String tag = "latest") {
        steps.sh "docker build -t ${image}:${tag} ."
    }

    def push(String image, String tag, String registry, String credId) {
        steps.withCredentials([
            steps.usernamePassword(credentialsId: credId, usernameVariable: "U", passwordVariable: "P")
        ]) {
            steps.sh """
                echo "\$P" | docker login ${registry} -u "\$U" --password-stdin
                docker push ${image}:${tag}
            """
        }
    }
}
```

### Loading a shared library in a pipeline

```groovy
// Method 1: Configured in Jenkins → Manage Jenkins → Global Pipeline Libraries
@Library("my-shared-lib") _                // load entire library, _ suppresses import warnings
@Library("my-shared-lib@main") _           // specific branch
@Library(["lib-a", "lib-b@v2.0"]) _       // multiple libraries

// Method 2: Dynamic loading (no pre-configuration needed)
library identifier: "my-lib@main",
    retriever: modernSCM([
        $class: "GitSCMSource",
        remote: "https://github.com/org/jenkins-shared-lib.git",
        credentialsId: "github-creds"
    ])

// Method 3: From SCM (Declarative)
pipeline {
    libraries {
        lib("my-shared-lib@main")
    }
    // ...
}
```

### Using the shared library

```groovy
@Library("my-shared-lib") _
import com.example.Docker

pipeline {
    agent any
    stages {
        stage("Build & Deploy") {
            steps {
                script {
                    // Use a vars/ function
                    buildImage(name: "my-app", tag: env.GIT_COMMIT)

                    // Use a class
                    def docker = new Docker(this)
                    docker.build("my-app", "v1.2.3")
                    docker.push("my-app", "v1.2.3", "registry.example.com", "docker-creds")
                }
                // vars/ functions used as steps (no script{} needed)
                deployApp(app: "my-app", tag: "v1.2.3", env: "production")
            }
        }
    }
}
```

### Loading a resource file from shared library

```groovy
// In your vars/ or src/ code
def script = libraryResource("scripts/health-check.sh")
writeFile(file: "health-check.sh", text: script)
sh "bash health-check.sh"
```

---

## 13. Docker in Jenkins Pipelines

### Agent-level Docker

```groovy
// Declarative — run entire pipeline in a container
pipeline {
    agent {
        docker {
            image "node:20-alpine"
            args  "-v /var/run/docker.sock:/var/run/docker.sock"  // DinD
            label "linux"
            registryUrl "https://registry.example.com"
            registryCredentialsId "docker-creds"
        }
    }
    stages {
        stage("Build") {
            steps {
                sh "npm ci && npm build"
            }
        }
    }
}
```

### Per-stage Docker agent

```groovy
pipeline {
    agent none   // no global agent
    stages {
        stage("Build") {
            agent { docker { image "node:20" } }
            steps { sh "npm ci && npm run build" }
        }
        stage("Test") {
            agent { docker { image "node:20" } }
            steps { sh "npm test" }
        }
        stage("Deploy") {
            agent { label "deploy-host" }   // bare metal for kubectl
            steps { sh "kubectl apply -f deploy.yaml" }
        }
    }
}
```

### Building and pushing images

```groovy
stage("Build Image") {
    steps {
        script {
            def image = docker.build("${REGISTRY}/${APP_NAME}:${GIT_COMMIT}")

            docker.withRegistry("https://${REGISTRY}", "docker-registry-creds") {
                image.push()
                image.push("latest")   // also push with 'latest' tag
            }
        }
    }
}
```

### Docker Compose in pipelines

```groovy
stage("Integration Tests") {
    steps {
        sh """
            docker compose -f docker-compose.test.yml up -d
            docker compose -f docker-compose.test.yml run --rm tests
        """
    }
    post {
        always {
            sh "docker compose -f docker-compose.test.yml down -v"
        }
    }
}
```

---

## 14. Common Patterns & Best Practices

### Multibranch Pipeline strategy

```groovy
// Jenkinsfile at repo root — Multibranch pipelines auto-discover branches/PRs

pipeline {
    agent any
    stages {
        stage("Test") { steps { sh "npm test" } }

        stage("Build") {
            when { not { changeRequest() } }   // skip on PRs
            steps { sh "docker build ." }
        }

        stage("Deploy Staging") {
            when { branch "develop" }
            steps { deployApp(env: "staging") }
        }

        stage("Deploy Production") {
            when { branch "main" }
            steps {
                input message: "Deploy to production?", ok: "Yes, deploy"
                deployApp(env: "production")
            }
        }
    }
}
```

### Reusable `when` conditions

```groovy
when { branch "main" }
when { branch pattern: "release/*", comparator: "GLOB" }
when { tag "v*" }
when { changeRequest() }                          // is a PR/MR
when { changeRequest target: "main" }            // PR targeting main
when { environment name: "DEPLOY_TO", value: "prod" }
when { expression { return params.DEPLOY == true } }
when { buildingTag() }
when { not { branch "main" } }
when { anyOf { branch "main"; branch "develop" } }
when { allOf { branch "main"; tag pattern: "v.*" } }
```

### Safe shell scripts in pipelines

```groovy
// Always use set -euo pipefail for multi-line scripts
sh """
    set -euo pipefail
    IMAGE_TAG=\$(git rev-parse --short HEAD)
    docker build -t myapp:\${IMAGE_TAG} .
    docker push myapp:\${IMAGE_TAG}
    echo \${IMAGE_TAG} > image_tag.txt
"""

// Use returnStatus when failure is acceptable
def rc = sh(script: "kubectl get deployment my-app", returnStatus: true)
if (rc != 0) {
    echo "Deployment does not exist yet — creating fresh"
}
```

### Version extraction patterns

```groovy
// From git tag
def version = sh(script: "git describe --tags --abbrev=0", returnStdout: true).trim()

// From package.json
def version = sh(script: "node -p \"require('./package.json').version\"", returnStdout: true).trim()

// From Maven pom.xml
def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()

// Short git commit
def commit = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
```

### Locking resources (prevent concurrent deploys)

```groovy
stage("Deploy Production") {
    // Requires: Lockable Resources Plugin
    lock(resource: "production-deploy", inversePrecedence: true) {
        milestone(label: "production-deploy")
        sh "helm upgrade --install myapp ./chart"
    }
}
```

### Handling unstable tests

```groovy
stage("Test") {
    steps {
        // catchError — continue pipeline, mark result
        catchError(buildResult: "UNSTABLE", stageResult: "FAILURE") {
            sh "npm test"
        }
    }
    post {
        always {
            junit(
                testResults: "test-results.xml",
                allowEmptyResults: true
            )
        }
    }
}
```

---

## 15. Gotchas, Bugs & Debugging

### CPS (Continuation Passing Style) transformation

Jenkins serialises pipeline state so builds can survive master restarts. Not all Groovy is CPS-compatible.

```groovy
// PROBLEM: Non-CPS-safe operations cause NotSerializableException
// e.g., using Java streams, closures with complex objects

// SOLUTION 1: @NonCPS annotation — skips CPS transformation (not serialised)
@NonCPS
def parseVersion(String text) {
    def matcher = text =~ /version:\s*(\S+)/
    return matcher ? matcher[0][1] : null
}

// SOLUTION 2: Move complex logic to a method
@NonCPS
def filterImages(List images) {
    return images.findAll { it.contains("nginx") }
}

// Then call it from pipeline
def filtered = filterImages(allImages)
```

### Variable scope in closures

```groovy
// BUG: loop variable captured by reference, not value
def jobs = [:]
["us-east-1", "eu-west-1"].each { region ->
    jobs[region] = {
        sh "deploy.sh ${region}"   // 'region' may be "eu-west-1" for ALL jobs!
    }
}

// FIX: capture with a local variable
["us-east-1", "eu-west-1"].each { region ->
    def r = region   // local copy
    jobs[r] = { sh "deploy.sh ${r}" }
}
```

### `sh` vs `bat` vs `powershell`

```groovy
// sh   → Linux/macOS
// bat  → Windows CMD
// powershell → Windows PowerShell

// Escaping in GStrings inside sh — use single $ for shell, ${ } for Groovy
sh "echo \$HOME"              // \$HOME → shell variable $HOME
sh "echo ${GROOVY_VAR}"       // ${GROOVY_VAR} → Groovy interpolation
sh 'echo $HOME'               // single-quoted → no Groovy interpolation, literal $HOME passed to shell
```

### Credentials leaking in logs

```groovy
// WRONG — never pass credentials via Groovy string interpolation
sh "docker login -u ${DOCKER_USER} -p ${DOCKER_PASS}"  // prints in logs!

// CORRECT — let shell resolve the variable
withCredentials([usernamePassword(credentialsId: "docker", usernameVariable: "U", passwordVariable: "P")]) {
    sh 'docker login -u "$U" -p "$P"'   // Jenkins masks ****
}
```

### Timeout not stopping `sh`

```groovy
// timeout {} wraps the step, but the shell process may still run
// Use sh with a timeout flag for safety
sh "timeout 300 kubectl rollout status deployment/my-app"

// Or combine both
timeout(time: 6, unit: "MINUTES") {
    sh "kubectl rollout status deployment/my-app --timeout=5m"
}
```

### `readFile` and `writeFile` require Groovy sandbox approval

```groovy
// Some pipeline steps require script approval in:
// Manage Jenkins → In-process Script Approval

// If you see: RejectedAccessException: Scripts not permitted to use ...
// → Request approval for the specific method
```

### Debugging tips

```groovy
// Print all environment variables
sh "env | sort"
echo env.getEnvironment().collect { k, v -> "${k}=${v}" }.join("\n")

// Print current build info
echo """
    Job:       ${env.JOB_NAME}
    Build:     ${env.BUILD_NUMBER}
    Branch:    ${env.BRANCH_NAME}
    Commit:    ${env.GIT_COMMIT}
    Workspace: ${env.WORKSPACE}
"""

// Groovy inspect (serialise to string)
echo config.inspect()

// Enable shell xtrace in sh steps
sh """
    set -x   # print each command before executing
    kubectl apply -f deploy.yaml
    kubectl rollout status deployment/my-app
"""

// Check serialisability
import org.jenkinsci.plugins.workflow.cps.DSL
assert config instanceof Serializable
```

---

## 16. Quick Reference Cheat Sheet

```groovy
// ─── GROOVY BASICS ──────────────────────────────────────────────────────────
def name  = "priya"                    // dynamic var
def list  = ["a", "b", "c"]           // list
def map   = [host: "localhost", port: 8080]  // map
def range = 1..5                       // range

list.each { echo it }                  // iterate
list.collect { it.toUpperCase() }      // map
list.findAll { it != "b" }            // filter
list.find { it == "a" }               // first match

map.each { k, v -> echo "$k=$v" }
def val = map.get("missing", "default")

// ─── STRING INTERPOLATION ───────────────────────────────────────────────────
def msg = "Hello, ${name}!"            // GString
def raw = 'No ${interpolation} here'   // single quotes = literal
def ml  = """
    multi-line
    interpolated: ${name}
"""

// ─── DECLARATIVE SKELETON ───────────────────────────────────────────────────
pipeline {
    agent { label "linux" }
    options { timeout(time: 30, unit: "MINUTES") }
    parameters { string(name: "TAG", defaultValue: "latest") }
    environment { IMAGE = "myapp:${params.TAG}" }
    stages {
        stage("Build") {
            when { branch "main" }
            steps {
                sh "docker build -t ${IMAGE} ."
            }
        }
    }
    post {
        success { echo "Done" }
        failure { slackSend channel: "#ops", message: "FAILED" }
        always  { cleanWs() }
    }
}

// ─── KEY STEPS ───────────────────────────────────────────────────────────────
sh "command"                                          // run shell
sh(script: "cmd", returnStdout: true).trim()         // capture output
sh(script: "cmd", returnStatus: true)                // capture exit code
checkout scm                                          // checkout SCM
writeFile(file: "f.txt", text: "content")            // write file
def c = readFile("f.txt").trim()                     // read file
archiveArtifacts "dist/**"                           // archive artifacts
stash name: "app", includes: "dist/**"               // stash
unstash "app"                                         // unstash
input message: "Continue?", ok: "Yes"                // wait for approval
timeout(time: 5, unit: "MINUTES") { ... }            // timeout block
retry(3) { sh "flaky-command" }                      // retry
withEnv(["KEY=value"]) { ... }                       // scoped env vars

// ─── CREDENTIALS ─────────────────────────────────────────────────────────────
withCredentials([string(credentialsId: "id", variable: "VAR")]) { ... }
withCredentials([usernamePassword(credentialsId: "id", usernameVariable: "U", passwordVariable: "P")]) { ... }
withCredentials([sshUserPrivateKey(credentialsId: "id", keyFileVariable: "KEY")]) { ... }
withCredentials([file(credentialsId: "id", variable: "FILE")]) { ... }

// ─── WHEN CONDITIONS ─────────────────────────────────────────────────────────
when { branch "main" }
when { tag "v*" }
when { changeRequest() }
when { not { branch "main" } }
when { anyOf { branch "main"; branch "develop" } }
when { expression { return params.DEPLOY == true } }

// ─── POST CONDITIONS ─────────────────────────────────────────────────────────
// always | success | failure | unstable | aborted | changed | fixed | regression
```

### Key rules at a glance

| Rule                              | Detail                                                         |
|-----------------------------------|----------------------------------------------------------------|
| Use Declarative first             | Easier to lint, restart from stage, Blue Ocean compatible      |
| `script {}` for complex logic     | Drop into Groovy inside Declarative stages                     |
| Always `set -euo pipefail`        | Prevents silent failures in multi-line `sh` blocks            |
| Use `\$VAR` for shell variables   | Avoid Groovy interpolating shell variable names                |
| Never echo credentials            | Use `withCredentials` — Jenkins auto-masks them                |
| Capture loop variables            | `def r = region` before closures to avoid reference capture   |
| `@NonCPS` for non-serialisable    | Annotate methods using Java streams, complex closures         |
| `returnStdout: true` + `.trim()`  | Always trim captured shell output                              |
| `Serializable` on classes         | All shared library classes must implement `Serializable`       |
| `cleanWs()` in post always        | Clean workspace to prevent disk fill-up on agents             |

---

*Part of DevOpsNotes / LANGUAGES — see also `00_YAML.md`, `01_JSON.md`, `03_HCL.md`*