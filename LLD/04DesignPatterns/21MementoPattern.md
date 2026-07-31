# Topic 26: Memento Pattern

---

## 1. Introduction

**Definition:**
The Memento Pattern is a **behavioral design pattern** that lets you CAPTURE and EXTERNALIZE an object's INTERNAL STATE (without violating ENCAPSULATION) so that the object can be RESTORED to THIS state LATER — enabling UNDO/ROLLBACK functionality WITHOUT exposing the object's INTERNAL structure to the CODE that manages the SAVED states.

**Why it exists / what problem it solves:**
Consider needing UNDO functionality for a TEXT editor, or a "SAVE POINT" system in a GAME. You need to CAPTURE an object's ENTIRE internal STATE at a GIVEN moment, so it can be RESTORED LATER. The NAIVE approach — making ALL the object's INTERNAL FIELDS `public` (or providing GETTERS/SETTERS for EVERYTHING) so EXTERNAL code can READ and later WRITE them BACK — SERIOUSLY violates ENCAPSULATION, EXPOSING internal DETAILS that SHOULD remain PRIVATE, and RISKING external code ACCIDENTALLY (or MALICIOUSLY) MODIFYING that STATE INCORRECTLY.

Memento solves this by having the OBJECT ITSELF (called the "ORIGINATOR") create a SPECIAL SNAPSHOT object (the "MEMENTO") CONTAINING its INTERNAL state — CRITICALLY, this MEMENTO'S CONTENTS are ONLY ACCESSIBLE BACK to the ORIGINATOR ITSELF (via careful ACCESS CONTROL, e.g., NESTED classes or PACKAGE-PRIVATE methods), NOT to the EXTERNAL code (the "CARETAKER") that simply STORES and MANAGES a HISTORY of these MEMENTOS WITHOUT ever PEEKING INSIDE them.

**When it should be used:**
- When you need to IMPLEMENT UNDO/ROLLBACK functionality, CAPTURING an OBJECT'S state at VARIOUS points in TIME
- When DIRECTLY EXPOSING an OBJECT'S internal STATE (via PUBLIC fields/getters) to ACHIEVE this would VIOLATE ENCAPSULATION in a WAY you want to AVOID
- When you need to SAVE/RESTORE "CHECKPOINTS" (e.g., GAME save STATES, DOCUMENT version HISTORY, TRANSACTION rollback POINTS)

**When it should NOT be used:**
- When the OBJECT'S state is SIMPLE enough that EXPOSING it DIRECTLY (via a SIMPLE COPY/GETTER) DOESN'T MEANINGFULLY VIOLATE encapsulation OR RISK MISUSE
- When CAPTURING FULL snapshots is TOO EXPENSIVE (in MEMORY) for FREQUENTLY-CHANGING, LARGE objects — CONSIDER STORING only INCREMENTAL DIFFS/DELTAS instead (discussed further in SECTION 8)
- When the NUMBER of SAVED states could GROW UNBOUNDED without a CLEAR STRATEGY for LIMITING/PRUNING old MEMENTOS, RISKING excessive MEMORY usage

**Advantages:**
- PRESERVES encapsulation — the ORIGINATOR'S internal STRUCTURE is NEVER EXPOSED to the CARETAKER managing the HISTORY
- Provides a CLEAN way to IMPLEMENT UNDO/ROLLBACK/CHECKPOINT functionality
- SEPARATES the RESPONSIBILITY of "REMEMBERING PAST states" (Caretaker) from "KNOWING HOW to SAVE/RESTORE STATE" (Originator)

**Disadvantages:**
- CAN be MEMORY-INTENSIVE if MEMENTOS are LARGE and MANY are STORED (e.g., a FULL snapshot for EVERY SINGLE keystroke in a TEXT editor)
- Adds SOME DESIGN complexity — CAREFULLY CONTROLLING what's ACCESSIBLE ONLY to the ORIGINATOR (versus the CARETAKER) REQUIRES thoughtful use of ACCESS MODIFIERS/NESTED classes
- If the ORIGINATOR'S internal STRUCTURE CHANGES significantly OVER TIME, OLD MEMENTOS (created BEFORE the change) might BECOME INCOMPATIBLE with a NEWER version of the ORIGINATOR (a REAL concern for LONG-LIVED, PERSISTED mementos)

---

## 2. Real-World Analogy

Think of **SAVING your progress in a VIDEO GAME**.

