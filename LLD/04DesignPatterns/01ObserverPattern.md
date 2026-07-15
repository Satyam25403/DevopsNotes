# Topic 6: Observer Design Pattern

---

## 1. Introduction

**Definition:**
The Observer Pattern is a **behavioral design pattern** that defines a one-to-many dependency between objects, such that when ONE object (the "Subject" or "Observable") changes state, ALL its dependent objects (the "Observers") are automatically notified and updated — without the Subject needing to know any specifics about who its Observers are or what they'll do with the notification.

**Why it exists / what problem it solves:**
Without Observer, if multiple parts of a system need to react to a change in some object's state, you'd typically end up with either:
- The changed object directly calling methods on every interested party — creating tight coupling (the Subject needs to know about EVERY specific class it must notify, and must be modified every time a new interested party is added)
- Polling — repeatedly checking "has anything changed?" from the interested parties' side, which wastes resources and introduces latency

Observer solves this by INVERTING the relationship: interested parties (Observers) REGISTER themselves with the Subject. The Subject simply broadcasts "something changed" to whoever is currently registered, without knowing or caring what each Observer specifically does with that information.

**When it should be used:**
- When a change to one object needs to automatically trigger updates in an unknown or varying number of other objects
- When you want to broadcast events/state changes without tightly coupling the broadcaster to specific subscriber implementations
- Event-driven systems, UI frameworks (button clicks notifying listeners), real-time data feeds

**When it should NOT be used:**
- When the "notification chain" becomes so complex that debugging WHO gets notified WHEN becomes difficult (a common real risk with deeply nested Observer chains)
- When strict ORDERING or GUARANTEED delivery is required and the simple Observer pattern doesn't provide it — a proper message queue (Kafka, RabbitMQ) may be more appropriate for production-grade, guaranteed-delivery scenarios
- When there's only ever exactly ONE interested party — a direct method call is simpler and clearer

**Advantages:**
- Loose coupling between the Subject and Observers — the Subject doesn't need to know concrete Observer classes
- Observers can be added/removed dynamically at runtime
- Supports broadcast communication to multiple objects efficiently, without polling

**Disadvantages:**
- Can lead to unexpected behavior if Observers have ordering dependencies that aren't explicitly managed
- Memory leaks are a REAL risk if Observers aren't properly unregistered when no longer needed (the Subject holds references to them, preventing garbage collection)
- Debugging can be harder — a change ripples through potentially many Observers, making the full effect of a state change less obvious by reading code linearly

---

## 2. Real-World Analogy

Think of **YouTube channel subscriptions and notifications**.

- A YouTuber (the **Subject**) uploads a new video.
- Every person who has SUBSCRIBED to that channel (the **Observers**) automatically gets a notification.
- The YouTuber doesn't personally know or maintain a list of "Alice, Bob, Carol..." — they just upload, and YouTube's platform handles notifying WHOEVER is currently subscribed.
- New viewers can SUBSCRIBE (register as an Observer) at any time, and existing subscribers can UNSUBSCRIBE (deregister) at any time — the YouTuber's upload process never changes regardless of how many subscribers come and go.
- Different subscribers might react differently to the SAME notification — one watches immediately, another ignores it, another comments — the YouTuber (Subject) doesn't control or care about this; it only handles the BROADCAST.

---

## 3. Theory

**Core idea:** A Subject maintains a list of Observers. When the Subject's state changes, it iterates through its Observer list and calls a common "update" method on each — without knowing their concrete types.

**Working mechanism:**
```
┌─────────────────────┐
│      Subject                    │
│  - observers: List<Observer>       │
│  + attach(observer)                  │
│  + detach(observer)                    │
│  + notifyObservers()                     │
└──────────┬──────────┘
           │ maintains a list of, and calls
           │ update() on each
           ↓
┌─────────────────────┐
│      <<interface>>          │
│         Observer                 │
│  + update(data): void              │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────┐
   │               │
┌──┴────┐    ┌──┴────┐
│ObserverA │    │ObserverB │
└────────┘    └────────┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Subject / Observable | The object being watched; maintains and notifies Observers |
| Observer | An object that wants to be notified of the Subject's changes |
| Attach/Subscribe | Registering an Observer with a Subject |
| Detach/Unsubscribe | Removing an Observer's registration |
| Notify | The Subject broadcasting a change to all registered Observers |

**Push vs Pull models:**
```
Push model: Subject sends the FULL data/details in the
            notification itself
            update(newTemperature, newHumidity, newPressure)

