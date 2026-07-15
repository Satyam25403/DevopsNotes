# Topic 8: Factory Design Pattern

---

## 1. Introduction

**Definition:**
The Factory Pattern (also called Factory Method) is a **creational design pattern** that defines an interface or method for creating an object, but lets subclasses or a dedicated factory class decide WHICH concrete class to instantiate — decoupling the CLIENT code (that needs an object) from the CONCRETE CLASSES (that get instantiated).

**Why it exists / what problem it solves:**
Without Factory, client code that needs to create objects typically uses `new ConcreteClass()` directly, scattered throughout the codebase. This creates problems:
- The client is TIGHTLY COUPLED to specific concrete classes — if you need to change WHICH class gets created (e.g., switch from `MySqlConnection` to `PostgresConnection`), you must find and edit every single place `new MySqlConnection()` appears.
- Object creation logic (which might involve conditional logic, configuration reading, or complex setup) gets duplicated everywhere a new instance is needed.
- Adding a new type requires touching client code directly — violates OCP.

Factory solves this by centralizing "which concrete class to create" decision-making in ONE place — client code asks the Factory for "a Shape" or "a Connection," and the Factory handles the decision of exactly which concrete implementation to return.

**When it should be used:**
- When the exact type of object needed isn't known until runtime (e.g., based on user input, configuration, or an external condition)
- When object creation involves non-trivial logic that shouldn't be duplicated across the codebase
- When you want client code to depend only on an abstraction/interface, never on concrete classes (DIP)

**When it should NOT be used:**
- When there's only ever ONE class that will ever be created — a Factory adds indirection with no real benefit
- When object creation is trivially simple (just `new SomeClass()` with no conditional logic) and unlikely to ever vary

**Advantages:**
- Decouples client code from concrete classes
- Centralizes object creation logic, avoiding duplication
- New types can be added by extending the Factory, without modifying client code (OCP)

**Disadvantages:**
- Adds an extra layer of abstraction/indirection, which can be unnecessary for simple cases
- Can lead to a large `switch`/`if-else` block INSIDE the factory itself, which — while centralized — still needs modification for every new type (a nuance discussed further in Common Mistakes)

---

## 2. Real-World Analogy

Think of **vehicle manufacturing**.

A car factory doesn't hand YOU (the customer) a pile of parts and expect you to assemble your own car. Instead, you tell the factory "I want a Sedan" or "I want an SUV," and the FACTORY handles all the internal complexity of assembling the correct vehicle — which parts to use, which assembly line to route it through, which specific configuration to apply.

You (the client) don't need to know HOW an SUV is actually built internally — you just ask for one, and you receive a fully-formed vehicle satisfying your request. If the factory later starts producing a new vehicle type (say, an Electric SUV), you still just ask for "an SUV" (or a new specific request), and the factory handles it — your INTERACTION with the factory doesn't fundamentally change.

---

## 3. Theory

**Core idea:** Instead of client code directly calling `new ConcreteClass()`, it calls a Factory method/class, passing some indicator of what's needed (a type, a configuration, an enum) — and the Factory returns an object typed as the common INTERFACE, while internally deciding which CONCRETE class to actually instantiate.

**Working mechanism:**
```
┌─────────────────────┐
│      Client Code                │
│  Shape shape = ShapeFactory.       │
│      createShape("CIRCLE");           │  ← client depends only
└──────────┬──────────┘     on the Shape interface,
           │ requests               NEVER on "Circle" directly
           ↓
┌─────────────────────┐
│      ShapeFactory                │
│  + createShape(type): Shape        │
│    (internally decides WHICH          │
│     concrete class to instantiate)      │
└──────────┬──────────┘
           │ returns an instance of
           ↓
┌─────────────────────┐
│      <<interface>>          │
│         Shape                    │
└──────────┬──────────┘
           △
   ┌───────┼───────┐
┌──┴────┐    ┌──┴────┐
│Circle    │    │Square    │
└────────┘    └────────┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Product | The common interface/abstract type the factory produces |
| Concrete Product | A specific implementation the factory can create |
| Factory | The class/method responsible for the creation decision |

**Two common variations:**
```
Simple Factory (not a formal GoF pattern, but extremely common):
   A single class with a method (often static) that takes a
   parameter and returns the appropriate concrete object.

Factory Method (the formal GoF pattern):
   An ABSTRACT method in a base class, OVERRIDDEN by subclasses,
   each subclass deciding which concrete product to create —
   pushes the "which concrete class" decision into subclassing
   rather than a single method's conditional logic.
