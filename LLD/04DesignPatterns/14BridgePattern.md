# Topic 19: Bridge Pattern

---

## 1. Introduction

**Definition:**
The Bridge Pattern is a **structural design pattern** that DECOUPLES an ABSTRACTION from its IMPLEMENTATION, allowing the two to VARY INDEPENDENTLY — instead of a single inheritance hierarchy trying to represent EVERY combination of "what it does" and "how it does it," Bridge SPLITS these into TWO SEPARATE class hierarchies connected by COMPOSITION (a "bridge"), rather than inheritance.

**Why it exists / what problem it solves:**
Imagine you're designing shapes (`Circle`, `Square`) that can each be rendered using DIFFERENT rendering engines (`VectorRenderer`, `RasterRenderer`). If you tried to model this with a SINGLE inheritance hierarchy, you'd need a class for EVERY COMBINATION: `VectorCircle`, `RasterCircle`, `VectorSquare`, `RasterSquare` — and if you added a THIRD renderer or a THIRD shape, the number of classes EXPLODES multiplicatively (this is known as "class explosion"). Worse, adding ONE new renderer means creating a NEW subclass for EVERY EXISTING shape.

Bridge solves this by recognizing that "shape type" (Circle vs Square) and "rendering mechanism" (Vector vs Raster) are TWO INDEPENDENT DIMENSIONS of variation. Instead of ONE combined hierarchy, Bridge creates TWO SEPARATE hierarchies — an "Abstraction" hierarchy (`Shape` → `Circle`, `Square`) and an "Implementation" hierarchy (`Renderer` → `VectorRenderer`, `RasterRenderer`) — connected by the Abstraction HOLDING a reference to an Implementation object. Now, adding a new shape means ONE new class; adding a new renderer ALSO means just ONE new class — no multiplicative explosion.

**When it should be used:**
- When you have TWO (or more) INDEPENDENT dimensions of variation for a concept, and combining them into ONE inheritance hierarchy would cause a COMBINATORIAL "class explosion"
- When you want to be able to SWITCH the implementation at RUNTIME, independent of the abstraction using it
- When you want CHANGES to the implementation hierarchy to NOT require recompiling/changing the abstraction hierarchy, and vice versa

**When it should NOT be used:**
- When there's genuinely only ONE dimension of variation — a simple, single inheritance hierarchy is sufficient and simpler
- When the two "dimensions" you're considering aren't ACTUALLY independent — if every abstraction only ever pairs with ONE specific implementation anyway, Bridge's added indirection provides no real benefit
- When the system is small/stable enough that class explosion was never going to be a REAL problem in practice

**Advantages:**
- Eliminates combinatorial class explosion by separating dimensions of variation into independent hierarchies
- Abstraction and Implementation can each EVOLVE independently, without affecting the other
- Implementation can be SWAPPED at runtime, since the Abstraction holds a REFERENCE (not an inheritance relationship) to it

**Disadvantages:**
- Adds an extra layer of INDIRECTION, which can make the code slightly harder to follow for those unfamiliar with the pattern
- Requires IDENTIFYING the right "independent dimensions" upfront — applying Bridge to dimensions that AREN'T truly independent adds complexity without corresponding benefit
- More classes/interfaces overall (though each is simpler) compared to a naive, single-hierarchy approach for SMALL numbers of combinations

---

## 2. Real-World Analogy

Think of **a TV remote control and the TVs it controls**.

A "remote control" is an ABSTRACTION — it has buttons like "power," "volume up," "channel up." The ACTUAL TV (Sony, Samsung, LG) is the IMPLEMENTATION — each brand implements "turn on," "increase volume," etc., in its OWN specific way internally. Critically, you don't need a SEPARATE "SonyRemote," "SamsungRemote," "LGRemote" class for EVERY combination of remote TYPE (basic remote, advanced remote with extra buttons) and TV BRAND — instead, ANY remote can work with ANY TV brand, because the remote just sends GENERIC commands ("power," "volume up") through a COMMON interface that EACH TV brand implements in its own way. The remote (abstraction) and the TV (implementation) vary INDEPENDENTLY — you can introduce a new TV brand without touching remote designs, and a new remote design without touching TV implementations.

