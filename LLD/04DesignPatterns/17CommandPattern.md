# Topic 22: Command Pattern

---

## 1. Introduction

**Definition:**
The Command Pattern is a **behavioral design pattern** that ENCAPSULATES a REQUEST (an action, along with all the data needed to perform it) as a STANDALONE OBJECT — this allows you to PARAMETERIZE clients with different requests, QUEUE or LOG requests, and support UNDOABLE operations, by treating "an action to be performed" as a first-class OBJECT rather than a direct method call.

**Why it exists / what problem it solves:**
Consider a remote control with programmable buttons, or a text editor supporting UNDO/REDO. In BOTH cases, you need to represent "an action to perform" as something that can be STORED, PASSED AROUND, QUEUED for LATER execution, or REVERSED — a DIRECT method call (`light.turnOn()`) doesn't give you any of this flexibility, because the ACTION and its INVOCATION are fused together at the call site, with no way to hold onto "the fact that this action should happen" as a separate, manipulable THING.

Command solves this by wrapping EACH action in its OWN object implementing a common `Command` interface (typically with an `execute()` method, and OFTEN an `undo()` method). The INVOKER (e.g., a remote control button) doesn't need to know WHAT the command actually DOES — it just calls `execute()` on WHATEVER `Command` object it's currently holding, fully DECOUPLING the button (invoker) from the SPECIFIC action and its receiver.

**When it should be used:**
- When you need to PARAMETERIZE objects with an ACTION to perform, decided at RUNTIME (e.g., configurable buttons, menu items)
- When you need to SUPPORT UNDO/REDO functionality — each Command can store what's needed to REVERSE its own effect
- When you need to QUEUE, LOG, or SCHEDULE requests for LATER or DEFERRED execution (e.g., a job queue, a macro-recording feature)
- When you want to DECOUPLE the object that INVOKES an action from the object that actually KNOWS HOW to perform it

**When it should NOT be used:**
- When actions are SIMPLE, DIRECT, and don't need to be QUEUED, LOGGED, UNDONE, or PARAMETERIZED — a plain method call is simpler and sufficient
- When there's no genuine need to TREAT an action as an OBJECT — introducing Command purely out of habit adds unnecessary INDIRECTION and boilerplate
- When UNDO/REDO or DEFERRED execution are NOT actual requirements — the EXTRA class-per-action overhead isn't justified without these needs

**Advantages:**
- Decouples the INVOKER from the RECEIVER and the SPECIFIC action being performed
- Naturally supports UNDO/REDO by having EACH Command remember how to REVERSE itself
- Commands can be QUEUED, LOGGED, or even SERIALIZED for later/deferred execution
- New commands can be added WITHOUT modifying the INVOKER (good OCP)

**Disadvantages:**
- Introduces an ADDITIONAL class PER distinct action/command type — can lead to MANY small classes for a system with MANY distinct actions
- UNDO support, in particular, can add REAL complexity — each Command must carefully track WHATEVER state is needed to properly REVERSE its effect
- Can feel like OVERKILL for simple, direct, one-off actions with no need for queuing/undo/logging

---

## 2. Real-World Analogy

Think of a **restaurant order slip**.

When you order food, the WAITER doesn't personally RUN to the kitchen and cook your meal — instead, they WRITE DOWN your order on a SLIP (encapsulating "what you want" as a standalone, PORTABLE object) and hand it to the KITCHEN. The waiter (the INVOKER) doesn't need to know HOW to cook ANYTHING — they just need to know how to CREATE and HAND OFF an order slip. The KITCHEN (the RECEIVER) is the one that actually KNOWS HOW to execute the order. Critically, the SLIP can be QUEUED (placed in a line of orders to prepare), and in SOME restaurants, an order can even be CANCELLED/MODIFIED before it's prepared — mirroring UNDO functionality.

---

## 3. Theory

