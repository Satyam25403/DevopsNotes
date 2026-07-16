# Topic 24: Iterator Design Pattern

---

## 1. Introduction

**Definition:**
The Iterator Pattern is a **behavioral design pattern** that provides a way to access the elements of a collection sequentially, without exposing the collection's underlying internal representation (array, linked list, tree, hash table, etc.) to the client traversing it.

**Why it exists / what problem it solves:**
Without Iterator, code that needs to traverse a collection must know the collection's INTERNAL STRUCTURE to do so — e.g., using an index (`for (int i = 0; i < arr.length; i++)`) for an array, or following `next` pointers manually for a linked list. This creates problems:
- Client code becomes tightly coupled to the SPECIFIC data structure being used — if the underlying structure later changes (say, from an array-backed list to a linked list), all traversal code that assumed array-style indexing breaks.
- Different collection types (trees, graphs, hash maps) each need COMPLETELY different traversal logic, and there's no uniform way to write code that works across all of them.
- Exposing internal structure (e.g., a public array field) for traversal purposes violates encapsulation (Topic 2).

Iterator solves this by providing a UNIFORM interface (`hasNext()`, `next()`) for traversal, regardless of what's actually happening internally — a client can traverse an `ArrayList`, a `LinkedList`, or a custom tree structure using the EXACT SAME code pattern.

**When it should be used:**
- When you need to traverse a collection without exposing its internal structure
- When you want to support MULTIPLE simultaneous traversals of the same collection (each iterator maintains its own position independently)
- When you want traversal code to work uniformly across different collection types

**When it should NOT be used:**
- When the collection is trivially simple and traversal needs are basic enough that a simple `for` loop over a known array is clearer (though in Java, using the built-in `Iterable`/enhanced-for-loop essentially gives you this pattern "for free" with negligible added complexity, so this caveat matters less in Java specifically)
- When you need RANDOM access (jumping to arbitrary positions) rather than sequential traversal — Iterator is fundamentally sequential

**Advantages:**
- Decouples traversal logic from the collection's internal structure
- Supports multiple independent traversals of the same collection simultaneously
- Provides a UNIFORM traversal interface across different collection types
- Encapsulation is preserved — the collection's internals stay hidden

**Disadvantages:**
- Adds a layer of indirection compared to direct index-based access
- For simple array-backed collections, may feel like unnecessary overhead compared to a plain indexed loop

---

## 2. Real-World Analogy

Think of a **TV remote control's channel-changing button**.

You don't need to know HOW the TV internally organizes its channel list (an array? a linked list? a database?) — you just press "next channel," and the TV shows you the next one. You press it again, and you get the one after that. The remote gives you a UNIFORM way to move through channels ONE AT A TIME, without you ever needing to know or care about the TV's internal channel-storage mechanism.

If the TV manufacturer completely REWROTE how channels are stored internally in a software update, your remote's "next channel" button would still work EXACTLY the same way from your perspective — the internal representation changed, but your traversal experience (the Iterator) stayed identical.

---

## 3. Theory

**Core idea:** A collection (the "Aggregate") provides a method to obtain an `Iterator` object. The `Iterator` exposes `hasNext()` (is there more to see?) and `next()` (give me the next element, and advance) — client code uses ONLY this interface, never reaching into the collection's internals directly.

**Working mechanism:**
```
┌─────────────────────┐
│      <<interface>>          │
│      Iterable<T>                 │
│  + iterator(): Iterator<T>         │
└──────────┬──────────┘
           △ (implements)
┌──────────┴──────────┐
│      BookShelf                   │
│  - books: List<Book>                │  ← internal representation
│                                        HIDDEN from client
│  + iterator(): Iterator<Book>            │
└──────────────────────┘
           returns
           ↓
┌─────────────────────┐
│      <<interface>>          │
│      Iterator<T>                 │
│  + hasNext(): boolean               │
│  + next(): T                          │
└──────────┬──────────┘
           △ (implements)
┌──────────┴──────────┐
│      BookShelfIterator           │
│  - currentPosition: int              │  ← tracks traversal state,
└──────────────────────┘     SEPARATE from the collection itself
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Aggregate / Iterable | The collection that can be iterated over |
| Iterator | The object that performs and tracks the traversal |
| `hasNext()` | Checks if there are more elements remaining |
| `next()` | Returns the current element and advances the position |

**Key structural insight:** the ITERATOR (not the collection itself) holds the traversal POSITION/STATE. This is what allows MULTIPLE iterators over the SAME collection to exist simultaneously, each at a different position, without interfering with each other.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      Iterable<T>                 │
├─────────────────────┤
│ + iterator(): Iterator<T>            │
└──────────┬──────────┘
           △ (implements)
┌──────────┴──────────┐            ┌─────────────────────┐
│      BookShelf                   │            │      <<interface>>          │
├─────────────────────┤            │      Iterator<T>                 │
│ - books: List<Book>       ◆─────┐    ├─────────────────────┤
│                                     │    │ + hasNext(): boolean               │
│ + addBook(Book b)                     │    │ + next(): T                          │
│ + iterator(): Iterator<Book>            │    └──────────┬──────────┘
└─────────────────────┘            │               △ (implements)
                                       │    ┌──────────┴──────────┐
                                       │    │      BookShelfIterator           │
                                       │    ├─────────────────────┤
                                       │    │ - shelf: BookShelf      ◇─────────┘
                                       │    │ - position: int                          │
                                       │    └─────────────────────┘
                                       │
                                       (composition: BookShelf
                                        owns its internal books list)
```

