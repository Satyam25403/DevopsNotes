# Topic 18: Prototype Pattern

---

## 1. Introduction

**Definition:**
The Prototype Pattern is a **creational design pattern** that creates NEW objects by COPYING (cloning) an existing object — the "prototype" — instead of instantiating a brand-new object from scratch via a constructor. The clone starts as an INDEPENDENT copy of the prototype's state, which can then be modified without affecting the original.

**Why it exists / what problem it solves:**
Sometimes creating an object from scratch is EXPENSIVE (e.g., it requires a costly database query, complex computation, or loading a large amount of data to reach a particular initial state), or the object's exact TYPE isn't known until runtime, making direct instantiation via `new SomeSpecificClass()` awkward or impossible without extensive `if-else`/`instanceof` checks.

Prototype solves this by letting you CLONE an already-configured, representative instance instead — since cloning typically COPIES existing in-memory data (rather than recomputing/re-fetching it), this can be significantly CHEAPER than full re-construction. It also elegantly solves the "unknown type at runtime" problem: if you have an OBJECT (of whatever concrete type it happens to be), you can simply ask IT to clone itself, without needing to know or check its specific class.

**When it should be used:**
- When object creation is EXPENSIVE (heavy computation, I/O, database access) and a similar, already-initialized instance already exists to copy from
- When the SPECIFIC concrete class of an object isn't known until runtime, but you have an INSTANCE of it and need another one just like it
- When you want to AVOID a large hierarchy of Factory subclasses (Topic 8/9) purely for the purpose of creating variations of pre-configured objects

**When it should NOT be used:**
- When object creation is CHEAP and simple — cloning adds unnecessary complexity (implementing `clone()`/copy logic) compared to a plain constructor call
- When the object contains resources that CANNOT be meaningfully copied (e.g., an open file handle, a network socket, a database connection) — cloning such objects raises serious questions about what a "copy" even MEANS
- When DEEP vs SHALLOW copy semantics are unclear or risky for the object's structure — getting this wrong is a common, serious source of bugs (discussed extensively in Section 8)

**Advantages:**
- Can be significantly CHEAPER than full re-construction for expensive-to-create objects
- Avoids tight coupling to specific concrete classes — cloning works via a common interface, regardless of the object's actual runtime type
- Reduces the need for a large hierarchy of Factory subclasses purely to produce pre-configured variants

**Disadvantages:**
- Implementing CORRECT cloning (especially DEEP cloning for objects with nested mutable references) can be genuinely tricky and error-prone
- Classes with CIRCULAR references between objects can make cloning logic complex to get right
- Overuse can make it unclear WHERE an object's "true" original source of truth lives, if many clones are floating around independently

---

## 2. Real-World Analogy

Think of **photocopying a filled-out form**.

Instead of writing out an ENTIRE form from scratch every time (re-entering your name, address, and other information that rarely changes), you PHOTOCOPY an already-filled-out template — instantly getting an EXACT duplicate of all that pre-filled information. You can then make a FEW small edits to the photocopy (e.g., update just the date or a specific field) WITHOUT affecting the original template, and WITHOUT needing to re-enter everything from scratch each time.

---

## 3. Theory

