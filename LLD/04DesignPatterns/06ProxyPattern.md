# Topic 11: Proxy Pattern

---

## 1. Introduction

**Definition:**
The Proxy Pattern is a **structural design pattern** that provides a SURROGATE or PLACEHOLDER object which controls access to another object (the "real subject") — the proxy implements the SAME interface as the real object, so clients interact with it transparently, while the proxy adds its own logic (access control, lazy loading, caching, logging, etc.) around the real object's operations.

**Why it exists / what problem it solves:**
Sometimes you need to add a layer of control BEFORE or AFTER accessing an object, without changing the object itself and without the client knowing the difference. For example: a heavyweight object (a large image, a remote database connection) might be expensive to create — you don't want to pay that cost until it's ACTUALLY needed. Or you might want to restrict WHO can access certain operations on an object, without cluttering the object's own code with access-control logic.

Proxy solves this by inserting an intermediary object — implementing the SAME interface as the real object — that the client talks to instead. The proxy decides when (or whether) to forward the call to the real object, and can layer in extra behavior around that forwarding.

**When it should be used:**
- When object creation is EXPENSIVE and you want to defer it until actually needed (**Virtual Proxy** / lazy loading)
- When you need to control ACCESS to an object based on permissions (**Protection Proxy**)
- When you want to add a LOCAL representative for an object that lives elsewhere (**Remote Proxy** — e.g., RMI, gRPC stubs)
- When you want to CACHE results of expensive operations transparently (**Caching Proxy**)
- When you need to LOG, monitor, or audit calls to an object without modifying the object itself

**When it should NOT be used:**
- When there's no real need for an extra layer of indirection — adding a Proxy "just in case" is unnecessary complexity
- When the added latency/complexity of an extra layer outweighs the benefit for a simple, cheap-to-create object
- When the added behavior is fundamentally different from "controlling access" — that's a better fit for Decorator (Topic 7) instead (see comparison in Section 11)

**Advantages:**
- Client code is completely unaware it's talking to a proxy instead of the real object — transparent substitution
- Enables lazy initialization, saving resources until genuinely needed
- Centralizes cross-cutting concerns (access control, logging, caching) without polluting the real object's code
- Follows OCP — new proxy behaviors can be added without modifying the real subject

**Disadvantages:**
- Adds an extra layer of indirection — every call goes through the proxy first, adding a small overhead
- Can make the system harder to understand/debug, since the "real" behavior is now split across two classes
- If overused, can lead to a maze of nested proxies that's hard to trace

---

## 2. Real-World Analogy

Think of a **credit card as a proxy for your bank account**.

You don't hand over stacks of cash from your actual bank vault every time you make a purchase — instead, you use a credit card, which acts as a PROXY: it looks like (and functions like) direct access to money, but it actually controls and mediates the real transaction happening behind the scenes with your bank account. The merchant doesn't care whether you paid with a proxy (card) or cash directly — the interface (payment) looks the same either way.

The credit card can ALSO add extra behavior: checking your available balance before approving the purchase (protection), logging the transaction (auditing), or even declining the transaction if the account is suspended (access control) — all WITHOUT the merchant or your bank account itself needing to know about these checks.

---

## 3. Theory

**Core idea:** Both the Proxy and the Real Subject implement the SAME interface. The Client holds a reference to this interface — it might be handed a Proxy OR the Real Subject, and cannot tell the difference. The Proxy internally holds a reference to the Real Subject and decides when/whether to delegate calls to it.

