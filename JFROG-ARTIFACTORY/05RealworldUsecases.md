# JFrog Artifactory - Advanced Features & Real-World Use Cases

Going beyond single-instance usage: replication for geo-distributed teams, high availability, Xray security scanning integration, and mature real-world operating practices.

## Table of Contents
- [Replication](#replication)
- [High Availability](#high-availability)
- [Xray Security Scanning Integration](#xray-security-scanning-integration)
- [Storage Backends at Scale](#storage-backends-at-scale)
- [Access Tokens and Identity Integration](#access-tokens-and-identity-integration)
- [Federation (Multi-Site Active-Active)](#federation-multi-site-active-active)
- [Common Pitfalls & War Stories](#common-pitfalls--war-stories)
- [Real-Life DevOps Use Case (End-to-End)](#real-life-devops-use-case-end-to-end)

---

## Replication

**Keeping repository content synchronized across multiple Artifactory instances — critical for geo-distributed teams or disaster recovery.**

**Visual:**
```
┌─────────────────┐   push replication   ┌─────────────────┐
│  Artifactory (US)      │ ──────────────────────→ │  Artifactory (EU)      │
│  Primary instance          │                             │  Replica instance          │
└─────────────────┘                             └─────────────────┘

Why this matters:
A developer in Europe pulling a large Docker image
from a US-based Artifactory instance experiences
significant latency due to geographic distance.
Replicating relevant repos to a EU-based instance
means EU developers pull from a LOCAL, fast instance
instead of crossing the Atlantic for every request.
```

**Configuring push replication:**
```json
{
  "cronExp": "0 0 */6 * * ?",
  "url": "https://artifactory-eu.internal/artifactory/docker-local",
  "username": "replicator",
  "password": "***",
  "enabled": true
}
```

**Visual — replication types:**
```
┌───────────────────────────────────────────┐
│  Push Replication    → Primary PUSHES updates    │
│                      to a replica on a schedule       │
│                                                  │
│  Pull Replication      → Replica PULLS updates from     │
│                      a remote source on a schedule         │
│                                                  │
│  Event-based Replication → Triggers replication IMMEDIATELY  │
│                      on artifact deploy, rather than            │
│                      waiting for the next scheduled run            │
└───────────────────────────────────────────┘
```

---

## High Availability

**Visual:**
```
┌─────────────────────────────────────────────────┐
│                  HA Cluster (3 nodes)                  │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐  │
│  │  Node 1        │   │  Node 2        │   │  Node 3        │  │
│  │  (active)         │   │  (active)         │   │  (active)         │  │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘  │
│       │                    │                    │              │
│       └────────────────────┼────────────────────┘              │
│                            ↓                                     │
│                  ┌──────────────────┐                           │
│                  │  Shared Filestore       │  ← S3/NFS/etc,         │
│                  │  + Shared Database         │     accessible          │
│                  │  (PostgreSQL)                 │     by ALL nodes           │
│                  └──────────────────┘                           │
└─────────────────────────────────────────────────┘

All nodes are ACTIVE simultaneously (not active/passive) —
a load balancer distributes requests across all nodes,
and if one node fails, the others continue serving
without interruption.
```

**Visual — why shared storage is mandatory for HA:**
```
If each node had its OWN local filesystem storage:
Node 1 receives an upload → artifact exists ONLY on
   Node 1's disk → Node 2/3 don't have it →
   requests routed to Node 2/3 for that artifact FAIL

With shared storage (S3, NFS, etc.):
Any node can read/write any artifact → truly
   stateless nodes → any node failing doesn't
   lose any data or availability
```

---

## Xray Security Scanning Integration

**JFrog Xray is a companion product that scans artifacts stored in Artifactory for vulnerabilities and license compliance issues — similar in spirit to Trivy, but deeply integrated into the artifact lifecycle itself.**

**Visual:**
```
┌──────────────┐   scans on upload   ┌──────────────┐
│  Artifactory       │ ──────────────────────→ │      Xray          │
│  (artifact stored)    │                          │  (vulnerability DB,   │
└──────────────┘                          │   license DB)             │
                                          └──────┬───────────┘
                                                 │
                                          ┌──────┴───────────┐
                                          │  Scan results attached │
                                          │  to the artifact's         │
                                          │  metadata in Artifactory     │
                                          └──────────────────┘
```

**Setting up a Watch (policy) in Xray:**
```
Xray → Watches → New Watch
  Name: "block-critical-cves"
  Resources: myapp-release-local
  Policy: CRITICAL severity → Block download
```

**Visual:**
```
Why this differs from a CI-time scan alone (like Trivy):
CI-time scanning catches issues at BUILD time —
but if a NEW CVE is discovered in a dependency
AFTER an artifact was already built and stored,
CI-time scanning won't retroactively catch it.

Xray continuously RE-EVALUATES stored artifacts
against updated vulnerability databases — an
artifact that was "clean" when built can later
get automatically BLOCKED from further downloads
if a new CVE affecting it is discovered, without
needing a new build/scan to trigger the check.
```

**Blocking a vulnerable artifact from being pulled:**
```
Developer attempts: docker pull artifactory/docker-local/myapp:1.2.0
        ↓
Xray Watch policy check: does this artifact violate
   an active "block on CRITICAL" policy?
        ↓
YES → pull request REJECTED, with a message pointing
   to which CVE caused the block
```

---

## Storage Backends at Scale

**Visual:**
```
┌──────────────────────────────────┐
│  Filesystem (local disk)              │  ← simplest, fine for small setups
├──────────────────────────────────┤
│  S3 / Azure Blob / GCS                  │  ← production standard for large
│                                       scale, durable, cost-efficient
├──────────────────────────────────┤
│  Filestore with Sharding                    │  ← splits binary storage across
│                                       multiple backends for extreme scale
└──────────────────────────────────┘
```

**S3 configuration (binarystore.xml):**
```xml
<config version="2">
  <chain template="s3-storage-v3-direct"/>
  <provider id="s3-storage-v3" type="s3-storage-v3">
    <bucketName>company-artifactory-storage</bucketName>
    <region>us-east-1</region>
  </provider>
</config>
```

**Visual:**
```
Why object storage matters at scale:
Same rationale as Loki (file 05 in that series) —
stateless Artifactory nodes, durable and cheap
storage, and effectively unlimited capacity without
manually provisioning ever-larger disks.
```

---

## Access Tokens and Identity Integration

**Visual:**
```
┌───────────────────────────────────────────┐
│  Auth Method            Use Case                  │
├───────────────────────────────────────────┤
│  Access Tokens               CI/CD pipelines, scripted    │
│                            automation (preferred over          │
│                            username/password)                     │
│  SAML/OIDC SSO                  Human users logging in via           │
│                            corporate identity provider                  │
│  LDAP/Active Directory              Human users via existing               │
│                            corporate directory                                │
└───────────────────────────────────────────┘
```

```bash
jf rt access-token-create --scope="applied-permissions/user" --expiry=2592000
```

**Visual:**
```
Why scoped, expiring tokens matter (same principle
as Vault's AppRole from the Vault notes):
A CI pipeline's Artifactory access token should be
narrowly scoped to only what that pipeline needs,
and should EXPIRE — limiting the blast radius if
a token were ever accidentally leaked in a log file.
```

---

## Federation (Multi-Site Active-Active)

**A newer feature beyond simple replication — allows TRUE active-active writes across multiple sites, not just one-way sync.**

**Visual:**
```
Replication (older model):
Primary (writes happen here) → Replica (read-only copy)

Federation (newer model):
┌──────────────┐  ←→ bidirectional  ┌──────────────┐
│  Site A                │      sync         │  Site B                │
│  (read AND write)          │                       │  (read AND write)          │
└──────────────┘                              └──────────────┘

Both sites can accept writes, and changes propagate
in BOTH directions — useful for genuinely distributed
teams that each need local write capability, not
just fast local reads.
```

---

## Common Pitfalls & War Stories

**Visual:**
```
Pitfall 1: "Storage costs growing uncontrollably"
Cause: No cleanup policy for dev/snapshot artifacts (file 03)
Fix: Scheduled AQL-based cleanup jobs, tiered retention
     by repository stage

Pitfall 2: "HA cluster nodes have inconsistent data"
Cause: Nodes configured with LOCAL filesystem storage
       instead of shared storage
Fix: Migrate to S3/NFS-backed shared storage before
     going into any HA configuration

Pitfall 3: "Dependency confusion security incident"
Cause: Virtual repo resolution order checks remote
       BEFORE local, allowing a public malicious
       package to shadow an internal one
Fix: Always order local repos before remote in
     virtual repo configuration (file 02)

Pitfall 4: "Can't answer 'where is this vulnerable
           dependency used' during an incident"
Cause: Build Info not being published as part of CI
Fix: Standardize JFrog CLI usage with build-publish
     across every pipeline (file 04)

Pitfall 5: "EU team's builds are painfully slow"
Cause: All traffic routed to a single US-based instance
Fix: Set up replication (or Federation) to a
     geographically closer instance
```

---

## Real-Life DevOps Use Case (End-to-End)

**Scenario:** A global company with development teams in the US, EU, and APAC needs Artifactory to be a resilient, secure, and performant part of their software supply chain — not just a single server that happens to store files.

**Full workflow the team builds:**

1. **High availability:** A 3-node HA cluster per major region, backed by shared S3 storage, so no single node failure disrupts artifact availability for that region's teams.
2. **Geo-replication:** Push replication configured between US, EU, and APAC instances for shared/common repositories (base Docker images, shared internal libraries), so every region's developers pull from a geographically close, fast instance rather than crossing continents for every request.
3. **Xray-based continuous scanning:** Every artifact is scanned on upload AND continuously re-evaluated as the vulnerability database updates, with a Watch policy blocking downloads of any artifact carrying an unresolved CRITICAL CVE — catching issues discovered AFTER an artifact was already built and stored, which CI-time-only scanning would miss entirely.
4. **Standardized Build Info:** every one of 60+ microservices publishes Build Info via JFrog CLI as a mandatory pipeline step, enabling org-wide dependency searches during security incidents within minutes rather than days.
5. **Governed promotion pipeline:** builds move through dev → staging → release local repos via `jf rt build-promote` only after passing automated gates, with every promotion's audit comment linked to the triggering CI run and, where relevant, the approving ticket.
6. **Scoped, expiring access tokens:** every CI pipeline uses a narrowly-scoped, auto-rotated access token rather than a shared admin credential, limiting blast radius if any single pipeline's credentials were compromised.
7. **Scheduled lifecycle cleanup:** AQL-based jobs prune dev/snapshot artifacts older than 14 days across all teams' repos automatically, keeping storage costs proportional to genuine need.

**Why this is "real DevOps," not just running a tool:** Artifactory here isn't just "where our jar files live" — it's a globally-distributed, continuously-security-scanned, fully-audited backbone connecting every build to every deployment, with governance (promotion gates, scoped tokens, cleanup policies) preventing the most common failure modes before they cause outages or security incidents. This is the difference between "we have an artifact repository" and "we have full traceability and control over every binary that has ever reached production."

---

This completes the JFrog Artifactory note series: **Introduction → Setup → Repository Types & Concepts → Practical Package Manager Usage → CI/CD Integration → Advanced/Real-World Usage.**