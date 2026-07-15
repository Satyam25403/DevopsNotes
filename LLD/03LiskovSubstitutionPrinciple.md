# Topic 4: Liskov Substitution Principle (LSP)

---

## 1. Introduction

**Definition:**
The Liskov Substitution Principle states: *"If `S` is a subtype of `T`, then objects of type `T` in a program may be replaced with objects of type `S` without altering any of the desirable properties of that program (correctness, task performed, etc.)."*

In simpler terms: **a subclass should be usable anywhere its parent class is expected, without the calling code needing to know or care about the difference, and without anything breaking.**

**Why LSP exists / what problem it solves:**
Inheritance (Topic 2) lets you create a subclass — but Java's compiler only checks that the subclass's method SIGNATURES match; it cannot check whether the subclass's BEHAVIOR still honors what callers reasonably expect from the parent class. LSP is the discipline that catches this gap. Without it, you can have code that:
- Compiles perfectly fine
- Passes a naive "is-a" relationship check
- But BREAKS at runtime or produces wrong results when a subclass is substituted in

This is often the MOST SUBTLE of the SOLID principles because violations don't show up as compiler errors — they show up as unexpected behavior, sometimes only in edge cases.

**When it should be used:**
- Any time you're designing an inheritance hierarchy (class extends class, or class implements interface) — LSP should be a constant check: "can I truly substitute this subtype anywhere the supertype is expected?"

**When it should NOT be a blocker:**
- LSP doesn't mean subclasses can't ADD new behavior — it means they can't BREAK the promises/contracts the parent already made. Adding genuinely new capability is fine; changing or weakening existing guaranteed behavior is the violation.

**Advantages:**
- Code using polymorphism (Topic 2) can be trusted to work correctly with ANY subtype, without special-casing
- Prevents a large class of subtle, hard-to-debug runtime bugs
- Enables safe, confident use of the Open/Closed Principle (Topic 3) — because new subtypes genuinely behave correctly as substitutes

**Disadvantages:**
- Not enforced by the compiler — requires careful design review and discipline
- Can require rethinking a "natural-seeming" inheritance hierarchy (like Rectangle/Square) that turns out to be conceptually flawed

---

## 2. Real-World Analogy

Think of an **electrical wall socket standard**.

If a socket is rated for "any standard plug," then EVERY plug claiming to be "standard" must:
- Fit physically (same shape/size)
- Carry the expected voltage safely
- Not do anything surprising (like randomly delivering double voltage)

Imagine a "SmartPlug" that LOOKS like a standard plug (fits the socket, same shape) but secretly delivers double the voltage when a certain smart-home condition is met. Any device you plug in, expecting "standard behavior," could be damaged — even though the SmartPlug technically "fits" and satisfies the physical "is-a plug" relationship.

This is EXACTLY what an LSP violation is: something that satisfies the surface-level "is-a" relationship (fits the socket) but violates the deeper BEHAVIORAL CONTRACT (delivers standard voltage) that callers rely on.

---

## 3. Theory

### The Classic Violation: Rectangle/Square

**Core idea walkthrough:**
```
Mathematically, "a square IS a rectangle" (a rectangle with
equal sides) — so it seems natural to model:

class Rectangle {
    void setWidth(double w);
    void setHeight(double h);
    double getArea();
}

class Square extends Rectangle {
    // To maintain "squareness," Square overrides setWidth
    // to ALSO update height (and vice versa):
    @Override
    void setWidth(double w) {
        this.width = w;
        this.height = w;  // forces height to match!
    }
    @Override
    void setHeight(double h) {
        this.width = h;
        this.height = h;
    }
}
```

**Why this violates LSP:**
```
Code written against Rectangle (the parent) reasonably assumes:
"Calling setWidth() changes ONLY the width; height stays the same."

void resizeRectangle(Rectangle r) {
    r.setWidth(5);
    r.setHeight(10);
    assert r.getArea() == 50;  // reasonable expectation for
                                 // ANY Rectangle... 
}

resizeRectangle(new Rectangle()); // assert PASSES (area = 50)
resizeRectangle(new Square());     // assert FAILS!
                                     // (setHeight(10) also changed
                                     //  width to 10, so area = 100,
                                     //  NOT 50 as expected)
```

