# Topic 17: Builder Pattern

---

## 1. Introduction

**Definition:**
The Builder Pattern is a **creational design pattern** that separates the CONSTRUCTION of a complex object from its REPRESENTATION, allowing the SAME step-by-step construction process to create DIFFERENT representations — and, more commonly in practice, providing a clean, readable way to construct objects that have MANY optional parameters or configuration steps, avoiding constructors with excessive parameter lists.

**Why it exists / what problem it solves:**
Consider a `House` object that could have MANY optional attributes: number of rooms, whether it has a garage, a swimming pool, a garden, solar panels, etc. A single constructor trying to accommodate ALL possible combinations of these options either needs an ENORMOUS number of parameters (most of which are irrelevant for any given house), or forces you to create numerous OVERLOADED constructors ("telescoping constructors") — both approaches quickly become unreadable and error-prone (e.g., accidentally swapping two `boolean` parameters of the same type in a long constructor call).

Builder solves this by introducing a SEPARATE `Builder` object with clearly-named methods for setting EACH optional attribute (`.withGarage()`, `.withPool()`, etc.), CHAINED together fluently, and a final `.build()` call that assembles the fully-configured object — dramatically improving readability and reducing the risk of parameter-order mistakes.

**When it should be used:**
- When an object has MANY parameters, especially OPTIONAL ones, and a single constructor would become unwieldy ("telescoping constructor" problem)
- When you want the CONSTRUCTED object to be IMMUTABLE once built, but still support a flexible, step-by-step configuration process beforehand
- When you want READABLE, self-documenting object construction code (`.withRooms(3).withGarage(true).build()` is far clearer than `new House(3, true, false, true, null, ...)`)

**When it should NOT be used:**
- When the object has only a FEW simple parameters — a normal constructor is simpler and doesn't need the extra Builder machinery
- When ALL parameters are genuinely REQUIRED (not optional) — Builder shines specifically for optional/many-combination scenarios; if everything's mandatory, a constructor (or Factory) may be more appropriate
- When object construction is a SINGLE, simple, atomic step with no meaningful "in-progress" configuration phase

**Advantages:**
- Eliminates telescoping constructors and improves readability significantly
- Allows the CONSTRUCTED object to be fully IMMUTABLE (all fields set once, at `build()` time), while still supporting flexible, incremental configuration
- Makes it easy to validate the FINAL configuration (in `build()`) before creating the object, catching invalid combinations early
- Method chaining (fluent API) produces SELF-DOCUMENTING construction code

**Disadvantages:**
- Introduces an ADDITIONAL Builder class per object type needing this treatment — more code to write and maintain
- Can feel like OVERKILL for simple objects with few parameters
- The Builder itself is typically MUTABLE during construction, which is a necessary but slightly less "pure" aspect compared to the fully immutable final object it produces

---

## 2. Real-World Analogy

Think of **ordering a custom sandwich at a sandwich shop (like Subway)**.

You don't hand the cashier a single, giant, rigid instruction card with EVERY possible topping pre-filled in a FIXED order. Instead, you build your sandwich STEP BY STEP: "White bread... turkey... add lettuce... add tomato... skip the onions... add mayo... toasted." Each step is a clear, named choice, and you can skip steps you don't want (no telescoping "give me bread, turkey, null, tomato, null, mayo, true" confusion). At the END, you say "that's it, make it" — and the sandwich (the FINAL object) is assembled EXACTLY according to your step-by-step specifications, and once made, it's a FIXED, completed thing (immutable).

---

## 3. Theory

**Core idea:** Instead of a single, monolithic constructor, introduce a SEPARATE `Builder` class (often as a `static` nested class within the target object) with individual, clearly-named methods for setting EACH optional/required attribute. Each setter method RETURNS the Builder itself (`this`), enabling FLUENT method chaining. A final `.build()` method constructs and returns the FULLY-configured target object, typically making it IMMUTABLE afterward.