Pull model: Subject sends a minimal notification, and
            Observers PULL whatever additional data they
            need FROM the Subject themselves
            update(subject) → observer calls subject.getTemperature()
                               only if it actually cares about temperature
```

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Observer                 │
├─────────────────────┤
│ + update(temp, humidity): void      │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────────────┐
   │                          │
┌──┴──────────┐    ┌──┴──────────┐
│ PhoneDisplay        │    │ TvDisplay              │
└────────────┘    └────────────┘

┌─────────────────────┐
│      WeatherStation             │
├─────────────────────┤
│ - observers: List<Observer>  ◇────→ (aggregation: WeatherStation
│ - temperature: double               │  holds Observer references,
│ - humidity: double                     │  but doesn't own their
├─────────────────────┤                │  lifecycle)
│ + attach(Observer o)                     │
│ + detach(Observer o)                        │
│ + setMeasurements(t, h)                        │
│ - notifyObservers()  (private, internal)          │
└─────────────────────┘
```

**Relationship explanation:**
- `WeatherStation` (the Subject) has an **aggregation** relationship with `Observer` (hollow diamond) — it holds a LIST of observer references but doesn't control their creation or destruction.
- `PhoneDisplay` and `TvDisplay` **implement** `Observer` — each can react to the SAME notification completely differently (one might show a simple icon, another a detailed graph).
- The Subject's `notifyObservers()` method iterates the list and calls `update()` polymorphically on each — the Subject's code doesn't change regardless of how many or which types of Observers are registered.

---

## 5. Java Implementation

```java
import java.util.ArrayList;
import java.util.List;

// ============================================
// Observer interface — the common contract
// ============================================
public interface Observer {
    void update(double temperature, double humidity);
}

// ============================================
// Subject interface — defines how Observers
// register/deregister themselves
// ============================================
public interface Subject {
    void attach(Observer observer);
    void detach(Observer observer);
    void notifyObservers();
}

// ============================================
// Concrete Subject — the object being "watched"
// ============================================
public class WeatherStation implements Subject {
    private final List<Observer> observers = new ArrayList<>();
    private double temperature;
    private double humidity;

    @Override
    public void attach(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void detach(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers() {
        // Iterates through ALL currently registered observers,
        // calling update() on each — WeatherStation has ZERO
        // knowledge of what PhoneDisplay or TvDisplay actually
        // DO with this data
        for (Observer observer : observers) {
            observer.update(temperature, humidity);
        }
    }

    // Called whenever new sensor readings come in
    public void setMeasurements(double temperature, double humidity) {
        this.temperature = temperature;
        this.humidity = humidity;
        notifyObservers(); // automatically broadcast the change
    }
}

// ============================================
// Concrete Observers — each reacts to the SAME
// notification differently
// ============================================
public class PhoneDisplay implements Observer {
    @Override
    public void update(double temperature, double humidity) {
        System.out.printf("[Phone] Temp: %.1f°C, Humidity: %.1f%%%n", temperature, humidity);
    }
}

public class TvDisplay implements Observer {
    @Override
    public void update(double temperature, double humidity) {
        System.out.printf("[TV] Current weather -> %.1f°C / %.1f%% humidity%n", temperature, humidity);
    }
}

// ============================================
// Demo
// ============================================
public class ObserverDemo {
    public static void main(String[] args) {
        WeatherStation station = new WeatherStation();

        Observer phone = new PhoneDisplay();
        Observer tv = new TvDisplay();

        station.attach(phone);
        station.attach(tv);

        // A single state change automatically notifies BOTH observers
        station.setMeasurements(25.5, 60.0);

        System.out.println("--- TV display unsubscribes ---");
        station.detach(tv);

        // Now only the phone gets notified
        station.setMeasurements(27.0, 55.0);
    }
}
```

**Key line-by-line notes:**
- `List<Observer> observers` — a collection of the INTERFACE type, never concrete classes; this is what allows any number of DIFFERENT observer types to register.
- `notifyObservers()` loops and calls `update()` polymorphically — `WeatherStation` never checks `instanceof PhoneDisplay` or similar; it treats every observer identically through the shared interface.
- `setMeasurements()` calling `notifyObservers()` internally — this is the "push" trigger point: whenever the Subject's state genuinely changes, it proactively broadcasts, rather than waiting for Observers to ask.
- `station.detach(tv)` — demonstrates dynamic, runtime deregistration; after this call, `tv` will never receive another update, and (assuming nothing else references it) becomes eligible for garbage collection.

