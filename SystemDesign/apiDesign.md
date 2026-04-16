# 🔌 API Design

> **Series:** System Design Notes  
> **Module:** 08 — API Design  
> **Prerequisites:** `02_load_balancing.md`, `03_caching.md`, `07_message_queues.md`, Basic HTTP knowledge

---

## 📌 Table of Contents

1. [What is an API?](#1-what-is-an-api)
2. [REST — Deep Dive](#2-rest--deep-dive)
3. [GraphQL — Deep Dive](#3-graphql--deep-dive)
4. [gRPC — Deep Dive](#4-grpc--deep-dive)
5. [WebSockets & Server-Sent Events](#5-websockets--server-sent-events)
6. [REST vs GraphQL vs gRPC vs WebSocket](#6-rest-vs-graphql-vs-grpc-vs-websocket)
7. [API Gateway](#7-api-gateway)
8. [Authentication & Authorization](#8-authentication--authorization)
9. [Rate Limiting](#9-rate-limiting)
10. [API Versioning](#10-api-versioning)
11. [Pagination](#11-pagination)
12. [Error Handling](#12-error-handling)
13. [Idempotency in APIs](#13-idempotency-in-apis)
14. [Real-World Architectures](#14-real-world-architectures)
15. [Common Mistakes](#15-common-mistakes)
16. [Interview Cheatsheet](#16-interview-cheatsheet)

---

## 1. What is an API?

> **Definition:** An API (Application Programming Interface) is a contract that defines how two systems communicate — what requests are valid, what responses look like, and what guarantees the provider makes.

```
Client                   API Contract               Server
  │                      ─────────────               │
  │── "GET /users/42" ──────────────────────────────►│
  │                                                  │  (look up user 42)
  │◄──── 200 OK { id: 42, name: "Satyam" } ──────────│

The API contract says:
  - What URL structure to use
  - What HTTP methods are valid
  - What request body/headers to send
  - What response format to expect
  - What error codes mean what
```

**Why API design matters:**

| Bad API | Good API |
|---|---|
| Inconsistent naming (`/getUser`, `/fetch_orders`) | Consistent nouns (`/users`, `/orders`) |
| No versioning → breaking clients silently | `/v1/`, `/v2/` versioned from day one |
| No pagination → returns 10M rows | Cursor-based pagination, max page size |
| Vague errors (`"error: true"`) | Structured errors with codes and messages |
| No rate limiting → DDoS vector | Rate limit headers + 429 responses |

---

## 2. REST — Deep Dive

> **REST (Representational State Transfer)** is an architectural style (not a protocol) defined by Roy Fielding in 2000. It uses standard HTTP methods to perform operations on *resources* identified by URLs.

### 2.1 Core Constraints

```
1. Stateless      → Server stores NO client session state.
                    Every request contains all info needed.
                    ✅ Enables horizontal scaling (any server handles any request)

2. Client-Server  → UI and data are separated. Client doesn't care about DB.
                    Server doesn't care about UI rendering.

3. Cacheable      → Responses must declare if they can be cached (Cache-Control).
                    ✅ Enables CDN, browser, proxy caching

4. Uniform Interface → Consistent URL structure + HTTP semantics across all resources.

5. Layered System → Client can't tell if it's talking to origin, LB, or CDN.

6. Code-on-Demand → Optional: server can send executable code (e.g., JavaScript)
```

---

### 2.2 HTTP Methods & Semantics

| Method | Meaning | Idempotent? | Safe? | Body? |
|---|---|---|---|---|
| `GET` | Read a resource | ✅ Yes | ✅ Yes | No |
| `POST` | Create a resource | ❌ No | ❌ No | Yes |
| `PUT` | Replace entire resource | ✅ Yes | ❌ No | Yes |
| `PATCH` | Update partial resource | ❌ No* | ❌ No | Yes |
| `DELETE` | Delete a resource | ✅ Yes | ❌ No | No |
| `HEAD` | GET without response body | ✅ Yes | ✅ Yes | No |
| `OPTIONS` | What methods are supported? | ✅ Yes | ✅ Yes | No |

> **Safe** = doesn't modify data  
> **Idempotent** = calling N times = calling once (same result)

---

### 2.3 Resource Naming — The Rules

```
✅ GOOD — Nouns, plural, lowercase, hierarchical:
  GET    /users              → list all users
  GET    /users/42           → get user 42
  POST   /users              → create a user
  PUT    /users/42           → replace user 42
  PATCH  /users/42           → update fields of user 42
  DELETE /users/42           → delete user 42

  GET    /users/42/orders    → orders belonging to user 42
  GET    /users/42/orders/7  → order 7 of user 42

❌ BAD — Verbs in URLs (anti-pattern):
  GET  /getUser
  POST /createOrder
  GET  /fetchUserOrders
  POST /deleteUser/42   ← wrong method AND wrong naming
```

**For actions that don't map cleanly to CRUD:**

```
POST /orders/42/cancel      ← "cancel" as sub-resource
POST /payments/42/refund    ← noun-ish actions as sub-resources
POST /users/42/password/reset
POST /reports/generate      ← if it's a long-running action
```

---

### 2.4 HTTP Status Codes — The Full Map

```
2xx — Success
  200 OK             → Standard success (GET, PUT, PATCH)
  201 Created        → Resource created (POST) — include Location header
  202 Accepted       → Async operation accepted, not yet complete
  204 No Content     → Success, no body (DELETE, PATCH with no return)

3xx — Redirection
  301 Moved Permanently   → Resource has new URL (SEO-safe redirect)
  304 Not Modified        → Cached version is still valid (ETag match)

4xx — Client Errors (your fault)
  400 Bad Request         → Malformed request, invalid input
  401 Unauthorized        → Not authenticated (no valid token)
  403 Forbidden           → Authenticated but not authorized for this
  404 Not Found           → Resource doesn't exist
  405 Method Not Allowed  → Wrong HTTP method for this URL
  409 Conflict            → State conflict (duplicate, version mismatch)
  410 Gone                → Resource permanently deleted (vs 404 = maybe)
  422 Unprocessable       → Valid syntax but semantically invalid
  429 Too Many Requests   → Rate limit exceeded
  
5xx — Server Errors (our fault)
  500 Internal Server Error  → Unhandled exception (generic)
  502 Bad Gateway            → Upstream service returned invalid response
  503 Service Unavailable    → Overloaded / maintenance / circuit open
  504 Gateway Timeout        → Upstream service timed out
```

> **Interview tip:** `401 vs 403` — **401 = who are you?** (auth failed), **403 = I know who you are, but no** (permission denied).

---

### 2.5 Request & Response Design

**Request anatomy:**

```
GET /users/42/orders?status=pending&limit=10 HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJSUzI1NiJ9...
Accept: application/json
X-Request-ID: f47ac10b-58cc-4372-a567-0e02b2c3d479

       ├─ Path param: /users/42         → identifies the resource
       ├─ Query param: ?status=pending  → filter/sort/paginate
       └─ Headers: auth, content type, tracing
```

**Response anatomy:**

```json
HTTP/1.1 200 OK
Content-Type: application/json
X-Request-ID: f47ac10b-58cc-4372-a567-0e02b2c3d479
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 847
X-RateLimit-Reset: 1704067200

{
  "data": [
    { "id": "ord-7", "status": "pending", "total": 99.99 },
    { "id": "ord-8", "status": "pending", "total": 14.50 }
  ],
  "pagination": {
    "cursor": "eyJpZCI6Im9yZC04In0=",
    "has_next": true,
    "limit": 10
  }
}
```

---

### 2.6 REST Caching

REST's biggest advantage over GraphQL for public APIs: **HTTP caching works out of the box**.

```
GET /products/42
  → Cacheable at browser, CDN, proxy level
  → Cache-Control: public, max-age=300

POST /orders
  → Never cached (mutating)

GET /users/42/recommendations  (personalised)
  → Cache-Control: private, max-age=60
```

---

## 3. GraphQL — Deep Dive

> **GraphQL** is a query language for APIs (not a protocol) developed by Facebook in 2012, open-sourced 2015. Clients declare *exactly* what data they need in a typed schema — no over-fetching, no under-fetching.

### 3.1 Core Concept — Schema First

```graphql
# Schema defines the contract — shared by client and server
type User {
  id: ID!
  name: String!
  email: String!
  orders: [Order!]!
}

type Order {
  id: ID!
  total: Float!
  status: String!
  createdAt: String!
}

type Query {
  user(id: ID!): User
  orders(userId: ID!, status: String): [Order!]!
}

type Mutation {
  createOrder(userId: ID!, items: [ItemInput!]!): Order!
  cancelOrder(id: ID!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

---

### 3.2 Over-fetching vs Under-fetching — The REST Problem GraphQL Solves

```
PROBLEM — REST over-fetching:
  GET /users/42
  Returns: { id, name, email, address, phone, dob, preferences, ... }
  App only needs: { id, name }
  → Wasted bandwidth, especially on mobile

PROBLEM — REST under-fetching (N+1 problem):
  App needs: user info + their last 5 orders + each order's items
  REST:
    GET /users/42                           (1 request)
    GET /users/42/orders?limit=5            (1 request)
    GET /orders/1/items                     (1 request)
    GET /orders/2/items                     (1 request)
    GET /orders/3/items                     (1 request)
    GET /orders/4/items                     (1 request)
    GET /orders/5/items                     (1 request)
  = 7 round trips → slow on mobile, chatty on server

GRAPHQL SOLUTION:
  Single query, get exactly what you need:

  query {
    user(id: "42") {
      name
      orders(limit: 5) {
        id
        total
        items {
          name
          qty
        }
      }
    }
  }
  = 1 round trip, exactly the fields needed ✅
```

---

### 3.3 Operations

```graphql
# QUERY — Read data
query GetUserProfile($userId: ID!) {
  user(id: $userId) {
    name
    email
    orders {
      id
      status
    }
  }
}

# MUTATION — Write data
mutation PlaceOrder($input: OrderInput!) {
  createOrder(input: $input) {
    id
    total
    status
  }
}

# SUBSCRIPTION — Real-time push (WebSocket under the hood)
subscription TrackOrder($orderId: ID!) {
  orderStatusChanged(orderId: $orderId) {
    id
    status
    updatedAt
  }
}
```

---

### 3.4 GraphQL Tradeoffs

| ✅ Pros | ❌ Cons |
|---|---|
| Client requests exactly what it needs | HTTP caching is hard (all POST to `/graphql`) |
| Single endpoint for all operations | N+1 problem if resolvers aren't batched (use DataLoader) |
| Strongly typed schema = self-documenting | Complex queries can hammer DB (need query depth/cost limits) |
| Great for mobile (minimal payload) | Higher server CPU than REST |
| Schema introspection (tooling is great) | Schema introspection can leak internal structure |
| Easy to evolve without versioning | Steeper learning curve |

**The N+1 Problem and DataLoader fix:**

```javascript
// NAIVE RESOLVER — triggers N DB queries for N orders
const resolvers = {
  Order: {
    user: (order) => db.users.findById(order.userId)  // N queries!
  }
}

// DATALOADER — batches all user lookups into ONE query
const userLoader = new DataLoader(async (userIds) => {
  const users = await db.users.findAll({ where: { id: userIds } });
  return userIds.map(id => users.find(u => u.id === id));
});

const resolvers = {
  Order: {
    user: (order) => userLoader.load(order.userId)  // batched ✅
  }
}
```

**Query depth limiting (prevent abuse):**

```javascript
// Protect against deeply nested malicious queries:
// query { user { orders { items { product { reviews { user { orders { ... } } } } } } } }

const depthLimit = require('graphql-depth-limit');
app.use('/graphql', graphqlHTTP({
  schema,
  validationRules: [depthLimit(5)]  // max 5 levels deep
}));
```

---

## 4. gRPC — Deep Dive

> **gRPC** (Google Remote Procedure Call) is a high-performance, contract-first RPC framework using **HTTP/2** as transport and **Protocol Buffers** as serialization. Designed for microservice-to-microservice communication.

### 4.1 How gRPC Works

```
1. Define service contract in .proto file
2. Generate server + client code (supports 10+ languages)
3. Client calls remote method as if it were a local function
4. Data serialized as binary Protobuf (not JSON)
5. Transmitted over HTTP/2 (multiplexed, persistent connection)
```

```protobuf
// user.proto — the contract
syntax = "proto3";

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc CreateUser (CreateUserRequest) returns (User);
  rpc StreamUserEvents (StreamRequest) returns (stream UserEvent);   // server streaming
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);        // bidirectional streaming
}

message GetUserRequest {
  string user_id = 1;
}

message User {
  string id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
}
```

```bash
# Generate client + server code
protoc --go_out=. --go-grpc_out=. user.proto
# Generates: user.pb.go (messages) + user_grpc.pb.go (service stubs)
```

---

### 4.2 gRPC Streaming Modes

```
1. UNARY (standard request-response):
   Client ──request──► Server ──response──► Client
   Like a regular function call.

2. SERVER STREAMING:
   Client ──request──► Server ──[stream of responses]──► Client
   Use: Live dashboard, large dataset download, event feed

3. CLIENT STREAMING:
   Client ──[stream of requests]──► Server ──response──► Client
   Use: File upload, IoT telemetry ingestion, batch processing

4. BIDIRECTIONAL STREAMING:
   Client ◄──[messages]──► Server  (full duplex)
   Use: Real-time chat, collaborative editing, gaming
```

---

### 4.3 Protocol Buffers vs JSON

```
JSON (REST):
  { "id": "42", "name": "Satyam", "age": 25 }
  → 36 bytes (text, human-readable, self-describing)

Protobuf (gRPC):
  [binary: 0a 02 34 32 12 06 53 61 74 79 61 6d 18 19]
  → ~14 bytes (binary, NOT human-readable, schema required)
  → ~60% smaller payload ✅
  → 5–7x faster serialization ✅
```

---

### 4.4 gRPC Tradeoffs

| ✅ Pros | ❌ Cons |
|---|---|
| 10x lower latency than REST (25ms vs 250ms) | No native browser support (needs gRPC-Web proxy) |
| Binary Protobuf: 30–50% smaller payload | Binary data = hard to debug (need grpcurl, BloomRPC) |
| HTTP/2: multiplexing, header compression | Steeper learning curve (proto files, codegen) |
| Bi-directional streaming built-in | Requires same .proto file on both sides |
| Strongly typed — catch errors at compile time | Less flexible than REST for public APIs |
| Code generation = no hand-rolled clients | Hard to test with simple curl |

---

## 5. WebSockets & Server-Sent Events

### WebSockets — Full Duplex

> A persistent, **bidirectional** TCP connection between client and server. Both sides can send messages at any time without a new HTTP request.

```
HTTP Upgrade handshake:
  Client: GET /ws HTTP/1.1
          Upgrade: websocket
          Connection: Upgrade
          
  Server: 101 Switching Protocols
          Upgrade: websocket

After handshake:
  Client ◄──────────────────────────► Server
         (full duplex, no req/res cycle)
```

**Use cases:**
- Live chat (WhatsApp Web, Slack, Discord)
- Real-time collaboration (Google Docs, Figma)
- Live game state
- Financial tickers (live price feeds)
- Live dashboards

**Scaling challenge:**

```
WebSocket connections are STATEFUL.
Each user is permanently connected to ONE server.

Without sticky routing:
  User A connected to Server 1
  User B connected to Server 2
  User A sends message to User B
  Server 1 doesn't know where User B's socket is! ❌

Solution: Pub/Sub backplane (Redis)
  All servers publish/subscribe to a shared Redis channel.
  Server 1 publishes message → Redis → Server 2 → User B ✅
```

---

### Server-Sent Events (SSE)

> **Unidirectional** push from server to client over a persistent HTTP connection. Simpler than WebSockets.

```
Client subscribes:
  GET /events HTTP/1.1
  Accept: text/event-stream

Server pushes:
  data: {"type": "order_update", "status": "shipped"}\n\n
  data: {"type": "price_change", "product": "42", "price": 99}\n\n

  (server keeps connection open, sends events as they occur)
```

| Feature | WebSocket | SSE |
|---|---|---|
| Direction | Bidirectional | Server → Client only |
| Protocol | Custom WS protocol | Plain HTTP |
| Auto-reconnect | Manual | ✅ Built-in |
| Browser support | ✅ Universal | ✅ Universal |
| Load balancer compat | Needs sticky sessions | ✅ Works fine |
| Use case | Chat, games, collab | Notifications, feeds, dashboards |

> **Rule:** If you only need server → client push, use **SSE** — simpler, works with HTTP/2, auto-reconnects. Use **WebSocket** only if you need bidirectional real-time communication.

---

## 6. REST vs GraphQL vs gRPC vs WebSocket

| Feature | REST | GraphQL | gRPC | WebSocket |
|---|---|---|---|---|
| **Transport** | HTTP/1.1, HTTP/2 | HTTP (POST) | HTTP/2 | TCP (WS upgrade) |
| **Data format** | JSON/XML | JSON | Protobuf (binary) | Any |
| **Contract** | Informal (OpenAPI) | Schema (SDL) | Strict (.proto) | None |
| **Latency** | Medium (~250ms) | Medium (~180ms) | Low (~25ms) | Very Low |
| **Throughput** | 20K req/s | 15K req/s | 50K req/s | Continuous |
| **Caching** | ✅ HTTP native | ❌ Hard (all POST) | ❌ Not HTTP | ❌ No |
| **Browser support** | ✅ Native | ✅ Native | ⚠️ Needs proxy | ✅ Native |
| **Streaming** | ❌ No | Subscriptions only | ✅ Full streaming | ✅ Full duplex |
| **Over/under-fetch** | ⚠️ Yes | ❌ No | ❌ No (typed) | N/A |
| **Debugging** | Easy (curl, Postman) | GraphiQL | Needs grpcurl | Needs WS tools |
| **Best for** | Public APIs, CRUD | Mobile, complex UI | Microservices | Real-time apps |

### Decision Tree

```
Need real-time bidirectional communication? (chat, gaming, collab)
  └── WebSocket (or SSE for server-push only)

Is this a public-facing API that 3rd parties will consume?
  └── REST (universal compatibility, easy tooling, HTTP caching)

Client needs to compose complex queries across many resources?
  └── GraphQL (mobile apps, BFF pattern, dashboards)

Internal microservice-to-microservice communication?
  └── gRPC (low latency, strongly typed, streaming support)

Need to stream large data or events between services?
  └── gRPC (server/client/bidirectional streaming)

Building a BFF (Backend for Frontend)?
  └── GraphQL (aggregate multiple services, tailored to each client)
```

---

## 7. API Gateway

> **Definition:** An API Gateway is a single entry point for all client requests in a microservices architecture. It handles cross-cutting concerns — routing, auth, rate limiting, logging — so individual services don't have to.

### What an API Gateway Does

```
┌─────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY                                 │
│                                                                     │
│  Incoming Request Pipeline:                                         │
│  ─────────────────────────────────────────────────────────────      │
│  TLS Termination                                                    │
│    → Authentication (validate JWT / API key)                        │
│    → Authorization  (check scopes / roles)                          │
│    → Rate Limiting  (check quota, reject if exceeded)               │
│    → Request Validation (schema, required fields)                   │
│    → Request Transformation (add/strip headers, reformat body)      │
│    → Routing (which backend service handles this?)                  │
│    → Load Balancing (which instance of that service?)               │
│    → ──► Backend Service                                            │
│                                                                     │
│  Outgoing Response Pipeline:                                        │
│  ──────────────────────────────────────────────────────────         │
│  Response Transformation (standardise format)                       │
│    → Cache Response (for GET requests)                              │
│    → Add rate limit headers                                         │
│    → Logging + Tracing (emit to observability stack)                │
│    → ──► Client                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Path-Based Routing

```
API Gateway routes by path to different backend services:

  /api/users/*       → User Service    (EC2 / ECS)
  /api/orders/*      → Order Service   (EC2 / ECS)
  /api/payments/*    → Payment Service (EC2 / ECS)
  /api/products/*    → Product Service (EC2 / ECS)
  /graphql           → GraphQL Gateway (Apollo)
  /ws/*              → WebSocket Service

Single domain: api.example.com
Client never knows the individual service addresses.
```

### BFF — Backend for Frontend Pattern

> Instead of one generic API gateway, create separate gateways tailored to each client type.

```
                         ┌─ [ Web BFF ]    ──► Returns rich HTML-ready data
Web Browser ────────────►│
                         │
Mobile App ─────────────►├─ [ Mobile BFF ] ──► Returns minimal JSON (battery/bandwidth)
                         │
Partner API ────────────►└─ [ Partner BFF ]──► Returns versioned, contractual API

All BFFs aggregate from same backend microservices.
Each is optimised for its client's needs.
```

**Used by:** Netflix (separate APIs for TV, mobile, web), Spotify, Amazon

### Gateway Providers

| Provider | Type | Notes |
|---|---|---|
| **AWS API Gateway** | Managed | REST + HTTP + WebSocket APIs, Lambda integration |
| **Kong** | Open-source / Cloud | Plugin ecosystem, high performance |
| **NGINX** | Open-source | Lightweight, battle-tested reverse proxy |
| **Envoy** | Open-source | Service mesh, gRPC, xDS, Istio sidecar |
| **Traefik** | Open-source | Container-native, auto-discovery |
| **Istio** | Service mesh | mTLS, observability, traffic splitting |
| **Cloudflare Gateway** | Cloud | Edge-based, DDoS included |

---

## 8. Authentication & Authorization

### Auth Methods Overview

```
Who are you?      → Authentication (AuthN)
What can you do?  → Authorization  (AuthZ)
```

### API Keys

```
Simple long-lived secret string, sent in header:
  Authorization: Bearer sk_live_abc123def456xyz

Server lookup: db.lookup(api_key) → get client identity + permissions

✅ Simple for server-to-server calls
✅ Easy to revoke
❌ Not user-specific
❌ No expiry (until manually rotated)
❌ If leaked, full access until revoked
```

---

### JWT (JSON Web Token)

> Stateless, self-contained token. Server encodes claims in a signed token. No DB lookup needed on each request.

```
JWT structure: header.payload.signature
  (base64url encoded, dot-separated)

Header:  { "alg": "RS256", "typ": "JWT" }
Payload: {
  "sub": "user-42",
  "name": "Satyam Shivam",
  "roles": ["user"],
  "iat": 1700000000,    ← issued at
  "exp": 1700003600     ← expires at (1 hour)
}
Signature: RS256(base64(header) + "." + base64(payload), private_key)
```

```
Validation flow:
  Request arrives with: Authorization: Bearer <jwt>
    ↓
  API Gateway / Middleware:
    1. Decode JWT (base64)
    2. Verify signature with PUBLIC key (no DB lookup) ✅
    3. Check exp claim — is token expired?
    4. Check sub, roles — does user have access to this endpoint?
    ↓
  If valid → pass to service
  If invalid → 401 Unauthorized
```

**JWT Tradeoffs:**

| ✅ Pros | ❌ Cons |
|---|---|
| Stateless — no DB lookup per request | Hard to revoke before expiry |
| Works across services (shared public key) | Token grows with more claims |
| Self-contained (claims embedded) | Must use short expiry + refresh tokens |

**Short-lived access + refresh token pattern:**

```
Access token:  TTL = 15 minutes (short)
Refresh token: TTL = 7 days    (long, stored in httpOnly cookie)

User authenticates → gets both tokens.
15 min later: access token expires.
Client uses refresh token → gets new access token.
If refresh token compromised → revoke in DB (small revocation list).
```

---

### OAuth 2.0

> Delegation protocol — allows a user to grant a 3rd-party app access to their data on another service WITHOUT giving the 3rd party their password.

```
Flows:
  Authorization Code (PKCE) → Web/Mobile apps where user logs in via provider
  Client Credentials         → Server-to-server (no user involved)
  Device Flow                → TV/CLI apps with no browser

Authorization Code flow:
  User clicks "Login with Google"
    → App redirects to Google with client_id + scope
    → User logs in at Google, consents
    → Google redirects back with auth_code
    → App exchanges code for access_token + refresh_token
    → App uses access_token to call Google APIs

scope examples:
  "read:profile"    → read user profile
  "write:orders"    → create orders
  "admin:*"         → full admin access
```

**Which method to use:**

| Scenario | Auth Method |
|---|---|
| Server-to-server internal API | API Key or mTLS |
| User logs in to your app | JWT (issued by your auth service) |
| 3rd party app accesses user data | OAuth 2.0 (Authorization Code + PKCE) |
| Service mesh internal traffic | mTLS (mutual TLS) |
| Microservice calling microservice | JWT (service-to-service scoped token) |

---

## 9. Rate Limiting

> Rate limiting controls how many requests a client can make in a time window. Protects against DDoS, abuse, and runaway clients.

### Algorithms

#### Token Bucket ⭐ Most Common

```
Each client has a bucket with capacity N.
Bucket refills at rate R tokens/second.
Each request costs 1 token.
If bucket is empty → 429 Too Many Requests.

capacity = 100 tokens
refill rate = 10 tokens/sec

t=0:   bucket=100
t=1:   5 requests → bucket=95 (+10 refill) → 100 (capped at max)
t=10:  burst of 100 requests → bucket=0
t=11:  request arrives → bucket has 10 tokens (refilled) → allow 10, reject rest
```

✅ Allows short bursts  
✅ Smooth average rate enforced  

---

#### Fixed Window Counter

```
Divide time into fixed windows (e.g., 1 minute).
Count requests per client per window.
If count > limit → 429.

Window: 12:00:00 – 12:00:59
  Client A: 98 requests → OK
  Client A: 99th request → OK
  Client A: 100th request → 429 ❌

At 12:01:00 → new window, counter resets.
```

❌ Thundering herd at window boundary: client can make 100 req at 12:00:59 and 100 at 12:01:00 = 200 in 2 seconds

---

#### Sliding Window Log

```
Keep a log of timestamps for each request.
On new request: remove timestamps older than window.
If log size > limit → 429.

More accurate than fixed window.
Higher memory: stores all timestamps.
```

---

#### Leaky Bucket

```
Requests enter bucket at any rate.
Bucket drains at fixed rate (smoothing).
If bucket full → 429.

Smooths traffic to a constant output rate.
Good for protecting backends that can only handle steady rates.
```

---

### Rate Limit Headers (Return These Always)

```http
HTTP/1.1 200 OK
X-RateLimit-Limit:     1000        ← total requests allowed per window
X-RateLimit-Remaining: 847         ← requests left in current window
X-RateLimit-Reset:     1704067200  ← Unix timestamp when window resets

HTTP/1.1 429 Too Many Requests
Retry-After: 30                    ← seconds until client can retry
X-RateLimit-Limit:     1000
X-RateLimit-Remaining: 0
```

---

### Rate Limit Granularity

```
By IP:          Simple, but shared IPs (NAT, corporate) penalize innocent users
By API Key:     Better for server-to-server
By User ID:     Best for authenticated users
By Endpoint:    /search has lower limit than /users (expensive operations)

Tiered limits:
  Free tier:    100 req/min
  Pro tier:    1000 req/min
  Enterprise: 10000 req/min
```

**Redis implementation (sliding window):**

```python
def is_rate_limited(client_id: str, limit: int = 100, window: int = 60) -> bool:
    key = f"rate:{client_id}"
    now = time.time()
    
    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, now - window)   # remove old entries
    pipe.zadd(key, {str(now): now})               # add current request
    pipe.zcard(key)                               # count in window
    pipe.expire(key, window)                      # auto-cleanup
    _, _, count, _ = pipe.execute()
    
    return count > limit   # True = rate limited
```

---

## 10. API Versioning

> Versioning lets you evolve your API without breaking existing clients.

### When to Version

```
NOT required (backward-compatible changes):
  ✅ Adding new optional fields to response
  ✅ Adding new optional request parameters
  ✅ Adding new endpoints
  ✅ Performance improvements

REQUIRED (breaking changes):
  ❌ Removing or renaming fields
  ❌ Changing field types
  ❌ Changing URL structure
  ❌ Changing HTTP method for an endpoint
  ❌ Changing error response format
```

### Versioning Strategies

#### URL Path Versioning ⭐ Recommended

```
GET /v1/users/42
GET /v2/users/42

✅ Explicit and visible
✅ Easy to test (just change URL)
✅ Easy to route at gateway level
✅ Can be bookmarked, logged, and cached easily
❌ URL "pollution"
```

#### Header Versioning

```
GET /users/42
Accept: application/vnd.myapi.v2+json
  OR
API-Version: 2

✅ Clean URLs
❌ Hard to test without custom headers
❌ Can't bookmark / share easily
❌ Not visible in logs by default
```

#### Query Parameter Versioning

```
GET /users/42?version=2

✅ Simple
❌ Can interfere with caching
❌ Easy to forget, inconsistent
```

### Versioning Lifecycle

```
v1 launched  ─────────────────────────────────────────────────────►
                    v2 launched  ──────────────────────────────────►
                                 v1 deprecated (announce sunset) ──►
                                                     v1 sunset (410 Gone)
                                        v3 launched ────────────────►

Best practices:
  ✅ Announce deprecation 6–12 months before sunset
  ✅ Send Deprecation headers on old versions
  ✅ Monitor old-version traffic to know when it's safe to kill
  ✅ Return 410 Gone (not 404) when version is fully removed
```

**Deprecation headers:**

```http
HTTP/1.1 200 OK
Deprecation: true
Sunset: Sat, 01 Jun 2025 00:00:00 GMT
Link: <https://api.example.com/v2/users>; rel="successor-version"
```

---

## 11. Pagination

> Never return unbounded lists. Pagination protects your DB and your clients.

### Offset Pagination

```
GET /orders?page=3&limit=20
GET /orders?offset=40&limit=20

SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 40;

Response:
{
  "data": [...],
  "pagination": {
    "page": 3,
    "limit": 20,
    "total": 1500,
    "pages": 75
  }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| Can jump to any page | Performance degrades at high offsets (DB scans all preceding rows) |
| Easy to implement | Results shift if data is inserted/deleted between pages (page drift) |
| User-familiar (page numbers) | Not suitable for real-time data |

**Best for:** Small-medium datasets with no real-time updates

---

### Cursor Pagination ⭐ Production Standard

```
GET /orders?limit=20                         ← first page
GET /orders?cursor=eyJpZCI6MjB9&limit=20     ← next page

Cursor = opaque token encoding position (e.g., base64 of last item's ID)

SELECT * FROM orders WHERE id > $last_id ORDER BY id LIMIT 20;

Response:
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9",
    "has_next": true,
    "limit": 20
  }
}
```

| ✅ Pros | ❌ Cons |
|---|---|
| O(1) performance regardless of dataset size | Can't jump to arbitrary page |
| No page drift (new inserts don't affect cursor) | Less intuitive for users |
| Scales to millions of rows | |

**Best for:** Large datasets, real-time data feeds, social media timelines

---

### Keyset Pagination

```
GET /orders?after_id=500&limit=20

WHERE id > 500 ORDER BY id LIMIT 20;

Visible key (ID or timestamp) used as page marker.
Similar to cursor but key is not opaque.
```

---

### Comparison Table

| Type | Performance | Jump to Page | Real-time Safe | Best For |
|---|---|---|---|---|
| **Offset** | Degrades at scale | ✅ Yes | ❌ No | Admin UIs, small datasets |
| **Cursor** | Constant O(1) | ❌ No | ✅ Yes | Feeds, timelines, large data |
| **Keyset** | Constant O(1) | ❌ No | ✅ Yes | Same as cursor, visible key |

---

## 12. Error Handling

> Errors should be machine-readable, human-readable, actionable, and consistent.

### Error Response Structure

```json
HTTP/1.1 400 Bad Request
Content-Type: application/problem+json

{
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": [
    {
      "field": "email",
      "code": "INVALID_FORMAT",
      "message": "Must be a valid email address"
    },
    {
      "field": "age",
      "code": "OUT_OF_RANGE",
      "message": "Must be between 18 and 120"
    }
  ],
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "documentation_url": "https://docs.example.com/errors/VALIDATION_ERROR",
  "timestamp": "2024-01-01T12:00:00Z"
}
```

**Rules:**
- `status` — HTTP status code (redundant in body for ease of parsing)
- `error` — machine-readable code (SCREAMING_SNAKE_CASE)
- `message` — human-readable description
- `details` — field-level validation errors (for 400s)
- `request_id` — for log correlation (user can send to support)
- **NEVER expose:** stack traces, DB query details, internal service names, file paths

---

## 13. Idempotency in APIs

> Making mutating operations safe to retry without side effects.

```
POST /payments  { amount: 100, user: 42 }
  → Network timeout → Did it go through? Unknown.
  → Client retries...
  → User charged twice? ❌

With idempotency key:
  POST /payments
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  { amount: 100, user: 42 }

Server behavior:
  First request:  process → store result against key → return 201
  Retry request:  look up key → return SAME 201 response → no double charge ✅
```

```python
def create_payment(idempotency_key: str, amount: float):
    # Check if already processed
    cached = redis.get(f"idem:{idempotency_key}")
    if cached:
        return json.loads(cached)  # Return previous result, do NOT reprocess
    
    # Process
    result = payment_gateway.charge(amount)
    
    # Cache result (24h TTL)
    redis.setex(f"idem:{idempotency_key}", 86400, json.dumps(result))
    return result
```

**Idempotent by default:**

| Method | Idempotent? | Notes |
|---|---|---|
| GET | ✅ Always | Read-only |
| PUT | ✅ Always | Replace = same result every time |
| DELETE | ✅ Effectively | Second delete → 404, same final state |
| POST | ❌ By default | Add Idempotency-Key header to make it safe |
| PATCH | ❌ By default | Unless using absolute values |

---

## 14. Real-World Architectures

### Stripe API — Gold Standard REST Design

```
Base URL: https://api.stripe.com/v1

Resource naming:
  /v1/charges         → Payment charges
  /v1/customers       → Customer records
  /v1/payment_intents → Payment intent objects

Auth: API key in Authorization header
  Authorization: Bearer sk_live_abc123...

Idempotency keys on every POST:
  Idempotency-Key: <unique UUID per operation>

Pagination: cursor-based
  GET /v1/charges?limit=10&starting_after=ch_abc

Error format: consistent machine-readable codes
  { "error": { "type": "card_error", "code": "card_declined", ... } }

Versioning: Date-based in header
  Stripe-Version: 2024-04-10
```

### GitHub — REST + GraphQL Dual API

```
REST API (v3):     https://api.github.com/repos/owner/repo
GraphQL API (v4):  https://api.github.com/graphql

Why two APIs?
  REST: backward compatibility, simple integrations, webhooks
  GraphQL: efficient for complex queries (fetch repo + issues + PRs + contributors 
           in one request vs 4 REST calls)

Rate limits:
  REST:      5,000 requests/hour (authenticated)
  GraphQL:   5,000 points/hour  (cost varies per query complexity)
```

### Netflix — BFF Pattern

```
API Gateway (Zuul / Spring Cloud Gateway)
  ├── /api/mobile/*    → Mobile BFF
  │                      - Compressed responses
  │                      - Image URLs with device-specific sizes
  │                      - Minimal metadata
  │
  ├── /api/web/*       → Web BFF
  │                      - Full metadata
  │                      - Rich content
  │
  └── /api/tv/*        → TV BFF
                          - Simplified navigation model
                          - Large image assets
```

### Uber — gRPC Internal APIs

```
All internal microservices communicate via gRPC:
  DispatchService.proto     → matches riders and drivers
  PricingService.proto      → calculates surge pricing
  LocationService.proto     → streams driver GPS positions

Why gRPC internally:
  ✅ Low latency (critical for real-time matching)
  ✅ Bidirectional streaming for driver location updates
  ✅ Strong types prevent subtle bugs across 500+ services
  ✅ ~40% smaller payloads than equivalent JSON

External REST API for drivers and riders:
  Mobile app → REST/HTTP/2 → API Gateway → gRPC → Internal Services
```

---

## 15. Common Mistakes

| Mistake | Why It's Bad | Fix |
|---|---|---|
| **Verbs in REST URLs** (`/getUser`) | Breaks REST contract, confuses clients | Use nouns: `/users/42` |
| **Using POST for everything** | Loses semantic meaning, breaks caching | Use correct HTTP methods |
| **No versioning** | Schema change breaks all clients simultaneously | URL versioning from day one |
| **No pagination** | Endpoint returns 10M rows → client/server OOM | Always paginate, max page size |
| **Returning 200 with error in body** | Clients can't distinguish success from failure without parsing body | Use correct HTTP status codes |
| **Exposing stack traces in errors** | Security vulnerability — reveals internals | Return only user-safe error messages |
| **No rate limiting** | Any client can DoS your API | Rate limit at gateway level |
| **Long-lived JWTs (no expiry)** | Compromised token = permanent access | Short TTL (15m) + refresh token |
| **No idempotency on POST** | Client retries → duplicate charges/records | Add Idempotency-Key support |
| **No request_id** | Debugging a production issue becomes impossible | Add X-Request-ID to every request/response |
| **Breaking change without version bump** | Silent client failures | New version for any breaking change |
| **Using GET for mutations** | Cached by browsers/CDNs → unexpected behavior | GET is read-only, always |

---

## 16. Interview Cheatsheet

### Quick Definitions

| Term | One-liner |
|---|---|
| **REST** | HTTP-based resource-oriented API style (stateless, uniform interface) |
| **GraphQL** | Client-driven query language — fetch exactly what you need |
| **gRPC** | RPC framework using Protobuf + HTTP/2 — for microservices |
| **API Gateway** | Single entry point handling routing, auth, rate limiting, logging |
| **JWT** | Stateless self-contained token — claims signed with private key |
| **OAuth 2.0** | Delegation protocol — 3rd party access without sharing passwords |
| **Rate Limiting** | Throttle requests per client to prevent abuse |
| **Idempotency Key** | Client-provided UUID to make retries safe |
| **Cursor Pagination** | Position-based pagination — no page drift, O(1) at scale |
| **BFF** | Backend for Frontend — gateway tailored to each client type |

### When to Use What

| Scenario | Recommendation |
|---|---|
| Public API, 3rd party consumers | REST + URL versioning + API key auth |
| Mobile app with complex data needs | GraphQL (reduce over/under-fetching) |
| Microservice-to-microservice internal | gRPC (low latency, typed, streaming) |
| Real-time chat / collaboration | WebSocket + Redis pub/sub backplane |
| Live feed, notifications | SSE (simpler than WebSocket for server→client) |
| Large dataset listing | Cursor-based pagination |
| Payment / idempotent mutations | Idempotency-Key header |
| Multi-service fanout gateway | API Gateway (Kong, AWS API Gateway) |
| Client-specific response shaping | BFF pattern |

### The Interview Answer Template

```
When asked "Design the API for X":

1. PROTOCOL choice:
   "Public-facing API → REST with JSON.
    Internal service calls → gRPC for low latency.
    Real-time events → WebSocket or SSE."

2. RESOURCE design:
   "Resources are nouns: /orders, /users, /products.
    Nested: /users/{id}/orders for ownership.
    Actions: POST /orders/{id}/cancel."

3. AUTH:
   "JWT tokens — 15 min expiry + refresh token.
    Validated at API gateway — services trust the gateway."

4. VERSIONING:
   "URL versioning from day one: /v1/.
    Breaking changes get a new version,
    old version deprecated with Sunset header."

5. RATE LIMITING:
   "Token bucket at API gateway level.
    429 with Retry-After header.
    Per-user limits based on plan tier."

6. PAGINATION:
   "Cursor-based for all list endpoints.
    Max limit capped at 100.
    Opaque cursor prevents scraping."
```

### Must-Know Interview Points

- ☑ **REST is stateless** — no server-side session. Auth via token on every request.
- ☑ **401 = not authenticated** (no valid token). **403 = not authorized** (valid token, wrong permissions).
- ☑ **GET is safe + idempotent. POST is neither** (add Idempotency-Key).
- ☑ **GraphQL caching is hard** — all requests are POSTs to one endpoint.
- ☑ **gRPC = HTTP/2 + Protobuf.** 10x lower latency than REST for internal services.
- ☑ **Cursor pagination** for production. Offset degrades at scale.
- ☑ **JWT = stateless.** Short TTL + refresh. Hard to revoke — that's the tradeoff.
- ☑ **API Gateway = single entry point.** Auth, rate limit, route here, not in every service.
- ☑ **Never expose stack traces** in error responses.
- ☑ **Version your API from day one** — retrofitting is painful.

---

*Sources: HelloInterview System Design API Design, DesignGurus REST vs GraphQL vs gRPC, Baeldung API comparison, SystemDesignSchool, AlgoMaster.io, Oneuptime API Gateway Architecture (2026), SystemDesign.one Best Practices, Strapi REST Design Guide, DEV Community API Protocols — combined with first-principles system design knowledge.*