---

## 3. Theory

**Core idea:** Define an `Abstraction` class (or hierarchy) that HOLDS a reference to an `Implementor` INTERFACE (not a specific concrete implementation). The Abstraction's methods DELEGATE the actual work to the Implementor. BOTH the Abstraction hierarchy (different kinds of "what") and the Implementor hierarchy (different kinds of "how") can grow INDEPENDENTLY, since they're connected only through this ONE reference/interface — the "bridge."

**Working mechanism:**
```
        Abstraction                    Implementor (interface)
       /            \                    /              \
  RefinedAbs1    RefinedAbs2      ConcreteImplA      ConcreteImplB

Abstraction HOLDS a reference to an Implementor,
and DELEGATES actual work to it — this reference
IS the "bridge" connecting the two hierarchies.
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Abstraction | The high-level class defining the "what" — holds a reference to an Implementor |
| Refined Abstraction | A specific variant/extension of the Abstraction (e.g., `Circle` extending `Shape`) |
| Implementor | The interface defining the "how" — the low-level operations the Abstraction delegates to |
| Concrete Implementor | A specific implementation of the Implementor interface (e.g., `VectorRenderer`) |

**Class responsibilities:** the Abstraction is responsible for the HIGH-LEVEL logic/API exposed to clients, while DELEGATING the actual LOW-LEVEL execution details to whichever Implementor it currently holds — neither hierarchy needs to know the INTERNAL details of the other.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      Shape (Abstraction)        │
├─────────────────────┤
│ # renderer: Renderer     ◇──────→ (aggregation — THIS
├─────────────────────┤       reference IS the "bridge"
│ + Shape(Renderer r)             connecting the two
│ + draw(): void  (abstract)        hierarchies)
└──────────┬──────────┘
           △ (inherits)
   ┌───────┴───────┐
   │                       │
┌──┴────────┐  ┌──┴────────┐
│ Circle             │  │ Square             │
│ + draw()             │  │ + draw()             │
│   { renderer.        │  │   { renderer.        │
│     renderCircle();      │  │     renderSquare(); }    │
│  }                        │  └────────────┘
└────────────┘

┌─────────────────────┐
│      <<interface>>          │
│      Renderer (Implementor)     │
├─────────────────────┤
│ + renderCircle(): void        │
│ + renderSquare(): void        │
└──────────┬──────────┘
           △ (implements)
   ┌───────┴───────┐
   │                       │
┌──┴────────────┐  ┌──┴────────────┐
│ VectorRenderer        │  │ RasterRenderer        │
└───────────────┘  └───────────────┘
```

**Relationship explanation:**
- `Shape` (the Abstraction) has an **aggregation** relationship with `Renderer` (the Implementor) — this reference is the "bridge" itself; `Shape` DELEGATES its actual drawing work to whichever `Renderer` it holds.
- `Circle` and `Square` **inherit** from `Shape` — this is the Abstraction hierarchy, representing DIFFERENT KINDS OF SHAPES, each implementing `draw()` by calling the APPROPRIATE method on its held `Renderer`.
- `VectorRenderer` and `RasterRenderer` **implement** `Renderer` — this is the SEPARATE Implementor hierarchy, representing DIFFERENT RENDERING MECHANISMS, completely INDEPENDENT of which shape is being drawn.
- Critically: ANY `Shape` subclass can be combined with ANY `Renderer` implementation at RUNTIME (e.g., `new Circle(new RasterRenderer())` or `new Circle(new VectorRenderer())`) — this is the multiplicative flexibility Bridge provides WITHOUT multiplicative CLASS COUNT.

---

## 5. Java Implementation