```

---

## 4. UML / Class Diagram

**Simple Factory variant (most common in practice):**
```
┌─────────────────────┐
│      <<interface>>          │
│         Shape                    │
├─────────────────────┤
│ + draw(): void                       │
└──────────┬──────────┘
           △
   ┌───────┼───────┐
   │               │
┌──┴────┐    ┌──┴────┐
│Circle    │    │Square    │
└────────┘    └────────┘

┌─────────────────────┐
│      ShapeFactory                │
├─────────────────────┤
│ + createShape(type: String): Shape   │  ← decides which concrete
└─────────────────────┘     class to instantiate
```

**Factory Method variant (formal GoF pattern):**
```
┌─────────────────────┐
│      <<abstract>>           │
│      ShapeFactory                │
├─────────────────────┤
│ + createShape(): Shape (abstract)    │  ← subclasses decide
└──────────┬──────────┘
           △ (extends)
   ┌───────┼───────┐
   │               │
┌──┴──────────┐ ┌──┴──────────┐
│CircleFactory       │ │SquareFactory       │
│createShape() {          │ │createShape() {          │
│  return new Circle(); }   │ │  return new Square(); }   │
└────────────┘ └────────────┘
```

**Relationship explanation:**
- In the Simple Factory version, `ShapeFactory` has a **dependency** on `Circle`/`Square` (it creates them internally) but the CLIENT only depends on `Shape` and `ShapeFactory` — never directly on `Circle`/`Square`.
- In the formal Factory Method version, `CircleFactory`/`SquareFactory` **extend** `ShapeFactory`, each overriding `createShape()` — this pushes the creation decision to SUBCLASSING rather than a conditional inside one method, itself a form of applying OCP to the factory's own internals.

---

## 5. Java Implementation

```java
// ============================================
// Product interface
// ============================================
public interface Shape {
    void draw();
}

// ============================================
// Concrete Products
// ============================================
public class Circle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Circle");
    }
}

public class Square implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Square");
    }
}

public class Triangle implements Shape {
    @Override
    public void draw() {
        System.out.println("Drawing a Triangle");
    }
}

// ============================================
// SIMPLE FACTORY — a single class centralizing
// the creation decision (most common in real codebases)
// ============================================
public class ShapeFactory {
    // Using an enum instead of raw Strings avoids typos and
    // gives compile-time safety on valid shape types
    public enum ShapeType {
        CIRCLE, SQUARE, TRIANGLE
    }

    public static Shape createShape(ShapeType type) {
        switch (type) {
            case CIRCLE:
                return new Circle();
            case SQUARE:
                return new Square();
            case TRIANGLE:
                return new Triangle();
            default:
                throw new IllegalArgumentException("Unknown shape type: " + type);
        }
    }
}

// ============================================
// FACTORY METHOD (formal GoF pattern) — an abstract
// creator class, with subclasses deciding what to create
// ============================================
public abstract class ShapeCreator {
    // The "factory method" — subclasses override this
    protected abstract Shape createShape();

    // A template method (previewing Topic 27) that uses the
    // factory method internally, without knowing the concrete type
    public void renderShape() {
        Shape shape = createShape();
        System.out.println("Rendering...");
        shape.draw();
    }
}

public class CircleCreator extends ShapeCreator {
    @Override
    protected Shape createShape() {
        return new Circle();
    }
}

public class SquareCreator extends ShapeCreator {
    @Override
    protected Shape createShape() {
        return new Square();
    }
}

// ============================================
// Demo
// ============================================
public class FactoryDemo {
    public static void main(String[] args) {
        // Using the Simple Factory — client depends ONLY on
        // ShapeFactory and Shape, never on Circle/Square/Triangle directly
        Shape shape1 = ShapeFactory.createShape(ShapeFactory.ShapeType.CIRCLE);
        shape1.draw();

        Shape shape2 = ShapeFactory.createShape(ShapeFactory.ShapeType.TRIANGLE);
        shape2.draw();

        // Using the Factory Method variant
        ShapeCreator creator = new SquareCreator();
        creator.renderShape();
    }
}
```

**Key line-by-line notes:**
- `ShapeFactory.createShape(ShapeType type)` — a `static` method centralizing the `new Circle()`/`new Square()`/`new Triangle()` decision in ONE place; client code never writes `new Circle()` directly.
- Using an `enum ShapeType` rather than a raw `String` parameter — avoids typo-prone string comparisons (`"CIRCEL"` silently failing) and gives compile-time checking of valid values.
- `ShapeCreator` (abstract) with `createShape()` as an `abstract` method — subclasses like `CircleCreator` decide the concrete type via OVERRIDING, rather than a conditional statement — this is the FORMAL Factory Method pattern, distinct from the simpler static-factory-method version above it.
- `renderShape()` calls `createShape()` internally WITHOUT knowing which concrete `Shape` it'll get back — it only knows it'll get SOME `Shape`, and can call `.draw()` on it polymorphically.

---

## 6. Dry Run

**Sample input:** `ShapeFactory.createShape(ShapeFactory.ShapeType.TRIANGLE)`

```
1. Client calls ShapeFactory.createShape(ShapeType.TRIANGLE)
   → This is a STATIC method call — no ShapeFactory object
     needs to be instantiated first