**Relationship explanation:**
- `BookShelf` **implements** `Iterable<T>` — signaling "I can be iterated over."
- `BookShelfIterator` **implements** `Iterator<T>` — providing the actual traversal mechanics.
- `BookShelfIterator` has an **aggregation** relationship with `BookShelf` (it needs a reference back to the shelf's data to traverse it), while `BookShelf` has a **composition** relationship with its internal `books` list (the list is integral to the shelf's own identity).
- Critically: `BookShelfIterator` holds its OWN `position` field — SEPARATE from `BookShelf` itself — meaning multiple iterators over the same shelf can each track their own independent progress.

---

## 5. Java Implementation

```java
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;
import java.util.NoSuchElementException;

// ============================================
// The element type being stored/iterated
// ============================================
public class Book {
    private final String title;

    public Book(String title) {
        this.title = title;
    }

    public String getTitle() {
        return title;
    }
}

// ============================================
// The Aggregate — implements Java's built-in
// Iterable interface, exposing iterator() but
// hiding its internal storage mechanism
// ============================================
public class BookShelf implements Iterable<Book> {
    private final List<Book> books = new ArrayList<>(); // internal representation, HIDDEN

    public void addBook(Book book) {
        books.add(book);
    }

    public int size() {
        return books.size();
    }

    // Returns a NEW iterator each time — each call gets its
    // OWN independent traversal position
    @Override
    public Iterator<Book> iterator() {
        return new BookShelfIterator(this);
    }

    // Package-private accessor used ONLY by the iterator —
    // still keeps the internal list itself fully encapsulated
    // from any OTHER external code
    Book getBookAt(int index) {
        return books.get(index);
    }
}

// ============================================
// The Concrete Iterator — holds its OWN traversal
// state (position), separate from the BookShelf itself
// ============================================
public class BookShelfIterator implements Iterator<Book> {
    private final BookShelf shelf;
    private int position = 0; // THIS iterator's own progress marker

    public BookShelfIterator(BookShelf shelf) {
        this.shelf = shelf;
    }

    @Override
    public boolean hasNext() {
        return position < shelf.size();
    }

    @Override
    public Book next() {
        if (!hasNext()) {
            throw new NoSuchElementException("No more books on the shelf");
        }
        Book book = shelf.getBookAt(position);
        position++; // advance THIS iterator's position only
        return book;
    }
}

// ============================================
// Demo
// ============================================
public class IteratorDemo {
    public static void main(String[] args) {
        BookShelf shelf = new BookShelf();
        shelf.addBook(new Book("Effective Java"));
        shelf.addBook(new Book("Clean Code"));
        shelf.addBook(new Book("Design Patterns"));

        // Manual iterator usage
        Iterator<Book> it1 = shelf.iterator();
        while (it1.hasNext()) {
            System.out.println("it1: " + it1.next().getTitle());
        }

        // Because BookShelf implements Iterable, Java's
        // enhanced for-loop works automatically too —
        // this is SYNTACTIC SUGAR for calling iterator()
        // and looping with hasNext()/next() internally
        for (Book book : shelf) {
            System.out.println("for-each: " + book.getTitle());
        }

        // TWO INDEPENDENT iterators over the SAME shelf,
        // each tracking its OWN position
        Iterator<Book> itA = shelf.iterator();
        Iterator<Book> itB = shelf.iterator();
        System.out.println("itA first: " + itA.next().getTitle());
        System.out.println("itA second: " + itA.next().getTitle());
        System.out.println("itB first: " + itB.next().getTitle()); // itB is UNAFFECTED by itA's progress
    }
}
```