When you HIT "SAVE," the GAME CAPTURES a SNAPSHOT of YOUR CURRENT progress (health, INVENTORY, POSITION, LEVEL) into a SAVE FILE. CRITICALLY, YOU (the PLAYER) don't NEED to UNDERSTAND the INTERNAL FORMAT of THAT save FILE — you JUST KNOW you can LOAD it LATER to RESTORE your GAME to EXACTLY that POINT. The SAVE FILE MANAGEMENT SYSTEM (LISTING your save SLOTS, LETTING you CHOOSE which ONE to LOAD) doesn't NEED to UNDERSTAND the INTERNAL CONTENTS of ANY save FILE either — it JUST STORES and RETRIEVES them; ONLY the GAME ITSELF (WHICH CREATED the save file) KNOWS HOW to properly INTERPRET and RESTORE FROM it.

---

## 3. Theory

**Core idea:** THREE roles WORK together: the **ORIGINATOR** (the object WHOSE state NEEDS saving/RESTORING) CREATES a **MEMENTO** object CAPTURING its CURRENT internal STATE. The **CARETAKER** STORES this MEMENTO (POSSIBLY ALONGSIDE OTHERS, FORMING a HISTORY) WITHOUT EVER EXAMINING or MODIFYING its CONTENTS. LATER, the ORIGINATOR can RESTORE its STATE FROM a PREVIOUSLY-SAVED MEMENTO, RETRIEVED FROM the CARETAKER.

**Working mechanism:**
```
Originator.save() → creates a Memento (capturing CURRENT state)
Caretaker.store(memento) → holds onto it, WITHOUT inspecting it

... time passes, Originator's state CHANGES ...

Caretaker.retrieve() → returns a PREVIOUSLY stored Memento
Originator.restore(memento) → RESTORES its state FROM the memento
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Originator | The object whose STATE needs to be SAVED/RESTORED |
| Memento | The SNAPSHOT object CAPTURING the Originator's state at a POINT in time |
| Caretaker | MANAGES/STORES Mementos, WITHOUT examining their CONTENTS |

**Class responsibilities:** the ORIGINATOR is RESPONSIBLE for BOTH CREATING Mementos (CAPTURING its OWN state) and RESTORING FROM them — it's the ONLY class that TRULY UNDERSTANDS a MEMENTO'S internal CONTENTS. The CARETAKER is RESPONSIBLE ONLY for STORING/ORGANIZING mementos (e.g., in a STACK for UNDO HISTORY) — it MUST NOT access or MODIFY their INTERNAL DATA.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      Editor (Originator)        │
├─────────────────────┤
│ - content: String                 │
├─────────────────────┤
│ + save(): EditorMemento          │  (CREATES a memento
│ + restore(EditorMemento)             CAPTURING current state)
│ + setContent(String)
└──────────┬──────────┘
           │ (creates / restores from)
           ▼
┌─────────────────────┐
│      <<static nested class>>    │
│      EditorMemento              │
├─────────────────────┤
│ - content: String  (PRIVATE —       │  (ONLY the Editor —
│   ACCESSIBLE only to Editor)          via its NESTED class
├─────────────────────┤       ACCESS — can READ this;
│ (package/private constructor        the Caretaker CANNOT)
│  and getter, restricted access)
└─────────────────────┘
           △
           │ (stores, WITHOUT examining contents)
┌─────────────────────┐
│      History (Caretaker)         │
├─────────────────────┤
│ - mementos: Stack<EditorMemento>  ◇──→ (aggregation: Caretaker
├─────────────────────┤       holds Mementos OPAQUELY,
│ + push(EditorMemento)                 treating them as a
│ + pop(): EditorMemento                 BLACK BOX)
└─────────────────────┘
```

**Relationship explanation:**
- `Editor` (ORIGINATOR) CREATES `EditorMemento` instances via `save()`, and can RESTORE its OWN state FROM one via `restore()` — this is a **dependency** relationship WHERE `Editor` KNOWS about `EditorMemento`'s FULL internal STRUCTURE.
- `EditorMemento` is COMMONLY implemented as a **static NESTED class** WITHIN `Editor` (or with PACKAGE-PRIVATE access) — this is the KEY MECHANISM for ENSURING that ONLY `Editor` can ACCESS its CONTENTS, while OUTSIDE code (like `History`) sees it as an OPAQUE OBJECT.
- `History` (CARETAKER) has an **aggregation** relationship with `EditorMemento` — it STORES REFERENCES to mementos (e.g., in a STACK) but NEVER examines their INTERNAL fields; it just PASSES them BACK to the `Editor` when RESTORATION is NEEDED.

---

## 5. Java Implementation

