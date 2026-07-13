# JFrog Artifactory - Practical Usage (Package Managers & REST API)

Hands-on configuration for the package managers DevOps engineers use daily, plus the REST API for scripted artifact management.

## Table of Contents
- [npm Configuration](#npm-configuration)
- [Docker Configuration](#docker-configuration)
- [Maven Configuration](#maven-configuration)
- [Python (pip) Configuration](#python-pip-configuration)
- [Generic File Upload/Download](#generic-file-uploaddownload)
- [REST API for Scripted Operations](#rest-api-for-scripted-operations)
- [Searching for Artifacts](#searching-for-artifacts)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## npm Configuration

**Setting the registry (project-level, via `.npmrc`):**
```
# .npmrc
registry=https://artifactory.internal/artifactory/api/npm/npm-all/
//artifactory.internal/artifactory/api/npm/npm-all/:_authToken=${ARTIFACTORY_TOKEN}
```

**Publishing your own package:**
```bash
npm publish --registry https://artifactory.internal/artifactory/api/npm/npm-local/
```

**Visual:**
```
Why publish to npm-local specifically, not npm-all:
Virtual repos (npm-all) are for CONSUMING/resolving
packages — publishing needs to target a specific
LOCAL repo directly, since that's where YOUR package
actually gets stored. Consuming later still goes
through the virtual repo, which includes that local
repo in its aggregation.
```

---

## Docker Configuration

**Docker repos in Artifactory require the Router port (typically via a subdomain or repository-path method).**

```bash
docker login artifactory.internal:8082

docker tag myapp:latest artifactory.internal:8082/docker-local/myapp:latest
docker push artifactory.internal:8082/docker-local/myapp:latest

docker pull artifactory.internal:8082/docker-remote/nginx:1.25
```

**Visual:**
```
Docker Repository Path Structure:
artifactory.internal:8082 / docker-local / myapp:latest
        ↑                        ↑              ↑
   Artifactory host        repository key    image:tag

Pulling a cached public image:
docker pull artifactory.internal:8082/docker-remote/nginx:1.25
        ↓
Artifactory checks: is nginx:1.25 already cached?
   No  → pulls from Docker Hub, caches, serves
   Yes → serves directly from cache
```

**Visual — why this matters for Docker Hub rate limits:**
```
Docker Hub enforces PULL RATE LIMITS for anonymous/free
accounts. Without caching:
100 CI pipeline runs/day × pulling the same base image
   = 100 separate pulls against Docker Hub's rate limit
   → CI pipelines start failing once the limit is hit

With Artifactory caching:
First pull fetches from Docker Hub → cached →
   remaining 99 pulls that day are served from
   Artifactory's cache, NEVER touching Docker Hub's
   rate limit at all
```

---

## Maven Configuration

**`settings.xml`:**
```xml
<settings>
  <servers>
    <server>
      <id>central</id>
      <username>${env.ARTIFACTORY_USER}</username>
      <password>${env.ARTIFACTORY_TOKEN}</password>
    </server>
  </servers>
  <mirrors>
    <mirror>
      <id>central</id>
      <url>https://artifactory.internal/artifactory/maven-all</url>
      <mirrorOf>*</mirrorOf>
    </mirror>
  </mirrors>
</settings>
```

**`pom.xml` (for publishing):**
```xml
<distributionManagement>
  <repository>
    <id>central</id>
    <url>https://artifactory.internal/artifactory/maven-release-local</url>
  </repository>
</distributionManagement>
```

```bash
mvn deploy
```

**Visual:**
```
Why <mirrorOf>*</mirrorOf> matters:
This tells Maven to route ALL dependency resolution
(not just Maven Central specifically) through the
single Artifactory virtual repo — including any
other mirrors/repos individual pom.xml files might
otherwise try to reference directly, keeping ALL
traffic centralized and cacheable.
```

---

## Python (pip) Configuration

**`pip.conf`:**
```ini
[global]
index-url = https://artifactory.internal/artifactory/api/pypi/pypi-all/simple
```

**Publishing a package (via twine):**
```bash
twine upload --repository-url https://artifactory.internal/artifactory/api/pypi/pypi-local/ dist/*
```

**Visual:**
```
Why /simple/ appears in the pip index-url:
pip expects registries to follow PEP 503's "Simple
Repository API" format — Artifactory's PyPI virtual
repos expose this exact interface, so pip works
completely transparently, unaware it's talking to
Artifactory rather than pypi.org directly.
```

---

## Generic File Upload/Download

**For arbitrary build artifacts that don't fit a specific package manager format (installers, compiled binaries, reports).**

```bash
# Upload
curl -u admin:password -T myapp-installer.exe \
  "https://artifactory.internal/artifactory/generic-local/releases/v2.0.0/myapp-installer.exe"

# Download
curl -u admin:password -O \
  "https://artifactory.internal/artifactory/generic-local/releases/v2.0.0/myapp-installer.exe"
```

**Visual:**
```
Generic repos are essentially a simple, organized
file store — useful for things like:
- Compiled installers/binaries not tied to a package manager
- Test reports, coverage reports archived per build
- Large datasets or ML model files
- Documentation bundles (PDFs, generated HTML docs)
```

---

## REST API for Scripted Operations

**Almost everything doable in the UI can be scripted via the REST API — essential for CI/CD automation.**

```bash
# Get info about a specific artifact
curl -u admin:password \
  "https://artifactory.internal/artifactory/api/storage/maven-release-local/com/mycompany/myapp/2.0.0/myapp-2.0.0.jar"

# Copy an artifact between repositories (used for promotion, file 04)
curl -X POST -u admin:password \
  "https://artifactory.internal/artifactory/api/copy/myapp-staging-local/com/mycompany/myapp/2.0.0/myapp-2.0.0.jar?to=/myapp-release-local/com/mycompany/myapp/2.0.0/myapp-2.0.0.jar"

# Delete an artifact
curl -X DELETE -u admin:password \
  "https://artifactory.internal/artifactory/myapp-dev-local/com/mycompany/myapp/0.1.0-SNAPSHOT/myapp-0.1.0-SNAPSHOT.jar"
```

**Visual:**
```
Common Automation Patterns:
┌──────────────────────────────────────────────────┐
│  Task                        API Use                    │
├──────────────────────────────────────────────────┤
│  Promote a build                 api/copy or api/move        │
│  Clean up old snapshots            api/search + api/delete       │
│  Check artifact metadata             api/storage                  │
│  Retrieve build info                    api/build                    │
│  Trigger a repository sync                api/replication              │
└──────────────────────────────────────────────────┘
```

---

## Searching for Artifacts

**Artifact Query Language (AQL) — Artifactory's own query syntax for complex searches.**

```bash
curl -X POST -u admin:password \
  -H "Content-Type: text/plain" \
  "https://artifactory.internal/artifactory/api/search/aql" \
  -d 'items.find({"repo":"myapp-dev-local","created":{"$before":"7d"}})'
```

**Visual:**
```
What this AQL query does:
Finds all items in "myapp-dev-local" created MORE
than 7 days ago — the exact kind of query used to
power an automated cleanup job that deletes stale
dev/snapshot builds, keeping storage costs under control.

AQL Query Anatomy:
items.find({ <criteria> })
        ↓
Returns a JSON list of matching artifacts, which a
script can then feed into a bulk delete operation.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer notices Artifactory's storage costs growing rapidly, largely due to years of accumulated dev/snapshot builds that nobody ever cleans up manually.

**What they build:**
1. Writes an **AQL query** finding all artifacts in any `*-dev-local` repository older than 14 days, using the pattern-based repo naming convention established during initial setup (file 02) to target ALL dev repos with one script, rather than one-off per-team cleanup.
2. Wraps this query in a **scheduled script** (run via a cron job or CI scheduled pipeline) that first logs what WOULD be deleted for a week (dry-run mode), giving teams visibility and a chance to object before actual deletion begins.
3. After the dry-run period, enables **actual deletion**, calling the `api/delete` endpoint for each matching artifact — reducing storage usage significantly within the first month.
4. Separately, configures the **Docker remote repository's** retrieval cache period appropriately, confirming via metrics that base image pulls are being served from cache rather than re-hitting Docker Hub on every single CI run — addressing a related Docker Hub rate-limit concern the team had been hitting periodically.
5. Documents the cleanup policy (dev builds retained 14 days, staging 90 days, release builds retained indefinitely) so teams understand and can plan around the lifecycle, rather than being surprised by artifacts disappearing.

**Why this matters:** Without active lifecycle management, binary repositories only ever grow — AQL-based scripted cleanup, applied consistently via the repository naming convention, is what keeps storage costs proportional to actual need rather than accumulating indefinitely.

---

Next: **04cicd_integration.md** — Build Info, Jenkins/GitHub Actions integration, and automated promotion pipelines.