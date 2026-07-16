# Topic 16: Singleton Pattern

---

## 1. Introduction

**Definition:**
The Singleton Pattern is a **creational design pattern** that ensures a class has EXACTLY ONE instance throughout the application's lifetime, and provides a GLOBAL, well-defined point of access to that instance — no other code path is allowed to create a second instance of that class.

**Why it exists / what problem it solves:**
Some resources genuinely should have only ONE instance shared across the entire application — a logging system, a configuration manager, a connection pool, a cache. If MULTIPLE instances of such a class were created accidentally, you could get INCONSISTENT state (e.g., two separate loggers writing to different files, two separate configuration objects with different values loaded), wasted resources (e.g., multiple expensive database connection pools), or subtle bugs that are hard to trace.

Singleton solves this by making the constructor PRIVATE (so no other code can call `new` on it directly) and providing a single STATIC access method (commonly `getInstance()`) that returns the SAME instance every time it's called, creating it only once (lazily or eagerly).

**When it should be used:**
- When EXACTLY ONE instance of a class is required, and this constraint is a genuine, meaningful application requirement (not just a convenience)
- For shared resources that should be COORDINATED globally — e.g., a single configuration source, a single logging facility, a single thread pool
- When you need a GLOBAL access point to that instance, without passing it explicitly through every layer of the application

**When it should NOT be used:**
- When you're using Singleton merely as a convenient way to avoid passing dependencies explicitly (this is a common ANTI-PATTERN — Singleton overuse creates hidden dependencies and makes testing significantly harder)
- When the class might genuinely need MULTIPLE instances in the future (e.g., for different configurations, different environments, or parallel testing) — Singleton makes this very difficult to retrofit
- When UNIT TESTING is a priority — Singletons introduce GLOBAL, SHARED state that's notoriously difficult to isolate/reset between tests

**Advantages:**
- Guarantees exactly one instance, preventing accidental duplication of a resource that should be unique
- Provides a convenient GLOBAL access point without needing to pass the instance through every layer
- Can support LAZY initialization, deferring the cost of creating the instance until it's actually needed

**Disadvantages:**
- Introduces GLOBAL, MUTABLE shared state — a well-known source of hidden coupling between unrelated parts of a codebase
- Makes UNIT TESTING significantly harder — tests can accidentally affect each other through shared Singleton state, and MOCKING a Singleton typically requires extra tooling/effort
- Can become an OVERUSED, lazy substitute for proper Dependency Injection, leading to tightly coupled, hard-to-maintain designs
- In multi-threaded environments, naive implementations can accidentally create MULTIPLE instances if thread-safety isn't handled correctly (discussed in Section 8)

---

## 2. Real-World Analogy

Think of a **country's central government**.

A country has EXACTLY ONE official central government at any given time — you don't have TWO competing central governments simultaneously issuing conflicting laws and currencies for the SAME country. Every citizen, business, and institution refers to THE SAME central government as the single, authoritative source of law and policy — there's a single, well-known "access point" (e.g., the capital city, the official government website) that everyone goes through, rather than each region independently creating its own separate governing body.

---

## 3. Theory

**Core idea:** Make the constructor `private` so external code cannot instantiate the class directly. Provide a `public static` method (`getInstance()`) that returns the SAME instance every time — creating it on the FIRST call (lazy initialization) or at CLASS-LOADING time (eager initialization), and simply returning the already-created instance on all subsequent calls.

**Working mechanism:**
```
Client 1 → Singleton.getInstance() ──┐
Client 2 → Singleton.getInstance() ──┼──→ SAME instance returned every time
Client 3 → Singleton.getInstance() ──┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Private constructor | Prevents external code from creating new instances directly via `new` |
| Static instance field | Holds the ONE shared instance, typically as a `private static` field |
| getInstance() | The public static access method returning the shared instance |
| Lazy initialization | Creating the instance only on the FIRST call to `getInstance()` |
| Eager initialization | Creating the instance immediately when the class is LOADED, before it's even requested |

**Class responsibilities:** the Singleton class is responsible for MANAGING its own single instance's lifecycle entirely internally — no external code should ever be able to bypass this and create additional instances.

---

## 4. UML / Class Diagram

```
┌─────────────────────────────┐
│      Logger                             │
├─────────────────────────────┤
│ - instance: Logger (static)         │  (the ONE shared instance,
├─────────────────────────────┤   stored as a static field)
│ - Logger()  (private constructor)     │
│ + getInstance(): Logger (static)      │
│ + log(message: String): void          │
└─────────────────────────────┘
           △ (self-reference via
             the static 'instance' field)