**Core idea:** Define a common `Cloneable`-style interface (or use Java's built-in `Cloneable`/`clone()` mechanism) declaring a `clone()` method. Each class implementing this interface provides its OWN logic for creating an independent COPY of itself — copying primitive fields directly, and carefully handling any OBJECT-REFERENCE fields (deciding whether they should be SHALLOW-copied, sharing the reference, or DEEP-copied, creating independent copies of the referenced objects too).

**Working mechanism:**
```
existingPrototype.clone()  ──→  returns a NEW, independent object
                                  with the SAME initial state
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Prototype | The existing object being cloned from |
| Clone | The newly created, independent copy |
| Shallow copy | Copies an object's fields directly; OBJECT-REFERENCE fields still point to the SAME underlying objects as the original |
| Deep copy | Recursively copies OBJECT-REFERENCE fields too, so the clone shares NO mutable state with the original |

**Class responsibilities:** each Prototype class is responsible for correctly implementing its OWN `clone()` logic — deciding, field by field, whether a SHALLOW or DEEP copy is appropriate for each reference-typed field, based on whether that field's underlying object should be SHARED or INDEPENDENTLY duplicated.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      Cloneable (or custom)   │
├─────────────────────┤
│ + clone(): Prototype        │
└──────────┬──────────┘
           △ (implements)
           │
┌──────────┴───────────────┐
│ GameCharacter                    │
├──────────────────────────┤
│ - name: String                     │
│ - level: int                          │
│ - inventory: List<String>  ◇──→ (reference field —
├──────────────────────────┤    DEEP-copying this
│ + clone(): GameCharacter          requires creating a
│   (copies name, level DIRECTLY;   NEW List, not just
│    creates a NEW inventory list   copying the reference)
│    to achieve a DEEP copy)
└──────────────────────────┘

┌─────────────────────┐
│      Client                      │
├─────────────────────┤
│ GameCharacter clone =           │
│   existingCharacter.clone();       │
└─────────────────────┘
```

**Relationship explanation:**
- `GameCharacter` **implements** the cloning contract (`Cloneable`/custom interface) — this is what enables `.clone()` to be called POLYMORPHICALLY, without the Client needing to know `GameCharacter`'s specific concrete type.
- The `inventory` field is an object REFERENCE — the `clone()` method must EXPLICITLY decide whether to copy this reference directly (shallow, meaning the ORIGINAL and the CLONE would share the SAME underlying `List` object — a common source of BUGS) or create a genuinely NEW `List` with copied contents (deep, ensuring true independence).
- The `Client` calls `.clone()` and receives an INDEPENDENT `GameCharacter` — from the Client's perspective, this looks like ordinary object creation, but internally it was achieved via COPYING rather than fresh construction.

---

## 5. Java Implementation

```java
import java.util.ArrayList;
import java.util.List;

// ============================================
// Prototype — implements a clone() method
// ============================================
public class GameCharacter implements Cloneable {
    private String name;
    private int level;
    private List<String> inventory; // reference field — needs careful cloning

    public GameCharacter(String name, int level) {
        this.name = name;
        this.level = level;
        this.inventory = new ArrayList<>();
    }

    public void addItem(String item) {
        inventory.add(item);
    }

    public void setName(String name) {
        this.name = name;
    }

    public void setLevel(int level) {
        this.level = level;
    }

    @Override
    public String toString() {
        return "GameCharacter{name='" + name + "', level=" + level
                + ", inventory=" + inventory + "}";
    }

    // ============================================
    // clone() — implements DEEP copy semantics
    // ============================================
    @Override
    public GameCharacter clone() {
        try {
            // super.clone() performs a SHALLOW copy first —
            // this copies name (String, immutable, safe to share)
            // and level (primitive, always safely copied) correctly,
            // but 'inventory' would still point to the SAME List
            // as the original at this point
            GameCharacter cloned = (GameCharacter) super.clone();

            // Explicitly DEEP-copy the mutable 'inventory' field —
            // creating a genuinely NEW List with the SAME contents,
            // so modifying the clone's inventory does NOT affect
            // the original's inventory, and vice versa
            cloned.inventory = new ArrayList<>(this.inventory);

            return cloned;
        } catch (CloneNotSupportedException e) {
            // This should never actually happen since we implement Cloneable,
            // but Java's checked-exception design requires handling it
            throw new AssertionError("Cloning not supported", e);
        }
    }
}

// ============================================
// Demo
// ============================================
public class PrototypeDemo {
    public static void main(String[] args) {
        // Create an expensive, representative "template" character
        GameCharacter warriorTemplate = new GameCharacter("Warrior Template", 1);
        warriorTemplate.addItem("Sword");
        warriorTemplate.addItem("Shield");

        // Clone it to create a new, INDEPENDENT character instance
        GameCharacter player1 = warriorTemplate.clone();
        player1.setName("Aragorn");
        player1.setLevel(5);
        player1.addItem("Health Potion"); // only affects player1's inventory

        GameCharacter player2 = warriorTemplate.clone();
        player2.setName("Legolas");

        System.out.println(warriorTemplate); // unaffected by clones' changes
        System.out.println(player1);
        System.out.println(player2);
    }
}
```

**Key line-by-line notes:**
- `super.clone()` invokes `Object`'s built-in `clone()` method, which performs a FIELD-BY-FIELD SHALLOW copy — correct and sufficient for `name` (an immutable `String`) and `level` (a primitive `int`), but INSUFFICIENT on its own for `inventory` (a mutable `List` reference).
- The explicit line `cloned.inventory = new ArrayList<>(this.inventory);` is what UPGRADES this from a shallow copy to a DEEP copy for the `inventory` field specifically — creating a genuinely separate `List` object with COPIED contents.
- `player1.addItem("Health Potion")` modifies ONLY `player1`'s own independent `inventory` list — thanks to the deep copy, this has ZERO effect on `warriorTemplate`'s or `player2`'s inventories.

---

## 6. Dry Run

**Sample input:**
```java
GameCharacter player1 = warriorTemplate.clone();
player1.addItem("Health Potion");
```

```
1. warriorTemplate.clone() called
   → GameCharacter.clone() executes on warriorTemplate
   → super.clone() performs a SHALLOW copy:
       → cloned.name = "Warrior Template" (String, immutable — safe as-is)
       → cloned.level = 1 (primitive — safely copied)
       → cloned.inventory = <SAME reference as warriorTemplate.inventory>
                             (still pointing to the SAME List object
                              at this intermediate point — NOT yet safe)
   → cloned.inventory = new ArrayList<>(this.inventory)
       → A BRAND NEW ArrayList is created, containing COPIES of
         warriorTemplate's inventory contents ("Sword", "Shield")
       → cloned.inventory now points to this NEW, INDEPENDENT list
   → clone() returns 'cloned'
   → player1 now references this fully-independent GameCharacter

2. player1.addItem("Health Potion") called
   → Adds "Health Potion" to player1's OWN inventory list
   → Since player1's inventory is a SEPARATE ArrayList object
     (from step 1's deep copy), warriorTemplate's inventory
     list remains COMPLETELY UNCHANGED
```

**What's happening in memory:** BEFORE the explicit deep-copy line executes, `cloned.inventory` and `warriorTemplate.inventory` point to the EXACT SAME heap object (a shallow copy's natural result for reference fields) — a genuine BUG risk if left unaddressed. The explicit `new ArrayList<>(this.inventory)` call allocates a SEPARATE list object on the heap, breaking this shared reference and ensuring TRUE independence between the original and the clone.