**Key line-by-line notes:**
- `BookShelf implements Iterable<Book>` — this is Java's STANDARD mechanism for making a class support the enhanced for-loop (`for (Book b : shelf)`), which is really just the Iterator pattern built directly into the language.
- `iterator()` returns a NEW `BookShelfIterator` EVERY TIME it's called — this is essential: if it returned the SAME shared iterator instance, two independent traversals would interfere with each other's position.
- `BookShelfIterator`'s `position` field is entirely SEPARATE from `BookShelf`'s own internal `books` list — this separation of "the data" from "the traversal position" is the structural heart of the pattern.
- `getBookAt(int index)` is package-private, not `public` — it's accessible to `BookShelfIterator` (same package) for traversal purposes, but still not exposed to ARBITRARY external code, preserving encapsulation as much as reasonably possible.

---

## 6. Dry Run

**Sample input:** The "TWO INDEPENDENT iterators" section — `itA.next()` called twice, then `itB.next()` called once.

```
1. shelf has books = [EffectiveJava, CleanCode, DesignPatterns]
   (indices 0, 1, 2)

2. Iterator<Book> itA = shelf.iterator()
   → new BookShelfIterator(shelf) created
   → itA.position = 0

3. Iterator<Book> itB = shelf.iterator()
   → ANOTHER new BookShelfIterator(shelf) created —
     a SEPARATE object from itA
   → itB.position = 0 (independently initialized)

4. itA.next() called
   → hasNext() checks: itA.position (0) < shelf.size() (3) → true
   → book = shelf.getBookAt(0) → "Effective Java"
   → itA.position becomes 1
   → returns "Effective Java"

5. itA.next() called AGAIN
   → hasNext(): itA.position (1) < 3 → true
   → book = shelf.getBookAt(1) → "Clean Code"
   → itA.position becomes 2
   → returns "Clean Code"

6. itB.next() called
   → hasNext(): itB.position (0) < 3 → true
     (itB.position is STILL 0 — completely unaffected
      by everything that happened to itA)
   → book = shelf.getBookAt(0) → "Effective Java"
   → itB.position becomes 1
   → returns "Effective Java"
```

**What's happening in memory:** `itA` and `itB` are TWO SEPARATE `BookShelfIterator` OBJECTS, each with its OWN `position` field, though both hold a reference to the SAME underlying `shelf` object. Advancing `itA.position` has ZERO effect on `itB.position`, because they're different objects entirely — this is precisely why Iterator supports multiple simultaneous, independent traversals of one collection.

---

## 7. Real-World Software Example

- **Java Collections Framework**: EVERY collection (`ArrayList`, `HashSet`, `LinkedList`, `TreeMap`) implements `Iterable`, providing a consistent `iterator()` method — this is THE most direct, ubiquitous real-world example of this pattern in daily Java development.
- **Database result set traversal**: `java.sql.ResultSet` provides `next()` for sequentially traversing query results, without the client needing to know how the database internally stores or fetches rows.
- **File system traversal**: iterating over files in a directory (`java.nio.file.DirectoryStream`) without needing to know the underlying file system's internal indexing structure.
- **Custom tree/graph traversal**: a binary tree class might provide DIFFERENT iterators (`inOrderIterator()`, `preOrderIterator()`, `postOrderIterator()`) — all exposing the SAME `hasNext()`/`next()` interface, while internally implementing very different traversal algorithms.

---

## 8. Internal Working

**Enhanced for-loop desugaring**: Java's `for (Book book : shelf) { ... }` is PURELY syntactic sugar — the compiler REWRITES this at compile time into:
```java
Iterator<Book> it = shelf.iterator();
while (it.hasNext()) {
    Book book = it.next();
    // loop body
}
```
This is a critical internal-working fact: there is NO special runtime mechanism for the enhanced for-loop beyond the compiler generating this exact `Iterator`-based code automatically. Understanding this is exactly why a class MUST implement `Iterable` to be used in a for-each loop — the compiler literally needs to call `.iterator()` on it.

**Fail-fast iterators (a real Java-specific internal-working detail)**: many JDK collection iterators (like `ArrayList`'s) are "fail-fast" — they track a `modCount` (modification count) on the underlying collection, and if the collection is structurally modified WHILE an iterator is active (e.g., calling `list.add()` during iteration, other than via the iterator's OWN `remove()` method), the iterator throws a `ConcurrentModificationException` on the NEXT `next()` call, rather than silently producing incorrect/undefined behavior.

**Memory:** each iterator object is small and short-lived — typically just holding a reference back to the collection and a position marker (or, for linked structures, a reference to the "current node").

---

## 9. Before vs After

**Before (exposing internal structure directly):**