2. Inside createShape(), the switch statement evaluates "type"
   → type == TRIANGLE matches the "case TRIANGLE:" branch
   → Executes: return new Triangle();
   → A new Triangle object is allocated on the heap

3. The reference returned is typed as "Shape" (the interface),
   but actually POINTS TO a Triangle object

4. Client: shape2 = <the returned Triangle, referenced as Shape>

5. shape2.draw() called
   → Dynamic dispatch (Topic 2): JVM checks actual type → Triangle
   → Invokes Triangle's draw() implementation
   → Prints: "Drawing a Triangle"
```

**What's happening in memory:** the client's variable (`shape2`) is DECLARED as `Shape`, meaning the COMPILER only allows calling methods that exist on the `Shape` interface — the client code literally CANNOT accidentally call some `Triangle`-specific method that isn't part of `Shape`, since the compiler doesn't even know (at the client's point of view) that it's specifically a `Triangle`. This enforced ignorance is precisely what keeps the client decoupled from concrete classes.

---

## 7. Real-World Software Example

- **JDBC's `DriverManager.getConnection()`**: you call `DriverManager.getConnection(url)`, and internally, the JDBC framework decides WHICH `Connection` implementation to return based on the URL's prefix (`jdbc:mysql://...` vs `jdbc:postgresql://...`) — you never write `new MySqlConnection()` directly.
- **`java.util.Calendar.getInstance()`**: returns a `Calendar` object, but the ACTUAL concrete class returned can vary based on locale/timezone settings — the client never directly instantiates a specific calendar implementation.
- **Spring's `BeanFactory`**: the entire concept of a Spring "bean" being created and injected is fundamentally a Factory pattern at the framework level — you declare "I need a `PaymentService`," and Spring's factory machinery decides which concrete implementation to construct and hand you.
- **Logger factories**: `LoggerFactory.getLogger(MyClass.class)` (SLF4J) — returns a `Logger` interface instance, with the actual concrete logging implementation (Logback, Log4j2) decided by the factory based on what's on the classpath.

---

## 8. Internal Working

**Static factory methods**: since `ShapeFactory.createShape()` is `static`, it's called on the CLASS itself, not on an instance — the JVM resolves this at COMPILE TIME (it's not a polymorphic/dynamic dispatch call at all, since there's no inheritance involved in calling a static method).

**The returned object itself IS still polymorphic**: even though the FACTORY METHOD call is static/direct, what it RETURNS (a `Shape` reference actually pointing to a `Circle`, `Square`, or `Triangle`) fully participates in dynamic dispatch for all SUBSEQUENT method calls on that object (like `.draw()`).

**Factory Method (formal) variant internals**: `renderShape()` calling `createShape()` is a NORMAL polymorphic call (not static) — since `ShapeCreator creator = new SquareCreator()`, calling `creator.renderShape()` internally calls `this.createShape()`, which DYNAMICALLY dispatches to `SquareCreator`'s override, returning a `Square`. This is a subtle but important detail: the abstract base class's method (`renderShape`) calls an ABSTRACT method (`createShape`) that gets resolved to the SUBCLASS's implementation at runtime — a pattern sometimes called "template method calling a hook method," directly foreshadowing Topic 27 (Template Method Pattern).

---

## 9. Before vs After

**Before (client tightly coupled to concrete classes):**

```java
public class ShapeRenderer {
    public void render(String type) {
        Shape shape;
        if (type.equals("CIRCLE")) {
            shape = new Circle();       // direct instantiation
        } else if (type.equals("SQUARE")) {
            shape = new Square();       // direct instantiation
        } else {
            throw new IllegalArgumentException("Unknown type");
        }
        shape.draw();
    }
}
// If used in 10 DIFFERENT places across the codebase, this
// exact if/else block (and its associated direct "new" calls)
// would need to be DUPLICATED in every one of those 10 places
```

