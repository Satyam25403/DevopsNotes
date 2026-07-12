# JMeter - Distributed & CLI Execution

Running real load tests the right way: non-GUI mode, scaling across multiple machines, and wiring JMeter into CI/CD pipelines.

## Table of Contents
- [Why Non-GUI Mode](#why-non-gui-mode)
- [Running in Non-GUI (CLI) Mode](#running-in-non-gui-cli-mode)
- [Generating HTML Dashboard Reports](#generating-html-dashboard-reports)
- [Distributed Testing (Master-Slave)](#distributed-testing-master-slave)
- [Docker-Based Distributed Testing](#docker-based-distributed-testing)
- [JMeter in CI/CD Pipelines](#jmeter-in-cicd-pipelines)
- [Setting Pass/Fail Thresholds](#setting-passfail-thresholds)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Why Non-GUI Mode

**Visual:**
```
GUI Mode Resource Usage:
┌─────────────────────────────────────────┐
│  JMeter Process                            │
│  ├── Rendering the tree UI                 │
│  ├── Rendering Listener graphs/tables live   │
│  ├── Storing full results for GUI display    │
│  └── ALSO trying to generate load             │
└─────────────────────────────────────────┘
Result: less capacity to actually generate load,
        and CPU spent on rendering skews timing results

CLI (Non-GUI) Mode Resource Usage:
┌─────────────────────────────────────────┐
│  JMeter Process                            │
│  ├── Generating load                        │
│  └── Writing results to a file                │
│  (no rendering overhead at all)              │
└─────────────────────────────────────────┘
Result: maximum capacity dedicated purely to
        generating accurate load
```

**Rule of thumb:** Build and debug in GUI mode with a handful of threads. Always execute real load tests in **non-GUI mode**.

---

## Running in Non-GUI (CLI) Mode

```bash
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/
```

**Flag breakdown:**
```
┌──────┬──────────────────────────────────────────┐
│ Flag  │ Meaning                                    │
├──────┼──────────────────────────────────────────┤
│  -n    │ Non-GUI mode                                │
│  -t    │ Test plan file (.jmx)                        │
│  -l    │ Log/results file (.jtl) — raw results          │
│  -e    │ Generate HTML report after the test run         │
│  -o    │ Output folder for the HTML report                │
│  -j    │ JMeter's own log file (engine logs, not results) │
└──────┴──────────────────────────────────────────┘
```

**Visual:**
```
Command Execution Flow:
jmeter -n -t test-plan.jmx -l results.jtl -e -o report/
         ↓
┌─────────────────────────────────────┐
│  Test runs (no window opens)           │
│  Console shows progress:                │
│  summary = 5000 in 00:05:23 =          │
│    15.4/s Avg: 234 Min: 89 Max: 3021   │
│    Err: 12 (0.24%)                      │
└─────────────────────────────────────┘
         ↓
results.jtl (raw data)  +  report/index.html (dashboard)
```

---

## Generating HTML Dashboard Reports

**Visual:**
```
report/index.html Dashboard Sections:
┌───────────────────────────────────────────────┐
│  APDEX (Application Performance Index)             │
│  Requests Summary: 5000 total, 12 errors (0.24%)   │
│                                                      │
│  Response Time Percentiles:                          │
│    50th percentile (median): 210ms                    │
│    90th percentile: 450ms                              │
│    95th percentile: 620ms                               │
│    99th percentile: 1200ms                               │
│                                                          │
│  Throughput over time (graph)                            │
│  Response Times over time (graph)                         │
│  Errors by type (table)                                     │
└───────────────────────────────────────────────────┘
```

**Why percentiles matter more than averages:**
```
Average response time: 234ms  ← sounds fine!

But:
50th percentile: 210ms   (half of users experienced this or better)
99th percentile: 1200ms  (1% of users waited over a full second)

If you have 100,000 requests/day, 1% = 1,000 requests
experiencing a genuinely bad, slow experience —
completely hidden by looking at the average alone.
```

If report generation from an existing results file is needed (without re-running the test):
```bash
jmeter -g results.jtl -o report/
```

---

## Distributed Testing (Master-Slave)

**When a single machine can't generate enough load, spread it across multiple machines.**

**Visual:**
```
┌─────────────────────────────────────────────────────┐
│                    Master Machine                       │
│         (controls test, aggregates results)              │
└───────────┬─────────────┬─────────────┬───────────────┘
            │             │             │
      ┌─────┴────┐  ┌─────┴────┐  ┌─────┴────┐
      │  Slave 1   │  │  Slave 2   │  │  Slave 3   │
      │ (generates │  │ (generates │  │ (generates │
      │  load)      │  │  load)      │  │  load)      │
      └─────┬────┘  └─────┬────┘  └─────┬────┘
            │             │             │
            └─────────────┴─────────────┘
                          ↓
                ┌──────────────────┐
                │  Target Application │
                └──────────────────┘

Each slave generates a portion of total virtual users.
Master coordinates start/stop and merges results.
```

### Setup

**On each slave machine:**
```bash
jmeter-server
# Starts RMI service, listens for master commands
```

**On the master machine:**
```bash
jmeter -n -t test-plan.jmx -R slave1-ip,slave2-ip,slave3-ip -l results.jtl
```

**Visual:**
```
-R flag distributes the SAME test plan to all slaves.

If Thread Group = 300 users total across 3 slaves:
Slave 1: 100 users
Slave 2: 100 users
Slave 3: 100 users
(JMeter divides threads evenly automatically)
```

⚠️ **Firewall/network requirement:** Master and slaves need RMI ports open between them (default 1099 + dynamic ports) — a common real-world friction point in segmented corporate networks.

---

## Docker-Based Distributed Testing

Containerizing slaves makes distributed testing far easier to spin up/tear down.

```bash
# Start slave containers
docker run -d --name jmeter-slave1 justb4/jmeter jmeter-server
docker run -d --name jmeter-slave2 justb4/jmeter jmeter-server

# Run master pointing at slave container IPs
docker run --rm -v $(pwd):/tests justb4/jmeter \
  -n -t /tests/test-plan.jmx \
  -R $(docker inspect -f '{{.NetworkSettings.IPAddress}}' jmeter-slave1),$(docker inspect -f '{{.NetworkSettings.IPAddress}}' jmeter-slave2) \
  -l /tests/results.jtl
```

**Visual:**
```
Why Docker for distributed testing:
- Spin up N slave containers on demand (Kubernetes Jobs work too)
- Tear down after the test — no leftover VMs costing money
- Consistent JMeter version across all slaves (no version drift)
```

**Kubernetes-based scaling (common in mature setups):**
```
Master Job ──creates──→ N Slave Pods (Deployment/Job)
                              ↓
                    Load generated from within
                    the same cluster/network as
                    the target service (realistic
                    internal-traffic simulation)
```

---

## JMeter in CI/CD Pipelines

### GitHub Actions Example

```yaml
name: Performance Test
on:
  workflow_dispatch:  # manual trigger, or on schedule

jobs:
  load-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run JMeter Test
        run: |
          docker run --rm -v $(pwd):/tests justb4/jmeter \
            -n -t /tests/test-plan.jmx \
            -l /tests/results.jtl \
            -e -o /tests/report

      - name: Upload Report
        uses: actions/upload-artifact@v4
        with:
          name: jmeter-report
          path: report/
```

**Visual:**
```
Pipeline Flow:
Checkout code → Run JMeter (Docker) → Generate HTML report → Upload as artifact
                                                                    ↓
                                                    Team downloads/views report
                                                    from the GitHub Actions run
```

### Jenkins with Performance Plugin

```groovy
stage('Performance Test') {
    steps {
        sh 'jmeter -n -t test-plan.jmx -l results.jtl'
    }
}
stage('Publish Results') {
    steps {
        perfReport sourceDataFiles: 'results.jtl'
    }
}
```

**Visual:**
```
Jenkins Performance Plugin:
results.jtl → Trend graphs across builds → visible in Jenkins job history
              (compare THIS release's performance against LAST release)
```

---

## Setting Pass/Fail Thresholds

A load test that just "produces a report" isn't automation — it needs to **fail the pipeline** when performance regresses.

```bash
jmeter -n -t test-plan.jmx -l results.jtl
```

Then parse `results.jtl` and enforce thresholds in a script:

```bash
#!/bin/bash
ERROR_RATE=$(awk -F',' 'NR>1{total++; if($8=="false") fail++} END{print (fail/total)*100}' results.jtl)

if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
  echo "FAIL: Error rate $ERROR_RATE% exceeds 1% threshold"
  exit 1
fi
echo "PASS: Error rate $ERROR_RATE% within threshold"
```

**Visual:**
```
Pipeline Gate Logic:
Run Test → Parse results.jtl → Check thresholds
                                      ↓
                    Error rate > 1%?  ──Yes──→ 🛑 FAIL pipeline
                    P95 latency > 2s? ──Yes──→ 🛑 FAIL pipeline
                              │
                             No
                              ↓
                        ✓ PASS → continue to deploy
```

Plugins like the **JMeter Maven Plugin** or **Jenkins Performance Plugin** can automate this threshold-checking instead of hand-rolled scripts.

---

## Real-Life DevOps Use Case

**Scenario:** A platform team wants performance regression testing automated as part of every release candidate — not a manual, occasional activity run before big launches only.

**What they build:**
1. A **dedicated `performance-tests` repo** containing versioned `.jmx` test plans for each critical service, reviewed via PR just like application code.
2. A **scheduled CI/CD pipeline** (nightly) that spins up **Docker-based distributed slaves** in a Kubernetes namespace, runs the load test against the staging environment, and tears the slaves down afterward — avoiding idle infrastructure cost.
3. **Threshold-based pass/fail logic**: P95 latency must stay under 500ms and error rate under 0.5%, compared against the **previous release's baseline** (not just an absolute number) — catching gradual performance regressions across releases (a "boiling frog" problem where each release is 5% slower but nobody notices until it's a full-blown outage).
4. Automatic **Jenkins Performance Plugin trend graphs** so engineering leadership can see performance trending over the last 20 releases at a glance.
5. When a threshold fails, the pipeline **automatically opens a ticket** tagging the team that owns the regressed endpoint, using the JMeter report's per-sampler breakdown to pinpoint exactly which API slowed down.

**Why this matters:** Performance testing that only runs "before a big launch" always finds problems too late to fix comfortably. Making it a routine, automated, threshold-gated part of the pipeline turns performance from a fire-drill into a continuously monitored quality attribute — the same philosophy as automated functional tests or SonarQube's Quality Gates.

---

Next: **05advanced_realworld_usecases.md** — plugins, real-time monitoring with InfluxDB/Grafana, advanced protocol testing, and broader real-world performance engineering practices.