**The core lesson:** `Square` satisfies the "is-a" relationship structurally (it compiles, it IS a `Rectangle` type), but it does NOT satisfy the BEHAVIORAL contract that code using `Rectangle` reasonably depends on. This is precisely what LSP is designed to catch.

### Contracts: Preconditions and Postconditions

**Important terminology:**
```
Precondition:  what must be TRUE before a method is called
               (e.g., "the amount parameter must be positive")

Postcondition: what must be TRUE after a method completes
               (e.g., "the balance reflects the deposited amount")

Invariant:     a condition that must ALWAYS hold true for
               an object, regardless of what methods are called
```

**LSP's formal rule regarding contracts:**
```
A subclass may:
- WEAKEN preconditions (accept MORE inputs than the parent
  promised to handle) — this is SAFE, since any caller
  satisfying the parent's stricter precondition still works
- STRENGTHEN postconditions (guarantee MORE than the parent
  promised) — this is SAFE too, since callers expecting the
  parent's weaker guarantee still get what they expected

A subclass must NEVER:
- STRENGTHEN preconditions (demand MORE than the parent
  required) — this BREAKS callers who relied on the parent's
  looser requirement
- WEAKEN postconditions (guarantee LESS than the parent
  promised) — this BREAKS callers relying on the parent's
  stronger guarantee
```

**Visual — applying this to Rectangle/Square:**
```
Rectangle.setWidth()'s implicit postcondition:
"After calling setWidth(w), width == w AND height is UNCHANGED"

Square.setWidth()'s actual behavior:
"After calling setWidth(w), width == w AND height == w too"
→ This WEAKENS the postcondition (height is no longer guaranteed
  unchanged) → LSP VIOLATION
```

### The Correct Fix

**Visual:**
```
The REAL problem: Square shouldn't inherit from a MUTABLE
Rectangle at all, because "being a square" and "being
independently resizable in width/height" are fundamentally
incompatible behavioral contracts.

Better design — separate the concept of "shape with an area"
from "independently resizable in two dimensions":

┌─────────────────────┐
│      <<interface>>          │
│         Shape                    │
│  + getArea(): double              │
└──────────┬──────────┘
           △
   ┌───────┼───────┐
   │               │
┌──┴────────┐  ┌──┴────────┐
│  Rectangle           │  │  Square              │
│  - width, height          │  │  - side                │
│  + setWidth(w)                │  │  + setSide(s)              │
│  + setHeight(h)                  │  └──────────┘
└──────────┘

Now NEITHER class inherits from the other — both
independently implement Shape, each with method
signatures that make sense for THEIR OWN behavior.
No broken assumptions, no LSP violation.
```

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Shape                    │
├─────────────────────┤
│ + getArea(): double                │
└──────────┬──────────┘
           △ (implements — NOT extends)
   ┌───────┼───────────────┐
   │                          │
┌──┴────────┐          ┌──┴────────┐
│  Rectangle           │          │  Square              │
├──────────┤          ├──────────┤
│ - width: double         │          │ - side: double            │
│ - height: double           │          ├──────────┤
├──────────┤          │ + setSide(s)              │
│ + setWidth(w)              │          │ + getArea()                  │
│ + setHeight(h)                │          └──────────┘
│ + getArea()                      │
└──────────┘
```

**Relationship explanation:** Both `Rectangle` and `Square` **implement** `Shape` (hollow triangle, dashed line) as SIBLINGS, not as parent/child. This is the corrected design — `Square` is no longer forced to inherit (and awkwardly override) `Rectangle`'s width/height-specific methods, which don't conceptually apply to a square's actual constraint (equal sides). Each class exposes ONLY the methods that make sense for its own behavior.

---

## 5. Java Implementation

```java
// ============================================
// DEMONSTRATING THE VIOLATION FIRST (for learning purposes)
// ============================================
class Rectangle {
    protected double width;
    protected double height;

