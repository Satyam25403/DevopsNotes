# Topic 10: Chain of Responsibility Pattern

---

## 1. Introduction

**Definition:**
The Chain of Responsibility Pattern is a **behavioral design pattern** that lets you pass a request along a chain of handler objects, where each handler decides EITHER to process the request OR pass it along to the next handler in the chain — decoupling the SENDER of a request from the KNOWLEDGE of which object will actually handle it.

**Why it exists / what problem it solves:**
Imagine a support-ticket system: a ticket might need to be handled by a Level-1 agent, escalated to Level-2, or escalated further to a Manager, depending on its severity. Without this pattern, you'd write a large `if-else`/`switch` block in ONE place that knows about EVERY possible handler and EVERY condition for routing to them — a rigid, hard-to-extend mess that violates OCP every time a new handler type is added.

Chain of Responsibility solves this by giving EACH handler ONLY the logic for its OWN decision ("can I handle this? if not, who's next?") and linking handlers together in a chain. The sender just hands the request to the FIRST handler and doesn't need to know anything about the rest of the chain.

**When it should be used:**
- When more than one object MAY handle a request, and the actual handler isn't known in advance
- When you want to issue a request to one of several objects WITHOUT explicitly specifying the receiver
- When the set of handlers (and their order) should be configurable/changeable at RUNTIME
- When you want to decouple request senders from request handlers entirely

**When it should NOT be used:**
- When exactly ONE object always handles a specific request type — a direct call is simpler and clearer
- When you need a GUARANTEE that the request WILL be handled — chains can silently let a request fall through with no handler if not designed carefully
- When performance is extremely critical and the chain could grow very long — each step adds overhead

**Advantages:**
- Decouples sender from receiver — sender doesn't need to know which handler will process the request
- Adding a new handler = adding a new class; existing handlers/chain code remains unchanged (good OCP)
- Chain order can be configured dynamically at runtime
- Each handler has a single, focused responsibility (good SRP)

**Disadvantages:**
- No guarantee a request will actually be handled by anyone (unless you explicitly build in a fallback/default handler)
- Debugging can be harder — following a request through a long chain requires tracing multiple objects
- If misconfigured, chain order can matter in ways that are easy to get wrong (a specific handler placed too late in the chain might never be reached because an earlier, broader handler already consumed the request)

---

## 2. Real-World Analogy

Think of **a company's expense approval process**.

If an employee submits a $50 expense, their direct **Team Lead** can approve it immediately. If it's $500, the Team Lead says "not my call" and passes it to the **Manager**. If it's $5,000, the Manager passes it further up to the **Director**. If it's $50,000, it goes all the way to the **VP**.

Critically: the employee submitting the expense doesn't need to know WHO will ultimately approve it — they just submit it to their Team Lead, and it automatically flows up the chain until someone with sufficient authority handles it. Each person in the chain only needs to know ONE thing: "can I approve this myself, or do I need to pass it up?"

---

## 3. Theory

**Core idea:** Each handler holds a reference to the NEXT handler in the chain. When a request arrives, the handler checks if it can process it. If yes, it does so (and optionally still passes it along, depending on design). If no, it forwards the request to the next handler. This continues until either a handler processes it, or the chain ends.

**Working mechanism:**
```
Client → Handler1 → Handler2 → Handler3 → (end of chain / null)
            │            │            │
        can I handle?  can I handle?  can I handle?
         (if no, pass forward)
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Handler (interface/abstract class) | Declares the method to handle a request, and holds a reference to the "next" handler |
| Concrete Handler | Implements the actual handling logic for its specific case; decides whether to handle or forward |
| Successor / Next | The reference each handler holds, pointing to the next handler in the chain |
| Client | Initiates the request by sending it to the FIRST handler in the chain |

**Class responsibilities:** each Concrete Handler is responsible for exactly TWO things — (1) deciding whether IT can handle the given request, and (2) if not, forwarding it to its successor. It should NOT know anything about handlers further down the chain beyond its immediate successor.

**Communication flow:** strictly linear and sequential by default — Client → Handler1 → Handler2 → ... — though the chain's SHAPE (which handlers, in what order) can be assembled dynamically at setup time.

---

## 4. UML / Class Diagram

```
┌─────────────────────────────┐
│      <<abstract>>                   │
│      Handler                            │
├─────────────────────────────┤
│ # nextHandler: Handler            ◇──┐  (aggregation: a Handler
├─────────────────────────────┤     │   holds a reference to the
│ + setNext(Handler): Handler        │   NEXT handler — NOT owned/
│ + handle(Request): void            │   created by it, just linked)
└──────────────┬───────────────┘     │
               △ (inherits)           │
       ┌───────┼───────────────┐     │
       │               │               │◄────────┘