**Working mechanism:**
```
Client → Subject (interface) ← RealSubject
                    ↑
                  Proxy ──(delegates to)──→ RealSubject
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Subject | The common interface implemented by both Proxy and RealSubject |
| RealSubject | The actual object that does the real work — often expensive to create or access |
| Proxy | The surrogate implementing the same interface, controlling access to RealSubject |
| Virtual Proxy | Defers creation of an expensive RealSubject until first use |
| Protection Proxy | Adds access-control checks before delegating |
| Remote Proxy | Represents an object that lives in a different address space/process/machine |
| Caching Proxy | Stores results of expensive RealSubject calls to avoid repeating them |

**Class responsibilities:** the Proxy is responsible for managing the LIFECYCLE and/or ACCESS to the RealSubject — the RealSubject itself remains completely unaware that a Proxy exists; it just does its normal work when called.

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│         Image                    │
├─────────────────────┤
│ + display(): void            │
└──────────┬──────────┘
           △ (implements)
   ┌───────┴───────┐
   │                       │
┌──┴────────────┐   ┌──┴────────────┐
│ RealImage             │   │ ProxyImage            │
├───────────────┤   ├───────────────┤
│ - filename: String       │   │ - filename: String       │
│ + RealImage(String)     │   │ - realImage: RealImage  ◇──→ (association:
│ + display()               │   │ + ProxyImage(String)       Proxy holds a
│   (loads from disk           │   │ + display()                reference to
│    in constructor)             │   │   (creates RealImage         RealSubject,
└───────────────┘   │    LAZILY, on first          created lazily)
                       │    display() call)
                       └───────────────┘

┌─────────────────────┐
│      Client                      │
├─────────────────────┤
│ + main()                     ────────→ (dependency: Client only
└─────────────────────┘             knows the Image interface —
                                        could be handed a RealImage
                                        OR a ProxyImage transparently)
```

**Relationship explanation:**
- `RealImage` and `ProxyImage` both **implement** the `Image` interface — this is what allows the Client to treat them interchangeably.
- `ProxyImage` has an **association** with `RealImage` — it holds a reference to it, but crucially does NOT create it until it's actually needed (lazy initialization), unlike a normal composition where the object would be created immediately in the constructor.
- `Client` depends only on the `Image` interface — it never directly references `RealImage` or `ProxyImage` by concrete type, so swapping one for the other requires zero client-code changes.

---

## 5. Java Implementation

```java
// ============================================
// Subject interface — common to Proxy and RealSubject
// ============================================
public interface Image {
    void display();
}

// ============================================
// RealSubject — the actual, expensive-to-create object
// ============================================
public class RealImage implements Image {
    private final String filename;

    // Simulates an EXPENSIVE operation (loading a large image from disk)
    // happening immediately at construction time
    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk();
    }

    private void loadFromDisk() {
        System.out.println("Loading image from disk: " + filename + " (expensive operation)");
    }

    @Override
    public void display() {
        System.out.println("Displaying image: " + filename);
    }
}

// ============================================
// Proxy — implements the SAME interface, controls
// access to (and lazily creates) the RealSubject
// ============================================
public class ProxyImage implements Image {
    private final String filename;
    // Not created immediately — stays null until first actually needed
    private RealImage realImage;

    public ProxyImage(String filename) {
        // Cheap operation — just storing the filename, NOT loading anything yet
        this.filename = filename;
    }

    @Override
    public void display() {
        // Lazy initialization: the expensive RealImage is created
        // ONLY the first time display() is actually called
        if (realImage == null) {
            System.out.println("(Proxy: real image not yet loaded, loading now...)");
            realImage = new RealImage(filename);
        }
        // Delegate the actual work to the RealSubject
        realImage.display();
    }
}

// ============================================
// Demo
// ============================================
public class ProxyDemo {
    public static void main(String[] args) {
        // Client creates a Proxy, NOT a RealImage directly —
        // no expensive loading happens here
        Image image = new ProxyImage("vacation_photo.jpg");

        System.out.println("Proxy created — no loading has happened yet.");

        // The expensive loading ONLY happens on this first call
        image.display();

        System.out.println("Calling display() again — no reload needed:");
        // Second call — realImage is already cached inside the proxy,
        // so no reloading occurs, just direct delegation
        image.display();
    }
}
```

**Key line-by-line notes:**
- `ProxyImage`'s constructor does almost nothing expensive — it just stores `filename`; contrast this with `RealImage`'s constructor, which immediately performs the expensive `loadFromDisk()`.
- The `if (realImage == null)` check inside `display()` is the crux of the Virtual Proxy pattern — it ensures the expensive object is created EXACTLY once, on first actual need, and reused afterward.
- After the RealImage is created, ALL the actual work (`realImage.display()`) is DELEGATED to it — the Proxy itself doesn't reimplement any of RealImage's core logic.

---

## 6. Dry Run

**Sample input:**
```java
Image image = new ProxyImage("vacation_photo.jpg");
image.display();
image.display();
```

