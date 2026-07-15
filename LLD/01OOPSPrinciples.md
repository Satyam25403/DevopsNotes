# Topic 2: OOP Principles

---

## 1. Introduction

**Definition:**
Object-Oriented Programming (OOP) is a programming paradigm built around the concept of **objects** — bundles of data (attributes) and behavior (methods) — rather than around functions and logic acting on separate data. OOP rests on **four pillars**:

| Pillar | One-line definition |
|---|---|
| **Encapsulation** | Bundling data and behavior together, hiding internal state behind controlled access |
| **Abstraction** | Exposing only essential details, hiding unnecessary implementation complexity |
| **Inheritance** | A class acquiring properties/behavior from another class, forming an "is-a" relationship |
| **Polymorphism** | The same interface/method call behaving differently depending on the actual object type |

**Why OOP exists / what problem it solves:**
Before OOP became mainstream, procedural programming used global data and functions that operated on it — as systems grew, this led to:
- Data being modified from anywhere, with no protection or control
- Code duplication, since similar entities had no way to share common structure
- Functions tightly bound to specific data shapes, making changes ripple everywhere
- No natural way to model real-world entities (a `Car`, a `User`, an `Order`) as cohesive units

OOP solves this by modeling software around real-world-like entities that own their own data and expose controlled behavior — directly enabling the modularity, extensibility, and testability goals discussed in Topic 1.

**When it should be used:**
- Virtually all non-trivial applications benefit from OOP's structuring — it's the default paradigm for Java, and for good reason
- Especially valuable when modeling domains with natural hierarchies/variations (different payment types, different vehicle types, different notification channels)