    public void setWidth(double width) {
        this.width = width;
    }

    public void setHeight(double height) {
        this.height = height;
    }

    public double getArea() {
        return width * height;
    }
}

// LSP VIOLATION: Square secretly changes BOTH dimensions
// when only one is set, breaking Rectangle's implicit contract
class Square extends Rectangle {
    @Override
    public void setWidth(double width) {
        this.width = width;
        this.height = width; // side effect callers don't expect!
    }

    @Override
    public void setHeight(double height) {
        this.width = height;
        this.height = height; // side effect callers don't expect!
    }
}

class LspViolationDemo {
    // This method reasonably assumes ANY Rectangle behaves
    // consistently: setWidth only affects width.
    static void demonstrateViolation(Rectangle rectangle) {
        rectangle.setWidth(5);
        rectangle.setHeight(10);
        System.out.println("Expected area: 50, Actual area: " + rectangle.getArea());
        // Prints "50" for a real Rectangle, but "100" for a Square!
    }

    public static void main(String[] args) {
        demonstrateViolation(new Rectangle()); // Actual area: 50.0 (correct)
        demonstrateViolation(new Square());     // Actual area: 100.0 (BROKEN!)
    }
}


// ============================================
// THE CORRECTED DESIGN — no inheritance between
// Rectangle and Square; both implement a common Shape interface
// ============================================
interface Shape {
    double getArea();
}

class RectangleFixed implements Shape {
    private double width;
    private double height;

    public RectangleFixed(double width, double height) {
        this.width = width;
        this.height = height;
    }

    public void setWidth(double width) { this.width = width; }
    public void setHeight(double height) { this.height = height; }

    @Override
    public double getArea() {
        return width * height;
    }
}

class SquareFixed implements Shape {
    private double side;

    public SquareFixed(double side) {
        this.side = side;
    }

    // Square exposes ITS OWN meaningful method — setSide —
    // rather than inheriting confusing setWidth/setHeight semantics
    public void setSide(double side) {
        this.side = side;
    }

    @Override
    public double getArea() {
        return side * side;
    }
}

class LspFixedDemo {
    // This method now works correctly and predictably with
    // ANY Shape — because Shape's contract (just getArea())
    // is genuinely honored by every implementation
    static void printArea(Shape shape) {
        System.out.println("Area: " + shape.getArea());
    }

    public static void main(String[] args) {
        printArea(new RectangleFixed(5, 10)); // Area: 50.0
        printArea(new SquareFixed(5));           // Area: 25.0
        // No broken assumptions, no surprising side effects —
        // each shape is used through the SAME simple contract
    }
}
```

**Key line-by-line notes:**
- `Square extends Rectangle` with overridden `setWidth`/`setHeight` — this COMPILES fine and even superficially "makes sense" (a square IS mathematically a special rectangle), but the BEHAVIOR breaks callers' reasonable expectations.
- `demonstrateViolation()` — deliberately written the way any reasonable developer would write code using a `Rectangle`, without any awareness that `Square` might sneakily behave differently — this is exactly how LSP violations bite you in real, unsuspecting code.
- The fixed version has `Rectangle` and `Square` as SIBLINGS implementing `Shape` — neither inherits behavior it can't honestly fulfill.

---

## 6. Dry Run

**Sample input:** `demonstrateViolation(new Square())`

```
1. new Square() created
   → A Square object allocated on heap; width=0, height=0
     (inherited fields from Rectangle)

2. rectangle.setWidth(5) called
   → Reference type: Rectangle, actual type: Square
   → Dynamic dispatch (Topic 2) resolves to SQUARE's
     overridden setWidth(), NOT Rectangle's
   → Inside Square.setWidth(5):
       this.width = 5
       this.height = 5   ← the surprising side effect!