```java
import java.util.Stack;

// ============================================
// Originator — the object whose state needs saving/restoring
// ============================================
public class Editor {
    private String content = "";

    public void setContent(String content) {
        this.content = content;
    }

    public String getContent() {
        return content;
    }

    // Creates a Memento CAPTURING the CURRENT state
    public EditorMemento save() {
        return new EditorMemento(this.content);
    }

    // RESTORES state FROM a previously-created Memento
    public void restore(EditorMemento memento) {
        this.content = memento.getSavedContent();
    }

    // ============================================
    // Memento — static NESTED class, so its internal
    // state is ACCESSIBLE only to Editor (the outer class)
    // ============================================
    public static class EditorMemento {
        // PRIVATE field — NOT accessible from OUTSIDE this
        // nested class, INCLUDING from the Caretaker (History)
        private final String savedContent;

        // Package-private/private-ish constructor — in PRACTICE,
        // making this constructor NON-public (or RELYING on the
        // NESTED-class relationship) restricts WHO can CREATE
        // a Memento with ARBITRARY content
        private EditorMemento(String content) {
            this.savedContent = content;
        }

        // Only Editor (the OUTER class) can call THIS, since
        // it's PRIVATE — the Caretaker literally CANNOT access
        // savedContent, PRESERVING encapsulation
        private String getSavedContent() {
            return savedContent;
        }
    }
}

// ============================================
// Caretaker — stores Mementos WITHOUT examining their contents
// ============================================
public class History {
    // Stack naturally supports UNDO: most-recently-saved
    // memento is RESTORED first (LIFO order)
    private final Stack<Editor.EditorMemento> mementos = new Stack<>();

    public void push(Editor.EditorMemento memento) {
        mementos.push(memento);
    }

    public Editor.EditorMemento pop() {
        if (mementos.isEmpty()) {
            return null;
        }
        return mementos.pop();
    }
}

// ============================================
// Demo
// ============================================
public class MementoDemo {
    public static void main(String[] args) {
        Editor editor = new Editor();
        History history = new History();

        editor.setContent("Version 1");
        history.push(editor.save()); // CHECKPOINT saved

        editor.setContent("Version 2");
        history.push(editor.save()); // ANOTHER checkpoint saved

        editor.setContent("Version 3 (unsaved mistake)");
        System.out.println("Before undo: " + editor.getContent());

        // UNDO — restore to the MOST RECENTLY saved checkpoint
        editor.restore(history.pop());
        System.out.println("After 1 undo: " + editor.getContent());

        editor.restore(history.pop());
        System.out.println("After 2nd undo: " + editor.getContent());
    }
}
```

**Key line-by-line notes:**
- `EditorMemento` is a `static` NESTED class INSIDE `Editor` — this is a COMMON Java IDIOM for the Memento pattern, since NESTED classes CAN access `private` MEMBERS of their ENCLOSING class'S OTHER nested classes CONTEXT, ENABLING `Editor` to freely CREATE/READ `EditorMemento` objects WHILE keeping THOSE SAME fields INACCESSIBLE to `History`.
- `History.push()`/`pop()` operate PURELY on `Editor.EditorMemento` REFERENCES — `History` NEVER calls `getSavedContent()` (WHICH it CAN'T, since it's `private`) — it SIMPLY STORES and RETURNS OPAQUE memento OBJECTS.
- `editor.restore(history.pop())` DEMONSTRATES the FULL ROUND-TRIP: RETRIEVE a PAST memento FROM the CARETAKER, and HAND it BACK to the ORIGINATOR, WHICH KNOWS HOW to EXTRACT and APPLY its CONTENTS.

---

## 6. Dry Run

**Sample input:**
```java
editor.setContent("Version 1");
history.push(editor.save());
editor.setContent("Version 2");
editor.restore(history.pop());
```

```
1. editor.setContent("Version 1")
   → editor.content = "Version 1"

2. editor.save() called
   → Editor.save() executes
   → RETURNS new EditorMemento("Version 1")
       → EditorMemento's PRIVATE constructor RUNS,
         savedContent = "Version 1"
   → history.push(thisMemento) called
       → History.push() STORES this memento on its INTERNAL stack
       → History has NO IDEA what "savedContent" ACTUALLY is —
         it just holds an OPAQUE reference

3. editor.setContent("Version 2")
   → editor.content = "Version 2" (Editor's LIVE state has CHANGED)

4. history.pop() called
   → RETURNS the PREVIOUSLY pushed EditorMemento (savedContent="Version 1")

5. editor.restore(thatMemento) called
   → Editor.restore() executes
   → memento.getSavedContent() called
       → SINCE this call ORIGINATES from WITHIN Editor's OWN CODE
         (Editor calling a METHOD on its OWN nested class), it's
         ALLOWED despite getSavedContent() being PRIVATE
       → RETURNS "Version 1"
   → this.content = "Version 1"
   → editor.content is NOW RESTORED to "Version 1"
```

