# Topic 20: Facade Pattern

---

## 1. Introduction

**Definition:**
The Facade Pattern is a **structural design pattern** that provides a SIMPLIFIED, HIGHER-LEVEL interface to a COMPLEX SUBSYSTEM consisting of MANY interacting classes — hiding the subsystem's internal complexity behind ONE clean, easy-to-use entry point, without removing the ability to access the subsystem's classes DIRECTLY if truly needed.

**Why it exists / what problem it solves:**
Imagine setting up a home theater system: turning on the projector, dimming the lights, powering the sound system, setting the correct input source, and starting the streaming device — EACH of these might be a SEPARATE subsystem with its OWN API. If every part of your application that wants to "watch a movie" needs to know and correctly SEQUENCE all FIVE of these separate operations, that's a lot of DUPLICATED, error-prone coordination logic scattered everywhere the feature is needed.

Facade solves this by introducing ONE class (`HomeTheaterFacade`) with a SINGLE, simple method (`watchMovie()`) that INTERNALLY coordinates all FIVE subsystem calls in the CORRECT order. Client code just calls `watchMovie()` — it doesn't need to know (or care) about the five underlying subsystems, their APIs, or the correct sequencing between them.

**When it should be used:**
- When a subsystem is COMPLEX, involving MANY classes/interfaces, and most CLIENTS only need a SIMPLE, common subset of its functionality
- When you want to DECOUPLE client code from a subsystem's internal structure, making it easier to CHANGE the subsystem's internals later without breaking clients
- When you want to provide a CLEAR, well-defined ENTRY POINT for common use cases, while still allowing advanced users to access subsystem classes DIRECTLY when they need finer control

**When it should NOT be used:**
- When the underlying subsystem is ALREADY simple enough that adding a Facade provides no real simplification benefit
- When clients GENUINELY need fine-grained access to EVERY subsystem class's full capabilities — an OVERLY restrictive Facade that hides TOO much can become an obstacle rather than a convenience
- When you find yourself adding SO MUCH logic to the Facade that it's no longer just "coordination/simplification" but has become a SEPARATE, complex subsystem in its own right (see Common Mistakes)

**Advantages:**
- Dramatically SIMPLIFIES client code for COMMON use cases, hiding subsystem complexity behind one clean entry point
- DECOUPLES clients from a subsystem's internal structure — the subsystem's internals CAN change without breaking client code that only uses the Facade
- Provides a natural, well-documented STARTING POINT for understanding/using a complex subsystem

**Disadvantages:**
- Can become a "GOD OBJECT" if it accumulates TOO MUCH logic/responsibility over time (see Common Mistakes)
- If OVERUSED to hide EVERYTHING, it can become an unnecessary bottleneck that PREVENTS legitimate advanced use cases from accessing subsystem classes directly
- Adds ONE more layer that developers need to understand IN ADDITION TO the subsystem itself, if they ever DO need to go beyond what the Facade provides

---

## 2. Real-World Analogy

Think of **a hotel concierge desk**.

As a guest, you don't personally coordinate with the housekeeping department, the restaurant, the spa, the local taxi company, AND the theater box office SEPARATELY every time you want "a nice evening out." Instead, you simply tell the CONCIERGE, "I'd like dinner reservations, theater tickets, and a taxi arranged for 7 PM" — and the concierge (the FACADE) handles ALL the underlying coordination with EACH separate department/vendor on your behalf. You still COULD call the restaurant directly yourself if you wanted very specific control over that ONE piece — the Facade doesn't PREVENT direct access, it just offers a much SIMPLER path for the common case.

---

## 3. Theory

**Core idea:** Identify a COMPLEX subsystem made up of MULTIPLE classes with INTRICATE interactions. Create ONE Facade class that provides SIMPLE, high-level methods, each of which INTERNALLY coordinates the NECESSARY calls to the various subsystem classes in the CORRECT sequence, hiding this complexity from the client.