**Problems:**
- The conditional creation logic is duplicated everywhere an object of this "family" is needed.
- Adding a new shape type means updating EVERY duplicated copy of this logic across the codebase.
- Client classes are coupled directly to `Circle`/`Square` — DIP violation.

**After (Factory pattern, as shown in Section 5):**
- The creation decision lives in EXACTLY ONE place (`ShapeFactory`).
- Any of the 10 places needing a shape simply call `ShapeFactory.createShape(type)` — zero duplicated conditional logic.
- Adding `Triangle` required changes ONLY inside `ShapeFactory` — every calling site remains untouched.

---

## 10. SOLID Principles Connection

- **DIP**: client code depends on the `Shape` interface (and the `ShapeFactory` abstraction point), never on concrete classes like `Circle` directly.
- **SRP**: `ShapeFactory`'s only responsibility is deciding which concrete `Shape` to create — it doesn't also handle drawing logic, business logic, etc.
- **OCP** (nuanced): the FORMAL Factory Method variant (using subclassing, like `CircleCreator`/`SquareCreator`) is genuinely OCP-compliant — adding a new shape means adding a new `ShapeCreator` subclass, with zero modification to existing code. The SIMPLE Factory variant (using a `switch` statement) is LESS perfectly OCP-compliant — adding a new shape STILL requires editing the `switch` statement inside `ShapeFactory`, though this is centralized to one place rather than scattered everywhere.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Factory pattern solve?
2. What's the difference between "Simple Factory" and the formal "Factory Method" pattern?
3. Give a real-world Java API example of the Factory pattern.

**Intermediate:**
4. Why might using an `enum` for the type parameter be preferable to a raw `String` in a factory method?
5. Does the Simple Factory variant (using a switch statement) fully satisfy the Open/Closed Principle? Why or why not?
6. How does the Factory pattern support the Dependency Inversion Principle?

**Advanced:**
7. **Factory Method vs Abstract Factory** — what's the key structural/conceptual difference?
   *Answer: Factory Method (this topic) creates ONE product, typically via a single overridable method, often with a single hierarchy of products. Abstract Factory (Topic 9) creates FAMILIES of RELATED products together (e.g., a "UIFactory" creating a matching Button AND Checkbox AND ScrollBar for a specific OS theme) — Abstract Factory is essentially "a factory of factories," coordinating multiple related Factory Methods together to ensure the products it creates are compatible/consistent with each other.*
8. How would you combine Factory with Strategy (Topic 5) to let client code obtain a strategy WITHOUT knowing concrete strategy class names?
9. What are the tradeoffs of a Factory method being `static` versus an instance method on an injectable Factory object?
   *Answer: A static factory method is simple but CANNOT be mocked/substituted in unit tests easily (static calls are harder to intercept), and cannot itself be swapped via dependency injection. An instance-based factory (injected as a dependency) is more testable and flexible, at the cost of slightly more setup.*
10. How does Spring's dependency injection container relate to the Factory pattern at a conceptual level?
11. What happens if a Factory needs to create an object requiring COMPLEX, multi-step construction (many optional parameters)? Does Factory alone solve this well, or would you reach for another pattern?
    *Answer: For complex construction with many optional parameters, the Builder pattern (Topic 17) is typically more appropriate than Factory alone — Factory decides WHICH class to instantiate; Builder handles HOW to assemble a complex object step-by-step. They're often used together: a Factory might internally use a Builder to construct the object it returns.*

---

## 12. Common Mistakes

- **Believing the Simple Factory (switch-statement) version is "fully OCP-compliant"** — it centralizes the decision, which is valuable, but STILL requires modifying the switch statement for new types; only the formal Factory Method (subclassing) variant achieves true OCP compliance for the creation logic itself.
- **Using string-based type parameters instead of enums** — prone to typos, no compile-time safety, harder to discover all valid options via IDE autocomplete.
- **Putting unrelated business logic inside the Factory** — a Factory's job is CREATION, not also validation, business rules, or persistence; mixing these violates SRP.
- **Overusing Factory for trivially simple object creation** — if there's genuinely only one class that will EVER be created, wrapping `new SimpleClass()` in a factory method adds pure indirection with no benefit.

---

## 13. Time Complexity

**Simple Factory with a switch/if-else**: O(1) for a small, fixed number of cases (switch statements are typically O(1) or close to it for a small number of branches) — not a meaningful performance concern in virtually any real system. **Factory Method (subclass-based)**: O(1) — a single virtual method dispatch to determine which subclass's `createShape()` runs.