**Working mechanism:**
```
new House.Builder()
    .rooms(3)
    .hasGarage(true)
    .hasPool(false)
    .build()   ──→   returns a fully-constructed, IMMUTABLE House object
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Product | The complex object being constructed (e.g., `House`) |
| Builder | The separate class providing step-by-step configuration methods |
| Fluent interface | A method-chaining style where each setter returns `this`, enabling `.method1().method2().method3()` chains |
| build() | The final method that assembles and returns the fully-configured Product |

**Class responsibilities:** the Builder is responsible for ACCUMULATING configuration choices (typically as its OWN mutable fields) and, at `build()` time, VALIDATING the accumulated configuration and constructing the FINAL, immutable Product object from it.

---

## 4. UML / Class Diagram

```
┌─────────────────────────────┐
│      House (Product)                    │
├─────────────────────────────┤
│ - rooms: int (final)                  │
│ - hasGarage: boolean (final)          │
│ - hasPool: boolean (final)            │
├─────────────────────────────┤
│ - House(Builder b)  (private constructor,│
│   takes a fully-configured Builder)      │
├─────────────────────────────┤
│      <<static nested class>>            │
│      Builder                            │
├─────────────────────────────┤
│ - rooms: int                    │
│ - hasGarage: boolean              │
│ - hasPool: boolean                │
├─────────────────────────────┤
│ + rooms(int): Builder            │  (each returns 'this' —
│ + hasGarage(boolean): Builder    │   enabling fluent chaining)
│ + hasPool(boolean): Builder      │
│ + build(): House                 │  (constructs the FINAL,
└─────────────────────────────┘   immutable House)
```

**Relationship explanation:**
- `Builder` is a **static nested class** WITHIN `House` — this is a COMMON Java idiom (not strictly required by the pattern, but very typical), keeping the Builder tightly coupled to the ONE Product type it constructs.
- `House`'s constructor is typically `private`, accepting a fully-configured `Builder` instance and COPYING its accumulated field values into `House`'s OWN `final` fields — this is what makes the FINAL `House` object immutable, even though the `Builder` itself was mutable during configuration.
- Each `Builder` setter method returns `this` (a **self-reference**, similar in spirit to Singleton's self-reference in Topic 16, but for an entirely different structural purpose — enabling FLUENT chaining, not managing a single shared instance).

---

## 5. Java Implementation

```java
// ============================================
// Product — the complex, IMMUTABLE object being built
// ============================================
public class House {
    // ALL fields are final — once constructed, a House cannot change
    private final int rooms;
    private final boolean hasGarage;
    private final boolean hasPool;
    private final boolean hasGarden;
    private final boolean hasSolarPanels;

    // Private constructor — ONLY the Builder can construct a House
    private House(Builder builder) {
        this.rooms = builder.rooms;
        this.hasGarage = builder.hasGarage;
        this.hasPool = builder.hasPool;
        this.hasGarden = builder.hasGarden;
        this.hasSolarPanels = builder.hasSolarPanels;
    }

    @Override
    public String toString() {
        return "House{rooms=" + rooms + ", garage=" + hasGarage
                + ", pool=" + hasPool + ", garden=" + hasGarden
                + ", solarPanels=" + hasSolarPanels + "}";
    }

    // ============================================
    // Builder — static nested class, handles step-by-step
    // configuration before producing the final House
    // ============================================
    public static class Builder {
        // Mutable fields, WITH sensible defaults for optional attributes
        private int rooms; // required — no sensible default
        private boolean hasGarage = false;
        private boolean hasPool = false;
        private boolean hasGarden = false;
        private boolean hasSolarPanels = false;

        // Required parameter is passed via the Builder's OWN constructor,
        // ensuring it can never be forgotten
        public Builder(int rooms) {
            this.rooms = rooms;
        }

        // Each optional setter returns 'this' — enabling FLUENT chaining
        public Builder hasGarage(boolean value) {
            this.hasGarage = value;
            return this;
        }

        public Builder hasPool(boolean value) {
            this.hasPool = value;
            return this;
        }

        public Builder hasGarden(boolean value) {
            this.hasGarden = value;
            return this;
        }

        public Builder hasSolarPanels(boolean value) {
            this.hasSolarPanels = value;
            return this;
        }