3. rectangle.setHeight(10) called
   → Dynamic dispatch resolves to Square's overridden setHeight()
   → Inside Square.setHeight(10):
       this.width = 10   ← width SILENTLY changed again!
       this.height = 10

4. rectangle.getArea() called
   → Rectangle's getArea() (not overridden in Square) executes:
       return width * height = 10 * 10 = 100

5. Output: "Expected area: 50, Actual area: 100.0"
   → The caller's REASONABLE expectation (based on how
     Rectangle "should" behave) is violated — not because
     of a bug in getArea() itself, but because setWidth/
     setHeight's OVERRIDDEN behavior broke an implicit contract
```

**What's happening conceptually:** the bug isn't in any single line of code — every individual line does exactly what it's written to do. The bug is a **contract violation**: `Square`'s override changes the MEANING of `setWidth`/`setHeight` in a way that code written against `Rectangle` cannot anticipate. This is why LSP violations are notoriously hard to catch through code review alone — you have to think about BEHAVIORAL CONTRACTS, not just "does this compile and does each method look reasonable in isolation."

---

## 7. Real-World Software Example

- **Java's own historical LSP debate**: `java.util.Stack extends Vector` — `Stack` is supposed to be LIFO (last-in-first-out), but because it extends `Vector`, it inherits methods like `get(index)` and `insertElementAt()` that let you violate LIFO ordering entirely. This is a well-known, real LSP violation in the JDK itself, often cited in interviews.
- **Ostrich/Bird problem**: a classic teaching example — `class Ostrich extends Bird` where `Bird` has a `fly()` method. Ostriches can't fly, so `Ostrich.fly()` either throws an exception or does nothing — either way, code expecting "any Bird can fly()" breaks when given an Ostrich.
- **Read-only collections**: if a `Collection` interface promises `add()` works, but an `ImmutableList` implementation throws `UnsupportedOperationException` on `add()`, this is an LSP violation — code expecting to safely add to "any Collection" breaks unexpectedly.

---

## 8. Internal Working

LSP violations are NOT caught by the JVM or the Java compiler — the compiler only checks:
- Method signatures match (return type, parameter types compatible)
- Access modifiers aren't more restrictive in the override

The compiler has NO WAY to verify BEHAVIORAL correctness (whether `setWidth` still "means" the same thing) — this is purely a DESIGN-TIME, human-judgment concern. This is precisely why LSP requires careful thought during design, rather than being something you can rely on tooling to catch (though some static analysis tools and thorough unit testing of substitutability can help catch violations empirically).

**Dynamic dispatch (from Topic 2) is what MAKES the violation possible to manifest**: because `rectangle.setWidth()` resolves to `Square`'s overridden version at runtime based on the actual object type, the "surprising" behavior only shows up when a `Square` is ACTUALLY substituted in — the exact scenario LSP is concerned with.

---

## 9. Before vs After

**Before (LSP violation, as shown in Section 5):** `Square extends Rectangle`, overriding `setWidth`/`setHeight` to maintain equal sides — compiles fine, breaks callers' reasonable expectations.

**After (LSP-compliant, as shown in Section 5):** `Rectangle` and `Square` are siblings implementing a shared `Shape` interface, each exposing only the methods that genuinely make sense for their own behavior (`setWidth`/`setHeight` for Rectangle; `setSide` for Square) — no inherited method needs to be overridden in a way that breaks its original meaning.

**Why this matters beyond just this example:** the GENERAL lesson is: **if making a subclass behaviorally correct requires overriding a method to do something meaningfully DIFFERENT from what callers of the parent class would expect, that's a signal the inheritance relationship itself may be wrong** — even if it seems intuitive ("a square IS a rectangle" mathematically).

---

## 10. SOLID Principles Connection

- **LSP directly enables OCP**: Open/Closed Principle relies on being able to add new subtypes without breaking existing code — but this ONLY works safely if those new subtypes genuinely honor LSP. An OCP-compliant design built on LSP-violating subtypes is a false sense of safety.
- **LSP relates to ISP**: if a subclass has to override a method to throw `UnsupportedOperationException` (like an immutable list's `add()`), it's often a sign the interface was too broad to begin with — ISP (splitting into smaller, more specific interfaces) can sometimes PREVENT the LSP violation from being possible in the first place.
- **LSP and SRP**: the Rectangle/Square problem partly stems from `Rectangle` conflating "a shape with an area" with "an independently width/height-resizable shape" — arguably two responsibilities that should have been separated from the start (SRP).

---

## 11. Interview Questions

**Beginner:**
1. State the Liskov Substitution Principle in your own words.
2. Why does the classic Square-extends-Rectangle design violate LSP, even though mathematically a square IS a rectangle?

**Intermediate:**
3. What are preconditions and postconditions, and how do they relate to LSP?
4. Give an example (other than Rectangle/Square) of a class hierarchy that violates LSP.
5. How would you detect an LSP violation during code review, given that the compiler won't catch it?
   *Answer: Look for overridden methods that throw exceptions the parent didn't, silently change more state than the parent method implied, or return values inconsistent with what callers of the parent type would reasonably expect based on the parent's documented/implied behavior.*

**Advanced:**
6. Explain why `java.util.Stack extends Vector` is considered an LSP violation in the JDK.
7. Can LSP violations occur with INTERFACES, not just class inheritance? Give an example.
   *Answer: Yes — e.g., an interface `Collection` with an `add()` method; if an `ImmutableList implements Collection` throws `UnsupportedOperationException` on `add()`, this violates LSP even though it's interface implementation, not class extension.*
8. How does LSP relate to "design by contract" as a broader software engineering concept?
9. Describe a real design decision where you'd choose composition over inheritance specifically to avoid an LSP violation.
10. Is it possible to satisfy LSP while still using inheritance for the Rectangle/Square-style problem? What would that require?
    *Answer: You'd need to make Rectangle IMMUTABLE (no setWidth/setHeight after construction) — then Square could safely extend it, since there's no "resize just one dimension" operation to violate. This shows LSP violations are often about MUTABLE state specifically.*

---

## 12. Common Mistakes

- **Assuming "is-a" in natural language automatically means "safe to inherit in code"** — mathematical/conceptual "is-a" relationships (square is-a rectangle) don't always translate to safe OOP inheritance, especially with MUTABLE state.
- **Overriding a method to throw an exception for "doesn't apply" cases** — a strong signal of an LSP violation (and often an ISP violation too) rather than a clever way to handle an edge case.
- **Not considering LSP when designing interfaces, only when using `extends`** — LSP applies equally to interface implementations, not just class inheritance.
- **Believing unit tests alone will catch LSP violations** — they'll only catch it if someone specifically writes a test substituting the subtype where the supertype is expected; it's easy to only test each class in isolation and miss substitutability entirely.

---

## 13. Time Complexity

Not applicable — LSP is a design/behavioral correctness principle, not an algorithmic one.

---

## 14. Java API Examples

- **`java.util.Stack extends Vector`** — the textbook JDK example of an LSP violation (discussed in Section 7); modern code is often advised to use `Deque` (via `ArrayDeque`) instead of `Stack` for this reason.
- **`java.util.Collections.unmodifiableList()`** — returns a `List` that throws `UnsupportedOperationException` on mutating calls like `add()`; this is a KNOWN, DOCUMENTED deviation from `List`'s general contract — Java's documentation explicitly warns about this, which is the responsible way to handle an unavoidable LSP tension (document it clearly, rather than silently surprising callers).
- **Comparable/Comparator contracts** — `compareTo()` implementations that don't honor mathematical consistency (e.g., if `a.compareTo(b) > 0` but `b.compareTo(a)` doesn't return `< 0`) violate an implicit LSP-style contract that sorting algorithms rely on, leading to unpredictable sort results.

---

## 15. Practice Problem

Consider this class hierarchy:
```java
class Bird {
    void fly() { System.out.println("Flying..."); }
}
class Sparrow extends Bird {}
class Ostrich extends Bird {
    @Override
    void fly() { throw new UnsupportedOperationException("Ostriches can't fly"); }
}
```
- Explain precisely why this violates LSP.
- Redesign this hierarchy so that code iterating over a list of `Bird` objects and calling `fly()` on each never risks a runtime exception, while still modeling the fact that ostriches can't fly.
- Consider: should "flying" even be a method on a general `Bird` base class at all?

---

## 16. Medium-Level Exercise

**Interview-style problem:** "You're given a `ReadOnlyList<T>` interface with a single method `get(index)`, and a `MutableList<T> extends ReadOnlyList<T>` adding `add(item)` and `remove(index)`. A method `processReadOnlyList(ReadOnlyList<T> list)` is written expecting to safely call `get()` on ANY `ReadOnlyList`. Is this design LSP-safe? What if someone later creates a class implementing `ReadOnlyList` where `get()` sometimes throws an exception depending on internal state?"

Think about:
- Where does the LSP contract actually live here — is it enough that method signatures match?
- How would you DOCUMENT or ENFORCE the behavioral contract of `get()` to prevent future violations?

---

## 17. Advanced LLD Scenario

**Design a Payment Processing Hierarchy** for a system supporting Credit Card, Debit Card, and Gift Card payments, where:
- All payment methods need a `processPayment(amount)` method
- Gift cards have a MAXIMUM balance and can partially fail (insufficient gift card balance) in ways credit/debit cards don't
- The system should be able to iterate over a `List<PaymentMethod>` and process payments uniformly, without special-casing any specific payment type

Consider:
- If `GiftCardPayment.processPayment()` needs to return a "partial success, remaining balance owed" result while `CreditCardPayment.processPayment()` only ever fully succeeds or fully fails, does this create an LSP tension?
- How would you design the RETURN TYPE of `processPayment()` (e.g., a `PaymentResult` object capturing success/partial/failure) so that EVERY implementation can honestly fulfill the SAME contract, avoiding the temptation to throw unexpected exceptions or silently behave differently?

*(This scenario shows LSP concerns arising naturally even OUTSIDE the classic Rectangle/Square example — anytime subtypes have genuinely different edge-case behaviors, the CONTRACT/return type needs careful design to keep every implementation truly substitutable.)*

---

## 18. Summary

**Definition:** LSP states that objects of a subtype must be substitutable for objects of their supertype without breaking the correctness of the program.

**Intent:** Ensure inheritance hierarchies are behaviorally safe, not just structurally/syntactically valid.

**Key classes:** The classic `Rectangle`/`Square` example; also `Bird`/`Ostrich`, and JDK's `Stack extends Vector`.

**Advantages:** Prevents subtle runtime bugs from polymorphic substitution; enables safe use of OCP.

**Disadvantages:** Not compiler-enforced — requires careful design judgment; can require rethinking "intuitive" inheritance relationships.

**Real-world use case:** JDK's `Stack extends Vector` is a widely-cited real LSP violation; `Collections.unmodifiableList()` is a documented, intentional exception to a collection's normal mutability contract.

**Java example:** `Square extends Rectangle` overriding `setWidth`/`setHeight` breaks callers' reasonable expectations — fixed by making `Rectangle` and `Square` siblings implementing a shared `Shape` interface.

**Interview takeaway:** Always ask "if I substitute a subtype here, could ANY reasonable caller be surprised by different behavior?" — and remember that mutable state is very often where LSP violations hide (an immutable design frequently sidesteps the problem entirely).

**One-line memory trick:** *"If it looks like a duck, quacks like a duck, but needs batteries and can't actually walk on land — it's not a safe substitute for 'duck.'"*

---

*End of Topic 4. Type "Next" to proceed to Topic 5: Strategy Design Pattern.*