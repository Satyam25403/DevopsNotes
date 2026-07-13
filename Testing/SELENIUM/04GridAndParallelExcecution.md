# Selenium - Grid & Parallel Execution

Scaling beyond one browser on one machine: Selenium Grid architecture, Docker-based execution, and running tests in parallel.

## Table of Contents
- [Why Parallel Execution Matters](#why-parallel-execution-matters)
- [Selenium Grid Architecture](#selenium-grid-architecture)
- [Running Selenium Grid with Docker](#running-selenium-grid-with-docker)
- [Connecting a Test Script to the Grid](#connecting-a-test-script-to-the-grid)
- [Parallel Execution with pytest-xdist](#parallel-execution-with-pytest-xdist)
- [Cross-Browser Testing Strategy](#cross-browser-testing-strategy)
- [Cloud-Based Grid Alternatives](#cloud-based-grid-alternatives)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Why Parallel Execution Matters

**Visual:**
```
Sequential Execution (one test after another):
Test 1 (30s) → Test 2 (30s) → Test 3 (30s) → ... → Test 100 (30s)
Total time: 100 × 30s = 3,000 seconds (50 minutes)

Parallel Execution (many tests simultaneously):
┌────────┐┌────────┐┌────────┐┌────────┐
│ Test 1     ││ Test 2     ││ Test 3     ││ Test 4     │  ... (10 at once)
│ (30s)        ││ (30s)        ││ (30s)        ││ (30s)        │
└────────┘└────────┘└────────┘└────────┘
Total time: 100 ÷ 10 parallel × 30s = 300 seconds (5 minutes)

A 10x reduction in total execution time, simply by
running multiple browser instances simultaneously
instead of one after another.
```

---

## Selenium Grid Architecture

**Visual:**
```
                    ┌───────────────────────┐
                    │      Grid Hub/Router          │
                    │  (receives test requests,        │
                    │   routes to available nodes)         │
                    └────────────┬───────────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ↓                  ↓                   ↓
        ┌──────────┐      ┌──────────┐        ┌──────────┐
        │  Node 1        │      │  Node 2        │        │  Node 3        │
        │  Chrome           │      │  Firefox           │        │  Chrome           │
        │  (2 sessions        │      │  (2 sessions        │        │  (2 sessions        │
        │   available)           │      │   available)           │        │   available)           │
        └──────────┘      └──────────┘        └──────────┘

Test scripts connect to the HUB, not to individual
nodes directly — the hub figures out which available
node/browser combination can fulfill each request.
```

**Flow in words:**
1. Your test script requests a session, specifying desired capabilities (e.g., "Chrome, version 120").
2. The **Hub/Router** checks which registered **Nodes** can satisfy that request.
3. The Hub forwards the session to an available matching node.
4. Your test's WebDriver commands flow through the hub to that specific node's browser instance.
5. Multiple tests requesting different (or even the same) browser type run **simultaneously** across different nodes.

---

## Running Selenium Grid with Docker

**The modern, standard way to run Grid — no manual node setup needed.**

```yaml
# docker-compose.yml
version: "3"
services:
  selenium-hub:
    image: selenium/hub:4.20
    ports:
      - "4442:4442"
      - "4443:4443"
      - "4444:4444"

  chrome-node:
    image: selenium/node-chrome:4.20
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
    deploy:
      replicas: 3   # 3 parallel Chrome nodes

  firefox-node:
    image: selenium/node-firefox:4.20
    depends_on:
      - selenium-hub
    environment:
      - SE_EVENT_BUS_HOST=selenium-hub
      - SE_EVENT_BUS_PUBLISH_PORT=4442
      - SE_EVENT_BUS_SUBSCRIBE_PORT=4443
    deploy:
      replicas: 2   # 2 parallel Firefox nodes
```

```bash
docker-compose up -d --scale chrome-node=3 --scale firefox-node=2
```

**Visual:**
```
Why Docker is the standard approach now:
- No manual browser/driver installation on each node
- Nodes are disposable — crash/restart cleanly
- Easy to scale up/down replica counts based on load
- Consistent, reproducible environment across
  every test run (same image = same browser version)
```

**Grid console:** `http://localhost:4444/ui` shows live node status, active sessions, and queue depth.

---

## Connecting a Test Script to the Grid

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

options = webdriver.ChromeOptions()

driver = webdriver.Remote(
    command_executor="http://localhost:4444/wd/hub",
    options=options
)

driver.get("https://example.com")
print(driver.title)
driver.quit()
```

**Visual:**
```
Key Difference from Local Execution:
Local:   webdriver.Chrome()
                → talks directly to a local chromedriver

Grid:    webdriver.Remote(command_executor="http://hub:4444/wd/hub")
                → talks to the Grid HUB, which routes to
                  whichever available node can fulfill it

The REST of your test script logic (find_element,
click, assertions) is IDENTICAL either way — only
how you construct the driver object changes.
```

---

## Parallel Execution with pytest-xdist

**Running multiple tests concurrently from a single test runner, distributing work across the Grid's available nodes.**

```bash
pip install pytest-xdist
pytest -n 4 tests/
```

**Visual:**
```
pytest -n 4
        ↓
Spawns 4 WORKER processes, each running a subset
of the total test suite
        ↓
Each worker independently creates its own
webdriver.Remote() session against the Grid
        ↓
Grid Hub distributes these 4 simultaneous session
requests across available nodes (Chrome/Firefox pool)
        ↓
4 tests genuinely execute AT THE SAME TIME,
each in its own isolated browser instance
```

⚠️ **Important:** each parallel test must be fully independent — no shared state (like the same user account being modified by two tests simultaneously) or tests will interfere with each other unpredictably.

---

## Cross-Browser Testing Strategy

**Visual:**
```
Not Every Test Needs Every Browser:
┌───────────────────────────────────────────────┐
│  Test Category           Browser Coverage             │
├───────────────────────────────────────────────┤
│  Critical user journeys      ALL supported browsers        │
│  (login, checkout, payment)    (Chrome, Firefox, Edge, Safari)│
│                                                      │
│  Standard functional tests       PRIMARY browser only            │
│  (most feature tests)              (usually Chrome, matching            │
│                                  majority of real user traffic)             │
│                                                      │
│  Visual/layout-specific tests       Browsers where visual bugs                │
│                                  are historically more common                  │
│                                  (e.g. Safari for CSS quirks)                     │
└───────────────────────────────────────────────┘
```

**Visual — why NOT run everything on every browser:**
```
100 tests × 4 browsers = 400 total test executions
   → 4x the CI time and infrastructure cost,
     for often marginal additional bug-catching value
     on tests that don't involve browser-specific
     rendering or JavaScript API differences

Smarter approach: run the FULL suite on your primary
browser (matching real user traffic data), and run
only the CRITICAL subset across all browsers —
balancing thoroughness against practical CI cost/time.
```

---

## Cloud-Based Grid Alternatives

**For teams that don't want to operate their own Grid infrastructure.**

**Visual:**
```
┌───────────────────────────────────────────────┐
│ Service            Notes                              │
├───────────────────────────────────────────────┤
│ BrowserStack           Huge device/browser matrix,          │
│                     including real mobile devices             │
│ Sauce Labs              Similar broad coverage, strong           │
│                     CI/CD integrations                              │
│ LambdaTest                Competitive pricing, growing                 │
│                     feature set                                          │
└───────────────────────────────────────────────┘
```

```python
driver = webdriver.Remote(
    command_executor="https://username:accesskey@hub.browserstack.com/wd/hub",
    options=options
)
```

**Visual:**
```
Self-hosted Grid vs Cloud Service tradeoff:
┌──────────────────┬─────────────────────────┐
│ Self-hosted Grid       │ Cloud Service                  │
├──────────────────┼─────────────────────────┤
│ Full control              │ Zero infrastructure to manage    │
│ Lower cost at scale          │ Access to real mobile devices        │
│ Requires ops effort              │ Pay-per-use, can be pricier         │
│                                │ at high volume                          │
└──────────────────┴─────────────────────────┘
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer needs to cut a 45-minute sequential Selenium test suite (200 tests) down to fit within a reasonable CI feedback loop, without losing coverage.

**What they build:**
1. Deploys **Selenium Grid via Docker Compose** in the CI environment itself (spun up fresh for each pipeline run, torn down after), with **5 Chrome node replicas** — sized based on the CI runner's available CPU/memory, avoiding over-provisioning nodes that would just contend for the same limited resources.
2. Uses **pytest-xdist with `-n 5`** to distribute the 200 tests across 5 parallel workers, each claiming its own Grid session — cutting execution time roughly 5x, from 45 minutes down to about 9 minutes.
3. Applies the **cross-browser strategy**: only the 20 tests covering critical paths (login, checkout, payment) run against Chrome, Firefox, AND Edge; the remaining 180 standard functional tests run against Chrome only — avoiding an unnecessary 3-4x multiplication of total test time for marginal additional coverage.
4. Ensures **test independence** by having each test create and clean up its own test data (e.g., a uniquely-generated test user per test, not a shared fixed account) — a prerequisite the team had to fix, since several existing tests were written assuming sequential execution and shared state, causing intermittent failures once run in parallel.
5. Documents Grid capacity planning: if the test suite grows further, the team knows to increase node replica count first, and add pytest-xdist workers to match, keeping node-to-worker ratio balanced rather than either resource sitting idle.

**Why this matters:** Parallelization only delivers real speedup if tests are genuinely independent — teams that skip fixing shared-state test data issues often find "parallel" execution actually produces MORE flaky failures than sequential execution did, for the wrong reasons entirely.

---

Next: **05cicd_and_realworld_usecases.md** — full CI/CD pipeline integration, and mature real-world Selenium testing practices.