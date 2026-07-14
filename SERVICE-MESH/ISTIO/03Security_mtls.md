# Service Mesh (Istio) - Security & mTLS

Zero-trust networking: automatic mutual TLS between every service, and fine-grained authorization policies controlling who can call whom.

## Table of Contents
- [The Zero-Trust Principle](#the-zero-trust-principle)
- [Mutual TLS (mTLS) Explained](#mutual-tls-mtls-explained)
- [PeerAuthentication: Enforcing mTLS](#peerauthentication-enforcing-mtls)
- [Certificate Management (Automatic)](#certificate-management-automatic)
- [AuthorizationPolicy: Who Can Call Whom](#authorizationpolicy-who-can-call-whom)
- [JWT-Based End-User Authentication](#jwt-based-end-user-authentication)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Zero-Trust Principle

**Visual:**
```
Traditional "Perimeter" Security Model:
┌─────────────────────────────────────┐
│         Firewall / Network Perimeter        │
│  ┌───────────────────────────────┐  │
│  │  Everything INSIDE is implicitly     │  │
│  │  trusted — services can call             │  │
│  │  each other freely, no internal              │  │
│  │  verification needed                            │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
Risk: if an attacker breaches the perimeter ONCE
(one compromised pod, one leaked credential), they
can move freely and call ANY internal service.

Zero-Trust Model (Istio's approach):
┌─────────────────────────────────────┐
│  EVERY service-to-service call is                │
│  individually authenticated (mTLS)                    │
│  AND authorized (AuthorizationPolicy) —                 │
│  regardless of whether it's "inside" the                  │
│  network perimeter or not                                    │
└─────────────────────────────────────┘
A compromised pod can ONLY do what its specific
identity is explicitly authorized to do — not
"anything, because it's on the internal network."
```

---

## Mutual TLS (mTLS) Explained

**Visual:**
```
Regular TLS (e.g. HTTPS browsing):
┌──────────┐                    ┌──────────┐
│  Browser       │  verifies server's   │  Web Server   │
│                  │ ←─────────────────│                  │
└──────────┘   certificate only    └──────────┘
Only the SERVER proves its identity to the client.
The client (browser) remains anonymous to the server.

Mutual TLS (mTLS):
┌──────────┐                    ┌──────────┐
│  Service A     │ ←── both verify ──→ │  Service B     │
└──────────┘   each other's identity  └──────────┘
BOTH sides prove their identity to each other —
Service B knows EXACTLY which service is calling it,
not just "someone with a valid connection."
```

**Visual — why this matters for microservices:**
```
Without mTLS:
Any pod that can reach Service B's IP/port can call it —
no cryptographic proof of WHICH service is actually calling

With mTLS:
Service B only accepts connections presenting a valid
certificate — and that certificate cryptographically
PROVES the caller's service identity (e.g. "this really
is the 'payments' service, not an impersonator")
```

---

## PeerAuthentication: Enforcing mTLS

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

**Visual:**
```
┌───────────────────────────────────────────────┐
│  Mode              Behavior                            │
├───────────────────────────────────────────────┤
│  STRICT               ONLY accepts mTLS connections;        │
│                     plaintext connections REJECTED             │
│                                                        │
│  PERMISSIVE             Accepts BOTH mTLS and plaintext          │
│                     connections — used during MIGRATION           │
│                     to Istio, so non-meshed services                │
│                     can still communicate during rollout                │
│                                                        │
│  DISABLE                 mTLS turned off entirely                        │
└───────────────────────────────────────────────┘
```

**Visual — the typical migration path:**
```
Day 1: No mesh at all → all traffic plaintext
Day 2: Istio installed, PERMISSIVE mode →
        meshed services use mTLS automatically,
        NON-meshed services (not yet migrated)
        can still connect via plaintext —
        nothing breaks during the transition
Day 30: All services migrated → switch to STRICT →
        plaintext no longer accepted anywhere,
        true zero-trust achieved
```

**Namespace-specific override:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: payments
spec:
  mtls:
    mode: STRICT
```

**Visual:**
```
Why namespace-level overrides matter:
The "payments" namespace might be ready for STRICT
mode weeks before the rest of the organization —
per-namespace policies let security-critical services
enforce strict rules FIRST, without waiting for
every single service company-wide to finish migrating.
```

---

## Certificate Management (Automatic)

**Visual:**
```
┌─────────────────────────────────────┐
│           istiod (Citadel function)         │
│  Acts as the mesh's Certificate               │
│  Authority (CA)                                    │
└──────────────┬───────────────────────┘
               │  issues short-lived certs
      ┌────────┼────────┐
      ↓                    ↓
┌──────────┐        ┌──────────┐
│  Sidecar A     │        │  Sidecar B     │
│  cert (valid       │        │  cert (valid       │
│   ~24 hours)          │        │   ~24 hours)          │
└──────────┘        └──────────┘

Automatic rotation:
Certificates are SHORT-LIVED (default ~24 hours) and
automatically ROTATED by istiod well before expiry —
no manual certificate management, no expired-cert
outages, and a compromised certificate has a naturally
limited window of usefulness.
```

**Verifying mTLS is active:**
```bash
istioctl authn tls-check reviews.default.svc.cluster.local
```

**Visual:**
```
Output shows:
HOST                                       STATUS      
reviews.default.svc.cluster.local            OK (mTLS)     

Confirms the connection to this service is
actually using mTLS, not silently falling back
to plaintext due to a misconfiguration.
```

---

## AuthorizationPolicy: Who Can Call Whom

**mTLS proves WHO is calling (authentication) — AuthorizationPolicy decides WHAT they're allowed to do (authorization).**

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: payments-policy
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payments
  action: ALLOW
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/checkout/sa/checkout-service"]
      to:
        - operation:
            methods: ["POST"]
            paths: ["/api/charge"]
```

**Visual:**
```
What this policy enforces:
┌─────────────────────────────────────┐
│  ONLY the "checkout-service" (identified   │
│  by its Kubernetes service account)             │
│  is allowed to POST to /api/charge                │
│  on the "payments" service                            │
└─────────────────────────────────────┘

Request from checkout-service → POST /api/charge
   → ALLOWED (matches the policy)

Request from some-random-other-service → POST /api/charge
   → DENIED (doesn't match any ALLOW rule)

Request from checkout-service → DELETE /api/charge/123
   → DENIED (method doesn't match — policy only allows POST)
```

**Visual — default-deny posture (recommended for security-critical namespaces):**
```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: deny-all
  namespace: payments
spec:
  {}  # empty spec with no rules = DENY ALL by default

Then ADD specific ALLOW policies on top, explicitly
permitting only the exact calls that should be allowed —
"deny by default, allow by exception" is the standard
zero-trust security posture, rather than "allow by
default, deny by exception."
```

---

## JWT-Based End-User Authentication

**Beyond service-to-service identity, Istio can also validate END-USER identity tokens (JWTs) at the mesh layer.**

```yaml
apiVersion: security.istio.io/v1beta1
kind: RequestAuthentication
metadata:
  name: jwt-auth
  namespace: payments
spec:
  selector:
    matchLabels:
      app: payments
  jwtRules:
    - issuer: "https://auth.company.com"
      jwksUri: "https://auth.company.com/.well-known/jwks.json"
```

**Visual:**
```
Without mesh-level JWT validation:
EVERY service must implement its OWN JWT validation
logic — duplicated across languages/teams, easy to
get subtly wrong in one service

With RequestAuthentication:
JWT validation happens ONCE, at the Envoy sidecar
level, BEFORE the request even reaches your
application code — application code can trust that
any request it receives already has a validated,
legitimate JWT (if one was required).
```

**Combining with AuthorizationPolicy for claim-based rules:**
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: require-admin-claim
spec:
  selector:
    matchLabels:
      app: admin-panel
  action: ALLOW
  rules:
    - when:
        - key: request.auth.claims[role]
          values: ["admin"]
```

**Visual:**
```
This policy ALLOWS access to "admin-panel" ONLY if
the validated JWT's "role" claim equals "admin" —
enforcing role-based access control at the mesh
layer, uniformly, without each service re-implementing
this check independently.
```

---

## Real-Life DevOps Use Case

**Scenario:** A financial services company must demonstrate zero-trust internal networking for a compliance audit, proving that services can only communicate with explicitly authorized counterparts.

**What the DevOps/security team implements:**
1. Rolls out Istio with **PeerAuthentication in PERMISSIVE mode** initially across the whole cluster, migrating namespace by namespace, then switches each namespace to **STRICT mode** once confirmed fully migrated — reaching cluster-wide STRICT mTLS within a planned 6-week rollout.
2. For the `payments` namespace specifically (highest compliance sensitivity), implements a **default-deny AuthorizationPolicy**, then adds narrow, explicit ALLOW rules — for example, only the `checkout-service` service account may POST to `/api/charge`, and only an internal `reconciliation-job` service account may GET `/api/transactions/export`.
3. Configures **RequestAuthentication** validating end-user JWTs issued by their corporate identity provider for any service exposed to end-users, ensuring application code never has to trust an unvalidated token.
4. Uses `istioctl authn tls-check` as part of a **scripted compliance report**, run monthly, confirming every service-to-service connection in the mesh is genuinely using mTLS — providing concrete, automatically-generated evidence for auditors rather than a manual, error-prone attestation process.
5. Documents the **default-deny policy pattern** as a mandatory template for any new namespace onboarded to the mesh, preventing teams from accidentally leaving a namespace in an "allow everything" state during initial setup.

**Why this matters:** Zero-trust networking isn't just a security nice-to-have for regulated industries — it's often an explicit compliance requirement, and Istio's mTLS-by-default plus AuthorizationPolicy combination is what makes "prove that only authorized services can call payments" answerable with a policy file and an automated report, rather than a lengthy manual network audit.

---

Next: **04observability_resilience.md** — automatic telemetry, distributed tracing, and how Istio's data feeds directly into Grafana/Kiali dashboards.