```
1. new ProxyImage("vacation_photo.jpg")
   → Constructor runs: filename = "vacation_photo.jpg"
   → realImage remains null
   → NO expensive loading happens here — cheap operation only

2. image.display() [first call]
   → ProxyImage.display() executes
   → realImage == null? YES
   → Prints: "(Proxy: real image not yet loaded, loading now...)"
   → realImage = new RealImage("vacation_photo.jpg")
       → RealImage constructor runs
       → loadFromDisk() called
       → Prints: "Loading image from disk: vacation_photo.jpg (expensive operation)"
   → realImage.display() called
       → Dynamic dispatch resolves to RealImage's display()
       → Prints: "Displaying image: vacation_photo.jpg"

3. image.display() [second call]
   → ProxyImage.display() executes
   → realImage == null? NO (already cached from step 2)
   → Skips the expensive loading entirely
   → realImage.display() called directly
       → Prints: "Displaying image: vacation_photo.jpg"
```

**What's happening in memory:** the `RealImage` object is created EXACTLY once, on the heap, the first time it's genuinely needed — the `ProxyImage` object holds onto this reference for the rest of its lifetime, avoiding any repeated expensive `loadFromDisk()` calls on subsequent `display()` invocations.

---

## 7. Real-World Software Example

- **Hibernate/JPA lazy loading**: entity associations (e.g., `@OneToMany` collections) are often backed by proxy objects that only fetch the actual data from the database when the association is first accessed — a textbook Virtual Proxy.
- **Spring AOP proxies**: Spring wraps beans in dynamic proxies to add cross-cutting behavior (transactions, security checks, logging) transparently, without modifying the bean's own code.
- **Java RMI (Remote Method Invocation)**: a client-side "stub" acts as a Remote Proxy, making a remote method call LOOK like a normal local method call, while actually handling network communication behind the scenes.
- **`java.lang.reflect.Proxy`**: Java's built-in dynamic proxy mechanism, used extensively by frameworks (Spring, Hibernate, mocking libraries like Mockito) to generate proxy classes at runtime.
- **Security/protection proxies**: an API gateway or access-control layer that checks permissions BEFORE forwarding a request to the actual backend service.

---

## 8. Internal Working

**Object creation:** the Proxy object is created immediately and cheaply; the RealSubject's creation is DEFERRED (in the Virtual Proxy variant) until the first delegated call actually requires it — this is the key mechanism behind the pattern's resource-saving benefit.

**Runtime interactions:** every call on the Proxy either (a) short-circuits with its own logic (e.g., a Protection Proxy denying access without ever touching the RealSubject), or (b) delegates to the RealSubject after performing its own pre/post logic (e.g., lazy-loading it first, or logging before/after the call).

**Memory usage:** the Proxy itself is lightweight (holds a `filename` and a nullable reference); the RealSubject's memory footprint is only incurred once it's actually instantiated — meaning a Proxy for a never-used object never pays that memory cost at all.

**Dynamic binding:** since both `Proxy` and `RealSubject` implement the same interface, the Client's calls are always resolved via dynamic dispatch to whichever concrete object it's actually holding — the Client's code never needs to change based on which one it has.

---

## 9. Before vs After

**Before (no Proxy — eager, unconditional loading):**

```java
public class ImageGalleryBad {
    private List<RealImage> images = new ArrayList<>();

    public ImageGalleryBad(List<String> filenames) {
        // ALL images are loaded from disk IMMEDIATELY, even if
        // the user never actually looks at most of them —
        // wasteful if the gallery has hundreds of images
        for (String filename : filenames) {
            images.add(new RealImage(filename));
        }
    }
}
```

**Problems:**
- Every single image is loaded EAGERLY at gallery construction time, regardless of whether the user will ever view it — wasteful for large galleries.
- No way to add access control, caching, or logging without directly modifying `RealImage`'s own code.
- Startup time is directly proportional to the TOTAL number of images, even if only a few are ever displayed.

**After (Proxy, as shown in Section 5):**
- Each image is represented by a lightweight `ProxyImage` at gallery construction time — the actual expensive loading only happens the FIRST time a specific image is displayed.
- Additional behaviors (caching, access checks, logging) can be added directly inside the Proxy's `display()` method without touching `RealImage` at all.

---

## 10. SOLID Principles Connection