```java
// ============================================
// Implementor interface — the "HOW" dimension
// ============================================
public interface Renderer {
    void renderCircle(float radius);
    void renderSquare(float side);
}

// ============================================
// Concrete Implementors — different rendering MECHANISMS
// ============================================
public class VectorRenderer implements Renderer {
    @Override
    public void renderCircle(float radius) {
        System.out.println("Drawing a circle of radius " + radius + " using VECTOR rendering (mathematical curves)");
    }

    @Override
    public void renderSquare(float side) {
        System.out.println("Drawing a square of side " + side + " using VECTOR rendering (mathematical lines)");
    }
}

public class RasterRenderer implements Renderer {
    @Override
    public void renderCircle(float radius) {
        System.out.println("Drawing a circle of radius " + radius + " using RASTER rendering (pixel grid)");
    }

    @Override
    public void renderSquare(float side) {
        System.out.println("Drawing a square of side " + side + " using RASTER rendering (pixel grid)");
    }
}

// ============================================
// Abstraction — the "WHAT" dimension, holds a Renderer
// ============================================
public abstract class Shape {
    // THIS reference is the "bridge" connecting Abstraction to Implementor
    protected final Renderer renderer;

    protected Shape(Renderer renderer) {
        this.renderer = renderer;
    }

    public abstract void draw();
}

// ============================================
// Refined Abstractions — different KINDS of shapes
// ============================================
public class Circle extends Shape {
    private final float radius;

    public Circle(Renderer renderer, float radius) {
        super(renderer);
        this.radius = radius;
    }

    @Override
    public void draw() {
        // Delegates the ACTUAL rendering work to whichever
        // Renderer this Circle was constructed with
        renderer.renderCircle(radius);
    }
}

public class Square extends Shape {
    private final float side;

    public Square(Renderer renderer, float side) {
        super(renderer);
        this.side = side;
    }

    @Override
    public void draw() {
        renderer.renderSquare(side);
    }
}

// ============================================
// Demo
// ============================================
public class BridgeDemo {
    public static void main(String[] args) {
        Renderer vectorRenderer = new VectorRenderer();
        Renderer rasterRenderer = new RasterRenderer();

        // ANY shape can be combined with ANY renderer at runtime —
        // 2 shapes x 2 renderers = 4 combinations, but only 4 CLASSES
        // total needed (2 shapes + 2 renderers), NOT 4 combined classes
        Shape vectorCircle = new Circle(vectorRenderer, 5.0f);
        Shape rasterCircle = new Circle(rasterRenderer, 5.0f);
        Shape vectorSquare = new Square(vectorRenderer, 3.0f);
        Shape rasterSquare = new Square(rasterRenderer, 3.0f);

        vectorCircle.draw();
        rasterCircle.draw();
        vectorSquare.draw();
        rasterSquare.draw();
    }
}
```

