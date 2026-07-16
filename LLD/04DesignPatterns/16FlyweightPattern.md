# Topic 21: Flyweight Pattern

---

## 1. Introduction

**Definition:**
The Flyweight Pattern is a **structural design pattern** that minimizes MEMORY usage by SHARING as much data as possible between MULTIPLE similar objects, rather than storing DUPLICATE copies of that shared data in EACH individual object — it works by splitting an object's state into INTRINSIC (shareable, common) state and EXTRINSIC (unique, context-specific) state.

**Why it exists / what problem it solves:**
Imagine a text editor that needs to represent EVERY single character on a page as an OBJECT (to track its font, size, color, position, etc.). If a page has 1 MILLION characters, and each character object STORES its own COMPLETE copy of font data (which might be a substantial, complex object), you'd end up with MASSIVE memory waste — because the MAJORITY of characters SHARE the exact SAME font, size, and style. Storing SEPARATE, DUPLICATE font data for EVERY single character object is enormously wasteful when THOUSANDS of characters share IDENTICAL font configurations.

Flyweight solves this by recognizing that the FONT DATA (typeface, size, style) is INTRINSIC — it's IDENTICAL and SHARED across MANY characters — while the character's POSITION on the page is EXTRINSIC — UNIQUE to each individual character instance. Flyweight creates and SHARES a SINGLE font-data object across ALL characters using that SAME font, while the POSITION (which genuinely differs per character) is passed in SEPARATELY each time it's needed, rather than being stored inside the shared object.

**When it should be used:**
- When an application needs to create a VERY LARGE NUMBER of similar objects, and the MEMORY cost of storing DUPLICATE shared data in each one is significant
- When MOST of an object's state can be made EXTRINSIC (passed in when needed) rather than stored, leaving only a SMALL amount of TRULY shared, INTRINSIC state to actually store in the shared Flyweight objects
- When object IDENTITY (each object being a DISTINCT, unique instance) isn't important — Flyweight objects are meant to be SHARED and are typically treated as IMMUTABLE

**When it should NOT be used:**
- When the NUMBER of objects isn't LARGE enough for memory savings to matter meaningfully — the added complexity of separating intrinsic/extrinsic state isn't worth it for a SMALL number of objects
- When MOST of an object's state is genuinely UNIQUE per instance (not shareable) — there's little to GAIN from Flyweight if there's little INTRINSIC state to actually share
- When objects need to be MUTABLE and INDEPENDENTLY modifiable — since Flyweight objects are SHARED, mutating one would incorrectly affect ALL OTHER objects sharing that SAME Flyweight instance

**Advantages:**
- Can DRAMATICALLY reduce memory usage when MANY similar objects share LARGE amounts of common data
- Centralizes SHARED data in ONE place, making it EASIER to update consistently (change the shared font data ONCE, and ALL objects using it are affected)
- Works naturally alongside a FACTORY (often called a "Flyweight Factory") that manages the POOL of shared Flyweight instances, ensuring instances are REUSED rather than duplicated

**Disadvantages:**
- Adds SIGNIFICANT design complexity — carefully splitting state into intrinsic vs extrinsic, and passing extrinsic state EXPLICITLY wherever it's needed, complicates the CODE compared to a simpler, non-shared design
- Introduces a TRADE-OFF: memory savings often come at the cost of SOME additional CPU overhead (e.g., passing extrinsic parameters, or looking up shared instances in a factory/pool)
- Flyweight objects being SHARED and typically IMMUTABLE can be a significant CONSTRAINT if the shared data genuinely needs to vary per-instance in ways not anticipated upfront

---

## 2. Real-World Analogy

Think of a **library's collection of books, versus each reader's own personal bookmark**.

A library doesn't print a BRAND NEW physical copy of "War and Peace" for EVERY single person who wants to read it — instead, the library maintains a SHARED pool of PHYSICAL BOOK COPIES (the INTRINSIC, shareable "content" — the actual text, the cover, the printing), and EACH reader who checks out a copy simply keeps track of THEIR OWN personal bookmark/progress (the EXTRINSIC, reader-specific "position" — which page they're currently on). The BOOK'S CONTENT is shared and reused across MANY readers; only the "where am I in this book" state is UNIQUE per reader, and it's tracked SEPARATELY from the book itself (e.g., on a bookmark, not printed permanently into the book).

---

## 3. Theory