---

## 7. Real-World Software Example

- **`Object.clone()`** itself, and Java's `Cloneable` marker interface — the most direct, built-in example of this pattern in the JDK.
- **Game development**: cloning pre-configured "template" enemies, characters, or items (as in this example) rather than reconstructing them from scratch for every spawn, especially when initial setup involves expensive computation or asset loading.
- **Document/configuration templates**: cloning a pre-configured "template" document or configuration object as a STARTING POINT for a new, customized instance, rather than rebuilding all the default settings from scratch each time.
- **Prototype registries in graphics/CAD software**: a palette of pre-configured shape "prototypes" that users clone and then customize (resize, recolor) individually, without needing to reconfigure every property from a blank slate each time.
- **`ArrayList`'s copy constructor (`new ArrayList<>(existingList)`)**: while not the FULL GoF Prototype pattern, this embodies the SAME core idea — creating a new collection by COPYING an existing one's contents.

---

## 8. Internal Working

**Shallow vs Deep copy — the CENTRAL internal concern:** `Object.clone()`'s DEFAULT behavior is a SHALLOW copy — it copies PRIMITIVE fields directly and copies REFERENCE fields as REFERENCES (meaning the original and the clone end up POINTING TO the SAME underlying objects for any mutable reference-typed fields). This is often INSUFFICIENT and requires MANUAL, explicit deep-copying of specific fields (as shown in Section 5) whenever those fields are MUTABLE and should NOT be shared between the original and the clone.