        // Final assembly step — can include VALIDATION before
        // constructing the immutable House
        public House build() {
            if (rooms <= 0) {
                throw new IllegalStateException("A house must have at least 1 room.");
            }
            if (hasPool && rooms < 2) {
                // Example of validating a COMBINATION of settings,
                // not just individual fields in isolation
                throw new IllegalStateException("Houses with a pool need at least 2 rooms.");
            }
            return new House(this);
        }
    }
}

// ============================================
// Demo
// ============================================
public class BuilderDemo {
    public static void main(String[] args) {
        // Fluent, readable construction — no telescoping constructor needed
        House house1 = new House.Builder(3)
                .hasGarage(true)
                .hasPool(true)
                .hasSolarPanels(true)
                .build();

        // A minimal house — only the REQUIRED parameter is specified,
        // all optional attributes use their sensible defaults
        House house2 = new House.Builder(1)
                .build();

        System.out.println(house1);
        System.out.println(house2);
    }
}
```

**Key line-by-line notes:**
- `House`'s constructor is `private` and takes a `Builder` — this ENSURES the ONLY way to create a `House` is through `Builder.build()`, never directly.
- `Builder`'s own constructor takes `rooms` as a REQUIRED parameter — this guarantees the one truly-mandatory field can NEVER be forgotten, while all OTHER fields remain optional via chained setter calls.
- `build()` includes VALIDATION logic checking a COMBINATION of settings (`hasPool && rooms < 2`) — this kind of cross-field validation is MUCH cleaner to express here than it would be scattered across multiple overloaded constructors.
- Each setter method (`hasGarage()`, `hasPool()`, etc.) returns `this`, which is what enables the FLUENT chained syntax seen in `BuilderDemo`.

---

## 6. Dry Run

**Sample input:**
```java
House house1 = new House.Builder(3)
        .hasGarage(true)
        .hasPool(true)
        .build();
```

```
1. new House.Builder(3)
   → Builder constructor runs: rooms = 3
   → hasGarage, hasPool, hasGarden, hasSolarPanels all default to false
   → A Builder object is allocated on the heap

2. .hasGarage(true) called on the Builder
   → this.hasGarage = true
   → returns 'this' (the SAME Builder object, now with hasGarage=true)

3. .hasPool(true) called on the RETURNED Builder reference
   → this.hasPool = true
   → returns 'this' again (STILL the same Builder object)

4. .build() called
   → Validates: rooms (3) > 0? YES
   → Validates: hasPool (true) && rooms < 2? (3 < 2 is FALSE) → validation PASSES
   → new House(this) called
       → House constructor runs, COPYING builder's field values:
           this.rooms = 3, this.hasGarage = true, this.hasPool = true,
           this.hasGarden = false, this.hasSolarPanels = false
   → build() returns this newly constructed, IMMUTABLE House object
   → house1 now references this House
```

**What's happening in memory:** a SINGLE `Builder` object is created and PROGRESSIVELY mutated across the chained calls (`hasGarage()`, `hasPool()`) — each call modifying the SAME object and returning a reference to itself. Only at the FINAL `build()` call is a SEPARATE, brand-new `House` object created, with its fields COPIED from the Builder's final accumulated state — after this point, the `House` object itself never changes (all its fields are `final`).

---

## 7. Real-World Software Example

- **`StringBuilder`/`StringBuffer`**: while not a strict "Product+Builder" GoF pair, they embody the SAME core idea — incrementally building up a complex String result step by step (`.append()`, `.append()`, ...), before a FINAL `.toString()` produces the result.
- **HTTP request builders**: libraries like OkHttp's `Request.Builder` — constructing an HTTP request with OPTIONAL headers, query parameters, body, etc., via fluent chained calls before a final `.build()`.
- **`java.time` classes**: while using slightly different mechanisms, patterns like `LocalDateTime` construction via multiple `.with...()` methods echo the SAME step-by-step configuration philosophy.
- **Lombok's `@Builder` annotation**: automatically GENERATES an entire Builder class (constructor, setters, `build()`) for a Java class, based purely on annotations — a very common, widely-used real-world shortcut for this EXACT pattern.
- **UI/dialog builders**: `AlertDialog.Builder` in Android — configuring a complex dialog (title, message, buttons, icon) step by step before `.create()`/`.show()`.

---

## 8. Internal Working

**Object creation cascade:** the Builder object itself is created FIRST (cheaply, holding only primitive/simple fields), progressively MUTATED through chained method calls, and FINALLY used to construct the immutable Product in a SINGLE `build()` call — meaning the EXPENSIVE/final object creation happens exactly once, at the very end.

**Memory layout:** during the "in-progress" configuration phase, ONLY the mutable Builder object exists in memory holding the accumulated state; the immutable Product object doesn't exist AT ALL until `build()` executes — this cleanly SEPARATES the "under construction" phase from the "finished, immutable" phase in terms of actual object lifecycle.

**Method chaining mechanism:** each setter method's `return this;` is the ENTIRE mechanism behind fluent chaining — there's no special language feature involved; it's simply RETURNING a reference to the SAME object so the NEXT method call can be applied directly to the result of the previous one.

---

## 9. Before vs After

**Before (no Builder — telescoping constructors):**

```java
public class HouseBad {
    private final int rooms;
    private final boolean hasGarage;
    private final boolean hasPool;
    private final boolean hasGarden;
    private final boolean hasSolarPanels;

