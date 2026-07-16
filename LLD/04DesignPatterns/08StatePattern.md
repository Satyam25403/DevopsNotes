# Topic 13: State Pattern

---

## 1. Introduction

**Definition:**
The State Pattern is a **behavioral design pattern** that allows an object to ALTER its behavior when its INTERNAL STATE changes — the object will appear to change its class. Instead of a single class containing a giant `if-else`/`switch` block checking "what state am I in?" before every operation, the pattern extracts each state into its OWN class implementing a common interface, and the object DELEGATES behavior to whichever State object it currently holds.

**Why it exists / what problem it solves:**
Consider a vending machine: its behavior when you insert a coin is completely different depending on whether it's currently `Idle`, has `CoinInserted`, is `Dispensing`, or is `OutOfStock`. Without this pattern, EVERY method (`insertCoin()`, `selectProduct()`, `dispense()`) would need its own giant conditional checking the CURRENT state, and adding a NEW state means editing EVERY ONE of these conditionals — a maintenance nightmare that violates OCP repeatedly.

State solves this by giving EACH state its own class, implementing the SAME state interface. The vending machine (the "Context") holds a reference to its CURRENT state object and simply delegates each operation to it. Each State class implements only the behavior relevant to ITS state, and is responsible for TRANSITIONING the context to the next state when appropriate.

**When it should be used:**
- When an object's behavior depends heavily on its CURRENT state, and it must change behavior at RUNTIME depending on that state
- When you have LARGE conditional statements that depend on the object's state, and those conditionals appear in MULTIPLE methods
- When state-specific behavior needs to be added/modified FREQUENTLY, and you want new states to be addable without touching existing state logic (good OCP)

**When it should NOT be used:**
- When there are only ONE or TWO simple states with minimal behavioral difference — a simple boolean flag and an `if-else` may be sufficient and simpler
- When states rarely change or the system is simple enough that a state machine adds unnecessary abstraction/ceremony
- When state transitions are extremely simple and unlikely to grow in complexity over time

**Advantages:**
- Eliminates large, repeated conditional blocks scattered across multiple methods
- Each state's logic is ENCAPSULATED in its own class — easy to understand, test, and modify in isolation
- Adding a new state = adding a new class; existing state classes remain unchanged (good OCP)
- Makes state TRANSITIONS explicit and easier to reason about/trace

**Disadvantages:**
- Introduces MORE classes — one per state — which can feel like overkill for simple state machines
- State transition logic can become scattered ACROSS multiple state classes (each state decides its own next state), making it harder to see the ENTIRE state machine at a glance in one place
- If not carefully designed, invalid state transitions might not be caught until runtime

---

## 2. Real-World Analogy

Think of a **traffic light**.

A traffic light's behavior when its timer expires is COMPLETELY different depending on its current state: if it's `Red`, the timer expiring means "switch to Green." If it's `Green`, expiring means "switch to Yellow." If it's `Yellow`, expiring means "switch to Red." The SAME external trigger ("timer expired") produces DIFFERENT behavior depending on the CURRENT internal state — and crucially, each state KNOWS what the next state should be.

You'd never want ONE giant method somewhere checking "if currently red, do X; if currently green, do Y; if currently yellow, do Z" — instead, it's far cleaner to think of "Red," "Green," and "Yellow" as SEPARATE, self-contained behaviors, each responsible for handling the timer-expiry event and deciding the transition.

---

## 3. Theory

**Core idea:** Define a `State` interface with methods for each possible ACTION/EVENT the context can experience. Create a CONCRETE State class per distinct state, each implementing these methods according to how THAT state should behave — including deciding what the NEXT state should be. The Context object holds a reference to its CURRENT State and delegates all relevant calls to it.

