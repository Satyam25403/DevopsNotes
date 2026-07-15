# Topic 7: Decorator Design Pattern

---

## 1. Introduction

**Definition:**
The Decorator Pattern is a **structural design pattern** that allows behavior to be added to an individual object dynamically, at runtime, by wrapping it in one or more "decorator" objects — without modifying the original object's class or affecting other instances of the same class.

**Why it exists / what problem it solves:**
Without Decorator, adding new combinations of behavior/features to a class typically leads to one of two problems:
- **Subclass explosion**: if you need "Coffee," "Coffee with Milk," "Coffee with Sugar," "Coffee with Milk and Sugar," "Coffee with Milk, Sugar, and Whipped Cream"... you'd need a NEW subclass for every single COMBINATION — this grows combinatorially and becomes unmanageable.
- **Modifying the base class directly**: adding a `hasMilk` boolean flag and an `if` check inside the base class for every possible add-on — violates OCP, and the class becomes bloated with every conceivable option.

Decorator solves this by wrapping an object with layers, each layer adding ONE specific piece of behavior, and layers can be combined in any order/combination WITHOUT needing a distinct class for every possible combination.

**When it should be used:**
- When you need to add responsibilities to individual objects dynamically and transparently, without affecting other objects of the same class
- When subclassing would lead to an impractical explosion of classes to cover every combination of optional features
- When you want to add/remove responsibilities at RUNTIME, not just at compile time via fixed subclassing

**When it should NOT be used:**
- When the set of possible combinations is small and stable — a few concrete subclasses might be simpler and more readable than a decorator chain
- When callers NEED to know the concrete type for some reason (Decorator relies on all layers sharing a common interface, so type-specific behavior becomes harder to access)

**Advantages:**
- Avoids subclass explosion — new behaviors are added as independent decorator classes, combinable in any order
- Follows OCP — new decorators can be added without touching existing code
- Behavior can be added/removed dynamically at runtime, unlike fixed inheritance

**Disadvantages:**
- Can result in many small, similar-looking decorator classes
- Debugging a deeply nested decorator chain can be harder — tracing exactly which layer produced a particular behavior/output takes more effort
- Order of decoration can matter and produce different results — easy to apply decorators in an unintended order

---

## 2. Real-World Analogy

Think of **ordering a pizza with toppings**.

You start with a **base pizza** (say, a plain Margherita). Then you can ADD toppings — extra cheese, mushrooms, olives, pepperoni — each topping WRAPS around (adds to) the base pizza, and each one adds to the price and description.

Critically:
- You don't need a separate "MargheritaWithCheeseAndMushroomsClass" pre-defined — you just ADD layers dynamically, in whatever combination you want.
- Each topping doesn't need to know about the OTHER toppings — "extra cheese" just adds cheese and its cost, regardless of whether mushrooms were added before or after it.
- The final pizza is still fundamentally "a pizza" (same interface/contract — you can still ask for its total price and description) no matter how many toppings have been layered on.

This is EXACTLY Decorator: a base object, wrapped by any number of decorator layers, each adding its own bit of behavior/cost, while the whole thing still honors the same overall contract.

---

## 3. Theory

**Core idea:** Both the original object and its decorators implement the SAME interface. A Decorator HOLDS a reference to the object it's wrapping (of that same interface type), and typically calls the wrapped object's method FIRST, then adds its own additional behavior before/after.

