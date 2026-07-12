# JMeter - Building Practical Test Plans

Hands-on: simulating a full multi-step user journey with correlation, parameterization, and session/cookie handling — the skills that separate a toy test plan from a realistic one.

## Table of Contents
- [The Correlation Problem](#the-correlation-problem)
- [Extracting Values with Regular Expression Extractor](#extracting-values-with-regular-expression-extractor)
- [Extracting Values with JSON Extractor](#extracting-values-with-json-extractor)
- [Handling Cookies and Sessions](#handling-cookies-and-sessions)
- [Parameterization Strategies](#parameterization-strategies)
- [Building a Full User Journey](#building-a-full-user-journey)
- [Controllers for Realistic Flows](#controllers-for-realistic-flows)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Correlation Problem

**Real applications return dynamic values (tokens, IDs, session keys) that must be reused in later requests.**

**Visual:**
```
Login Response:
{
  "auth_token": "abc123xyz789",   ← different every time!
  "user_id": 4821
}

Next Request (Get Orders) NEEDS this token:
GET /orders
Header: Authorization: Bearer abc123xyz789

Problem: You can't hardcode "abc123xyz789" in your test plan —
it's different for every login, every test run.

Solution: CORRELATION
Extract the token dynamically from Response 1,
store it as a variable, inject it into Request 2.
```

**Visual — the correlation flow:**
```
┌─────────────┐    extract    ┌──────────────┐    inject    ┌──────────────┐
│ Login Request│ ────────────→ │ ${auth_token} │ ───────────→ │ Orders Request │
│ (Sampler)     │   (Extractor) │  (variable)    │  (in header)  │  (Sampler)      │
└─────────────┘               └──────────────┘               └──────────────┘
```

---

## Extracting Values with Regular Expression Extractor

Used for text/HTML/plain responses.

```
Right-click Login Sampler → Add → Post Processors → Regular Expression Extractor

Reference Name: auth_token
Regular Expression: "auth_token":"(.*?)"
Template: $1$
Match No.: 1
```

**Visual:**
```
Response Body:
{"auth_token":"abc123xyz789","user_id":4821}
                ↑___________↑
                captured group $1$

Result stored as: ${auth_token} = abc123xyz789

Usage in next request:
Header: Authorization: Bearer ${auth_token}
```

---

## Extracting Values with JSON Extractor

**Preferred for JSON APIs — more robust than regex.**

```
Right-click Login Sampler → Add → Post Processors → JSON Extractor

Names of created variables: auth_token
JSON Path expressions: $.auth_token
```

**Visual:**
```
Response Body:
{
  "auth_token": "abc123xyz789",
  "user": { "id": 4821, "name": "Alice" }
}

JSON Path: $.auth_token         → auth_token = abc123xyz789
JSON Path: $.user.id            → user_id = 4821
JSON Path: $.user.name          → user_name = Alice

Why JSON Extractor > Regex for JSON APIs:
- Handles nested objects/arrays naturally
- Less fragile to whitespace/formatting changes
- Easier to read and maintain
```

---

## Handling Cookies and Sessions

Most web apps use cookies for session management — JMeter needs an explicit manager for this.

```
Right-click Test Plan → Add → Config Element → HTTP Cookie Manager
```

**Visual:**
```
Without Cookie Manager:
Request 1: Login → Server sets Set-Cookie: JSESSIONID=abc123
Request 2: Get Profile → No cookie sent → Server thinks unauthenticated ❌

With Cookie Manager:
Request 1: Login → Cookie Manager captures JSESSIONID=abc123
Request 2: Get Profile → Cookie Manager automatically attaches it ✓

Each virtual user (thread) gets its OWN cookie storage,
so 100 users don't share/collide on session state.
```

---

## Parameterization Strategies

**Visual:**
```
┌─────────────────────────────────────────────────────────┐
│ Strategy                    When to Use                     │
├─────────────────────────────────────────────────────────┤
│ CSV Data Set Config           Bulk external data (users,       │
│                              product IDs) from a file            │
│ User Defined Variables        Static config (base URL,           │
│                              environment name)                    │
│ __Random() function            Generate random numbers on-the-fly │
│ __time() function              Timestamps for uniqueness           │
│ Counter                        Sequential incrementing values       │
└─────────────────────────────────────────────────────────┘
```

**Function examples (used inline in fields with `${}`):**
```
${__Random(1000,9999,)}      → random number between 1000-9999
${__time(,)}                 → current timestamp in ms
${__UUID()}                  → random UUID (great for unique order IDs)
```

**Visual:**
```
Body Data field:
{
  "order_id": "ORDER-${__UUID()}",
  "quantity": ${__Random(1,5,)}
}

Result per virtual user:
User1: {"order_id": "ORDER-a1b2c3d4-...", "quantity": 3}
User2: {"order_id": "ORDER-e5f6g7h8-...", "quantity": 1}
```

---

## Building a Full User Journey

**Scenario: Login → Browse Products → Add to Cart → Checkout**

**Visual:**
```
Test Plan
 └── Thread Group (200 users, 60s ramp-up)
      ├── HTTP Cookie Manager (Config)
      ├── CSV Data Set Config (users.csv → username, password)
      │
      ├── [1] POST /login
      │      Body: {"username":"${username}","password":"${password}"}
      │      └── JSON Extractor → auth_token
      │
      ├── Constant Timer (2000ms)  ← think time
      │
      ├── [2] GET /products
      │      Header: Authorization: Bearer ${auth_token}
      │      └── JSON Extractor → product_id (first product returned)
      │
      ├── Constant Timer (3000ms)
      │
      ├── [3] POST /cart/add
      │      Body: {"product_id": ${product_id}, "quantity": 1}
      │      └── Response Assertion → contains "cart_updated"
      │
      ├── Constant Timer (2000ms)
      │
      └── [4] POST /checkout
             Header: Authorization: Bearer ${auth_token}
             └── Response Assertion → contains "order_confirmed"
```

Each of the 200 virtual users runs this exact sequence — logging in as a different CSV-driven user, extracting a fresh token, browsing an actual returned product ID (not hardcoded), and completing checkout.

---

## Controllers for Realistic Flows

Not every user follows the same path — Controllers let you model branching, realistic behavior.

**Visual:**
```
┌───────────────────────────────────────────────────────┐
│ Controller               Purpose                          │
├───────────────────────────────────────────────────────┤
│ If Controller              Conditional branching (e.g. only  │
│                           checkout if cart wasn't empty)       │
│ Loop Controller             Repeat a sub-sequence N times       │
│ Random Controller            Randomly pick one of several       │
│                           children (simulates varied behavior)  │
│ Throughput Controller         Only run this branch X% of the time│
│                           (e.g. 80% just browse, 20% checkout)   │
└───────────────────────────────────────────────────────┘
```

**Visual — realistic traffic mix using Throughput Controller:**
```
Thread Group (1000 users)
 ├── Throughput Controller "Browsers" (80%)
 │    └── Browse products only, no checkout
 │
 └── Throughput Controller "Buyers" (20%)
      └── Browse → Add to cart → Checkout

This mimics real e-commerce traffic:
most visitors browse, only a fraction actually buy —
testing 100% checkout traffic would be unrealistic
and would over-stress the payment service specifically.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is asked to load-test a newly redesigned checkout flow before a product launch, and the QA team reports the test results "don't match what we see in production monitoring."

**Root cause investigation and fix:**
1. Discovers the existing test plan **hardcoded a single auth token** captured once during test-plan creation — meaning every one of the 500 virtual users was hitting the API as the exact same authenticated user. This masked real concurrency bugs (e.g., a race condition in per-user cart state) because there was no actual per-user isolation happening.
2. Fixes this by adding a **CSV Data Set Config** with 500 real test accounts and a **JSON Extractor** to dynamically capture each user's own token at login.
3. Notices the test plan had **zero think-time** between requests, causing unrealistically high requests-per-second that doesn't match real user behavior — adds a **Gaussian Random Timer** calibrated from real production analytics (average time between page views).
4. Adds a **Throughput Controller** split (70% browse-only, 30% full checkout) after checking real conversion-funnel analytics, instead of testing 100% of virtual users completing checkout — which had been creating artificial, unrealistic load specifically on the payment gateway.
5. Re-runs the corrected test plan — the new results align far more closely with real production monitoring dashboards, and a genuine concurrency bug in the cart service is finally reproduced and fixed.

**Why this matters:** A load test that doesn't correlate dynamic values, vary user data, or reflect realistic traffic mix will give you confident-looking numbers that are simply wrong — arguably worse than not testing at all, because it creates false assurance.

---

Next: **04distributed_and_cli_execution.md** — running JMeter in non-GUI/CLI mode, distributed load testing across multiple machines, and CI/CD integration.