**Core idea:** Split an object's total state into TWO categories: INTRINSIC state (data that's IDENTICAL across many instances and can therefore be SHARED — stored INSIDE the Flyweight object itself) and EXTRINSIC state (data that's UNIQUE per usage context — passed IN as a parameter WHENEVER an operation needs it, rather than stored in the Flyweight). A "Flyweight Factory" manages a POOL of these shared Flyweight instances, returning an EXISTING instance if one with the SAME intrinsic state already exists, rather than creating a NEW one.

**Working mechanism:**
```
FlyweightFactory.getFlyweight(intrinsicKey)
    → if a Flyweight with THIS intrinsic state already exists in the pool,
        return the EXISTING shared instance
    → otherwise, create a NEW Flyweight, store it in the pool, and return it

Client calls: flyweight.operation(extrinsicState)
    → extrinsic state is passed IN at call time, NOT stored in the Flyweight
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Intrinsic state | Shareable data STORED inside the Flyweight object (identical across usages) |
| Extrinsic state | Context-specific data passed IN by the client at call time (unique per usage) |
| Flyweight | The shared object storing ONLY intrinsic state |
| Flyweight Factory | Manages the pool of Flyweights, ensuring reuse instead of duplication |

**Class responsibilities:** the Flyweight is responsible for storing ONLY the shareable, intrinsic data, and its operations must ACCEPT extrinsic state as PARAMETERS rather than assuming it's stored internally. The Flyweight Factory is responsible for MANAGING the shared pool, ensuring identical intrinsic state maps to the SAME shared instance.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      Glyph                       │
├─────────────────────┤
│ + render(x, y): void          │  (x, y are EXTRINSIC —
└──────────┬──────────┘   passed in, NOT stored)
           △ (implements)
           │
┌──────────┴───────────────┐
│ CharacterGlyph (Flyweight)      │
├──────────────────────────┤
│ - character: char (INTRINSIC)      │
│ - fontFamily: String (INTRINSIC)   │
│ - fontSize: int (INTRINSIC)         │
├──────────────────────────┤
│ + render(x, y)                   │  (uses stored intrinsic
│   { print char at (x,y) }              state + passed-in x,y)
└──────────────────────────┘

┌─────────────────────┐
│      GlyphFactory                │
├─────────────────────┤
│ - pool: Map<String, Glyph>  ◇──→ (aggregation: Factory
├─────────────────────┤       manages/OWNS the SHARED
│ + getGlyph(char, font,           pool of Flyweight
│    size): Glyph                        instances)
│   (returns EXISTING glyph if
│    one with this intrinsic
│    state already exists, else
│    creates and caches a NEW one)
└─────────────────────┘

┌─────────────────────┐
│      Document (Client)          │
├─────────────────────┤
│ - glyphs: List<GlyphPosition>    │  (stores EXTRINSIC
├─────────────────────┤       position data SEPARATELY,
│ + render(): void                    alongside a REFERENCE
└─────────────────────┘       to the SHARED Glyph)
```

**Relationship explanation:**
- `CharacterGlyph` **implements** `Glyph` — its `render()` method takes `x, y` (EXTRINSIC) as PARAMETERS, while using its OWN stored `character`, `fontFamily`, `fontSize` (INTRINSIC) fields.
- `GlyphFactory` has an **aggregation** relationship with `Glyph` — it MAINTAINS the SHARED pool, ensuring that requesting a glyph with the SAME character/font/size returns the EXISTING shared instance rather than creating a duplicate.
- `Document` stores a LIST of (position, glyph-reference) PAIRS — the POSITIONS are unique per character occurrence in the document (EXTRINSIC, stored OUTSIDE the shared Glyph), while MANY entries in this list might reference the EXACT SAME shared `CharacterGlyph` instance (e.g., every occurrence of the letter "e" in 12pt Times New Roman shares ONE glyph object).

---

## 5. Java Implementation

```java
import java.util.HashMap;
import java.util.Map;
import java.util.ArrayList;
import java.util.List;

// ============================================
// Flyweight interface — operations accept EXTRINSIC state
// as parameters, rather than storing it internally
// ============================================
public interface Glyph {
    void render(int x, int y); // x, y are EXTRINSIC — passed in each call
}

// ============================================
// Concrete Flyweight — stores ONLY INTRINSIC (shareable) state
// ============================================
public class CharacterGlyph implements Glyph {
    // ALL of these are INTRINSIC — identical for every character
    // sharing the SAME character/font/size combination
    private final char character;
    private final String fontFamily;
    private final int fontSize;

    public CharacterGlyph(char character, String fontFamily, int fontSize) {
        this.character = character;
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
    }

    @Override
    public void render(int x, int y) {
        // Uses INTRINSIC state (character, font) stored in THIS object,
        // combined with EXTRINSIC state (x, y) passed in for THIS
        // SPECIFIC rendering call
        System.out.println("Rendering '" + character + "' [" + fontFamily
                + ", " + fontSize + "pt] at position (" + x + ", " + y + ")");
    }
}

// ============================================
// Flyweight Factory — manages the SHARED pool of Glyphs,
// ensuring REUSE instead of duplication
// ============================================
public class GlyphFactory {
    // Pool keyed by the COMBINATION of intrinsic state —
    // identical combinations map to the SAME shared instance
    private final Map<String, Glyph> pool = new HashMap<>();

    public Glyph getGlyph(char character, String fontFamily, int fontSize) {
        String key = character + "-" + fontFamily + "-" + fontSize;

        // Return the EXISTING shared instance if one already exists
        // for this EXACT intrinsic-state combination
        if (!pool.containsKey(key)) {
            System.out.println("Creating NEW glyph for key: " + key);
            pool.put(key, new CharacterGlyph(character, fontFamily, fontSize));
        }
        return pool.get(key);
    }

    public int getPoolSize() {
        return pool.size();
    }
}

// ============================================
// Client — Document stores EXTRINSIC positions alongside
// REFERENCES to SHARED Glyph instances
// ============================================
public class Document {
    private final GlyphFactory glyphFactory = new GlyphFactory();
    // Each entry pairs a SHARED Glyph reference with its
    // UNIQUE, EXTRINSIC position in the document
    private final List<GlyphPlacement> placements = new ArrayList<>();

    private static class GlyphPlacement {
        Glyph glyph;
        int x, y;
        GlyphPlacement(Glyph glyph, int x, int y) {
            this.glyph = glyph; this.x = x; this.y = y;
        }
    }

    public void addCharacter(char c, String font, int size, int x, int y) {
        // Requests a (possibly SHARED) Glyph from the factory —
        // this document doesn't create duplicate glyph objects
        // for repeated characters using the SAME font/size
        Glyph glyph = glyphFactory.getGlyph(c, font, size);
        placements.add(new GlyphPlacement(glyph, x, y));
    }

    public void render() {
        for (GlyphPlacement p : placements) {
            // EXTRINSIC position (p.x, p.y) is passed IN at render time
            p.glyph.render(p.x, p.y);
        }
    }

    public int getUniqueGlyphCount() {
        return glyphFactory.getPoolSize();
    }
}

// ============================================
// Demo
// ============================================
public class FlyweightDemo {
    public static void main(String[] args) {
        Document doc = new Document();

        // "hello" — 5 characters, but 'l' repeats — same font/size
        doc.addCharacter('h', "Arial", 12, 0, 0);
        doc.addCharacter('e', "Arial", 12, 10, 0);
        doc.addCharacter('l', "Arial", 12, 20, 0);
        doc.addCharacter('l', "Arial", 12, 30, 0); // SAME intrinsic state as previous 'l'
        doc.addCharacter('o', "Arial", 12, 40, 0);

        doc.render();

        // Even though 5 characters were added, only 4 UNIQUE glyph
        // objects were actually created (both 'l's SHARE one instance)
        System.out.println("Total characters placed: 5");
        System.out.println("Unique glyph objects created: " + doc.getUniqueGlyphCount());
    }
}
```

**Key line-by-line notes:**
- `CharacterGlyph.render(int x, int y)` takes `x, y` as PARAMETERS — this is the CRUCIAL design decision that keeps position OUT of the shared object's stored state, allowing the SAME `CharacterGlyph` instance to be rendered at MANY different positions.
- `GlyphFactory.getGlyph()` checks the pool BEFORE creating a new instance — this is what ACTUALLY achieves the memory savings; without this check-and-reuse logic, EVERY call would create a wasteful NEW object even for IDENTICAL intrinsic state.
- `Document`'s `placements` list stores the EXTRINSIC `x, y` position ALONGSIDE a reference to the (possibly SHARED) `Glyph` — this is where the "MISSING" state (position) that was DELIBERATELY excluded from the Flyweight itself ACTUALLY lives.

---

## 6. Dry Run

**Sample input:** adding two 'l' characters, both in "Arial" 12pt, at DIFFERENT positions

```
1. doc.addCharacter('l', "Arial", 12, 20, 0) [FIRST 'l']
   → glyphFactory.getGlyph('l', "Arial", 12) called
   → key = "l-Arial-12"
   → pool.containsKey("l-Arial-12")? NO
   → Prints: "Creating NEW glyph for key: l-Arial-12"
   → new CharacterGlyph('l', "Arial", 12) created, stored in pool
   → getGlyph() returns this NEW glyph instance (call it glyphL)
   → placements.add(new GlyphPlacement(glyphL, 20, 0))

2. doc.addCharacter('l', "Arial", 12, 30, 0) [SECOND 'l']
   → glyphFactory.getGlyph('l', "Arial", 12) called
   → key = "l-Arial-12" (SAME key as before)
   → pool.containsKey("l-Arial-12")? YES (from step 1)
   → NO new object created — the EXISTING glyphL instance is returned
   → placements.add(new GlyphPlacement(glyphL, 30, 0))
   → NOTE: this SAME glyphL reference is now stored TWICE in
     placements, but with DIFFERENT (x, y) values each time

3. doc.render() eventually calls:
   → placements[2].glyph.render(20, 0) → renders 'l' at (20, 0)
   → placements[3].glyph.render(30, 0) → renders 'l' at (30, 0)
   → BOTH calls use the SAME glyphL object, but produce DIFFERENT
     output because (x, y) is passed IN EACH TIME as EXTRINSIC state
```

**What's happening in memory:** only ONE `CharacterGlyph` object exists for 'l'-Arial-12pt, REGARDLESS of how many TIMES that character/font/size combination appears in the document. The `placements` list holds MULTIPLE small `GlyphPlacement` entries (each just a reference + two ints), but they all POINT to the SAME shared `CharacterGlyph` object when the intrinsic state matches — this is precisely the memory-saving mechanism Flyweight provides.

---

## 7. Real-World Software Example

- **Text editors / word processors**: representing EACH character on a page as an object, SHARING font/style data across characters using the SAME formatting (directly mirroring this example).
- **`java.lang.Integer.valueOf()` caching**: Java CACHES and REUSES `Integer` objects for values between -128 and 127 (autoboxing), avoiding REDUNDANT object creation for these COMMON, small integer values — a genuine, built-in Flyweight-style optimization.
- **`String` interning (`String.intern()`)**: Java's STRING POOL reuses IDENTICAL string literal objects across the application, rather than creating DUPLICATE `String` objects for the SAME text content.
- **Game development — particle systems / tile-based maps**: THOUSANDS of visually IDENTICAL game objects (e.g., grass tiles, bullet particles) SHARE the SAME underlying sprite/texture data (intrinsic), while each individual INSTANCE only tracks its OWN position/velocity (extrinsic).
- **GUI icon/rendering caches**: reusing SHARED icon/image objects across MANY UI elements that display the SAME icon, rather than loading/storing DUPLICATE copies.

---

## 8. Internal Working

**Object creation:** the Flyweight Factory's POOL (typically a `Map`) is checked BEFORE any new Flyweight object is created — this LOOKUP-then-create-if-absent logic is the CENTRAL mechanism enabling reuse; without it, Flyweight provides NO benefit at all.

**Memory layout:** the KEY memory saving comes from HAVING FEWER total Flyweight objects in memory than there are LOGICAL "usages" of that data — e.g., a MILLION characters in a document might only require a FEW HUNDRED unique `CharacterGlyph` objects (one per unique character/font/size combination actually used), with the REMAINING million-minus-a-few-hundred "usages" simply being REFERENCES to these shared objects, plus their own SMALL extrinsic data.

**Extrinsic state handling:** extrinsic state is NEVER stored inside the Flyweight — it must be PASSED IN as a parameter to EVERY operation that needs it, and the CLIENT (or an intermediate structure, like `Document`'s `placements` list) is responsible for TRACKING and SUPPLYING this extrinsic state correctly for each USAGE context.

---

## 9. Before vs After

**Before (no Flyweight — duplicated state per object):**

```java
public class CharacterGlyphBad {
    private char character;
    private String fontFamily;
    private int fontSize;
    private int x; // EXTRINSIC state INCORRECTLY stored INSIDE the object
    private int y;

    public CharacterGlyphBad(char character, String fontFamily, int fontSize, int x, int y) {
        this.character = character;
        this.fontFamily = fontFamily;
        this.fontSize = fontSize;
        this.x = x;
        this.y = y;
    }
    // Each of these fields is DUPLICATED for EVERY SINGLE character
    // object created — even when MANY characters share the EXACT
    // SAME font/size, EACH one stores its OWN separate copy
}

// For a document with 1,000,000 characters, this creates
// 1,000,000 SEPARATE objects, EACH storing its OWN COPY of
// font/size data — even though MOST characters likely share
// just a HANDFUL of distinct font/size combinations
```

**Problems:**
- EVERY character object stores its OWN COPY of `fontFamily` and `fontSize`, even though THOUSANDS of characters might share the EXACT SAME values — massive, avoidable memory DUPLICATION.
- If the SHARED font data needs updating (e.g., correcting a typo in a font family name used throughout), it must be updated INDIVIDUALLY in EVERY object that happens to reference it, rather than in ONE shared location.
- For LARGE documents, this approach can consume FAR more memory than necessary, potentially causing PERFORMANCE issues or even out-of-memory conditions at sufficient SCALE.

**After (Flyweight, as shown in Section 5):**
- INTRINSIC state (character, font, size) is stored ONCE per UNIQUE combination, in a SHARED `CharacterGlyph` object, REUSED across ALL characters with matching intrinsic state.
- EXTRINSIC state (position) is stored SEPARATELY, in LIGHTWEIGHT `GlyphPlacement` entries — MUCH cheaper than duplicating the ENTIRE glyph object per occurrence.

---

## 10. SOLID Principles Connection

- **SRP**: the Flyweight (`CharacterGlyph`) is responsible ONLY for INTRINSIC data/rendering logic; the Factory is responsible ONLY for MANAGING the shared pool; the Client (`Document`) is responsible for TRACKING extrinsic state — these are cleanly SEPARATED concerns.
- **OCP**: NEW types of Flyweights (e.g., supporting a NEW font family) can be added WITHOUT modifying EXISTING Flyweight instances or the Factory's core pooling LOGIC.
- **Encapsulation tension (worth noting honestly)**: Flyweight REQUIRES the client (or an intermediate structure) to EXPLICITLY manage and PASS extrinsic state — this is a DELIBERATE trade-off, somewhat INCREASING coupling between the client and the Flyweight's INTERNAL state-splitting design, in EXCHANGE for the memory savings gained.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Flyweight pattern solve, in your own words?
2. What's the difference between INTRINSIC and EXTRINSIC state?
3. Why is a Flyweight Factory typically needed ALONGSIDE the Flyweight objects themselves?

**Intermediate:**
4. Why must Flyweight objects generally be treated as IMMUTABLE? What would go WRONG if a shared Flyweight's intrinsic state were MODIFIED after creation?
5. Give an example of Flyweight-style optimization ALREADY built into the JDK (e.g., `Integer` caching, `String` interning).
6. How does the "check the pool BEFORE creating a new instance" logic in the Flyweight Factory relate conceptually to SINGLETON (Topic 16)? What's the KEY structural difference?
   *Answer: Singleton guarantees EXACTLY ONE instance for an ENTIRE class. A Flyweight Factory guarantees ONE SHARED instance PER UNIQUE COMBINATION of intrinsic state — meaning there can be MANY DIFFERENT Flyweight instances overall (one per unique character/font/size combo, for example), but NO DUPLICATES for the SAME combination. Flyweight is like having "MULTIPLE mini-Singletons," ONE per distinct intrinsic-state key, rather than ONE single global instance.*

**Advanced:**
7. How would you decide WHERE to draw the line between INTRINSIC and EXTRINSIC state for a GIVEN object? What questions would guide this decision?
   *Answer: Ask: "Is this piece of data IDENTICAL across MANY instances/usages, or does it genuinely VARY per usage?" Data that's SHARED and UNCHANGING across usages belongs INSIDE the Flyweight (intrinsic); data that's UNIQUE per usage context must be PASSED IN externally (extrinsic). A useful heuristic: if STORING a piece of data inside EVERY instance would mean storing MANY IDENTICAL COPIES of the SAME value, it's a strong CANDIDATE for being intrinsic instead.*
8. Discuss the memory-vs-CPU TRADE-OFF inherent in Flyweight — what ADDITIONAL costs (lookup overhead, extra parameter passing) does the pattern introduce in EXCHANGE for memory savings?
9. How would you make a Flyweight Factory THREAD-SAFE, ensuring CONCURRENT requests for the SAME intrinsic-state combination don't accidentally create DUPLICATE instances due to a race condition (similar to Singleton's thread-safety concerns from Topic 16)?
10. Could Flyweight be COMBINED with Prototype (Topic 18)? Discuss a scenario where you might CLONE a Flyweight's intrinsic state to create a NEW, independent (non-shared) variant.

---

## 12. Common Mistakes

- **Accidentally storing EXTRINSIC state INSIDE the Flyweight object** — this is the MOST COMMON mistake, and it COMPLETELY defeats the pattern's purpose, since it means EACH "usage" would need its OWN Flyweight instance again (since extrinsic state DIFFERS per usage), eliminating the sharing benefit entirely.
- **Applying Flyweight to a SMALL number of objects where memory savings are NEGLIGIBLE** — the ADDED design complexity isn't justified unless the SCALE of object creation is genuinely LARGE.
- **Making Flyweight objects MUTABLE** — since Flyweight instances are SHARED across MANY usages, mutating ONE would incorrectly affect ALL OTHER usages sharing that SAME instance; Flyweights should be treated as EFFECTIVELY IMMUTABLE once created.
- **Forgetting THREAD-SAFETY in the Flyweight Factory's pool** — CONCURRENT access without proper synchronization could lead to DUPLICATE instances being created for the SAME intrinsic-state key, similar to Singleton's classic race-condition bug (Topic 16).

---

## 13. Time Complexity

- **Time:** O(1) average for `getGlyph()` pool lookups (assuming a HASH-based pool implementation) — a SMALL, CONSTANT overhead compared to direct object creation, in EXCHANGE for the memory savings.
- **Space:** O(u) for the Flyweight pool, where u = the number of UNIQUE intrinsic-state combinations actually used (potentially FAR smaller than the TOTAL number of logical object "usages," which is the ENTIRE POINT of the pattern) + O(n) for the EXTRINSIC state stored per usage (typically much CHEAPER than full object duplication).

---

## 14. Java API Examples

- **`Integer.valueOf(int)`** (and similar for `Byte`, `Short`, `Long`, `Character`): CACHES and REUSES boxed objects for small, COMMON values (typically -128 to 127), avoiding REDUNDANT object creation — direct Flyweight-style behavior built into the JDK.
- **`String.intern()`**: adds a string to (or retrieves an EXISTING equal string FROM) the JVM's STRING POOL, ensuring IDENTICAL string content is REPRESENTED by the SAME shared object.
- **`java.lang.Boolean.valueOf(boolean)`**: returns ONE of exactly TWO cached, SHARED `Boolean` instances (`Boolean.TRUE`/`Boolean.FALSE`) rather than creating NEW objects.
- **Font/Glyph rendering in AWT/Swing**: internally, MANY graphics systems SHARE font/glyph rendering data across TEXT elements using the SAME font, conceptually mirroring this pattern's INTENT.

---

## 15. Practice Problem

Implement a **Flyweight for a forest rendering system**: create a `TreeType` Flyweight class storing INTRINSIC data (`name`, `color`, `textureData` — simulate this as a `String` representing expensive texture data). Implement a `TreeTypeFactory` managing a POOL of `TreeType` instances. Implement a `Tree` class representing an INDIVIDUAL tree INSTANCE in the forest, storing EXTRINSIC data (`x`, `y` position) PLUS a REFERENCE to its SHARED `TreeType`. Demonstrate placing 10,000 trees using only 3 DISTINCT `TreeType`s, and print how MANY actual `TreeType` objects were created versus the TOTAL number of trees placed.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **Flyweight-based system for rendering a large-scale MMO game map** containing MILLIONS of terrain tiles (grass, water, rock, sand). Each terrain TYPE has EXPENSIVE-to-load texture and physics data (intrinsic), while EACH individual tile INSTANCE on the map needs its OWN position and, POSSIBLY, minor per-tile variations (e.g., slight color tinting for visual variety). Design how you'd handle the 'minor per-tile variation' requirement WITHOUT breaking the Flyweight pattern's core memory-saving benefit."

Think about:
- Whether "slight color tinting" should be considered INTRINSIC (requiring MANY MORE distinct Flyweight variants, one per tint value) or EXTRINSIC (passed in at RENDER time, alongside position) — which choice PRESERVES more of the memory-saving benefit, and what's the TRADE-OFF?
- How you'd decide the THRESHOLD at which a "would-be extrinsic" variation is SMALL/DISCRETE enough (e.g., only 5 possible tint levels) that it might ALTERNATIVELY be modeled as ADDITIONAL intrinsic Flyweight variants instead, versus being GENUINELY continuous/unique enough that it MUST be extrinsic.

---

## 17. Advanced LLD Scenario

**Design a Ride-Sharing App's Real-Time Map Rendering System** using Flyweight, where:
- The map needs to render THOUSANDS of vehicle ICONS simultaneously (cars, bikes, scooters), each icon SHARING the SAME visual asset (image/sprite data) PER vehicle TYPE — this is clearly INTRINSIC, shareable data
- EACH individual vehicle on the map has its OWN, CONSTANTLY updating (x, y) position, rotation/heading angle, and a UNIQUE driver ID — clearly EXTRINSIC, per-instance data that CHANGES FREQUENTLY (every few seconds, as GPS updates arrive)
- Consider the PERFORMANCE implications of THOUSANDS of vehicles UPDATING their extrinsic position DATA every few seconds — since this EXTRINSIC state changes CONSTANTLY while the INTRINSIC Flyweight (the shared icon) remains STABLE, how does this affect your DESIGN of the "placement" data structure (analogous to `Document`'s `placements` list in Section 5)?

Consider:
- Why this scenario is an EXCELLENT candidate for Flyweight — a RELATIVELY SMALL number of DISTINCT vehicle TYPES/icons (intrinsic, expensive visual assets) versus a POTENTIALLY HUGE and CONSTANTLY CHANGING number of INDIVIDUAL vehicle instances (extrinsic position data)
- How the FREQUENT UPDATES to extrinsic state (position, changing every few seconds) reinforces WHY this data should NEVER be baked into the shared Flyweight object itself — if position were INTRINSIC, updating ONE vehicle's position would incorrectly affect EVERY OTHER vehicle SHARING that SAME icon Flyweight
- How this scenario, together with the text-editor and forest-rendering examples, demonstrates a RECURRING pattern: Flyweight shines specifically when there's a SMALL, STABLE set of "types" (shared, intrinsic) against a LARGE, DYNAMIC set of "instances" (unique, extrinsic) — recognizing this SHAPE quickly across DIFFERENT domains is a hallmark of strong LLD interview performance

---

## 18. Summary

**Definition:** Flyweight minimizes memory usage by sharing common (intrinsic) state across many similar objects, while keeping instance-specific (extrinsic) state separate, passed in only when needed.

**Intent:** Support a LARGE number of fine-grained objects efficiently, by SHARING as much data as possible rather than duplicating it across every instance.

**Key classes:** `Flyweight` (stores intrinsic state, accepts extrinsic state as parameters), `Flyweight Factory` (manages the shared pool, ensuring reuse), `Client` (tracks and supplies extrinsic state alongside Flyweight references).

**Advantages:** Dramatic memory savings for large numbers of similar objects; centralizes shared data for easy updates; naturally pairs with a managing Factory.

**Disadvantages:** Significant design complexity in splitting intrinsic/extrinsic state; memory savings trade off against some CPU/lookup overhead; Flyweights must be treated as immutable.

**Real-world use case:** Text editor character/glyph rendering, `Integer`/`String` caching in Java, game engine tile/particle systems, map rendering with shared icons.

**Java example:** `GlyphFactory` sharing `CharacterGlyph` instances across identical character/font/size combinations, with position passed in as extrinsic state at render time.

**Interview takeaway:** Be ready to clearly articulate the INTRINSIC vs EXTRINSIC state distinction with a concrete example, and to explain WHY Flyweight objects must remain IMMUTABLE — this is the crux of understanding the pattern correctly, versus merely reciting "it saves memory" without grasping the underlying mechanism.

**One-line memory trick:** *"A library shares ONE physical copy of a book across MANY readers — only each reader's OWN bookmark (their reading position) is unique to them."*

---

*End of Topic 21. Type "Next" to proceed to Topic 22: Command Pattern.*