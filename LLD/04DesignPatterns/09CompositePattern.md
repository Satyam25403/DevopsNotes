# Topic 14: Composite Pattern

---

## 1. Introduction

**Definition:**
The Composite Pattern is a **structural design pattern** that lets you compose objects into TREE structures to represent PART-WHOLE hierarchies, and treat INDIVIDUAL objects (leaves) and COMPOSITIONS of objects (composites/branches) UNIFORMLY through a common interface — client code doesn't need to distinguish between "a single item" and "a group of items."

**Why it exists / what problem it solves:**
Consider a file system: a `File` is a single, simple item, while a `Folder` can contain OTHER files AND folders (recursively). If client code needs to compute the "total size" of something, it shouldn't need to write separate logic for "if it's a file, do X; if it's a folder, do Y (which requires looping through its children, possibly recursively)." This kind of type-checking scattered throughout client code becomes unwieldy fast, especially as the hierarchy gets deeper.

Composite solves this by defining a COMMON interface (e.g., `FileSystemItem`) that BOTH `File` and `Folder` implement. A `Folder` internally holds a list of CHILD `FileSystemItem`s (which could be more Files OR more Folders) and implements operations (like `getSize()`) by DELEGATING to its children and combining their results — recursively. Client code simply calls `getSize()` on ANYTHING, without caring whether it's a single File or an entire nested Folder tree.

**When it should be used:**
- When you need to represent PART-WHOLE hierarchies (trees) of objects
- When you want client code to treat INDIVIDUAL objects and GROUPS of objects UNIFORMLY, via the same interface
- When operations need to be applied RECURSIVELY across a tree structure (e.g., "total size," "render everything," "count all items")

**When it should NOT be used:**
- When there's no genuine tree/hierarchy structure — if your data is fundamentally FLAT, Composite adds unnecessary complexity
- When leaves and composites have SIGNIFICANTLY different capabilities that can't be sensibly unified under one interface (forcing an unnatural common interface can violate ISP — see Section 10)
- When you need STRICT type safety distinguishing leaves from composites at compile time — Composite's uniform interface can make this harder to enforce

**Advantages:**
- Client code is dramatically simplified — treats a single object and an entire subtree identically
- Makes it easy to add NEW kinds of components (new leaf types, or even new composite types) without breaking existing client code
- Naturally supports RECURSIVE operations across arbitrarily deep hierarchies

**Disadvantages:**
- Can make the design OVERLY general — it may become hard to restrict what kinds of children a specific composite is ALLOWED to contain (e.g., preventing a `File` from having children at all requires care)
- Some operations genuinely make more sense for ONLY leaves or ONLY composites, but the pattern's uniform interface can force awkward "no-op" or exception-throwing implementations for the other type (see Common Mistakes)

---

## 2. Real-World Analogy

Think of a **file system, exactly as it works on your own computer**.

A single music file (`song.mp3`) has a size, say 5 MB. A folder (`Music/`) containing THAT song file, plus another folder (`Playlists/`) which itself contains more files — the WHOLE folder ALSO has a "size," but it's computed by adding up the sizes of everything INSIDE it, recursively, however deeply nested it is.

Critically: when you right-click a file OR a folder and select "Properties" to see its size, you're using the EXACT SAME action on both — you don't need a special "Folder Properties" command versus a "File Properties" command. The operating system treats files and folders UNIFORMLY through the same "get size" operation, even though internally a folder's size calculation is fundamentally different (recursive) from a file's (direct/stored value).

---

## 3. Theory

**Core idea:** Define a common `Component` interface declaring operations relevant to BOTH individual items (Leaves) and groups (Composites). Leaf classes implement these operations directly. Composite classes implement them by ITERATING over their children and DELEGATING/aggregating results — since children are ALSO `Component`s, this delegation works recursively no matter how deep the tree goes.

