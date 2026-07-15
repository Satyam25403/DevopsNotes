# Topic 3: SOLID Principles

---

## 1. Introduction

**Definition:**
SOLID is an acronym for five design principles that, together, guide object-oriented software toward being maintainable, flexible, and easy to extend. Coined by Robert C. Martin ("Uncle Bob"), each letter represents one principle:

| Letter | Principle | One-line definition |
|---|---|---|
| **S** | Single Responsibility Principle | A class should have only ONE reason to change |
| **O** | Open/Closed Principle | Open for extension, closed for modification |
| **L** | Liskov Substitution Principle | Subtypes must be substitutable for their base types |
| **I** | Interface Segregation Principle | Clients shouldn't depend on methods they don't use |
| **D** | Dependency Inversion Principle | Depend on abstractions, not concrete implementations |

> **Note:** Liskov Substitution Principle gets its own dedicated deep-dive in Topic 4, since it's subtle and commonly misunderstood. This topic covers it at an introductory level and covers the other four in full depth.

**Why SOLID exists / what problem it solves:**
OOP (Topic 2) gives you the TOOLS (classes, inheritance, polymorphism) — but tools can still be used badly. SOLID is a set of GUIDELINES for using those tools well. Without SOLID discipline, even "properly" object-oriented code can become:
- Fragile (a small change in one class breaks several others)
- Rigid (adding a new feature requires touching many existing files)
- Hard to test (classes are tangled together with hard dependencies)

**When it should be used:**
- Any codebase expected to evolve over time (which is nearly all real production code)
- Especially valuable in systems with multiple developers, or long-lived codebases

**When it should NOT be over-applied:**
- Applying every SOLID principle maximally to a tiny, one-off script adds unnecessary abstraction layers — SOLID is a guide for managing complexity, not a checklist to blindly maximize
- Over-applying DIP (creating an interface for every single class "just in case") is a very real, common anti-pattern called "interface bloat"

**Advantages:**
- Produces flexible, maintainable, testable code
- Reduces the "ripple effect" of changes
- Makes code easier for new team members to understand and safely extend

**Disadvantages:**
- Can lead to excessive abstraction/indirection if applied dogmatically
- Requires experience to know WHEN to apply which principle — a common interview discussion point itself

---

## 2. Real-World Analogy

Think of a **restaurant kitchen**:

- **SRP**: The chef cooks, the waiter serves, the cashier handles payment — each role has ONE job. If the chef also had to handle payments, a change in payment processing (switching payment providers) would risk disrupting cooking operations too.
- **OCP**: The restaurant's menu can be EXTENDED with new dishes without needing to retrain the entire kitchen staff on how to take orders or serve food — the process is closed for modification, open for new dishes to be added.
- **LSP** (brief): If the menu says "any dessert can be substituted for the free dessert offer," then EVERY dessert on the menu must genuinely work as a valid substitute — if "fruit salad" secretly takes 45 minutes to prepare while everything else takes 5, it breaks the promise the menu made.
- **ISP**: A dishwasher doesn't need to know how to take customer orders — job descriptions (interfaces) should be specific to what a role ACTUALLY needs to do, not one giant "do everything" job description.
- **DIP**: The restaurant's ordering PROCESS depends on "a person who can take orders" (a waiter role/abstraction), not specifically on "Bob" (a concrete person) — if Bob quits, any trained waiter can substitute in without changing the restaurant's ordering process itself.

---

## 3. Theory

### S — Single Responsibility Principle (SRP)

**Core idea:** A class should have exactly one reason to change — meaning it should have one, well-defined responsibility.

```
Violates SRP:                       Follows SRP:
┌─────────────────────┐        ┌──────────┐  ┌──────────┐  ┌──────────┐
│  Invoice                    │        │  Invoice     │  │  InvoicePrinter│  │  InvoiceRepo   │
│  - calculateTotal()             │   →    │  - calculate  │  │  - print()       │  │  - save()        │
│  - printInvoice()                 │        │    Total()      │  └──────────┘  └──────────┘
│  - saveToDatabase()                  │        └──────────┘
└─────────────────────┘
   3 reasons to change: pricing logic,
   printing format, AND database schema
   all in ONE class
```