**Working mechanism:**
```
┌─────────────────────┐
│      <<interface>>          │
│         Coffee                   │
│  + cost(): double                  │
│  + description(): String             │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────────────┐
   │                          │
┌──┴────────┐    ┌──────┴──────────┐
│  SimpleCoffee     │    │  CoffeeDecorator      │  (abstract)
└──────────┘    │  - wrappedCoffee: Coffee │
                    └──────┬──────────┘
                           △
                   ┌───────┼───────┐
                   │               │
              ┌────┴────┐    ┌────┴────┐
              │MilkDecorator │    │SugarDecorator│
              └────────┘    └────────┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Component | The common interface both the base object and decorators implement |
| Concrete Component | The base/original object being decorated |
| Decorator | An abstract or concrete class that WRAPS a Component and adds behavior |
| Concrete Decorator | A specific decoration (e.g., MilkDecorator) |

**Key structural insight:** a Decorator IS-A Component (implements the same interface) AND HAS-A Component (holds a wrapped reference) — this dual relationship is what allows decorators to be STACKED indefinitely; a decorator wrapping a decorator wrapping a decorator, all still honoring the same interface.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Coffee                   │
├─────────────────────┤
│ + cost(): double                   │
│ + description(): String              │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────────────────┐
   │                              │
┌──┴────────┐            ┌──┴──────────────┐
│  SimpleCoffee     │            │  CoffeeDecorator (abstract) │
│  cost() = 2.0        │            ├────────────────┤
└──────────┘            │ # wrappedCoffee: Coffee ◆─────┐
                             ├────────────────┤          │ (composition:
                             │ + cost() { delegates          │  decorator OWNS
                             │   to wrappedCoffee.cost() }     │  the wrapped
                             └────────┬───────────┘          │  reference)
                                     △                          │
                             ┌───────┼───────┐                  │
                             │               │                  │
                      ┌──────┴────┐   ┌─────┴─────┐            │
                      │MilkDecorator│   │SugarDecorator│            │
                      │cost() { return   │cost() { return          │
                      │  wrapped.cost()     │  wrapped.cost()            │
                      │  + 0.5 }              │  + 0.25 }                    │
                      └──────────┘   └───────────┘◄───────────────┘
```

**Relationship explanation:**
- `SimpleCoffee` and `CoffeeDecorator` both **implement** `Coffee` — they honor the same contract.
- `CoffeeDecorator` has a **composition** relationship with `Coffee` (filled diamond) — the wrapped coffee is INTEGRAL to the decorator; a decorator without something to wrap is meaningless.
- `MilkDecorator`/`SugarDecorator` **extend** `CoffeeDecorator`, inheriting the wrapping mechanism, and each adds its OWN specific cost/description logic on top of whatever it wraps.
- Critically: `wrappedCoffee` could itself be ANOTHER decorator (e.g., `MilkDecorator` wrapping a `SugarDecorator` wrapping a `SimpleCoffee`) — this recursive structure is what enables stacking multiple decorations.

---

## 5. Java Implementation