**When it should NOT be over-applied:**
- Not every problem needs a deep class hierarchy — simple utility/stateless logic is sometimes cleaner as plain functions (Java's `Math` class, or static utility methods)
- Overusing inheritance where composition would be simpler is a very common real mistake (covered in Common Mistakes)

**Advantages:**
- Models real-world entities intuitively
- Encourages code reuse (inheritance, composition)
- Encourages loose coupling and modularity (encapsulation, abstraction)
- Enables flexible, extensible code (polymorphism)

**Disadvantages:**
- Can lead to excessive class hierarchies if misused
- Inheritance misuse creates tight coupling between parent/child classes
- Steeper learning curve for beginners compared to straightforward procedural code

---

## 2. Real-World Analogy

Think of a **Car**:

- **Encapsulation**: The engine, fuel system, and wiring are hidden under the hood. You interact with the car through a steering wheel, pedals, and buttons — you don't need to know HOW the engine converts fuel into motion; you just press the accelerator. The car's internals are protected from you accidentally breaking them.
- **Abstraction**: When you press the brake pedal, you don't think about hydraulic brake fluid, calipers, and rotors — you just think "stop the car." The complexity is hidden behind a simple interface (the pedal).
- **Inheritance**: A `SportsCar` and a `SedanCar` are both fundamentally "a Car" — they share common properties (wheels, engine, steering) but each adds its own specific characteristics (a sports car might have a turbo boost feature).
- **Polymorphism**: Pressing the "start" button behaves differently depending on the car — a petrol car cranks an engine, an electric car silently powers up its motor. Same action (the button), different behavior depending on the actual car type.

---

## 3. Theory

### Encapsulation

**Core idea:** Keep an object's internal state `private`, and expose only necessary operations via `public` methods (getters/setters, or better, meaningful behavior methods).

```
Working mechanism:
┌─────────────────────────────┐
│  class BankAccount               │
│  ┌───────────────────────┐  │
│  │ private double balance;      │  │  ← hidden, cannot be
│  └───────────────────────┘  │     accessed directly from outside
│  ┌───────────────────────┐  │
│  │ public void deposit(amt)      │  │  ← controlled access point;
│  │ public void withdraw(amt)     │  │     can enforce rules
│  │ public double getBalance()     │  │     (e.g. "balance can't go negative")
│  └───────────────────────┘  │
└─────────────────────────────┘
```

**Important terminology:** access modifiers (`private`, `protected`, `public`, package-private/default), getters/setters, invariants (rules a class enforces about its own valid state).

### Abstraction

**Core idea:** Define WHAT an object can do (the contract) without exposing HOW it does it. In Java, achieved via `interface` and `abstract class`.

```
Working mechanism:
┌─────────────────────┐
│  <<interface>>            │
│  Shape                       │
│  + calculateArea(): double     │  ← WHAT (the contract)
└──────────┬──────────┘
           │
┌──────────┴──────────┐
│  Circle                     │
│  calculateArea() {              │  ← HOW (the implementation,
│    return PI * radius * radius; │     hidden from the caller)
│  }                             │
└─────────────────────┘
```

**Important terminology:** interface, abstract class, abstract method, concrete method.

### Inheritance

**Core idea:** A class (subclass/child) can acquire fields and methods from another class (superclass/parent), modeling an "is-a" relationship, and can override or extend behavior.

```
Working mechanism:
┌─────────────────┐
│  class Vehicle          │
│  protected String brand;   │
│  void move() {...}            │
└────────┬────────┘
         △  extends
┌────────┴────────┐
│  class Car               │
│  (inherits brand, move())   │
│  + int numDoors;                │  ← adds its own new field
└─────────────────┘
```

**Important terminology:** superclass/parent, subclass/child, `extends`, `super`, method overriding, "is-a" vs "has-a" relationship.

### Polymorphism

**Core idea:** The SAME method call produces DIFFERENT behavior depending on the actual runtime type of the object. Two forms:
- **Compile-time (static) polymorphism**: method overloading — same method name, different parameter signatures, resolved at compile time.
- **Runtime (dynamic) polymorphism**: method overriding — a subclass provides its own implementation of a parent's method, resolved at RUNTIME based on the actual object type.

```
Working mechanism (runtime polymorphism):
Shape shape = new Circle();   // reference type: Shape, actual type: Circle
shape.calculateArea();         // JVM looks at the ACTUAL object (Circle)
                                 // at RUNTIME to decide which calculateArea()
                                 // implementation to call — NOT the reference
                                 // type (Shape)
```

**Important terminology:** overloading, overriding, dynamic dispatch, virtual method table (vtable) — covered in depth in Section 8.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Shape                    │
├─────────────────────┤
│ + calculateArea(): double        │
│ + calculatePerimeter(): double      │
└──────────┬──────────┘
           △  (implements)
   ┌───────┼───────┐
   │               │
┌──┴────────┐  ┌──┴────────┐
│  Circle           │  │  Rectangle           │
├──────────┤  ├──────────┤
│ - radius: double     │  │ - width: double        │
│                        │  │ - height: double          │
├──────────┤  ├──────────┤
│ + calculateArea()      │  │ + calculateArea()          │
│ + calculatePerimeter()   │  │ + calculatePerimeter()        │
└──────────┘  └──────────┘
```

**Relationship explanation:**
- `Circle` and `Rectangle` **implement** the `Shape` interface (hollow triangle, dashed line) — this is **Abstraction**: both promise to provide `calculateArea()`, but each does so differently — this differing implementation IS **Polymorphism** in action.
- `radius`, `width`, `height` are `private` (`-` in UML notation) — this is **Encapsulation**.
- If `Circle` and `Rectangle` shared a common `AbstractShape` base class providing shared logic (e.g., a `describe()` method calling `calculateArea()`), that would demonstrate **Inheritance**.

---

## 5. Java Implementation

```java
// ============================================
// ENCAPSULATION: BankAccount hides its balance,
// exposes only controlled operations
// ============================================
public class BankAccount {
    private double balance; // private: cannot be accessed directly from outside

    public BankAccount(double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.balance = initialBalance;
    }

    // Controlled access point — enforces the rule "cannot withdraw more than balance"
    public void withdraw(double amount) {
        if (amount > balance) {
            throw new IllegalStateException("Insufficient funds");
        }
        balance -= amount;
    }

    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit amount must be positive");
        }
        balance += amount;
    }

    // Read-only access — no setter for balance, since it should only
    // change via deposit()/withdraw(), never set directly
    public double getBalance() {
        return balance;
    }
}


// ============================================
// ABSTRACTION: Shape defines WHAT every shape must
// be able to do, without saying HOW
// ============================================
public interface Shape {
    double calculateArea();       // contract only — no implementation here
    double calculatePerimeter();
}