**Working mechanism:**
```
Context.request() → currentState.handle(context)
                          │
                    (each State decides its OWN behavior,
                     and may call context.setState(newState)
                     to TRANSITION to a different state)
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Context | The object whose behavior varies by state; holds a reference to its current State |
| State (interface) | Declares the methods representing state-dependent behavior |
| Concrete State | Implements the State interface for ONE specific state, including its OWN transition logic |
| Transition | The act of the Context switching from one Concrete State to another |

**Class responsibilities:** each Concrete State is responsible for (1) implementing the CORRECT behavior for its specific state, and (2) deciding the appropriate NEXT state (often by calling `context.setState(...)`) when an event occurs that should trigger a transition.

**Communication flow:** Context receives a call (e.g., `insertCoin()`) → delegates to `currentState.insertCoin(this)` → the State's implementation runs its logic and MAY change the Context's state by calling `context.setState(newState)`.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      VendingMachineState        │
├─────────────────────┤
│ + insertCoin(ctx): void    │
│ + selectProduct(ctx): void │
│ + dispense(ctx): void       │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼───────────┬───────────────┐
   │                          │                     │
┌──┴──────┐  ┌──┴──────────┐  ┌──┴──────────┐
│ IdleState        │  │ HasCoinState        │  │ SoldOutState        │
└──────────┘  └──────────────┘  └──────────────┘

┌─────────────────────┐
│      VendingMachine (Context)  │
├─────────────────────┤
│ - currentState: VendingMachineState  ◇──→ (aggregation: Context
├─────────────────────┤         holds ONE current State
│ + setState(State)                  reference, swapped at
│ + insertCoin()                     runtime during transitions)
│ + selectProduct()
│ + dispense()
└─────────────────────┘
```

**Relationship explanation:**
- `IdleState`, `HasCoinState`, `SoldOutState` all **implement** `VendingMachineState` — each provides its OWN behavior for the SAME set of operations.
- `VendingMachine` (the Context) **aggregates** a `VendingMachineState` — it holds exactly ONE current state reference at any given time, and this reference is REASSIGNED (not recreated as a new Context) during transitions.
- Each Concrete State typically has a **dependency** on the Context type (passed as a parameter, e.g., `insertCoin(VendingMachine ctx)`) so it can call `ctx.setState(...)` to trigger a transition — this is how transitions are decentralized across State classes rather than centralized in the Context.

---

## 5. Java Implementation

```java
// ============================================
// State interface — declares state-dependent behavior
// ============================================
public interface VendingMachineState {
    void insertCoin(VendingMachine machine);
    void selectProduct(VendingMachine machine);
    void dispense(VendingMachine machine);
}

// ============================================
// Concrete State — machine is idle, waiting for a coin
// ============================================
public class IdleState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin inserted. Please select a product.");
        // Transition: Idle -> HasCoin
        machine.setState(machine.getHasCoinState());
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Insert a coin first.");
        // No transition — stays Idle
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Insert a coin and select a product first.");
    }
}

// ============================================
// Concrete State — coin inserted, waiting for product selection
// ============================================
public class HasCoinState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Coin already inserted. Please select a product.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Product selected. Dispensing...");
        // Transition: HasCoin -> Dispensing (represented directly
        // by calling dispense() as the natural next step here)
        machine.setState(machine.getDispensingState());
        machine.dispense(); // immediately proceed to dispensing logic
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first.");
    }
}

// ============================================
// Concrete State — actively dispensing a product
// ============================================
public class DispensingState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Please wait, dispensing in progress.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Please wait, dispensing in progress.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        machine.decrementStock();
        System.out.println("Product dispensed. Thank you!");
        // Transition back to Idle (or SoldOut if stock ran out)
        if (machine.getStock() > 0) {
            machine.setState(machine.getIdleState());
        } else {
            machine.setState(machine.getSoldOutState());
        }
    }
}

// ============================================
// Concrete State — no stock remaining
// ============================================
public class SoldOutState implements VendingMachineState {
    @Override
    public void insertCoin(VendingMachine machine) {
        System.out.println("Sorry, machine is sold out. Coin returned.");
    }

    @Override
    public void selectProduct(VendingMachine machine) {
        System.out.println("Sorry, machine is sold out.");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Cannot dispense — sold out.");
    }
}

// ============================================
// Context — delegates to its current state,
// and holds shared state instances + stock count
// ============================================
public class VendingMachine {
    // States are typically created ONCE and reused (each is stateless
    // itself, so this is safe and efficient — similar to Topic 12's
    // Null Object reuse idea)
    private final VendingMachineState idleState = new IdleState();
    private final VendingMachineState hasCoinState = new HasCoinState();
    private final VendingMachineState dispensingState = new DispensingState();
    private final VendingMachineState soldOutState = new SoldOutState();

    private VendingMachineState currentState;
    private int stock;

    public VendingMachine(int initialStock) {
        this.stock = initialStock;
        // Start in Idle, unless there's no stock to begin with
        this.currentState = (stock > 0) ? idleState : soldOutState;
    }

    // Called by Concrete States to TRANSITION the machine
    public void setState(VendingMachineState state) {
        this.currentState = state;
    }

    // Delegating methods — Context doesn't implement any
    // state-specific LOGIC itself, just forwards to currentState
    public void insertCoin() {
        currentState.insertCoin(this);
    }

    public void selectProduct() {
        currentState.selectProduct(this);
    }

    public void dispense() {
        currentState.dispense(this);
    }

    public void decrementStock() {
        stock--;
    }

    public int getStock() {
        return stock;
    }

    // Getters for shared state instances (used by Concrete States
    // to reference each other for transitions)
    public VendingMachineState getIdleState() { return idleState; }
    public VendingMachineState getHasCoinState() { return hasCoinState; }
    public VendingMachineState getDispensingState() { return dispensingState; }
    public VendingMachineState getSoldOutState() { return soldOutState; }
}

// ============================================
// Demo
// ============================================
public class StatePatternDemo {
    public static void main(String[] args) {
        VendingMachine machine = new VendingMachine(1); // only 1 item in stock

        machine.insertCoin();      // Idle -> HasCoin
        machine.selectProduct();   // HasCoin -> Dispensing -> Idle/SoldOut

        // Now sold out (stock was 1, now 0)
        machine.insertCoin();      // SoldOut behavior: coin returned
    }
}
```