**Core idea:** Define a `Command` interface with an `execute()` method (and OPTIONALLY `undo()`). EACH CONCRETE Command class WRAPS a specific ACTION, holding a REFERENCE to the "RECEIVER" (the object that actually KNOWS HOW to perform the action) plus WHATEVER parameters are needed. An "INVOKER" holds a `Command` reference and calls `execute()` on it WITHOUT knowing anything about the SPECIFIC action or receiver involved.

**Working mechanism:**
```
Invoker.execute() → command.execute() → receiver.action()
                                      (the Command DELEGATES
                                       the ACTUAL work to its
                                       held Receiver reference)
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Command | The interface declaring `execute()` (and often `undo()`) |
| Concrete Command | A specific action, holding a reference to its Receiver + needed parameters |
| Receiver | The object that actually KNOWS HOW to perform the requested action |
| Invoker | Holds and TRIGGERS a Command, without knowing its specific implementation |
| Client | Creates Concrete Command instances and CONFIGURES them with the appropriate Receiver |

**Class responsibilities:** each Concrete Command is responsible for KNOWING WHICH Receiver method(s) to call, and with WHAT parameters, to perform ITS specific action — and, if UNDO is supported, for REMEMBERING whatever STATE is needed to REVERSE that action later.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      Command                     │
├─────────────────────┤
│ + execute(): void             │
│ + undo(): void                 │
└──────────┬──────────┘
           △ (implements)
           │
┌──────────┴───────────────┐
│ LightOnCommand                  │
├──────────────────────────┤
│ - light: Light           ◇──→ (association: Command
├──────────────────────────┤   holds a reference to
│ + execute()                       its RECEIVER)
│   { light.turnOn(); }
│ + undo()
│   { light.turnOff(); }
└──────────────────────────┘

┌─────────────────────┐
│      Light (Receiver)           │
├─────────────────────┤
│ + turnOn(): void               │
│ + turnOff(): void              │
└─────────────────────┘

┌─────────────────────┐
│      RemoteControl (Invoker)    │
├─────────────────────┤
│ - command: Command    ◇──────→ (aggregation: Invoker
├─────────────────────┤       holds a Command WITHOUT
│ + setCommand(Command)             knowing its concrete
│ + pressButton(): void              type or the RECEIVER
│ + pressUndo(): void                behind it)
└─────────────────────┘
```

**Relationship explanation:**
- `LightOnCommand` **implements** `Command` and has an **association** with `Light` (the RECEIVER) — it holds a REFERENCE and DELEGATES the ACTUAL work to it.
- `RemoteControl` (the INVOKER) **aggregates** a `Command` reference — critically, it has NO KNOWLEDGE of `Light` or `LightOnCommand` SPECIFICALLY; it just calls `command.execute()`/`command.undo()` on WHATEVER `Command` it's currently holding.
- This THREE-WAY separation (Invoker, Command, Receiver) is what allows the SAME `RemoteControl` to trigger COMPLETELY DIFFERENT actions (turning on a light, playing music, opening a garage door) simply by being CONFIGURED with a DIFFERENT `Command` object — no changes needed to `RemoteControl` itself.

---

## 5. Java Implementation

