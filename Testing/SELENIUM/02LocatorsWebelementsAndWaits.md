# Selenium - Locators, WebElements & Waits

The core theory every Selenium script depends on: finding elements reliably, and handling the fact that web pages load asynchronously.

## Table of Contents
- [WebElement Basics](#webelement-basics)
- [Locator Strategies](#locator-strategies)
- [Choosing the Right Locator](#choosing-the-right-locator)
- [The Asynchronous Page Problem](#the-asynchronous-page-problem)
- [Implicit Waits](#implicit-waits)
- [Explicit Waits](#explicit-waits)
- [Expected Conditions](#expected-conditions)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## WebElement Basics

**A WebElement represents one specific element on the page — a button, input field, link, or any DOM node.**

```python
element = driver.find_element(By.ID, "login-button")
element.click()

text_content = element.text
is_shown = element.is_displayed()
is_clickable = element.is_enabled()
```

**Visual:**
```
Common WebElement Actions:
┌───────────────────────────────────────────┐
│  .click()             → click the element        │
│  .send_keys("text")     → type into an input field     │
│  .clear()                 → clear an input field's value    │
│  .text                     → get the visible text content       │
│  .get_attribute("href")      → get an HTML attribute value          │
│  .is_displayed()               → check if visible on screen             │
│  .is_enabled()                   → check if interactable                    │
└───────────────────────────────────────────┘
```

---

## Locator Strategies

**A Locator is HOW you tell Selenium which element you want.**

```python
from selenium.webdriver.common.by import By

driver.find_element(By.ID, "username")
driver.find_element(By.NAME, "email")
driver.find_element(By.CLASS_NAME, "btn-primary")
driver.find_element(By.TAG_NAME, "button")
driver.find_element(By.LINK_TEXT, "Sign Up")
driver.find_element(By.CSS_SELECTOR, "div.card > button")
driver.find_element(By.XPATH, "//button[contains(text(),'Submit')]")
```

**Visual:**
```
┌─────────────┬────────────────────────────────────┐
│ Strategy         │ Example HTML                            │
├─────────────┼────────────────────────────────────┤
│ ID                │ <input id="username">                     │
│ NAME               │ <input name="email">                         │
│ CLASS_NAME           │ <button class="btn-primary">                   │
│ TAG_NAME               │ <button>...</button>                              │
│ LINK_TEXT                │ <a href="...">Sign Up</a>                            │
│ CSS_SELECTOR                │ any valid CSS selector syntax                         │
│ XPATH                         │ any valid XPath expression                                │
└─────────────┴────────────────────────────────────┘
```

---

## Choosing the Right Locator

**Visual:**
```
Priority Order (most to least preferred):

1. ID                  → fastest, most stable, unique by HTML spec
                        driver.find_element(By.ID, "submit-btn")

2. CSS_SELECTOR             → fast, readable, widely understood
                        driver.find_element(By.CSS_SELECTOR, "[data-testid='submit']")

3. XPATH (only when needed)     → powerful but slower, can be fragile,
                        USE when you need to navigate relationships
                        (e.g. "find the button inside THIS specific div")
                        driver.find_element(By.XPATH, "//div[@class='modal']//button")

4. LINK_TEXT/CLASS_NAME              → convenient but easily broken by
                        minor UI text/styling changes
```

**Visual — the data-testid pattern (best practice):**
```
Problem: CSS classes and text content change often
(designers restyle buttons, copy gets updated) —
tests break for reasons that have NOTHING to do
with actual functional bugs.

Solution: developers add a dedicated, STABLE attribute
purely for testing purposes:
<button data-testid="submit-order">Place Order</button>

driver.find_element(By.CSS_SELECTOR, "[data-testid='submit-order']")

This locator survives text changes, class renames,
and styling updates — it only breaks if the element
is functionally removed, which is exactly when you
WANT the test to fail.
```

---

## The Asynchronous Page Problem

**This is the single most common source of flaky Selenium tests.**

**Visual:**
```
What Actually Happens When a Page Loads:
┌────────────────────────────────────────┐
│  1. Initial HTML loads                       │
│  2. JavaScript begins executing                 │
│  3. An API call fires to fetch data                │
│  4. ...time passes (network latency)...              │
│  5. Data arrives, JavaScript renders the                 │
│     actual content into the DOM                            │
└────────────────────────────────────────┘

If your script tries to find an element at step 1-4,
before step 5 completes:
driver.find_element(By.ID, "product-list")
→ NoSuchElementException — the element genuinely
  doesn't exist in the DOM YET, even though it will
  exist a few hundred milliseconds later.

This is NOT a bug in Selenium — it's an accurate
reflection of how modern JavaScript-heavy pages
actually load.
```

---

## Implicit Waits

**A global setting: tells Selenium to keep RETRYING element lookups for up to N seconds before giving up.**

```python
driver.implicitly_wait(10)  # applies to ALL find_element calls
```

**Visual:**
```
Without implicit wait:
find_element() called → element not in DOM yet →
   FAILS IMMEDIATELY (0 second tolerance)

With implicitly_wait(10):
find_element() called → element not in DOM yet →
   Selenium RETRIES every short interval →
   element appears at 2.3 seconds →
   returns successfully (didn't wait the full 10s,
   just until it was found, up to that maximum)
```

⚠️ **Known limitation:** implicit waits apply uniformly to every single element lookup in the whole script — they don't let you wait for something SPECIFIC (like "wait until this spinner disappears" or "wait until this text changes"). That's what explicit waits are for.

---

## Explicit Waits

**Wait for a SPECIFIC condition, on a SPECIFIC element, before proceeding — the recommended, more precise approach.**

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, 10)
element = wait.until(EC.presence_of_element_located((By.ID, "product-list")))
```

**Visual:**
```
Explicit Wait Flow:
wait.until(condition) called
        ↓
Selenium checks the condition every ~500ms
        ↓
Condition met? → return immediately, proceed
Condition NOT met after 10s? → raise TimeoutException

Unlike implicit waits, this can wait for MEANINGFUL
conditions beyond "does the element exist" — like
"is it clickable" or "has this text appeared."
```

---

## Expected Conditions

**Visual:**
```
┌───────────────────────────────────────────────────┐
│ Condition                     Use Case                   │
├───────────────────────────────────────────────────┤
│ presence_of_element_located       Element exists in DOM        │
│                                (may not be visible yet)            │
│                                                        │
│ visibility_of_element_located       Element exists AND is             │
│                                actually visible on screen               │
│                                                        │
│ element_to_be_clickable                Element is visible AND              │
│                                enabled (not disabled/covered)                │
│                                                        │
│ text_to_be_present_in_element              Specific text has appeared             │
│                                inside an element                                     │
│                                                        │
│ invisibility_of_element_located                Wait for something (e.g. a               │
│                                loading spinner) to DISAPPEAR                              │
│                                                        │
│ alert_is_present                                  Wait for a JS alert/confirm                  │
│                                dialog to appear                                                 │
└───────────────────────────────────────────────────┘
```

**Combining conditions in a realistic flow:**
```python
wait = WebDriverWait(driver, 10)

# Wait for a loading spinner to disappear
wait.until(EC.invisibility_of_element_located((By.CLASS_NAME, "spinner")))

# THEN wait for the actual button to be clickable
button = wait.until(EC.element_to_be_clickable((By.ID, "checkout-btn")))
button.click()
```

**Visual:**
```
Why order matters here:
Spinner disappearing → THEN checking clickability
mirrors the REAL user experience — a human wouldn't
try clicking a button hidden behind a loading spinner
either. Matching your wait strategy to actual UI
behavior produces much more reliable tests.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer inherits a Selenium test suite that fails intermittently in CI (passes locally, fails ~20% of the time in the pipeline) — a classic "flaky test" problem.

**Root cause investigation and fix:**
1. Notices the suite relies entirely on `driver.implicitly_wait(5)` with no explicit waits anywhere — meaning every single element lookup gets the same generic 5-second tolerance, regardless of what's actually happening on that specific page.
2. Identifies that the CI environment's shared runners are simply SLOWER than developers' local machines (more resource contention, other jobs running concurrently) — pages that load a checkout button within 2 seconds locally sometimes take 6+ seconds in CI, exceeding the fixed 5-second implicit wait.
3. Replaces the blanket implicit wait with **targeted explicit waits** using `element_to_be_clickable` specifically on the handful of elements known to be slow (those waiting on API responses), while leaving fast, immediately-available elements without any wait overhead at all.
4. Adds an explicit `invisibility_of_element_located` wait for a loading spinner that appears during checkout — the ACTUAL root cause of most flaky failures, since the button technically existed in the DOM the whole time but was visually covered and non-functional while the spinner displayed.
5. Also **removes a mix of implicit AND explicit waits used together** (a known Selenium anti-pattern that can cause unpredictable, compounding wait times) — standardizing on explicit waits exclusively.

**Why this matters:** Flaky tests that fail "randomly" are almost always a timing/wait strategy problem, not a genuine application bug — and mixing implicit and explicit waits, or relying on a single blanket wait duration for a whole test suite, are the two most common root causes.

---

Next: **03practical_scripting_patterns.md** — the Page Object Model, handling forms/dropdowns/alerts, and building maintainable real-world test scripts.