**Key line-by-line notes:**
- `VendingMachine.insertCoin()`, `selectProduct()`, and `dispense()` all simply DELEGATE to `currentState` — none of them contain any `if-else` checking WHICH state the machine is in; the polymorphic dispatch handles that entirely.
- Each Concrete State's methods decide, INDEPENDENTLY, whether a transition should happen — e.g., `IdleState.insertCoin()` transitions to `HasCoinState`, while `IdleState.selectProduct()` does NOT transition (stays Idle).
- State instances (`idleState`, `hasCoinState`, etc.) are created ONCE inside `VendingMachine` and REUSED across transitions — since the states themselves hold no per-transaction data, this avoids unnecessary object churn (similar reasoning to Null Object's singleton-style reuse in Topic 12).

---

## 6. Dry Run

**Sample input:**
```java
VendingMachine machine = new VendingMachine(1);
machine.insertCoin();
machine.selectProduct();
machine.insertCoin();
```

```
1. new VendingMachine(1)
   → stock = 1
   → currentState = idleState (since stock > 0)

2. machine.insertCoin() called
   → Delegates to currentState.insertCoin(machine)
   → currentState is IdleState → IdleState.insertCoin() executes
   → Prints: "Coin inserted. Please select a product."
   → machine.setState(machine.getHasCoinState())
   → currentState is now hasCoinState

3. machine.selectProduct() called
   → Delegates to currentState.selectProduct(machine)
   → currentState is HasCoinState → HasCoinState.selectProduct() executes
   → Prints: "Product selected. Dispensing..."
   → machine.setState(machine.getDispensingState())
   → currentState is now dispensingState
   → machine.dispense() called immediately
       → Delegates to currentState.dispense(machine)
       → currentState is DispensingState → DispensingState.dispense() executes
       → machine.decrementStock() → stock becomes 0
       → Prints: "Product dispensed. Thank you!"
       → machine.getStock() > 0? NO (stock == 0)
       → machine.setState(machine.getSoldOutState())
       → currentState is now soldOutState

4. machine.insertCoin() called
   → Delegates to currentState.insertCoin(machine)
   → currentState is SoldOutState → SoldOutState.insertCoin() executes
   → Prints: "Sorry, machine is sold out. Coin returned."
```

**What's happening in memory:** `VendingMachine`'s `currentState` field is simply REASSIGNED to point to different pre-existing State objects over time — no new State objects are created during transitions (they were all created once, upfront, in the constructor); only the REFERENCE changes, and each subsequent call to `insertCoin()`/`selectProduct()`/`dispense()` dynamically dispatches to whichever Concrete State `currentState` currently references.

---

## 7. Real-World Software Example

- **Order processing systems**: an `Order` object transitioning through `Placed` → `Shipped` → `Delivered` → `Cancelled` states, with each state defining which operations are valid (e.g., you can't "ship" an already-`Delivered` order).
- **TCP connection state machines**: `Closed` → `Listen` → `SynReceived` → `Established` → `Closed`, where each state has very different valid operations and transitions — a textbook example of a formal state machine.
- **Media player state**: `Playing`, `Paused`, `Stopped` states, where pressing "play" behaves differently depending on the CURRENT state (resume from Paused, versus start fresh from Stopped).
- **Document workflow/approval systems**: `Draft` → `UnderReview` → `Approved`/`Rejected`, where each state restricts which actions are currently valid.
- **Game character states**: `Idle`, `Running`, `Jumping`, `Attacking` — each state defining different valid transitions and behaviors for the SAME input events (e.g., pressing "jump" while `Attacking` may do nothing, while pressing it while `Idle` triggers a jump).

---

## 8. Internal Working

**Object creation:** Concrete State instances are typically created ONCE (often at Context construction time, as shown in Section 5) and REUSED across all transitions, since most states are STATELESS themselves (holding no per-instance data beyond perhaps references needed for transitions).

**Runtime interactions / call flow:** every Context method call is a SINGLE indirection — `context.operation()` → `currentState.operation(context)` — with the ACTUAL logic and any transition decision happening entirely inside the Concrete State's method body.

**Memory usage:** O(1) additional memory for the Context's `currentState` reference; if states are shared/reused (as in Section 5), the total memory for all possible states is fixed regardless of how MANY transitions occur over the object's lifetime.

**Dynamic binding:** the SAME call site (`currentState.insertCoin(this)`) resolves to DIFFERENT Concrete State implementations depending purely on which object `currentState` currently references — this is the exact mechanism (dynamic dispatch) that eliminates the need for explicit `if-else`/`switch` state checks.

---

## 9. Before vs After

**Before (no State pattern — one giant conditional per method):**

```java
public class VendingMachineBad {
    private enum State { IDLE, HAS_COIN, DISPENSING, SOLD_OUT }
    private State state = State.IDLE;
    private int stock = 1;

    public void insertCoin() {
        // EVERY method needs its OWN switch over the SAME states —
        // adding a new state means editing ALL of these methods
        switch (state) {
            case IDLE:
                System.out.println("Coin inserted.");
                state = State.HAS_COIN;
                break;
            case HAS_COIN:
                System.out.println("Coin already inserted.");
                break;
            case DISPENSING:
                System.out.println("Please wait, dispensing.");
                break;
            case SOLD_OUT:
                System.out.println("Sold out. Coin returned.");
                break;
        }
    }

    public void selectProduct() {
        // ANOTHER switch over the SAME states, duplicated logic structure
        switch (state) {
            case IDLE:
                System.out.println("Insert a coin first.");
                break;
            case HAS_COIN:
                System.out.println("Dispensing...");
                state = State.DISPENSING;
                // ... dispense logic inlined here too
                break;
            // ... more cases
        }
    }
    // dispense() would need YET ANOTHER switch over the same states
}
```

**Problems:**
- The SAME state-checking logic (`switch (state)`) is DUPLICATED across every method — `insertCoin()`, `selectProduct()`, `dispense()` all need their own copy.
- Adding a NEW state (e.g., `MAINTENANCE`) means editing EVERY SINGLE method's switch statement — high risk of forgetting one, and violates OCP severely.
- State-specific logic for a SINGLE state (e.g., everything `HAS_COIN` needs to do) is SCATTERED across multiple methods rather than being cohesively grouped in one place.

**After (State pattern, as shown in Section 5):**
- Each state's ENTIRE behavior (across `insertCoin()`, `selectProduct()`, `dispense()`) is COHESIVELY grouped in ONE Concrete State class.
- Adding a new state (e.g., `MaintenanceState`) means writing ONE new class implementing the interface — zero changes needed to existing state classes or the Context's delegating methods.

---

## 10. SOLID Principles Connection

- **SRP**: each Concrete State class has exactly ONE responsibility — defining behavior for its specific state, including its own transition logic.
- **OCP**: adding a new state requires a new class implementing `VendingMachineState` — no existing state class or the Context's core delegating methods need modification.
- **DIP**: `VendingMachine` depends only on the `VendingMachineState` abstraction for its `currentState` field — never on specific concrete state classes directly (aside from initial construction/getters).
- **LSP**: any `VendingMachineState` implementation can be substituted as `currentState` without breaking the Context's delegating logic — the Context's method bodies work identically regardless of WHICH concrete state is currently active.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the State pattern solve compared to using an `enum` with `switch` statements scattered across multiple methods?
2. In the vending machine example, which class decides WHEN to transition from one state to another?
3. Why is behavior said to be "delegated" rather than "conditioned" in the State pattern?

**Intermediate:**
4. Why are Concrete State instances often reused/shared (like Singletons) rather than created fresh on every transition?
5. How would you prevent an INVALID transition (e.g., calling `dispense()` directly while in `IdleState`) from causing incorrect behavior?
6. **State vs Strategy** — precisely distinguish them (an almost-guaranteed interview question, given both were covered in this series — Strategy in Topic 5).
   *Answer: Structurally, State and Strategy look nearly IDENTICAL (both involve a Context holding a reference to an interface, delegating behavior to the current implementation). The difference is INTENT and WHO controls the current implementation: in Strategy, the CLIENT explicitly CHOOSES and injects a strategy, and it typically does NOT change on its own during the object's lifecycle. In State, the STATE OBJECTS THEMSELVES often decide to TRANSITION the Context to a different state as a natural consequence of behavior — the Context's "current implementation" changes AUTOMATICALLY over time as part of the object's natural lifecycle, driven from WITHIN the pattern rather than by external client choice.*

**Advanced:**
7. How would you implement a State pattern where TRANSITION logic is centralized in the Context (rather than decentralized across State classes, as shown in Section 5)? What are the tradeoffs of each approach?
8. Discuss how the State pattern relates to formal Finite State Machines (FSMs) — could you generate State pattern code automatically from an FSM diagram/specification?
9. How would you handle a state that needs to perform SETUP logic (e.g., start a timer) when ENTERING it, and CLEANUP logic when LEAVING it? (Hint: consider adding `onEnter()`/`onExit()` hooks to the State interface.)
10. Could the State pattern be combined with the Observer pattern (Topic 6) so that OTHER parts of the system are notified whenever the Context transitions states? Describe how you'd design this.

---

## 12. Common Mistakes

- **Letting the Context itself contain state-checking conditionals ANYWAY** (e.g., `if (currentState instanceof HasCoinState)`) — this defeats the entire purpose of the pattern; all state-specific logic should live INSIDE the Concrete State classes, not be re-introduced in the Context via type checks.
- **Forgetting to reset/transition state correctly on ERROR paths** — e.g., if `dispense()` fails partway through (out of stock discovered mid-dispense), forgetting to transition back to an appropriate state can leave the Context stuck.
- **Over-applying State pattern to trivial two-state scenarios** — a simple boolean flag with an `if-else` may be entirely sufficient when there are only two simple states with minimal behavioral difference.
- **Scattering transition logic so widely across states that the OVERALL state machine becomes hard to visualize** — for complex state machines, it's often worth ALSO maintaining a separate diagram/documentation of the full transition graph, since the code itself doesn't show the "big picture" in one place.

---

## 13. Time Complexity

- **Time:** O(1) per operation — a single virtual dispatch to the current state's method, regardless of how many total states exist.
- **Space:** O(n) for n total possible states (each Concrete State instance, typically created once and reused) + O(1) for the Context's `currentState` reference.

---

## 14. Java API Examples

- **`java.util.concurrent.Future`**: internally manages state transitions (e.g., pending, completed, cancelled) that affect the behavior of methods like `get()` and `cancel()`.
- **Thread states in the JVM** (`NEW`, `RUNNABLE`, `BLOCKED`, `WAITING`, `TERMINATED`, as seen via `Thread.getState()`): conceptually mirror State pattern's idea, even though the JVM's internal implementation isn't necessarily a textbook GoF State pattern.
- **javax.faces / JSF UI component lifecycle**: components pass through defined LIFECYCLE states (e.g., `RESTORE_VIEW`, `APPLY_REQUEST_VALUES`, `RENDER_RESPONSE`), each with distinct behavior — conceptually related to formal state machines.
- **Spring Statemachine library**: an explicit framework for building State-pattern-style state machines in Spring applications, directly implementing these concepts for workflow/order-processing systems.

---

## 15. Practice Problem

Implement a **State pattern for a Document Approval Workflow**: create a `DocumentState` interface with methods `submitForReview()`, `approve()`, and `reject()`. Implement `DraftState`, `UnderReviewState`, `ApprovedState`, and `RejectedState`, where: `DraftState.submitForReview()` transitions to `UnderReviewState`; `UnderReviewState.approve()` transitions to `ApprovedState`; `UnderReviewState.reject()` transitions to `RejectedState`; and calling any INVALID operation for the current state (e.g., trying to `approve()` a `Draft` document) prints an appropriate "not allowed" message. Demonstrate the full lifecycle of a document from Draft to Approved.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **ride-sharing trip lifecycle** using the State pattern: a `Trip` moves through `Requested` → `DriverAssigned` → `InProgress` → `Completed` states (with a possible `Cancelled` state reachable from `Requested` or `DriverAssigned`, but NOT from `InProgress` or `Completed`). Design the State classes such that invalid transitions (e.g., cancelling an already-`Completed` trip) are handled gracefully."

Think about:
- How you'd represent the RESTRICTION that `Cancelled` is only reachable from CERTAIN states, not all of them — does each state simply not implement a `cancel()` transition (i.e., its `cancel()` method does nothing/prints an error)?
- Whether ANY additional data (e.g., a cancellation reason, or a driver's ID once assigned) needs to be tracked, and where that data should LIVE — in the Context, or passed into State methods?

---

## 17. Advanced LLD Scenario

**Design an ATM Machine's Complete Transaction Flow** using the State pattern, where:
- The ATM moves through states like `Idle` (no card inserted) → `CardInserted` (awaiting PIN) → `PinValidated` (awaiting transaction type) → `Dispensing` (dispensing cash) → back to `Idle`, with error states like `IncorrectPin` (allowing limited retries before `CardRetained`) and `InsufficientFunds`
- Different operations (`insertCard()`, `enterPin()`, `selectTransaction()`, `withdrawCash()`, `ejectCard()`) should behave COMPLETELY differently depending on the CURRENT state — e.g., `enterPin()` only makes sense in `CardInserted` state, and should be REJECTED (or ignored) in `Idle` state
- Consider the interaction between State pattern and PIN-retry-limit logic — where should the "3 incorrect attempts" COUNTER live: in the Context (ATM), or duplicated/passed through the `IncorrectPin` state?

Consider:
- How this scenario's complexity (multiple valid AND invalid transitions, error-handling states, retry logic) demonstrates the State pattern's real value over ad-hoc conditionals — the SHEER NUMBER of distinct states here would make a single giant switch-based implementation extremely unwieldy
- How you'd represent the "3 incorrect PIN attempts leads to CardRetained" rule cleanly — this requires the Context (or the State itself) to track SOME piece of mutable data (an attempt counter) alongside the pure state transition logic, showing that real-world State pattern implementations often need a BIT of state-related data beyond just "which state am I in"
- Why this scenario, similar to the vending machine, benefits from EACH state deciding its OWN specific next-state transitions, rather than centralizing ALL transition logic in the ATM class itself

---

## 18. Summary

**Definition:** State allows an object to alter its behavior when its internal state changes, by delegating state-dependent behavior to interchangeable Concrete State objects implementing a common interface.

**Intent:** Eliminate large conditional blocks scattered across multiple methods by encapsulating each state's behavior (and its own transition logic) into a separate, cohesive class.

**Key classes:** `Context` (holds current State reference, delegates operations), `State` interface (declares state-dependent methods), `ConcreteState` implementations (one per distinct state, including transition logic).

**Advantages:** Eliminates repeated conditionals; cohesive, encapsulated state-specific logic; easy to add new states (good OCP); explicit, traceable transitions.

**Disadvantages:** More classes to manage; transition logic can be scattered across multiple state classes; risk of unhandled invalid transitions if not carefully designed.

**Real-world use case:** Order lifecycle management, TCP connection states, media player playback states, document approval workflows, ATM/vending machine transaction flows.

**Java example:** `VendingMachine` delegating to `IdleState`/`HasCoinState`/`DispensingState`/`SoldOutState`, each deciding its own transitions.

**Interview takeaway:** Be ready to clearly distinguish State from Strategy — nearly identical structure, but State involves the CURRENT implementation changing AUTOMATICALLY over the object's lifecycle (driven internally), while Strategy involves the CLIENT explicitly choosing and injecting a fixed algorithm (driven externally).

**One-line memory trick:** *"A traffic light's behavior when the timer expires depends entirely on whether it's currently Red, Green, or Yellow — same trigger, different behavior, based on current state."*

---

*End of Topic 13. Type "Next" to proceed to Topic 14: Composite Pattern.*