```java
// ============================================
// Command interface
// ============================================
public interface Command {
    void execute();
    void undo();
}

// ============================================
// Receiver — the object that actually KNOWS HOW
// to perform the requested action
// ============================================
public class Light {
    private final String location;

    public Light(String location) {
        this.location = location;
    }

    public void turnOn() {
        System.out.println(location + " light: ON");
    }

    public void turnOff() {
        System.out.println(location + " light: OFF");
    }
}

// ============================================
// Concrete Commands — each wraps ONE specific action,
// holding a reference to its Receiver
// ============================================
public class LightOnCommand implements Command {
    private final Light light;

    public LightOnCommand(Light light) {
        this.light = light;
    }

    @Override
    public void execute() {
        light.turnOn();
    }

    @Override
    public void undo() {
        // UNDO reverses THIS command's specific effect
        light.turnOff();
    }
}

public class LightOffCommand implements Command {
    private final Light light;

    public LightOffCommand(Light light) {
        this.light = light;
    }

    @Override
    public void execute() {
        light.turnOff();
    }

    @Override
    public void undo() {
        light.turnOn();
    }
}

// ============================================
// Invoker — holds and TRIGGERS a Command,
// without knowing its specific implementation
// ============================================
public class RemoteControl {
    private Command currentCommand;
    // Keeps track of the LAST executed command, to support undo
    private Command lastCommand;

    public void setCommand(Command command) {
        this.currentCommand = command;
    }

    public void pressButton() {
        currentCommand.execute();
        lastCommand = currentCommand; // remember for potential undo
    }

    public void pressUndo() {
        if (lastCommand != null) {
            lastCommand.undo();
            lastCommand = null; // simple single-level undo
        } else {
            System.out.println("Nothing to undo.");
        }
    }
}

// ============================================
// Demo
// ============================================
public class CommandDemo {
    public static void main(String[] args) {
        Light livingRoomLight = new Light("Living Room");

        Command lightOn = new LightOnCommand(livingRoomLight);
        Command lightOff = new LightOffCommand(livingRoomLight);

        RemoteControl remote = new RemoteControl();

        remote.setCommand(lightOn);
        remote.pressButton(); // Living Room light: ON

        remote.pressUndo();   // Living Room light: OFF (undo of "on")

        remote.setCommand(lightOff);
        remote.pressButton(); // Living Room light: OFF

        remote.pressUndo();   // Living Room light: ON (undo of "off")
    }
}
```

**Key line-by-line notes:**
- `RemoteControl` never references `Light`, `LightOnCommand`, or `LightOffCommand` DIRECTLY — it only knows the `Command` INTERFACE, meaning it could JUST AS EASILY be configured to control a garage door, a stereo, or ANYTHING else implementing `Command`.
- Each Concrete Command's `undo()` implements the PRECISE INVERSE of its `execute()` — `LightOnCommand.undo()` calls `turnOff()`, while `LightOffCommand.undo()` calls `turnOn()` — this pairing is what makes UNDO functionality possible.
- `RemoteControl.lastCommand` provides a SIMPLE, single-level UNDO — a more SOPHISTICATED implementation might use a STACK of executed commands to support MULTIPLE levels of undo/redo (explored further in Section 16).

---

## 6. Dry Run

**Sample input:**
```java
remote.setCommand(lightOn);
remote.pressButton();
remote.pressUndo();
```

```
1. remote.setCommand(lightOn)
   → remote.currentCommand = lightOn (a LightOnCommand instance)

2. remote.pressButton() called
   → currentCommand.execute() called
       → Dynamic dispatch resolves to LightOnCommand.execute()
       → light.turnOn() called → Prints: "Living Room light: ON"
   → remote.lastCommand = lightOn (saved for potential undo)

3. remote.pressUndo() called
   → lastCommand is NOT null (it's lightOn)
   → lastCommand.undo() called
       → Dynamic dispatch resolves to LightOnCommand.undo()
       → light.turnOff() called → Prints: "Living Room light: OFF"
   → remote.lastCommand = null
```

**What's happening in memory:** `RemoteControl` holds ONLY a REFERENCE to a `Command` object (`currentCommand`/`lastCommand`) — it has NO knowledge of `Light`'s existence. When `execute()`/`undo()` is called, DYNAMIC DISPATCH resolves to the SPECIFIC Concrete Command's implementation, which THEN makes its OWN call to its held `Light` REFERENCE — this is a TWO-LEVEL delegation: Invoker → Command → Receiver.

---

## 7. Real-World Software Example

