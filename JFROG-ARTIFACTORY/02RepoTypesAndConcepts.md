# JFrog Artifactory - Repository Types & Core Concepts

The core theory behind how Artifactory organizes artifacts: repository types, layouts, and permission targets.

## Table of Contents
- [The Three Repository Types Deep Dive](#the-three-repository-types-deep-dive)
- [Local Repositories](#local-repositories)
- [Remote Repositories](#remote-repositories)
- [Virtual Repositories](#virtual-repositories)
- [Repository Layouts](#repository-layouts)
- [Permission Targets](#permission-targets)
- [Repository Naming Conventions](#repository-naming-conventions)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Three Repository Types Deep Dive

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│  Local          →  Storage YOU control directly            │
│                   (push your own artifacts here)               │
│                                                           │
│  Remote            →  A proxy/cache in front of an EXTERNAL      │
│                   source (you don't control the content,           │
│                   only whether it's cached)                           │
│                                                           │
│  Virtual              →  An AGGREGATION layer — not real storage,        │
│                   just a unified access point combining                    │
│                   multiple local/remote repos                                 │
└─────────────────────────────────────────────────────┘

Analogy:
Local repo    = your own warehouse, you control what's in it
Remote repo    = a cached storefront window for a supplier's catalog
Virtual repo     = a single shopping mall entrance that leads to
                  both your warehouse AND the supplier storefront,
                  without the shopper needing to know which is which
```

---

## Local Repositories

**Where YOUR build outputs live — internal libraries, Docker images, release artifacts.**

**Visual:**
```
┌─────────────────────────────────────────┐
│  Local Repository: "myapp-release-local"   │
│  ┌───────────────────────────────────┐  │
│  │  myapp-1.0.0.jar                       │  │
│  │  myapp-1.1.0.jar                       │  │
│  │  myapp-2.0.0.jar                       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

Common local repo naming pattern by lifecycle stage:
myapp-dev-local       → snapshot/nightly builds
myapp-staging-local     → builds promoted for QA validation
myapp-release-local        → production-ready, immutable releases
```

**Creating via REST API:**
```bash
curl -X PUT "https://artifactory.internal/artifactory/api/repositories/myapp-release-local" \
  -H "Content-Type: application/json" \
  -u admin:password \
  -d '{
    "rclass": "local",
    "packageType": "maven"
  }'
```

**Visual — immutability best practice:**
```
Release repositories should typically be configured
"non-overridable" — once version 2.0.0 is published,
it CANNOT be silently replaced by a different artifact
under the same version number. This guarantees that
"myapp:2.0.0" always refers to the exact same bytes,
everywhere, forever — critical for reproducible builds
and audit trails.
```

---

## Remote Repositories

**A caching proxy in front of an external source.**

**Visual:**
```
┌─────────────────────────────────────────┐
│  Remote Repository: "docker-remote"           │
│  URL: https://registry-1.docker.io               │
│                                                │
│  Behavior:                                        │
│  Request for "nginx:1.25" NOT cached yet →           │
│      fetch from Docker Hub → cache locally →              │
│      serve to requester                                        │
│                                                │
│  Request for "nginx:1.25" ALREADY cached →              │
│      serve DIRECTLY from local cache,                        │
│      no trip to Docker Hub at all                                │
└─────────────────────────────────────────┘
```

**Configuring cache expiration:**
```json
{
  "rclass": "remote",
  "packageType": "npm",
  "url": "https://registry.npmjs.org",
  "retrievalCachePeriodSecs": 43200
}
```

**Visual:**
```
Why cache expiration matters:
retrievalCachePeriodSecs: 43200 (12 hours)
→ Artifactory re-checks with the upstream source at
  most every 12 hours for METADATA (like "what's the
  latest version") — but already-downloaded ARTIFACT
  CONTENT is cached indefinitely (immutable versions
  never need re-fetching, since e.g. npm package
  content for a specific version never changes).
```

---

## Virtual Repositories

**One URL, multiple underlying repos, resolved in a defined order.**

**Visual:**
```
┌─────────────────────────────────────────┐
│  Virtual Repository: "maven-all"              │
│  Included (in resolution order):                   │
│    1. maven-local        (your own artifacts)         │
│    2. maven-remote        (Maven Central cache)          │
│    3. maven-partner-remote  (a partner's private repo)      │
└─────────────────────────────────────────┘

Request for "com.mycompany:utils:1.0"
        ↓
Check maven-local FIRST → FOUND → serve immediately
(never even checks maven-remote or the partner repo)

Request for "org.apache.commons:commons-lang3:3.14"
        ↓
Check maven-local → not found
Check maven-remote → found (or fetched/cached from
                      Maven Central) → serve
```

**Visual — why resolution ORDER matters:**
```
If a malicious actor could publish a package with the
SAME NAME as your internal package to a public registry
("dependency confusion" attack), checking your LOCAL
repo FIRST ensures your legitimate internal package is
always resolved first — never accidentally shadowed
by a same-named public package.

This ordering is a genuine, actively exploited real-world
security concern, not just a theoretical configuration
detail.
```

---

## Repository Layouts

**Defines the expected URL/path structure for artifacts within a repository — varies by package type.**

**Visual:**
```
Maven Layout Pattern:
[organization]/[module]/[version]/[module]-[version].[ext]

Example resolved path:
com/mycompany/myapp/2.0.0/myapp-2.0.0.jar

Generic/simple layout (for arbitrary files):
[repo-key]/[any-custom-path-you-choose]/[filename]

Example:
generic-local/releases/v2.0.0/myapp-installer.exe
```

**Visual:**
```
Why layouts matter:
Each package manager (Maven, npm, Docker, etc.) has
ITS OWN expected directory/naming convention — Artifactory
enforces the correct layout automatically based on the
repository's configured package type, so tools like `mvn`
or `docker` can interact with it exactly as they would
with any standard-compliant registry.
```

---

## Permission Targets

**Fine-grained access control: which users/groups can read/deploy/delete on which repository paths.**

**Visual:**
```
┌─────────────────────────────────────────────────┐
│  Permission Target: "payments-team-access"           │
│                                                        │
│  Repositories: payments-*-local                          │
│  Users/Groups: payments-team                                │
│  Permissions:                                                  │
│    Read: ✓        Deploy: ✓       Delete: ✗                       │
│                                                        │
│  Permission Target: "readonly-org-wide"                          │
│  Repositories: *-release-local                                       │
│  Users/Groups: all-employees                                             │
│  Permissions:                                                                │
│    Read: ✓        Deploy: ✗       Delete: ✗                                     │
└─────────────────────────────────────────────────┘
```

**Visual — the principle in action:**
```
Payments team can:
- Read AND deploy to payments-dev-local, payments-release-local
- Cannot delete anything (prevents accidental/malicious removal)

Everyone else in the company can:
- READ any team's *-release-local repo (transparency,
  useful for cross-team dependency discovery)
- Cannot deploy to or modify ANY other team's repository

This mirrors the same least-privilege principle covered
in the Vault notes — scope access to exactly what's
needed, nothing more.
```

---

## Repository Naming Conventions

**Visual:**
```
A common, scalable naming pattern:

[team-or-app]-[stage]-[type]

Examples:
payments-dev-local
payments-release-local
platform-docker-local
company-npm-remote
company-npm-all (virtual)

Why this matters at scale:
With 40+ microservices, each potentially needing
dev/staging/release repos across multiple package
types, a CONSISTENT naming convention is what keeps
the repository list navigable and scriptable —
automation (like the promotion pipelines in file 04)
can rely on predictable naming patterns instead of
a special case per team.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is designing the repository structure for a company migrating 40 microservices onto Artifactory, and wants to prevent both a dependency confusion security risk and future naming chaos.

**What they design:**
1. Establishes a **strict naming convention**: `<team>-<stage>-<packagetype>-local` for local repos (e.g., `payments-release-docker-local`), documented and enforced via a repository-creation script rather than manual ad-hoc naming by each team.
2. For every virtual repository, configures **local repos to resolve BEFORE remote repos** — specifically to prevent dependency confusion attacks, where an attacker could otherwise publish a malicious package to a public registry using the same name as an internal-only package.
3. Sets up **permission targets** so each team can read/deploy only to their own `<team>-*-local` repositories, while everyone company-wide has READ-ONLY access to all `*-release-local` repos — enabling cross-team dependency reuse without cross-team write access risk.
4. Marks all `*-release-local` repositories as **non-overridable**, guaranteeing that once `payments-service:2.0.0` is published, that exact artifact can never be silently replaced — a hard requirement from their compliance team for audit purposes.
5. Documents the full naming/permission scheme in the team's internal wiki, and builds a small validation script that runs periodically, flagging any repository that doesn't conform to the agreed pattern — catching drift before it becomes an unmanageable mess across 40 teams.

**Why this matters:** Repository structure decisions made early are expensive to retrofit later — getting naming conventions, resolution order (security), and permission scoping right from the start prevents both a real security vulnerability class (dependency confusion) and the organizational chaos of 40 teams each inventing their own inconsistent repository layout.

---

Next: **03practical_usage_package_managers.md** — hands-on: configuring npm, Docker, Maven, and pip to use Artifactory, plus the REST API for scripted uploads/downloads.