┌──────┴────────┐ ┌────┴──────────┐ ┌──┴─────────────┐
│ Level1Handler        │ │ Level2Handler         │ │ ManagerHandler          │
│ + handle(Request)      │ │ + handle(Request)      │ │ + handle(Request)      │
└────────────────┘ └───────────────┘ └────────────────┘

┌─────────────────────┐
│      Client                      │
├─────────────────────┤
│ + submitRequest()            ────────→ (dependency: Client only
└─────────────────────┘             knows the FIRST Handler,
                                        nothing about the rest)
```

**Relationship explanation:**
- `Level1Handler`, `Level2Handler`, `ManagerHandler` all **inherit** from the abstract `Handler` — they share the common `nextHandler` reference and `setNext()` logic, only overriding `handle()`'s specific decision logic.
- `Handler` has an **aggregation** relationship with itself (`nextHandler: Handler`) — a handler references the next one but doesn't own its lifecycle; the chain is assembled externally (typically in `main()` or a factory/builder).
- `Client` has only a **dependency** on the FIRST `Handler` in the chain — it's completely unaware of how many handlers exist beyond that, or their internal order.

---

## 5. Java Implementation

```java
// ============================================
// Request — the data being passed through the chain
// ============================================
public class ExpenseRequest {
    private final double amount;
    private final String description;

    public ExpenseRequest(double amount, String description) {
        this.amount = amount;
        this.description = description;
    }

    public double getAmount() {
        return amount;
    }

    public String getDescription() {
        return description;
    }
}

// ============================================
// Abstract Handler — declares the chain-linking
// mechanism and the template for handling
// ============================================
public abstract class ExpenseApprover {
    // Reference to the NEXT handler in the chain — null means "end of chain"
    protected ExpenseApprover nextApprover;

    // Fluent method to build the chain: returns the passed-in handler
    // so calls can be chained like: h1.setNext(h2).setNext(h3)
    public ExpenseApprover setNext(ExpenseApprover next) {
        this.nextApprover = next;
        return next;
    }

    // Template method: subclasses implement the actual decision logic
    public abstract void approve(ExpenseRequest request);

    // Helper for subclasses to forward the request if they can't handle it
    protected void forwardToNext(ExpenseRequest request) {
        if (nextApprover != null) {
            nextApprover.approve(request);
        } else {
            // End of chain reached with nobody able to approve —
            // important to handle this explicitly rather than
            // silently doing nothing
            System.out.println("No approver found for expense: "
                    + request.getDescription() + " ($" + request.getAmount() + ")");
        }
    }
}

// ============================================
// Concrete Handlers — each with its OWN approval limit
// ============================================
public class TeamLeadApprover extends ExpenseApprover {
    private static final double LIMIT = 100.0;

    @Override
    public void approve(ExpenseRequest request) {
        if (request.getAmount() <= LIMIT) {
            System.out.println("Team Lead approved expense: "
                    + request.getDescription() + " ($" + request.getAmount() + ")");
        } else {
            // Can't handle it myself — pass it up the chain
            forwardToNext(request);
        }
    }
}

public class ManagerApprover extends ExpenseApprover {
    private static final double LIMIT = 1000.0;

    @Override
    public void approve(ExpenseRequest request) {
        if (request.getAmount() <= LIMIT) {
            System.out.println("Manager approved expense: "
                    + request.getDescription() + " ($" + request.getAmount() + ")");
        } else {
            forwardToNext(request);
        }
    }
}

public class DirectorApprover extends ExpenseApprover {
    private static final double LIMIT = 10000.0;

    @Override
    public void approve(ExpenseRequest request) {
        if (request.getAmount() <= LIMIT) {
            System.out.println("Director approved expense: "
                    + request.getDescription() + " ($" + request.getAmount() + ")");
        } else {
            forwardToNext(request);
        }
    }
}