**Key line-by-line notes:**
- `Shape`'s constructor REQUIRES a `Renderer` — this is the explicit "bridge" being established at construction time, though it could ALSO be changed later via a setter if runtime-swappable rendering were needed.
- `Circle.draw()` and `Square.draw()` each call the APPROPRIATE method on `renderer` (`renderCircle()` vs `renderSquare()`) — the SHAPE knows WHICH operation is appropriate for itself, but doesn't know or care HOW that operation is actually implemented internally.
- Notice that adding a THIRD renderer (e.g., `RayTracingRenderer`) requires writing ONE new class implementing `Renderer` — `Circle` and `Square` need ZERO changes. Similarly, adding a THIRD shape (e.g., `Triangle`) requires ONE new class extending `Shape` — `VectorRenderer` and `RasterRenderer` need ZERO changes (aside from adding a `renderTriangle()` method to the `Renderer` interface, discussed further in Section 10's OCP note).

---

## 6. Dry Run

**Sample input:** `new Circle(new RasterRenderer(), 5.0f).draw()`

```
1. new RasterRenderer()
   → A RasterRenderer object is allocated on the heap

2. new Circle(rasterRendererInstance, 5.0f)
   → Circle's constructor runs, calling super(renderer)
   → Shape's constructor sets this.renderer = rasterRendererInstance
   → Circle's own field: this.radius = 5.0f

3. circle.draw() called
   → Circle.draw() executes
   → renderer.renderCircle(radius) called
       → Dynamic dispatch resolves renderer's ACTUAL type
         (RasterRenderer) → calls RasterRenderer's renderCircle()
       → Prints: "Drawing a circle of radius 5.0 using RASTER
         rendering (pixel grid)"
```

**What's happening in memory:** the `Circle` object holds a REFERENCE to a `RasterRenderer` object — TWO separate objects on the heap, connected via this ONE reference (the "bridge"). When `draw()` is called, `Circle` doesn't perform ANY rendering logic itself — it simply forwards the call to its held `Renderer` reference, which resolves via dynamic dispatch to whichever CONCRETE renderer was actually supplied at construction time.

---

## 7. Real-World Software Example

- **JDBC (Java Database Connectivity)**: the `java.sql.Driver`/`Connection`/`Statement` APIS form the ABSTRACTION your application code uses, while EACH database vendor (MySQL, PostgreSQL, Oracle) provides its OWN CONCRETE implementation of these interfaces — your application code (the abstraction side) doesn't change regardless of WHICH database driver (implementation side) is plugged in.
- **Cross-platform UI frameworks**: an abstract `Window`/`Button` concept (abstraction) that can be rendered using DIFFERENT underlying platform-specific rendering engines (implementation) — Windows, macOS, Linux — without needing a separate `Window` subclass PER platform.
- **SLF4J logging facade**: your application code uses the SLF4J `Logger` ABSTRACTION, while the ACTUAL logging IMPLEMENTATION (Logback, Log4j2, java.util.logging) can be swapped WITHOUT changing any application code — a genuine, widely-used real-world Bridge.
- **Device driver architectures**: an operating system's abstract "printer" interface (abstraction) that can work with MANY different concrete printer DRIVERS (implementation) from different manufacturers.

---

## 8. Internal Working

**Object creation:** typically, the CONCRETE Implementor is chosen and constructed FIRST (or injected via dependency injection), and then PASSED INTO the Abstraction's constructor — establishing the "bridge" connection at the moment the Abstraction object is created (though it can also be set/swapped later via a setter method if runtime flexibility is needed).

**Runtime interactions / call flow:** every Abstraction method call that needs LOW-LEVEL work DELEGATES to the held Implementor reference — this is a SINGLE, direct method call (via dynamic dispatch), not a chain or recursive structure; Bridge's structure is fundamentally SIMPLER at the call-flow level than patterns like Composite or Decorator.

**Memory usage:** each Abstraction instance holds ONE reference to an Implementor — Implementor instances CAN be shared across MULTIPLE Abstraction instances (e.g., the SAME `RasterRenderer` instance could be reused by MANY different `Circle`/`Square` objects), since the Implementor itself typically holds no shape-specific state.

**Dynamic binding — TWO levels:** Bridge involves dynamic dispatch at TWO INDEPENDENT levels — first, WHICH Abstraction subclass's `draw()` method runs (Circle's vs Square's), and SEPARATELY, WHICH Implementor's method actually executes (VectorRenderer's vs RasterRenderer's) — these TWO dispatches are entirely INDEPENDENT of each other, which is the exact mechanism enabling the "any combination" flexibility.

---

## 9. Before vs After

**Before (no Bridge — combined inheritance hierarchy, "class explosion"):**

```java
// Attempting to model BOTH shape type AND rendering mechanism
// through a SINGLE inheritance hierarchy:
public abstract class ShapeBad {
    public abstract void draw();
}

public class VectorCircle extends ShapeBad {
    private float radius;
    @Override public void draw() { /* vector circle drawing logic */ }
}

public class RasterCircle extends ShapeBad {
    private float radius;
    @Override public void draw() { /* raster circle drawing logic */ }
}

public class VectorSquare extends ShapeBad {
    private float side;
    @Override public void draw() { /* vector square drawing logic */ }
}

public class RasterSquare extends ShapeBad {
    private float side;
    @Override public void draw() { /* raster square drawing logic */ }
}
// Adding a THIRD renderer (e.g., RayTracing) means creating
// RayTracingCircle, RayTracingSquare — TWO MORE classes.
// Adding a THIRD shape (e.g., Triangle) means creating
// VectorTriangle, RasterTriangle — TWO MORE classes.
// This is the "class explosion": N shapes x M renderers = N*M classes.
```

**Problems:**
- The number of classes grows MULTIPLICATIVELY (N shapes × M renderers) — with just 2 shapes and 2 renderers, that's already 4 classes; scaling to 5 shapes and 4 renderers would require 20 SEPARATE classes.
- Adding a SINGLE new renderer requires creating a NEW subclass for EVERY EXISTING shape — massive code duplication and maintenance burden.
- Rendering logic (e.g., "how vector rendering draws a circle") is DUPLICATED/scattered across MULTIPLE classes rather than being cohesively grouped in ONE place per renderer.

**After (Bridge, as shown in Section 5):**
- Only N + M classes are needed (2 shapes + 2 renderers = 4 classes total) — REGARDLESS of how many COMBINATIONS are actually used.
- Adding a new renderer requires ONE new class implementing `Renderer` — ZERO changes to `Circle`/`Square`.
- Adding a new shape requires ONE new class extending `Shape` — ZERO changes to `VectorRenderer`/`RasterRenderer`.

---

## 10. SOLID Principles Connection

- **OCP (with an important nuance)**: Bridge is EXCELLENT for OCP when adding a NEW shape (Abstraction) OR a new renderer (Implementor) INDEPENDENTLY. However, similar to Abstract Factory's tension (Topic 9), adding a genuinely NEW OPERATION to the Implementor interface (e.g., `renderTriangle()`) still requires updating EVERY existing Concrete Implementor — this is a real, honest tradeoff worth mentioning.
- **SRP**: `Shape` subclasses are responsible ONLY for representing shape-specific data/logic (radius, side length); `Renderer` implementations are responsible ONLY for the MECHANICS of rendering — these are cleanly separated concerns.
- **DIP**: `Shape` depends only on the `Renderer` ABSTRACTION (interface), never on a specific concrete renderer — enabling ANY compatible renderer to be plugged in.
- **LSP**: any `Renderer` implementation can be substituted wherever a `Renderer` is expected, and `Shape` subclasses behave correctly regardless of WHICH concrete renderer is actually supplied.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Bridge pattern solve, in your own words?
2. What is "class explosion," and how does Bridge prevent it?
3. In the shape/renderer example, what are the TWO independent dimensions of variation?

**Intermediate:**
4. Why is the connection between Abstraction and Implementor described as COMPOSITION rather than INHERITANCE? Why does this matter?
5. Could the Renderer be SWAPPED at runtime for an EXISTING Shape instance (e.g., changing a Circle from vector to raster rendering after it's already been created)? What would this require in the design?
6. Give a real-world JDK or framework example of Bridge (e.g., JDBC, SLF4J) and explain WHICH part is the Abstraction and WHICH is the Implementor.

**Advanced:**
7. **Bridge vs Adapter** — precisely distinguish them (a natural, very common comparison, since both were covered in this series — Adapter in Topic 15).
   *Answer: Adapter is applied RETROACTIVELY/AFTER THE FACT, to make an ALREADY-EXISTING, incompatible class work with a client's expected interface — it's fundamentally a FIX for a compatibility problem that already exists. Bridge is a PROACTIVE, UPFRONT design decision, made BEFORE any incompatibility exists, deliberately SEPARATING two INDEPENDENT dimensions of variation to prevent a combinatorial class explosion — it's not fixing a mismatch, but preventing a SCALABILITY problem. Structurally they can look SIMILAR (both involve one object holding/delegating to another via an interface), but their INTENT and the TIMING of when they're applied differ fundamentally.*
8. **Bridge vs Strategy** — both involve a class holding a reference to an interchangeable interface. What's the conceptual DIFFERENCE in INTENT?
   *Answer: Strategy (Topic 5) is about making ONE SPECIFIC ALGORITHM/BEHAVIOR interchangeable within an OTHERWISE STABLE class — the FOCUS is on swapping ONE behavioral aspect. Bridge is about separating TWO ENTIRE HIERARCHIES (Abstraction AND Implementor) that BOTH have MULTIPLE VARIANTS, specifically to prevent COMBINATORIAL explosion between them — the SCOPE and INTENT (preventing class explosion across TWO growing hierarchies) is broader and more structural than Strategy's more FOCUSED "swap one algorithm" goal.*
9. How would you extend this example to support a THIRD independent dimension of variation (e.g., "color scheme" IN ADDITION to shape type and rendering mechanism)? Does Bridge naturally extend to more than two dimensions, or does it become unwieldy?
10. Discuss how the Abstraction's constructor REQUIRING an Implementor (as shown in Section 5) relates to DEPENDENCY INJECTION principles — is Bridge, in a sense, a specific APPLICATION of DI-style thinking to the problem of hierarchy design?

---

## 12. Common Mistakes

- **Applying Bridge to dimensions that AREN'T actually independent** — if, in reality, every "shape" only EVER pairs with ONE specific "renderer" and this will NEVER change, the added indirection provides no genuine benefit and is unnecessary complexity.
- **Confusing Bridge with Adapter** — remember: Bridge is a PROACTIVE, upfront design decision to prevent class explosion; Adapter is a REACTIVE fix for an EXISTING interface mismatch (see Interview Question 7).
- **Failing to identify class explosion EARLY enough** — if a codebase ALREADY has a large, unwieldy combined inheritance hierarchy (like the "Before" example), RETROFITTING Bridge later is a much bigger refactoring effort than designing WITH Bridge from the start when the "two independent dimensions" pattern is first recognized.
- **Putting too much LOGIC in the Abstraction that really belongs in the Implementor** (or vice versa) — misjudging WHERE the line between "what" and "how" belongs can blur the clean separation Bridge is meant to provide.

---

## 13. Time Complexity

Not meaningfully different from a normal virtual method call plus ONE additional delegation — O(1) overhead for the Abstraction-to-Implementor delegation call. The pattern's value is purely STRUCTURAL/organizational (preventing class explosion), not algorithmic.

---

## 14. Java API Examples

- **JDBC**: `java.sql.Connection`, `Statement`, `ResultSet` form the ABSTRACTION your application code uses uniformly; EACH database vendor's driver provides the CONCRETE Implementor-side logic.
- **SLF4J**: the `org.slf4j.Logger` interface is the ABSTRACTION application code depends on; Logback, Log4j2, or `java.util.logging` bindings serve as the swappable IMPLEMENTATION underneath.
- **AWT's peer-based architecture** (historical): AWT components delegated their ACTUAL rendering to platform-specific "peer" objects — a classic Bridge-style separation between the cross-platform API (abstraction) and native platform rendering (implementation).
- **Java's `java.util.concurrent.Executor`/`ExecutorService`**: the Executor ABSTRACTION (submit tasks) is decoupled from the SPECIFIC threading/execution IMPLEMENTATION (fixed thread pool, cached thread pool, single-threaded, etc.) — different `ExecutorService` implementations can be swapped without changing task-submission code.

---

## 15. Practice Problem

Implement a **Bridge pattern for a messaging system**: create a `MessageSender` (Implementor) interface with a `sendMessage(String content)` method, implemented by `EmailSender` and `SMSSender`. Create an abstract `Notification` (Abstraction) class holding a `MessageSender`, with subclasses `UrgentNotification` (prepends "URGENT: " to the content before sending) and `RegularNotification` (sends content as-is). Demonstrate all four combinations (Urgent+Email, Urgent+SMS, Regular+Email, Regular+SMS) working correctly with only 4 total classes.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **cross-platform media player** using Bridge: a `MediaPlayer` abstraction hierarchy (`AudioPlayer`, `VideoPlayer`) needs to work across DIFFERENT operating system audio/video APIs (`WindowsMediaAPI`, `MacMediaAPI`, `LinuxMediaAPI` — the Implementor hierarchy). Design this so that adding support for a NEW operating system requires writing only ONE new Implementor class, and adding a NEW media type (e.g., `ImageViewer`) requires only ONE new Abstraction class."

Think about:
- What OPERATIONS the Implementor interface needs to expose (e.g., `playAudio()`, `playVideo()`, or more GENERIC primitives that both audio and video playback can build on top of) — how granular should the Implementor's interface be?
- How this scenario's structure directly parallels the shape/renderer example — recognizing this recurring "two independent dimensions" SHAPE quickly, across different domains, is a hallmark of strong LLD interview performance.

---

## 17. Advanced LLD Scenario

**Design a Multi-Channel Notification System for an Enterprise Application** using Bridge, where:
- The ABSTRACTION hierarchy represents DIFFERENT KINDS of notifications (`SecurityAlert`, `MarketingPromo`, `SystemMaintenanceNotice`), each with its OWN content-formatting logic and business rules (e.g., `SecurityAlert` always marks itself high-priority; `MarketingPromo` includes an unsubscribe link)
- The IMPLEMENTOR hierarchy represents DIFFERENT DELIVERY CHANNELS (`EmailChannel`, `SMSChannel`, `PushNotificationChannel`, `SlackChannel`), each handling the LOW-LEVEL mechanics of ACTUALLY delivering a message through that specific channel (including channel-specific formatting limits, e.g., SMS having a character limit that requires truncation)
- The system needs to support SENDING THE SAME notification kind through MULTIPLE different channels (e.g., a `SecurityAlert` might need to go out via BOTH email AND SMS simultaneously) — consider whether this requires the Abstraction to hold a LIST of Implementors rather than just ONE, and how that would change the design

Consider:
- How this scenario's "two independent dimensions" (notification KIND vs delivery CHANNEL) directly mirrors the shape/renderer structure, despite being an entirely different business domain — a strong indicator of genuinely understanding the PATTERN'S SHAPE rather than just memorizing ONE example
- How you'd handle CHANNEL-SPECIFIC formatting constraints (e.g., SMS's character limit) WITHOUT leaking this concern into the Abstraction hierarchy — this concern belongs ENTIRELY within each Concrete Implementor, preserving the clean separation Bridge is meant to provide
- Whether supporting "one notification kind, multiple simultaneous channels" is BEST solved by modifying Bridge itself (e.g., an Abstraction holding a `List<NotificationChannel>`) or by COMBINING Bridge with another pattern already covered (e.g., Composite, Topic 14, to represent "a composite delivery channel" made up of multiple individual channels) — this is a genuinely interesting design question with reasonable arguments for either approach

---

## 18. Summary

**Definition:** Bridge decouples an abstraction from its implementation so both can vary independently, by connecting two SEPARATE class hierarchies through composition (a "bridge" reference) rather than combining them into one inheritance hierarchy.

**Intent:** Prevent combinatorial "class explosion" when a concept has TWO (or more) INDEPENDENT dimensions of variation, and allow each dimension to evolve/extend without affecting the other.

**Key classes:** `Abstraction` (high-level "what," holds an Implementor reference), `Refined Abstraction` (specific variants of the abstraction), `Implementor` (interface for the "how"), `Concrete Implementor` (specific implementations of the how).

**Advantages:** Eliminates class explosion; abstraction and implementation evolve independently; implementation swappable at runtime.

**Disadvantages:** Extra layer of indirection; requires correctly identifying genuinely independent dimensions upfront; more classes overall for small combination counts.

**Real-world use case:** JDBC (application code vs database drivers), SLF4J (logging API vs logging implementation), cross-platform UI/media frameworks.

**Java example:** `Shape` (Circle/Square) holding a `Renderer` (Vector/Raster), allowing any shape-renderer combination with only 4 total classes instead of 4+ combined ones.

**Interview takeaway:** Be ready to clearly distinguish Bridge (a PROACTIVE, upfront design decision to prevent class explosion across two growing hierarchies) from Adapter (a REACTIVE fix for an already-existing interface mismatch, Topic 15) and from Strategy (a more FOCUSED swap of a single algorithm, Topic 5) — this trio of "delegation-based" patterns is frequently confused, and clear INTENT-based distinctions are a strong interview signal.

**One-line memory trick:** *"One remote control, many TV brands — the remote (abstraction) and the TV (implementation) vary completely independently."*

---

*End of Topic 19. Type "Next" to proceed to Topic 20: Facade Pattern.*