// ============================================
// INHERITANCE + POLYMORPHISM:
// Circle and Rectangle each implement Shape
// differently — same method NAME, different BEHAVIOR
// ============================================
public class Circle implements Shape {
    private final double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }

    @Override
    public double calculatePerimeter() {
        return 2 * Math.PI * radius;
    }
}

public class Rectangle implements Shape {
    private final double width;
    private final double height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return width * height;
    }

    @Override
    public double calculatePerimeter() {
        return 2 * (width + height);
    }
}


// ============================================
// A base class demonstrating classic INHERITANCE
// (extends, not implements) with a shared, reusable method
// ============================================
public abstract class AbstractShape implements Shape {
    private final String name;

    protected AbstractShape(String name) { // protected: accessible to subclasses
        this.name = name;
    }

    // A CONCRETE method, shared by ALL subclasses — no need to
    // reimplement this logic in Circle/Rectangle
    public void describe() {
        System.out.printf("%s -> Area: %.2f, Perimeter: %.2f%n",
                name, calculateArea(), calculatePerimeter());
        // Note: calculateArea() here calls the OVERRIDDEN version
        // in whichever subclass this actually is — this is
        // polymorphism combined with inheritance
    }
}

public class CircleV2 extends AbstractShape {
    private final double radius;

    public CircleV2(double radius) {
        super("Circle"); // calling the parent constructor
        this.radius = radius;
    }

    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }

    @Override
    public double calculatePerimeter() {
        return 2 * Math.PI * radius;
    }
}


// ============================================
// Demonstrating POLYMORPHISM explicitly
// ============================================
public class ShapeDemo {
    public static void main(String[] args) {
        // COMPILE-TIME polymorphism (overloading) example:
        System.out.println(sum(2, 3));         // calls int version
        System.out.println(sum(2.5, 3.5));       // calls double version

        // RUNTIME polymorphism (overriding) example:
        Shape[] shapes = { new Circle(5), new Rectangle(4, 6) };
        for (Shape shape : shapes) {
            // The reference type is "Shape" for BOTH,
            // but the ACTUAL method invoked depends on the
            // real object type at runtime — dynamic dispatch
            System.out.println("Area: " + shape.calculateArea());
        }
    }

    // Overloaded methods — same name, different parameter types (compile-time)
    static int sum(int a, int b) { return a + b; }
    static double sum(double a, double b) { return a + b; }
}
```

**Key line-by-line notes:**
- `private double balance` — encapsulation: nothing outside `BankAccount` can touch `balance` directly.
- `public interface Shape` — abstraction: defines a contract, zero implementation.
- `implements Shape` — both `Circle` and `Rectangle` fulfill the SAME contract differently — this is the foundation of polymorphism.
- `@Override` — signals to both the compiler and future readers that this method is intentionally providing a subclass-specific implementation of a parent/interface method.
- `abstract class AbstractShape` — provides a SHARED concrete method (`describe()`) while still leaving `calculateArea()`/`calculatePerimeter()` abstract — a mix of inheritance (shared code) and abstraction (unimplemented contract).
- `super("Circle")` — explicitly invokes the parent class's constructor, a core mechanic of inheritance in Java.

---

## 6. Dry Run

**Sample input:** Running `ShapeDemo.main()` with `shapes = { new Circle(5), new Rectangle(4, 6) }`.

```
1. sum(2, 3) called
   → Compiler sees two INT arguments at COMPILE TIME
   → Resolves to sum(int, int) → returns 5

2. sum(2.5, 3.5) called
   → Compiler sees two DOUBLE arguments at COMPILE TIME
   → Resolves to sum(double, double) → returns 6.0
   (Both of the above are STATIC/compile-time polymorphism —
    decided before the program even runs)

3. new Circle(5) created
   → Heap allocates a Circle object, radius = 5
   → shapes[0] holds a reference typed as "Shape" but
     POINTING to an actual Circle object

4. new Rectangle(4, 6) created
   → Heap allocates a Rectangle object, width=4, height=6
   → shapes[1] holds a reference typed as "Shape" but
     POINTING to an actual Rectangle object

5. Loop iteration 1: shape = shapes[0] (reference type Shape,
   actual type Circle)
   → shape.calculateArea() called
   → JVM checks: what is the ACTUAL type of the object
     this reference points to? → Circle
   → JVM invokes Circle's calculateArea() → Math.PI * 5 * 5 = 78.54
   → prints "Area: 78.54"

