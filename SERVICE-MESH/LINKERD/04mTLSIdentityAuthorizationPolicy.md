# Linkerd Security - mTLS, Identity, Authorization Policy

How Linkerd secures service-to-service traffic automatically, and how to lock down access with policy - with visual diagrams.

## Table of Contents
- [Automatic mTLS](#automatic-mtls)
- [Identity - Trust Anchors and Issuer Certs](#identity---trust-anchors-and-issuer-certs)
- [Certificate Rotation](#certificate-rotation)
- [Verifying mTLS](#verifying-mtls)
- [Server Resource](#server-resource)
- [AuthorizationPolicy](#authorizationpolicy)
- [MeshTLSAuthentication and NetworkAuthentication](#meshtlsauthentication-and-networkauthentication)
- [Default-Deny Zero Trust Setup](#default-deny-zero-trust-setup)

---

## Automatic mTLS

**Every TCP connection between two meshed pods is automatically upgraded to mutual TLS - no configuration required.**

**Visual:**
```
Pod A                                    Pod B
┌─────────────────┐                    ┌─────────────────┐
│ app             │                    │ app             │
│   ↓↑            │                    │   ↓↑            │
│ linkerd-proxy   │══════mTLS══════════│ linkerd-proxy   │
│  (has cert A)   │  both sides verify │  (has cert B)   │
└─────────────────┘  each other's cert └─────────────────┘

Each proxy:
1. Presents its own short-lived certificate
2. Verifies the peer's certificate against the shared trust anchor
3. Encrypts all traffic (even plain HTTP becomes encrypted in transit)

App code sees plain HTTP - encryption is fully transparent
```

---

## Identity - Trust Anchors and Issuer Certs

**The `identity` control plane component issues short-lived (default 24h) TLS certs to every proxy.**

**Visual:**
```
┌─────────────────────────────────────────────┐
│           Trust Anchor (root CA)             │
│         valid for years, rarely rotates      │
└───────────────────┬───────────────────────────┘
                     │ signs
┌────────────────────▼──────────────────────────┐
│         Issuer Certificate                    │
│  (linkerd-identity-issuer, ~1 year validity)  │
└───────────────────┬───────────────────────────┘
                     │ signs (auto, every ~24h)
┌────────────────────▼──────────────────────────┐
│    Per-Proxy Leaf Certificates                 │
│  (linkerd-proxy in every pod, short-lived)     │
└─────────────────────────────────────────────┘

Default install auto-generates all of this.
Production best practice: bring your OWN trust anchor
via cert-manager so it can be rotated/audited centrally.
```

### Install with your own trust anchor (production pattern)

```bash
step certificate create root.linkerd.cluster.local ca.crt ca.key \
  --profile root-ca --no-password --insecure

step certificate create identity.linkerd.cluster.local issuer.crt issuer.key \
  --profile intermediate-ca --not-after 8760h --ca ca.crt --ca-key ca.key \
  --no-password --insecure

linkerd install \
  --identity-trust-anchors-file ca.crt \
  --identity-issuer-certificate-file issuer.crt \
  --identity-issuer-key-file issuer.key \
  | kubectl apply -f -
```

---

## Certificate Rotation

**cert-manager is the standard production tool to auto-rotate the issuer certificate before it expires.**

```bash
helm install linkerd-control-plane linkerd-edge/linkerd-control-plane \
  -n linkerd \
  --set identity.issuer.scheme=kubernetes.io/tls \
  --set-file identityTrustAnchorsPEM=ca.crt
```

**Visual:**
```
cert-manager
┌───────────────────────────────┐
│ Certificate: linkerd-identity- │
│              issuer            │
│ Renews automatically before    │
│ expiry, stores in a Secret     │
└───────────────┬───────────────┘
                │ watched by
┌───────────────▼───────────────┐
│ linkerd-identity control plane │
│ picks up renewed cert          │
│ automatically (no restart)     │
└────────────────────────────────┘

⚠️ Expired trust anchor = mesh-wide outage (all mTLS fails)
   Alerting on cert expiry is a MUST for production clusters
```

### Check certificate expiry

```bash
linkerd check --proxy
```

**Visual:**
```
linkerd-identity-data-plane
✓ data plane proxies certificate match CA
✓ trust anchors are within their validity period

If this check fails → certificates are expiring soon or mismatched
```

---

## Verifying mTLS

### linkerd viz edges

```bash
linkerd viz edges deploy -n my-app
```

**Output Example:**
```
SRC          DST        SECURED
frontend     backend    √
frontend     legacy     ×   ← not meshed, plaintext
```

### linkerd viz tap (inspect a live connection's TLS status)

```bash
linkerd viz tap deploy/backend -n my-app -o json | jq '.tls'
```

**Visual:**
```
√ = tls: "true" in tap output → connection is mTLS encrypted
× = tls: "no_identity" or "disabled" → plaintext, investigate
```

---

## Server Resource

**A `Server` resource describes a specific port on a set of pods that policy can be applied to (replaces older "unauthenticated" flags).**

```yaml
apiVersion: policy.linkerd.io/v1beta3
kind: Server
metadata:
  name: backend-http
  namespace: my-app
spec:
  podSelector:
    matchLabels:
      app: backend
  port: 8080
  proxyProtocol: HTTP/2
```

**Visual:**
```
Server = "this port, on these pods, is a thing I can write policy for"

Deployment: backend (pods labeled app=backend)
        │
        ▼
Server: backend-http (port 8080)
        │
        ▼
AuthorizationPolicy attaches HERE (see below)
```

---

## AuthorizationPolicy

**Defines WHO is allowed to call a `Server` - the core zero-trust building block.**

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: AuthorizationPolicy
metadata:
  name: backend-allow-frontend
  namespace: my-app
spec:
  targetRef:
    group: policy.linkerd.io
    kind: Server
    name: backend-http
  requiredAuthenticationRefs:
  - group: policy.linkerd.io
    kind: MeshTLSAuthentication
    name: frontend-only
```

**Visual:**
```
Without AuthorizationPolicy:
Any meshed pod ──→ backend:8080   (allowed by default, mesh-wide)

With AuthorizationPolicy (frontend-only):
frontend ──→ backend:8080         ✓ ALLOWED (matches MeshTLSAuthentication)
other-svc ──→ backend:8080        ✗ DENIED (connection refused)

Enforced at the DESTINATION proxy - backend's own sidecar
rejects unauthorized callers before traffic reaches the app.
```

---

## MeshTLSAuthentication and NetworkAuthentication

**Define WHO counts as "authorized" - by mesh identity (ServiceAccount) or by network/IP.**

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: MeshTLSAuthentication
metadata:
  name: frontend-only
  namespace: my-app
spec:
  identities:
  - "frontend.my-app.serviceaccount.identity.linkerd.cluster.local"
```

```yaml
apiVersion: policy.linkerd.io/v1alpha1
kind: NetworkAuthentication
metadata:
  name: internal-networks
  namespace: my-app
spec:
  networks:
  - cidr: "10.0.0.0/8"
```

**Visual:**
```
MeshTLSAuthentication → identity-based ("only ServiceAccount frontend")
NetworkAuthentication → IP/CIDR-based ("only from internal subnet")

Real production pattern: identity-based auth is stronger since
it survives pod IP changes/rescheduling; use CIDR only for
legacy/unmeshed clients that must reach a meshed service.
```

---

## Default-Deny Zero Trust Setup

**Lock down a namespace so nothing is reachable unless explicitly authorized - the recommended production posture.**

```bash
linkerd install --default-inbound-policy deny | kubectl apply -f -
```

**Visual:**
```
Default policy options:
all-unauthenticated   → allow all traffic, even unmeshed (least secure)
all-authenticated      → allow any meshed client (mTLS required, default)
cluster-authenticated  → allow only clients inside the cluster
deny                   → allow nothing until explicit AuthorizationPolicy exists (most secure)

Zero-trust flow:
1. Set default-inbound-policy: deny (cluster-wide or per-namespace annotation)
2. Every Service starts with NO access
3. Explicitly grant access via Server + AuthorizationPolicy per service
4. Result: fully auditable, least-privilege service-to-service access
```

### Per-namespace override

```yaml
metadata:
  annotations:
    config.linkerd.io/default-inbound-policy: "deny"
```

---

## Visual Summary

```
1. mTLS - automatic, zero config
   Every meshed connection encrypted + mutually verified

2. Identity - trust anchor → issuer → per-proxy leaf certs
   Use cert-manager + your own PKI in production

3. Verify
   linkerd viz edges deploy -n ns
   linkerd check --proxy

4. Authorize (zero trust)
   Server            → defines the port/pods
   MeshTLSAuthentication → defines WHO (identity/ServiceAccount)
   AuthorizationPolicy   → ties them together (WHO can call WHAT)

5. Harden
   default-inbound-policy: deny
   → nothing reachable unless explicitly authorized
```

---

This guide covers Linkerd's security model - automatic mTLS, identity/certificate management, and authorization policy for zero-trust production clusters, with visual representations of each concept.