**Working mechanism:**
```
           Component (interface)
          /                    \
       Leaf                  Composite
   (implements directly)   (holds List<Component> children,
                             delegates + combines results)
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Component | The common interface/abstract class shared by Leaf and Composite |
| Leaf | A single, indivisible object with no children |
| Composite | A container object that holds child Components (Leaves and/or other Composites) |
| Child | Any Component contained within a Composite |

**Class responsibilities:** a Leaf is responsible for implementing operations DIRECTLY based on its own data. A Composite is responsible for MANAGING its children (add/remove) and implementing operations by DELEGATING to each child and combining their results, without needing to know whether any given child is itself a Leaf or another Composite.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      FileSystemItem            │
├─────────────────────┤
│ + getSize(): long             │
│ + display(indent): void       │
└──────────┬──────────┘
           △ (implements)
   ┌───────┴────────────────┐
   │                                │
┌──┴────────────┐   ┌──┴──────────────────┐
│ File                     │   │ Folder                        │
├───────────────┤   ├────────────────────┤
│ - name: String              │   │ - name: String                    │
│ - size: long                   │   │ - children: List<FileSystemItem>  ◇──→ (aggregation:
│ + getSize()                   │   │ + add(FileSystemItem)                Folder holds
│   { return size; }            │   │ + remove(FileSystemItem)             MULTIPLE children,
└───────────────┘   │ + getSize()                        each of which is
                       │   {loop over children,              itself a FileSystemItem
                       │    sum their getSize()}             — Leaf OR another Folder)
                       └────────────────────┘
```

**Relationship explanation:**
- `File` (Leaf) and `Folder` (Composite) both **implement** `FileSystemItem` — client code interacting with EITHER uses the SAME interface, with no need to check which one it's dealing with.
- `Folder` has an **aggregation** relationship with `FileSystemItem` (via its `children` list) — a Folder doesn't OWN its children in an exclusive lifecycle sense (they could theoretically be moved elsewhere), and critically, each CHILD could be a `File` OR ANOTHER `Folder`, enabling the RECURSIVE tree structure.
- This recursive self-reference (`Folder` containing `FileSystemItem`s, one possible concrete type of which is `Folder` itself) is the STRUCTURAL heart of the Composite pattern — it's what allows arbitrarily deep nesting.

---

## 5. Java Implementation

```java
import java.util.ArrayList;
import java.util.List;

// ============================================
// Component interface — common to Leaf and Composite
// ============================================
public interface FileSystemItem {
    long getSize();
    void display(String indent);
}

// ============================================
// Leaf — a single file, no children
// ============================================
public class File implements FileSystemItem {
    private final String name;
    private final long size; // in bytes

    public File(String name, long size) {
        this.name = name;
        this.size = size;
    }

    @Override
    public long getSize() {
        // Direct, simple computation — no delegation needed
        return size;
    }

    @Override
    public void display(String indent) {
        System.out.println(indent + "- " + name + " (" + size + " bytes)");
    }
}

// ============================================
// Composite — a folder containing other FileSystemItems
// (which may themselves be Files or other Folders)
// ============================================
public class Folder implements FileSystemItem {
    private final String name;
    // Children can be ANY mix of File and Folder objects
    private final List<FileSystemItem> children = new ArrayList<>();

    public Folder(String name) {
        this.name = name;
    }

    public void add(FileSystemItem item) {
        children.add(item);
    }

    public void remove(FileSystemItem item) {
        children.remove(item);
    }

    @Override
    public long getSize() {
        long totalSize = 0;
        // Delegates to EACH child's getSize() — if a child is itself
        // a Folder, this recursively triggers ITS own summation logic,
        // and so on, however deep the nesting goes
        for (FileSystemItem child : children) {
            totalSize += child.getSize();
        }
        return totalSize;
    }

    @Override
    public void display(String indent) {
        System.out.println(indent + "+ " + name + "/ (" + getSize() + " bytes total)");
        // Recursively display every child, indented one level deeper
        for (FileSystemItem child : children) {
            child.display(indent + "  ");
        }
    }
}

// ============================================
// Demo — client code treats File and Folder UNIFORMLY
// ============================================
public class CompositeDemo {
    public static void main(String[] args) {
        File song1 = new File("song1.mp3", 5_000_000);
        File song2 = new File("song2.mp3", 4_500_000);
        File playlistFile = new File("favorites.m3u", 2_000);

        Folder musicFolder = new Folder("Music");
        musicFolder.add(song1);
        musicFolder.add(song2);

        Folder playlistsFolder = new Folder("Playlists");
        playlistsFolder.add(playlistFile);

        // A Folder can contain OTHER Folders — recursive nesting
        Folder rootFolder = new Folder("MyDocuments");
        rootFolder.add(musicFolder);
        rootFolder.add(playlistsFolder);

        // Client code calls getSize() and display() IDENTICALLY,
        // whether on a single File or an entire nested Folder tree
        rootFolder.display("");
        System.out.println("Total size: " + rootFolder.getSize() + " bytes");
    }
}
```