### O — Open/Closed Principle (OCP)

**Core idea:** Software entities should be OPEN for extension (you can add new behavior) but CLOSED for modification (you don't need to edit existing, already-tested code to do so).

```
Violates OCP:                              Follows OCP:
if (type.equals("CIRCLE")) {...}          interface Shape { calculateArea(); }
else if (type.equals("SQUARE")) {...}     class Circle implements Shape {...}
// Adding a new shape means EDITING          class Square implements Shape {...}
// this existing method                    // Adding Triangle = NEW class,
                                           // zero changes to existing code
```

### L — Liskov Substitution Principle (LSP) — Introductory Overview

**Core idea:** If `B` is a subtype of `A`, objects of type `A` should be replaceable with objects of type `B` WITHOUT altering the correctness of the program.

```
A quick preview (full depth in Topic 4):
class Rectangle { setWidth(w); setHeight(h); getArea(); }
class Square extends Rectangle { 
    // overrides setWidth to ALSO change height (to stay a square)
    // This BREAKS the expectation that setWidth only affects width —
    // code written for Rectangle may now behave unexpectedly
    // when given a Square instead.
}
```
*(We'll dissect exactly why this is a violation, and how to properly fix it, in Topic 4.)*

### I — Interface Segregation Principle (ISP)

**Core idea:** Clients should not be forced to depend on methods they don't use — prefer several small, specific interfaces over one large, general-purpose interface.

```
Violates ISP:                              Follows ISP:
interface Worker {                        interface Workable { work(); }
    work();                               interface Eatable { eat(); }
    eat();
}                                          class HumanWorker implements Workable, Eatable {...}
class RobotWorker implements Worker {     class RobotWorker implements Workable {...}
    work() {...}                           // Robot no longer forced to implement
    eat() { /* doesn't apply! */ }         // a meaningless eat() method
}
```

### D — Dependency Inversion Principle (DIP)

**Core idea:** High-level modules should not depend on low-level modules — both should depend on abstractions. Abstractions should not depend on details; details should depend on abstractions.

```
Violates DIP:                              Follows DIP:
class OrderService {                      class OrderService {
    private MySqlDatabase db;                 private Database db; // interface!
    // tightly coupled to ONE specific         public OrderService(Database db) {
    // database implementation                    this.db = db; // injected
}                                                }
                                           }
                                           // OrderService works with ANY
                                           // Database implementation
```

---

## 4. UML / Class Diagram

Showing DIP visually, since it's the principle most tied to a structural diagram:

```
┌─────────────────────┐
│      <<interface>>          │
│         Database                 │
├─────────────────────┤
│ + save(data): void                │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────┐
   │               │
┌──┴────────┐  ┌──┴────────┐
│  MySqlDatabase     │  │  MongoDatabase       │
└──────────┘  └──────────┘

┌─────────────────────┐
│      OrderService              │
├─────────────────────┤
│ - database: Database  ◇──────────→ (depends on the INTERFACE,
│ + OrderService(Database db)     │      never on MySqlDatabase or
└─────────────────────┘      MongoDatabase directly)
```

**Relationship explanation:** `OrderService` has an **association** (dependency) with the `Database` **interface** — NOT with any concrete class. `MySqlDatabase` and `MongoDatabase` both **implement** `Database`. This is the structural shape of Dependency Inversion: the high-level module (`OrderService`) and low-level modules (`MySqlDatabase`, `MongoDatabase`) both depend on the SAME abstraction, rather than the high-level module depending directly on a low-level one.

---

## 5. Java Implementation

```java
// ============================================
// S — Single Responsibility Principle
// ============================================
public class Invoice {
    private final double amount;

    public Invoice(double amount) {
        this.amount = amount;
    }

    public double calculateTax() {
        return amount * 0.18; // ONLY responsible for tax calculation logic
    }

    public double getAmount() {
        return amount;
    }
}

// Printing is a SEPARATE responsibility, in its own class
public class InvoicePrinter {
    public void print(Invoice invoice) {
        System.out.printf("Invoice amount: %.2f, Tax: %.2f%n",
                invoice.getAmount(), invoice.calculateTax());
    }
}

// Persistence is ALSO a separate responsibility
public class InvoiceRepository {
    public void save(Invoice invoice) {
        System.out.println("Saving invoice to database...");
        // actual DB logic would go here
    }
}


// ============================================
// O — Open/Closed Principle
// ============================================
public interface DiscountStrategy {
    double applyDiscount(double amount);
}

public class NoDiscount implements DiscountStrategy {
    public double applyDiscount(double amount) { return amount; }
}

public class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    public PercentageDiscount(double percent) { this.percent = percent; }
    public double applyDiscount(double amount) {
        return amount - (amount * percent / 100);
    }
}

// Adding a NEW discount type (e.g. FlatDiscount) requires
// ZERO changes to any existing class — this class is
// CLOSED for modification, OPEN for extension via new
// DiscountStrategy implementations
public class PriceCalculator {
    private final DiscountStrategy discountStrategy;

    public PriceCalculator(DiscountStrategy discountStrategy) {
        this.discountStrategy = discountStrategy;
    }

    public double calculateFinalPrice(double originalPrice) {
        return discountStrategy.applyDiscount(originalPrice);
    }
}


// ============================================
// I — Interface Segregation Principle
// ============================================
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

// A human worker needs BOTH behaviors
public class HumanWorker implements Workable, Eatable {
    public void work() { System.out.println("Human working..."); }
    public void eat() { System.out.println("Human eating lunch..."); }
}

// A robot worker ONLY needs Workable — it's never forced
// to implement a meaningless eat() method
public class RobotWorker implements Workable {
    public void work() { System.out.println("Robot working..."); }
}


// ============================================
// D — Dependency Inversion Principle
// ============================================
public interface Database {
    void save(String data);
}

public class MySqlDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

public class MongoDatabase implements Database {
    public void save(String data) {
        System.out.println("Saving to MongoDB: " + data);
    }
}

// OrderService depends on the Database ABSTRACTION,
// injected via the constructor — NEVER on a concrete class
public class OrderService {
    private final Database database;

    // Constructor injection — the standard way to achieve DIP in Java
    public OrderService(Database database) {
        this.database = database;
    }

    public void placeOrder(String orderDetails) {
        // business logic here...
        database.save(orderDetails);
    }
}


// ============================================
// Demo tying it all together
// ============================================
public class SolidDemo {
    public static void main(String[] args) {
        // SRP in action
        Invoice invoice = new Invoice(1000.0);
        new InvoicePrinter().print(invoice);
        new InvoiceRepository().save(invoice);

        // OCP in action
        PriceCalculator calculator = new PriceCalculator(new PercentageDiscount(10));
        System.out.println("Final price: " + calculator.calculateFinalPrice(500.0));

        // ISP in action
        Workable robot = new RobotWorker();
        robot.work(); // no eat() forced upon it

        // DIP in action — swapping databases requires ZERO
        // changes to OrderService itself
        OrderService serviceWithMySql = new OrderService(new MySqlDatabase());
        serviceWithMySql.placeOrder("Order #123");

        OrderService serviceWithMongo = new OrderService(new MongoDatabase());
        serviceWithMongo.placeOrder("Order #124");
    }
}
```

**Key line-by-line notes:**
- `Invoice`, `InvoicePrinter`, `InvoiceRepository` — each has exactly one job; a change to how invoices are PRINTED never risks breaking TAX CALCULATION logic (SRP).
- `PriceCalculator` takes a `DiscountStrategy` in its constructor — new discount types never require editing `PriceCalculator` (OCP).
- `RobotWorker implements Workable` (not the combined `Worker`) — it's never forced to have an `eat()` method it can't meaningfully implement (ISP).
- `OrderService`'s constructor accepts a `Database` interface, not `MySqlDatabase` directly — this is **constructor-based dependency injection**, the standard Java mechanism for achieving DIP.

---

## 6. Dry Run

**Sample input:** Running the DIP portion — creating an `OrderService` with `MySqlDatabase`, then with `MongoDatabase`.

```
1. new MySqlDatabase()
   → Heap allocates a MySqlDatabase object

2. new OrderService(mySqlDatabaseInstance)
   → OrderService's constructor runs
   → this.database = mySqlDatabaseInstance
   → NOTE: the field type is "Database" (the interface),
     but it's actually POINTING to a MySqlDatabase object

3. serviceWithMySql.placeOrder("Order #123")
   → Inside placeOrder(), database.save("Order #123") is called
   → JVM checks: what is the ACTUAL type behind "database"?
     → MySqlDatabase
   → Invokes MySqlDatabase's save() → prints "Saving to MySQL: Order #123"

4. new MongoDatabase() + new OrderService(mongoDatabaseInstance)
   → A SEPARATE OrderService instance, this time database
     actually points to a MongoDatabase object

5. serviceWithMongo.placeOrder("Order #124")
   → database.save("Order #124") called
   → Actual type is MongoDatabase this time
   → Invokes MongoDatabase's save() → prints "Saving to MongoDB: Order #124"
```

**What's happening in memory:** `OrderService`'s CODE never changed between steps 2 and 4 — the exact same `placeOrder()` method body works correctly with either database, because it only ever calls methods declared on the `Database` interface. This is dynamic dispatch (from Topic 2) being leveraged specifically to achieve the flexibility DIP promises.

---

## 7. Real-World Software Example

- **SRP**: Spring's typical layered architecture — `Controller` (handles HTTP), `Service` (business logic), `Repository` (data access) — each layer has one clear responsibility.
- **OCP**: Payment gateway integrations — adding "Apple Pay" as a new payment method shouldn't require modifying the existing `CreditCardPayment` or `PayPalPayment` classes.
- **ISP**: Java's own `Collection` interface hierarchy — `List`, `Set`, `Queue` are separate interfaces rather than one giant "do everything" collection interface, so a `Set` implementation isn't forced to implement list-specific, index-based methods.
- **DIP**: Spring Framework's entire dependency injection system is built around DIP — you `@Autowired` an interface, and Spring injects whichever concrete bean is configured, without your code ever referencing the concrete class.

---

## 8. Internal Working

SOLID principles are **design-time/structural** guidelines — they don't have a distinct "runtime mechanism" the way polymorphism does. However, their EFFECTS at runtime are worth understanding:

- **DIP's runtime mechanism IS dynamic dispatch** (from Topic 2) — the flexibility DIP provides is only POSSIBLE because Java resolves interface method calls at runtime based on actual object type, via the vtable mechanism.
- **Dependency Injection frameworks (like Spring)** automate DIP at a framework level: at application startup, Spring's IoC (Inversion of Control) container reads configuration (annotations or XML), constructs the appropriate concrete objects, and INJECTS them into constructors — meaning the actual "which implementation gets used" decision happens once, at startup, in a centralized place, rather than being hardcoded throughout your business logic.

---

## 9. Before vs After

**Before (violating multiple SOLID principles at once):**

```java
public class OrderManager {
    private MySqlDatabase database = new MySqlDatabase(); // DIP violation: concrete dependency

    public void processOrder(String type, double amount) {
        double finalAmount = amount;
        if (type.equals("PREMIUM")) {       // OCP violation: editing this for new types
            finalAmount = amount * 0.9;
        }

        database.save("Order: " + finalAmount);  // SRP violation: order logic + persistence mixed
        System.out.println("Printing receipt..."); // SRP violation: printing logic also mixed in
    }
}
```

**Problems:**
- Hardcoded `MySqlDatabase` — switching databases means editing this class (violates DIP).
- String-based discount type checking — adding a new discount tier means modifying this method (violates OCP).
- Order processing, persistence, AND printing are all bundled into one method (violates SRP).

**After (SOLID-compliant, using the classes built in Section 5):**
- `OrderService` depends on the `Database` interface, injected via constructor (DIP).
- `PriceCalculator` uses `DiscountStrategy` implementations, extensible without modification (OCP).
- `Invoice`, `InvoicePrinter`, `InvoiceRepository` each handle exactly one concern (SRP).

---

## 10. SOLID Principles Connection

*(This section is somewhat redundant for this particular topic, since the whole topic IS SOLID — but here's how the principles reinforce each other:)*

- **SRP and ISP are closely related**: SRP is about class responsibilities; ISP is the same idea applied specifically to interface design.
- **OCP is often ACHIEVED using DIP**: you extend behavior (OCP) by introducing new implementations of an abstraction (DIP) rather than editing existing code.
- **LSP is a PRECONDITION for OCP to work safely**: if subtypes aren't properly substitutable (LSP violation), then "extending" via new subtypes (OCP) can silently break calling code.

---

## 11. Interview Questions

**Beginner:**
1. What does each letter in SOLID stand for?
2. Give a simple example of a class that violates the Single Responsibility Principle.
3. What's the difference between "open for extension" and "closed for modification"?

**Intermediate:**
4. How does Dependency Injection relate to the Dependency Inversion Principle?
   *Answer: DI is a TECHNIQUE (a way of supplying dependencies from outside, e.g., via constructor); DIP is the PRINCIPLE (depend on abstractions). DI is commonly used to IMPLEMENT DIP, but they're conceptually distinct — DIP is achievable without a DI framework, just by manually passing interface-typed dependencies.*
5. Why might a large, general-purpose interface be problematic? How does ISP address this?
6. Give an example of when following OCP might be over-engineering.
   *Answer: If a piece of logic (e.g., a specific tax calculation) has genuinely never changed in years and has no realistic anticipated variants, wrapping it in an interface + strategy pattern preemptively adds indirection without real benefit.*

**Advanced:**
7. How would you refactor a class that violates BOTH SRP and DIP simultaneously? Walk through your approach.
8. Explain how Spring's `@Autowired` mechanism embodies the Dependency Inversion Principle.
9. Can a class violate OCP while still technically compiling and working correctly? Explain with an example.
   *Answer: Yes — OCP violations are about MAINTAINABILITY, not correctness. A class using if/else type-checking for behavior works fine functionally, but every new case requires editing existing code, which is the actual violation.*
10. How do SRP and the "God class" anti-pattern (from Topic 1) relate?
11. Discuss a scenario where strictly following ISP might lead to too many small interfaces. How do you strike a balance?
12. Why is LSP often considered the hardest SOLID principle to apply correctly? (Full answer in Topic 4.)

---

## 12. Common Mistakes

- **Confusing SRP with "a class should only have one method"** — SRP is about ONE REASON TO CHANGE, not literally one method; a class can have many small, cohesive methods and still satisfy SRP.
- **Over-applying DIP** — creating an interface for every single class "just in case," even for classes with no realistic alternative implementation ever expected — this is needless indirection, not good design.
- **Violating LSP while believing you're using inheritance correctly** — the classic Square/Rectangle problem (Topic 4) is a prime example of code that COMPILES and even superficially "works," while still violating LSP in subtle ways.
- **Treating SOLID as absolute rules rather than guidelines** — SOLID principles can sometimes be in tension with each other or with simplicity; experienced engineers make judgment calls, not blind rule-following.

---

## 13. Time Complexity

Not applicable — SOLID principles are structural/design guidelines, not algorithms.

---

## 14. Java API Examples

- **SRP**: `java.text.SimpleDateFormat` handles ONLY date formatting/parsing — it doesn't also handle date arithmetic (that's `Calendar`/`java.time` classes' job).
- **OCP**: The `java.util.Comparator` interface — sorting algorithms in `Collections.sort()` never need modification to support a new sorting criterion; you just supply a new `Comparator` implementation.
- **ISP**: `java.util.List` vs `java.util.Set` vs `java.util.Queue` — separate, focused interfaces rather than one giant `Collection` with every possible method.
- **DIP**: The entire JDBC API — application code depends on `java.sql.Connection`/`Statement` interfaces, never on vendor-specific driver classes directly.

---

## 15. Practice Problem

Take this SRP/OCP-violating class and refactor it:

```java
public class ReportGenerator {
    public String generateReport(String format, java.util.List<String> data) {
        StringBuilder sb = new StringBuilder();
        if (format.equals("CSV")) {
            for (String item : data) sb.append(item).append(",");
        } else if (format.equals("JSON")) {
            sb.append("[");
            for (String item : data) sb.append("\"").append(item).append("\",");
            sb.append("]");
        }
        System.out.println("Report generated, sending email...");
        return sb.toString();
    }
}
```

- Identify which SOLID principles are violated.
- Refactor using an interface for report formatting, and separate the "sending email" responsibility into its own class.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a logging system that can log messages to Console, File, or a remote Logging Service, where the specific destination should be configurable, and adding a new destination (e.g., a database logger) shouldn't require changing any existing code."

Think about:
- Which SOLID principle(s) does this scenario primarily test?
- How would you structure the interface(s) involved?
- How would the "main" application code obtain and use the logger, without hardcoding a specific destination?

---

## 17. Advanced LLD Scenario

**Design a Ride-Sharing Fare Calculation System** (like Uber/Lyft) where:
- Different ride types (Economy, Premium, Pool) have different fare calculation rules
- New ride types may be introduced in the future (e.g., "Uber Black")
- The system must apply surge pricing multipliers independently of the base fare logic
- Different payment methods (Card, Wallet, Cash) must be supported, with new ones added over time

Consider:
- **SRP**: separate fare calculation from payment processing from surge pricing logic
- **OCP**: new ride types and payment methods should be addable via new classes, not by editing existing fare/payment logic
- **DIP**: the core `RideService` should depend on `FareStrategy` and `PaymentMethod` abstractions, not concrete implementations
- **ISP**: consider whether a single "Ride" interface trying to cover fare calculation, payment, AND driver matching would need to be split into more focused interfaces

*(This scenario naturally leads toward the Strategy pattern for fare calculation and payment methods — which we'll formalize in Topic 5.)*

---

## 18. Summary

**Definition:** SOLID is a set of five design principles (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) guiding maintainable, flexible object-oriented design.

**Intent:** Reduce fragility and rigidity in code, making systems easier to extend and maintain as they grow.

**Key classes:** N/A (principles apply across all class design — concrete examples per principle shown in Section 5).

**Advantages:** Flexible, maintainable, testable code; reduced ripple effects from changes.

**Disadvantages:** Risk of over-engineering if applied dogmatically; requires judgment to apply appropriately.

**Real-world use case:** Spring Framework's entire architecture is essentially SOLID principles made concrete — layered responsibilities (SRP), extensible strategies (OCP), dependency-injected interfaces (DIP).

**Java example:** `OrderService` depending on a `Database` interface (DIP), `PriceCalculator` using swappable `DiscountStrategy` implementations (OCP).

**Interview takeaway:** Be ready to identify SOLID violations in a given code snippet, explain the SPECIFIC problem each violation causes (not just name the principle), and refactor accordingly — this is one of the most commonly tested skills in LLD interviews.

**One-line memory trick:** *"SOLID: one job per class (S), extend don't edit (O), substitutes must behave (L), slim interfaces (I), depend on contracts not concretes (D)."*

---

*End of Topic 3. Type "Next" to proceed to Topic 4: Liskov Substitution Principle (deep dive).*