---

## 6. Dry Run

**Sample input:** Running `ObserverDemo.main()`.

```
1. new WeatherStation() → observers = empty list

2. new PhoneDisplay() + new TvDisplay()
   → Two separate objects allocated on heap

3. station.attach(phone) → observers = [phone]
   station.attach(tv)      → observers = [phone, tv]

4. station.setMeasurements(25.5, 60.0)
   → temperature = 25.5, humidity = 60.0
   → notifyObservers() called internally
   → Loop iteration 1: observers[0] = phone
       → phone.update(25.5, 60.0) called
       → Dynamic dispatch resolves to PhoneDisplay's update()
       → Prints: "[Phone] Temp: 25.5°C, Humidity: 60.0%"
   → Loop iteration 2: observers[1] = tv
       → tv.update(25.5, 60.0) called
       → Dynamic dispatch resolves to TvDisplay's update()
       → Prints: "[TV] Current weather -> 25.5°C / 60.0% humidity"

5. station.detach(tv)
   → observers list now = [phone] (tv reference removed)

6. station.setMeasurements(27.0, 55.0)
   → notifyObservers() called
   → Loop only has ONE element now: phone
   → phone.update(27.0, 55.0) called
   → Prints: "[Phone] Temp: 27.0°C, Humidity: 55.0%"
   → tv is NEVER called — it received no notification
     because it's no longer in the observers list
```

**What's happening in memory:** the `observers` list holds REFERENCES to Observer objects. `detach()` simply removes a reference from this list — it doesn't destroy the `TvDisplay` object itself (if something else still holds a reference to it elsewhere, it remains alive; otherwise, it becomes eligible for garbage collection). This is precisely why FORGETTING to detach observers that are no longer needed is a real memory leak risk — the Subject's list keeps a live reference indefinitely.

---

## 7. Real-World Software Example

- **Stock market price notifications**: a `Stock` object (Subject) notifies registered `Trader` objects (Observers) whenever its price changes, without the stock needing to know which specific trading algorithms or dashboards are watching it.
- **UI event listeners**: a button (Subject) notifies all registered `ActionListener` (Observer) implementations when clicked — the exact mechanism behind Java Swing/AWT's event handling.
- **Reactive programming frameworks** (RxJava, Project Reactor): built ENTIRELY around a more sophisticated version of Observer — Observables emitting items to Subscribers.
- **Model-View-Controller (MVC) frameworks**: the Model (Subject) notifies Views (Observers) when data changes, so the UI automatically refreshes without the Model needing to know about specific UI components.

---

## 8. Internal Working

**Object creation and references:** the Subject's `observers` list holds normal object references (pointers) — no special JVM mechanism is involved beyond standard collection behavior.

**Method dispatch:** exactly like Strategy, Observer relies on DYNAMIC DISPATCH — `observer.update(...)` resolves to whichever concrete class's implementation the ACTUAL object belongs to, via the vtable lookup mechanism from Topic 2.

**A critical internal-working concern — memory leaks:**
```
Subject (WeatherStation)
   observers: [phone, tv, ...]
                  ↑
   As long as WeatherStation is ALIVE and holds these
   references, the Observer objects CANNOT be garbage
   collected — even if no other part of the program
   still needs them.

This is why long-lived Subjects (e.g., a singleton event
bus) that ACCUMULATE observers without ever detaching
them are a classic source of memory leaks in long-running
applications (a very real production issue, not just
theoretical) — sometimes called a "lapsed listener" problem.

Mitigation: explicitly detach() observers when they're
no longer needed (e.g., in a UI component's cleanup/
destroy lifecycle method), or use WEAK REFERENCES
(java.lang.ref.WeakReference) so the garbage collector
CAN reclaim an observer even if the Subject still
technically references it.
```

---

## 9. Before vs After

**Before (tightly coupled, manual notification):**

```java
public class WeatherStationBad {
    private double temperature;
    private PhoneDisplay phoneDisplay;  // hardcoded, concrete dependency
    private TvDisplay tvDisplay;            // hardcoded, concrete dependency

    public void setTemperature(double temperature) {
        this.temperature = temperature;
        // Manually calling EACH specific display directly —
        // adding a new display type means editing THIS method
        phoneDisplay.updateTemp(temperature);
        tvDisplay.updateTemp(temperature);
    }
}
```