**Key line-by-line notes:**
- `Folder.getSize()` doesn't check "is this child a File or a Folder?" anywhere — it simply calls `child.getSize()` on EVERY child, and dynamic dispatch correctly resolves to `File.getSize()` (direct value) or `Folder.getSize()` (recursive sum) as appropriate.
- The `children` list is typed as `List<FileSystemItem>`, NOT `List<File>` or `List<Folder>` — this is what enables a `Folder` to contain an ARBITRARY MIX of both, at any depth.
- `display()` recursively calls itself on children with an increasing `indent` string — demonstrating how Composite naturally supports operations that need to traverse the ENTIRE tree structure.

---

## 6. Dry Run

**Sample input:** `rootFolder.getSize()` from the demo above (rootFolder contains musicFolder [song1: 5,000,000 + song2: 4,500,000] and playlistsFolder [playlistFile: 2,000])

```
1. rootFolder.getSize() called
   → Folder.getSize() executes on rootFolder
   → totalSize = 0
   → Loop over rootFolder's children: [musicFolder, playlistsFolder]

2. child = musicFolder → musicFolder.getSize() called
   → Dynamic dispatch resolves to Folder.getSize() (musicFolder IS a Folder)
   → totalSize (local to THIS call) = 0
   → Loop over musicFolder's children: [song1, song2]
       → song1.getSize() → dispatch resolves to File.getSize() → returns 5,000,000
       → totalSize += 5,000,000 → totalSize = 5,000,000
       → song2.getSize() → dispatch resolves to File.getSize() → returns 4,500,000
       → totalSize += 4,500,000 → totalSize = 9,500,000
   → musicFolder.getSize() returns 9,500,000

3. Back in rootFolder.getSize(): totalSize += 9,500,000 → totalSize = 9,500,000

4. child = playlistsFolder → playlistsFolder.getSize() called
   → Dynamic dispatch resolves to Folder.getSize() (playlistsFolder IS a Folder)
   → Loop over playlistsFolder's children: [playlistFile]
       → playlistFile.getSize() → dispatch resolves to File.getSize() → returns 2,000
       → totalSize (local) = 2,000
   → playlistsFolder.getSize() returns 2,000

5. Back in rootFolder.getSize(): totalSize += 2,000 → totalSize = 9,502,000
   → rootFolder.getSize() returns 9,502,000
```

**What's happening in memory:** this is a classic RECURSIVE call pattern — each `Folder.getSize()` call pushes a NEW stack frame for every nested Folder encountered, with the innermost (deepest) Folders/Files resolving FIRST and their results bubbling back UP through the call stack as each frame's loop continues and accumulates `totalSize`.

---

## 7. Real-World Software Example

- **File systems** (as in this example): files and folders, `du`/`Get-ChildItem`-style size calculations, recursive directory traversal.
- **GUI component trees**: a `Panel` (composite) can contain `Button`s, `Label`s (leaves), and OTHER `Panel`s — operations like `render()` or `getBounds()` are computed uniformly across the whole tree, recursively.
- **Organization charts**: an `Employee` (leaf, individual contributor) versus a `Manager` (composite, has direct reports who are themselves `Employee`s or other `Manager`s) — operations like "total headcount under this person" or "total salary budget" are computed recursively.
- **XML/HTML DOM trees**: an `Element` node can contain child `Element`s or text nodes — operations like `render()`, `getTextContent()` work uniformly across the tree.
- **Menu systems**: a `MenuItem` (leaf) versus a `Menu` (composite containing more `MenuItem`s or sub-`Menu`s) — rendering the entire menu structure works uniformly regardless of nesting depth.

---

## 8. Internal Working

**Object creation:** the tree is typically built INCREMENTALLY via `add()` calls — a `Composite`'s children list starts empty and grows as items/subtrees are attached; there's no requirement for the whole tree to be constructed atomically.