┌─────────────────────┐
│      Client A                    │
├─────────────────────┤
│ Logger.getInstance().log(...)    │────┐
└─────────────────────┘        │
                                    ├──→ BOTH calls return
┌─────────────────────┐        │    the SAME Logger instance
│      Client B                    │────┘
├─────────────────────┤
│ Logger.getInstance().log(...)    │
└─────────────────────┘
```

**Relationship explanation:**
- `Logger` holds a STATIC reference to ITSELF (`instance`) — this is a special, class-level (not instance-level) self-association, unique to the Singleton pattern.
- Both `Client A` and `Client B` call the SAME static method `getInstance()` — neither client CREATES a `Logger` directly (the constructor is private, preventing this entirely); both receive references to the EXACT SAME object.
- There is NO traditional "association/aggregation" arrow between clients and `Logger` in the usual sense — clients access the Singleton through its STATIC method, not through a constructor-injected or otherwise held reference (though the RETURNED reference can, of course, be stored/held afterward).

---

## 5. Java Implementation

```java
// ============================================
// Singleton — Thread-safe, lazily-initialized Logger
// (using the "Initialization-on-demand holder" idiom,
// widely regarded as the best modern approach in Java)
// ============================================
public class Logger {

    // Private constructor — prevents external instantiation via 'new'
    private Logger() {
        System.out.println("Logger instance created.");
    }

    // Static nested "holder" class — NOT loaded into memory until
    // getInstance() is first called, giving us LAZY initialization
    // WITHOUT needing explicit synchronization on every call
    private static class LoggerHolder {
        private static final Logger INSTANCE = new Logger();
    }

    // Public static access point — always returns the SAME instance
    public static Logger getInstance() {
        // Accessing LoggerHolder.INSTANCE triggers the JVM to load
        // LoggerHolder (and thus create the Logger instance) the
        // FIRST time this method is called — the JVM's class-loading
        // mechanism GUARANTEES this is thread-safe, with no
        // explicit synchronized keyword needed
        return LoggerHolder.INSTANCE;
    }

    public void log(String message) {
        System.out.println("[LOG] " + message);
    }
}

