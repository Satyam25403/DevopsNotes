# Selenium - Installation & Setup

Getting Selenium installed, managing browser drivers, and running your very first automated browser script.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Installing Selenium (Python)](#installing-selenium-python)
- [The Driver Management Problem](#the-driver-management-problem)
- [Selenium Manager (Automatic Driver Handling)](#selenium-manager-automatic-driver-handling)
- [Your First Script](#your-first-script)
- [Running in Headless Mode](#running-in-headless-mode)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## Prerequisites

**Visual:**
```
┌──────────────────────────────────────────┐
│  Requirement          Why                     │
├──────────────────────────────────────────┤
│  A supported language      Python, Java, JS,         │
│  (pick one)                  C#, Ruby, Kotlin           │
│  A real browser installed       Chrome, Firefox, Edge,     │
│                                Safari (must actually be       │
│                                present on the machine)           │
└──────────────────────────────────────────┘
```

---

## Installing Selenium (Python)

```bash
pip install selenium
```

**Visual:**
```
Other Language Equivalents:
┌──────────┬───────────────────────────────────┐
│ Java         │ Add selenium-java to Maven/Gradle    │
│ JavaScript     │ npm install selenium-webdriver           │
│ C#              │ dotnet add package Selenium.WebDriver       │
│ Ruby              │ gem install selenium-webdriver                  │
└──────────┴───────────────────────────────────┘
```

**Verify:**
```bash
python -c "import selenium; print(selenium.__version__)"
# 4.20.0
```

---

## The Driver Management Problem

**Historically, one of the most annoying parts of Selenium setup — you needed to manually download the exact browser driver matching your exact browser version.**

**Visual:**
```
The Old Pain Point:
Chrome updates itself automatically (v120 → v121) →
   chromedriver v120 no longer matches →
   scripts start failing with cryptic version
   mismatch errors →
   manually re-download the correct chromedriver
   version → repeat this dance forever
```

---

## Selenium Manager (Automatic Driver Handling)

**Since Selenium 4.6+, this pain point is solved automatically.**

```python
from selenium import webdriver

driver = webdriver.Chrome()
```

**Visual:**
```
What happens behind the scenes:
webdriver.Chrome() called
        ↓
Selenium Manager checks: what version of Chrome is
   actually installed on this machine?
        ↓
Automatically downloads the MATCHING chromedriver
   version if not already cached
        ↓
Launches the browser session — no manual driver
   download/PATH configuration needed at all
```

**Visual — when you'd still configure manually:**
```
Automatic (recommended default):
driver = webdriver.Chrome()

Manual override (e.g. pinned CI environment,
specific driver path required):
from selenium.webdriver.chrome.service import Service
service = Service(executable_path="/usr/local/bin/chromedriver")
driver = webdriver.Chrome(service=service)
```

---

## Your First Script

```python
from selenium import webdriver
from selenium.webdriver.common.by import By

driver = webdriver.Chrome()

driver.get("https://example.com")
print(driver.title)

search_box = driver.find_element(By.NAME, "q")
search_box.send_keys("Selenium WebDriver")
search_box.submit()

driver.quit()
```

**Visual:**
```
Script Execution Flow:
┌────────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────┐
│ Launch Chrome │→ │ Navigate to    │→ │ Find element,    │→ │ Close        │
│ (new window     │  │ example.com      │  │ interact with it    │  │ browser        │
│  opens)             │  │                    │  │                       │  │ (driver.quit())  │
└────────────┘  └─────────────┘  └──────────────┘  └──────────┘

Note: driver.quit() is CRITICAL — without it, browser
processes can linger and accumulate, especially in
CI environments running many tests sequentially.
```

**Visual — quit() vs close():**
```
driver.close()  → closes the CURRENT browser tab/window only
driver.quit()    → closes ALL windows AND ends the entire
                   WebDriver session, cleaning up the
                   background driver process too

Best practice: use quit() at the end of every test,
typically in a teardown/fixture, to guarantee cleanup
even if the test fails partway through.
```

---

## Running in Headless Mode

**Headless means running the browser WITHOUT a visible UI window — essential for CI/CD servers that have no display.**

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

options = Options()
options.add_argument("--headless=new")
options.add_argument("--window-size=1920,1080")

driver = webdriver.Chrome(options=options)
driver.get("https://example.com")
print(driver.title)
driver.quit()
```

**Visual:**
```
Headed Mode (default, visible window):
┌──────────────────────┐
│  🖥️  Actual browser window   │  ← useful for local debugging,
│      visibly opens and         │     watching what the script
│      you can watch it work        │     is actually doing
└──────────────────────┘

Headless Mode (no visible window):
┌──────────────────────┐
│  (nothing visually appears)  │  ← required for CI/CD servers
│  Browser runs in the             │     with no display, faster,
│  background, fully functional      │     lower resource usage
└──────────────────────┘

Same test logic works in EITHER mode — headless
is simply a launch configuration, not a different API.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up the first Selenium-based smoke test for a newly deployed staging environment, to run automatically after every deployment.

**What they do:**
1. Writes a Python script using `webdriver.Chrome()` with **Selenium Manager's automatic driver handling** — deliberately avoiding manually pinning a chromedriver version, since the CI runner's Chrome version may update independently over time, and automatic matching avoids a recurring maintenance chore.
2. Configures the script to run in **headless mode with an explicit window size** (`--window-size=1920,1080`) — without this, headless Chrome defaults to a small window size that can cause responsive-design elements to render differently than expected, leading to confusing test failures unrelated to any real bug.
3. Ensures `driver.quit()` runs in a **try/finally block** (or test framework teardown fixture), so even if an assertion fails mid-test, the browser process is always cleaned up — preventing an accumulation of zombie Chrome processes on the CI runner after hundreds of pipeline runs.
4. Runs the script locally first in **headed mode** to visually confirm it behaves correctly, then switches to headless only once verified — debugging is much faster when you can actually watch the browser interact with the page.

**Why this matters:** Getting driver management, headless configuration, and cleanup right at the very start avoids the most common early frustrations teams have with Selenium — mysterious version mismatches, viewport-related test flakiness, and CI runners slowly grinding to a halt from leaked browser processes.

---

Next: **02core_concepts_locators_elements.md** — the theory behind locators, WebElements, and handling asynchronous page loading with waits.