**Runtime interactions / call flow:** operations on a Composite ALWAYS follow the shape "loop over children, call the SAME operation on each, combine results" — this loop-and-delegate structure is what makes the RECURSION happen naturally, since some children may themselves be Composites that repeat the exact same loop-and-delegate logic.

**Memory usage:** a Composite holds a COLLECTION (e.g., `List<FileSystemItem>`) of REFERENCES to its children — the actual child objects may be shared or exist independently; the tree's total memory is proportional to the total number of nodes (Leaves + Composites) across the whole structure.

**Call stack depth:** for deeply recursive operations (like `getSize()` traversing a very deep tree), the call stack depth is proportional to the TREE'S DEPTH, not the total number of nodes — a very "bushy" but SHALLOW tree uses much less stack space than a very DEEP but narrow one.

---

## 9. Before vs After

**Before (no Composite — explicit type-checking everywhere):**

```java
public class FileSystemCalculatorBad {
    // Client code must EXPLICITLY distinguish File from Folder
    // EVERYWHERE it needs to compute a size — extremely fragile
    public long getTotalSize(Object item) {
        if (item instanceof File) {
            return ((File) item).getRawSize();
        } else if (item instanceof FolderBad) {
            FolderBad folder = (FolderBad) item;
            long total = 0;
            for (Object child : folder.getChildren()) {
                // RECURSIVE call, but the type-checking logic
                // must be repeated at EVERY level, and any NEW
                // type of "file system item" added later requires
                // updating THIS method too
                total += getTotalSize(child);
            }
            return total;
        }
        throw new IllegalArgumentException("Unknown item type");
    }
}
```

**Problems:**
- EVERY operation (not just size — display, search, delete, etc.) needs its OWN copy of this `instanceof` type-checking logic, scattered throughout client code.
- Adding a NEW type of file-system item (e.g., a `SymbolicLink`) requires editing EVERY SUCH method across the ENTIRE codebase — severe OCP violation.
- The recursive traversal logic is tangled together WITH the type-checking, making both harder to read and maintain.

**After (Composite, as shown in Section 5):**
- `Folder.getSize()` and `File.getSize()` each contain ONLY their own logic — no type-checking, no `instanceof` anywhere.
- Client code (`CompositeDemo`) calls `getSize()` uniformly, with ZERO awareness of whether it's dealing with a File or an entire Folder subtree.
- Adding a new item type (e.g., `SymbolicLink implements FileSystemItem`) requires writing ONE new class — no existing code needs to change.

---

## 10. SOLID Principles Connection

- **OCP**: adding a new type of `FileSystemItem` (e.g., a new leaf or composite variant) requires only a new class — existing client code and existing Composite/Leaf classes remain unchanged.
- **LSP**: both `File` and `Folder` can be used ANYWHERE a `FileSystemItem` is expected, and behave correctly — this substitutability is exactly what enables uniform client-side treatment.
- **SRP**: `File` is responsible only for its OWN size/display; `Folder` is responsible only for MANAGING its children and aggregating their results — each class has a clear, focused responsibility.
- **ISP consideration**: if the common `FileSystemItem` interface is forced to include operations that ONLY make sense for one side (e.g., `add()`/`remove()` only make sense for Folders, not individual Files), this can create an awkward interface that leaf classes must implement as no-ops or exceptions — a genuine, common tension worth discussing explicitly in interviews (see Common Mistakes).

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Composite pattern solve, in your own words?
2. In the file system example, what's the KEY structural feature that allows arbitrarily deep nesting of folders within folders?
3. Why doesn't `Folder.getSize()` need to check whether each child is a File or another Folder?

**Intermediate:**
4. What are the tradeoffs of putting `add()`/`remove()` methods directly on the common `Component` interface (so BOTH Leaf and Composite have them) versus only on the Composite class? (Hint: relates to ISP.)
5. How would you handle an operation that ONLY makes sense for Composites (e.g., "count direct children") when called on a Leaf? What are your options (exception, no-op returning 0, etc.), and what are the tradeoffs of each?
6. How does the Composite pattern naturally support RECURSIVE operations without the client code needing to write any explicit recursion itself?