- **GUI menu items / toolbar buttons**: EACH menu item (e.g., "Cut," "Copy," "Paste") is often implemented as a `Command` object, allowing the SAME action to be triggered from MULTIPLE UI elements (a menu item, a toolbar button, a keyboard shortcut) WITHOUT duplicating the action's logic.
- **Undo/Redo in text editors / IDEs**: EVERY edit operation (insert text, delete text, format) is often represented as a `Command`, STORED in a HISTORY STACK, enabling UNDO by calling `undo()` on the MOST RECENT command, and REDO by RE-executing it.
- **Job queues / task scheduling systems**: representing EACH "job to be run" as a `Command` object that can be QUEUED, PERSISTED, and executed LATER (possibly on a DIFFERENT thread/machine) — e.g., `Runnable` in Java IS essentially a Command.
- **Transactional systems / database operations**: representing EACH database operation as a `Command` that can be LOGGED, and potentially ROLLED BACK (undone) if a transaction FAILS.
- **Macro recording**: recording a SEQUENCE of user actions as a LIST of `Command` objects, which can LATER be REPLAYED (re-executed) as a single "macro."

---

## 8. Internal Working

**Object creation:** Concrete Command objects are typically created and CONFIGURED (with their Receiver + any needed parameters) by the CLIENT, then HANDED to the Invoker — the Invoker itself doesn't create Commands, it simply RECEIVES and TRIGGERS them.

**Runtime interactions / call flow:** the Invoker calls `command.execute()` — a SINGLE virtual dispatch resolves to the SPECIFIC Concrete Command, WHICH THEN calls its OWN Receiver's method(s) — this is a TWO-HOP delegation chain (Invoker → Command → Receiver), with the Invoker COMPLETELY UNAWARE of the SECOND hop.

**Undo mechanism internals:** for UNDO to work correctly, EACH Concrete Command must REMEMBER whatever STATE is necessary to REVERSE its effect — for SIMPLE actions (like turning a light on/off), this might just mean KNOWING the INVERSE action; for MORE COMPLEX actions (like "replace this text"), the Command might need to STORE the PREVIOUS value BEFORE the change, so `undo()` can RESTORE it.

**Command History / Stack:** supporting MULTIPLE levels of undo/redo typically requires the Invoker (or a SEPARATE "CommandManager") to maintain a STACK (or TWO stacks — one for undo history, one for redo history) of EXECUTED commands, rather than tracking just the SINGLE `lastCommand` shown in Section 5's simplified example.

---

## 9. Before vs After

**Before (no Command — direct coupling between Invoker and Receiver):**

```java
public class RemoteControlBad {
    private Light light;

    public RemoteControlBad(Light light) {
        this.light = light;
    }

    public void pressButton(String action) {
        // The Invoker DIRECTLY knows about Light AND has to
        // interpret WHAT ACTION to perform via string/enum checks —
        // tightly coupled, and NOT reusable for OTHER kinds of devices
        if (action.equals("ON")) {
            light.turnOn();
        } else if (action.equals("OFF")) {
            light.turnOff();
        }
        // Adding support for a GARAGE DOOR or STEREO would require
        // EDITING this method, adding MORE conditional branches,
        // and the RemoteControl becomes tightly coupled to EVERY
        // device type it might ever need to control
    }
}
```

**Problems:**
- `RemoteControlBad` is TIGHTLY coupled to `Light` SPECIFICALLY — reusing it for a DIFFERENT device (garage door, stereo) requires MODIFYING this class directly.
- There's NO clean way to support UNDO — you'd need to manually track WHAT the PREVIOUS action was and IMPLEMENT reversal logic INLINE, likely with EVEN MORE conditional branching.
- Adding NEW actions means EDITING `pressButton()`'s CONDITIONAL logic — poor OCP.

**After (Command, as shown in Section 5):**
- `RemoteControl` is COMPLETELY DECOUPLED from `Light` (or ANY specific device) — it works with ANY `Command` implementation.
- UNDO is supported NATURALLY, since EACH Command KNOWS how to REVERSE itself.
- Adding a NEW action (e.g., controlling a GARAGE DOOR) means WRITING a NEW `Command` implementation — ZERO changes to `RemoteControl`.

---

## 10. SOLID Principles Connection