- **SRP**: `RealImage` is responsible ONLY for loading and displaying image data; `ProxyImage` is responsible ONLY for controlling access to (and the lifecycle of) the RealImage — these are two clearly separated responsibilities.
- **OCP**: new proxy BEHAVIORS (caching, logging, access control) can be added by creating new Proxy variants or extending the existing one, without ever modifying `RealImage`'s code.
- **LSP**: `ProxyImage` can be substituted anywhere an `Image` is expected, and client code behaves correctly regardless — this substitutability is the entire foundation of the pattern's transparency.
- **DIP**: `Client` depends only on the `Image` abstraction, never on `RealImage` or `ProxyImage` concretely — enabling the proxy to be swapped in/out freely.

---

## 11. Interview Questions

**Beginner:**
1. What is the main purpose of the Proxy pattern?
2. What are the common TYPES of proxies (virtual, protection, remote, caching)? Give a one-line description of each.
3. Why does the Proxy need to implement the SAME interface as the RealSubject?

**Intermediate:**
4. How does a Virtual Proxy save resources compared to eagerly creating the RealSubject?
5. Give a concrete example of a Protection Proxy — how would you implement access-control checks inside a proxy?
6. What's the difference between a Proxy and a simple wrapper/utility method that just calls another method?
   *Answer: A Proxy implements the SAME interface as the real object and is designed to be TRANSPARENTLY substitutable wherever the real object is expected — client code doesn't know or care it's a proxy. A plain wrapper method is just a helper function; it doesn't provide interface-level substitutability or polymorphic transparency.*

**Advanced:**
7. **Proxy vs Decorator** — precisely distinguish them (a very common interview follow-up after covering both, see Topic 7).
   *Answer: Structurally, Proxy and Decorator can look nearly IDENTICAL in code (both implement the same interface as the object they wrap, and delegate calls to it). The difference is INTENT: Decorator is about ADDING new responsibilities/behavior to an object, often STACKING multiple decorators for cumulative effect. Proxy is about CONTROLLING ACCESS to an object (deferring creation, restricting access, adding remote-call semantics) — it typically doesn't "add features" in the same layered sense; it manages HOW/WHEN/WHETHER the real object is reached.*
8. How does Spring's AOP proxy mechanism relate to this pattern, and what's the difference between JDK dynamic proxies and CGLIB proxies?
9. What happens if a Proxy needs to support MULTIPLE cross-cutting concerns simultaneously (e.g., both caching AND access control)? How would you structure this cleanly?
   *Answer: You could chain MULTIPLE proxies together (a CachingProxy wrapping a ProtectionProxy wrapping the RealSubject, each implementing the same interface) — this shows Proxy and Chain of Responsibility/Decorator-style layering can compose naturally when each proxy has a single, focused concern.*
10. How does Java's `java.lang.reflect.Proxy` create proxy classes at RUNTIME rather than compile time, and what's the tradeoff versus writing a proxy class by hand?

---

## 12. Common Mistakes

- **Forgetting to check if the RealSubject is already created** in a Virtual Proxy — leading to redundant expensive re-creation on every call instead of true lazy-loading-and-caching behavior.
- **Confusing Proxy with Decorator** because they look structurally similar — remember the INTENT difference: access control/lifecycle management (Proxy) versus behavior augmentation (Decorator).
- **Putting too much unrelated logic inside a single Proxy** — if a Proxy starts handling caching, access control, AND logging all at once, consider splitting these into SEPARATE, chainable proxies instead (better SRP).
- **Not handling thread-safety in lazy initialization** — in multi-threaded contexts, the simple `if (realImage == null)` check can create a race condition where multiple threads each create their own RealImage; proper double-checked locking or synchronization is needed for thread-safe lazy proxies.

---

## 13. Time Complexity

- **Time:** O(1) additional overhead per call for the proxy's own logic (a null check, a permission check, etc.) before/after delegating — the delegation itself is a single virtual method call.
- **Space:** O(1) additional space for the proxy's own fields (a reference to RealSubject, plus any caching data structures if used) — RealSubject's own space is unaffected, just potentially deferred.

---

## 14. Java API Examples

