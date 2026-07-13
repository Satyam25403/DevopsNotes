# Selenium - Practical Scripting Patterns

Hands-on patterns for building maintainable test scripts: the Page Object Model, handling forms/dropdowns, alerts, frames, and multi-window scenarios.

## Table of Contents
- [The Maintainability Problem](#the-maintainability-problem)
- [Page Object Model (POM)](#page-object-model-pom)
- [Handling Forms and Input Fields](#handling-forms-and-input-fields)
- [Handling Dropdowns (Select Elements)](#handling-dropdowns-select-elements)
- [Handling Alerts and Confirm Dialogs](#handling-alerts-and-confirm-dialogs)
- [Handling iFrames](#handling-iframes)
- [Handling Multiple Windows/Tabs](#handling-multiple-windowstabs)
- [Actions API for Complex Interactions](#actions-api-for-complex-interactions)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Maintainability Problem

**Visual:**
```
Without any structure:
test_login.py:    driver.find_element(By.ID, "username")...
test_checkout.py:  driver.find_element(By.ID, "username")...
test_profile.py:    driver.find_element(By.ID, "username")...

UI changes: the username field's ID changes from
"username" to "user-email" →
   MUST update the SAME locator in every single test
   file that references it → easy to miss one,
   tests break inconsistently
```

---

## Page Object Model (POM)

**A design pattern: each page of the application gets its own class, containing that page's locators and actions. Tests interact with the page object, never directly with raw locators.**

```python
# pages/login_page.py
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

class LoginPage:
    USERNAME_INPUT = (By.ID, "username")
    PASSWORD_INPUT = (By.ID, "password")
    SUBMIT_BUTTON = (By.CSS_SELECTOR, "[data-testid='login-submit']")
    ERROR_MESSAGE = (By.CLASS_NAME, "error-message")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    def load(self):
        self.driver.get("https://example.com/login")

    def login(self, username, password):
        self.driver.find_element(*self.USERNAME_INPUT).send_keys(username)
        self.driver.find_element(*self.PASSWORD_INPUT).send_keys(password)
        self.driver.find_element(*self.SUBMIT_BUTTON).click()

    def get_error_message(self):
        element = self.wait.until(EC.visibility_of_element_located(self.ERROR_MESSAGE))
        return element.text
```

**Test file using the Page Object:**
```python
# tests/test_login.py
from pages.login_page import LoginPage

def test_invalid_login_shows_error(driver):
    login_page = LoginPage(driver)
    login_page.load()
    login_page.login("wronguser", "wrongpass")
    assert login_page.get_error_message() == "Invalid credentials"

def test_valid_login_redirects_to_dashboard(driver):
    login_page = LoginPage(driver)
    login_page.load()
    login_page.login("validuser", "validpass")
    assert "dashboard" in driver.current_url
```

**Visual:**
```
Now, when the UI changes:
Username field ID changes → update ONCE in
   LoginPage.USERNAME_INPUT → EVERY test using
   LoginPage automatically uses the corrected locator

Structure:
┌─────────────────────────────────────┐
│  pages/                                  │
│    login_page.py     ← locators + actions   │
│    checkout_page.py   ← locators + actions     │
│    profile_page.py      ← locators + actions      │
│  tests/                                       │
│    test_login.py          ← uses LoginPage           │
│    test_checkout.py        ← uses CheckoutPage           │
└─────────────────────────────────────┘

Tests read like plain English describing USER BEHAVIOR
("login with these credentials, expect this error") —
NOT cluttered with raw CSS selectors and low-level clicks.
```

---

## Handling Forms and Input Fields

```python
username_field = driver.find_element(By.ID, "username")
username_field.clear()
username_field.send_keys("testuser@example.com")
```

**Visual:**
```
Why .clear() before .send_keys() matters:
Input fields sometimes have pre-filled placeholder
or default text (from browser autofill, or a
previous test run reusing the same session) —
without clearing first, new text gets APPENDED
to existing content instead of replacing it,
producing a garbled, unintended value.
```

---

## Handling Dropdowns (Select Elements)

```python
from selenium.webdriver.support.ui import Select

dropdown = Select(driver.find_element(By.ID, "country"))
dropdown.select_by_visible_text("United States")
dropdown.select_by_value("US")
dropdown.select_by_index(0)
```

**Visual:**
```
⚠️ Important distinction:
The Select class ONLY works on genuine HTML <select> elements.

Many modern UIs use CUSTOM dropdown components built
from <div>s and JavaScript (not real <select> tags) —
for those, Select() will fail; you must instead:
1. Click the div that opens the custom dropdown
2. Wait for the option list to appear
3. Click the specific option element directly

Always inspect the actual HTML first to determine
which approach applies.
```

---

## Handling Alerts and Confirm Dialogs

```python
from selenium.webdriver.support import expected_conditions as EC

driver.find_element(By.ID, "delete-btn").click()

wait.until(EC.alert_is_present())
alert = driver.switch_to.alert
print(alert.text)
alert.accept()   # clicks "OK"
# alert.dismiss()  # clicks "Cancel"
```

**Visual:**
```
Why switch_to.alert is necessary:
Native browser alert()/confirm()/prompt() dialogs are
OUTSIDE the normal DOM — find_element() cannot see or
interact with them at all. Selenium requires explicitly
switching context to the alert before interacting with it.

┌───────────────────────────────┐
│  Are you sure you want to delete?  │
│      [Cancel]      [OK]              │
└───────────────────────────────┘
   driver.switch_to.alert
   → gives you a handle to interact with THIS dialog
```

---

## Handling iFrames

```python
driver.switch_to.frame("payment-iframe")
driver.find_element(By.ID, "card-number").send_keys("4111111111111111")
driver.switch_to.default_content()  # back to the main page
```

**Visual:**
```
Why this matters:
Elements INSIDE an <iframe> are NOT part of the main
page's DOM as far as Selenium's default context is
concerned — attempting find_element() on an iframe's
internal content WITHOUT switching context first
results in NoSuchElementException, even though the
element is clearly visible on screen.

┌─────────────────────────────┐
│  Main Page                       │
│  ┌───────────────────────┐  │
│  │  <iframe>                    │  │  ← must switch_to.frame()
│  │   Payment form fields          │  │     to interact with this
│  │  </iframe>                     │  │
│  └───────────────────────┘  │
└─────────────────────────────┘

Common real-world case: third-party payment processors
(Stripe, PayPal) often embed their form fields inside
an iframe for PCI compliance/security isolation.
```

---

## Handling Multiple Windows/Tabs

```python
main_window = driver.current_window_handle

driver.find_element(By.LINK_TEXT, "Open in new tab").click()

for handle in driver.window_handles:
    if handle != main_window:
        driver.switch_to.window(handle)
        break

print(driver.title)  # now interacting with the NEW tab
driver.close()
driver.switch_to.window(main_window)  # back to the original tab
```

**Visual:**
```
Window Handle Flow:
Main Window (handle: A)
        │
        │ clicks a link that opens a new tab
        ↓
Main Window (A)  +  New Tab (handle: B)
        │
   driver still "looking at" A by default —
   must explicitly switch_to.window(B) to
   interact with the new tab's content
```

---

## Actions API for Complex Interactions

**For interactions beyond simple clicks — hover, drag-and-drop, right-click, keyboard combinations.**

```python
from selenium.webdriver.common.action_chains import ActionChains

actions = ActionChains(driver)

# Hover over a menu to reveal a submenu
menu = driver.find_element(By.ID, "main-menu")
actions.move_to_element(menu).perform()

# Drag and drop
source = driver.find_element(By.ID, "draggable")
target = driver.find_element(By.ID, "dropzone")
actions.drag_and_drop(source, target).perform()

# Right-click (context click)
element = driver.find_element(By.ID, "file-item")
actions.context_click(element).perform()
```

**Visual:**
```
Why a separate Actions API:
element.click() only supports a SINGLE, simple click.
Real-world UI interactions (hover-triggered menus,
drag-and-drop file uploads, right-click context menus)
require a CHAIN of low-level mouse/keyboard events,
which ActionChains lets you compose and execute together.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is helping a QA team refactor an unmaintainable test suite where every test file directly contains dozens of hardcoded locators, and a recent UI redesign broke nearly all 80 existing tests.

**What they do:**
1. Introduces the **Page Object Model**, extracting locators and page-specific actions out of the 80 scattered test files into a clean `pages/` directory — one class per distinct page/component.
2. Identifies that the checkout flow embeds a **third-party payment iframe**, and that several previously "flaky" tests were actually failing because nobody had added the required `switch_to.frame()` call — the tests were trying to interact with elements that, from Selenium's default context, simply didn't exist.
3. Replaces several tests that were manually implementing multi-step hover/click sequences with raw `ActionChains`-free workarounds (using JavaScript execution hacks) with proper **ActionChains-based hover interactions**, making the tests both more reliable and far more readable.
4. After the refactor, the next UI redesign only requires updating **locators in the relevant Page Object classes** — the 80 test files themselves remain untouched, since they only ever called high-level page methods like `login_page.login(...)`.

**Why this matters:** The Page Object Model isn't just a "nice to have" organizational preference — it's what determines whether a test suite survives a UI redesign with a 20-minute locator update, or requires days of tracking down and fixing scattered, duplicated locators across dozens of files.

---

Next: **04selenium_grid_and_parallel_execution.md** — scaling test execution with Selenium Grid, Docker-based browsers, and running tests in parallel.