**What's happening in memory:** the `EditorMemento` OBJECT EXISTS INDEPENDENTLY on the HEAP, HOLDING a COPY of the content STRING at the TIME `save()` was CALLED. `History` holds a REFERENCE to THIS object WITHOUT ever READING its FIELDS. WHEN `restore()` is called, `Editor` — and ONLY `Editor` — is ABLE to REACH INSIDE the memento (via its PRIVATE ACCESS, GRANTED by the NESTED-class RELATIONSHIP) to EXTRACT the SAVED content and APPLY it BACK to its OWN LIVE state.

---

## 7. Real-World Software Example

- **Undo/Redo in TEXT editors and IDEs**: EACH "SAVE POINT" or EDIT STATE can be REPRESENTED as a MEMENTO, ALLOWING the EDITOR to REVERT to a PREVIOUS state WITHOUT EXPOSING its INTERNAL document MODEL to THE undo-HISTORY MANAGER.
- **Database TRANSACTION rollback**: a TRANSACTION MANAGER can CAPTURE a "SAVEPOINT" (CONCEPTUALLY a MEMENTO) BEFORE a SERIES of OPERATIONS, ALLOWING a ROLLBACK to THAT point if SOMETHING FAILS.
- **Game SAVE/LOAD systems**: CAPTURING a PLAYER'S/GAME WORLD'S COMPLETE state INTO a SAVE FILE (a MEMENTO), MANAGED by a SAVE-file SYSTEM (CARETAKER) that DOESN'T NEED to UNDERSTAND the GAME'S internal DATA structures.
- **Version control SYSTEMS (conceptually)**: EACH COMMIT can be THOUGHT of as a MEMENTO CAPTURING the ENTIRE PROJECT'S state at a POINT in TIME, ALLOWING RESTORATION to ANY PREVIOUS COMMIT.
- **`java.io.Serializable`-based STATE snapshots**: SERIALIZING an OBJECT to BYTES (and LATER DESERIALIZING it) is a COMMON, PRACTICAL WAY to IMPLEMENT MEMENTO-style STATE CAPTURE/RESTORE in JAVA, ESPECIALLY WHEN the OBJECT'S structure is COMPLEX.

---

## 8. Internal Working

**Object creation:** EACH CALL to `save()` CREATES a NEW `Memento` OBJECT, TYPICALLY COPYING the CURRENT VALUES of the ORIGINATOR'S RELEVANT fields — the MEMENTO'S CONTENTS are FROZEN at the MOMENT of CREATION and DON'T CHANGE EVEN IF the ORIGINATOR'S LIVE state LATER CHANGES.

**Memory/scaling CONSIDERATIONS — FULL snapshots vs INCREMENTAL DIFFS:** for LARGE, FREQUENTLY-CHANGING objects, STORING a COMPLETE, FULL SNAPSHOT for EVERY SAVE POINT can BECOME EXPENSIVE in MEMORY. A COMMON REAL-WORLD OPTIMIZATION is to STORE ONLY the INCREMENTAL DIFFERENCE ("DELTA") BETWEEN CONSECUTIVE STATES, RATHER THAN a FULL COPY EACH TIME — THIS TRADES SOME ADDITIONAL COMPUTATION (REBUILDING a FULL STATE by REPLAYING DELTAS) FOR SIGNIFICANTLY REDUCED MEMORY usage, ESPECIALLY WHEN CHANGES BETWEEN SAVES are SMALL RELATIVE to the TOTAL STATE SIZE.

**Access CONTROL mechanism:** the CORE "TRICK" ENABLING ENCAPSULATION PRESERVATION is JAVA'S NESTED-CLASS ACCESS RULES — a `private` MEMBER of a NESTED class IS ACCESSIBLE from the ENCLOSING (OUTER) class'S code, EVEN THOUGH IT'S NOT ACCESSIBLE from ANY OTHER, UNRELATED class (LIKE `History`) — THIS is PRECISELY WHY THE `EditorMemento` NESTED-class APPROACH WORKS TO KEEP `savedContent` HIDDEN FROM the CARETAKER.

---

## 9. Before vs After

**Before (no Memento — ENCAPSULATION violated for UNDO purposes):**

```java
public class EditorBad {
    // Fields made PUBLIC (or given PUBLIC getters/setters)
    // JUST so external "history" code can SAVE/RESTORE them —
    // this EXPOSES internal structure UNNECESSARILY
    public String content = "";
}

public class HistoryBad {
    private Stack<String> savedContents = new Stack<>();

    public void save(EditorBad editor) {
        savedContents.push(editor.content); // DIRECT field access
    }

    public void undo(EditorBad editor) {
        editor.content = savedContents.pop(); // DIRECTLY overwrites
    }
}
```

