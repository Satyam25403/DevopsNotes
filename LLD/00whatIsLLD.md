# Topic 1: What is Low-Level Design (LLD) and Why It Is Important?

---

## 1. Introduction

**Definition:**
Low-Level Design (LLD) is the process of designing the internal structure of a software system at the **class level** — defining classes, their attributes, methods, relationships, and interactions to implement the functionality specified by a High-Level Design (HLD).

Where **HLD** answers "what components exist and how do they talk to each other?" (e.g., "we need a Payment Service, an Order Service, and a Notification Service, communicating via REST"), **LLD** answers "how exactly is each of these components built internally, class by class?" (e.g., "the Payment Service has a `PaymentProcessor` interface, implemented by `CreditCardProcessor` and `UpiProcessor`, using the Strategy pattern...").

**Visual:**
```
┌─────────────────────────────────────────────────┐
│                 System Design Spectrum                    │
├─────────────────────────────────────────────────┤
│                                                           │
│  HLD (High-Level Design)                                     │
│  "What are the big pieces, and how do they connect?"            │
│  → Services, databases, load balancers, message queues            │
│  → Diagrams: architecture diagrams, sequence diagrams                 │
│                                                           │
│  LLD (Low-Level Design)                                          │
│  "How is EACH piece built internally, class by class?"              │
│  → Classes, interfaces, design patterns, method signatures             │
│  → Diagrams: UML class diagrams, object interaction diagrams               │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

**Why LLD exists / what problem it solves:**
Without deliberate LLD, code tends to evolve into a tangled mess where:
- Classes have too many responsibilities (a "God class" doing everything)
- Adding a new feature requires modifying existing, already-tested code (high regression risk)
- Code is tightly coupled — changing one class breaks three others
- The same logic is duplicated in multiple places because there's no clear ownership

LLD, done well using OOP principles and design patterns, produces code that is:
- **Modular** — each class has one clear responsibility
- **Extensible** — new features can be added without rewriting existing code
- **Testable** — classes can be tested independently, since dependencies are decoupled
- **Maintainable** — a new engineer can understand and safely modify the code

**When it should be used:**
- Any non-trivial system where multiple classes interact (which is almost every real system)
- Before writing code for a new feature/module, especially in an interview or a real production system expected to evolve
- When you anticipate future requirements changing (new payment methods, new notification channels, etc.)

**When it should NOT be over-applied:**
- For genuinely simple, one-off scripts, applying heavy design patterns is over-engineering — this is a real anti-pattern in itself (adding unnecessary abstraction "just in case")
- Don't design for hypothetical future requirements that have no realistic chance of occurring — YAGNI ("You Aren't Gonna Need It") is as important as good design

**Advantages:**
- Produces maintainable, extensible, testable code
- Makes team collaboration easier (clear class responsibilities/contracts)
- Reduces bugs from tight coupling and duplicated logic
- Directly maps to what's asked in SDE interviews at top companies

**Disadvantages:**
- Over-engineering risk — applying patterns where simple code would do
- Takes more upfront thinking time than "just write something that works"
- Can add indirection that makes code harder to trace for very simple cases

---

## 2. Real-World Analogy

Think of building a **house**.

- **HLD** is the architect's blueprint: "there's a kitchen here, a bedroom there, plumbing runs this way, electrical runs that way." It tells you the big rooms and how they connect, but not how the kitchen cabinets are built.
- **LLD** is the detailed carpentry plan for EACH room: "the kitchen cabinet is built from these specific panels, joined with these specific hinges, following this specific assembly order." It's the actual, buildable detail.

A construction company with a great HLD blueprint but no LLD would have workers guessing how to actually build each room — leading to inconsistent, poorly-built results. Similarly, a system with a great HLD architecture diagram but no LLD ends up with each engineer building their piece however they personally feel like it — leading to inconsistent, poorly-structured code.

---

## 3. Theory

**Core idea:** LLD translates a functional requirement into a concrete set of classes, interfaces, and their relationships — such that the resulting code is correct, and also maintainable and extensible over time.

**Working mechanism — the typical LLD process:**
```
1. Gather requirements (functional + non-functional)
2. Identify the core entities/nouns (these often become classes)
   e.g., "Design a parking lot" → ParkingLot, ParkingSpot, Vehicle, Ticket