**Problems:**
- `WeatherStationBad` must know about EVERY concrete display type, and hold direct references to each — violates DIP.
- Adding a new display (e.g., a `SmartWatchDisplay`) requires MODIFYING this class directly — violates OCP.
- No way to add/remove displays at RUNTIME — they're fixed fields, not a dynamic collection.

**After (Observer pattern, as shown in Section 5):**
- `WeatherStation` holds a generic `List<Observer>` — it never needs to know concrete display types.
- New display types are added by simply creating a new class implementing `Observer` and calling `attach()` — zero changes to `WeatherStation`.
- Displays can be attached/detached dynamically, at runtime, as many times as needed.

---

## 10. SOLID Principles Connection

- **OCP**: new Observer types can be added without modifying the Subject.
- **SRP**: the Subject's only responsibility is managing its own state and broadcasting changes; it has no knowledge of or responsibility for what each Observer does with that information.
- **DIP**: the Subject depends on the `Observer` interface abstraction, never on concrete observer classes.
- **ISP** (worth considering): if the `Observer` interface's `update()` method forces ALL observers to receive data they don't need (e.g., a single `update()` method with 10 parameters, most irrelevant to any given observer), this can quietly become an ISP violation — sometimes addressed by using more granular, event-specific notification methods, or a generic `Event` object observers can selectively inspect.

---

## 11. Interview Questions

**Beginner:**
1. What is the Observer pattern, and what problem does it solve?
2. What's the difference between "push" and "pull" notification models in Observer?
3. Give a real-world example of the Observer pattern besides weather stations.

**Intermediate:**
4. What is a potential memory-related risk with the Observer pattern, and how would you mitigate it?
5. How would you ensure the ORDER in which observers are notified is predictable/controllable?
6. What's the difference between the Observer pattern and simply using a `List<Runnable>` and calling `run()` on each?
   *Answer: Structurally very similar for simple cases — the key semantic difference is that Observer typically PASSES DATA about what changed (via update() parameters), whereas a bare Runnable list has no built-in mechanism for passing the changed state, though you could design a Runnable-based system similarly with lambdas capturing data.*

**Advanced:**
7. **Observer vs Mediator pattern** — how do they differ in terms of communication structure?
   *Answer: Observer is typically ONE-TO-MANY (one subject broadcasting to many observers, unaware of each other). Mediator (Topic 25) is often MANY-TO-MANY, with a central Mediator object coordinating communication BETWEEN multiple objects that would otherwise need to reference each other directly — Mediator centralizes complex interaction logic, while Observer centralizes simple broadcast notification.*
