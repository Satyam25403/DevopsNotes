# Topic 25: Mediator Design Pattern

---

## 1. Introduction

**Definition:**
The Mediator Pattern is a **behavioral design pattern** that defines an object (the "Mediator") which encapsulates HOW a set of other objects interact with each other — so those objects don't need to reference each other directly, reducing the dependencies between them to just ONE central point.

**Why it exists / what problem it solves:**
Without Mediator, when many objects need to communicate with each other DIRECTLY, you get a "many-to-many" web of dependencies — every object potentially needs a reference to every OTHER object it might need to talk to. As the number of objects grows, the number of potential direct connections grows QUADRATICALLY (n objects can have up to n×(n-1)/2 direct relationships), making the system:
- Extremely tightly coupled — changing one object's interface risks breaking many others that directly reference it
- Hard to understand — the interaction logic is SCATTERED across every participating object, rather than existing in one clear place
- Hard to reuse — an object designed to directly talk to 5 specific other objects can't easily be reused in a different context with different collaborators

Mediator solves this by having ALL objects communicate THROUGH a central Mediator, rather than directly with each other — reducing n-to-n dependencies down to n-to-1 (each object only needs to know about the Mediator).

**When it should be used:**
- When a set of objects communicate in complex, well-defined ways, and you want to centralize that interaction logic in one place
- When objects are TOO tightly coupled to each other, making the system hard to change or reuse
- When you want to reuse an individual object in a DIFFERENT context, but it's currently hardwired to communicate with a fixed set of specific other objects

**When it should NOT be used:**
- When interactions are genuinely simple (just one or two objects communicating) — introducing a Mediator adds unnecessary indirection
- When the Mediator itself starts becoming a "God class" (Topic 1) containing ALL the system's business logic — this is a real risk, discussed in Disadvantages

**Advantages:**
- Reduces coupling from many-to-many down to many-to-one
- Centralizes complex interaction/coordination logic in one understandable place
- Individual "colleague" objects become more reusable, since they no longer depend on each other directly

**Disadvantages:**
- The Mediator itself can become overly complex — a common risk is the Mediator absorbing so much logic that it becomes its own "God class" (Topic 1), simply moving the complexity problem rather than truly solving it
- A single Mediator becomes a central point that ALL communication flows through — if poorly designed, this can become a bottleneck or single point of failure conceptually

---

## 2. Real-World Analogy

Think of an **Air Traffic Control (ATC) tower**.

Imagine if every pilot had to communicate DIRECTLY with every OTHER pilot in the sky to coordinate landings, takeoffs, and flight paths — with dozens of planes in the air, this would require an impractical web of direct pilot-to-pilot radio communications, with massive risk of miscoordination.

Instead, EVERY pilot communicates ONLY with the ATC tower (the Mediator). The tower coordinates who lands when, who takes off when, and what altitude/path each plane should follow — pilots never need to directly coordinate with each other. The tower CENTRALIZES all the complex coordination logic, and each pilot's job is simplified to just "listen to and follow the tower's instructions."

If a NEW plane enters the airspace, it only needs to establish communication with the ONE tower — not with every other plane already in the sky.

---

## 3. Theory

**Core idea:** Instead of "Colleague" objects communicating directly with each other, they all communicate through a shared `Mediator` interface. Each Colleague holds a reference to the Mediator (not to other Colleagues), and the Mediator contains the logic for how messages/events should be routed and coordinated between Colleagues.

**Working mechanism:**
```
Without Mediator (tight coupling, many-to-many):
┌────────┐ ←──→ ┌────────┐
│Colleague A│         │Colleague B│
└───┬────┘         └────┬───┘
    │  ↖                 ↗  │
    │    ╲             ╱    │
    ↓     ╲           ╱     ↓
┌────────┐   ╲       ╱   ┌────────┐
│Colleague D│    ╲     ╱    │Colleague C│
└────────┘     ↖ ╱ ↗     └────────┘
   (every colleague potentially references
    every other colleague directly)

With Mediator (centralized, many-to-one):
┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│Colleague A│  │Colleague B│  │Colleague C│  │Colleague D│
└────┬───┘  └────┬───┘  └────┬───┘  └────┬───┘
     │              │              │              │
     └──────────────┴──────┬───────┴──────────────┘
                            ↓
                    ┌─────────────┐
                    │      Mediator      │
                    │ (coordinates ALL      │
                    │  interactions)          │
                    └─────────────┘
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Mediator | The interface/class centralizing communication logic |
| Concrete Mediator | A specific implementation coordinating a specific set of Colleagues |
| Colleague | An object that communicates through the Mediator, rather than directly with other Colleagues |

**Class responsibilities:** each Colleague is only responsible for its OWN behavior, and for notifying the Mediator when something relevant happens — it delegates ALL cross-object coordination logic to the Mediator.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Mediator                 │
├─────────────────────┤
│ + notify(sender, event): void      │
└──────────┬──────────┘
           △ (implements)
┌──────────┴──────────┐
│      ChatRoomMediator            │
├─────────────────────┤
│ - participants: List<User>  ◇──────┐
│ + addParticipant(User u)              │
│ + notify(sender, event)                 │
└─────────────────────┘              │
                                        │ (aggregation: Mediator
┌─────────────────────┐              │  knows about, but doesn't
│      User (abstract)             │              │  OWN, its participants)
├─────────────────────┤              │
│ # mediator: Mediator    ◇────────────┘ (each User references
│ + send(message)                          the Mediator, NEVER
│ + receive(message)                          other Users directly)
└──────────┬──────────┘
           △ (extends)
┌──────────┴──────────┐
│      ConcreteUser                │
└─────────────────────┘
```