**Working mechanism:**
```
Client → Facade.simpleOperation()
              │
              ├──→ SubsystemA.doThis()
              ├──→ SubsystemB.doThat()
              └──→ SubsystemC.doTheOther()
         (Facade coordinates ALL of this internally;
          Client only ever calls ONE simple method)
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Facade | The single class providing a simplified, high-level interface |
| Subsystem classes | The multiple, complex, interacting classes hidden behind the Facade |
| Client | Code that uses the Facade's simple interface, unaware of subsystem complexity |

**Class responsibilities:** the Facade is responsible for KNOWING WHICH subsystem classes to call, in WHAT ORDER, with WHAT parameters, to accomplish a COMMON, higher-level task — it does NOT reimplement any of the subsystem's actual logic itself, only COORDINATES existing subsystem behavior.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      Client                      │
└──────────┬──────────┘
           │ (depends on)
           ▼
┌─────────────────────┐
│      HomeTheaterFacade          │
├─────────────────────┤
│ - projector: Projector    ◇──┐
│ - lights: Lights          ◇──┤ (aggregation: Facade holds
│ - soundSystem: SoundSystem ◇──┤  references to ALL subsystem
│ - streamer: StreamingDevice◇──┘  classes it coordinates)
├─────────────────────┤
│ + watchMovie(): void          │
│ + endMovie(): void             │
└─────────────────────┘
           │ (coordinates calls to)
    ┌──────┼──────┬──────────┐
    ▼             ▼         ▼           ▼
┌────────┐ ┌────────┐ ┌────────────┐ ┌────────────────┐
│Projector│ │ Lights │ │SoundSystem      │ │StreamingDevice        │
└────────┘ └────────┘ └────────────┘ └────────────────┘
```

**Relationship explanation:**
- `HomeTheaterFacade` has an **aggregation** relationship with EACH subsystem class (`Projector`, `Lights`, `SoundSystem`, `StreamingDevice`) — it holds REFERENCES to them but doesn't OWN their entire lifecycle exclusively (they could exist and be used independently too).
- `Client` has a **dependency** on `HomeTheaterFacade` ONLY — critically, the Client does NOT need direct references to `Projector`, `Lights`, etc., though NOTHING prevents it from ALSO holding such references if it needs finer-grained control for some OTHER specific task.
- The Facade's methods (`watchMovie()`) internally make MULTIPLE calls to DIFFERENT subsystem classes, in a SPECIFIC sequence — this sequencing knowledge is the CORE value the Facade provides.

---

## 5. Java Implementation

```java
// ============================================
// Subsystem classes — each with its OWN, independent API
// ============================================
public class Projector {
    public void turnOn() {
        System.out.println("Projector: turning on");
    }
    public void setInput(String source) {
        System.out.println("Projector: input set to " + source);
    }
    public void turnOff() {
        System.out.println("Projector: turning off");
    }
}

public class Lights {
    public void dim(int level) {
        System.out.println("Lights: dimming to " + level + "%");
    }
    public void turnOn() {
        System.out.println("Lights: turning on to full brightness");
    }
}

public class SoundSystem {
    public void turnOn() {
        System.out.println("SoundSystem: turning on");
    }
    public void setVolume(int level) {
        System.out.println("SoundSystem: volume set to " + level);
    }
    public void turnOff() {
        System.out.println("SoundSystem: turning off");
    }
}

public class StreamingDevice {
    public void turnOn() {
        System.out.println("StreamingDevice: turning on");
    }
    public void play(String movie) {
        System.out.println("StreamingDevice: playing '" + movie + "'");
    }
    public void stop() {
        System.out.println("StreamingDevice: stopping playback");
    }
}

// ============================================
// Facade — provides a SIMPLE, high-level interface,
// hiding the COMPLEX coordination of subsystem classes
// ============================================
public class HomeTheaterFacade {
    private final Projector projector;
    private final Lights lights;
    private final SoundSystem soundSystem;
    private final StreamingDevice streamer;

    public HomeTheaterFacade(Projector projector, Lights lights,
                              SoundSystem soundSystem, StreamingDevice streamer) {
        this.projector = projector;
        this.lights = lights;
        this.soundSystem = soundSystem;
        this.streamer = streamer;
    }

    // ONE simple method call replaces coordinating FOUR separate
    // subsystems in the CORRECT sequence
    public void watchMovie(String movie) {
        System.out.println("--- Getting ready to watch a movie ---");
        lights.dim(10);
        projector.turnOn();
        projector.setInput("HDMI-1");
        soundSystem.turnOn();
        soundSystem.setVolume(15);
        streamer.turnOn();
        streamer.play(movie);
    }

    public void endMovie() {
        System.out.println("--- Shutting down the home theater ---");
        streamer.stop();
        soundSystem.turnOff();
        projector.turnOff();
        lights.turnOn();
    }
}

// ============================================
// Demo — Client uses the SIMPLE Facade interface
// ============================================
public class FacadeDemo {
    public static void main(String[] args) {
        Projector projector = new Projector();
        Lights lights = new Lights();
        SoundSystem soundSystem = new SoundSystem();
        StreamingDevice streamer = new StreamingDevice();

        HomeTheaterFacade homeTheater = new HomeTheaterFacade(
                projector, lights, soundSystem, streamer);

        // Client calls ONE method — no need to know about
        // FOUR separate subsystems or their correct sequencing
        homeTheater.watchMovie("Inception");

        System.out.println();
        homeTheater.endMovie();
    }
}
```