3. Identify the actions/verbs (these often become methods)
   e.g., parkVehicle(), calculateFee(), issueTicket()
4. Identify relationships between entities
   e.g., ParkingLot HAS-MANY ParkingSpots (composition)
        Vehicle IS-A Car/Bike/Truck (inheritance)
5. Apply OOP principles + SOLID principles to refine the design
6. Identify where design patterns naturally fit
   e.g., different vehicle types needing different fee calculation
        → Strategy pattern
7. Draw the UML class diagram
8. Implement in code
```

**Important terminology:**

| Term | Meaning |
|---|---|
| Class | A blueprint defining attributes and behaviors |
| Object | A specific instance of a class |
| Interface | A contract defining WHAT a class must do, not HOW |
| Abstract class | A partial implementation, some methods defined, some left abstract |
| Association | A general "uses" relationship between classes |
| Aggregation | A "has-a" relationship where the parts can exist independently of the whole |
| Composition | A stronger "has-a" relationship where the parts CANNOT exist without the whole |

**Class responsibilities:** Every class should have a single, clear reason to exist (this foreshadows the Single Responsibility Principle, covered in Topic 3).

**Communication flow:** Objects communicate by calling each other's public methods — never reaching into another object's internal state directly. This is **encapsulation**, one of the four pillars of OOP (covered fully in Topic 2).

---

## 4. UML / Class Diagram

Since Topic 1 is conceptual (not a specific pattern), here's a simple illustrative diagram showing the RELATIONSHIP TYPES you'll see used throughout every future topic:

```
┌────────────────┐
│  <<interface>>       │
│  PaymentMethod           │
├────────────────┤
│ + pay(amount): void       │
└────────┬───────┘
         △  (implements — dashed line + hollow triangle)
         │
┌────────┴───────┐
│  CreditCardPayment       │
└────────────────┘

┌────────────────┐         ┌────────────────┐
│  Car                       │ ──────▷│  Engine                     │  ← Association
│  (uses Engine)               │         │                                │     (Car "uses a" Engine)
└────────────────┘         └────────────────┘

┌────────────────┐         ┌────────────────┐
│  Library                       │ ◇──────│  Book                          │  ← Aggregation
│  (has books, but                 │         │  (can exist WITHOUT                │     (hollow diamond)
│   books can exist                  │         │   the library existing)                │
│   independently)                     │         └────────────────┘
└────────────────┘

┌────────────────┐         ┌────────────────┐
│  House                          │ ◆──────│  Room                          │  ← Composition
│  (owns rooms; rooms                 │         │  (CANNOT exist without              │     (filled diamond)
│   die when house is                    │         │   the house existing)                    │
│   destroyed)                              │         └────────────────┘
└────────────────┘
```

**Explaining each relationship:**
- **Association**: "Car uses an Engine" — a general relationship, weakest coupling
- **Aggregation**: "Library has Books" — but a Book can exist without ANY library owning it (e.g., you can have a personal book not in any library)
- **Composition**: "House has Rooms" — a Room cannot meaningfully exist if the House is destroyed; strong ownership, lifecycle-bound
- **Inheritance** (△ hollow triangle): "CreditCardPayment IS-A PaymentMethod" — implements the interface's contract
- **Dependency** (dashed arrow, not shown above): "ClassA temporarily uses ClassB" — e.g., a method parameter, weakest and most temporary relationship

Understanding these distinctions precisely is something interviewers frequently probe — expect a question like "what's the difference between aggregation and composition?" in nearly every LLD interview.

---

## 5. Java Implementation

Since Topic 1 has no single "pattern" to implement, here's a **minimal illustrative example** showing how a plain, unstructured (LLD-less) approach compares to a properly structured one — setting up the mental model for every topic that follows.

```java
// ============================================
// A tiny illustrative domain: a simple Order system
// ============================================