---

## 14. Java API Examples

- **`java.util.Calendar.getInstance()`**: returns different concrete `Calendar` implementations based on locale.
- **`java.text.NumberFormat.getInstance()`**: similarly returns different formatting implementations based on locale.
- **`javax.xml.parsers.DocumentBuilderFactory.newInstance()`**: a well-known, explicitly-named Factory in the JDK itself.
- **`java.sql.DriverManager.getConnection()`**: decides which JDBC driver's `Connection` implementation to return based on the connection URL.
- **SLF4J's `LoggerFactory.getLogger()`**: returns a `Logger` interface instance, with the actual backing implementation resolved via classpath scanning.

---

## 15. Practice Problem

Implement a **notification factory**: a `Notification` interface with `send(message)`, concrete implementations `EmailNotification`, `SmsNotification`, and `PushNotification`, and a `NotificationFactory` with a static `createNotification(NotificationType type)` method (using an enum) that returns the appropriate implementation. Demonstrate client code that requests each type and calls `send()` without ever referencing the concrete classes directly.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a document export system supporting PDF, Word, and Excel export formats. The system should allow adding NEW export formats in the future with minimal changes to existing code, and different parts of the application (a report module, a user-data-export module) should be able to request an exporter without knowing the concrete export implementation classes."

Think about:
- Would you use a Simple Factory or the formal Factory Method (subclass-based) approach here? What factors would drive that decision?
- How would you handle a NEW export format (say, CSV) being added six months from now — what exactly needs to change, and what stays untouched?

---

## 17. Advanced LLD Scenario

**Design a Ride-Sharing Vehicle Assignment System** (like Uber) where:
- Different ride types (Economy, Premium, Pool, XL) require DIFFERENT vehicle matching logic and different `Vehicle` object configurations
- New ride types may be introduced in new markets (e.g., "Uber Moto" for motorcycle rides in certain countries) without requiring changes to the core ride-request handling flow
- The system needs to eventually construct fairly complex `Vehicle` objects with many optional attributes (capacity, amenities, fare multiplier, driver rating requirements)

Consider:
- How a `VehicleFactory` (or `RideTypeFactory`) would let the core ride-matching logic remain UNAWARE of specific vehicle/ride-type classes, depending only on a common `Vehicle` or `RideType` abstraction
- Whether, given the "fairly complex objects with many optional attributes" requirement, you'd want to combine this Factory with the Builder pattern (Topic 17) internally — using Factory to decide WHICH type of vehicle configuration to build, and Builder to actually ASSEMBLE it step by step
- How this connects back to the Strategy-based fare calculation scenario from Topic 5 — a `VehicleFactory` might also be responsible for wiring up the CORRECT `FareStrategy` for each vehicle/ride type it creates

---

## 18. Summary

**Definition:** Factory decouples client code from concrete class instantiation by centralizing the "which concrete class to create" decision in a dedicated factory method or class.

**Intent:** Let client code depend only on an abstraction, while a factory decides the concrete implementation — enabling easier extension and reducing duplicated creation logic.

**Key classes:** `Product` interface, `ConcreteProduct` implementations, `Factory` (Simple Factory: one class with a creation method; Factory Method: an abstract creator with subclasses overriding the creation step).

**Advantages:** Decouples client from concrete classes; centralizes creation logic; supports OCP (fully, in the Factory Method variant).

**Disadvantages:** Unnecessary indirection for trivially simple creation; Simple Factory variant still requires modification for new types.

**Real-world use case:** JDBC's `DriverManager.getConnection()`, SLF4J's `LoggerFactory`, Spring's bean creation machinery.

**Java example:** `ShapeFactory.createShape(ShapeType)` returning `Circle`/`Square`/`Triangle` via a centralized static method; `ShapeCreator` subclasses (`CircleCreator`/`SquareCreator`) as the formal Factory Method variant.

**Interview takeaway:** Be precise about the difference between Simple Factory (a pragmatic, commonly-used pattern, NOT formally one of the GoF 23) and the formal Factory Method pattern (subclass-based, fully OCP-compliant) — and be ready to immediately follow up with how Factory Method differs from Abstract Factory (Topic 9), since this comparison is asked in nearly every LLD interview covering creational patterns.

**One-line memory trick:** *"You ask the car factory for 'a Sedan' — you don't personally weld the chassis together; the factory decides how, you just receive the finished car."*

---

*End of Topic 8. Type "Next" to proceed to Topic 9: Abstract Factory Design Pattern.*