**Key line-by-line notes:**
- `HomeTheaterFacade`'s constructor takes references to ALL FOUR subsystem classes — these could be created directly, injected via a Dependency Injection framework, or constructed internally within the Facade itself (a common variant, discussed further in Interview Question 5).
- `watchMovie()` internally calls SEVEN separate subsystem method calls, in a SPECIFIC, carefully considered SEQUENCE — this sequencing knowledge is EXACTLY the complexity the Facade hides from the Client.
- The Client (`FacadeDemo`) NEVER calls `projector.turnOn()` or `soundSystem.setVolume()` directly — it interacts ONLY through the Facade's simple `watchMovie()`/`endMovie()` methods.

---

## 6. Dry Run

**Sample input:** `homeTheater.watchMovie("Inception")`

```
1. homeTheater.watchMovie("Inception") called
   → HomeTheaterFacade.watchMovie() executes
   → Prints: "--- Getting ready to watch a movie ---"
   → lights.dim(10) called → Prints: "Lights: dimming to 10%"
   → projector.turnOn() called → Prints: "Projector: turning on"
   → projector.setInput("HDMI-1") called → Prints: "Projector: input set to HDMI-1"
   → soundSystem.turnOn() called → Prints: "SoundSystem: turning on"
   → soundSystem.setVolume(15) called → Prints: "SoundSystem: volume set to 15"
   → streamer.turnOn() called → Prints: "StreamingDevice: turning on"
   → streamer.play("Inception") called → Prints: "StreamingDevice: playing 'Inception'"
   → watchMovie() returns
```

**What's happening in memory:** the `HomeTheaterFacade` object holds REFERENCES to four ALREADY-EXISTING subsystem objects. A SINGLE client call (`watchMovie()`) triggers a SEQUENCE of SEVEN separate method calls on FOUR DIFFERENT objects — all of this coordination logic lives ENTIRELY within the Facade's method body; the Client's call stack shows ONLY the ONE call it made, with the Facade internally FANNING OUT to the subsystem calls beneath it.

---

## 7. Real-World Software Example