- **SRP**: each Concrete Command has EXACTLY one responsibility — INVOKING (and potentially REVERSING) ONE SPECIFIC action on its Receiver.
- **OCP**: NEW commands can be ADDED without MODIFYING the Invoker OR existing Command classes.
- **DIP**: `RemoteControl` depends ONLY on the `Command` ABSTRACTION, never on SPECIFIC Receiver classes (`Light`, etc.) directly.
- **LSP**: ANY `Command` implementation can be SUBSTITUTED wherever a `Command` is expected, and the Invoker's behavior REMAINS correct regardless of WHICH concrete command is actually being used.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Command pattern solve, in your own words?
2. What are the THREE main roles in the Command pattern (Invoker, Command, Receiver), and what is EACH responsible for?
3. Why doesn't the Invoker need to know ANYTHING about the Receiver?

**Intermediate:**
4. How does Command NATURALLY support UNDO/REDO functionality? What must EACH Concrete Command TRACK to make this work?
5. How would you implement MULTIPLE levels of undo/redo (not just a SINGLE `lastCommand`, as shown in Section 5)? (Hint: consider using a STACK, or TWO stacks.)
6. How does Java's `Runnable` interface relate CONCEPTUALLY to the Command pattern?
   *Answer: `Runnable`'s SINGLE method, `run()`, is CONCEPTUALLY equivalent to `execute()` — a `Runnable` ENCAPSULATES "an action to be performed" as an OBJECT, which can be PASSED to a `Thread`, an `ExecutorService`, or QUEUED for LATER execution — this IS essentially the Command pattern, WITHOUT the UNDO capability (since `Runnable` doesn't declare an UNDO method).*

**Advanced:**
7. **Command vs Strategy** — precisely distinguish them (both involve ENCAPSULATING behavior as an OBJECT — a natural comparison, since Strategy was covered in Topic 5).
   *Answer: Strategy encapsulates an INTERCHANGEABLE ALGORITHM for accomplishing a TASK, typically INVOKED IMMEDIATELY by the CLIENT with the SAME method signature EVERY TIME (e.g., "sort THIS data" using WHICHEVER sorting strategy). Command encapsulates a REQUEST/ACTION, often with its OWN PARAMETERS/RECEIVER BAKED IN, designed to be POTENTIALLY QUEUED, LOGGED, UNDONE, or executed LATER/DEFERRED rather than IMMEDIATELY. Structurally SIMILAR (both involve a CLIENT providing an INTERCHANGEABLE object implementing a COMMON interface), but Command's INTENT centers on TREATING AN ACTION AS DATA (to be stored/deferred/reversed), while Strategy's INTENT centers on SWAPPING interchangeable ALGORITHMS for an IMMEDIATE operation.*
8. How would you implement a "MACRO command" — a Command that, when EXECUTED, runs a SEQUENCE of OTHER Commands? How does this relate to the COMPOSITE pattern (Topic 14)?
   *Answer: A `MacroCommand` implementing `Command`, internally holding a `List<Command>`, and IMPLEMENTING `execute()` by LOOPING through and EXECUTING each contained Command (and `undo()` by UNDOING them, typically in REVERSE order) — this is a DIRECT application of COMPOSITE (Topic 14), treating a GROUP of commands UNIFORMLY as if it were a SINGLE command.*
9. How would you support QUEUING commands for ASYNCHRONOUS/DEFERRED execution (e.g., a background JOB PROCESSING system)? What ADDITIONAL considerations arise (e.g., SERIALIZING commands, handling FAILURES)?
10. Discuss how Command relates to the concept of "REQUEST OBJECTS" in WEB frameworks (e.g., an HTTP request being PROCESSED through a PIPELINE of handlers) — is this a NATURAL extension of the Command pattern's CORE idea?

---

## 12. Common Mistakes