/**
 * Represents an item within an order.
 * Encapsulation: fields are private, accessed only via methods.
 */
public class OrderItem {
    private final String name;
    private final double price;
    private final int quantity;

    public OrderItem(String name, double price, int quantity) {
        this.name = name;
        this.price = price;
        this.quantity = quantity;
    }

    // A method, not a public field — encapsulation in action
    public double getSubtotal() {
        return price * quantity;
    }

    public String getName() {
        return name;
    }
}

/**
 * Represents a customer's order — composed of OrderItems.
 * This is COMPOSITION: OrderItems don't meaningfully exist
 * outside the context of an Order in this domain.
 */
public class Order {
    private final String orderId;
    private final java.util.List<OrderItem> items;

    public Order(String orderId) {
        this.orderId = orderId;
        this.items = new java.util.ArrayList<>();
    }

    public void addItem(OrderItem item) {
        items.add(item);
    }

    // Single responsibility: this class knows how to calculate ITS OWN total
    public double calculateTotal() {
        double total = 0;
        for (OrderItem item : items) {
            total += item.getSubtotal();
        }
        return total;
    }

    public String getOrderId() {
        return orderId;
    }
}

/**
 * Demonstrates usage — this would typically be a separate
 * "driver" or service class, not mixed into the domain classes.
 */
public class OrderDemo {
    public static void main(String[] args) {
        Order order = new Order("ORD-1001");
        order.addItem(new OrderItem("Laptop", 999.99, 1));
        order.addItem(new OrderItem("Mouse", 25.50, 2));

        System.out.println("Order " + order.getOrderId() +
                            " total: $" + order.calculateTotal());
    }
}
```

**Line-by-line explanation of key design choices:**
- `private final String name` — fields are private and final where possible: encapsulation + immutability, preventing external code from corrupting internal state.
- `OrderItem` has NO knowledge of `Order` — it doesn't know or care what order it belongs to. This is intentional **loose coupling**: `OrderItem` can be tested, reused, or reasoned about completely independently.
- `Order.calculateTotal()` — the responsibility for calculating a total belongs to `Order`, not to some external utility function that reaches into `Order`'s internals. This is the **Single Responsibility Principle** in action, previewing Topic 3.
- The `OrderDemo` class is separate from the domain classes (`Order`, `OrderItem`) — driver/demonstration code should never be mixed into your core domain model classes.

---

## 6. Dry Run

**Sample input:** Creating an order with a laptop (qty 1) and a mouse (qty 2).

**Step-by-step execution:**
```
1. new Order("ORD-1001")
   → A new Order object is created in memory.
   → orderId = "ORD-1001", items = empty ArrayList

2. new OrderItem("Laptop", 999.99, 1)
   → A new OrderItem object created: name="Laptop", price=999.99, quantity=1
   → This object exists independently in memory, referenced by
     nothing yet except the local temporary reference in main()

3. order.addItem(laptopItem)
   → The Order object's internal `items` list now holds a
     REFERENCE to the laptopItem object (not a copy)
   → items = [laptopItem]

4. new OrderItem("Mouse", 25.50, 2) + order.addItem(mouseItem)
   → items = [laptopItem, mouseItem]

5. order.calculateTotal() is called
   → total = 0
   → Loop iteration 1: item = laptopItem
        → item.getSubtotal() called → returns 999.99 * 1 = 999.99
        → total = 999.99
   → Loop iteration 2: item = mouseItem
        → item.getSubtotal() called → returns 25.50 * 2 = 51.00
        → total = 1050.99
   → returns 1050.99

6. System.out.println(...) prints:
   "Order ORD-1001 total: $1050.99"