**Problems:**
- `EditorBad.content` MUST be `public` (or have a PUBLIC SETTER) PURELY to SUPPORT the UNDO FEATURE — this EXPOSES INTERNAL state to ANY OTHER CODE in the APPLICATION, WHICH could ACCIDENTALLY (or CARELESSLY) MODIFY it in WAYS UNRELATED to the UNDO FEATURE'S NEEDS.
- If `EditorBad`'s INTERNAL representation LATER CHANGES (e.g., SPLITTING `content` into MULTIPLE fields, or CHANGING its TYPE), `HistoryBad`'s CODE WOULD NEED to be UPDATED TOO, since IT DIRECTLY DEPENDS on `EditorBad`'s SPECIFIC internal STRUCTURE.
- There's NO CLEAR BOUNDARY BETWEEN "WHAT'S PART of the UNDO mechanism" and "WHAT'S JUST GENERAL, PUBLIC editor STATE" — EVERYTHING is EXPOSED THE SAME WAY.

**After (Memento, as shown in Section 5):**
- `Editor.content` REMAINS `private` — ONLY `Editor` ITSELF can READ/WRITE it DIRECTLY.
- `History` interacts ONLY with OPAQUE `EditorMemento` OBJECTS, NEVER TOUCHING `Editor`'s INTERNAL fields DIRECTLY.
- If `Editor`'s INTERNAL representation CHANGES LATER, ONLY `Editor`'s OWN CODE (INCLUDING its NESTED `EditorMemento` class) needs UPDATING — `History` remains COMPLETELY UNAFFECTED.

---

## 10. SOLID Principles Connection

- **SRP**: `Editor` (Originator) is RESPONSIBLE for its OWN state AND for CREATING/RESTORING FROM Mementos; `History` (Caretaker) is RESPONSIBLE ONLY for MANAGING the COLLECTION of saved Mementos — CLEANLY SEPARATED CONCERNS.
- **Encapsulation (CLOSELY related to, though NOT strictly PART of, SOLID)**: Memento is FUNDAMENTALLY ABOUT PRESERVING ENCAPSULATION WHILE STILL SUPPORTING external STATE-management NEEDS (UNDO history) — a GENUINE DESIGN GOAL THIS PATTERN DIRECTLY ADDRESSES.
- **OCP**: ADDING NEW WAYS to MANAGE HISTORY (e.g., a `History` VARIANT LIMITING the NUMBER of SAVED STATES) can be DONE WITHOUT MODIFYING `Editor` OR `EditorMemento` AT ALL.

---

## 11. Interview Questions

**Beginner:**
1. What PROBLEM does the Memento pattern SOLVE, in YOUR own WORDS?
2. What are the THREE ROLES in the Memento pattern, and WHAT is EACH RESPONSIBLE for?
3. Why SHOULDN'T the CARETAKER be ABLE to READ a MEMENTO'S internal CONTENTS?

**Intermediate:**
4. How does JAVA'S NESTED-class ACCESS MECHANISM help IMPLEMENT Memento WITHOUT VIOLATING encapsulation?
5. What's the TRADEOFF BETWEEN STORING FULL SNAPSHOTS VERSUS INCREMENTAL DIFFS/DELTAS for MEMENTOS? WHEN would you CHOOSE ONE OVER the OTHER?
6. How would you LIMIT the NUMBER of SAVED mementos (e.g., ONLY KEEPING the LAST 10 UNDO STEPS) WITHOUT MODIFYING the `Editor`/`EditorMemento` classes THEMSELVES?

**Advanced:**
7. **Memento vs Command (with UNDO support)** — BOTH PATTERNS can SUPPORT UNDO functionality (COMMAND was COVERED in TOPIC 22). PRECISELY DISTINGUISH THEM.
   *Answer: COMMAND'S UNDO APPROACH TYPICALLY STORES the SPECIFIC ACTION and ENOUGH INFORMATION to REVERSE IT (e.g., "the OPPOSITE OPERATION," or the SPECIFIC OLD VALUE THAT WAS CHANGED) — it's FOCUSED on REVERSING a DISCRETE OPERATION. MEMENTO CAPTURES a COMPLETE (or PARTIAL) SNAPSHOT of an OBJECT'S ENTIRE STATE at a POINT in TIME, ALLOWING FULL RESTORATION REGARDLESS of WHAT SPECIFIC OPERATIONS LED to THAT STATE. COMMAND'S UNDO is "REVERSE THIS SPECIFIC ACTION"; MEMENTO'S RESTORE is "GO BACK to THIS EXACT PREVIOUS STATE" — THEY CAN EVEN be COMBINED (e.g., a COMMAND THAT INTERNALLY USES a MEMENTO to CAPTURE STATE BEFORE EXECUTING, ENABLING a SIMPLE, GENERIC UNDO WITHOUT EACH COMMAND NEEDING CUSTOM REVERSAL LOGIC).*