8. How does the Observer pattern relate to the publish-subscribe (pub-sub) messaging pattern used in distributed systems (e.g., Kafka)? What's different at scale?
   *Answer: Conceptually very similar (publishers broadcast, subscribers receive) but pub-sub systems like Kafka add: persistence (messages aren't lost if a subscriber is temporarily down), decoupling across PROCESS/NETWORK boundaries (not just within one JVM), and guaranteed delivery/ordering semantics that the basic in-memory Observer pattern doesn't provide.*
9. How would you implement thread-safety in an Observer pattern where multiple threads might call `attach()`/`detach()`/`notifyObservers()` concurrently?
10. What's the "lapsed listener" problem, and how do WeakReferences help address it?
11. Design considerations: should `notifyObservers()` be called automatically inside every state-changing method (like `setMeasurements()`), or should it be a separate, manually-triggered step? What are the tradeoffs?

---

## 12. Common Mistakes

- **Forgetting to detach observers**, leading to memory leaks in long-running applications — especially common in UI code where a component is "removed from the screen" but never properly unregistered from an event source.
- **Modifying the observers list WHILE iterating over it** (e.g., an Observer's `update()` method calling `detach()` on itself) — this can throw a `ConcurrentModificationException` in Java; typically solved by iterating over a COPY of the list, or using a thread-safe/copy-on-write collection.
- **Assuming a specific notification order** without explicitly designing for it — `ArrayList` iteration order is insertion order, but this is an implementation detail that shouldn't be silently relied upon without deliberate design if order genuinely matters.
- **Putting too much logic in the Subject about WHAT to notify** — if the Subject starts checking `instanceof` to decide which observers get which data, that's a sign the design has drifted away from Observer's clean decoupling.

---

## 13. Time Complexity

**Notification broadcast**: O(n) where n is the number of registered observers — each observer's `update()` is called exactly once per notification. **Attach/Detach**: O(1) for a simple list append, or O(n) for `ArrayList.remove()` (since it needs to find and shift elements) — using a `HashSet` for observers instead of a `List` can make detach O(1) if ordering doesn't matter.

---

## 14. Java API Examples

- **`java.util.Observer`/`java.util.Observable`**: Java historically shipped a built-in Observer implementation in `java.util` — though it's been DEPRECATED since Java 9, largely because `Observable` was a class (not an interface), forcing single inheritance limitations, and lacked proper thread-safety guarantees. This is itself a good interview discussion point.
- **`java.beans.PropertyChangeListener`**: the modern-era recommended replacement pattern for the deprecated `Observable`, widely used in JavaBeans-style property change notification.
- **Swing/AWT event listeners**: `ActionListener`, `MouseListener`, etc. — every GUI button/component acts as a Subject, and registered listeners act as Observers.
- **RxJava's `Observable`/`Subscriber`**: a full reactive-programming evolution of this same core pattern, adding operators (map, filter, merge) for composing event streams.

---

## 15. Practice Problem

Implement a **stock price alert system**: a `Stock` class (Subject) with a `setPrice(double price)` method that notifies registered `Investor` observers whenever the price changes. Each `Investor` should have a "threshold" price they care about, and their `update()` method should only print an alert if the new price crosses their specific threshold (going above it). Demonstrate multiple investors with different thresholds all observing the same stock.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a notification system for an e-commerce order tracking feature, where a customer can subscribe to updates about their order's status (Placed, Shipped, Out for Delivery, Delivered). Multiple notification channels (Email, SMS, Push Notification) should all receive the SAME status update event, and a customer should be able to unsubscribe from tracking at any time (e.g., after cancelling the order)."

Think about:
- Should there be ONE Observer interface for all three channels, or does each channel need different data, suggesting separate handling?
- How would you avoid notifying a channel the customer has explicitly disabled (e.g., they don't want SMS, only email)?

---

## 17. Advanced LLD Scenario

**Design a Real-Time Auction System** (like eBay) where:
- Multiple bidders are watching the SAME auction item
- When a new bid is placed, ALL watching bidders must be notified immediately of the new current price
- The auction itself shouldn't need to know how many bidders are watching, or how each one chooses to be notified (in-app, email, SMS)
- When an auction ends, ALL observers need a FINAL notification with the winning bid, and then should be automatically cleaned up (detached) to avoid memory leaks after the auction closes

Consider:
- How the `Auction` class (Subject) manages potentially large numbers of watching `Bidder` observers efficiently
- How you'd handle the END-OF-AUCTION cleanup to avoid the "lapsed listener" memory leak problem discussed in Section 8
- Whether real-time, MANY simultaneous auctions (each with potentially thousands of watchers) might push you toward a proper pub-sub messaging system (like Kafka) rather than a purely in-memory Observer implementation — and what factors would drive that decision

---

## 18. Summary

**Definition:** Observer defines a one-to-many dependency where a Subject automatically notifies all registered Observers when its state changes.

**Intent:** Enable loosely-coupled, broadcast-style communication — the Subject doesn't need to know concrete Observer types.

**Key classes:** `Subject`/`Observable` (maintains observer list, broadcasts changes), `Observer` interface (the notification contract), Concrete Observers (actual reactions to changes).

**Advantages:** Loose coupling, dynamic runtime subscription/unsubscription, supports broadcast to multiple unknown parties.

**Disadvantages:** Memory leak risk (lapsed listener problem) if not detached properly; can make control flow harder to trace; no built-in delivery guarantees.

**Real-world use case:** YouTube subscriptions/notifications, stock price alerts, UI event listeners (Swing/AWT), reactive programming (RxJava).

**Java example:** `WeatherStation` (Subject) notifying `PhoneDisplay`/`TvDisplay` (Observers) whenever new measurements arrive.

**Interview takeaway:** Be ready to explain the "lapsed listener" memory leak risk unprompted, and to clearly distinguish Observer from Mediator (broadcast vs. centralized coordination) and from full pub-sub messaging systems (in-memory, no delivery guarantees vs. persistent, guaranteed delivery across processes).

**One-line memory trick:** *"YouTube: creators upload, subscribers get notified — the creator never personally calls each subscriber."*

---

*End of Topic 6. Type "Next" to proceed to Topic 7: Decorator Design Pattern.*