```java
// ============================================
// Component interface — the shared contract
// ============================================
public interface Coffee {
    double cost();
    String description();
}

// ============================================
// Concrete Component — the base object being decorated
// ============================================
public class SimpleCoffee implements Coffee {
    @Override
    public double cost() {
        return 2.0;
    }

    @Override
    public String description() {
        return "Coffee";
    }
}

// ============================================
// Abstract Decorator — implements Coffee AND holds
// a wrapped Coffee reference (the dual relationship)
// ============================================
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrappedCoffee; // composition: this decorator WRAPS a Coffee

    protected CoffeeDecorator(Coffee coffee) {
        this.wrappedCoffee = coffee;
    }

    // Default behavior: just delegate to the wrapped object.
    // Concrete decorators will override these to ADD their own behavior.
    @Override
    public double cost() {
        return wrappedCoffee.cost();
    }

    @Override
    public String description() {
        return wrappedCoffee.description();
    }
}

// ============================================
// Concrete Decorators — each adds ONE specific behavior
// ============================================
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public double cost() {
        return wrappedCoffee.cost() + 0.5; // adds to whatever was wrapped
    }

    @Override
    public String description() {
        return wrappedCoffee.description() + " + Milk";
    }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public double cost() {
        return wrappedCoffee.cost() + 0.25;
    }

    @Override
    public String description() {
        return wrappedCoffee.description() + " + Sugar";
    }
}

public class WhippedCreamDecorator extends CoffeeDecorator {
    public WhippedCreamDecorator(Coffee coffee) {
        super(coffee);
    }

    @Override
    public double cost() {
        return wrappedCoffee.cost() + 0.75;
    }

    @Override
    public String description() {
        return wrappedCoffee.description() + " + Whipped Cream";
    }
}

// ============================================
// Demo
// ============================================
public class DecoratorDemo {
    public static void main(String[] args) {
        // Start with a plain coffee
        Coffee coffee = new SimpleCoffee();
        System.out.printf("%s: $%.2f%n", coffee.description(), coffee.cost());

        // Wrap it with milk
        coffee = new MilkDecorator(coffee);
        System.out.printf("%s: $%.2f%n", coffee.description(), coffee.cost());

        // Wrap AGAIN with sugar (now: milk-wrapped coffee, wrapped in sugar)
        coffee = new SugarDecorator(coffee);
        System.out.printf("%s: $%.2f%n", coffee.description(), coffee.cost());

        // Wrap a THIRD time with whipped cream
        coffee = new WhippedCreamDecorator(coffee);
        System.out.printf("%s: $%.2f%n", coffee.description(), coffee.cost());

        // A DIFFERENT combination/order, built fresh — demonstrating
        // decorators can be combined in any order without new classes
        Coffee customCoffee = new SugarDecorator(new WhippedCreamDecorator(new SimpleCoffee()));
        System.out.printf("%s: $%.2f%n", customCoffee.description(), customCoffee.cost());
    }
}
```

**Key line-by-line notes:**
- `CoffeeDecorator implements Coffee` AND holds `protected final Coffee wrappedCoffee` — this is the exact dual relationship (IS-A and HAS-A) that defines the pattern.
- `MilkDecorator.cost()` calls `wrappedCoffee.cost()` FIRST, then adds its own `+ 0.5` — each layer builds on whatever it wraps, without needing to know how many OTHER layers exist beneath it.
- `coffee = new MilkDecorator(coffee)` — REASSIGNING the same variable, now pointing to a decorator wrapping the PREVIOUS value — this is how layers stack incrementally in code.
- The final `customCoffee` line shows a DIFFERENT combination/order built directly, without needing any new class — pure runtime composition.

---

## 6. Dry Run

**Sample input:** `new WhippedCreamDecorator(new SugarDecorator(new MilkDecorator(new SimpleCoffee())))`, then calling `.cost()`.

```
1. new SimpleCoffee() → innermost object, call it S
2. new MilkDecorator(S) → wraps S, call it M (M.wrappedCoffee = S)
3. new SugarDecorator(M) → wraps M, call it Su (Su.wrappedCoffee = M)
4. new WhippedCreamDecorator(Su) → wraps Su, call it W (W.wrappedCoffee = Su)

Calling W.cost():
   → W.cost() executes: return wrappedCoffee.cost() + 0.75
   → Must first evaluate wrappedCoffee.cost(), which is Su.cost()
        → Su.cost() executes: return wrappedCoffee.cost() + 0.25
        → Must first evaluate wrappedCoffee.cost(), which is M.cost()
             → M.cost() executes: return wrappedCoffee.cost() + 0.5
             → Must first evaluate wrappedCoffee.cost(), which is S.cost()
                  → S.cost() executes: return 2.0 (base case, no wrapping)
             → M.cost() = 2.0 + 0.5 = 2.5
        → Su.cost() = 2.5 + 0.25 = 2.75
   → W.cost() = 2.75 + 0.75 = 3.5

Final result: 3.5
```

**What's happening in memory:** this is a RECURSIVE call chain — each decorator's `cost()` call is SUSPENDED, waiting on the result of the WRAPPED object's `cost()` call, all the way down to `SimpleCoffee` (the base case with no further wrapping), then the results BUBBLE BACK UP, each layer adding its own increment on the way. This is structurally identical to how recursive function calls unwind — each `CoffeeDecorator` object forms one "frame" in a chain that must fully resolve inward before resolving outward.