// ============================================
// Demo
// ============================================
public class SingletonDemo {
    public static void main(String[] args) {
        Logger logger1 = Logger.getInstance();
        logger1.log("First message");

        Logger logger2 = Logger.getInstance();
        logger2.log("Second message");

        // Proving logger1 and logger2 are the SAME object
        System.out.println("Are logger1 and logger2 the same instance? "
                + (logger1 == logger2));
    }
}
```

**Key line-by-line notes:**
- The constructor is `private` — this is the FUNDAMENTAL mechanism preventing `new Logger()` from being called anywhere OUTSIDE the class itself.
- `LoggerHolder` is a `private static` NESTED class — the JVM does NOT load (and thus does NOT create `INSTANCE`) until `LoggerHolder` is FIRST referenced, which only happens inside `getInstance()` — this achieves LAZY initialization.
- The JVM's CLASS-LOADING mechanism itself guarantees that `LoggerHolder` is loaded (and `INSTANCE` created) EXACTLY ONCE, even under concurrent multi-threaded access — this is why this idiom is considered THREAD-SAFE without needing explicit `synchronized` blocks (see Section 8 for a deeper comparison with other approaches).
- `logger1 == logger2` (reference equality, not `.equals()`) evaluates to `true`, PROVING both variables point to the EXACT SAME object in memory.

---

## 6. Dry Run

**Sample input:**
```java
Logger logger1 = Logger.getInstance();
Logger logger2 = Logger.getInstance();
```

```
1. Logger.getInstance() called [first time, from logger1's assignment]
   → JVM needs to access LoggerHolder.INSTANCE
   → LoggerHolder class has NOT been loaded yet
   → JVM loads LoggerHolder NOW, which triggers:
       → new Logger() executed → private constructor runs
       → Prints: "Logger instance created."
       → INSTANCE now holds a reference to this newly created Logger
   → getInstance() returns LoggerHolder.INSTANCE
   → logger1 now references this Logger object

2. Logger.getInstance() called [second time, from logger2's assignment]
   → LoggerHolder is ALREADY loaded (from step 1)
   → JVM does NOT reload the class or recreate INSTANCE
   → getInstance() simply returns the EXISTING LoggerHolder.INSTANCE
   → logger2 now references the SAME Logger object as logger1

3. logger1 == logger2
   → Both variables hold references to the EXACT SAME heap object
   → Evaluates to true
```

**What's happening in memory:** the `Logger` object is created EXACTLY ONCE, on the heap, triggered by the JVM's class-loading mechanism the FIRST time `LoggerHolder` is accessed. EVERY subsequent call to `getInstance()`, from ANYWHERE in the application, simply returns this SAME, already-existing reference — no new `Logger` object is ever created again for the lifetime of the application.

---

## 7. Real-World Software Example

- **Logging frameworks** (Log4j, SLF4J-backed loggers): a single, shared `Logger` instance per class/application, ensuring all log output goes through a CONSISTENT, centrally-configured pipeline.
- **Configuration managers**: a single source of truth for application configuration, loaded ONCE and shared everywhere, preventing inconsistent configuration state across different parts of the application.
- **Connection pools / thread pools**: a single, shared pool managing a LIMITED set of expensive resources (database connections, threads), coordinated centrally rather than duplicated.
- **Caching layers**: a single, shared in-memory cache instance, ensuring all parts of the application read/write to the SAME cached data rather than maintaining inconsistent separate caches.
- **`java.lang.Runtime`**: Java's own `Runtime` class is itself a Singleton, accessed via `Runtime.getRuntime()`, representing the single JVM runtime environment.

---

## 8. Internal Working

**Object creation:** the KEY internal challenge is ensuring EXACTLY ONE instance is created, even under CONCURRENT multi-threaded access — several implementation approaches exist, each with different tradeoffs:

| Approach | Thread-Safe? | Lazy? | Notes |
|---|---|---|---|
| Eager initialization (`static final Logger INSTANCE = new Logger();` directly on the class) | Yes (JVM guarantees static init is thread-safe) | No (created at class load, even if never used) | Simple, but wastes resources if never actually used |
| Naive lazy (`if (instance == null) instance = new Logger();`) | **NO** — race condition possible | Yes | Two threads could BOTH see `null` and BOTH create separate instances — a real, classic bug |
| Synchronized method (`synchronized getInstance()`) | Yes | Yes | Thread-safe, but every call pays SYNCHRONIZATION overhead, even after the instance already exists |
| Double-checked locking | Yes (with `volatile`) | Yes | Avoids synchronization overhead on already-initialized calls, but subtle/error-prone to implement correctly |
| **Initialization-on-demand holder** (Section 5's approach) | **Yes** | **Yes** | Relies on JVM class-loading guarantees; no explicit synchronization needed; widely considered the CLEANEST modern approach |
| Enum-based Singleton (`enum Logger { INSTANCE; ... }`) | Yes | Depends | JVM guarantees enum instances are created exactly once; ALSO provides free protection against reflection/serialization attacks (see Common Mistakes) |

**Runtime interactions:** every call to `getInstance()` either creates the instance (first call only) or simply RETURNS the already-existing reference — from the SECOND call onward, this is essentially a cheap field access.

**Memory usage:** exactly ONE instance exists for the ENTIRE application lifetime (unless explicitly destroyed/reset, which Singleton doesn't typically support) — this is by design, but also means the instance's memory is held for as long as the application runs.

---

## 9. Before vs After

**Before (no Singleton — accidental multiple instances):**

```java
public class LoggerBad {
    // Public constructor — ANYONE can create a new instance
    public LoggerBad() {
        System.out.println("LoggerBad instance created.");
    }

    public void log(String message) {
        System.out.println("[LOG] " + message);
    }
}

// Elsewhere in the codebase, DIFFERENT parts of the application
// might EACH accidentally create their OWN LoggerBad instance:
LoggerBad logger1 = new LoggerBad(); // Module A
LoggerBad logger2 = new LoggerBad(); // Module B — a SEPARATE instance!
```

**Problems:**
- NOTHING prevents multiple modules from independently creating their OWN `LoggerBad` instances — if each instance had its OWN internal state (e.g., a buffered list of pending log entries, or its own open file handle), this could lead to INCONSISTENT behavior or resource conflicts (e.g., two file handles writing to the same log file concurrently, causing corruption).
- There's no CENTRAL, well-known access point — every part of the codebase needs its OWN reference passed around or separately constructed, with no guarantee of consistency.

**After (Singleton, as shown in Section 5):**
- The PRIVATE constructor makes it IMPOSSIBLE to accidentally create a second instance anywhere in the codebase.
- Every part of the application calls the SAME `Logger.getInstance()`, guaranteeing they all share the EXACT SAME object and its internal state.

---

## 10. SOLID Principles Connection

- **SRP tension (worth discussing honestly)**: Singleton technically bundles TWO responsibilities into one class — (1) the actual business logic the class performs (e.g., logging), AND (2) managing its OWN single-instance lifecycle (the `getInstance()` machinery). This is a well-known, often-cited critique of Singleton with respect to SRP.
- **OCP**: mostly NEUTRAL — Singleton itself doesn't directly help or hurt OCP, though its GLOBAL state can make EXTENDING behavior (e.g., subclassing) awkward, since `getInstance()` is tied to one specific class.
- **DIP tension (a genuine, important critique)**: code that calls `Logger.getInstance()` directly is depending on a CONCRETE class (and a static method) rather than an ABSTRACTION injected from outside — this is a well-known critique that Singleton, used carelessly, can violate DIP and make dependencies HARDER to see, mock, and test, compared to properly INJECTING a shared instance via constructor/dependency injection (even if that injected instance happens to be a Singleton "under the hood," managed by a framework like Spring).

---

## 11. Interview Questions

**Beginner:**
1. What is the Singleton pattern, and what real-world problem does it solve?
2. Why is the constructor made `private` in a Singleton implementation?
3. What's the difference between LAZY and EAGER initialization for a Singleton?

**Intermediate:**
4. What's WRONG with this naive lazy Singleton implementation in a multi-threaded environment?
   ```java
   public static Logger getInstance() {
       if (instance == null) {
           instance = new Logger();
       }
       return instance;
   }
   ```
   *Answer: Two threads could BOTH check `instance == null` at roughly the SAME time, BOTH see it as `null` (since neither has finished creating it yet), and BOTH proceed to create SEPARATE `Logger` instances — violating the entire premise of Singleton. This is a classic RACE CONDITION.*
5. Explain how the "Initialization-on-demand holder" idiom achieves thread safety WITHOUT explicit synchronization.
6. Why is Singleton often criticized as an ANTI-PATTERN when overused? What's the alternative for managing shared instances more cleanly?
   *Answer: Overreliance on Singleton creates GLOBAL, hidden dependencies that are hard to trace, and makes UNIT TESTING significantly harder (tests can leak state into each other via shared Singleton instances, and mocking a Singleton typically requires extra tooling). The commonly preferred alternative is proper DEPENDENCY INJECTION — letting a framework (like Spring) manage a SINGLE shared instance and INJECT it explicitly wherever needed, making dependencies visible and testable, while STILL achieving the "one shared instance" goal.*

**Advanced:**
7. How can Singleton be BROKEN using Java's REFLECTION API (calling the private constructor directly via reflection)? How would you defend against this?
   *Answer: `Constructor.setAccessible(true)` can bypass the `private` access modifier and directly invoke the constructor, creating a SECOND instance. A common defense is to THROW an exception inside the constructor if `instance` is ALREADY non-null, or to use an ENUM-based Singleton, which the JVM specifically protects against reflection-based instantiation.*
8. How can Singleton be BROKEN via Java SERIALIZATION (deserializing a Singleton instance creates a NEW object)? How would you defend against this?
   *Answer: Implementing `readResolve()` to return the EXISTING singleton instance instead of the newly deserialized one prevents this. Enum-based Singletons ALSO handle this automatically, since Java guarantees enum serialization returns the SAME enum constant.*
9. How does Singleton behave in a system with MULTIPLE CLASSLOADERS (e.g., in complex application server environments)? Why might you unexpectedly end up with MULTIPLE "singleton" instances in such environments?
10. How does Spring's DEFAULT bean scope ("singleton" scope) relate to the classic GoF Singleton pattern? Are they IDENTICAL, and if not, what's the key difference?
    *Answer: They're conceptually SIMILAR (one shared instance) but NOT identical: Spring's "singleton scope" means ONE instance PER SPRING CONTAINER (ApplicationContext), not necessarily one instance for the ENTIRE JVM — if multiple Spring containers exist, each could have its OWN "singleton" bean instance. Also, Spring-managed singletons are typically constructed via NORMAL public constructors and managed/injected by the FRAMEWORK, rather than using a private constructor + static `getInstance()` — this avoids many of the classic GoF Singleton's DIP/testability criticisms.*

---

## 12. Common Mistakes

- **Using the naive, non-thread-safe lazy initialization** shown in Interview Question 4 — a classic, easy-to-miss concurrency bug.
- **Overusing Singleton as a lazy substitute for Dependency Injection** — reaching for `getInstance()` everywhere instead of properly injecting shared dependencies creates hidden coupling and hurts testability significantly.
- **Forgetting that Singleton is vulnerable to REFLECTION and SERIALIZATION attacks** if not specifically defended against (see Interview Questions 7 and 8) — a plain private-constructor Singleton is NOT automatically bulletproof against these.
- **Not considering CLASSLOADER boundaries** in complex environments (application servers, OSGi, etc.) — assuming "only one instance ever, guaranteed" without understanding the classloader context can lead to surprising bugs.

---

## 13. Time Complexity

- **Time:** O(1) for `getInstance()` after the FIRST call (a simple field access); the FIRST call incurs whatever cost the constructor itself requires (potentially non-trivial, e.g., loading configuration files).
- **Space:** O(1) — exactly one instance's worth of memory, for the ENTIRE application lifetime.

---

## 14. Java API Examples

- **`java.lang.Runtime`**: accessed via `Runtime.getRuntime()`, representing the single JVM runtime environment — a genuine, built-in Singleton.
- **`java.awt.Desktop`**: accessed via `Desktop.getDesktop()`, representing a single, shared desktop integration point.
- **Spring Framework's default "singleton" bean scope**: the framework manages ONE instance per bean definition per `ApplicationContext`, injected wherever needed (see Interview Question 10 for the nuanced comparison with classic GoF Singleton).
- **`java.util.logging.Logger.getLogger(String name)`**: while technically returning DIFFERENT logger instances for DIFFERENT names, each NAMED logger is effectively a Singleton WITHIN its own name/hierarchy — repeated calls with the SAME name return the SAME logger instance.

---

## 15. Practice Problem

Implement a **thread-safe Singleton for an `AppConfig` class**, using the "Initialization-on-demand holder" idiom shown in Section 5. `AppConfig` should hold a `Map<String, String>` of configuration key-value pairs, loaded ONCE when the instance is first created (simulate this with a hardcoded map for the exercise), and expose a `get(String key): String` method. Demonstrate that multiple calls to `AppConfig.getInstance()` from different parts of a demo program all return the SAME configuration data.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **thread-safe connection pool Singleton** for a database connection pool. The pool should be created with a FIXED maximum size (e.g., 10 connections), and expose `acquireConnection()` and `releaseConnection(Connection conn)` methods. Ensure that the Singleton's INTERNAL pool management (tracking which connections are currently in use) is ALSO thread-safe, not just the singleton-instance creation itself."

Think about:
- The DIFFERENCE between making `getInstance()` thread-safe (ensuring only ONE pool object exists) versus making the POOL'S INTERNAL OPERATIONS thread-safe (ensuring `acquireConnection()`/`releaseConnection()` don't corrupt shared state under concurrent access) — these are TWO SEPARATE concerns that both need addressing.
- What Java concurrency utilities (e.g., `java.util.concurrent.BlockingQueue`, `synchronized` blocks, `ReentrantLock`) might help implement the pool's internal thread-safety correctly.

---

## 17. Advanced LLD Scenario

**Design a Distributed Cache Client Singleton for a Microservices Architecture**, where:
- A single application-wide Singleton `CacheClient` manages the connection to a shared distributed cache (e.g., Redis), ensuring all parts of the application reuse the SAME underlying connection pool rather than each creating their OWN redundant connections
- The system must support GRACEFUL testing — allowing unit tests to substitute a FAKE/in-memory cache implementation WITHOUT relying on the real Singleton's global state (directly addressing the well-known Singleton-vs-testability tension discussed in Section 10)
- Consider whether the "true" Singleton pattern (private constructor + static `getInstance()`) is even the BEST fit here, versus using a Dependency-Injection-managed "effectively singleton" bean (as Spring would provide) — revisit Interview Question 10's distinction

Consider:
- How you'd redesign this SPECIFICALLY to be more testable — e.g., extracting a `CacheClient` INTERFACE, with a real `RedisCacheClient` implementation managed as a Spring singleton bean (injected via constructor), versus a `FakeCacheClient` used in tests — demonstrating that the GOAL ("one shared instance in production") doesn't REQUIRE the classic GoF private-constructor pattern to achieve it
- Why this redesign directly resolves the DIP/testability critique raised in Section 10, while STILL achieving the core "single shared instance" benefit that motivated using Singleton in the first place
- How this connects to a broader, important interview point: modern, framework-based codebases (Spring, Dagger, Guice) often achieve Singleton's BENEFITS through Dependency Injection frameworks RATHER than the classic GoF implementation — recognizing this evolution is a mark of practical, real-world LLD understanding

---

## 18. Summary

**Definition:** Singleton ensures a class has exactly one instance throughout the application's lifetime, providing a global, well-defined access point to that instance.

**Intent:** Guarantee a single, shared instance for resources that must be coordinated globally (loggers, configuration, connection pools), preventing accidental duplication and inconsistent state.

**Key classes:** the Singleton class itself, with a private constructor, a static instance-holding mechanism, and a public static `getInstance()` access method.

**Advantages:** Guarantees exactly one instance; provides convenient global access; supports lazy initialization to defer creation cost.

**Disadvantages:** Introduces global mutable state; significantly complicates unit testing; often overused as a substitute for proper Dependency Injection; naive implementations can have thread-safety bugs.

**Real-world use case:** Logging frameworks, configuration managers, connection/thread pools, caching layers, `java.lang.Runtime`.

**Java example:** `Logger` using the "Initialization-on-demand holder" idiom for thread-safe, lazy singleton creation.

**Interview takeaway:** Always be ready to discuss the WELL-KNOWN critiques of Singleton (testability, hidden dependencies, DIP violation) alongside its benefits — and be ready to explain how modern Dependency-Injection frameworks (like Spring) achieve the SAME "one shared instance" goal in a more testable, less globally-coupled way. This balanced, critical perspective is a strong interview signal.

**One-line memory trick:** *"A country has exactly ONE central government — everyone refers to the SAME authoritative source, through the SAME well-known access point."*

---

*End of Topic 16. Type "Next" to proceed to Topic 17: Builder Pattern.*