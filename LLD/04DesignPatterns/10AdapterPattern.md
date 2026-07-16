# Topic 15: Adapter Pattern

---

## 1. Introduction

**Definition:**
The Adapter Pattern is a **structural design pattern** that allows objects with INCOMPATIBLE interfaces to work together, by wrapping one object (the "Adaptee") in a NEW class (the "Adapter") that TRANSLATES calls from the interface the CLIENT expects into calls the Adaptee actually understands.

**Why it exists / what problem it solves:**
Imagine you're integrating a third-party payment gateway library into your application, but its API (`processPaymentInCents(long cents)`) doesn't match the interface your application's checkout logic expects (`pay(double dollars)`). You CAN'T modify the third-party library's code (it's external/legacy/closed-source), and rewriting your ENTIRE checkout system around the third-party's specific API would tightly couple you to that ONE vendor.

Adapter solves this by introducing a small "translator" class that implements the interface YOUR code expects, internally converts the call (e.g., dollars → cents), and delegates to the Adaptee's actual method. Your client code never needs to know the Adaptee's API exists at all — it just calls the familiar interface, and the Adapter handles translation transparently.

**When it should be used:**
- When you need to use an EXISTING class, but its interface doesn't match what your code expects, and you CAN'T (or shouldn't) modify that existing class
- When integrating THIRD-PARTY libraries, legacy code, or external APIs into a system designed around a DIFFERENT interface
- When you want to make FUTURE classes with similar mismatches easy to integrate, by reusing the same adaptation approach

**When it should NOT be used:**
- When you CAN simply modify the class in question to match the expected interface directly — Adapter adds an extra layer that isn't needed if a direct fix is possible
- When the interfaces are ALREADY compatible or nearly identical — introducing an Adapter for a trivial mismatch is unnecessary overhead
- When the mismatch is so fundamental that TRANSLATION would require reimplementing significant logic rather than just reshaping the CALL — at that point, a different approach (wrapping with real added logic, i.e., Facade or a new implementation) may be more honest

**Advantages:**
- Allows integration of INCOMPATIBLE interfaces without modifying either the client code or the existing (Adaptee) class
- Follows OCP — new Adaptees can be integrated by writing new Adapters, without touching existing client code
- Keeps client code clean and decoupled from third-party/legacy API specifics

**Disadvantages:**
- Adds an extra layer of indirection — every call passes through the Adapter's translation logic
- Can accumulate complexity if MANY different Adaptees need adapting, requiring many small Adapter classes to maintain
- If the interface mismatch is deep (e.g., completely different data models, not just method signatures), the Adapter can become a substantial, non-trivial piece of translation logic itself

---

## 2. Real-World Analogy

Think of a **power plug adapter when traveling internationally**.

Your laptop charger has a plug shaped for North American outlets (Type A), but you're in Europe, where outlets are shaped differently (Type C/E/F). You don't REBUILD your laptop charger to have a European-shaped plug — instead, you use a small TRAVEL ADAPTER: one side accepts your existing North American plug (the interface you already have), and the other side fits into the European outlet (the interface the "environment" expects). The adapter doesn't generate electricity itself — it simply RESHAPES the connection so two incompatible interfaces can work together.

---

## 3. Theory

**Core idea:** Define an `Adapter` class that implements the TARGET interface (what the Client expects). Internally, the Adapter holds a reference to the `Adaptee` (the existing, incompatible class) and TRANSLATES each Target interface method call into the appropriate Adaptee method call(s) — converting parameters, combining calls, or reshaping return values as needed.

**Working mechanism:**
```
Client → Target (interface) ← Adapter → (delegates, translating) → Adaptee
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Target | The interface the CLIENT expects/already uses |
| Adaptee | The EXISTING class with an incompatible interface that needs to be integrated |
| Adapter | The class implementing Target, internally translating calls to the Adaptee |
| Client | Code that uses the Target interface, unaware an Adapter/Adaptee exists behind it |

**Class responsibilities:** the Adapter's ENTIRE responsibility is TRANSLATION — converting Target-shaped calls into Adaptee-shaped calls (and potentially Adaptee-shaped results back into Target-shaped results) — it should not contain any BUSINESS logic beyond this translation.

**Two common variants:**
- **Object Adapter** (composition-based): the Adapter HOLDS a reference to an Adaptee instance and delegates to it. This is the MORE COMMON, more flexible approach in Java (works even with `final` Adaptee classes).
- **Class Adapter** (inheritance-based): the Adapter EXTENDS the Adaptee class directly. Requires multiple inheritance of behavior, which Java doesn't support for classes (only interfaces), making this variant LESS common in Java specifically.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      PaymentProcessor           │
├─────────────────────┤
│ + pay(double dollars): void  │
└──────────┬──────────┘
           △ (implements)
           │
┌──────────┴───────────────┐
│ ThirdPartyGatewayAdapter        │
├──────────────────────────┤
│ - gateway: ThirdPartyGateway  ◇──→ (association: Adapter
├──────────────────────────┤       holds a reference to
│ + pay(double dollars)               the Adaptee, and
│   { long cents =                    DELEGATES to it after
│     Math.round(dollars * 100);      translating the call)
│     gateway.processPaymentInCents(cents); }
└──────────────────────────┘

┌─────────────────────┐
│      ThirdPartyGateway (Adaptee)│
├─────────────────────┤
│ + processPaymentInCents(long)  │
│   (EXISTING class — cannot          │
│    be modified)                        │
└─────────────────────┘

┌─────────────────────┐
│      CheckoutService (Client)  │
├─────────────────────┤
│ - processor: PaymentProcessor  ────────→ (dependency: Client
│ + checkout(double amount)          only knows the Target
└─────────────────────┘             interface, never the
                                        Adaptee directly)
```

**Relationship explanation:**
- `ThirdPartyGatewayAdapter` **implements** `PaymentProcessor` (the Target interface) — this is what makes it usable ANYWHERE a `PaymentProcessor` is expected.
- `ThirdPartyGatewayAdapter` has an **association** with `ThirdPartyGateway` (the Adaptee) — it holds a reference and DELEGATES to it, but the Adaptee class itself remains COMPLETELY untouched/unmodified.
- `CheckoutService` (the Client) depends ONLY on the `PaymentProcessor` interface — it has NO knowledge of `ThirdPartyGateway`'s existence or its cents-based API; the Adapter fully hides this mismatch.

---

## 5. Java Implementation

```java
// ============================================
// Target interface — what the CLIENT already expects
// ============================================
public interface PaymentProcessor {
    void pay(double dollars);
}

// ============================================
// Adaptee — an EXISTING third-party class with an
// INCOMPATIBLE interface (works in cents, not dollars,
// and has a completely different method name)
// ============================================
public class ThirdPartyGateway {
    // We CANNOT modify this class — imagine it's a compiled
    // JAR from an external vendor
    public void processPaymentInCents(long cents) {
        System.out.println("ThirdPartyGateway: processing " + cents + " cents");
    }
}

// ============================================
// Adapter — implements Target, translates calls to Adaptee
// (this is the OBJECT ADAPTER variant — composition-based)
// ============================================
public class ThirdPartyGatewayAdapter implements PaymentProcessor {
    private final ThirdPartyGateway gateway;

    public ThirdPartyGatewayAdapter(ThirdPartyGateway gateway) {
        this.gateway = gateway;
    }

    @Override
    public void pay(double dollars) {
        // TRANSLATION happens here: dollars -> cents,
        // and the DIFFERENT method name is bridged
        long cents = Math.round(dollars * 100);
        gateway.processPaymentInCents(cents);
    }
}

// ============================================
// Client — depends ONLY on the Target interface,
// completely unaware of ThirdPartyGateway's existence
// ============================================
public class CheckoutService {
    private final PaymentProcessor processor;

    public CheckoutService(PaymentProcessor processor) {
        this.processor = processor;
    }

    public void checkout(double amount) {
        System.out.println("Checking out $" + amount);
        // Client calls pay() in the SHAPE it expects —
        // it has no idea an Adapter/Adaptee exist behind this call
        processor.pay(amount);
    }
}

// ============================================
// Demo
// ============================================
public class AdapterDemo {
    public static void main(String[] args) {
        ThirdPartyGateway legacyGateway = new ThirdPartyGateway();
        // Wrap the incompatible Adaptee in our Adapter
        PaymentProcessor processor = new ThirdPartyGatewayAdapter(legacyGateway);

        CheckoutService checkout = new CheckoutService(processor);
        checkout.checkout(49.99);
    }
}
```

**Key line-by-line notes:**
- `ThirdPartyGateway` is left COMPLETELY unmodified — this represents a class we don't own or can't change (e.g., a third-party JAR).
- `ThirdPartyGatewayAdapter.pay()` performs the CORE translation: converting `double dollars` into `long cents` (using `Math.round()` to handle floating-point precision safely) before calling the Adaptee's differently-named method.
- `CheckoutService` never references `ThirdPartyGateway` or `ThirdPartyGatewayAdapter` by concrete type — it only knows `PaymentProcessor`, meaning a FUTURE swap to a different payment gateway (with its own Adapter) requires ZERO changes to `CheckoutService`.

---

## 6. Dry Run

**Sample input:** `checkout.checkout(49.99)`

```
1. checkout.checkout(49.99) called
   → CheckoutService.checkout() executes
   → Prints: "Checking out $49.99"
   → processor.pay(49.99) called
       → Dynamic dispatch resolves processor's ACTUAL type
         (ThirdPartyGatewayAdapter) → calls its pay() method

2. ThirdPartyGatewayAdapter.pay(49.99) executes
   → cents = Math.round(49.99 * 100) = Math.round(4999.0) = 4999
   → gateway.processPaymentInCents(4999) called
       → ThirdPartyGateway.processPaymentInCents() executes
       → Prints: "ThirdPartyGateway: processing 4999 cents"
```

**What's happening in memory:** the `CheckoutService` object holds a reference typed as `PaymentProcessor` (the Target interface), but at runtime this reference actually points to a `ThirdPartyGatewayAdapter` instance — which itself holds a reference to a `ThirdPartyGateway` instance. The TRANSLATION (dollars → cents) happens entirely within the Adapter's single method call, invisible to both the Client and (functionally) to the Adaptee, which simply receives a normal call in its OWN expected shape.

---

## 7. Real-World Software Example

- **Third-party API integrations**: wrapping an external payment gateway, shipping provider, or social media API (each with its own unique method signatures) behind a CONSISTENT internal interface your application already uses.
- **Legacy system integration**: adapting an OLD, unchangeable legacy module's interface to work with a MODERN application architecture, without rewriting the legacy code.
- **`java.util.Arrays.asList()`**: adapts a native Java array into the `List` interface, allowing array data to be used wherever a `List` is expected.
- **`InputStreamReader`/`OutputStreamWriter`**: adapt Java's BYTE-based I/O streams (`InputStream`/`OutputStream`) into CHARACTER-based `Reader`/`Writer` interfaces.
- **Spring's `HandlerAdapter`**: adapts different types of Spring MVC controllers (annotated, interface-based, etc.) into a UNIFORM interface the `DispatcherServlet` can invoke consistently.

---

## 8. Internal Working

**Object creation:** the Adapter is typically constructed by wrapping an EXISTING Adaptee instance (Object Adapter style) — the Adaptee itself is created and managed independently; the Adapter simply holds a reference to it.

**Runtime interactions / call flow:** every call on the Adapter follows the SAME shape — receive a call in the Target's shape, TRANSFORM the parameters/logic as needed, then make a SINGLE (or occasionally multiple) call(s) to the Adaptee in ITS expected shape, and optionally transform the RETURN value back into the Target's expected shape before returning.

**Memory usage:** minimal overhead — the Adapter typically holds just ONE reference to the Adaptee, plus whatever transient local variables are needed during translation (e.g., the computed `cents` value, which doesn't persist beyond the method call).

**Dynamic binding:** `CheckoutService`'s `processor.pay()` call resolves via dynamic dispatch to WHICHEVER concrete `PaymentProcessor` implementation it's holding — this could be `ThirdPartyGatewayAdapter` today, or a COMPLETELY different Adapter for a different payment provider tomorrow, with zero changes needed in `CheckoutService`.

---

## 9. Before vs After

**Before (no Adapter — client tightly coupled to incompatible API):**

```java
public class CheckoutServiceBad {
    private final ThirdPartyGateway gateway;

    public CheckoutServiceBad(ThirdPartyGateway gateway) {
        this.gateway = gateway;
    }

    public void checkout(double amount) {
        System.out.println("Checking out $" + amount);
        // Client code must know the ADAPTEE'S specific API
        // (cents, differently-named method) and perform the
        // translation ITSELF, inline, mixed in with business logic
        long cents = Math.round(amount * 100);
        gateway.processPaymentInCents(cents);
    }
}
```

**Problems:**
- `CheckoutService` is directly coupled to `ThirdPartyGateway`'s SPECIFIC API — switching payment providers later means REWRITING `CheckoutService` itself.
- The dollars-to-cents TRANSLATION logic is mixed directly into business logic (`checkout()`), rather than being cleanly separated into its own dedicated class.
- If MULTIPLE places in the codebase need to call the payment gateway, this same translation logic would need to be DUPLICATED everywhere, or extracted later (which the Adapter pattern does UPFRONT, cleanly).

**After (Adapter, as shown in Section 5):**
- `CheckoutService` depends ONLY on the clean `PaymentProcessor` interface — it has ZERO knowledge of cents, `ThirdPartyGateway`, or any vendor-specific details.
- Switching payment providers later means writing a NEW Adapter class implementing `PaymentProcessor` — `CheckoutService` needs NO changes at all.

---

## 10. SOLID Principles Connection

- **DIP**: `CheckoutService` depends only on the `PaymentProcessor` abstraction, never on `ThirdPartyGateway` directly — enabling ANY compatible Adapter to be substituted freely.
- **OCP**: integrating a NEW third-party gateway means writing a NEW Adapter class — no existing code (`CheckoutService`, other Adapters) needs modification.
- **SRP**: the Adapter's ONLY responsibility is translation between two interfaces — it doesn't mix in business logic or duplicate the Adaptee's actual payment-processing behavior.
- **LSP**: `ThirdPartyGatewayAdapter` can be substituted anywhere a `PaymentProcessor` is expected, and `CheckoutService`'s behavior remains entirely correct — this substitutability underlies the whole pattern.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Adapter pattern solve, in your own words?
2. Why can't you simply MODIFY the Adaptee class directly in most real-world scenarios where Adapter is used?
3. What's the difference between "Target," "Adaptee," and "Adapter" in this pattern?

**Intermediate:**
4. What's the difference between an OBJECT Adapter (composition-based) and a CLASS Adapter (inheritance-based)? Why is the Object Adapter more common in Java specifically?
   *Answer: Class Adapter would require the Adapter to EXTEND the Adaptee class directly, inheriting its implementation — but Java doesn't support multiple inheritance of CLASSES (only interfaces), which severely limits this approach if the Adapter also needs to implement the Target interface AND extend something else, or if the Adaptee is `final`. Object Adapter, using composition (holding a reference to the Adaptee), avoids this limitation entirely and is far more flexible — hence its dominance in Java codebases.*
5. Give an example from the JDK itself where Adapter is used (e.g., `Arrays.asList()`, `InputStreamReader`).
6. How would you handle an Adaptee whose method THROWS a checked exception that the Target interface's method signature doesn't declare?

**Advanced:**
7. **Adapter vs Facade** — precisely distinguish them (a very common interview follow-up, since both "wrap" other code).
   *Answer: Adapter's INTENT is to make an EXISTING, INCOMPATIBLE interface match what the client ALREADY expects — it's about interface TRANSLATION for ONE (usually pre-existing) class. Facade's INTENT is to provide a SIMPLIFIED, higher-level interface over a COMPLEX SUBSYSTEM (often involving MULTIPLE classes), REDUCING complexity rather than translating a mismatch. An Adapter typically wraps ONE Adaptee to fix a compatibility problem; a Facade typically wraps MANY subsystem classes to fix a COMPLEXITY problem. They can even be used TOGETHER (a Facade might internally use several Adapters).*
8. **Adapter vs Bridge** — precisely distinguish them (both involve delegation to another object, and both were/are covered in this series — Bridge in Topic 19).
   *Answer: Adapter is typically applied AFTER the fact, retrofitting compatibility between an EXISTING incompatible class and a client's expected interface — it's fundamentally a FIX for a mismatch that already exists. Bridge is a DESIGN-TIME decision, made UPFRONT, deliberately SEPARATING an abstraction from its implementation so BOTH can vary independently — it's not fixing a pre-existing incompatibility, but proactively preventing a class explosion from combining multiple dimensions of variation.*
9. How would you design an Adapter to work with MULTIPLE different Adaptees that all need to be exposed through the SAME Target interface (e.g., supporting Stripe, PayPal, AND a legacy gateway, all as `PaymentProcessor`s)?
10. Could Adapter be combined with Factory (Topic 8) to decide WHICH concrete Adapter to instantiate based on configuration? Describe how you'd design this.

---

## 12. Common Mistakes

- **Putting BUSINESS LOGIC inside the Adapter**, beyond pure translation — an Adapter should be a thin translation layer; if it starts making DECISIONS beyond reshaping calls/data, that logic likely belongs elsewhere (e.g., in the Client or a separate service).
- **Confusing Adapter with Facade** — remember: Adapter fixes an INCOMPATIBILITY for typically ONE class; Facade simplifies COMPLEXITY across typically MANY classes (see Interview Question 7).
- **Forgetting to handle EDGE CASES in translation** — e.g., the dollars-to-cents conversion needs `Math.round()` to handle floating-point precision correctly; naive truncation (`(long) (dollars * 100)`) could introduce OFF-BY-ONE-CENT bugs due to floating-point representation issues.
- **Creating an Adapter when you actually COULD have modified the class directly** — if you OWN the code and there's no real reason not to change it, a direct fix is simpler than introducing an unnecessary layer of indirection.

---

## 13. Time Complexity

Not meaningfully different from a normal method call — O(1) additional overhead for the Adapter's translation logic (typically simple arithmetic/reshaping) before delegating to the Adaptee. The pattern's value is structural/integration-focused, not algorithmic.

---

## 14. Java API Examples

- **`java.util.Arrays.asList(T... array)`**: adapts a native array into the `List<T>` interface, allowing array-based data to be used with List-based APIs.
- **`java.io.InputStreamReader` / `OutputStreamWriter`**: adapt byte-oriented streams (`InputStream`/`OutputStream`) into character-oriented `Reader`/`Writer` interfaces, handling character encoding translation along the way.
- **`java.util.Collections.list(Enumeration<T>)`**: adapts the LEGACY `Enumeration` interface into a modern `ArrayList`.
- **Spring's `HandlerAdapter` interface**: adapts VARIOUS types of Spring MVC controllers into a consistent invocation interface that `DispatcherServlet` can call uniformly, regardless of the controller's own underlying style.

---

## 15. Practice Problem

Implement an **Adapter for a legacy temperature sensor**: your application expects a `TemperatureSensor` interface with a `getTemperatureCelsius(): double` method, but the legacy `LegacyFahrenheitSensor` class only provides `readFahrenheit(): double`. Write a `FahrenheitToCelsiusAdapter` that implements `TemperatureSensor`, wraps a `LegacyFahrenheitSensor`, and correctly converts Fahrenheit to Celsius internally. Demonstrate client code using the Adapter without any awareness of the underlying Fahrenheit sensor.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design an **adapter layer for multiple notification providers**: your application has a `NotificationSender` interface with `send(String recipient, String message)`. You need to integrate THREE different third-party SMS/email providers, each with COMPLETELY different APIs (different method names, different parameter orders, one requiring a separate 'connect' step before sending, one returning a status code you need to interpret). Design Adapters for all three, ensuring your application's notification-sending logic never needs to know which underlying provider is actually being used."

Think about:
- How you'd handle the provider that requires a SEPARATE connection/initialization step — does this logic belong INSIDE the Adapter's `send()` method (connecting on each call), or should the Adapter manage connection state across multiple calls?
- How you'd handle the provider returning a STATUS CODE that needs interpreting — should the `NotificationSender` interface's `send()` method return something (e.g., a boolean or enum), and how would each Adapter translate its OWN provider's status representation into that common return type?

---

## 17. Advanced LLD Scenario

**Design a Multi-Cloud Storage Abstraction Layer** using Adapter, where:
- Your application defines a clean `CloudStorage` interface with methods like `uploadFile(String path, byte[] data)`, `downloadFile(String path): byte[]`, and `deleteFile(String path)`
- You need to integrate with THREE different cloud providers' SDKs (AWS S3, Google Cloud Storage, Azure Blob Storage), each with WILDLY different APIs, authentication mechanisms, and even different CONCEPTUAL models (e.g., S3 uses "buckets and keys," Azure uses "containers and blobs" — similar concepts, different terminology and parameter shapes)
- The system should allow SWITCHING cloud providers via configuration, without any changes to application code that uses `CloudStorage`

Consider:
- How each provider's SDK-specific authentication/client setup should be encapsulated WITHIN its respective Adapter (e.g., `S3StorageAdapter` internally manages an `AmazonS3` client instance), so the `CloudStorage` interface itself stays completely free of provider-specific concerns
- How this scenario connects back to the "Adapter vs Facade" distinction (Section 11) — since each cloud SDK is itself a fairly COMPLEX subsystem, is each Adapter here ALSO functioning somewhat like a Facade (simplifying that SDK's complexity) in addition to translating its interface? Discuss why these two patterns often blend together naturally in real-world integration code
- How you'd test application code that depends on `CloudStorage` WITHOUT actually hitting any real cloud provider — write (conceptually) a `MockCloudStorageAdapter` or similar test double, and discuss how the Adapter pattern's clean interface separation makes this straightforward

---

## 18. Summary

**Definition:** Adapter allows objects with incompatible interfaces to work together by wrapping one object in a class that translates calls between the interface the client expects and the interface the existing object actually provides.

**Intent:** Enable INTEGRATION of an existing, incompatible class into a system designed around a different interface, without modifying either the client or the existing class.

**Key classes:** `Target` (interface the client expects), `Adaptee` (existing, incompatible class), `Adapter` (implements Target, translates calls to Adaptee).

**Advantages:** Enables integration without modifying either side; good OCP for adding new adaptees; keeps client code clean and decoupled from third-party specifics.

**Disadvantages:** Adds a layer of indirection; can accumulate many small Adapter classes; deep interface mismatches can make the Adapter itself non-trivial.

**Real-world use case:** Third-party payment gateway integration, legacy system integration, `Arrays.asList()`, `InputStreamReader`, multi-cloud storage abstraction layers.

**Java example:** `ThirdPartyGatewayAdapter` translating a `pay(double dollars)` call into the Adaptee's `processPaymentInCents(long cents)` call.

**Interview takeaway:** Be ready to clearly distinguish Adapter (fixing an interface MISMATCH for typically one class) from Facade (simplifying COMPLEXITY across typically many classes) and from Bridge (a proactive, upfront design decision rather than a retrofit) — these three "wrapping" patterns are frequently confused, and clearly separating their INTENTS is a strong interview signal.

**One-line memory trick:** *"A travel plug adapter doesn't generate electricity — it just reshapes the connection so two incompatible plugs can work together."*

---

*End of Topic 15. Type "Next" to proceed to Topic 16: Singleton Pattern.*