**Relationship explanation:**
- `ChatRoomMediator` **implements** `Mediator`, and holds an **aggregation** relationship with `User` objects (it knows about them, needs to route messages to them, but doesn't control their lifecycle).
- `User` (abstract) holds an **association** with `Mediator` — every `User` knows about the Mediator, but crucially, `User` objects have NO direct reference to OTHER `User` objects.
- When a `ConcreteUser` calls `send(message)`, it goes to `mediator.notify(this, message)` — the Mediator then decides how to route/broadcast this to the OTHER users, without the sending user ever needing to know who those other users are.

---

## 5. Java Implementation

```java
import java.util.ArrayList;
import java.util.List;

// ============================================
// Mediator interface
// ============================================
public interface ChatMediator {
    void sendMessage(String message, User sender);
    void addUser(User user);
}

// ============================================
// Concrete Mediator — centralizes the logic of
// routing messages between all registered Users
// ============================================
public class ChatRoom implements ChatMediator {
    private final List<User> users = new ArrayList<>();

    @Override
    public void addUser(User user) {
        users.add(user);
    }

    @Override
    public void sendMessage(String message, User sender) {
        // The Mediator decides HOW to route this — here, broadcast
        // to everyone EXCEPT the original sender
        for (User user : users) {
            if (user != sender) {
                user.receive(message, sender.getName());
            }
        }
    }
}

// ============================================
// Abstract Colleague — holds a reference to the
// Mediator, NEVER to other Colleagues directly
// ============================================
public abstract class User {
    protected final ChatMediator mediator;
    protected final String name;

    protected User(ChatMediator mediator, String name) {
        this.mediator = mediator;
        this.name = name;
    }

    public String getName() {
        return name;
    }

    public void send(String message) {
        System.out.println(name + " sends: " + message);
        // Delegates ALL routing/coordination to the Mediator —
        // this User has ZERO knowledge of who else is in the chat
        mediator.sendMessage(message, this);
    }

    public abstract void receive(String message, String senderName);
}

// ============================================
// Concrete Colleague
// ============================================
public class ChatUser extends User {
    public ChatUser(ChatMediator mediator, String name) {
        super(mediator, name);
    }

    @Override
    public void receive(String message, String senderName) {
        System.out.println(name + " received from " + senderName + ": " + message);
    }
}

// ============================================
// Demo
// ============================================
public class MediatorDemo {
    public static void main(String[] args) {
        ChatMediator chatRoom = new ChatRoom();

        User alice = new ChatUser(chatRoom, "Alice");
        User bob = new ChatUser(chatRoom, "Bob");
        User charlie = new ChatUser(chatRoom, "Charlie");

        chatRoom.addUser(alice);
        chatRoom.addUser(bob);
        chatRoom.addUser(charlie);

        // Alice sends a message — she NEVER directly references
        // Bob or Charlie; the ChatRoom (Mediator) handles routing
        alice.send("Hello everyone!");

        System.out.println("---");
        bob.send("Hey Alice!");
    }
}
```

**Key line-by-line notes:**
- `User` holds `protected final ChatMediator mediator` — NOT a list of other `User` objects; this is the structural core of the pattern.
- `send()` calls `mediator.sendMessage(message, this)` — delegating the "who should receive this" decision entirely to the Mediator.
- `ChatRoom.sendMessage()` contains the actual COORDINATION LOGIC (broadcast to everyone except the sender) — this logic lives in exactly ONE place, rather than being duplicated or scattered across every `User`.
- Adding a FOURTH user requires ZERO changes to `ChatUser`, `User`, or existing users — just `chatRoom.addUser(newUser)`.

---

## 6. Dry Run

**Sample input:** `alice.send("Hello everyone!")`

```
1. alice.send("Hello everyone!") called
   → Prints: "Alice sends: Hello everyone!"
   → Calls: mediator.sendMessage("Hello everyone!", alice)
     (mediator here is the SAME chatRoom instance alice
      was constructed with)

2. Inside ChatRoom.sendMessage("Hello everyone!", alice):
   → Loops through users = [alice, bob, charlie]
   → Iteration 1: user = alice
       → Check: user != sender → alice != alice → FALSE
       → SKIPPED (sender doesn't receive their own message)
   → Iteration 2: user = bob
       → Check: bob != alice → TRUE
       → bob.receive("Hello everyone!", "Alice") called
       → Prints: "Bob received from Alice: Hello everyone!"
   → Iteration 3: user = charlie
       → Check: charlie != alice → TRUE
       → charlie.receive("Hello everyone!", "Alice") called
       → Prints: "Charlie received from Alice: Hello everyone!"
```

**What's happening in memory:** `alice` never held a reference to `bob` or `charlie` at any point — the ENTIRE routing decision happened inside `ChatRoom`, which DOES hold references to all three users (needed to actually deliver messages). This is the key structural trade: the Mediator necessarily knows about all Colleagues, but each Colleague only needs to know about the Mediator — reducing the total number of DIRECT reference relationships in the system significantly as the number of Colleagues grows.

---

## 7. Real-World Software Example

- **Air Traffic Control systems** (the analogy, also genuinely how it works in real aviation software architecture) — planes/pilots communicate through the tower, not directly with each other.
- **Chat room / messaging applications** (as implemented above) — a chat room server acts as a Mediator, routing messages between connected clients who never directly reference each other's connections.
- **GUI dialog box coordination**: classic GoF example — a `DialogMediator` might coordinate that "when this dropdown changes, disable that checkbox, and update this text field" — the dropdown, checkbox, and text field UI components don't reference each other directly; they all report events to and receive commands from the mediator.
- **Microservices orchestration/API Gateway patterns**: an API Gateway or orchestrator service can act as a Mediator between multiple downstream microservices, so services don't need direct knowledge of every OTHER service they might indirectly need to coordinate with.

---

## 8. Internal Working

**Object references:** each `User` holds exactly ONE reference (to the `Mediator`) instead of potentially many (to every other Colleague) — this is a pure REFERENCE-COUNT reduction at the memory/object-graph level, which is precisely what reduces coupling.

**Method dispatch:** `mediator.sendMessage(...)` and `user.receive(...)` are both standard polymorphic (dynamic dispatch) calls, exactly as covered in Topic 2 — nothing special happens at the JVM level; the PATTERN's value is entirely in how the object graph/reference structure is DESIGNED, not in any special runtime mechanism.

**Complexity distribution:** it's worth explicitly noting WHERE complexity lives before and after applying Mediator — without it, the COORDINATION LOGIC is scattered in small pieces across every Colleague; WITH Mediator, that same total amount of logic is concentrated in ONE class. Mediator doesn't reduce the TOTAL complexity of the system — it relocates and centralizes it, which is usually (but not always, see Disadvantages) an improvement for maintainability.

---

## 9. Before vs After

**Before (direct many-to-many coupling):**

```java
public class UserBad {
    private final String name;
    private List<UserBad> otherUsers = new ArrayList<>(); // DIRECT references to peers!

    public UserBad(String name) {
        this.name = name;
    }

    public void addPeer(UserBad user) {
        otherUsers.add(user);
    }

    public void send(String message) {
        System.out.println(name + " sends: " + message);
        // Directly iterating and calling EVERY known peer —
        // this user must maintain its OWN list of everyone
        // it needs to talk to
        for (UserBad peer : otherUsers) {
            peer.receive(message, name);
        }
    }

    public void receive(String message, String senderName) {
        System.out.println(name + " received from " + senderName + ": " + message);
    }
}
```

**Problems:**
- EVERY `UserBad` must maintain its OWN list of peers, and these lists must all stay in sync manually (if Alice knows about Bob, does Bob also correctly know about Alice? Easy to get wrong).
- Adding a NEW user means updating EVERY existing user's peer list — an O(n) update operation scattered across n different objects for just ONE new addition.
- `UserBad` cannot be reused in a DIFFERENT chat context without dragging along its specific hardwired peer relationships.

**After (Mediator pattern, as shown in Section 5):**
- Each `User` holds only ONE reference (to the Mediator) — no peer-list synchronization problem exists at all.
- Adding a new user requires ONE call: `chatRoom.addUser(newUser)` — no changes needed to any EXISTING user.
- `ChatUser` objects are more reusable, since they don't hardwire relationships to specific other users.

---

## 10. SOLID Principles Connection

- **SRP**: `ChatRoom` (Mediator) has exactly one responsibility — coordinating message routing; `User` has exactly one responsibility — sending/receiving its OWN messages.
- **OCP**: adding new `User` types, or even changing the ROUTING LOGIC (e.g., private messaging instead of broadcast), requires changes ONLY within the Mediator, not across every Colleague.
- **DIP**: `User` depends on the `ChatMediator` INTERFACE, never on a concrete `ChatRoom` class directly — allowing different Mediator implementations (e.g., a `ModeratedChatRoom` that filters messages) to be substituted without changing `User` at all.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Mediator pattern solve?
2. How does Mediator reduce coupling between objects?
3. Give a real-world example of Mediator besides air traffic control.

**Intermediate:**
4. What's the risk of the Mediator itself becoming overly complex? How would you mitigate this?
5. How does adding a new Colleague object differ in effort between a Mediator-based design and a direct many-to-many design?
6. Does the Mediator pattern eliminate coupling entirely, or does it just RESTRUCTURE where the coupling exists?
   *Answer: It restructures coupling — every Colleague still depends on the Mediator (so coupling isn't literally zero), but it converts many-to-many dependencies into many-to-one, which is a significant net reduction in the total number of direct relationships in the system.*

**Advanced:**
7. **Mediator vs Observer pattern** — precisely distinguish them (this is a nearly-guaranteed interview follow-up after covering both).
   *Answer: Observer (Topic 6) is typically ONE-TO-MANY — a single Subject broadcasts to multiple independent Observers, who don't coordinate with each other or with the Subject beyond receiving notifications. Mediator is typically MANY-TO-MANY, coordinated through a CENTRAL object — Colleagues can both SEND and RECEIVE through the Mediator, and the Mediator often contains genuine DECISION-MAKING/coordination logic (like "if the dropdown changes to X, disable the checkbox"), not just passive broadcasting. In short: Observer is about NOTIFICATION; Mediator is about COORDINATION.*
8. Could Mediator and Observer be used TOGETHER in the same system? Give an example.
   *Answer: Yes — a Mediator itself could internally use Observer to notify its Colleagues of routing decisions, rather than calling their methods directly; this is a common, valid combination, especially if Colleagues need to react to Mediator-driven events without tight coupling to the Mediator's specific method signatures.*
9. How would you prevent the Mediator from becoming a "God class" as more coordination logic accumulates over time?
   *Answer: Consider splitting a single overly-complex Mediator into SEVERAL smaller, more focused Mediators, each handling a specific SUBSET of coordination concerns — or extracting specific coordination RULES into their own Strategy (Topic 5) objects that the Mediator delegates to, rather than keeping all logic inline.*
10. In a microservices architecture, how does an API Gateway or orchestration service relate conceptually to the Mediator pattern?
11. What are the testing implications of the Mediator pattern — is it easier or harder to unit test a Colleague object in isolation compared to the "before" many-to-many design?

---

## 12. Common Mistakes

- **Letting the Mediator absorb ALL business logic**, turning it into an unmanageable "God class" (Topic 1) — Mediator should centralize COORDINATION logic specifically, not become a dumping ground for every piece of unrelated logic in the system.
- **Colleagues still holding direct references to each other "just for convenience," alongside the Mediator reference** — this half-defeats the purpose; if Colleagues bypass the Mediator sometimes, the coupling-reduction benefit is undermined.
- **Confusing Mediator with Facade** (Topic 20, covered elsewhere in this series) — a Facade simplifies a single interface to a SUBSYSTEM for EXTERNAL client convenience; Mediator specifically coordinates INTERNAL communication BETWEEN a set of collaborating objects that are aware of and actively participate in the pattern.
- **Applying Mediator to a genuinely simple 2-object interaction** — the overhead of introducing a Mediator interface and implementation isn't justified when there's no real many-to-many complexity to manage.

---

## 13. Time Complexity

**Broadcasting a message to n Colleagues**: O(n) — the Mediator must iterate through and notify each registered Colleague. **Adding a new Colleague**: O(1) (typically just adding to an internal list) — a significant improvement over the "before" design, which required O(n) updates across EVERY existing object to register a new peer relationship.

---

## 14. Java API Examples

- **`java.util.concurrent.Executor`/`ExecutorService`**: acts somewhat like a Mediator between submitted tasks and the underlying thread pool — tasks don't manage threads directly; they submit to the Executor, which coordinates execution.
- **JMS (Java Message Service) / message brokers**: a message broker acts as a Mediator between producers and consumers, who never communicate directly with each other.
- **Spring's `ApplicationEventPublisher`**: allows different Spring beans to communicate indirectly by publishing/listening to events THROUGH the Spring context, rather than holding direct references to each other — a Mediator-like communication style, blended with Observer-style event listening.
- **GUI frameworks' dialog/form coordination logic**: often implemented as a form-level controller class acting as a Mediator between individual input components.

---

## 15. Practice Problem

Implement a simplified **Air Traffic Control system**: a `ControlTower` (Mediator) coordinating `Airplane` objects (Colleagues). Each `Airplane` should be able to `requestLanding()`, which goes through the tower — the tower should only APPROVE the landing if no other airplane is CURRENTLY landing (maintain a simple "runway busy" boolean state in the tower), otherwise it should queue or deny the request. Demonstrate multiple airplanes requesting landing, with the tower coordinating so only one lands at a time.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a smart home automation system where multiple devices (Thermostat, Lights, Security Camera, Door Lock) need to coordinate behavior — e.g., when the Security Camera detects motion at night, the Lights should turn on and the Thermostat should NOT change; when the Door Lock is engaged for the night, the Thermostat should switch to 'night mode.' Devices should not need direct references to each other."

Think about:
- How would a `SmartHomeMediator` centralize these cross-device coordination RULES, so adding a new rule (e.g., "when all Lights are off for 30 minutes, engage Door Lock automatically") doesn't require modifying any individual device class?
- How might this design avoid becoming an unmanageable single class as MORE devices and MORE coordination rules are added over time (referencing the "God class" risk discussed in Section 12)?

---

## 17. Advanced LLD Scenario

**Design a Ride-Sharing Driver-Rider Matching Coordination System** (like Uber) where:
- Multiple available Drivers and multiple waiting Riders need to be matched efficiently
- Drivers and Riders should NOT hold direct references to each other before a match is confirmed
- The matching logic (nearest driver, highest-rated driver, surge-aware matching — connecting back to Topic 5's Strategy pattern) needs to be centralized and easily modifiable
- Once a match IS confirmed, the Driver and Rider need a way to communicate directly for the duration of that specific ride (e.g., in-app messaging, live location sharing) — WITHOUT reverting to the earlier "many-to-many hardwired" problem for ALL drivers/riders in the system

Consider:
- How a `MatchingMediator` centralizes the COORDINATION of matching Drivers to Riders, using an injected `MatchingStrategy` (Strategy pattern) for the actual matching ALGORITHM, keeping these two concerns cleanly separated
- How, ONCE a match is confirmed, you might create a SEPARATE, narrowly-scoped direct communication channel between JUST that one Driver-Rider pair (rather than exposing ALL drivers to ALL riders) — recognizing that Mediator is most valuable for coordinating the LARGER, many-to-many matching pool, while the post-match 1:1 communication is a fundamentally different, much simpler concern
- How this system's design might evolve at real scale (thousands of simultaneous drivers/riders) — would a SINGLE in-memory Mediator object be practical, or would this naturally push toward a distributed message broker/coordination service (recalling the message queue and event-driven concepts touched on in your SystemDesign notes)?

---

## 18. Summary

**Definition:** Mediator centralizes complex communication/coordination logic between a set of objects, so they communicate through the Mediator rather than directly with each other.

**Intent:** Reduce many-to-many object coupling down to many-to-one, centralizing coordination logic in one understandable place.

**Key classes:** `Mediator` interface, `ConcreteMediator` (coordination logic), `Colleague` (participants communicating through the Mediator).

**Advantages:** Significantly reduced coupling; centralized, easier-to-understand coordination logic; more reusable Colleague objects.

**Disadvantages:** Risk of the Mediator itself becoming an overly complex "God class"; doesn't eliminate coupling, just restructures it.

**Real-world use case:** Air traffic control, chat room servers, GUI dialog coordination, API gateways/orchestration in microservices.

**Java example:** `ChatRoom` (Mediator) coordinating message routing between `ChatUser` (Colleague) instances, none of which reference each other directly.

**Interview takeaway:** Be ready to precisely distinguish Mediator from Observer — Observer is passive one-to-many BROADCAST notification; Mediator is active many-to-many COORDINATION with real decision-making logic centralized in one place. This comparison is asked almost every time both patterns are covered.

**One-line memory trick:** *"Air traffic control: pilots talk to the tower, never directly to each other."*

---

*End of Topic 25. Type "Next" to proceed to Topic 26: Memento Pattern.*