8. How would you HANDLE VERSIONING/COMPATIBILITY ISSUES IF a MEMENTO CREATED BY an OLDER VERSION of the ORIGINATOR CLASS NEEDS to be RESTORED BY a NEWER VERSION of THAT CLASS (e.g., AFTER a SOFTWARE UPDATE ADDS NEW FIELDS)?
9. Discuss HOW you MIGHT IMPLEMENT Memento USING JAVA SERIALIZATION (SERIALIZING the ORIGINATOR'S STATE to BYTES) INSTEAD of a HAND-WRITTEN NESTED CLASS. WHAT are the TRADEOFFS (PERFORMANCE, FLEXIBILITY, VERSIONING) COMPARED to the MANUAL APPROACH SHOWN in SECTION 5?
10. How would you SUPPORT BOTH UNDO AND REDO USING MEMENTO (SIMILAR to COMMAND'S TWO-STACK APPROACH DISCUSSED in TOPIC 22)?

---

## 12. Common Mistakes

- **MAKING the MEMENTO'S internal STATE ACCESSIBLE to the CARETAKER** (e.g., PROVIDING a PUBLIC GETTER on the MEMENTO CLASS) — THIS DEFEATS the ENTIRE PURPOSE of the PATTERN, SINCE the CARETAKER SHOULD TREAT MEMENTOS as COMPLETELY OPAQUE.
- **STORING an UNBOUNDED NUMBER of MEMENTOS WITHOUT ANY LIMIT/PRUNING STRATEGY** — for LONG-RUNNING APPLICATIONS (e.g., an EDITOR SESSION LASTING HOURS), THIS can LEAD to SIGNIFICANT, UNNECESSARY MEMORY GROWTH OVER TIME.
- **CAPTURING MUTABLE OBJECT REFERENCES INSIDE a MEMENTO WITHOUT PROPERLY DEEP-COPYING THEM** (RELATED to PROTOTYPE'S SHALLOW-vs-DEEP COPY CONCERNS, TOPIC 18) — IF a MEMENTO STORES a REFERENCE to a MUTABLE OBJECT (RATHER than a TRUE COPY/SNAPSHOT of ITS VALUE AT THAT TIME), LATER CHANGES to THAT OBJECT COULD INCORRECTLY "LEAK" INTO the SUPPOSEDLY-FROZEN MEMENTO.
- **CONFUSING MEMENTO WITH SIMPLY MAKING FIELDS PUBLIC "FOR CONVENIENCE"** — THE WHOLE POINT of the PATTERN is AVOIDING THIS EXACT SHORTCUT WHILE STILL SUPPORTING SAVE/RESTORE FUNCTIONALITY.

---

## 13. Time Complexity

- **Time:** O(n) to CREATE a MEMENTO (WHERE n = the SIZE of the STATE being CAPTURED/COPIED); O(1) to STORE/RETRIEVE a MEMENTO from a STACK-based CARETAKER.
- **Space:** O(n) PER MEMENTO (PROPORTIONAL to the CAPTURED STATE'S SIZE) × O(h) for h TOTAL STORED MEMENTOS in HISTORY — POTENTIALLY SIGNIFICANT for LARGE OBJECTS WITH MANY SAVE POINTS, UNLESS an INCREMENTAL-DIFF STRATEGY (SECTION 8) is USED INSTEAD.

---

## 14. Java API Examples

- **`java.io.Serializable`**: WIDELY USED as a PRACTICAL MECHANISM for IMPLEMENTING MEMENTO-STYLE STATE CAPTURE — SERIALIZING an OBJECT'S STATE to a BYTE ARRAY (OR FILE) EFFECTIVELY CREATES a MEMENTO, WITH DESERIALIZATION SERVING as the RESTORE OPERATION.
- **`javax.swing.undo.UndoManager`**: SWING'S BUILT-IN UNDO/REDO FRAMEWORK, CONCEPTUALLY RELATED to BOTH MEMENTO and COMMAND, MANAGING a HISTORY of "UNDOABLE EDITS."
- **Database SAVEPOINTS (`java.sql.Savepoint`)**: JDBC'S `Connection.setSavepoint()` CONCEPTUALLY MIRRORS MEMENTO — CAPTURING a POINT WITHIN a TRANSACTION THAT CAN BE ROLLED BACK to, WITHOUT the APPLICATION CODE NEEDING to UNDERSTAND the DATABASE'S INTERNAL TRANSACTION-LOG STRUCTURE.
- **Version CONTROL APIs / LIBRARIES**: MANY LIBRARIES MANAGING "SNAPSHOTS" or "CHECKPOINTS" of APPLICATION STATE (e.g., in GAME ENGINES or SIMULATION FRAMEWORKS) EMPLOY MEMENTO-STYLE DESIGNS INTERNALLY.

---

## 15. Practice Problem

Implement a **MEMENTO for a SIMPLE CALCULATOR**: CREATE a `Calculator` (ORIGINATOR) CLASS with a `currentValue` (double) FIELD, SUPPORTING `add(double)`/`subtract(double)` OPERATIONS. IMPLEMENT a NESTED `CalculatorMemento` CLASS (WITH `private` ACCESS to ITS SAVED VALUE), a `save()` METHOD ON `Calculator`, and a `restore(CalculatorMemento)` METHOD. IMPLEMENT a `CalculatorHistory` (CARETAKER) USING a `Stack<CalculatorMemento>`. DEMONSTRATE PERFORMING SEVERAL OPERATIONS, SAVING AT EACH STEP, and UNDOING BACK to an EARLIER VALUE.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "DESIGN a **MEMENTO-based SYSTEM for a FORM-FILLING WIZARD** (a MULTI-STEP FORM, LIKE a CHECKOUT FLOW) THAT ALLOWS USERS to NAVIGATE 'BACK' to a PREVIOUS STEP WITHOUT LOSING THEIR PREVIOUSLY-ENTERED DATA on THAT STEP, WHILE ALSO SUPPORTING a 'START OVER' FEATURE THAT DISCARDS ALL SAVED STATES. Design the `FormWizard` (ORIGINATOR), a `FormMemento` (CAPTURING ALL FORM FIELDS AT a GIVEN STEP), and a `WizardHistory` (CARETAKER) SUPPORTING BOTH 'GO BACK' (POP the MOST RECENT MEMENTO) AND 'START OVER' (CLEAR ALL MEMENTOS)."

Think about:
- WHETHER EACH "STEP" of the WIZARD SHOULD BE ITS OWN SEPARATE MEMENTO, OR WHETHER a SINGLE, CUMULATIVE MEMENTO SHOULD CAPTURE ALL FIELDS ENTERED SO FAR AT EACH STEP — WHAT are the TRADEOFFS of EACH APPROACH?
- HOW "START OVER" (CLEARING ALL history) DIFFERS FUNCTIONALLY from a SINGLE "UNDO" (POPPING JUST ONE MEMENTO), AND HOW the `WizardHistory` CARETAKER's API SHOULD REFLECT THIS DISTINCTION CLEARLY.

---

## 17. Advanced LLD Scenario

**DESIGN a COLLABORATIVE WHITEBOARD APPLICATION'S UNDO/REDO SYSTEM** USING MEMENTO, WHERE:
- MULTIPLE USERS can DRAW SHAPES, TEXT, and IMAGES on a SHARED WHITEBOARD SIMULTANEOUSLY, and EACH USER NEEDS THEIR OWN INDEPENDENT UNDO/REDO HISTORY for THEIR OWN ACTIONS (WHILE STILL SEEING OTHER USERS' CHANGES IN REAL-TIME)
- THE WHITEBOARD'S FULL STATE (ALL SHAPES, POSITIONS, STYLES) COULD BECOME QUITE LARGE OVER a LONG SESSION — CONSIDER WHETHER FULL SNAPSHOTS (as SHOWN in SECTION 5) or INCREMENTAL DELTAS (DISCUSSED in SECTION 8) WOULD BE MORE APPROPRIATE HERE, GIVEN POTENTIALLY FREQUENT, SMALL CHANGES (e.g., DRAGGING a SHAPE GENERATES MANY INTERMEDIATE POSITION UPDATES)
- CONSIDER HOW "UNDO MY LAST ACTION" INTERACTS WITH OTHER USERS' CONCURRENT CHANGES — IF USER A UNDOES THEIR OWN SHAPE-CREATION, BUT USER B HAS SINCE MODIFIED THAT SAME SHAPE, WHAT SHOULD HAPPEN? (THIS ECHOES THE SAME GENUINELY HARD MULTI-USER CONFLICT PROBLEM DISCUSSED IN TOPIC 22'S ADVANCED SCENARIO for COLLABORATIVE DOCUMENT EDITING)

Consider:
- WHY AN INCREMENTAL-DELTA APPROACH (STORING ONLY "WHAT CHANGED" PER ACTION, RATHER THAN a FULL WHITEBOARD SNAPSHOT PER UNDO STEP) IS LIKELY FAR MORE PRACTICAL HERE, GIVEN POTENTIALLY THOUSANDS of SMALL, INCREMENTAL CHANGES DURING a TYPICAL SESSION — DIRECTLY APPLYING SECTION 8'S MEMORY-SCALING DISCUSSION to a CONCRETE, REALISTIC SCENARIO
- HOW PER-USER UNDO HISTORY (EACH USER HAVING THEIR OWN INDEPENDENT STACK of MEMENTOS for THEIR OWN ACTIONS ONLY) DIFFERS STRUCTURALLY from a SINGLE, GLOBAL UNDO HISTORY SHARED by EVERYONE — AND WHY THE FORMER IS GENERALLY THE EXPECTED, INTUITIVE BEHAVIOR IN REAL COLLABORATIVE TOOLS (e.g., FIGMA, GOOGLE DOCS)
- HOW THIS SCENARIO'S "MULTI-USER UNDO CONFLICT" CHALLENGE CONNECTS BACK TO TOPIC 22'S DISCUSSION of OPERATIONAL TRANSFORMATION/CRDTs — REINFORCING THAT BOTH MEMENTO AND COMMAND, WHILE POWERFUL FOR SINGLE-USER UNDO, REQUIRE SIGNIFICANTLY MORE SOPHISTICATED TECHNIQUES WHEN EXTENDED TO GENUINELY CONCURRENT, MULTI-USER ENVIRONMENTS

---

## 18. Summary

**Definition:** MEMENTO CAPTURES and EXTERNALIZES an OBJECT'S internal STATE WITHOUT VIOLATING ENCAPSULATION, ALLOWING the OBJECT to BE RESTORED to THAT STATE LATER.

**Intent:** SUPPORT UNDO/ROLLBACK/CHECKPOINT FUNCTIONALITY WHILE KEEPING the OBJECT'S internal STRUCTURE HIDDEN FROM the CODE MANAGING the SAVED-STATE HISTORY.

**Key classes:** `Originator` (the OBJECT whose STATE is SAVED/RESTORED), `Memento` (the SNAPSHOT, TYPICALLY a NESTED CLASS with RESTRICTED ACCESS), `Caretaker` (MANAGES/STORES mementos WITHOUT examining THEIR CONTENTS).

**Advantages:** PRESERVES ENCAPSULATION; CLEAN UNDO/ROLLBACK/CHECKPOINT SUPPORT; CLEARLY SEPARATES "REMEMBERING PAST STATES" FROM "KNOWING HOW to SAVE/RESTORE."

**Disadvantages:** CAN be MEMORY-INTENSIVE for LARGE, FREQUENTLY-SAVED OBJECTS; REQUIRES CAREFUL ACCESS-CONTROL DESIGN; OLD MEMENTOS MAY BECOME INCOMPATIBLE IF the ORIGINATOR'S STRUCTURE CHANGES OVER TIME.

**Real-world use case:** UNDO/REDO in TEXT EDITORS/IDEs, DATABASE TRANSACTION SAVEPOINTS, GAME SAVE/LOAD SYSTEMS, VERSION-CONTROL-STYLE SNAPSHOTS.

**Java example:** `Editor` (ORIGINATOR) CREATING `EditorMemento` (a `private`-ACCESS NESTED CLASS) SNAPSHOTS, MANAGED OPAQUELY by a `History` (CARETAKER) USING a STACK.

**Interview takeaway:** BE READY to CLEARLY DISTINGUISH MEMENTO (CAPTURING a COMPLETE STATE SNAPSHOT for FULL RESTORATION) FROM COMMAND'S UNDO SUPPORT (REVERSING a SPECIFIC, DISCRETE ACTION, TOPIC 22) — AND BE READY to DISCUSS the FULL-SNAPSHOT VERSUS INCREMENTAL-DELTA TRADEOFF for MANAGING MEMORY USAGE in REAL-WORLD, LARGE-SCALE UNDO SYSTEMS.

**One-line memory trick:** *"A GAME SAVE FILE — you don't NEED to UNDERSTAND its INTERNAL FORMAT to LOAD it BACK; ONLY the GAME ITSELF TRULY KNOWS HOW to READ it."*

---

*End of Topic 26. Type "Next" to proceed to Topic 27: Template Method Pattern.*