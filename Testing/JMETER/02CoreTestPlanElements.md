# JMeter - Core Test Plan Elements

The core theory behind every JMeter test plan: Thread Groups, Samplers, Config Elements, Assertions, Timers, and Listeners.

## Table of Contents
- [Thread Groups Deep Dive](#thread-groups-deep-dive)
- [Samplers](#samplers)
- [Config Elements](#config-elements)
- [Assertions](#assertions)
- [Timers](#timers)
- [Listeners](#listeners)
- [Execution Order Rules](#execution-order-rules)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Thread Groups Deep Dive

**A Thread Group represents a pool of virtual users executing the same scenario.**

**Visual:**
```
Thread Group Settings:
┌─────────────────────────────────────────────┐
│  Number of Threads (users): 100                │
│  Ramp-up Period (seconds): 60                  │
│  Loop Count: 5   (or "Infinite")                │
└─────────────────────────────────────────────┘

Behavior:
100 users, ramping up over 60 seconds,
each looping through the test plan 5 times

Timeline:
0s ────────────────────────────────────→ 60s
│  users starting gradually               │
│  1.67 users/second on average             │
└──────────────────────────────────────────┘
      Then each of the 100 users repeats
      their full scenario 5 times
```

### Special Thread Group Types

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Type                    Purpose                        │
├─────────────────────────────────────────────────────┤
│ Thread Group             Standard: fixed users, ramp-up  │
│ Stepping Thread Group    Gradual step increases           │
│                          (needs plugin)                    │
│ Ultimate Thread Group    Complex custom load patterns       │
│                          (needs plugin)                    │
│ Concurrency Thread Group  Maintains target concurrency      │
│                          instead of just starting threads   │
└─────────────────────────────────────────────────────┘
```

**Visual — Stepping pattern for realistic load ramps:**
```
Users
100 │                          ┌────────
 75 │                 ┌────────┘
 50 │        ┌────────┘
 25 │ ┌──────┘
  0 └─┴──────┴────────┴────────┴────────→ time
    Step-based ramp-up (used to find exactly
    which load level causes degradation)
```

---

## Samplers

**A Sampler is the actual request sent to the system under test.**

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Sampler Type          Protocol Tested                 │
├───────────────────────────────────────────────────┤
│ HTTP Request           Web APIs, websites               │
│ JDBC Request           Databases directly                 │
│ JMS Publisher/Subscriber Message queues                    │
│ FTP Request            File transfer                       │
│ TCP Sampler            Raw TCP-level protocols              │
│ SOAP/XML-RPC Request   Legacy SOAP web services              │
└───────────────────────────────────────────────────┘
```

**HTTP Request Sampler Fields:**
```
┌──────────────────────────────────────┐
│  Server Name: api.example.com          │
│  Port: 443                              │
│  Protocol: https                        │
│  Method: POST                            │
│  Path: /api/orders                       │
│  Body Data: {"item":"sku123","qty":2}   │
│  Headers: Content-Type: application/json│
└──────────────────────────────────────┘
```

---

## Config Elements

**Config Elements provide shared settings/data used by samplers within their scope.**

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│ Config Element              Purpose                     │
├─────────────────────────────────────────────────────┤
│ HTTP Request Defaults        Shared server/port/protocol   │
│                              so you don't repeat per-sampler│
│ HTTP Header Manager          Common headers (auth token,     │
│                              content-type) across requests    │
│ CSV Data Set Config           Feeds test data (usernames,       │
│                              product IDs) from a file           │
│ User Defined Variables        Static variables (base URL,        │
│                              env-specific values)                 │
└─────────────────────────────────────────────────────┘
```

### CSV Data Set Config Example

**users.csv:**
```
username,password
user1,pass1
user2,pass2
user3,pass3
```

**Config:**
```
Filename: users.csv
Variable Names: username,password
Recycle on EOF: True
Sharing mode: All threads
```

**Visual:**
```
Without CSV Data:
All 1000 virtual users log in as "testuser" ❌
→ Unrealistic, may hit "session already active" logic,
   doesn't test per-user data isolation

With CSV Data:
User 1 → username=user1, password=pass1
User 2 → username=user2, password=pass2
...
→ Realistic distinct-user simulation ✓
```

**Using the variable in a sampler:**
```
Path: /login
Body: {"username": "${username}", "password": "${password}"}
```

---

## Assertions

**Assertions validate that the response is actually correct — not just that a response arrived.**

**Visual:**
```
Without Assertions:
Server returns HTTP 200 with body: {"error": "Invalid product ID"}
→ JMeter counts this as SUCCESS (got a response!) ❌
  (This is a classic false-positive trap.)

With Response Assertion checking body contains "product_name":
Same response → Assertion FAILS → correctly flagged as an error ✓
```

**Common Assertion Types:**
```
┌───────────────────────────────────────────────────┐
│ Response Assertion    → Check status code/body text   │
│ Duration Assertion     → Fail if response takes too long│
│ Size Assertion          → Fail if response size is off    │
│ JSON Assertion           → Validate JSON structure/values  │
└───────────────────────────────────────────────────┘
```

**Response Assertion Example:**
```
Field to Test: Response Code
Pattern: 200
Test Type: Equals

Field to Test: Response Data
Pattern: "product_name"
Test Type: Contains
```

---

## Timers

**Timers introduce pauses between requests, simulating realistic human "think time."**

**Visual:**
```
Without a Timer:
User1: Request1 → Request2 → Request3  (fired instantly, back-to-back)
→ Unrealistic — real users pause to read/think between actions

With Constant Timer (2000ms):
User1: Request1 → [wait 2s] → Request2 → [wait 2s] → Request3
→ Mimics a real user browsing pace
```

**Common Timer Types:**
```
Constant Timer          → Fixed pause (e.g. always 2 seconds)
Uniform Random Timer     → Random pause within a range
Gaussian Random Timer    → Random pause following a bell curve
                           (models human behavior most realistically)
```

---

## Listeners

**Listeners collect and display results — but they come with a performance cost.**

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Listener                    Use Case                   │
├───────────────────────────────────────────────────┤
│ View Results Tree            Debugging (see full          │
│                              request/response) — GUI only│
│ Summary Report                Aggregate stats (avg,          │
│                              min, max, throughput)             │
│ Aggregate Report               Similar, per-sampler table       │
│ Response Time Graph             Visual trend over time            │
└───────────────────────────────────────────────────┘
```

⚠️ **Critical performance warning:**
```
"View Results Tree" stores EVERY request/response in memory.

100 users × full response bodies stored = JMeter itself
runs out of memory before the server under test does.

Rule: Use View Results Tree ONLY while building/debugging
      with a few threads. DISABLE or remove it before
      running a real load test with hundreds/thousands
      of users.
```

---

## Execution Order Rules

**Visual:**
```
JMeter processes elements in this priority order,
REGARDLESS of their visual position in the tree:

1. Configuration Elements   (e.g. CSV Data Set Config)
2. Pre-Processors
3. Timers
4. Sampler                   ← the actual request fires here
5. Post-Processors
6. Assertions
7. Listeners

Visual tree position only matters for SCOPE
(what the element applies to), not execution order.

Example:
Thread Group
 ├── CSV Data Set Config        ← runs first (config)
 ├── Constant Timer               ← runs before sampler (timer)
 ├── HTTP Request                  ← the actual request (sampler)
 │    └── Response Assertion        ← runs after (assertion, child-scoped)
 └── Summary Report                  ← collects results last (listener)
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is building a load test for a login + checkout flow and needs it to reflect real user behavior, not just blast requests.

**What they configure:**
1. A **CSV Data Set Config** feeding 5,000 unique test accounts, so each virtual user logs in as a distinct person — this also catches bugs where the backend incorrectly caches data per "session" rather than per "user."
2. A **Gaussian Random Timer** between actions (browse → add to cart → checkout) to simulate realistic think-time instead of a robotic instant-fire pattern that would produce misleadingly low latency numbers.
3. A **Response Assertion** on the checkout endpoint checking the JSON body contains `"order_status": "confirmed"` — catching a real bug once where the server returned HTTP 200 but a generic error JSON body due to a downstream payment service failure.
4. Removes **View Results Tree** before the actual load run (it was only there for initial debugging with 3 threads) and instead uses a **Summary Report** + writes results to a `.jtl` file for later analysis — keeping JMeter's own memory footprint low during a 5,000-user test.

**Why this matters:** A load test that doesn't use realistic data, think-time, and correctness assertions produces numbers that look good but don't reflect reality — this is one of the most common mistakes when teams first adopt JMeter.

---

Next: **03building_test_plans_practical.md** — hands-on: building a full multi-step user journey test plan with correlation, parameterization, and session handling.