```

**What's happening in memory:** `Order` holds references (pointers) to `OrderItem` objects, not copies of their data. Each `getSubtotal()` call is a normal (non-polymorphic, since there's no inheritance here) method dispatch — the JVM knows exactly which method to call at compile time since `OrderItem` isn't part of any inheritance hierarchy in this simple example. (We'll see runtime polymorphism/dynamic dispatch in detail once we reach Strategy, Factory, and other patterns involving interfaces with multiple implementations.)

---

## 7. Real-World Software Example

- **Amazon's order/checkout system**: internally composed of dozens of classes like `Cart`, `CartItem`, `PricingEngine`, `TaxCalculator`, `ShippingCalculator` — each with a single responsibility, composed together. This is LLD applied at massive scale.
- **Uber's ride-matching system**: classes like `Driver`, `Rider`, `Trip`, `PricingStrategy`, `MatchingAlgorithm` — where different pricing/matching strategies are swapped using patterns you'll learn in this series (Strategy, Factory).
- **Every Spring Boot application** you've ever used is a giant, well-structured LLD exercise: `Controller`, `Service`, `Repository` classes each with single, clear responsibilities, wired together via dependency injection.

---

## 8. Internal Working

For this conceptual topic, "internal working" is about how the **JVM's object model** supports the LLD principles we rely on:

- **Object creation**: every `new` keyword allocates memory on the **heap** for the object's fields; the reference (a pointer) is what's stored on the stack (for local variables) or within other objects.
- **Method dispatch**: for non-overridden, non-polymorphic method calls (like `getSubtotal()` above), the JVM resolves the method at compile time — this is called **static binding**. Once we introduce interfaces and inheritance (starting with OOP Principles in Topic 2, and every pattern after), method calls become resolved at RUNTIME based on the object's actual type — this is **dynamic binding**, the mechanism that makes polymorphism and nearly every design pattern in this series actually work.
- **Encapsulation enforcement**: `private` fields are enforced by the JVM's access control at compile time — attempting to access `order.items` directly from outside the class would be a compile error, not just a style violation.

---

## 9. Before vs After

**Before (no LLD discipline — a "God class" anti-pattern):**

```java
public class OrderProcessor {
    public double processOrder(String[] itemNames, double[] prices, int[] quantities,
                                 String paymentType, String cardNumber, String upiId,
                                 boolean sendEmail, boolean sendSms) {
        double total = 0;
        for (int i = 0; i < itemNames.length; i++) {
            total += prices[i] * quantities[i];
        }

        if (paymentType.equals("CARD")) {
            System.out.println("Charging card " + cardNumber + " amount: " + total);
        } else if (paymentType.equals("UPI")) {
            System.out.println("Charging UPI " + upiId + " amount: " + total);
        }

        if (sendEmail) {
            System.out.println("Sending email confirmation...");
        }
        if (sendSms) {
            System.out.println("Sending SMS confirmation...");
        }

        return total;
    }
}
```

**Problems with this:**
- **One method does everything**: calculating totals, processing payment, AND sending notifications — violates single responsibility badly.
- **Parallel arrays** (`itemNames`, `prices`, `quantities`) instead of a proper `OrderItem` class — fragile, error-prone, no encapsulation.
- **Adding a new payment type** (say, "WALLET") means editing this method directly — modifying existing, already-tested code, risking regression (violates Open/Closed Principle, Topic 3).
- **Untestable**: you can't test "payment charging" independently from "total calculation" — they're welded together in one method.
- **String-based type checking** (`paymentType.equals("CARD")`) instead of polymorphism — this pattern will come up again and again as the exact anti-pattern that Strategy and Factory patterns solve.

**After (proper LLD, previewing patterns to come):**

```java
// Each responsibility is its own class/interface
public interface PaymentMethod {
    void pay(double amount);
}