**Circular reference challenge:** if `GameCharacter` had a field referencing ANOTHER object which, in turn, referenced BACK to the `GameCharacter` itself (a circular reference), naive recursive deep-copying could enter an INFINITE LOOP — real-world deep-clone implementations often need to track "already cloned" objects (e.g., via an `IdentityHashMap`) to handle this correctly.

**Serialization-based deep cloning (an alternative technique):** some codebases implement deep cloning by SERIALIZING an object to bytes and then DESERIALIZING it back into a new object — this automatically handles arbitrarily deep/complex object graphs correctly (including circular references, in many serialization frameworks), at the cost of being significantly SLOWER than manual field-by-field cloning.

---

## 9. Before vs After

**Before (no Prototype — expensive/awkward re-construction):**

```java
public class GameCharacterFactoryBad {
    // Imagine constructing a fully-configured character from scratch
    // requires EXPENSIVE operations — e.g., loading default inventory
    // items from a database or config file EVERY single time
    public static GameCharacter createWarrior(String name, int level) {
        GameCharacter character = new GameCharacter(name, level);
        // Simulating an EXPENSIVE setup process repeated on EVERY creation
        character.addItem(loadItemFromDatabase("Sword"));
        character.addItem(loadItemFromDatabase("Shield"));
        return character;
    }

    private static String loadItemFromDatabase(String itemName) {
        // Simulated expensive I/O operation
        System.out.println("Expensive DB lookup for: " + itemName);
        return itemName;
    }
}
```

**Problems:**
- EVERY new warrior character re-triggers the EXPENSIVE "load items from database" operation, even though the DEFAULT starting inventory is IDENTICAL for every warrior — wasteful, repeated work.
- If the "warrior" starting configuration needs to change slightly, this logic (and the expensive setup calls) must be updated in ONE specific factory method, rather than simply updating ONE pre-configured template object.