- **Forgetting to implement PROPER undo logic** — if `undo()` doesn't CORRECTLY reverse EXACTLY what `execute()` did (especially for COMPLEX, STATEFUL actions), UNDO can leave the system in an INCONSISTENT state.
- **Putting TOO MUCH business logic directly INSIDE the Command, rather than DELEGATING to the Receiver** — a well-designed Command should mostly COORDINATE the call to its Receiver, not REIMPLEMENT the Receiver's actual logic itself.
- **Using Command for SIMPLE, ONE-OFF actions with NO need for queuing/undo/logging** — unnecessary CEREMONY when a DIRECT method call would suffice.
- **Not considering THREAD-SAFETY when Commands are QUEUED for ASYNCHRONOUS execution** — if a Command's Receiver or PARAMETERS are MUTABLE and SHARED across THREADS, CONCURRENT execution can introduce SUBTLE bugs.

---

## 13. Time Complexity

- **Time:** O(1) per `execute()`/`undo()` call (a SINGLE additional level of INDIRECTION/dispatch compared to a DIRECT method call); O(n) for a MACRO command executing n SUB-commands.
- **Space:** O(1) additional overhead PER Command object (typically just a REFERENCE to its Receiver plus a FEW parameters); O(h) for a COMMAND HISTORY supporting UNDO/REDO, where h = the NUMBER of commands CURRENTLY tracked in history.

---

## 14. Java API Examples

- **`java.lang.Runnable`**: the CLOSEST built-in JDK analog to Command's `execute()` method (via `run()`), WIDELY used with `Thread`, `ExecutorService`, and SIMILAR concurrency APIs.
- **`javax.swing.Action`**: an INTERFACE specifically designed for the Command pattern in SWING applications, ENCAPSULATING an ACTION that can be ATTACHED to MULTIPLE UI components (menu items, buttons, keyboard shortcuts) SIMULTANEOUSLY.
- **`java.util.concurrent.Callable<V>`**: SIMILAR to `Runnable`, but SUPPORTS returning a RESULT and THROWING checked exceptions — another CLOSE analog to the Command pattern's CORE idea.
- **Spring's `@Transactional` and command-style SERVICE methods**: while not a STRICT GoF Command implementation, the CONCEPT of ENCAPSULATING a UNIT OF WORK (with potential ROLLBACK on failure) STRONGLY echoes the Command pattern's INTENT around UNDO-ABLE operations.

---

## 15. Practice Problem

Implement a **Command pattern for a simple text editor's Undo functionality**: create a `TextEditor` (Receiver) with an `insertText(String text)` method (APPENDING to an internal `StringBuilder`) and a `deleteLastInsert()` method (for undoing). Implement an `InsertTextCommand` (Command) storing the TEXT that was inserted, with `execute()` calling `insertText()` and `undo()` calling `deleteLastInsert()`. Implement a simple `CommandHistory` (Invoker-adjacent) using a `Stack<Command>` to support MULTIPLE levels of undo. Demonstrate inserting SEVERAL pieces of text and then UNDOING them in REVERSE order.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **Command-based system for a smart home hub**, supporting MULTIPLE devices (`Light`, `Thermostat`, `Lock`), EACH with THEIR OWN specific commands (`LightOnCommand`, `SetTemperatureCommand`, `LockDoorCommand`). Implement a `CommandHistory` class supporting BOTH `undo()` AND `redo()` — using TWO STACKS (one for UNDO history, one for REDO history) — such that after UNDOING an action, REDOING it re-applies the SAME action, and EXECUTING a NEW command after an UNDO CLEARS the redo stack (as most real applications do)."

Think about:
- WHY executing a NEW command AFTER an undo should CLEAR the redo stack — what would happen (and why would it be CONFUSING/incorrect) if OLD redo history were KEPT around after a NEW, DIFFERENT action has been performed?
- How `SetTemperatureCommand`'s undo logic DIFFERS from `LightOnCommand`'s — since REVERSING "set temperature to 72°F" requires KNOWING the PREVIOUS temperature (which must be CAPTURED at COMMAND-CREATION time, or EXECUTION time), UNLIKE `LightOnCommand`'s SIMPLE, FIXED inverse ("on" always undoes to "off").