- **Spring Boot "starters"**: a SINGLE dependency (e.g., `spring-boot-starter-web`) acts as a Facade over MANY underlying libraries/configurations (embedded Tomcat, Jackson for JSON, Spring MVC setup) — developers get SENSIBLE DEFAULTS through one simple entry point, without manually configuring each underlying piece.
- **`javax.faces.context.FacesContext`** (JSF): provides a SIMPLIFIED entry point coordinating MANY underlying JSF subsystems (request/response handling, view management, etc.).
- **Compiler front-ends**: a SINGLE `compile()` method that internally coordinates LEXING, PARSING, SEMANTIC ANALYSIS, and CODE GENERATION — each a complex subsystem in its own right, hidden behind ONE simple entry point for typical usage.
- **ORM frameworks (Hibernate's `Session`)**: provides a SIMPLIFIED interface over the COMPLEX underlying machinery of SQL generation, connection management, caching, and object-relational mapping.
- **Payment gateway SDKs**: a SINGLE `charge(amount, card)` method that internally coordinates fraud detection, currency conversion, tokenization, and the ACTUAL payment network communication.

---

## 8. Internal Working

**Object creation:** the Facade typically holds REFERENCES to ALREADY-EXISTING subsystem instances (injected via constructor, as shown in Section 5) — though some Facade implementations INTERNALLY construct their OWN subsystem instances if clients never need to access them independently.

**Runtime interactions / call flow:** a SINGLE Facade method call FANS OUT into MULTIPLE subsystem calls, executed SEQUENTIALLY (or occasionally in parallel, depending on the subsystem's requirements) — the Facade's method body is essentially a SCRIPT coordinating this sequence.

**Memory usage:** minimal ADDITIONAL overhead — the Facade itself typically holds only REFERENCES to subsystem objects, not duplicating any of their actual data or state.

**Comparison to Mediator (a natural point of confusion, discussed further in Interview Questions):** Facade coordinates calls in ONE DIRECTION — Client → Facade → Subsystems — the subsystems typically DON'T need to communicate back through the Facade to EACH OTHER. This is structurally SIMPLER than Mediator (Topic 25, upcoming), where OBJECTS communicate WITH EACH OTHER indirectly THROUGH a central Mediator, in BOTH directions.

---

## 9. Before vs After

**Before (no Facade — client directly coordinates subsystems):**

```java
public class MovieWatchingClientBad {
    public void watchMovie(Projector projector, Lights lights,
                            SoundSystem soundSystem, StreamingDevice streamer,
                            String movie) {
        // Client code must know ALL FOUR subsystems AND
        // the CORRECT sequence to coordinate them —
        // this logic would need to be DUPLICATED everywhere
        // "watching a movie" is needed in the application
        lights.dim(10);
        projector.turnOn();
        projector.setInput("HDMI-1");
        soundSystem.turnOn();
        soundSystem.setVolume(15);
        streamer.turnOn();
        streamer.play(movie);
    }
}
```

**Problems:**
- EVERY place in the application that wants to "watch a movie" needs to KNOW about all FOUR subsystems and the CORRECT sequence — this coordination logic gets DUPLICATED wherever the feature is needed.
- If the CORRECT sequence changes (e.g., a new subsystem is added, or the order needs adjusting), EVERY duplicated copy of this logic needs to be found and updated.
- Client code becomes CLUTTERED with low-level subsystem coordination details that aren't really its concern.

**After (Facade, as shown in Section 5):**
- ALL coordination logic lives in ONE place (`HomeTheaterFacade.watchMovie()`) — Client code is REDUCED to a single, clear method call.
- If the correct sequence changes, ONLY the Facade needs updating — ALL client code automatically benefits without any changes on their end.

---

## 10. SOLID Principles Connection

- **SRP**: the Facade's SOLE responsibility is COORDINATING subsystem calls for common use cases — it doesn't reimplement subsystem logic itself, keeping this responsibility cleanly separated from the subsystems' own individual responsibilities.
- **DIP**: client code can depend on the Facade (or even a Facade INTERFACE, for even looser coupling) rather than needing to depend on MULTIPLE separate subsystem classes directly.
- **OCP consideration**: the Facade CAN be extended with NEW simple methods for NEW common use cases without modifying EXISTING methods or subsystem classes — though changes to the SUBSYSTEMS' own APIs may still require Facade updates, this is generally CONTAINED to one place rather than scattered across client code.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Facade pattern solve, in your own words?
2. Does using a Facade PREVENT client code from accessing subsystem classes directly? Why or why not?
3. In the home theater example, what's the SPECIFIC complexity the Facade is hiding?

**Intermediate:**
4. How does Facade help with DECOUPLING client code from a subsystem's internal structure? What happens if the subsystem's internals CHANGE later?
5. Should the Facade CONSTRUCT its own subsystem instances internally, or receive them via CONSTRUCTOR INJECTION (as shown in Section 5)? Discuss the tradeoffs of each approach.
6. Can a system have MULTIPLE different Facades over the SAME underlying subsystem, each tailored for a DIFFERENT common use case? Give an example of when this might make sense.

**Advanced:**
7. **Facade vs Adapter** — precisely distinguish them (a very common interview follow-up, since both "wrap" other code — this comparison also appeared in Topic 15's Interview Questions).
   *Answer: Facade's intent is to SIMPLIFY a COMPLEX subsystem (typically MANY classes) by providing ONE easier, higher-level interface — it REDUCES complexity. Adapter's intent is to TRANSLATE between an ALREADY-EXISTING, INCOMPATIBLE interface and what a client EXPECTS — it fixes a COMPATIBILITY mismatch, typically for ONE class. A Facade might even use SEVERAL Adapters internally, if some of the subsystems it coordinates have INCOMPATIBLE interfaces that ALSO need translating.*
8. **Facade vs Mediator** — precisely distinguish them (both involve a central coordinating object, and Mediator will be covered in Topic 25).
   *Answer: Facade coordinates calls in effectively ONE DIRECTION — Client calls Facade, Facade calls Subsystems — the SUBSYSTEMS typically don't need to be AWARE of each other or communicate BACK through the Facade. Mediator is specifically designed so that MULTIPLE objects communicate WITH EACH OTHER INDIRECTLY through the Mediator, in BOTH directions — the Mediator actively manages ONGOING, BIDIRECTIONAL communication BETWEEN objects, whereas Facade provides a SIMPLER, more ONE-SHOT coordination of a sequence of calls.*
9. How would you prevent a Facade from turning into a "GOD OBJECT" over time, as MORE and MORE functionality gets added to it? What warning signs would you look for?
10. Could a Facade be combined with Singleton (Topic 16)? In what scenario would providing a Facade as a Singleton make sense?

---

## 12. Common Mistakes

- **Letting the Facade accumulate TOO MUCH logic over time**, turning it into a "GOD OBJECT" that knows and does FAR too much — if a Facade starts containing SIGNIFICANT business logic beyond pure COORDINATION, it's a sign that logic should probably live elsewhere (perhaps in a dedicated SERVICE class), with the Facade remaining a THIN coordination layer.
- **Making the Facade the ONLY way to access ANY subsystem functionality**, even for legitimate ADVANCED use cases that need finer control — a well-designed Facade SIMPLIFIES the common case WITHOUT blocking direct subsystem access for those who genuinely need it.
- **Confusing Facade with Adapter** — remember: Facade SIMPLIFIES complexity across MANY classes; Adapter TRANSLATES an incompatibility for typically ONE class (see Interview Question 7).
- **Creating a Facade for a subsystem that ISN'T actually complex** — if there are only ONE or TWO simple classes involved, a Facade adds an unnecessary layer without meaningful simplification benefit.

---

## 13. Time Complexity

Not meaningfully different from directly calling the underlying subsystem methods — O(1) additional overhead for the Facade's coordination logic itself (though the TOTAL work done, across all the subsystem calls it fans out to, is whatever those calls collectively require). The pattern's value is entirely about SIMPLIFICATION and DECOUPLING, not algorithmic efficiency.

---

## 14. Java API Examples

- **Spring Boot Starters**: a single dependency acting as a Facade over numerous underlying library configurations, providing SENSIBLE DEFAULTS for common use cases.
- **`javax.faces.context.FacesContext`**: coordinates MANY underlying JSF request-processing subsystems behind a simplified access point.
- **Hibernate's `Session` interface**: provides a SIMPLIFIED entry point over the complex machinery of SQL generation, caching, and object-relational mapping.
- **`java.net.URL`'s `openConnection()`/`openStream()`** methods: provide a SIMPLIFIED entry point over the more complex, lower-level networking machinery involved in actually establishing a connection and reading data.

---

## 15. Practice Problem

Implement a **Facade for an online order checkout process**: create subsystem classes `InventoryService` (checks/reserves stock), `PaymentService` (charges the customer), and `ShippingService` (schedules delivery). Implement an `OrderFacade` with a single `placeOrder(String itemId, int quantity, String cardNumber, String address)` method that coordinates all three subsystems in the correct sequence (check inventory → charge payment → schedule shipping), and handles the case where inventory is insufficient by NOT proceeding with payment or shipping.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **Facade for a video conferencing application's 'Start Meeting' feature**. The underlying subsystems include: `CameraService` (initializes the webcam), `MicrophoneService` (initializes audio input), `NetworkService` (establishes a connection to conferencing servers), `ScreenShareService` (available but NOT started by default), and `RecordingService` (available but NOT started by default). Design the Facade so that `startMeeting()` initializes ONLY camera, microphone, and network by default, while STILL allowing advanced users to access `ScreenShareService` and `RecordingService` DIRECTLY when needed."

Think about:
- How this scenario illustrates the IMPORTANT point (from Common Mistakes) that a Facade should SIMPLIFY the COMMON case WITHOUT blocking access to subsystem features that aren't part of that common case — how would you expose `ScreenShareService`/`RecordingService` for advanced use, while keeping `startMeeting()` itself simple?
- Whether the Facade should return REFERENCES to the underlying subsystem objects (allowing advanced direct access) or keep them ENTIRELY private — discuss the tradeoffs of each design choice.

---

## 17. Advanced LLD Scenario

**Design an E-Commerce Order Fulfillment Facade** for a large-scale retail platform, where:
- The underlying subsystems include: `InventoryService`, `PricingService` (applies discounts/taxes), `FraudDetectionService`, `PaymentGatewayService`, `WarehouseRoutingService` (determines WHICH warehouse should fulfill the order), and `NotificationService` (sends order confirmation)
- A `CheckoutFacade` needs to coordinate ALL SIX of these subsystems in the CORRECT sequence for the COMMON "place a standard order" use case, while ALSO supporting variations like "place an order with a promotional code" or "place an expedited order requiring different warehouse routing logic" — WITHOUT the Facade becoming a bloated "god object" trying to handle EVERY possible variation as special-cased branches
- Consider how STRATEGY (Topic 5) might be combined WITH Facade here — e.g., different "checkout flows" (standard, promotional, expedited) could each be represented as a STRATEGY that the Facade DELEGATES to, rather than the Facade itself containing large `if-else` blocks for each variation

Consider:
- Why simply ADDING more and more special-cased branches DIRECTLY inside `CheckoutFacade` (for promotional orders, expedited orders, etc.) would eventually violate the "avoid god object" principle discussed in Common Mistakes — and how COMBINING Facade with Strategy addresses this by keeping the Facade's OWN responsibility focused on ORCHESTRATION, while DELEGATING variation-specific LOGIC to interchangeable Strategy objects
- How this scenario demonstrates that REAL-WORLD Facades often need to be DESIGNED alongside OTHER patterns (not used in complete isolation) to remain CLEAN and maintainable as a system's complexity grows over time
- Why RECOGNIZING the signs of Facade complexity creep EARLY (as discussed in Interview Question 9) is a genuinely valuable, PRACTICAL skill for maintaining large, evolving codebases — not just an academic concern

---

## 18. Summary

**Definition:** Facade provides a simplified, higher-level interface to a complex subsystem consisting of many interacting classes, hiding internal complexity behind one clean entry point.

**Intent:** Reduce complexity for common use cases and decouple client code from a subsystem's internal structure, while still allowing direct subsystem access for advanced needs.

**Key classes:** `Facade` (the simplified entry point, coordinating subsystem calls), the `Subsystem classes` themselves (each with their own complex, independent APIs).

**Advantages:** Dramatically simplifies client code; decouples clients from subsystem internals; provides a clear, well-documented starting point for complex functionality.

**Disadvantages:** Risk of becoming a bloated "god object" over time; can unnecessarily restrict advanced use cases if overused; adds one more layer to understand.

**Real-world use case:** Spring Boot starters, Hibernate's `Session`, payment gateway SDKs, home theater/media control systems.

**Java example:** `HomeTheaterFacade.watchMovie()` coordinating `Projector`, `Lights`, `SoundSystem`, and `StreamingDevice` in the correct sequence behind one simple method call.

**Interview takeaway:** Be ready to clearly distinguish Facade (simplifying COMPLEXITY across MANY classes) from Adapter (fixing an interface MISMATCH for typically ONE class, Topic 15) and from Mediator (managing ONGOING, BIDIRECTIONAL communication BETWEEN objects, Topic 25) — these coordinating/wrapping patterns are frequently confused, and clearly separating their INTENTS is a strong interview signal.

**One-line memory trick:** *"Call the hotel concierge instead of coordinating with the restaurant, spa, and taxi company separately yourself."*

---

*End of Topic 20. Type "Next" to proceed to Topic 21: Flyweight Pattern.*