**After (Prototype, as shown in Section 5):**
- The EXPENSIVE setup (loading/configuring `warriorTemplate`'s default inventory) happens EXACTLY ONCE, upfront.
- Every SUBSEQUENT character creation is a CHEAP clone operation (copying already-in-memory data), rather than repeating the expensive setup process each time.

---

## 10. SOLID Principles Connection

- **OCP**: new "template" prototypes can be introduced (e.g., a `MageTemplate`, `ArcherTemplate`) without modifying the `GameCharacter` class or its `clone()` logic — client code simply clones whichever pre-configured prototype is appropriate.
- **DIP**: client code can work with the COMMON `Cloneable`/prototype interface, calling `.clone()` WITHOUT needing to know the object's SPECIFIC concrete class — useful when a collection of DIFFERENT prototype subtypes need to be cloned uniformly.
- **SRP**: each Prototype class's `clone()` method has a FOCUSED responsibility — correctly duplicating ITS OWN state (deciding shallow vs deep per field) — separate from whatever BUSINESS logic the class also contains.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Prototype pattern solve, in your own words?
2. What's the difference between a SHALLOW copy and a DEEP copy?
3. Why might cloning an existing object be CHEAPER than constructing a new one from scratch?

**Intermediate:**
4. Why does `Object.clone()`'s DEFAULT behavior (a shallow copy) often need to be SUPPLEMENTED with manual deep-copying logic for specific fields?
5. What's a CIRCULAR REFERENCE, and why does it complicate deep-cloning logic? How might you handle it?
6. Compare Prototype with Builder (Topic 17) — both are creational patterns, but how do they differ in APPROACH to object creation?
   *Answer: Builder constructs an object STEP BY STEP from INDIVIDUAL configuration choices, typically STARTING from a "blank slate." Prototype creates a new object by COPYING an ALREADY-EXISTING, fully-configured instance, then OPTIONALLY modifying specific fields on the resulting copy. Builder is about ASSEMBLING from parts; Prototype is about DUPLICATING a whole, pre-configured instance.*

**Advanced:**
7. Why does Java's `Cloneable` interface/`Object.clone()` mechanism have a reputation for being somewhat "broken" or awkward compared to other languages' cloning mechanisms? What alternatives do experienced Java developers often prefer?
   *Answer: `Cloneable` is a MARKER interface with NO methods of its own — it merely signals to `Object.clone()` that cloning is ALLOWED, which is an unusual, non-obvious design. `clone()`'s SHALLOW-by-default behavior is easy to get WRONG (forgetting to deep-copy a mutable field is a common bug), and its checked-exception (`CloneNotSupportedException`) handling is often awkward. Many experienced developers PREFER alternative approaches: a dedicated COPY CONSTRUCTOR (e.g., `new GameCharacter(existingCharacter)`), a dedicated static factory method (`GameCharacter.copyOf(existing)`), or SERIALIZATION-based deep-copying for complex object graphs — all considered CLEANER and less error-prone than relying on `Object.clone()`.*
8. How would you implement a "Prototype Registry" — a central lookup (e.g., a `Map<String, GameCharacter>`) of NAMED, pre-configured prototypes, allowing client code to request a clone by NAME rather than holding direct references to prototype instances? Discuss how this connects Prototype with Factory-like lookup behavior.
9. How would you handle deep-cloning an object graph with CIRCULAR references correctly, avoiding infinite recursion? (Hint: track already-cloned objects using object IDENTITY, e.g., an `IdentityHashMap<Object, Object>`.)
10. In what scenario would a SERIALIZATION-based deep clone (serialize to bytes, then deserialize) be preferable to MANUAL field-by-field deep cloning, despite being slower?

---

## 12. Common Mistakes

- **Forgetting to deep-copy MUTABLE reference fields**, accidentally leaving the clone and original SHARING the same underlying mutable object — a classic, subtle bug where modifying one unexpectedly affects the other.
- **Assuming `Object.clone()`'s default (shallow) behavior is "good enough" without CAREFULLY reviewing EVERY field** — every reference-typed field needs a DELIBERATE decision: shallow (intentionally shared) or deep (intentionally independent)?
- **Not handling `CloneNotSupportedException` sensibly** — since implementing `Cloneable` should make this exception effectively unreachable, silently swallowing it (rather than wrapping it in an unchecked exception, as shown in Section 5) can hide genuine programming errors.
- **Choosing Prototype when a simple COPY CONSTRUCTOR would be clearer** — many experienced Java developers prefer a dedicated constructor (e.g., `new GameCharacter(existingCharacter)`) over `Object.clone()`'s awkward mechanics, for exactly the reasons discussed in Interview Question 7.

---

## 13. Time Complexity

- **Time:** O(n) for a deep copy, where n = the total size of the object graph being copied (proportional to the number of fields/nested objects that need duplicating) — versus whatever cost the ORIGINAL, from-scratch construction would have required (which could be MUCH higher for genuinely expensive setups, like database round-trips).
- **Space:** O(n) additional memory for the newly created clone's independent copy of the object graph.

---

## 14. Java API Examples

- **`Object.clone()` / `Cloneable`**: Java's own, built-in (if somewhat criticized) mechanism directly implementing this pattern.
- **`ArrayList(Collection<? extends E> c)`** (the copy constructor): creates a new `ArrayList` populated with COPIES of an existing collection's element REFERENCES (a shallow copy at the collection level) — a widely-used, simpler alternative to `Cloneable` for collections specifically.
- **`java.util.Date`'s `clone()` method**: a real, JDK-provided example of `Object.clone()`-based cloning in practice.
- **Serialization-based deep cloning**: using `ObjectOutputStream`/`ObjectInputStream` to serialize an object graph to bytes and deserialize it back, producing an independent DEEP copy automatically — a common technique for complex object graphs, despite the performance cost.

---

## 15. Practice Problem

Implement a **Prototype for a `Document` class** representing a text document with a `title`, `content` (String), and a `List<String> tags`. Implement `clone()` correctly, ensuring the `tags` list is DEEP-copied so that modifying a clone's tags does NOT affect the original document's tags. Demonstrate creating an original `Document`, cloning it, modifying the clone's title and adding a tag to the clone, and showing the original remains unaffected.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **Prototype Registry for a graphic design tool**: implement a `Shape` interface (implemented by `Circle`, `Rectangle`, etc., each with properties like color and dimensions) with a `clone()` method. Build a `ShapeRegistry` class that stores NAMED pre-configured prototype shapes (e.g., 'default-red-circle', 'default-blue-rectangle') in a `Map<String, Shape>`, and provides a `createShape(String prototypeName): Shape` method that returns a CLONE of the requested named prototype, ready for the caller to customize further (e.g., reposition, resize) WITHOUT affecting the stored prototype itself."

Think about:
- Why the `ShapeRegistry` itself must return a CLONE (not the original stored prototype reference) from `createShape()` — what would go wrong if it accidentally returned the SAME stored instance every time?
- How this registry-based design connects Prototype with Factory Method (Topic 8) conceptually — both provide a way to get a NEW object without the caller needing to use `new SpecificClass()` directly, but via fundamentally different underlying mechanisms (cloning vs. fresh construction).

---

## 17. Advanced LLD Scenario

**Design a Level Editor for a Video Game** using Prototype, where:
- Level designers need to place MANY instances of pre-configured "prefab" game objects (e.g., a specific type of enemy with pre-set health, AI behavior parameters, and equipped items; a specific type of collectible with pre-set value and visual appearance) throughout a level
- Each prefab may have DEEPLY NESTED configuration (e.g., an enemy has a `List<Item>` of equipped items, and each `Item` itself has its OWN nested stat-modifier objects) — requiring careful, THOROUGH deep-cloning to ensure placing 50 clones of the "same" enemy in a level doesn't result in all 50 enemies SHARING the same equipped-item objects (which would cause chaos if one enemy's item was later modified at runtime)
- Consider building a PROTOTYPE REGISTRY (as explored in Section 16) mapping prefab names to fully-configured template instances, allowing the level editor's UI to offer a palette of "drag and drop" prefabs