// ============================================
// Demo — assembling the chain and using it
// ============================================
public class ChainOfResponsibilityDemo {
    public static void main(String[] args) {
        // Build the chain: TeamLead -> Manager -> Director
        ExpenseApprover teamLead = new TeamLeadApprover();
        ExpenseApprover manager = new ManagerApprover();
        ExpenseApprover director = new DirectorApprover();

        teamLead.setNext(manager).setNext(director);

        // Client only ever talks to the FIRST handler (teamLead)
        teamLead.approve(new ExpenseRequest(50, "Office supplies"));
        teamLead.approve(new ExpenseRequest(500, "Team offsite"));
        teamLead.approve(new ExpenseRequest(8000, "New laptops"));
        teamLead.approve(new ExpenseRequest(50000, "Office lease"));
    }
}
```

**Key line-by-line notes:**
- `nextApprover` is `protected` so subclasses can access it directly if needed, but the RECOMMENDED path is always through `forwardToNext()` to keep the null-check logic in ONE place.
- `setNext()` returns the handler passed IN (not `this`) — this is what enables the fluent chain-building syntax `teamLead.setNext(manager).setNext(director)`.
- Each Concrete Handler's `approve()` follows the SAME shape: check condition → handle OR forward — this consistent shape is what makes the pattern easy to extend with new handler types.
- The final `DirectorApprover` forwarding to `forwardToNext()` when `nextApprover` is `null` demonstrates explicit handling of "end of chain, nobody could approve" rather than silently failing.

---

## 6. Dry Run

**Sample input:** `teamLead.approve(new ExpenseRequest(8000, "New laptops"))`

```
1. teamLead.approve(request) called [request.amount = 8000]
   → TeamLeadApprover.approve() executes
   → 8000 <= 100? NO
   → forwardToNext(request) called
       → nextApprover (manager) is not null
       → manager.approve(request) called

2. ManagerApprover.approve() executes
   → 8000 <= 1000? NO
   → forwardToNext(request) called
       → nextApprover (director) is not null
       → director.approve(request) called

3. DirectorApprover.approve() executes
   → 8000 <= 10000? YES
   → Prints: "Director approved expense: New laptops ($8000.0)"
   → Method returns, unwinding back through Manager's and
     TeamLead's forwardToNext() calls (which have nothing
     left to do), back to main()