```java
public class BookShelfBad {
    public List<Book> books = new ArrayList<>(); // PUBLIC — internal structure exposed!
}

// Client code:
BookShelfBad shelf = new BookShelfBad();
for (int i = 0; i < shelf.books.size(); i++) {  // client must know it's a List,
    System.out.println(shelf.books.get(i));       // and use index-based access directly
}
```

**Problems:**
- `books` is `public` — any external code can directly modify the internal list, bypassing any validation `BookShelfBad` might otherwise want to enforce (breaks encapsulation, Topic 2).
- If `BookShelfBad` later changes its internal storage from a `List` to, say, a `Set` (no indexing) or a custom linked structure, EVERY piece of client code using `shelf.books.get(i)` breaks immediately.
- No way to have two INDEPENDENT traversals with separate positions without manually tracking separate index variables externally.

**After (Iterator pattern, as shown in Section 5):**
- `books` stays entirely private; client code interacts ONLY through `iterator()`, `hasNext()`, `next()`.
- `BookShelf` could switch its internal storage to ANY other structure, and as long as `iterator()` is updated accordingly, ALL client code using the standard Iterator interface continues working UNCHANGED.
- Multiple independent iterators are trivially supported, since each is its OWN object with its OWN position.

---

## 10. SOLID Principles Connection

- **SRP**: `BookShelf` is responsible for STORING books; `BookShelfIterator` is responsible for TRAVERSING them — two distinct responsibilities, cleanly separated into two classes.
- **OCP**: adding a NEW traversal order (e.g., a `ReverseBookShelfIterator`) requires creating a new class implementing `Iterator<Book>`, without modifying `BookShelf` or the existing `BookShelfIterator`.
- **DIP**: client code depends on the `Iterator<Book>` and `Iterable<Book>` ABSTRACTIONS, never on the concrete `BookShelfIterator` class directly (the enhanced for-loop, for instance, only ever calls interface methods).
- **Encapsulation** (Topic 2, not formally part of SOLID but closely related): the collection's internal storage mechanism remains fully hidden from any code performing traversal.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Iterator pattern solve?
2. What are the two core methods every Iterator typically provides?
3. How does Java's enhanced for-loop relate to the Iterator pattern?

**Intermediate:**
4. Why does `iterator()` need to return a NEW iterator object on each call, rather than a shared one?
5. What is a "fail-fast" iterator, and why does the JDK implement its iterators this way?
6. Can two iterators over the SAME collection operate independently? Explain why or why not, referencing where the traversal "position" is actually stored.

**Advanced:**
7. How would you implement MULTIPLE different traversal orders (e.g., in-order vs pre-order for a tree) using the Iterator pattern, without duplicating the tree's core storage logic?
8. What happens if you modify a `List` directly (not via the iterator's own `remove()`) while iterating over it with a `for-each` loop? Explain the underlying mechanism that causes this.
   *Answer: A `ConcurrentModificationException` is thrown — the iterator detects the underlying collection's `modCount` changed unexpectedly between `next()` calls, indicating an unsynchronized structural modification occurred outside the iterator's own control.*
9. How does the Iterator pattern support the Open/Closed Principle when adding new collection types to a codebase?
10. Compare Iterator to simply returning a `List` copy of the internal data for the client to loop over directly. What are the tradeoffs?
    *Answer: Returning a copy avoids exposing the LIVE internal structure (arguably safer in some respects) but has a real performance/memory cost for large collections (copying everything up front), and doesn't naturally support lazy/on-demand traversal (e.g., an infinite or very large generated sequence) the way a true Iterator can, which can compute/fetch the next element only when actually requested.*
11. How would you design an Iterator for a data source that doesn't fit neatly in memory (e.g., streaming rows from a very large database table)?

---

## 12. Common Mistakes

- **Reusing a single Iterator instance across multiple independent traversal needs** — if `iterator()` is implemented to return the SAME cached iterator object every time instead of a fresh one, two separate traversal attempts will interfere with each other's position.
- **Exposing the internal collection directly "just to make iteration easier"** — defeats the entire purpose of the pattern; the whole point is enabling traversal WITHOUT exposing internals.
- **Modifying a collection while iterating over it without using the iterator's own removal mechanism** — leads to `ConcurrentModificationException` in Java's standard collections; if removal during iteration is genuinely needed, use `Iterator.remove()` specifically, which is designed to handle this safely.
- **Forgetting to handle the "no more elements" case properly** — `next()` should throw a clear exception (like `NoSuchElementException`) when called after `hasNext()` would have returned `false`, rather than failing with a confusing, unrelated error.

---

## 13. Time Complexity

**`hasNext()` and `next()`**: typically O(1) for array-backed or linked-list-backed collections — each call does a small, constant amount of work. **Full traversal**: O(n) where n is the number of elements — visiting each element exactly once. Note that for certain structures (e.g., a tree requiring traversal-order bookkeeping), individual `next()` calls might have slightly more overhead, but the FULL traversal remains O(n) overall in virtually all practical implementations.

---

## 14. Java API Examples

- **`java.util.Iterator`** and **`java.lang.Iterable`**: the pattern is BUILT DIRECTLY into the Java language itself — any class implementing `Iterable` automatically works with the enhanced for-loop.
- **`java.util.ListIterator`**: an ENHANCED iterator specifically for `List`s, additionally supporting backward traversal (`hasPrevious()`/`previous()`) and in-place modification (`set()`, `add()`) during iteration — demonstrating the pattern can be extended with richer capabilities while keeping the same core traversal contract.
- **`java.util.Scanner`**: conceptually similar — `hasNext()`/`next()` for sequentially reading tokens from an input source, hiding whether the source is a file, a string, or standard input.
- **Java Streams (`java.util.stream.Stream`)**: while not a DIRECT implementation of the classic `Iterator` interface, Streams represent an evolution of the same underlying idea — sequential, on-demand access to elements, with the added capability of composable intermediate operations (map, filter) before final consumption.

---

## 15. Practice Problem

Implement a custom `Playlist` class (holding `Song` objects) that implements `Iterable<Song>`, along with a custom `PlaylistIterator`. Then implement a SECOND iterator type — a `ShuffleIterator` — that traverses the SAME playlist's songs in a randomized order (without modifying the playlist's actual underlying storage order). Demonstrate that both iterator types work correctly and independently over the same `Playlist` instance.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design an iterator for a `SocialMediaFeed` that needs to merge posts from multiple sources (Friends, Pages, Groups) into a SINGLE chronological feed, without the client code needing to know that the underlying data actually comes from three separate internal lists."