Consider:
- Why THOROUGH deep-cloning is absolutely CRITICAL in this scenario — a single missed shallow-copied nested field (e.g., forgetting to deep-copy the `List<Item>`, or forgetting that EACH `Item` inside it ALSO needs its own nested objects deep-copied) could cause SUBTLE, hard-to-diagnose bugs where seemingly independent enemies mysteriously affect each other at runtime
- How you might use a SERIALIZATION-based deep-clone approach (Interview Question 10) for this scenario specifically, given how DEEPLY nested prefab configurations can become — weighing this against the PERFORMANCE cost, especially if MANY clones need to be created rapidly during level loading
- How this scenario's "prefab palette" concept in game engines (a very real, industry-standard feature in engines like Unity) is a DIRECT, practical application of the Prototype pattern combined with a Registry, connecting this LLD concept to genuine, widely-used game development practice

---

## 18. Summary

**Definition:** Prototype creates new objects by copying (cloning) an existing, representative instance rather than constructing from scratch, allowing the clone to be independently modified afterward.

**Intent:** Avoid the cost of expensive object construction by copying existing, already-initialized state, and avoid tight coupling to specific concrete classes when the exact type isn't known until runtime.

**Key classes:** the `Cloneable`/custom prototype interface, and concrete classes implementing `clone()` with carefully considered shallow-vs-deep copy semantics for each field.

**Advantages:** Can be cheaper than full reconstruction; decouples cloning from specific concrete classes; reduces the need for extensive Factory hierarchies purely for pre-configured variants.

**Disadvantages:** Correct deep-cloning is genuinely tricky to implement; circular references complicate cloning; overuse can obscure where an object's "true" source of truth lives.

**Real-world use case:** Game engine "prefab" systems, document/configuration templates, prototype registries in CAD/graphics tools, `Object.clone()`/`Cloneable` itself.

**Java example:** `GameCharacter.clone()` using `super.clone()` for a shallow copy, then EXPLICITLY deep-copying the mutable `inventory` list to ensure true independence.

**Interview takeaway:** Be ready to discuss WHY `Object.clone()`/`Cloneable` is often considered awkward in Java, and what ALTERNATIVES (copy constructors, static factory copy methods, serialization-based cloning) experienced developers often prefer — this nuanced, critical perspective is a strong interview signal.

**One-line memory trick:** *"Photocopy a filled-out form instead of writing a new one from scratch — then just update the one or two fields that need to change."*

---

*End of Topic 18. Type "Next" to proceed to Topic 19: Bridge Pattern.*