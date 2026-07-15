# Topic 5: Strategy Design Pattern

---

## 1. Introduction

**Definition:**
The Strategy Pattern is a **behavioral design pattern** that defines a family of interchangeable algorithms, encapsulates each one in its own class, and makes them interchangeable at runtime — allowing the algorithm to vary independently from the code that uses it.

**Why it exists / what problem it solves:**
Without Strategy, varying behavior is typically implemented using conditional logic (`if/else` or `switch` on a "type" flag) directly inside a class — exactly the anti-pattern flagged repeatedly in Topics 1 and 3. This creates:
- A class that must be MODIFIED every time a new variant of the behavior is needed (violates OCP)
- Duplicated conditional logic if multiple places in the codebase need to make the same "which algorithm" decision
- Difficult unit testing, since testing ONE algorithm variant means invoking the whole conditional-laden method

Strategy solves this by extracting each algorithm variant into its OWN class implementing a common interface, and having the "context" class hold a REFERENCE to whichever strategy is currently active — swappable at runtime, with zero conditional logic in the context class itself.

**When it should be used:**
- When you have multiple ways of performing the same conceptual operation (different sorting algorithms, different payment methods, different pricing/discount rules, different route-finding algorithms)
- When you want to eliminate large conditional blocks that select behavior based on a type/flag
- When different clients might need different variants of an algorithm at different times, even for the SAME object

**When it should NOT be used:**
- When there's genuinely only ONE way to do something, and no realistic expectation of variation — introducing a Strategy interface for a single, unchanging implementation is needless indirection
- When the number of strategies is very small (2) and unlikely to grow, a simple `if/else` may honestly be clearer and is not automatically wrong

**Advantages:**
- Eliminates conditional complexity in the context class
- New strategies can be added without modifying existing code (OCP)
- Each strategy is independently testable
- Strategies can be swapped at RUNTIME, not just configured once at compile time

**Disadvantages:**
- Increases the number of classes in the codebase (one per strategy)
- Client code must be aware of the different strategies to choose the appropriate one (though this can be mitigated by combining Strategy with Factory, covered in Topic 8)

---

## 2. Real-World Analogy

Think of **Google Maps route selection**.

When you ask for directions, Google Maps can compute your route using different STRATEGIES:
- **Fastest route** (optimizes for time)
- **Shortest route** (optimizes for distance)
- **Avoid tolls** (optimizes to exclude toll roads)
- **Avoid highways** (optimizes to exclude highways)

The MAP APPLICATION itself (the "context") doesn't change — it still shows you a route from A to B. What changes is WHICH ALGORITHM is used to calculate that route, and you can switch between these algorithms at any time (even mid-trip) without needing a different app. Each routing algorithm is a self-contained "strategy" that the map app delegates to, and the app doesn't need to know the internal details of how each one works — it just calls "calculate route" and gets a result back.

---

## 3. Theory

**Core idea:** Define a family of algorithms, encapsulate each as a class implementing a common interface, and make the CONTEXT class hold a reference to the currently active strategy — delegating the actual work to it, rather than implementing the logic itself.

**Working mechanism:**
```
┌─────────────────────┐
│      Context                    │
│  - strategy: Strategy (interface)  │  ← holds a REFERENCE to
│  + setStrategy(s)                    │     an interface, not a
│  + executeStrategy()                    │     concrete implementation
└──────────┬──────────┘
           │ delegates to
           ↓
┌─────────────────────┐
│      <<interface>>          │
│         Strategy                 │
│  + execute(): Result               │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────┐
   │               │
┌──┴────┐    ┌──┴────┐
│StrategyA │    │StrategyB │
└────────┘    └────────┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Context | The class that USES a strategy, delegating work to it |
| Strategy interface | The common contract all algorithm variants implement |
| Concrete Strategy | A specific algorithm implementation |

**Class responsibilities:**
- The **Context** doesn't know or care HOW a strategy does its job — only that it can call a common method and get a result.
- Each **Concrete Strategy** is fully self-contained and independently swappable.

**Communication flow:** Client code creates a `Context`, supplies (injects) a specific `Strategy` implementation (often via constructor or a setter), and calls the Context's method — which internally delegates to whichever strategy was supplied. This is DEPENDENCY INJECTION (Topic 3's DIP) applied specifically to swap BEHAVIOR, not just data sources.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      PaymentContext            │
├─────────────────────┤
│ - paymentStrategy: PaymentStrategy ◇──────┐
├─────────────────────┤                  │
│ + setPaymentStrategy(s)               │
│ + checkout(amount)                          │
└─────────────────────┘                  │
                                            │ (association/aggregation:
                                            │  Context HOLDS a Strategy,
                                            │  but doesn't OWN its lifecycle)
                                            ↓
                              ┌─────────────────────┐
                              │      <<interface>>          │
                              │      PaymentStrategy            │
                              ├─────────────────────┤
                              │ + pay(amount): void               │
                              └──────────┬──────────┘
                                         △ (implements)
                                 ┌───────┼───────┐
                                 │               │
                       ┌────────┴──┐    ┌────┴────────┐
                       │CreditCardPayment │    │  UpiPayment     │
                       └──────────┘    └────────────┘
```