Think about:
- How would a single `FeedIterator` internally coordinate advancing through THREE separate underlying sources while presenting ONE unified, chronologically-ordered `hasNext()`/`next()` interface to the client?
- What happens if one of the three underlying sources is exhausted before the others — how should the iterator handle this gracefully?

---

## 17. Advanced LLD Scenario

**Design a Music Streaming Service's "Browse History" feature** (like Spotify) where:
- Users can traverse their listening history sequentially, potentially spanning MILLIONS of entries over years of usage — loading the ENTIRE history into memory at once would be impractical
- The system should support traversing history in different orders (most recent first, oldest first, by most-played)
- Multiple parts of the UI might need to traverse the SAME user's history simultaneously (e.g., a "recently played" widget and a full history page open at once) without interfering with each other's position

Consider:
- How Iterator naturally supports LAZY, ON-DEMAND traversal — fetching the NEXT batch of history entries from a database or paginated API only when `next()` is actually called, rather than loading everything upfront
- How you'd design multiple iterator implementations (`RecentFirstIterator`, `MostPlayedIterator`) all conforming to the same `Iterator<HistoryEntry>` interface, letting client UI code remain agnostic to which specific ordering is currently active
- How this "lazy, on-demand, paginated" iterator concept connects to real production systems — this is essentially how many real-world API client libraries implement pagination internally (fetching the next page transparently when the current page's iterator is exhausted)

---

## 18. Summary

**Definition:** Iterator provides sequential access to a collection's elements without exposing its underlying internal representation.

**Intent:** Decouple traversal logic from collection implementation details; support multiple independent, simultaneous traversals.

**Key classes:** `Iterable` (the collection, exposing `iterator()`), `Iterator` (`hasNext()`/`next()`), Concrete implementations of each.

**Advantages:** Encapsulation-preserving traversal; uniform interface across different collection types; supports independent simultaneous traversals.

**Disadvantages:** Slight indirection overhead versus direct indexed access; fundamentally sequential, not suited for random access.

**Real-world use case:** The ENTIRE Java Collections Framework; database `ResultSet` traversal; paginated API client libraries.

**Java example:** `BookShelf` (Iterable) with `BookShelfIterator`, demonstrating multiple independent traversals of the same shelf.

**Interview takeaway:** Be ready to explain EXACTLY how Java's enhanced for-loop desugars into `iterator()`/`hasNext()`/`next()` calls at compile time — this concrete, mechanical understanding (rather than just "it's built in") is what distinguishes a strong answer.

**One-line memory trick:** *"A TV remote's 'next channel' button — you never need to know how channels are stored internally, you just get the next one."*

---

*End of Topic 24. Type "Next" to proceed to Topic 25: Mediator Pattern.*