```

**What's happening in memory:** this is a simple sequential chain of method calls, each one nested inside the previous (`teamLead.approve()` → `forwardToNext()` → `manager.approve()` → `forwardToNext()` → `director.approve()`) — essentially a chain of calls on the call stack, unwinding once the REQUEST is finally handled (or the chain ends).

---

## 7. Real-World Software Example

- **Servlet Filters (Java EE / Jakarta EE)**: each `Filter` in a filter chain either processes the request/response or calls `chain.doFilter()` to pass it to the next filter — a textbook Chain of Responsibility implementation.
- **Logging frameworks (java.util.logging, Log4j)**: a `Logger` can pass a log record up to its parent logger, which may have its OWN handlers, continuing up the hierarchy until it reaches the root logger.
- **Exception handling / middleware chains** in frameworks like Spring (`HandlerInterceptor` chains) — each interceptor decides whether to process and continue, or short-circuit the chain.
- **Customer support ticketing systems**: tickets escalate from Level-1 support → Level-2 → Engineering, exactly matching the pattern's structure.
- **Event bubbling in GUI frameworks**: a click event can be handled by the clicked component, or bubble up to its parent container, and further up, until something handles it (or it reaches the root with no handler).

---

## 8. Internal Working

**Object creation:** handlers are constructed independently, then LINKED together via `setNext()` calls — the chain's shape is decided at ASSEMBLY time, entirely separate from each handler's own internal logic.

**Runtime interactions / call flow:** each `approve()` call either terminates the chain (by handling the request) or makes a DIRECT method call to `nextApprover.approve()` — this means the chain executes as a straightforward sequence of NESTED method calls, growing the call stack by one frame per handler traversed, until either handled or the chain ends.

**Memory usage:** each handler holds exactly ONE reference (`nextApprover`) regardless of how long the chain is — memory overhead is O(1) per handler, and the chain itself is just a singly-linked list of handler objects.

**Dynamic binding:** the specific type of `nextApprover` (whether it's a `ManagerApprover`, `DirectorApprover`, or any future handler type) is resolved via dynamic dispatch at each `nextApprover.approve()` call — the `Handler` reference type never needs to change even as new handler types are introduced.

---

## 9. Before vs After

**Before (no Chain of Responsibility — one giant conditional):**

```java
public class ExpenseApprovalBad {
    public void approve(ExpenseRequest request) {
        double amount = request.getAmount();

        // ALL approval logic crammed into ONE method —
        // adding a new approval level means editing THIS
        // method and risking breaking existing logic
        if (amount <= 100) {
            System.out.println("Team Lead approved: " + request.getDescription());
        } else if (amount <= 1000) {
            System.out.println("Manager approved: " + request.getDescription());
        } else if (amount <= 10000) {
            System.out.println("Director approved: " + request.getDescription());
        } else {
            System.out.println("No approver found: " + request.getDescription());
        }
    }
}
```

**Problems:**
- ONE method knows about EVERY approval level and its exact threshold — violates SRP badly (this class has as many responsibilities as there are approval levels).
- Adding a NEW approval level (e.g., "VP approves up to $100,000") means editing this EXISTING method, risking breaking the logic for levels that already work correctly — violates OCP.
- The approval ORDER is hardcoded as `if-else` structure — reordering or reconfiguring at runtime (e.g., for a different department with different rules) is very awkward.

**After (Chain of Responsibility, as shown in Section 5):**
- Each approval level is its OWN class with its OWN single responsibility — adding "VP approval" means writing ONE new class and adding it to the chain, without touching ANY existing handler.
- The chain's structure (order, which handlers are included) can be assembled dynamically at startup — even reconfigurable per-department or per-environment.

---

## 10. SOLID Principles Connection

- **SRP**: each Concrete Handler (`TeamLeadApprover`, `ManagerApprover`, etc.) has EXACTLY one responsibility — deciding whether IT can approve a given request.
- **OCP**: adding a new handler (e.g., `VPApprover`) requires writing a new class and linking it into the chain — zero changes needed to any EXISTING handler class.
- **DIP**: each handler depends only on the abstract `ExpenseApprover` type for its `nextApprover` reference — never on a specific concrete handler class, so handlers are fully interchangeable and reorderable.
- **LSP**: any `ExpenseApprover` subclass can be substituted into any position in the chain without breaking the chain's overall behavior — the chain-building code (`setNext()`) works identically regardless of WHICH concrete handlers are involved.

---

## 11. Interview Questions

**Beginner:**
1. What problem does Chain of Responsibility solve compared to a single large if-else/switch block?
2. In your own words, what does each handler in the chain need to know about?
3. What happens if a request reaches the end of the chain without being handled?

**Intermediate:**
4. How would you dynamically reconfigure the ORDER of handlers in the chain at runtime? What design considerations does this raise?
5. Can a handler in the chain choose to BOTH process a request AND still pass it forward? Give a scenario where this makes sense (e.g., logging/auditing).
6. How does this pattern relate to the "Decorator" pattern (Topic 7) — both involve linked/chained objects. What's the KEY structural difference?
   *Answer: Decorator focuses on ADDING behavior/responsibilities to an object while preserving its interface — EVERY decorator in the chain typically DOES execute (wrapping the call). Chain of Responsibility focuses on ROUTING a request to WHICHEVER handler can process it — often only ONE handler in the chain actually "does the work," and the rest simply forward without acting.*

**Advanced:**
7. How would you implement a chain where MULTIPLE handlers should all get a chance to act on the request (not stopping at the first one that "can handle" it) — e.g., an event that should notify ALL interested handlers, not just the first?
8. What are the risks of a very LONG chain, both in terms of performance and maintainability? How would you mitigate them?
9. How does Java's Servlet `FilterChain` implement this pattern, and what's the role of `chain.doFilter()` versus NOT calling it?
   *Answer: Each `Filter.doFilter()` receives a reference to the chain; calling `chain.doFilter(request, response)` explicitly passes control to the NEXT filter (or the final servlet); NOT calling it deliberately SHORT-CIRCUITS the chain — e.g., an authentication filter that rejects an unauthenticated request and never calls `chain.doFilter()`, stopping further processing entirely.*
10. Compare Chain of Responsibility with the Strategy pattern (Topic 5) — both involve interchangeable behavior, but how do they differ structurally and in intent?
    *Answer: Strategy involves the CLIENT explicitly choosing and injecting ONE algorithm/strategy to use. Chain of Responsibility involves the REQUEST traveling through a series of potential handlers, with the ACTUAL handler determined dynamically at runtime based on each handler's own internal check — the client doesn't choose the handler directly at all.*

---

## 12. Common Mistakes

- **Forgetting to handle "end of chain, nobody handled it"** — silently doing nothing when a request falls through the entire chain can hide real bugs; always have an explicit fallback/default behavior.
- **Building overly long chains for simple problems** — if there are only two or three fixed conditions that will never grow, a chain may be unnecessary ceremony compared to a simple conditional.
- **Handlers that know too much about EACH OTHER** (e.g., a handler checking "am I the last one?" or referencing a specific handler class by name) — this breaks the decoupling that's the entire point of the pattern.
- **Not considering handler ORDER carefully** — placing a broad/catch-all handler too EARLY in the chain can prevent more specific handlers further down from ever being reached.

---

## 13. Time Complexity

- **Time:** O(n) in the worst case, where n = number of handlers in the chain, since a request may need to traverse the ENTIRE chain before being handled (or reaching the end).
- **Space:** O(n) for the chain structure itself (n handler objects, each holding one reference) — plus O(n) call-stack depth in the worst case, since each `forwardToNext()` call is a NESTED call, not a loop.

---

## 14. Java API Examples

- **Servlet Filters (`javax.servlet.Filter` / `jakarta.servlet.Filter`)**: the canonical Java EE example — a chain of filters processing a request before it reaches the actual servlet.
- **`java.util.logging.Logger`**: loggers form a hierarchy where a log record can propagate from a child logger up to parent loggers, each with its own handlers, mirroring Chain of Responsibility.
- **Spring Security's `FilterChainProxy`**: builds a chain of security filters (authentication, authorization, CSRF checks, etc.), each deciding whether to act and/or pass the request along.
- **Spring MVC's `HandlerInterceptor`** chains: interceptors registered for a request can each choose to continue the chain or short-circuit it (e.g., on failed authorization).

---

## 15. Practice Problem

Implement a **Chain of Responsibility for HTTP request validation**: create handlers `AuthenticationHandler`, `AuthorizationHandler`, and `RateLimitHandler`, each of which either PASSES the request along (if its specific check succeeds) or REJECTS it (stopping the chain) with an appropriate error message. Demonstrate a request that passes all checks, and one that gets rejected partway through.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design a **log processing system** where a log message with a SEVERITY level (`DEBUG`, `INFO`, `WARNING`, `ERROR`) should be routed to the appropriate handler: `DEBUG`/`INFO` messages are just written to a console handler, `WARNING` messages are written to a file handler AND the console, and `ERROR` messages are written to file, console, AND trigger an email alert handler. Design this using Chain of Responsibility such that adding a new severity level or a new handler type requires minimal changes to existing code."

Think about:
- Whether a request should stop at the FIRST handler that processes it, or continue through MULTIPLE handlers (hint: this scenario needs "handle and continue" behavior, not "handle and stop").
- How you'd structure the base `Handler` class differently to support this "multiple handlers act" behavior versus the strict "only one handler acts" behavior shown in Section 5.

---

## 17. Advanced LLD Scenario

**Design a Vending Machine's Coin/Denomination Handling System** using Chain of Responsibility, where:
- The machine needs to determine change to give back, breaking down an amount into denominations (e.g., $1, $0.25, $0.10, $0.05, $0.01) — each "denomination handler" in the chain handles as many of ITS denomination as possible, then passes the REMAINDER to the next handler
- New denominations (e.g., a $0.50 coin) should be easy to add to the chain without modifying existing handlers
- Consider how this problem is subtly DIFFERENT from the expense-approval example — here, EVERY handler in the chain may contribute SOMETHING to the final result (partial handling), rather than exactly ONE handler fully owning the request

Consider:
- How you'd modify the base `Handler` class to return/accumulate a RESULT (the change breakdown) rather than simply having "one handler wins" semantics
- How this relates back to the "Common Mistakes" point about not all Chain of Responsibility implementations being "first handler that can process, stops there" — some legitimately need "let everyone contribute, then combine results"
- Why this pattern is a much more natural, industry-recognized fit for this problem than, say, Strategy (Topic 5) would be

---

## 18. Summary

**Definition:** Chain of Responsibility passes a request along a chain of handler objects, where each handler decides to process the request or forward it to the next handler, decoupling senders from the specific object that ultimately handles the request.

**Intent:** Avoid coupling the sender of a request to a specific receiver by giving multiple objects a chance to handle it, with the chain's shape configurable independently of the sender.

**Key classes:** `Handler` (abstract, holds `nextHandler` reference), `ConcreteHandler` implementations (each with its own condition/logic), `Client` (talks only to the first handler).

**Advantages:** Decouples sender from receiver; easy to add new handlers (good OCP); runtime-configurable chain order; each handler has single responsibility.

**Disadvantages:** No guarantee a request gets handled unless explicitly designed for; can be hard to debug/trace; handler order matters and misordering causes subtle bugs.

**Real-world use case:** Servlet filter chains, logging hierarchies, support ticket escalation, GUI event bubbling.

**Java example:** `ExpenseApprover` chain — `TeamLeadApprover` → `ManagerApprover` → `DirectorApprover`, each approving within its own limit or forwarding.

**Interview takeaway:** Be ready to distinguish "first handler that can process, stops there" (most common textbook version) from "every handler contributes, then results are combined" (a legitimate variant) — recognizing which variant a given problem calls for shows deeper understanding.

**One-line memory trick:** *"Pass it up the chain until someone can approve it — like an expense report climbing the org chart."*

---

*End of Topic 10. Type "Next" to proceed to Topic 11: Proxy Pattern.*