**Relationship explanation:**
- `PaymentContext` has an **aggregation** relationship with `PaymentStrategy` (hollow diamond) — the Context HOLDS a reference to a strategy, but the strategy object can exist and be created independently of any specific Context (e.g., you might create a `CreditCardPayment` object before deciding which context will use it).
- `CreditCardPayment` and `UpiPayment` **implement** `PaymentStrategy` (hollow triangle, dashed line) — this is the polymorphic swap point.
- The KEY structural signal of Strategy: the Context's field is typed as the INTERFACE, never a concrete class.

---

## 5. Java Implementation

```java
// ============================================
// Strategy interface — the common contract
// ============================================
public interface PaymentStrategy {
    void pay(double amount);
}

// ============================================
// Concrete Strategies — each encapsulates ONE
// specific payment algorithm/mechanism
// ============================================
public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;
    private final String cvv;

    public CreditCardPayment(String cardNumber, String cvv) {
        this.cardNumber = cardNumber;
        this.cvv = cvv;
    }

    @Override
    public void pay(double amount) {
        // In reality, this would call a payment gateway API
        System.out.printf("Paid %.2f using Credit Card ending in %s%n",
                amount, cardNumber.substring(cardNumber.length() - 4));
    }
}

public class UpiPayment implements PaymentStrategy {
    private final String upiId;

    public UpiPayment(String upiId) {
        this.upiId = upiId;
    }

    @Override
    public void pay(double amount) {
        System.out.printf("Paid %.2f using UPI ID: %s%n", amount, upiId);
    }
}

public class PayPalPayment implements PaymentStrategy {
    private final String email;

    public PayPalPayment(String email) {
        this.email = email;
    }

    @Override
    public void pay(double amount) {
        System.out.printf("Paid %.2f using PayPal account: %s%n", amount, email);
    }
}

// ============================================
// Context — delegates payment processing to
// whichever strategy is currently set, WITHOUT
// knowing or caring how each one actually works
// ============================================
public class ShoppingCart {
    private PaymentStrategy paymentStrategy; // holds the INTERFACE type
    private double totalAmount;

    public ShoppingCart() {
        this.totalAmount = 0;
    }

    public void addItem(double price) {
        totalAmount += price;
    }

    // Strategy can be set/changed at RUNTIME
    public void setPaymentStrategy(PaymentStrategy paymentStrategy) {
        this.paymentStrategy = paymentStrategy;
    }

    public void checkout() {
        if (paymentStrategy == null) {
            throw new IllegalStateException("Payment strategy not set");
        }
        // Delegation: ShoppingCart doesn't know HOW payment happens,
        // just that it CAN happen via the pay() method
        paymentStrategy.pay(totalAmount);
    }
}

// ============================================
// Demo
// ============================================
public class StrategyDemo {
    public static void main(String[] args) {
        ShoppingCart cart = new ShoppingCart();
        cart.addItem(499.99);
        cart.addItem(149.50);

        // Choosing Credit Card at checkout
        cart.setPaymentStrategy(new CreditCardPayment("1234567812345678", "123"));
        cart.checkout();

        // The SAME cart, but the customer changes their mind
        // and pays via UPI instead — swapped at runtime,
        // zero changes needed to ShoppingCart's code
        cart.setPaymentStrategy(new UpiPayment("user@bank"));
        cart.checkout();
    }
}
```

