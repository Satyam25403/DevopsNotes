# JMeter - Installation & Setup

Getting JMeter installed, launching the GUI, and building your very first test plan.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installation Options](#installation-options)
- [Launching the GUI](#launching-the-gui)
- [JMeter GUI Layout](#jmeter-gui-layout)
- [Building Your First Test Plan](#building-your-first-test-plan)
- [Running the Test](#running-the-test)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Prerequisites

**Visual:**
```
┌──────────────────────────────────────────┐
│  Requirement       Why                      │
├──────────────────────────────────────────┤
│  Java 8+           JMeter runs on the JVM    │
│  4GB+ RAM           Simulating many threads   │
│                     needs memory                │
│  (Optional) Docker  For containerized/CI runs   │
└──────────────────────────────────────────┘
```

```bash
java -version
# openjdk version "17.0.2" ...
```

---

## Installation Options

### Option 1: Manual Binary Install

```bash
wget https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-5.6.3.tgz
tar -xvzf apache-jmeter-5.6.3.tgz
export PATH=$PATH:$(pwd)/apache-jmeter-5.6.3/bin
```

### Option 2: Package Manager (Mac)

```bash
brew install jmeter
```

### Option 3: Docker (great for CI/CD)

```bash
docker run -v $(pwd):/tests justb4/jmeter -n -t /tests/test-plan.jmx -l /tests/results.jtl
```

**Visual:**
```
Installation Comparison:
┌──────────────┬─────────────────────────────────────┐
│ Method         │ Best For                              │
├──────────────┼─────────────────────────────────────┤
│ Manual binary  │ Local GUI test-plan building            │
│ Homebrew        │ Quick local setup on Mac                │
│ Docker           │ CI/CD pipelines, consistent versions    │
└──────────────┴─────────────────────────────────────┘
```

---

## Launching the GUI

```bash
jmeter
```

**Visual:**
```
Terminal Output:
┌────────────────────────────────────┐
│ WARN: GUI mode is not recommended    │
│ for load test, only for creating     │
│ test scripts.                         │
└────────────────────────────────────┘
         ↓
   GUI Window Opens
```

⚠️ **Important theory point:** The GUI is for *building and debugging* test plans with small numbers of threads. Actually *running* a real load test should always happen in **non-GUI/CLI mode** (covered in file 04) — running heavy load through the GUI consumes extra resources on the very machine trying to measure performance, skewing results.

---

## JMeter GUI Layout

**Visual:**
```
┌────────────────────────────────────────────────────────┐
│  File  Edit  Search  Run  Options  Tools  Help            │
├───────────────┬──────────────────────────────────────────┤
│  Test Plan     │                                            │
│  ├─ Thread Grp │        Configuration Panel                 │
│  │  ├─ Sampler │        (changes based on what's            │
│  │  ├─ Assert. │         selected on the left tree)          │
│  │  └─ Listener│                                            │
│  └─ Config Elem│                                            │
│  (Tree View)   │                                            │
└───────────────┴──────────────────────────────────────────┘
      ↑                          ↑
Left: element tree        Right: properties of
(structure of your          selected element
test plan)
```

---

## Building Your First Test Plan

### Step 1: Add a Thread Group

```
Right-click Test Plan → Add → Threads (Users) → Thread Group
```

**Configure:**
```
Number of Threads (users): 10
Ramp-up Period (seconds): 10
Loop Count: 1
```

**Visual:**
```
Ramp-up Explained:
Number of Threads: 10
Ramp-up Period: 10 seconds

Timeline:
0s   1s   2s   3s   4s   5s   6s   7s   8s   9s   10s
│    │    │    │    │    │    │    │    │    │    │
T1   T2   T3   T4   T5   T6   T7   T8   T9   T10
↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓    ↓
(one new virtual user starts every 1 second)

Why ramp up gradually instead of all at once?
Mimics realistic traffic growth, and avoids the
test tool itself creating an unrealistic instant-spike
artifact in the results.
```

### Step 2: Add an HTTP Request Sampler

```
Right-click Thread Group → Add → Sampler → HTTP Request
```

**Configure:**
```
Protocol: https
Server Name: api.example.com
Path: /products
Method: GET
```

### Step 3: Add a Listener to See Results

```
Right-click Thread Group → Add → Listener → View Results Tree
```

**Visual:**
```
Test Plan Structure So Far:
Test Plan
 └── Thread Group (10 users, 10s ramp-up)
      ├── HTTP Request Sampler (GET /products)
      └── View Results Tree (Listener)
```

---

## Running the Test

Click the green **▶ Start** button (or `Ctrl+R`).

**Visual:**
```
View Results Tree Output:
┌─────────────────────────────────────┐
│  GET /products         ✓ 200 OK       │
│  GET /products         ✓ 200 OK       │
│  GET /products         ✓ 200 OK       │
│  GET /products         ❌ 500 Error    │
│  ...                                    │
├─────────────────────────────────────┤
│  Response Body   Response Headers      │
│  Sampler Result   (raw response data)   │
└─────────────────────────────────────┘

Each row = one request from one virtual user
Green = passed, Red = failed (error or assertion failure)
```

### Saving the Test Plan

```
File → Save As → my-first-test.jmx
```

**Visual:**
```
.jmx file = XML under the hood
┌──────────────────────────────┐
│ <jmeterTestPlan>               │
│   <ThreadGroup>                 │
│     <HTTPSamplerProxy>          │
│     ...                          │
│ </jmeterTestPlan>               │
└──────────────────────────────┘

This file is what gets committed to Git and
run in CI/CD pipelines (see file 04).
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer needs to quickly validate that a newly deployed staging environment can handle basic traffic before handing it to QA for full testing.

**What they do:**
1. Uses the **JMeter GUI** locally to quickly build a simple test plan hitting the 5 most critical API endpoints (login, search, checkout, etc.) — GUI is fine here since it's just a handful of test threads for validation, not a real load test.
2. Saves the `.jmx` file and commits it to the team's `performance-tests` Git repository, so it's version-controlled like any other code artifact.
3. Runs a **quick manual smoke test** with 5 threads to confirm the environment responds correctly (functional sanity check) before running heavier load.
4. Only after confirming basic functionality does the engineer move to **non-GUI CLI mode** (file 04) to run the actual heavy load test — avoiding wasting time debugging "is the environment even reachable" issues under full load.

**Why this order matters:** Jumping straight into a 10,000-user load test against a broken/misconfigured environment wastes time and produces confusing results. GUI-based small-scale validation first, heavy CLI-based load testing second, is the standard practical workflow.

---

Next: **02core_elements_test_plan.md** — deep dive into Thread Groups, Samplers, Config Elements, Assertions, and Listeners.