6. Loop iteration 2: shape = shapes[1] (reference type Shape,
   actual type Rectangle)
   → shape.calculateArea() called
   → JVM checks actual type → Rectangle
   → JVM invokes Rectangle's calculateArea() → 4 * 6 = 24.0
   → prints "Area: 24.0"
```

**What's happening in memory:** this is **dynamic dispatch** — even though the compiler only knows `shape` is "some kind of Shape," the JVM resolves the ACTUAL method to call at runtime by checking the object's real type, via a mechanism called the **virtual method table** (detailed in Section 8). This is precisely why polymorphism works — the same line of code (`shape.calculateArea()`) produces different behavior depending on which object is actually in memory at that moment.

---

## 7. Real-World Software Example

- **Encapsulation**: Java's `ArrayList` hides its internal array and resizing logic — you only interact via `add()`, `get()`, `remove()`, never touching the internal array directly.
- **Abstraction**: JDBC's `Connection` interface — you call `connection.createStatement()` without knowing or caring whether you're talking to MySQL, PostgreSQL, or Oracle underneath.
- **Inheritance**: Android's `Activity` classes all extend a base `Activity` class, inheriting lifecycle methods (`onCreate`, `onResume`) while adding their own screen-specific logic.
- **Polymorphism**: Spring's `@Service` classes implementing a common interface (e.g., `PaymentService`) — the Spring container injects whichever concrete implementation is configured, and calling code doesn't change regardless of which one is actually wired in.

---

## 8. Internal Working

**Encapsulation:** enforced entirely at COMPILE TIME by the Java compiler checking access modifiers — there's no runtime overhead; attempting to access a `private` field from outside the class is a compilation error, not a runtime check.

**Inheritance memory layout:** when a subclass object is created, the JVM allocates memory for BOTH the subclass's own fields AND the fields it inherited from every superclass, laid out contiguously. A `Circle extends AbstractShape` object contains the `name` field's memory (from `AbstractShape`) plus `radius`'s memory (from `Circle`) in the same object.

**Polymorphism / dynamic dispatch — the vtable mechanism:**
```
Every class with overridable methods has an associated
"virtual method table" (vtable) — essentially an array
of method pointers.

Circle's vtable:                Rectangle's vtable:
┌─────────────────────┐        ┌─────────────────────┐
│ calculateArea → Circle's    │        │ calculateArea → Rectangle's  │
│               implementation │        │               implementation   │
│ calculatePerimeter → Circle's │        │ calculatePerimeter → Rectangle's│
└─────────────────────┘        └─────────────────────┘

When you call shape.calculateArea():
1. JVM looks at the OBJECT HEADER (metadata stored with
   every object) to find its ACTUAL class (Circle or Rectangle)
2. JVM looks up that class's vtable
3. JVM calls whichever method pointer is stored at the
   "calculateArea" slot in THAT specific vtable