---

## 7. Real-World Software Example

- **Java I/O streams** — the textbook real-world example: `new BufferedReader(new InputStreamReader(new FileInputStream("file.txt")))`. Each layer ADDS behavior (buffering, character decoding) around the base stream, all sharing the common `Reader`/`InputStream` interface.
- **Spring's `HttpServletRequestWrapper`** — wraps an HTTP request to add/modify behavior (e.g., adding security headers, modifying parameters) without changing the original request object's class.
- **UI component libraries** — adding a scrollbar, border, or shadow to a UI widget by wrapping it, rather than needing a `ScrollableBorderedShadowedButton` subclass for every combination.
- **Middleware chains in web frameworks** — each middleware (logging, authentication, compression) wraps the next, adding its own behavior before/after delegating to the wrapped handler.

---

## 8. Internal Working

**Object creation:** each decorator layer is a genuinely SEPARATE object on the heap — wrapping 4 layers means 4 distinct objects exist in memory, each holding a reference to the one it wraps, forming a LINKED CHAIN (conceptually similar to a linked list, but via composition rather than a `next` pointer).

**Method dispatch:** each call like `wrappedCoffee.cost()` is a normal polymorphic (dynamic dispatch) call — the JVM doesn't know at compile time whether `wrappedCoffee` is a `SimpleCoffee` or another `CoffeeDecorator`; it resolves this at runtime via the vtable, exactly as covered in Topic 2.

**Call stack visualization:**
```
W.cost()
  └─ calls Su.cost()
       └─ calls M.cost()
            └─ calls S.cost()
                 └─ returns 2.0 (base case)
            └─ M.cost() returns 2.5
       └─ Su.cost() returns 2.75
  └─ W.cost() returns 3.5

Each decorator layer adds ONE stack frame — a very
deep decorator chain (dozens of layers) could theoretically
contribute to stack depth, though in practice this is
rarely a real concern for typical use cases (a handful
of layers, not hundreds).
```

---

## 9. Before vs After

**Before (subclass explosion anti-pattern):**

```java
public class Coffee { public double cost() { return 2.0; } }
public class CoffeeWithMilk extends Coffee { public double cost() { return 2.5; } }
public class CoffeeWithSugar extends Coffee { public double cost() { return 2.25; } }
public class CoffeeWithMilkAndSugar extends Coffee { public double cost() { return 2.75; } }
public class CoffeeWithMilkSugarAndCream extends Coffee { public double cost() { return 3.5; } }
// ... every NEW combination needs a BRAND NEW class,
// growing COMBINATORIALLY as more optional add-ons exist
```

**Problems:**
- With just 4 optional add-ons, you'd need up to 2^4 = 16 classes to cover every possible combination — utterly unmanageable as more options are added.
- Adding a FIFTH add-on (say, Vanilla) means potentially doubling the number of existing classes needed to cover all NEW combinations.
- Massive code duplication — `CoffeeWithMilkAndSugar` and `CoffeeWithMilkSugarAndCream` both repeat the milk+sugar cost logic independently.

**After (Decorator pattern, as shown in Section 5):**
- Exactly ONE class per add-on (`MilkDecorator`, `SugarDecorator`, `WhippedCreamDecorator`) — 4 classes total, regardless of how many COMBINATIONS are needed.
- Combinations are built at RUNTIME by wrapping in whatever order/combination is needed — no new classes required, ever, for a new combination.
- Adding a 5th add-on means adding exactly ONE new decorator class — zero impact on existing combinations.

---

## 10. SOLID Principles Connection