    // "Telescoping constructor" anti-pattern — MULTIPLE overloaded
    // constructors, each adding ONE more parameter
    public HouseBad(int rooms) {
        this(rooms, false, false, false, false);
    }

    public HouseBad(int rooms, boolean hasGarage) {
        this(rooms, hasGarage, false, false, false);
    }

    public HouseBad(int rooms, boolean hasGarage, boolean hasPool) {
        this(rooms, hasGarage, hasPool, false, false);
    }

    // ... and so on, potentially MANY more overloads,
    // each barely distinguishable from the others

    public HouseBad(int rooms, boolean hasGarage, boolean hasPool,
                     boolean hasGarden, boolean hasSolarPanels) {
        this.rooms = rooms;
        this.hasGarage = hasGarage;
        this.hasPool = hasPool;
        this.hasGarden = hasGarden;
        this.hasSolarPanels = hasSolarPanels;
    }
}

// Calling code is HARD TO READ and ERROR-PRONE:
HouseBad house = new HouseBad(3, true, false, true, false);
// Which boolean is which?? Easy to accidentally swap parameters
// of the SAME type without the compiler catching the mistake.
```

**Problems:**
- Numerous nearly-identical overloaded constructors ("telescoping constructors") that are HARD to maintain and easy to get wrong.
- Calling code (`new HouseBad(3, true, false, true, false)`) is COMPLETELY unreadable — it's impossible to tell WHICH boolean corresponds to WHICH attribute without checking the constructor signature every time.
- The compiler CANNOT catch parameter-order mistakes when MULTIPLE parameters share the same type (e.g., accidentally swapping `hasGarage` and `hasPool`, both `boolean`).

**After (Builder, as shown in Section 5):**
- Calling code (`new House.Builder(3).hasGarage(true).hasPool(true).build()`) is SELF-DOCUMENTING — each attribute is clearly NAMED at the call site, eliminating any ambiguity.
- Only ONE constructor overload is needed (on the Builder, for the truly required parameter) — no telescoping proliferation.
- Cross-field VALIDATION logic has ONE clear home (`build()`), rather than being scattered/duplicated across multiple constructor overloads.

---

## 10. SOLID Principles Connection

- **SRP**: `House` is responsible ONLY for representing a fully-configured, immutable house; `Builder` is responsible ONLY for the STEP-BY-STEP configuration process and validation — these are cleanly separated concerns.
- **OCP**: adding a NEW optional attribute (e.g., `hasBasement`) means adding ONE new field + ONE new setter method to `Builder` — existing client code calling OTHER combinations of setters remains completely unaffected.
- **Immutability as a design goal (related to but distinct from SOLID)**: Builder is a KEY enabling pattern for achieving IMMUTABLE objects with MANY optional fields — a design goal strongly valued in modern Java (especially with records/value objects) for thread-safety and predictability.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Builder pattern solve, in your own words?
2. What is the "telescoping constructor" problem, and how does Builder avoid it?
3. Why does each setter method in the Builder return `this`?

**Intermediate:**
4. Why is `House`'s constructor typically made `private` when using the Builder pattern?
5. Where should VALIDATION logic live in a Builder-based design, and why is `build()` often the right place for CROSS-FIELD validation (checking combinations of settings) rather than individual setters?
6. Is the `Builder` class itself required to be a STATIC NESTED class inside the Product? What are the tradeoffs of making it a separate, standalone class instead?

**Advanced:**
7. **Builder vs Factory/Abstract Factory** — precisely distinguish them (a natural comparison, since both are creational patterns covered in this series — Topics 8 and 9).
   *Answer: Factory Method and Abstract Factory are primarily about DECOUPLING the CLIENT from knowing WHICH CONCRETE CLASS to instantiate — the focus is on CHOOSING among different types/families of objects. Builder is primarily about managing the STEP-BY-STEP CONSTRUCTION PROCESS of a SINGLE, often COMPLEX object with MANY optional parameters — the focus is on HOW an object with many parts gets ASSEMBLED, not on choosing between different concrete types. They can be COMBINED (e.g., a Factory that internally uses a Builder to construct its products).*
8. How does Lombok's `@Builder` annotation relate to this pattern? What tradeoffs come with using a code-generation tool versus writing the Builder by hand?
9. How would you design a Builder that enforces a SPECIFIC ORDER of method calls at COMPILE TIME (e.g., you MUST call `.rooms()` before `.hasGarage()`) — this is sometimes called the "Fluent Builder with staged interfaces" or "Telescoping Builder" technique. Discuss the tradeoffs of this added complexity.
10. Can Builder be used to construct DIFFERENT REPRESENTATIONS from the SAME construction process (the ORIGINAL GoF intent, e.g., a "Director" class that uses the SAME Builder interface to construct either a "Wooden House" or a "Stone House" representation)? Describe how a `Director` class might fit into this pattern.

---

## 12. Common Mistakes

- **Making the Builder's setters MODIFY the Product directly instead of accumulating state in the Builder itself** — this defeats much of the point, since it couples the Product's mutability to the construction process rather than cleanly separating "under construction" (mutable Builder) from "finished" (immutable Product).
- **Forgetting VALIDATION entirely, or scattering it across individual setters instead of centralizing it in `build()`** — this can allow an INVALID combination of settings to slip through undetected until much later, when the bug is harder to trace back to its source.
- **Using Builder for objects with only a FEW simple parameters** — unnecessary ceremony when a straightforward constructor would be perfectly clear and sufficient.
- **Confusing Builder with Factory** — remember: Builder is about the ASSEMBLY PROCESS of ONE complex object; Factory is about CHOOSING among different concrete types (see Interview Question 7).

---

## 13. Time Complexity

Not meaningfully different from direct construction — O(1) per setter call (a simple field assignment) and O(1) for the final `build()` call (a single object construction, plus whatever validation logic is included). The pattern's value is entirely about READABILITY and CORRECTNESS of construction, not algorithmic efficiency.

---

## 14. Java API Examples

- **`java.lang.StringBuilder`**: the canonical, widely-recognized example of INCREMENTAL, step-by-step construction (`.append()` chains) culminating in a final result (`.toString()`).
- **`java.util.stream.Stream.Builder`**: Java's Streams API provides an explicit `Stream.Builder` for incrementally constructing a Stream before a final `.build()` call.
- **`javax.swing` / UI dialog builders**: many UI frameworks provide fluent Builder-style APIs for configuring complex dialogs/components before finally displaying them.
- **Lombok's `@Builder` annotation**: widely used in real-world Java codebases to AUTO-GENERATE an entire Builder class for a POJO, eliminating the boilerplate of writing one by hand.

---

## 15. Practice Problem

Implement a **Builder for a `Pizza` class**: `Pizza` should have a REQUIRED `size` (e.g., "Small"/"Medium"/"Large") and OPTIONAL attributes: `hasExtraCheese`, `hasStuffedCrust`, and a `List<String> toppings` (allowing multiple toppings to be added one at a time via a repeated `.addTopping(String)` builder method, rather than a single fixed list). Ensure the final `Pizza` object is fully immutable, and that `build()` validates that at least a `size` was specified.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **Builder for an HTTP Request object** (`HttpRequest`), supporting a required URL, an optional HTTP method (defaulting to `GET`), optional headers (potentially MULTIPLE headers, added one at a time via `.addHeader(String key, String value)`), and an optional request body (only valid for `POST`/`PUT` methods). Ensure `build()` throws a validation error if a body is set but the method is `GET`."

Think about:
- How you'd store MULTIPLE headers internally within the Builder (e.g., a `Map<String, String>` or `List<Map.Entry<String, String>>`), and how `build()` should copy this into an IMMUTABLE structure on the final `HttpRequest` object (to prevent the caller from mutating the headers AFTER construction via a lingering reference to the original mutable map).
- How the "body only valid for POST/PUT" cross-field validation rule illustrates WHY validation belongs in `build()` rather than in individual setters (which can't yet know about OTHER fields that might be set later in the chain).

---

## 17. Advanced LLD Scenario

**Design a Meal Ordering System for a Food Delivery App** using Builder, where:
- A `MealOrder` object has a REQUIRED restaurant ID and delivery address, plus OPTIONAL attributes: multiple `MenuItem`s (added incrementally via `.addItem(MenuItem item, int quantity)`), a special instructions note, a scheduled delivery time (versus "ASAP" by default), a tip amount, and a promo code
- The `build()` method needs to perform SEVERAL cross-field validations: at least ONE item must be added; if a promo code is provided, it must be validated against SOME business rule (e.g., minimum order value); if a scheduled delivery time is provided, it must be at least 30 minutes in the future
- Consider whether this Builder should be COMBINED with other patterns covered so far — e.g., could the promo code validation logic itself be implemented using STRATEGY (Topic 5), allowing different promo validation rules to be plugged in without modifying the Builder's `build()` method directly?

Consider:
- How accumulating MULTIPLE items (via repeated `.addItem()` calls) differs structurally from simply setting a SINGLE optional field — this requires the Builder to maintain an INTERNAL mutable collection (e.g., a `List<OrderLineItem>`) that grows across MULTIPLE chained calls, illustrating that Builder isn't limited to simple field-by-field configuration
- Why `build()`'s VALIDATION logic here is meaningfully more complex than the House/Pizza examples — this scenario demonstrates that real-world Builders often need to enforce genuine BUSINESS RULES, not just simple "field X requires field Y" checks
- How this scenario, like others in this series, shows multiple patterns (Builder + Strategy) naturally composing together in a realistic system, rather than being used in isolation

---

## 18. Summary

**Definition:** Builder separates the construction of a complex object from its representation, providing a step-by-step, fluent configuration process that produces a fully-assembled (often immutable) object via a final `build()` call.

**Intent:** Avoid telescoping constructors and improve readability/correctness when constructing objects with many (especially optional) parameters, while enabling the final object to remain immutable.

**Key classes:** `Product` (the complex object, often with a private constructor), `Builder` (accumulates configuration via chained setter methods, validates, and constructs the Product via `build()`).

**Advantages:** Eliminates telescoping constructors; supports immutable final objects; centralizes cross-field validation; produces self-documenting, readable construction code.

**Disadvantages:** Extra class/boilerplate per object type; overkill for simple objects; the Builder itself is necessarily mutable during construction.

**Real-world use case:** `StringBuilder`, HTTP request builders (OkHttp), UI dialog builders, Lombok's `@Builder`, complex order/configuration objects.

**Java example:** `House.Builder` fluently configuring optional attributes (`hasGarage`, `hasPool`, etc.) before `build()` produces an immutable `House`.

**Interview takeaway:** Be ready to clearly distinguish Builder (managing the ASSEMBLY of ONE complex object with many parts) from Factory/Abstract Factory (choosing among DIFFERENT concrete types/families) — a very common point of confusion since both are creational patterns.

**One-line memory trick:** *"Ordering a custom sandwich, step by step — bread, then meat, then toppings — versus handing over one giant, rigid, pre-filled order form."*

---

*End of Topic 17. Type "Next" to proceed to Topic 18: Prototype Pattern.*