public class CreditCardPayment implements PaymentMethod {
    private final String cardNumber;
    public CreditCardPayment(String cardNumber) { this.cardNumber = cardNumber; }
    public void pay(double amount) {
        System.out.println("Charging card " + cardNumber + " amount: " + amount);
    }
}

public interface NotificationService {
    void notify(String message);
}

public class EmailNotification implements NotificationService {
    public void notify(String message) {
        System.out.println("Email: " + message);
    }
}

// The orchestrating class depends on ABSTRACTIONS, not concrete details
public class OrderProcessor {
    private final PaymentMethod paymentMethod;
    private final NotificationService notificationService;

    public OrderProcessor(PaymentMethod paymentMethod, NotificationService notificationService) {
        this.paymentMethod = paymentMethod;
        this.notificationService = notificationService;
    }

    public void processOrder(Order order) {
        paymentMethod.pay(order.calculateTotal());
        notificationService.notify("Order " + order.getOrderId() + " confirmed!");
    }
}
```

**Why this is better:**
- Adding a new payment type = adding a NEW class implementing `PaymentMethod` — **zero changes** to `OrderProcessor`. This is the Open/Closed Principle.
- Each class is independently testable — you can test `CreditCardPayment` without ever touching `OrderProcessor`.
- `OrderProcessor` depends on the `PaymentMethod` **interface**, not a specific implementation — this is Dependency Inversion, and it's exactly the mechanism behind the Strategy pattern (Topic 5).

---

## 10. SOLID Principles Connection

Since Topic 1 is foundational, here's a preview of how the "After" example above already touches every SOLID principle (each gets its own deep dive in Topics 3-4):

- **SRP (Single Responsibility)**: `CreditCardPayment` only handles payment; `EmailNotification` only handles notifications; `OrderProcessor` only orchestrates.
- **OCP (Open/Closed)**: Adding `UpiPayment` requires zero changes to `OrderProcessor` — it's open for extension, closed for modification.
- **LSP (Liskov Substitution)**: Any `PaymentMethod` implementation can be substituted for another without breaking `OrderProcessor`'s expectations.
- **ISP (Interface Segregation)**: `PaymentMethod` has exactly one method — no class is forced to implement methods it doesn't need.
- **DIP (Dependency Inversion)**: `OrderProcessor` depends on the `PaymentMethod` abstraction, not on a concrete class like `CreditCardPayment` directly.

---

## 11. Interview Questions

**Beginner:**
1. What is the difference between High-Level Design and Low-Level Design?
   *Answer: HLD defines system components/architecture (services, databases); LLD defines the internal class-level structure implementing each component.*
2. Why do we need design patterns at all — why not just write code that works?
   *Answer: Working code isn't the same as maintainable, extensible code. Patterns solve recurring design problems in proven, well-understood ways, making code easier to extend and for other engineers to understand.*
3. What is encapsulation, and why does it matter?
   *Answer: Hiding internal state behind private fields, exposing only necessary behavior via public methods — prevents external code from corrupting internal state and allows internal implementation to change without breaking callers.*

**Intermediate:**
4. What's the difference between aggregation and composition? Give an example of each.
5. Why is "programming to an interface, not an implementation" considered good practice?
6. What is coupling, and why is loose coupling desirable?
7. Explain the difference between a class and an object with an example.

**Advanced:**
8. How would you approach an LLD interview question like "Design a parking lot" from scratch?
   *Answer: Identify nouns → classes (ParkingLot, ParkingSpot, Vehicle), identify verbs → methods (parkVehicle, calculateFee), identify relationships (composition: ParkingLot has ParkingSpots), then look for varying behavior that suggests a design pattern (different vehicle types/fee structures → Strategy).*
9. When would you deliberately choose NOT to apply a design pattern, even if one technically fits?
   *Answer: When the added abstraction/indirection isn't justified by genuine anticipated complexity — over-engineering a simple, unlikely-to-change piece of code hurts readability without real benefit.*
10. How does LLD relate to testability?
11. What's the risk of a "God class," and how do you identify one during code review?
12. Explain static binding vs dynamic binding and how this relates to polymorphism in Java.

---

## 12. Common Mistakes

- **Jumping straight to code without identifying classes/relationships first** — leads to a design that's discovered ad-hoc rather than deliberately structured; in interviews, this reads as disorganized thinking.
- **Over-engineering simple problems** — applying 4 design patterns to a problem that needs one class; interviewers notice unjustified complexity as much as they notice missing structure.
- **Confusing HLD and LLD scope** — spending interview time discussing load balancers and databases when asked an LLD question that wants class diagrams and method signatures.
- **Ignoring non-functional requirements** — e.g., not considering thread-safety in a Singleton (Topic 16) because the LLD discussion focused purely on the happy path.
- **Not asking clarifying questions before designing** — a "Design a parking lot" question has many valid designs depending on constraints (multiple floors? multiple vehicle types? payment integration?) — assuming instead of asking is a common interview misstep.

---

## 13. Time Complexity

Not directly applicable to this conceptual topic — LLD is about structure and maintainability, not algorithmic complexity. (Time/space complexity becomes relevant in specific patterns later, e.g., Flyweight's memory savings, or Iterator's traversal complexity.)

---

## 14. Java API Examples

Java's own standard library is a masterclass in LLD:
- **Collections Framework**: `List` interface with `ArrayList`/`LinkedList` implementations — programming to an interface, swappable implementations (previews Strategy/Polymorphism).
- **`java.io`**: `InputStream`/`OutputStream` hierarchies use Decorator (Topic 7) extensively — e.g., wrapping a `FileInputStream` with a `BufferedInputStream`.
- **JDBC**: `DriverManager` uses Factory pattern (Topic 8) to return the correct `Connection` implementation for whichever database driver is registered.
- **`java.util.Comparator`**: a textbook real-world use of the Strategy pattern (Topic 5) — different sorting strategies plugged into `Collections.sort()`.

---

## 15. Practice Problem

Take the "Before" `OrderProcessor` God-class example from Section 9. Without looking ahead to future topics, try refactoring it yourself:
- Identify at least 3 separate responsibilities currently bundled into one method.
- Sketch (in comments or a simple diagram) what classes/interfaces you'd introduce.
- Don't worry about using "correct" design pattern names yet — just focus on separating responsibilities and removing the string-based `if/else` type-checking.

---

## 16. Medium-Level Exercise

*(Not applicable to this foundational topic — this section will be used starting from pattern-specific topics.)*

---

## 17. Advanced LLD Scenario

*(Not applicable to this foundational topic — this section will be used starting from pattern-specific topics.)*

---

## 18. Summary

**Definition:** LLD is the process of designing a system's internal class-level structure — classes, interfaces, relationships, and interactions — to implement functionality defined at the HLD level.

**Intent:** Produce maintainable, extensible, testable code by deliberately structuring responsibilities before writing implementation.

**Key classes:** N/A (foundational topic — applies to every class you'll ever design).

**Advantages:** Modular, extensible, testable, maintainable code; supports team collaboration; directly tested in SDE interviews.

**Disadvantages:** Risk of over-engineering; more upfront design time; unnecessary indirection if overused on simple problems.

**Real-world use case:** Every production system of nontrivial size — Amazon's checkout flow, Uber's ride-matching, any Spring Boot application.

**Java example:** Refactoring a God-class `OrderProcessor` into `PaymentMethod`, `NotificationService`, and `OrderProcessor` (orchestrator) — previewing Strategy and Dependency Inversion.

**Interview takeaway:** Always identify nouns (classes), verbs (methods), and relationships (association/aggregation/composition) before writing code in an LLD interview — this demonstrates structured thinking, which interviewers explicitly look for.

**One-line memory trick:** *"HLD is the blueprint of the house; LLD is the carpentry plan for each room."*

---

*End of Topic 1. Type "Next" to proceed to Topic 2: OOP Principles.*