**Key line-by-line notes:**
- `private PaymentStrategy paymentStrategy` — the field type is the INTERFACE, never a concrete class; this is the structural heart of the pattern.
- `setPaymentStrategy()` — allows changing the strategy AT RUNTIME, not just once at construction — this is what distinguishes Strategy from simply passing a fixed dependency via constructor (though constructor injection is also a valid way to supply a Strategy if runtime swapping isn't needed).
- `checkout()` calls `paymentStrategy.pay(totalAmount)` — a single line of delegation; `ShoppingCart` has ZERO knowledge of card numbers, UPI IDs, or PayPal emails.
- Adding `PayPalPayment` required ZERO changes to `ShoppingCart` — this is the Open/Closed Principle (Topic 3) directly realized through the Strategy pattern.

---

## 6. Dry Run

**Sample input:** Running `StrategyDemo.main()`.

```
1. new ShoppingCart()
   → Heap allocates a ShoppingCart object; totalAmount = 0,
     paymentStrategy = null

2. cart.addItem(499.99) → totalAmount = 499.99
   cart.addItem(149.50) → totalAmount = 649.49

3. new CreditCardPayment("1234567812345678", "123")
   → Heap allocates a CreditCardPayment object

4. cart.setPaymentStrategy(creditCardPaymentInstance)
   → cart.paymentStrategy now REFERENCES the CreditCardPayment
     object (field type is PaymentStrategy, actual type is
     CreditCardPayment)

5. cart.checkout() called
   → paymentStrategy is not null, proceeds
   → paymentStrategy.pay(649.49) called
   → Dynamic dispatch (Topic 2): JVM checks the ACTUAL type
     behind paymentStrategy → CreditCardPayment
   → Invokes CreditCardPayment's pay() method
   → Prints: "Paid 649.49 using Credit Card ending in 5678"

6. new UpiPayment("user@bank") + cart.setPaymentStrategy(upiPaymentInstance)
   → cart.paymentStrategy NOW references the UpiPayment object
     instead — the PREVIOUS CreditCardPayment reference is
     overwritten (eligible for garbage collection if nothing
     else references it)

7. cart.checkout() called AGAIN
   → paymentStrategy.pay(649.49) called
   → Dynamic dispatch resolves to UpiPayment this time
   → Prints: "Paid 649.49 using UPI ID: user@bank"
```

**What's happening in memory:** `ShoppingCart`'s `checkout()` METHOD BODY never changed between steps 5 and 7 — the exact same line of code (`paymentStrategy.pay(totalAmount)`) produced different behavior purely because the OBJECT referenced by `paymentStrategy` changed. This is Strategy's core mechanic: swapping BEHAVIOR by swapping which object a reference points to, leveraging polymorphism.

---

## 7. Real-World Software Example

- **Payment methods** (as implemented above) — nearly every e-commerce checkout system uses this exact pattern.
- **`java.util.Comparator`** — sorting strategies. `Collections.sort(list, comparator)` — the SAME sort algorithm (context) delegates the actual comparison LOGIC (strategy) to whatever `Comparator` is passed in.
- **Compression algorithms** — a file archiver might support ZIP, RAR, or 7z compression strategies, chosen at runtime based on user preference or file type.
- **Route calculation** (the Google Maps analogy) — fastest/shortest/toll-avoiding route strategies.
- **Validation strategies** — different validation rules for different form types, each implementing a common `Validator` interface.

---

## 8. Internal Working

**Object creation:** each Concrete Strategy (`CreditCardPayment`, `UpiPayment`) is a normal object allocated on the heap, exactly like any other object — there's nothing special about strategy objects at the memory level.

**Method dispatch:** the KEY mechanism is dynamic dispatch (Topic 2's vtable lookup) — when `paymentStrategy.pay(amount)` is called, the JVM looks up the ACTUAL object's class (not the `PaymentStrategy` interface's declared type) via its vtable, and invokes the correct implementation. This is EXACTLY the same underlying mechanism that makes ALL polymorphism work — Strategy is essentially "polymorphism, deliberately structured as a reusable design pattern."

**Memory consideration:** if a Context object holds a reference to a Strategy object, and that reference is later reassigned (as in step 6 of the dry run), the PREVIOUS strategy object becomes eligible for garbage collection — assuming nothing else in the program still holds a reference to it.

---

## 9. Before vs After

**Before (conditional-logic anti-pattern):**

```java
public class ShoppingCartBad {
    private double totalAmount;

    public void checkout(String paymentType, String detail1, String detail2) {
        if (paymentType.equals("CREDIT_CARD")) {
            System.out.println("Paying " + totalAmount + " via card " + detail1);
        } else if (paymentType.equals("UPI")) {
            System.out.println("Paying " + totalAmount + " via UPI " + detail1);
        } else if (paymentType.equals("PAYPAL")) {
            System.out.println("Paying " + totalAmount + " via PayPal " + detail1);
        }
        // Adding a new payment method means EDITING this method,
        // and the confusing detail1/detail2 params mean different
        // things depending on paymentType — fragile and unclear
    }
}
```

**Problems:**
- Adding a new payment method requires modifying `checkout()` directly (violates OCP).
- `detail1`/`detail2` parameters have AMBIGUOUS meaning depending on `paymentType` — error-prone.
- Testing "credit card payment logic" in isolation is impossible; it's welded into this one large method alongside every other payment type.

**After (Strategy pattern, as shown in Section 5):**
- Each payment method is its own class, with clearly-named, type-safe fields (`cardNumber`, `upiId`, `email`) instead of ambiguous generic parameters.
- Adding `PayPalPayment` (or any new method) requires ZERO changes to `ShoppingCart`.
- Each strategy class is independently unit-testable.

---

## 10. SOLID Principles Connection

- **OCP**: new strategies (payment methods) are added via new classes, never by modifying `ShoppingCart` or the `PaymentStrategy` interface.
- **SRP**: `ShoppingCart` is responsible for cart/checkout orchestration only; each `PaymentStrategy` implementation is responsible only for its own payment mechanism.
- **LSP**: any `PaymentStrategy` implementation can be substituted into `ShoppingCart` without breaking its behavior — assuming each implementation genuinely fulfills the simple "process a payment of this amount" contract.
- **DIP**: `ShoppingCart` depends on the `PaymentStrategy` ABSTRACTION, never on a concrete class — this is DIP applied specifically to swappable BEHAVIOR.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Strategy pattern solve?
2. What's the difference between the "Context" and the "Strategy" in this pattern?
3. Give a real-world example of the Strategy pattern outside of payment processing.

**Intermediate:**
4. How does Strategy help satisfy the Open/Closed Principle?
5. Can a Strategy be changed AFTER the Context object has already been created? How?
6. What's the difference between Strategy and simply using an `if/else` block? When might `if/else` actually be preferable?
   *Answer: If/else is preferable when there are only 2-3 stable, unlikely-to-grow variants, and the added class overhead of Strategy isn't justified. Strategy shines when the number of variants is larger or expected to grow.*

**Advanced:**
7. **Strategy vs State pattern** — how are they structurally similar, and what's the key CONCEPTUAL difference?
   *Answer: Structurally, both involve a Context holding a reference to an interface with multiple implementations. The difference is INTENT: Strategy is about the CLIENT/caller choosing an algorithm explicitly (e.g., "use UPI payment"); State (Topic 13) is about the OBJECT ITSELF transitioning between states autonomously based on its own internal logic (e.g., a traffic light transitioning from Red to Green on its own). In Strategy, the context rarely changes strategies on its own; in State, state transitions are often triggered internally.*
8. How would you combine Strategy with the Factory pattern (Topic 8) to avoid client code needing to know about every concrete strategy class?
9. Is Strategy compatible with dependency injection frameworks like Spring? How would you implement runtime-configurable strategy selection in Spring?
10. What are the tradeoffs of using an enum with a `switch` statement versus a full Strategy pattern implementation?
11. How does Java's `Comparator` interface exemplify the Strategy pattern in the standard library?

---

## 12. Common Mistakes

- **Putting logic in the Context that should belong in a Strategy** — if `ShoppingCart.checkout()` starts having `if (paymentStrategy instanceof CreditCardPayment)` checks, the whole point of the pattern has been defeated; the Context should NEVER need to know the concrete type of its strategy.
- **Creating a Strategy interface with too many methods** — if different strategies genuinely can't all meaningfully implement every method in the interface, that's an ISP violation (Topic 3) hiding inside your Strategy design.
- **Forgetting that strategies can be swapped at runtime** — some implementations only inject a strategy once via constructor and never allow it to change, missing the DYNAMIC flexibility that's part of the core value proposition (though this is a valid choice if runtime swapping genuinely isn't needed).
- **Over-applying Strategy to a genuinely single, unchanging algorithm** — not every method that "does one thing" needs to be extracted into a Strategy interface; apply this pattern where REAL variation exists or is reasonably anticipated.

---

## 13. Time Complexity

Not applicable in a general sense — Strategy is a structural pattern, and the time complexity depends entirely on WHICH concrete strategy is executing (e.g., a "shortest path" routing strategy might be O(V log V) with Dijkstra's algorithm, while a "fastest path" strategy considering live traffic might have different complexity). The PATTERN itself adds only O(1) overhead (a single virtual method dispatch) on top of whatever the chosen strategy's own complexity is.

---

## 14. Java API Examples

- **`java.util.Comparator`**: the quintessential Strategy example — `list.sort(comparator)` delegates the comparison LOGIC to whatever `Comparator` implementation is passed.
- **`java.util.concurrent.ThreadPoolExecutor`**: accepts a `RejectedExecutionHandler` strategy, determining what happens when a task is submitted but the pool is saturated (discard, throw exception, run in caller's thread, etc.) — different strategies for the SAME underlying "what to do when rejected" problem.
- **`javax.servlet.Filter`** chains in Java web frameworks: each filter can be seen as a strategy for handling a request, composed together.
- **Spring's `PlatformTransactionManager`**: different implementations (JPA, JDBC, JTA) provide different transaction-handling strategies, injected wherever transaction management is needed.

---

## 15. Practice Problem

Implement a **text formatting system** with a `TextFormatter` strategy interface, and at least three concrete strategies: `UpperCaseFormatter`, `LowerCaseFormatter`, and `TitleCaseFormatter` (capitalizing the first letter of each word). Build a `TextEditor` context class that holds a `TextFormatter` and has a method `format(String text)` that delegates to the current strategy. Demonstrate swapping strategies at runtime on the same `TextEditor` instance.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a ride-fare calculation system where fares can be calculated using different pricing strategies: `NormalPricing` (base rate × distance), `SurgePricing` (base rate × distance × surge multiplier), and `PromotionalPricing` (base rate × distance, with a flat discount applied). The system should support switching pricing strategy based on real-time conditions (e.g., high demand triggers surge pricing) without modifying the core fare calculation flow."

Think about:
- What does the `Context` class look like here — is it the `Ride` object, or a separate `FareCalculator`?
- How would REAL-TIME conditions (like demand level) determine which strategy gets selected? Does this suggest combining Strategy with another pattern?

---

## 17. Advanced LLD Scenario

**Design a Ride-Sharing Matching & Pricing Engine** (like Uber) where:
- Different cities/regions may use different DRIVER-MATCHING algorithms (nearest-driver-first, highest-rated-driver-first, load-balanced across drivers)
- Different fare calculation strategies apply based on ride type (Economy, Premium, Pool) — connecting back to the Advanced Scenario in Topic 3
- The system should support A/B testing NEW matching algorithms for a subset of users without disrupting the existing, proven algorithm for everyone else

Consider:
- How does Strategy make A/B testing a new matching algorithm straightforward — what would need to change to route 5% of users to a `NewMatchingStrategy` while everyone else uses `StandardMatchingStrategy`?
- How would you avoid the `RideService` (context) ever needing an `if (strategy instanceof NewMatchingStrategy)` check — keeping the A/B testing DECISION logic separate from the actual matching/fare logic itself?

*(This scenario shows Strategy combined with real production concerns like gradual rollouts — the same conceptual pattern used in your Istio/Service Mesh canary deployment notes, just applied at the application code level instead of the infrastructure/traffic-routing level.)*

---

## 18. Summary

**Definition:** Strategy defines a family of interchangeable algorithms, encapsulating each in its own class, allowing the algorithm to vary independently from the client using it.

**Intent:** Eliminate conditional logic for selecting behavior; enable adding new algorithm variants without modifying existing code.

**Key classes:** `Context` (holds and delegates to a strategy), `Strategy` interface (the common contract), `ConcreteStrategy` implementations (the actual algorithms).

**Advantages:** OCP-compliant extensibility, independently testable algorithms, runtime swappability.

**Disadvantages:** Increases class count; clients must know which strategy to choose (mitigated by combining with Factory).

**Real-world use case:** Payment method selection, `Comparator`-based sorting, route calculation algorithms.

**Java example:** `ShoppingCart` (context) delegating to `PaymentStrategy` implementations (`CreditCardPayment`, `UpiPayment`, `PayPalPayment`).

**Interview takeaway:** Be ready to distinguish Strategy from State precisely (client-driven choice vs. object-driven autonomous transition) — this comparison question comes up very frequently.

**One-line memory trick:** *"Google Maps: same app, same destination, but you pick fastest/shortest/no-tolls — the ROUTE algorithm is swappable, the APP isn't."*

---

*End of Topic 5. Type "Next" to proceed to Topic 6: Observer Design Pattern.*