- **OCP**: new decorators (add-ons) can be introduced without modifying `SimpleCoffee`, `CoffeeDecorator`, or any existing decorator.
- **SRP**: each decorator has exactly one responsibility — adding ONE specific behavior/cost increment.
- **LSP**: any `Coffee` (whether `SimpleCoffee` or any decorated combination) can be substituted wherever a `Coffee` is expected — calling `.cost()` or `.description()` always works correctly, regardless of how many layers deep the object actually is.
- **DIP**: `CoffeeDecorator` depends on the `Coffee` ABSTRACTION for its wrapped reference, never on a specific concrete class — this is what allows it to wrap EITHER a `SimpleCoffee` OR another decorator interchangeably.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Decorator pattern solve?
2. Why does Decorator avoid the "subclass explosion" problem?
3. Give a real-world Java example of the Decorator pattern.

**Intermediate:**
4. Explain the dual "IS-A and HAS-A" relationship that defines a Decorator.
5. Does the ORDER in which decorators are applied matter? Give an example where it might.
   *Answer: Yes, potentially — e.g., a "CompressionDecorator" applied before vs after an "EncryptionDecorator" produces genuinely different results (compress-then-encrypt vs encrypt-then-compress typically yields different output sizes and behavior), so order can be semantically significant depending on what the decorators actually do.*
6. How would you remove a SPECIFIC decorator from the middle of an existing chain (e.g., remove just the "Sugar" layer from a coffee that has Milk, Sugar, and Cream applied)?
   *Answer: This is a known LIMITATION of Decorator — since each layer only knows about the one it directly wraps, removing a layer from the MIDDLE of an existing chain isn't straightforward; you'd typically need to rebuild the chain from scratch without that specific layer.*

**Advanced:**
7. **Decorator vs Proxy pattern** — how are they structurally similar, and what's the key conceptual difference?
   *Answer: Structurally nearly IDENTICAL — both wrap an object implementing the same interface and hold a reference to it. The difference is INTENT: Decorator's purpose is to ADD NEW BEHAVIOR/responsibilities to an object (like adding milk to coffee); Proxy's purpose (Topic 11) is to CONTROL ACCESS to an object (lazy loading, access control, remote proxying) WITHOUT adding new user-facing behavior — the proxy typically presents the EXACT SAME behavior, just with added access-control logic invisible to the client's expectations.*
8. How does Java's I/O stream design (`BufferedReader(InputStreamReader(FileInputStream(...)))`) exemplify Decorator? What would the alternative (without Decorator) look like?
9. Can Decorator be combined with Factory pattern (Topic 8) to simplify constructing complex decorator chains for client code? How?
10. What's a potential performance consideration with deeply nested decorator chains?
11. How does Decorator relate to the Composite pattern (Topic 14) — could a Decorator be thought of as a Composite with exactly one child?

---

## 12. Common Mistakes

- **Forgetting to delegate to the wrapped object** — a common bug is a decorator overriding a method but forgetting to call `wrappedCoffee.someMethod()` internally, silently LOSING the wrapped object's behavior entirely instead of building on top of it.
- **Assuming decorators can be freely reordered without consequence** — as discussed in Q5 above, order can matter; assuming it never does is a design risk.
- **Using Decorator when simple composition/configuration would do** — if the "combinations" are actually few and stable, a simpler design (e.g., a coffee object with a `Set<AddOn> addOns` field) might be more straightforward than a full decorator chain.
- **Confusing Decorator with Inheritance-based extension** — Decorator is specifically for RUNTIME, DYNAMIC combination; if you only ever need ONE fixed combination decided at compile time, a regular subclass might genuinely be simpler.

---

## 13. Time Complexity

**Method calls through a decorator chain of depth n**: O(n) — each layer adds one additional method call/stack frame before reaching the base object. For a typical use case (a handful of decorators), this is negligible; for pathologically deep chains, it's worth being aware the cost scales linearly with chain depth.

---

## 14. Java API Examples