- **`java.lang.reflect.Proxy`**: Java's built-in mechanism for creating dynamic proxy CLASSES at runtime, implementing a given set of interfaces and routing all method calls through an `InvocationHandler`.
- **Hibernate's lazy-loaded entity proxies**: generated proxy classes (often via bytecode manipulation libraries like Byte Buddy/CGLIB) that defer database fetches until fields are actually accessed.
- **Spring AOP proxies**: transactional (`@Transactional`), security (`@PreAuthorize`), and caching (`@Cacheable`) annotations are all implemented under the hood using dynamically generated proxies wrapping the actual bean.
- **RMI stubs**: client-side stub classes generated to represent remote objects, implementing the same remote interface and handling the network communication transparently.

---

## 15. Practice Problem

Implement a **Protection Proxy for a `Document` interface** with a `view()` and `edit()` method. Create a `RealDocument` class, and a `ProtectedDocumentProxy` that takes a `userRole` (e.g., `"VIEWER"` or `"EDITOR"`) in its constructor — `view()` should always be allowed, but `edit()` should only be delegated to the RealDocument if the role is `"EDITOR"`; otherwise print an access-denied message. Demonstrate both a Viewer and an Editor attempting to edit the document.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **caching proxy for an expensive database query service**. Given a `DataService` interface with a `fetchData(String key)` method, implement a `CachingDataServiceProxy` that stores results of previous queries in an internal cache (e.g., a `HashMap`), and only delegates to the real `DatabaseDataService` when the requested key hasn't been queried before."

Think about:
- How you'd handle CACHE INVALIDATION if the underlying data changes (this reveals a common real-world complexity of caching proxies).
- Whether this Caching Proxy could be COMBINED with a Protection Proxy for the same service — and how you'd structure that composition cleanly.

---

## 17. Advanced LLD Scenario

**Design a Video Streaming Platform's Content Delivery Layer** using Proxy, where:
- Loading a full video file is EXPENSIVE (large file size) — a `VideoProxy` should defer actually loading/streaming the video data until the user presses "play," while allowing metadata (title, thumbnail, duration) to be shown IMMEDIATELY without triggering the expensive load
- Certain videos are REGION-RESTRICTED — the proxy layer should check the user's region and deny playback access before ever attempting to load restricted content (Protection Proxy behavior)
- Frequently-watched videos should be served from a LOCAL CACHE/CDN edge node rather than re-fetching from the origin server every time (Caching Proxy behavior)

Consider:
- How THREE different proxy behaviors (virtual/lazy loading, protection/region-check, caching/CDN) might need to be COMPOSED together for a single video request, and whether you'd implement this as one large proxy class or a CHAIN of smaller, focused proxies (revisit the "Common Mistakes" discussion above)
- How this scenario's layered structure connects back to the Proxy vs Decorator distinction (Section 11) — even though multiple proxies are stacked here, each one is fundamentally about controlling ACCESS (whether/when/how the real video data is reached), not about adding new user-facing FEATURES to the video itself
- Why Remote Proxy concepts also apply here, since the actual video data likely lives on a remote origin server, not the client's local machine

---

## 18. Summary

**Definition:** Proxy provides a surrogate object implementing the same interface as a real object, controlling access to it (via lazy loading, access checks, remote representation, or caching) transparently to the client.

**Intent:** Control access to an object — deferring its creation, restricting who can use it, representing it across a network boundary, or caching its results — without the client needing to know the difference.

**Key classes:** `Subject` (common interface), `RealSubject` (the actual object), `Proxy` (implements Subject, holds a reference to RealSubject, controls access to it).

**Advantages:** Transparent client-side substitution; enables lazy initialization; centralizes cross-cutting concerns; good OCP for adding new proxy behaviors.

**Disadvantages:** Extra layer of indirection and overhead; can complicate debugging; risk of proxy-chain sprawl if overused.

**Real-world use case:** Hibernate lazy-loaded entities, Spring AOP proxies, RMI stubs, API gateway access control, CDN caching layers.

**Java example:** `ProxyImage` lazily creating and delegating to a `RealImage` only on first `display()` call.

**Interview takeaway:** Always be ready to clearly distinguish Proxy from Decorator — same-looking structure, but Proxy is about controlling ACCESS while Decorator is about ADDING behavior. Stating this distinction clearly, unprompted, is a strong signal.

**One-line memory trick:** *"A credit card is a proxy for your bank account — same interface (payment), but with checks and controls layered in."*

---

*End of Topic 11. Type "Next" to proceed to Topic 12: Null Object Pattern.*