---

## 17. Advanced LLD Scenario

**Design a Collaborative Document Editing System** (like Google Docs) using Command, where:
- EVERY edit operation (insert text, delete text, format text, insert image) by ANY user is REPRESENTED as a `Command` object, ENABLING both INDIVIDUAL undo/redo AND, more CHALLENGINGLY, SYNCHRONIZING these commands ACROSS MULTIPLE users editing the SAME document SIMULTANEOUSLY
- Consider how COMMANDS need to be SERIALIZABLE (convertible to/from a TRANSMITTABLE format like JSON) so they can be SENT over the NETWORK to OTHER users' clients and RE-EXECUTED there, ensuring EVERYONE'S document stays IN SYNC
- Consider the COMPLEXITY of UNDO in a MULTI-USER context — if User A undoes THEIR OWN last edit, but User B has SINCE made edits that DEPEND on User A's change, what happens? (This is a genuinely HARD, real-world problem addressed by TECHNIQUES like "Operational Transformation" or "CRDTs" in PRODUCTION collaborative editors — you're not expected to FULLY solve it, but articulating the CHALLENGE clearly is valuable.)

Consider:
- Why REPRESENTING each edit as a DISCRETE, SELF-CONTAINED `Command` object is FOUNDATIONAL to enabling BOTH the UNDO/REDO feature AND the NETWORK SYNCHRONIZATION feature SIMULTANEOUSLY — the SAME Command objects SERVE double duty for BOTH purposes
- How this scenario ILLUSTRATES that Command, in SUFFICIENTLY COMPLEX real-world systems, OFTEN needs to be COMBINED with SERIALIZATION mechanisms and, POTENTIALLY, MORE SOPHISTICATED conflict-resolution ALGORITHMS beyond the SIMPLE stack-based undo/redo shown in EARLIER sections
- Why ARTICULATING the LIMITS of a SIMPLE pattern (acknowledging that REAL multi-user undo is a GENUINELY HARD problem beyond BASIC Command/stack mechanics) is ITSELF a STRONG signal of SENIOR-LEVEL engineering judgment in an INTERVIEW setting

---

## 18. Summary

**Definition:** Command encapsulates a request as a standalone object, decoupling the invoker of an action from the object that knows how to perform it, and enabling queuing, logging, and undo/redo.

**Intent:** Treat "an action to be performed" as a first-class object, allowing it to be parameterized, deferred, queued, logged, or reversed, rather than being fused directly into a method call.

**Key classes:** `Command` (interface with `execute()`/`undo()`), `Concrete Command` (wraps a specific action + its Receiver), `Receiver` (knows how to actually perform the action), `Invoker` (holds and triggers a Command without knowing its specifics).

**Advantages:** Decouples invoker from receiver; naturally supports undo/redo; enables queuing/logging of requests; good OCP for adding new commands.

**Disadvantages:** Additional class per distinct action; undo logic can add real complexity; can be overkill for simple, direct actions.

**Real-world use case:** GUI menu/toolbar actions, undo/redo in editors, job queues, macro recording, `Runnable`/`Callable` in Java concurrency.

**Java example:** `RemoteControl` (Invoker) triggering `LightOnCommand`/`LightOffCommand` (Commands) which delegate to `Light` (Receiver), with paired `execute()`/`undo()` logic.

**Interview takeaway:** Be ready to clearly distinguish Command (treating an ACTION AS DATA, for deferred/queued/undoable execution) from Strategy (swapping an INTERCHANGEABLE ALGORITHM for IMMEDIATE execution, Topic 5) — and be ready to connect Command with Composite (Topic 14) when discussing "macro commands" that bundle multiple commands together.

**One-line memory trick:** *"A restaurant order slip — the waiter doesn't cook, they just hand off a portable, queueable request to the kitchen that actually knows how."*

---

*End of Topic 22. Type "Next" to proceed to Topic 23: Interpreter Pattern.*