This lookup happens at RUNTIME (hence "dynamic dispatch"),
and is why the SAME line of code can invoke DIFFERENT
actual method implementations depending on the real object type.
```

**Overloading (compile-time polymorphism) internal working:** the compiler determines which overloaded method to call based purely on the argument types visible at compile time — there's no runtime lookup at all; the correct method is baked into the bytecode directly.

---

## 9. Before vs After

**Before (no OOP — procedural style with global-like data):**

```java
public class ShapeCalculator {
    // Using a "type" string and raw fields instead of proper objects
    public static double calculateArea(String type, double dim1, double dim2) {
        if (type.equals("CIRCLE")) {
            return Math.PI * dim1 * dim1; // dim2 unused, confusing
        } else if (type.equals("RECTANGLE")) {
            return dim1 * dim2;
        }
        throw new IllegalArgumentException("Unknown shape type");
    }
}
```

**Problems:**
- No encapsulation — `dim1`/`dim2` mean different things depending on shape type, easy to misuse.
- Adding a new shape (Triangle) means editing this method directly — no Open/Closed Principle.
- String-based type checking instead of true polymorphism — exactly the anti-pattern flagged in Topic 1.
- No abstraction — callers must know the internal "type string" convention.

**After (proper OOP, as shown in Section 5):**
- Each shape is its own class, encapsulating its own specific data (`radius` vs `width`/`height`) — no more confusing, overloaded parameter meanings.
- Adding a new shape (`Triangle implements Shape`) requires ZERO changes to existing code — genuinely open for extension.
- `calculateArea()` behaves polymorphically — calling code doesn't need `if/else` type-checking at all.

---

## 10. SOLID Principles Connection

- **SRP**: Each shape class is responsible only for its own area/perimeter calculation — `Circle` doesn't know or care about `Rectangle`'s logic.
- **OCP**: New shapes can be added by creating new classes implementing `Shape`, without modifying `Circle`, `Rectangle`, or any calling code.
- **LSP**: Any `Shape` implementation can be substituted wherever a `Shape` is expected — `ShapeDemo`'s loop works identically regardless of which concrete shape is in the array.
- **ISP**: `Shape` has exactly the methods every shape genuinely needs — no shape is forced to implement irrelevant methods.
- **DIP**: Code that uses shapes (like `ShapeDemo`) depends on the `Shape` abstraction, never on concrete classes like `Circle` directly.

*(SOLID gets its own dedicated deep-dive in Topics 3-4 — this is a preview showing how OOP's four pillars are the foundation SOLID builds upon.)*

---

## 11. Interview Questions

**Beginner:**
1. What are the four pillars of OOP? Give a one-line definition of each.
2. What is the difference between an abstract class and an interface in Java?
   *Answer: An abstract class can have both concrete and abstract methods, constructors, and instance fields; an interface (pre-Java 8) could only declare method signatures. Since Java 8+, interfaces can have `default` and `static` methods too, but still cannot have constructors or instance state (only constants).*
3. What is method overloading, and how is it different from overriding?

**Intermediate:**
4. Explain dynamic method dispatch with an example.
5. Can a `private` method be overridden in Java? Why or why not?
   *Answer: No — private methods aren't inherited/visible to subclasses at all, so there's nothing to override; a same-named private method in a subclass is a completely separate, unrelated method.*
6. What is the difference between "is-a" and "has-a" relationships, and which OOP pillar does each relate to?
7. Why does Java not support multiple inheritance of classes, but does support it for interfaces?
   *Answer: Multiple class inheritance creates ambiguity when two parent classes have conflicting field/method implementations (the "diamond problem"). Interfaces (pre-default-methods) had no implementation to conflict, so multiple interface inheritance was safe; default methods introduced some resolution rules but still avoid the state-conflict issue classes would have.*

**Advanced:**
8. How does the JVM implement runtime polymorphism internally? (Expected: vtable/dynamic dispatch mechanism, as in Section 8.)
9. Why is "favor composition over inheritance" a widely recommended principle? When would you still choose inheritance?
10. Explain how encapsulation supports the Open/Closed Principle.
11. What's the difference between overriding a method and hiding a static method in Java?
    *Answer: Static methods are resolved at compile time based on the REFERENCE type, not the actual object type — "hiding" a static method doesn't participate in dynamic dispatch, unlike instance method overriding.*
12. Can constructors be inherited in Java? Why or why not?

---

## 12. Common Mistakes

- **Overusing inheritance instead of composition** — creating deep, fragile class hierarchies (`Animal → Bird → Penguin` breaks when `Penguin.fly()` is called, since penguins can't fly) when a more flexible composition-based design (a `FlyBehavior` interface, previewing the Strategy pattern) would be more appropriate.
- **Forgetting `@Override`** — without it, a typo in a method signature silently creates a NEW overloaded method instead of overriding the intended one, and the compiler won't catch it. Always use `@Override` to let the compiler verify your intent.
- **Public fields instead of encapsulated access** — exposing mutable public fields directly defeats encapsulation entirely, allowing any external code to put the object into an invalid state.
- **Confusing abstraction with encapsulation** — they're related but distinct: encapsulation is about HIDING data/state; abstraction is about HIDING complexity/implementation details behind a simpler interface.
- **Breaking Liskov Substitution via inheritance misuse** — making a subclass that can't honestly fulfill its parent's contract (the classic `Square extends Rectangle` problem, covered in depth in Topic 4).

---

## 13. Time Complexity

Not directly applicable — OOP pillars are structural/design concepts, not algorithms. However, it's worth noting: **dynamic dispatch (polymorphism) has a small, constant-time (O(1)) overhead** compared to a direct static method call, due to the vtable lookup — negligible in virtually all real applications, but worth knowing it's not literally "free."

---

## 14. Java API Examples

- **Encapsulation**: `java.util.Date` (older API) hides its internal long timestamp representation behind methods like `getTime()`.
- **Abstraction**: `java.sql.Connection`, `java.sql.Statement` — interfaces implemented differently per database vendor's driver.
- **Inheritance**: `java.util.AbstractList` provides a base implementation that `ArrayList` and others build upon.
- **Polymorphism**: `java.util.Comparator` — every collection sorting method accepts ANY `Comparator` implementation polymorphically; `Collections.sort(list, comparator)` doesn't care which specific comparator is passed.

---

## 15. Practice Problem

Design a small class hierarchy for `Employee` types: a base concept of an Employee, with `FullTimeEmployee` (fixed monthly salary) and `ContractEmployee` (hourly rate × hours worked) both needing a `calculateMonthlySalary()` method.

- Decide: should this use an interface, an abstract class, or both? Justify your choice.
- Write the class(es) demonstrating all four OOP pillars somewhere in your design.
- Don't worry about a "perfect" answer — focus on correctly applying encapsulation (private fields), abstraction (a contract for `calculateMonthlySalary`), inheritance (shared behavior if any), and polymorphism (a list of `Employee` calling `calculateMonthlySalary()` uniformly).

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a simple library system's book-lending mechanism (without worrying about the whole library yet) — you have `Book` objects, and different types of members (`StudentMember`, `FacultyMember`) who have different borrowing limits and loan durations. Model this using proper OOP principles."

Think about:
- What's common across all member types (shared fields/behavior) vs what varies (borrowing limits, loan duration)?
- Where would abstraction (interface or abstract class) fit naturally?
- How would polymorphism let a `Library` class handle any member type uniformly?

---

## 17. Advanced LLD Scenario

**Design a Notification System** that can send notifications via Email, SMS, and Push Notification, where:
- New notification channels may be added in the future (e.g., Slack, WhatsApp)
- Each channel has different configuration needs (email needs an SMTP server, SMS needs a phone gateway)
- The system calling code should be able to send a notification without knowing which specific channel implementation is being used

Consider how the four OOP pillars apply:
- **Abstraction**: a `NotificationChannel` interface defining `send(message)`
- **Encapsulation**: each channel class hides its own configuration details (SMTP credentials, gateway API keys)
- **Inheritance**: if channels share common logic (e.g., logging, retry behavior), a shared abstract base class could hold it
- **Polymorphism**: the calling code (e.g., a `NotificationService`) holds a `List<NotificationChannel>` and calls `send()` on each, uniformly, regardless of actual channel type

*(This scenario is effectively the seed of the Strategy pattern and Factory pattern, which we'll cover formally in upcoming topics — notice how naturally OOP principles alone lead toward these patterns.)*

---

## 18. Summary

**Definition:** OOP is a paradigm organizing software around objects that bundle data and behavior, built on four pillars: Encapsulation, Abstraction, Inheritance, and Polymorphism.

**Intent:** Model real-world entities intuitively, enabling modular, reusable, extensible, and maintainable code.

**Key classes:** N/A (foundational paradigm — applies universally across every future topic).

**Advantages:** Encourages reuse, modularity, loose coupling, and flexible extensibility.

**Disadvantages:** Can lead to over-engineered hierarchies if inheritance is overused instead of composition.

**Real-world use case:** Nearly all Java frameworks — Spring's dependency-injected interfaces, JDBC's vendor-agnostic `Connection`, the Collections Framework's `List`/`Map` abstractions.

**Java example:** `Shape` interface with `Circle`/`Rectangle` implementations — demonstrating all four pillars together in one small example.

**Interview takeaway:** Be ready to define all four pillars precisely, explain dynamic dispatch mechanically (vtable lookup), and recognize when inheritance is being misused where composition would be more appropriate — this last point especially impresses interviewers.

**One-line memory trick:** *"A car: hood hides the engine (Encapsulation), the pedal hides the mechanism (Abstraction), a SportsCar IS-A Car (Inheritance), and pressing 'start' does different things in different cars (Polymorphism)."*

---

*End of Topic 2. Type "Next" to proceed to Topic 3: SOLID Principles.*