- **`java.io` streams**: `BufferedInputStream`, `DataInputStream`, `InputStreamReader` — all wrap a base `InputStream`/`Reader`, adding buffering, primitive-type reading, or character decoding respectively. This is THE canonical Decorator example referenced in nearly every Java textbook.
- **`java.util.Collections.synchronizedList()` / `unmodifiableList()`**: these wrap a `List` and return a NEW `List` that adds synchronization or read-only enforcement, without modifying the original list's class — a form of the Decorator pattern applied to collections.
- **Servlet Filters / Spring `HandlerInterceptor`**: each filter wraps the next in a chain, adding behavior (logging, authentication) before/after delegating to the next filter/handler.

---

## 15. Practice Problem

Implement a **pizza ordering system** using Decorator: a `Pizza` interface with `cost()` and `description()`, a `PlainPizza` base implementation, and at least three topping decorators (`CheeseTopping`, `MushroomTopping`, `OliveTopping`). Demonstrate building at least two DIFFERENT pizzas with different combinations/orders of toppings, printing each one's final cost and description.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a text processing system where a base `TextSource` (plain text) can be wrapped with optional processing steps: `UpperCaseDecorator`, `TrimWhitespaceDecorator`, and `RemoveSpecialCharactersDecorator`. Clients should be able to apply any combination of these steps, in any order, to a piece of text, and the system should support adding NEW processing steps in the future without modifying existing decorators."

Think about:
- Would the ORDER of applying `UpperCaseDecorator` vs `TrimWhitespaceDecorator` ever produce different results? Why or why not?
- How would you design this so a client could easily build "apply trim, then remove special characters, then uppercase" without deeply nested constructor calls being hard to read?

---

## 17. Advanced LLD Scenario

**Design a Notification Message Enhancement System** for a messaging platform, where:
- A base `Message` can be enhanced with optional features: `EncryptionDecorator` (encrypts the message body), `CompressionDecorator` (compresses the payload), `TimestampDecorator` (adds a timestamp), and `SignatureDecorator` (adds a digital signature for verification)
- Different message types (urgent alerts, regular chat messages, system logs) may need DIFFERENT combinations of these enhancements
- The ORDER of encryption vs compression matters significantly for correctness (compressing already-encrypted data is typically ineffective, since encrypted data has high entropy and doesn't compress well) — the design should make it easy to enforce a SENSIBLE default ordering while still allowing flexibility

Consider:
- How you'd structure the decorators so that `send(message)` at the end always operates on the FULLY decorated message, regardless of which specific combination was applied
- Whether you'd want to provide a "builder"-style helper (foreshadowing the Builder pattern, Topic 17) to make constructing common combinations less error-prone and more readable than raw nested constructor calls
- How this system's design compares structurally to the Java I/O streams example from Section 14 — recognizing the SAME underlying pattern in a different domain

---

## 18. Summary

**Definition:** Decorator dynamically adds new behavior/responsibilities to an individual object by wrapping it in one or more decorator objects sharing the same interface.

**Intent:** Avoid subclass explosion when many optional feature combinations are needed; allow behavior to be added/removed at runtime.

**Key classes:** `Component` interface, `ConcreteComponent` (base object), `Decorator` (abstract, holds wrapped Component reference), `ConcreteDecorator` implementations.

**Advantages:** Avoids combinatorial subclassing; OCP-compliant; runtime flexibility in combining behaviors.

**Disadvantages:** Many small similar classes; harder debugging through nested layers; order-of-application can matter.

**Real-world use case:** Java I/O streams (`BufferedReader(InputStreamReader(FileInputStream(...)))`); web middleware/filter chains.

**Java example:** `Coffee` interface with `SimpleCoffee` base and `MilkDecorator`/`SugarDecorator`/`WhippedCreamDecorator` layered dynamically.

**Interview takeaway:** Be ready to precisely distinguish Decorator from Proxy (adding behavior vs controlling access) — this is one of the most frequently asked pattern-comparison questions in LLD interviews.

**One-line memory trick:** *"Pizza toppings: same base pizza, any combination of toppings layered on — no need for a pre-made class for every possible topping combo."*

---

*End of Topic 7. Type "Next" to proceed to Topic 8: Factory Design Pattern.*