**Advanced:**
7. Compare Composite with the Decorator pattern (Topic 7) — both involve a tree-like/recursive structure of objects implementing a common interface. What's the KEY difference?
   *Answer: Decorator wraps a SINGLE object with ADDITIONAL behavior in a linear chain (each decorator has exactly ONE wrapped component) — its structure is fundamentally a LIST/chain, even if that chain can be arbitrarily long. Composite represents a TRUE TREE where a single node (Composite) can have MULTIPLE children, each of which could independently be a Leaf or another Composite — its structure is fundamentally BRANCHING, not linear. Both use "recursive-looking" delegation, but Decorator's shape is a chain while Composite's shape is a tree.*
8. How would you implement a "find all items matching X" search operation across a Composite tree? Would this be more naturally implemented USING the Visitor pattern (Topic 28, upcoming) instead of adding a `search()` method directly to the Component interface?
9. Discuss the "transparency vs safety" design tradeoff in Composite: putting child-management methods (`add`/`remove`) on the Component interface itself makes the design more "transparent" (uniform) but less "safe" (Leaves must handle calls that don't make sense for them). What would you choose, and why?
10. How does a `JPanel` (Swing) or similar GUI container hierarchy embody the Composite pattern? What operations get delegated recursively across the component tree?

---

## 12. Common Mistakes

- **Forcing Leaf classes to implement Composite-only operations poorly** — e.g., a `File.add()` method that either throws an `UnsupportedOperationException` or silently does nothing; this is a well-known, genuine design tension in Composite (discussed further in Interview Question 9) — being AWARE of it and articulating the tradeoff is more valuable than pretending it doesn't exist.
- **Not handling CYCLES in the tree** — if a Folder somehow ends up containing itself (directly or indirectly, through a chain of additions), a RECURSIVE operation like `getSize()` will infinite-loop/stack-overflow; real systems typically need to guard against this during `add()`.
- **Recomputing expensive operations repeatedly** — `Folder.getSize()` as shown recalculates the ENTIRE subtree's size on EVERY call; for large trees called frequently, this can be a real performance concern, and CACHING (with proper invalidation when children change) may be warranted.
- **Confusing Composite with a plain nested data structure (like a JSON tree)** — the DEFINING feature of Composite specifically is the UNIFORM INTERFACE shared by Leaf and Composite, enabling client code to treat both identically; a nested data structure without this shared interface/uniform treatment isn't really Composite.

---

## 13. Time Complexity

- **Time:** O(n) for operations like `getSize()` that must visit EVERY node in the tree, where n = total number of nodes (Files + Folders) in the ENTIRE subtree.
- **Space:** O(n) for the tree structure itself (n total nodes, each holding references to its children); O(d) additional call-stack space during recursive traversal, where d = the tree's MAXIMUM DEPTH.

---

## 14. Java API Examples

- **Swing's `Container`/`Component` hierarchy**: `JPanel` (a `Container`, effectively a Composite) can hold other `Component`s, including other `JPanel`s — rendering and layout operations recurse naturally through this tree.
- **`org.w3c.dom.Node`** (used in Java's built-in XML DOM APIs): both element nodes (which can have CHILD nodes) and leaf nodes (like text nodes) implement the common `Node` interface, allowing uniform tree traversal.
- **`java.awt.Component`/`java.awt.Container`**: the foundational AWT hierarchy mirrors the same Composite structure as Swing's, since Swing is built on top of AWT.
- **File I/O — `java.nio.file.Files.walk()`**: while not a strict textbook Composite implementation, the conceptual TREE traversal of directories and files mirrors the same recursive, uniform-interface idea.

---

## 15. Practice Problem

Implement a **Composite pattern for an Organization Chart**: create an `Employee` interface with a `getTotalSalaryBudget()` method. Implement `IndividualContributor` (a Leaf, with a fixed salary) and `Manager` (a Composite, which has a salary of its own PLUS manages a list of direct reports, who may themselves be `IndividualContributor`s or other `Manager`s). Demonstrate computing the total salary budget for an entire multi-level reporting structure with a single method call at the top.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **UI rendering system for a form builder** using Composite. A `FormElement` interface should be implemented by both individual input fields (`TextField`, `Checkbox` — Leaves) and grouping containers (`FieldGroup`, `Section` — Composites, which can contain OTHER FormElements, including nested groups). Implement a `render()` operation that produces an indented text representation of the ENTIRE form structure, and a `validate()` operation that returns whether ALL fields in the ENTIRE structure (recursively) are currently valid."

Think about:
- Whether `validate()` for a Composite should return `true` only if ALL children are valid, or use some other aggregation logic — and how this generalizes the "sum" logic used for `getSize()` in the file system example to a DIFFERENT kind of aggregation (boolean AND, instead of numeric sum).
- How you'd handle a `FieldGroup` that itself has SOME validation logic (e.g., "at least one field in this group must be filled") IN ADDITION to delegating to its children's own validation — this reveals that Composite operations aren't always PURE delegation; sometimes the Composite itself contributes some of its own logic too.

---

## 17. Advanced LLD Scenario

**Design a Restaurant Menu System** using Composite, where:
- A `MenuComponent` interface is implemented by individual `MenuItem`s (Leaves — e.g., "Margherita Pizza," $12) and by `Menu`/`SubMenu` groupings (Composites — e.g., "Appetizers," "Main Courses," which can contain items OR further nested sub-menus like "Vegetarian Mains" within "Main Courses")
- The system needs to support operations like: `getTotalPrice()` (sum of all items, useful for a "combo meal" feature bundling several items/sub-menus together), `print()` (rendering the entire menu with proper nested indentation), and `applyDiscount(percentage)` (recursively applying a discount to EVERY item in a subtree — e.g., "20% off all Appetizers")
- Consider how a "Combo Meal" feature (bundling e.g., one Appetizer + one Main Course + one Drink, at a special bundled price) might ITSELF be modeled as a Composite, blurring the line between "a special composite grouping" and "a specially-priced leaf-like offering"

Consider:
- How `applyDiscount()` demonstrates a MUTATING recursive operation (unlike the read-only `getSize()`/`getTotalPrice()` examples) — walking the ENTIRE subtree and modifying EVERY leaf's price, which requires careful thought about whether discounts should STACK if applied multiple times, or whether Composite needs to track "original price" separately from "current discounted price"
- Why treating a "Combo Meal" as a Composite (bundling multiple existing MenuItems/Menus at a special price) is a MUCH cleaner design than writing bespoke, hardcoded combo-pricing logic elsewhere in the system — this scenario is a strong example of Composite's real power: RE-USING existing structure for a new, composed feature
- How this scenario, the organization chart, and the file system all share the EXACT same underlying structural shape (tree, uniform interface, recursive aggregation) despite being from completely different business domains — recognizing this recurring shape quickly is a hallmark of strong LLD interview performance

---

## 18. Summary

**Definition:** Composite lets you compose objects into tree structures representing part-whole hierarchies, treating individual objects (Leaves) and groups of objects (Composites) uniformly through a shared interface.

**Intent:** Allow client code to work with individual objects and compositions of objects INTERCHANGEABLY, without needing to distinguish between them, and to support recursive operations across arbitrarily deep hierarchies naturally.

**Key classes:** `Component` (common interface), `Leaf` (individual object, no children), `Composite` (container holding child `Component`s, delegates and aggregates operations recursively).

**Advantages:** Dramatically simplifies client code; easy to add new leaf/composite types (good OCP); naturally supports recursive tree operations.

**Disadvantages:** Can force an overly general interface (ISP tension between Leaf-only and Composite-only operations); risk of tree cycles if not guarded against; potential performance cost for repeated recursive recalculation.

**Real-world use case:** File systems, GUI component trees, organization charts, XML/HTML DOM trees, restaurant menu systems.

**Java example:** `Folder` recursively summing `getSize()` across a mix of `File` and nested `Folder` children, all implementing `FileSystemItem`.

**Interview takeaway:** Be ready to clearly distinguish Composite's TREE/branching structure from Decorator's LINEAR CHAIN structure (Topic 7) — both use recursive-looking delegation through a shared interface, but their underlying SHAPE is fundamentally different.

**One-line memory trick:** *"Right-click 'Properties' on a single file or an entire folder — same action, same interface, whether it's one item or a thousand nested inside."*

---

*End of Topic 14. Type "Next" to proceed to Topic 15: Adapter Pattern.*