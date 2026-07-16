# Topic 23: Interpreter Pattern

---

## 1. Introduction

**Definition:**
The Interpreter Pattern is a **behavioral design pattern** that defines a REPRESENTATION for a language's GRAMMAR, along with an INTERPRETER that uses this representation to EVALUATE (interpret) SENTENCES/EXPRESSIONS in that language — each GRAMMAR RULE is represented as a CLASS, and INTERPRETING an expression means RECURSIVELY evaluating a TREE of these rule-representing objects.

**Why it exists / what problem it solves:**
Consider needing to EVALUATE simple mathematical or logical EXPRESSIONS at RUNTIME — e.g., a user TYPES in `"5 + 3 * 2"`, or a BUSINESS RULES engine needs to EVALUATE `"age > 18 AND hasLicense"`. HARD-CODING logic to PARSE and EVALUATE every POSSIBLE expression using STRING manipulation and MANUAL parsing logic quickly becomes UNWIELDY and HARD to extend as the GRAMMAR grows more COMPLEX (more OPERATORS, more RULES).

Interpreter solves this by MODELING the GRAMMAR itself as a CLASS HIERARCHY — EACH TYPE of grammar RULE (a NUMBER, an ADDITION operation, a MULTIPLICATION operation, a VARIABLE, an AND/OR expression) becomes its OWN CLASS implementing a COMMON `Expression` interface with an `interpret()` method. A COMPLEX expression is represented as a TREE of these objects (mirroring the expression's STRUCTURE), and INTERPRETING it means RECURSIVELY calling `interpret()` on the ROOT, which CASCADES down through the WHOLE tree.

**When it should be used:**
- When you have a SIMPLE, WELL-DEFINED GRAMMAR/LANGUAGE that needs to be REPEATEDLY PARSED and EVALUATED (e.g., SEARCH query syntax, SIMPLE scripting/rule languages, mathematical EXPRESSIONS)
- When the GRAMMAR is SIMPLE enough that REPRESENTING it as a CLASS HIERARCHY remains MANAGEABLE (Interpreter does NOT scale well to LARGE, COMPLEX grammars — see disadvantages)
- When EFFICIENCY of PARSING/EVALUATION is NOT the PRIMARY concern (Interpreter TRADES some PERFORMANCE for a CLEAN, EXTENSIBLE representation of grammar RULES)

**When it should NOT be used:**
- When the GRAMMAR is COMPLEX (many RULES, many EDGE CASES) — Interpreter's CLASS-PER-RULE approach becomes UNWIELDY; a PROPER parser GENERATOR tool (like ANTLR, YACC) is a BETTER fit for COMPLEX grammars
- When PERFORMANCE is CRITICAL — the RECURSIVE, OBJECT-based evaluation approach is GENERALLY SLOWER than a HAND-OPTIMIZED, DIRECT parsing/evaluation approach
- When the EXPRESSION LANGUAGE ALREADY has a WELL-ESTABLISHED, MATURE library/tool AVAILABLE (e.g., using a REGEX library instead of BUILDING your OWN mini pattern-matching GRAMMAR)

**Advantages:**
- Makes it EASY to EXTEND the GRAMMAR with NEW RULES/EXPRESSION types (each is JUST a NEW class implementing the COMMON interface)
- The GRAMMAR's STRUCTURE is DIRECTLY reflected in the CODE'S class HIERARCHY, making it RELATIVELY easy to UNDERSTAND the LANGUAGE'S structure by READING the classes
- NATURALLY supports RECURSIVE, NESTED expressions (since EXPRESSIONS can CONTAIN OTHER expressions as CHILDREN, mirroring COMPOSITE's tree structure, Topic 14)

**Disadvantages:**
- Does NOT SCALE well to COMPLEX grammars — the NUMBER of CLASSES grows with the NUMBER of GRAMMAR rules, and COMPLEX grammars can require MANY, MANY classes
- GENERALLY SLOWER than a PURPOSE-BUILT parser/evaluator, due to the OVERHEAD of RECURSIVE OBJECT-based dispatch
- For ANYTHING beyond a SIMPLE, well-contained grammar, a DEDICATED PARSER GENERATOR tool is TYPICALLY a BETTER engineering choice

---

## 2. Real-World Analogy

Think of **translating a simple recipe written in a controlled, restricted vocabulary**.

Imagine a COOKING instruction language with JUST a FEW types of SENTENCES: "ADD [ingredient]", "STIR for [N] minutes", "BAKE at [temperature] for [N] minutes". EACH sentence TYPE has its OWN CLEAR structure and MEANING. To EXECUTE a RECIPE written in this LANGUAGE, you'd INTERPRET each LINE according to WHICH "type" of instruction it IS — an "ADD" instruction is HANDLED differently from a "BAKE" instruction, but BOTH follow the SAME overall PATTERN of "read this TYPE of instruction, and DO the corresponding ACTION." Interpreter FORMALIZES this — EACH SENTENCE TYPE becomes its OWN CLASS that KNOWS HOW to "execute ITSELF."

---

## 3. Theory

**Core idea:** Define an `Expression` INTERFACE with an `interpret(Context)` method. Create TWO KINDS of CONCRETE Expression classes: **Terminal Expressions** (representing the BASIC, ATOMIC elements of the GRAMMAR — e.g., a NUMBER, a VARIABLE) and **Non-Terminal Expressions** (representing COMPOSITE RULES that COMBINE other expressions — e.g., ADDITION combines TWO sub-expressions). INTERPRETING a COMPLEX expression means BUILDING a TREE of these objects (mirroring the expression's GRAMMATICAL structure) and CALLING `interpret()` on the ROOT, which RECURSIVELY cascades DOWN through the ENTIRE tree.

**Working mechanism:**
```
        AddExpression (non-terminal)
        /                          \
NumberExpression(5)        MultiplyExpression (non-terminal)
                              /                    \
                    NumberExpression(3)      NumberExpression(2)

interpret() on AddExpression RECURSIVELY calls interpret()
on its CHILDREN, COMBINING their RESULTS: 5 + (3 * 2) = 11
```

**Important terminology:**
| Term | Meaning |
|---|---|
| Expression | The common interface declaring `interpret()` |
| Terminal Expression | Represents an ATOMIC grammar element (e.g., a number, a variable) |
| Non-Terminal Expression | Represents a COMPOSITE rule COMBINING other expressions (e.g., addition, AND/OR) |
| Context | Holds GLOBAL information needed DURING interpretation (e.g., VARIABLE values) |

**Class responsibilities:** a TERMINAL Expression is responsible for RETURNING its OWN, DIRECT value. A NON-TERMINAL Expression is responsible for RECURSIVELY INTERPRETING its CHILD expressions and COMBINING their RESULTS according to its OWN GRAMMAR rule (e.g., addition SUMS its children's results).

---

## 4. UML / Class Diagram

```
┌─────────────────────┐
│      <<interface>>          │
│      Expression                  │
├─────────────────────┤
│ + interpret(): int             │
└──────────┬──────────┘
           △ (implements)
   ┌───────┼────────────────┐
   │                                │
┌──┴──────────────┐  ┌──┴──────────────────┐
│ NumberExpression         │  │ AddExpression                 │
│ (TERMINAL)                  │  │ (NON-TERMINAL)                    │
├──────────────────┤  ├──────────────────────┤
│ - value: int                 │  │ - left: Expression       ◇──┐
│ + interpret()                 │  │ - right: Expression      ◇──┤ (composition:
│   { return value; }             │  ├──────────────────────┤  Non-Terminal
└──────────────────┘  │ + interpret()                    │  HOLDS other
                          │   { return left.interpret()      │  Expressions —
                          │     + right.interpret(); }        │  RECURSIVE tree
                          └──────────────────────┘  structure, like
                                                                  COMPOSITE, Topic 14)

┌─────────────────────┐
│      MultiplyExpression         │
│      (NON-TERMINAL)              │
├─────────────────────┤
│ - left: Expression                │
│ - right: Expression               │
│ + interpret()                     │
│   { return left.interpret()          │
│     * right.interpret(); }           │
└─────────────────────┘
```

**Relationship explanation:**
- `NumberExpression` (TERMINAL) **implements** `Expression` DIRECTLY, RETURNING its STORED value with NO further RECURSION.
- `AddExpression`/`MultiplyExpression` (NON-TERMINAL) **implement** `Expression` and hold REFERENCES to `left`/`right` CHILD Expressions — this is a **composition** relationship STRUCTURALLY VERY SIMILAR to COMPOSITE (Topic 14)'s TREE structure, where a NODE can CONTAIN OTHER nodes RECURSIVELY.
- Calling `interpret()` on the ROOT of the TREE TRIGGERS a RECURSIVE CASCADE of `interpret()` calls DOWN through EVERY node, with RESULTS COMBINING back UP as the RECURSION UNWINDS.

---

## 5. Java Implementation

```java
// ============================================
// Expression interface — common to ALL grammar rules
// ============================================
public interface Expression {
    int interpret();
}

// ============================================
// Terminal Expression — represents an ATOMIC value,
// requires NO further recursion to evaluate
// ============================================
public class NumberExpression implements Expression {
    private final int value;

    public NumberExpression(int value) {
        this.value = value;
    }

    @Override
    public int interpret() {
        // DIRECTLY returns its OWN stored value — no recursion needed
        return value;
    }
}

// ============================================
// Non-Terminal Expressions — COMBINE other expressions
// according to a SPECIFIC grammar rule
// ============================================
public class AddExpression implements Expression {
    private final Expression left;
    private final Expression right;

    public AddExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }

    @Override
    public int interpret() {
        // RECURSIVELY interprets BOTH children, then COMBINES
        // their results according to THIS rule's semantics (addition)
        return left.interpret() + right.interpret();
    }
}

public class MultiplyExpression implements Expression {
    private final Expression left;
    private final Expression right;

    public MultiplyExpression(Expression left, Expression right) {
        this.left = left;
        this.right = right;
    }

    @Override
    public int interpret() {
        return left.interpret() * right.interpret();
    }
}

// ============================================
// Demo — manually building the TREE for: 5 + (3 * 2)
// ============================================
public class InterpreterDemo {
    public static void main(String[] args) {
        // Building the expression tree representing: 5 + (3 * 2)
        Expression five = new NumberExpression(5);
        Expression three = new NumberExpression(3);
        Expression two = new NumberExpression(2);

        Expression multiplication = new MultiplyExpression(three, two); // 3 * 2
        Expression addition = new AddExpression(five, multiplication);  // 5 + (3 * 2)

        // Interpreting the ROOT triggers the ENTIRE recursive cascade
        int result = addition.interpret();
        System.out.println("Result of 5 + (3 * 2) = " + result);
    }
}
```

**Key line-by-line notes:**
- `NumberExpression.interpret()` is the BASE CASE of the RECURSION — it simply RETURNS its OWN value, with NO further DELEGATION.
- `AddExpression.interpret()` and `MultiplyExpression.interpret()` both RECURSIVELY call `interpret()` on THEIR `left`/`right` CHILDREN FIRST, then COMBINE the RESULTS according to their OWN specific OPERATION — this is EXACTLY the same RECURSIVE-delegation STRUCTURE seen in COMPOSITE (Topic 14).
- The TREE (`five`, `three`, `two`, `multiplication`, `addition`) is BUILT MANUALLY in this SIMPLIFIED example — in a REAL system, a SEPARATE PARSER would CONSTRUCT this TREE AUTOMATICALLY from a RAW STRING like `"5 + 3 * 2"`.

---

## 6. Dry Run

**Sample input:** `addition.interpret()` where `addition = AddExpression(five, MultiplyExpression(three, two))`

```
1. addition.interpret() called [AddExpression: left=five, right=multiplication]
   → AddExpression.interpret() executes
   → Calls left.interpret() → five.interpret()
       → NumberExpression.interpret() executes → RETURNS 5 (base case)
   → Calls right.interpret() → multiplication.interpret()
       → MultiplyExpression.interpret() executes
       → Calls left.interpret() → three.interpret()
           → NumberExpression.interpret() executes → RETURNS 3
       → Calls right.interpret() → two.interpret()
           → NumberExpression.interpret() executes → RETURNS 2
       → RETURNS 3 * 2 = 6
   → addition.interpret() RETURNS 5 + 6 = 11
```

**What's happening in memory:** this is a CLASSIC RECURSIVE TREE TRAVERSAL — CALLING `interpret()` on the ROOT PUSHES a STACK FRAME, which THEN calls `interpret()` on its CHILDREN (pushing MORE stack frames), UNTIL reaching the LEAF (TERMINAL) nodes, which RETURN IMMEDIATELY (the BASE CASE) — RESULTS then PROPAGATE BACK UP through the STACK as EACH frame COMBINES its CHILDREN's results and RETURNS.

---

## 7. Real-World Software Example

- **Regular expression engines**: internally, a REGEX PATTERN is often COMPILED into a TREE/GRAPH of "expression" objects (LITERAL characters, QUANTIFIERS, ALTERNATIONS), which are then INTERPRETED/EVALUATED against an INPUT string.
- **SQL query PARSERS/EVALUATORS**: some SIMPLE query engines REPRESENT a QUERY'S WHERE clause as a TREE of CONDITION expressions (AND/OR/comparison NODES), EVALUATED against EACH row.
- **Business rules engines**: EVALUATING CONFIGURABLE, USER-DEFINED business RULES (e.g., "IF age > 18 AND hasLicense THEN approve") REPRESENTED as an EXPRESSION tree.
- **Simple calculator/spreadsheet FORMULA evaluators**: PARSING and EVALUATING mathematical EXPRESSIONS typed by USERS (as in THIS example).
- **Template engines**: SOME simple TEMPLATE languages (variable SUBSTITUTION, CONDITIONALS) are IMPLEMENTED using an EXPRESSION-tree-based INTERPRETER approach.

---

## 8. Internal Working

**Object creation:** the EXPRESSION TREE is typically CONSTRUCTED by a SEPARATE PARSER component (NOT shown in THIS simplified example, WHICH builds the tree MANUALLY) — the PARSER reads a RAW STRING representation of the EXPRESSION and BUILDS the CORRESPONDING tree of `Expression` objects, MIRRORING the GRAMMAR'S structure.

**Runtime interactions / call flow:** INTERPRETING an EXPRESSION is a STRAIGHTFORWARD RECURSIVE TRAVERSAL — EACH NON-TERMINAL node's `interpret()` call CASCADES DOWN to its CHILDREN FIRST (POST-ORDER style: CHILDREN are EVALUATED BEFORE the PARENT COMBINES their RESULTS), UNTIL reaching TERMINAL nodes which RETURN DIRECTLY.

**Memory usage:** MEMORY is PROPORTIONAL to the SIZE of the EXPRESSION tree (NUMBER of NODES) — for a SIMPLE expression like `"5 + 3 * 2"`, this is a SMALL, MANAGEABLE tree, but a COMPLEX GRAMMAR with MANY NESTED expressions could REQUIRE a SIGNIFICANTLY larger tree, and a CORRESPONDINGLY DEEPER call STACK during INTERPRETATION.

**Comparison to Composite (Topic 14):** Interpreter's TREE structure is STRUCTURALLY nearly IDENTICAL to COMPOSITE's — BOTH involve NODES that can CONTAIN OTHER nodes RECURSIVELY, with a COMMON interface ENABLING UNIFORM treatment. The KEY DIFFERENCE is INTENT: COMPOSITE is about REPRESENTING PART-WHOLE HIERARCHIES (files/folders, org charts); INTERPRETER is SPECIFICALLY about REPRESENTING and EVALUATING a LANGUAGE'S GRAMMAR.

---

## 9. Before vs After

**Before (no Interpreter — ad-hoc STRING parsing/evaluation):**

```java
public class ExpressionEvaluatorBad {
    // Attempting to evaluate SIMPLE arithmetic expressions using
    // MANUAL string manipulation — becomes UNWIELDY FAST as
    // the GRAMMAR grows (parentheses, operator PRECEDENCE, etc.)
    public int evaluate(String expression) {
        // Extremely FRAGILE, HARD-CODED parsing logic — this
        // EXAMPLE doesn't even handle OPERATOR PRECEDENCE correctly,
        // and would need EXTENSIVE, ERROR-PRONE string-manipulation
        // code to handle NESTED parentheses, MULTIPLE operators, etc.
        String[] tokens = expression.split(" ");
        int result = Integer.parseInt(tokens[0]);
        for (int i = 1; i < tokens.length; i += 2) {
            String operator = tokens[i];
            int operand = Integer.parseInt(tokens[i + 1]);
            if (operator.equals("+")) {
                result += operand;
            } else if (operator.equals("*")) {
                result *= operand; // WRONG precedence handling!
            }
            // Adding SUPPORT for a NEW operator means EDITING
            // this method's CONDITIONAL logic AGAIN
        }
        return result;
    }
}
```

**Problems:**
- MANUALLY parsing/evaluating STRINGS with CONDITIONAL logic becomes EXTREMELY FRAGILE and HARD to get CORRECT as the GRAMMAR grows (handling OPERATOR PRECEDENCE, PARENTHESES, and MORE OPERATORS CORRECTLY requires INCREASINGLY COMPLEX, ERROR-PRONE STRING-manipulation code).
- Adding a NEW OPERATOR/GRAMMAR RULE means EDITING this METHOD'S conditional LOGIC directly — POOR OCP.
- The GRAMMAR'S STRUCTURE is NOT REPRESENTED anywhere in the CODE — it's IMPLICIT in the PARSING logic, making the CODE HARDER to UNDERSTAND and REASON about.

**After (Interpreter, as shown in Section 5):**
- The GRAMMAR'S STRUCTURE is EXPLICITLY represented as a TREE of `Expression` OBJECTS — EASIER to UNDERSTAND and REASON about.
- Adding a NEW GRAMMAR RULE (e.g., SUBTRACTION) means WRITING ONE NEW class IMPLEMENTING `Expression` — EXISTING expression CLASSES remain UNCHANGED.
- CORRECT OPERATOR PRECEDENCE is HANDLED NATURALLY by HOW the TREE is STRUCTURED (e.g., `MultiplyExpression` being NESTED INSIDE `AddExpression`'s RIGHT child CORRECTLY reflects that MULTIPLICATION happens FIRST).

---

## 10. SOLID Principles Connection

- **OCP**: ADDING a NEW GRAMMAR RULE (e.g., `SubtractExpression`, `DivideExpression`) requires ONLY a NEW class — EXISTING Expression classes and the OVERALL interpretation MECHANISM remain UNCHANGED.
- **SRP**: EACH Expression CLASS has EXACTLY ONE responsibility — INTERPRETING its OWN SPECIFIC grammar RULE.
- **LSP**: ANY `Expression` implementation can be SUBSTITUTED as a CHILD of ANOTHER expression, and INTERPRETATION REMAINS CORRECT REGARDLESS of WHICH SPECIFIC expression TYPES are INVOLVED.

---

## 11. Interview Questions

**Beginner:**
1. What problem does the Interpreter pattern SOLVE, in your OWN words?
2. What's the DIFFERENCE between a TERMINAL expression and a NON-TERMINAL expression?
3. Why is INTERPRETING an expression DESCRIBED as a RECURSIVE process?

**Intermediate:**
4. How does OPERATOR PRECEDENCE (e.g., MULTIPLICATION before ADDITION) get CORRECTLY REFLECTED in the Interpreter pattern's DESIGN?
5. Who is TYPICALLY responsible for BUILDING the EXPRESSION tree from a RAW STRING INPUT? Is THIS PART of the Interpreter pattern ITSELF?
   *Answer: BUILDING the tree from RAW text is TYPICALLY the JOB of a SEPARATE PARSER/LEXER component — the Interpreter PATTERN ITSELF is CONCERNED with HOW to REPRESENT and EVALUATE the ALREADY-CONSTRUCTED tree, NOT with THE PARSING PROCESS that PRODUCES it. In PRACTICE, the TWO are OFTEN BUILT TOGETHER, but they're CONCEPTUALLY SEPARATE CONCERNS.*
6. Compare Interpreter with COMPOSITE (Topic 14) — both involve RECURSIVE TREE structures. What's the KEY DIFFERENCE in INTENT?
   *Answer: Composite is about REPRESENTING PART-WHOLE HIERARCHIES UNIFORMLY (files/folders, ORG charts) — its FOCUS is STRUCTURAL. Interpreter is SPECIFICALLY about REPRESENTING and EVALUATING a LANGUAGE'S GRAMMAR — its FOCUS is on INTERPRETING MEANING according to LANGUAGE RULES. STRUCTURALLY they're NEARLY IDENTICAL (RECURSIVE tree, COMMON interface), but their DOMAIN/PURPOSE DIFFERS.*

**Advanced:**
7. Why does Interpreter NOT SCALE well to COMPLEX grammars? What ALTERNATIVE approaches (e.g., PARSER GENERATORS like ANTLR) are TYPICALLY used INSTEAD for COMPLEX, REAL-WORLD LANGUAGES?
   *Answer: COMPLEX GRAMMARS require MANY, MANY DISTINCT grammar RULES, EACH becoming its OWN CLASS — this can result in an UNWIELDY NUMBER of SMALL classes, and MAINTAINING/EXTENDING such a LARGE class HIERARCHY becomes INCREASINGLY DIFFICULT. TOOLS like ANTLR or YACC GENERATE EFFICIENT PARSERS/EVALUATORS AUTOMATICALLY from a FORMAL GRAMMAR SPECIFICATION, AVOIDING the NEED to MANUALLY WRITE and MAINTAIN a LARGE CLASS hierarchy, and TYPICALLY PRODUCING SIGNIFICANTLY MORE EFFICIENT parsers.*
8. How would you EXTEND this EXAMPLE to SUPPORT VARIABLES (e.g., interpreting `"x + 3"` WHERE `x`'s VALUE is PROVIDED SEPARATELY)? (Hint: consider a "CONTEXT" object PASSED to `interpret()`, MAPPING variable NAMES to VALUES.)
9. Discuss the PERFORMANCE IMPLICATIONS of the RECURSIVE, OBJECT-based EVALUATION approach COMPARED to a MORE DIRECT, IMPERATIVE parsing/evaluation APPROACH (e.g., a HAND-WRITTEN RECURSIVE-DESCENT parser WITHOUT the OBJECT-per-rule OVERHEAD).
10. How does a REGEX ENGINE'S INTERNAL COMPILATION of a PATTERN into an EXECUTABLE STRUCTURE relate CONCEPTUALLY to the Interpreter PATTERN?

---

## 12. Common Mistakes

- **Applying Interpreter to a GRAMMAR that's TOO COMPLEX** — leading to an UNMANAGEABLE NUMBER of classes; a DEDICATED parser GENERATOR is USUALLY the BETTER CHOICE beyond a CERTAIN complexity THRESHOLD.
- **Conflating the PARSER (which BUILDS the tree from RAW text) with the INTERPRETER PATTERN ITSELF (which DEFINES how to EVALUATE an ALREADY-BUILT tree)** — these are RELATED but CONCEPTUALLY DISTINCT RESPONSIBILITIES.
- **Not HANDLING OPERATOR PRECEDENCE CORRECTLY when BUILDING the expression TREE** — the TREE'S STRUCTURE MUST CORRECTLY REFLECT PRECEDENCE RULES (e.g., MULTIPLICATION nested WITHIN addition when it SHOULD happen FIRST); getting THIS wrong PRODUCES INCORRECT evaluation RESULTS.
- **Ignoring PERFORMANCE IMPLICATIONS** for PERFORMANCE-CRITICAL applications — Interpreter's RECURSIVE, OBJECT-heavy APPROACH is GENERALLY SLOWER than a MORE DIRECT evaluation STRATEGY.

---

## 13. Time Complexity

- **Time:** O(n) to INTERPRET an EXPRESSION tree, WHERE n = the TOTAL NUMBER of NODES in the tree (EVERY node's `interpret()` is CALLED EXACTLY once).
- **Space:** O(n) for the TREE STRUCTURE itself + O(d) ADDITIONAL call-STACK SPACE during RECURSIVE interpretation, WHERE d = the TREE'S MAXIMUM DEPTH.

---

## 14. Java API Examples

- **`java.util.regex.Pattern`**: INTERNALLY, REGEX patterns are COMPILED into an INTERNAL representation CONCEPTUALLY SIMILAR to an EXPRESSION tree, WHICH is THEN "INTERPRETED" against INPUT text DURING MATCHING.
- **Spring Expression Language (SpEL)**: PARSES and EVALUATES EXPRESSIONS like `"#{someObject.someProperty}"` USING an INTERNAL tree-based EVALUATION mechanism CONCEPTUALLY ALIGNED with Interpreter.
- **JSTL / EL (Expression Language) in JSP**: EVALUATES EXPRESSIONS EMBEDDED in JSP PAGES USING a SIMILAR tree-based INTERPRETATION APPROACH.
- **`javax.script` (JSR 223) scripting ENGINES**: PROVIDE a WAY to EMBED and EXECUTE SCRIPTS (e.g., JavaScript) WITHIN JAVA applications — the UNDERLYING SCRIPT ENGINES TYPICALLY use SOME FORM of INTERPRETATION (or COMPILATION) INTERNALLY.

---

## 15. Practice Problem

Implement an **Interpreter for BOOLEAN LOGIC expressions**: create an `Expression` interface with `interpret(): boolean`. Implement `VariableExpression` (a TERMINAL, LOOKING UP a NAMED boolean VALUE from a `Map<String, Boolean>` CONTEXT), `AndExpression`, and `OrExpression` (NON-TERMINALS, COMBINING TWO SUB-expressions with LOGICAL AND/OR). Demonstrate EVALUATING an EXPRESSION TREE representing `"isAdult AND (hasLicense OR hasPermit)"` GIVEN a CONTEXT MAP with SAMPLE boolean VALUES for EACH variable.

---

## 16. Medium-Level Exercise

**Interview-style problem:** "Design an **Interpreter for a SIMPLE SEARCH QUERY language**, SUPPORTING EXPRESSIONS like `"category:electronics AND price<500"`. Design an `Expression` INTERFACE where `interpret(Product product): boolean` DETERMINES whether a GIVEN `Product` MATCHES the EXPRESSION. Implement `CategoryEquals`, `PriceLessThan` (TERMINALS, CHECKING a SPECIFIC product FIELD), and `AndExpression`/`OrExpression` (NON-TERMINALS). Demonstrate FILTERING a LIST of PRODUCTS using a BUILT expression TREE."

Think about:
- How this SCENARIO ILLUSTRATES that the "CONTEXT" being INTERPRETED against ISN'T ALWAYS a SIMPLE VARIABLE map — HERE, it's an ENTIRE `Product` OBJECT being TESTED against EACH TERMINAL expression's SPECIFIC field-CHECKING logic.
- WHETHER this DESIGN could be COMBINED with SPECIFICATION-style PREDICATE composition (a COMMON ALTERNATIVE to Interpreter for THIS EXACT kind of "FILTERING" USE case) — DISCUSS the TRADEOFFS of Interpreter VERSUS SIMPLY using JAVA's `Predicate<T>` INTERFACE WITH `.and()`/`.or()` COMBINATORS DIRECTLY (WHICH JAVA ALREADY PROVIDES built-in).

---

## 17. Advanced LLD Scenario

**Design a Configurable Business Rules Engine for a Loan Approval System** using Interpreter, where:
- BUSINESS ANALYSTS (NON-programmers) need to be able to DEFINE and MODIFY approval RULES like `"(creditScore > 700 AND income > 50000) OR (creditScore > 800)"` WITHOUT REQUIRING a CODE deployment for EVERY rule CHANGE
- The SYSTEM needs to PARSE these RULES from a CONFIGURATION FILE/DATABASE (represented as TEXT strings) into an EXPRESSION TREE, and EVALUATE them against EACH LOAN APPLICATION'S data
- Consider the TRADEOFFS of BUILDING a CUSTOM Interpreter-based SOLUTION VERSUS ADOPTING an EXISTING, MATURE rules-ENGINE LIBRARY (e.g., Drools) for a REAL, PRODUCTION-GRADE system — this IS a GENUINE, PRACTICAL engineering DECISION POINT

Consider:
- Why a SIMPLE, HAND-ROLLED Interpreter MIGHT be APPROPRIATE for a SMALL, STABLE set of RULE TYPES (comparisons, AND/OR combinations), but WOULD LIKELY become UNWIELDY if the BUSINESS later NEEDS MUCH MORE SOPHISTICATED rule CAPABILITIES (NESTED conditionals, FUNCTION calls, DATE arithmetic, etc.)
- How this SCENARIO directly CONNECTS to INTERVIEW QUESTION 7's DISCUSSION of Interpreter's SCALABILITY LIMITS — a STRONG CANDIDATE should be ABLE to ARTICULATE WHEN a HAND-ROLLED Interpreter is the RIGHT choice VERSUS WHEN a MATURE, EXISTING RULES ENGINE or PARSER GENERATOR would be a BETTER, MORE PRAGMATIC engineering DECISION
- Why THIS SCENARIO, LIKE OTHERS in THIS series, REWARDS CANDIDATES who can DISCUSS NOT JUST "HOW to IMPLEMENT the pattern" but ALSO "WHEN NOT to REACH for it, and WHAT to use INSTEAD" — a HALLMARK of GENUINE, PRACTICAL SENIOR-level DESIGN judgment

---

## 18. Summary

**Definition:** Interpreter defines a representation for a language's grammar as a class hierarchy, along with a way to interpret sentences in that language by recursively evaluating a tree of these grammar-rule objects.

**Intent:** Provide a clean, extensible way to represent and evaluate a simple, well-defined grammar, making it easy to add new grammar rules without modifying existing ones.

**Key classes:** `Expression` (common interface with `interpret()`), `Terminal Expression` (atomic grammar elements), `Non-Terminal Expression` (composite rules combining sub-expressions), `Context` (holds external data like variable values).

**Advantages:** Easy to extend with new grammar rules; grammar structure is directly reflected in code; naturally supports recursive, nested expressions.

**Disadvantages:** Does not scale well to complex grammars; generally slower than purpose-built parsers; often better replaced by parser generator tools for anything non-trivial.

**Real-world use case:** Regex engines, SQL/business rules evaluators, template engines, Spring Expression Language (SpEL), simple calculator/formula evaluators.

**Java example:** `AddExpression`/`MultiplyExpression` recursively interpreting `NumberExpression` leaves to evaluate `5 + (3 * 2)`.

**Interview takeaway:** Be ready to clearly distinguish Interpreter (representing and evaluating a LANGUAGE'S GRAMMAR) from Composite (representing PART-WHOLE HIERARCHIES, Topic 14) despite their nearly identical recursive tree structure — and be ready to discuss WHEN Interpreter's scalability limits mean a parser generator or existing rules-engine library is the more pragmatic real-world choice.

**One-line memory trick:** *"Each type of recipe instruction — ADD, STIR, BAKE — knows how to execute itself; a complex recipe is just a tree of these self-executing instructions."*

---

*End of Topic 23. Type "Next" to proceed to Topic 24: Iterator Pattern.*