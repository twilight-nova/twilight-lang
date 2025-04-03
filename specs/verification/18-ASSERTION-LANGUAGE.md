# Twilight Assertion Language Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the syntax, scope, allowed expressions, and semantics of the assertion language embedded within Twilight. This language enables developers to formally specify properties of their code, such as preconditions, postconditions, and invariants. These specifications can then be automatically checked using formal verification tools (via Verification Condition Generation) or optionally executed as runtime checks during testing and debugging.

### 1.2 Target Audience

* Twilight Developers (writing specifications for their contracts)
* Compiler Implementers (`twilightc` parser, semantic analysis, VCGen, runtime assertion backend)
* Verification Tool Developers (consuming compiler-generated VCs)

### 1.3 Design Philosophy

* **Expressiveness:** Provide sufficient expressive power within assertion conditions to state meaningful properties about contract state and behavior.
* **Purity:** Restrict assertion condition expressions to a pure, side-effect-free subset of the Twilight language to ensure they do not alter program state and are suitable for logical translation.
* **Dual Semantics:** Define clear semantics for both logical interpretation (used by formal verification tools) and optional runtime execution (used for dynamic checking).
* **Integration:** Integrate assertion syntax naturally into the Twilight language.

## 2. Assertion Syntax and Scope

Twilight provides three primary assertion statements:

### 2.1 Syntax

* **Precondition:**
    ```twilight
    assert_pre(condition: bool_expr, message?: string_literal);
    ```
* **Postcondition:**
    ```twilight
    assert_post(condition: bool_expr, message?: string_literal);
    ```
* **Invariant:**
    ```twilight
    assert_invariant(condition: bool_expr, message?: string_literal);
    ```

Where:

* `condition`: A boolean expression conforming to the purity rules defined in Section 3.
* `message?`: An optional string literal providing context if the assertion fails at runtime. This message is ignored by static verification tools but used if runtime assertion checking is enabled.

### 2.2 Scope Rules (Placement Restrictions)

Assertion statements are only permitted in specific locations:

* `assert_pre`: Must appear only at the very beginning of a function body, before any other statements or declarations. Multiple `assert_pre` statements are permitted and are logically combined with AND (`&&`).
* `assert_post`: Must appear only at the end of a function body, conceptually evaluated immediately before any `return` statement (including implicit returns). It can access the function's return value (if any) and use the `old()` construct (See Section 3.5). Multiple `assert_post` statements are permitted and are logically combined with AND (`&&`).
* `assert_invariant`:
    * **Loop Invariant:** Must appear only as the first statement(s) immediately inside the body of a `loop` or `while` construct. It specifies a property that must hold at the beginning of *every* loop iteration.
    * **Contract Invariant:** Must appear only directly within a `contract { ... }` block, outside of any function or storage block. It specifies a property that must hold true at the end of the `init` constructor and before and after the execution of any public function of the contract.

## 3. Assertion Condition Expression Language

The `condition` expression within any `assert_*` statement must be pure and adhere to the following rules. The compiler MUST reject assertions containing disallowed constructs.

### 3.1 Purity Requirement

Assertion condition expressions must be free of side effects. They cannot modify contract storage, emit events, revert, panic, or call functions that perform these actions. Their evaluation must depend only on the program state (constants, immutable bindings, function arguments, storage state, results of pure function calls) and not on external factors.

### 3.2 Allowed Constructs

* **Literals:** Boolean literals (`true`, `false`), integer literals (all forms). String literals are only allowed for the optional `message` argument.
* **Constants:** References to `const` items defined globally or locally.
* **Immutable Bindings:** References to `val` bindings and immutable function parameters.
* **Storage Reads:** Accessing contract state via `storage.variable` or `storage.map[key_expr]` provided `key_expr` itself adheres to these purity rules.
* **Pure Operators:**
    * Boolean: `&&`, `||`, `!`
    * Comparison: `==`, `!=`, `<`, `>`, `<=`, `>=`
    * Arithmetic: `+`, `-`, `*`, `/`, `%` (Interpreted mathematically for verification, see Section 4.1; subject to runtime semantics if checked dynamically, see Section 4.2).
    * Bitwise: `&`, `|`, `^`, `<<`, `>>` (Right shift (`>>`) is logical for unsigned, arithmetic for signed), `!` (Bitwise not).
* **Pure Function Calls:** Calls to functions explicitly marked or statically inferred by the compiler as `pure` (See Section 3.4). Standard library functions operating on primitive types (e.g., numeric conversions if pure) are generally allowed.
* **Tuple/Struct Field Access:** Accessing fields of tuples or structs (e.g., `my_tuple.0`, `my_struct.field`) if the aggregate value itself is accessible according to these rules.
* **`old(expression)`:** Special construct allowed only within `assert_post` conditions (See Section 3.5).

### 3.3 Disallowed Constructs

The following are **NOT** permitted within assertion condition expressions:

* Assignments (`=`).
* Mutation of `var` bindings or mutable data structures.
* Calls to non-pure functions (including functions that write to storage, emit events, revert, panic, or call other non-pure functions/HFIs).
* `emit` statements.
* `require` statements.
* `panic!` calls.
* Control flow constructs (`if`, `loop`, `while`, `for`, `match`) *within* the condition expression itself.
* Creation of new complex objects (`vector`, `map`) that involve runtime allocation or potentially impure constructors.

### 3.4 Pure Functions

* **Definition:** A function is considered `pure` for assertion purposes if its execution is guaranteed **not** to:
    1.  Modify persistent storage (`state_write`).
    2.  Emit events (`emit`).
    3.  Revert or Panic (explicitly or implicitly, excluding potential mathematical domain errors handled by verification logic).
    4.  Call any other non-pure functions or HFI calls with side effects.
    5.  Mutate state external to its local scope (e.g., through mutable references, though Twilight's model restricts this).
* **Determination:** The `twilightc` compiler performs purity analysis (e.g., analyzing TIR for side-effecting instructions/calls) to determine if a function meets these criteria. Future versions might allow explicit `#[pure]` annotations.
* **Usage:** Only functions determined to be pure may be called within assertion condition expressions.

### 3.5 `old(expression)` Expression

* **Syntax:** `old(expr)`
* **Scope:** Allowed **only** within the `condition` expression of an `assert_post` statement.
* **Semantics:** Evaluates the inner `expr` (which must itself be a valid pure assertion expression, typically involving storage reads) in the state that existed at the **entry point** of the function containing the `assert_post`. This allows comparing the state before and after the function's execution.
    ```twilight
    // Example assert_post using old()
    assert_post(storage.counter == old(storage.counter) + 1);
    ```

## 4. Semantics

Assertion statements have dual semantics depending on the context (static verification vs. runtime checking).

### 4.1 Logical Semantics (for Static Verification / VCGen)

* **Purpose:** Defines the meaning used when translating assertions into Verification Conditions (VCs) for tools like SMT solvers.
* **Translation:** Assertion conditions are translated into logical predicates in a suitable formal logic (e.g., first-order logic with theories for arrays and bitvectors, as used in SMT-LIB 2).
* **Arithmetic:** Arithmetic operators (`+`, `-`, `*`, `/`, `%`) are typically interpreted using their unbounded mathematical semantics or the corresponding theories in the target logic (e.g., SMT BitVector arithmetic, which has defined wrapping behavior, or mathematical Integers). The goal is usually to prove functional correctness assuming no runtime overflows, or to prove the absence of overflows as a separate property. Runtime revert-on-overflow is generally *not* modeled directly in the core logical translation for functional VCs.
* **Proof Obligation:**
    * `assert_pre(P)` generates `FunctionEntryState => P_logical`.
    * `assert_post(Q)` generates `FunctionPrecondition => wp(FunctionBody, Q_logical)`, where `Q_logical` handles `old()` references to the entry state.
    * Loop `assert_invariant(I)` generates VCs for invariant establishment before the loop and preservation by the loop body.
    * Contract `assert_invariant(I)` generates VCs for establishment by `init` and preservation by public functions.

*(Details of VC Generation are in `specs/verification/19-vcgen.md`)*

### 4.2 Runtime Semantics (for Runtime Assertion Checks)

* **Purpose:** Defines the behavior when assertions are compiled into executable checks (enabled via a compiler flag like `--enable-runtime-asserts`).
* **Execution:** If enabled, the assertion's `condition` expression is evaluated using standard Twilight runtime semantics at the specified program point (function entry/exit, loop entry/exit).
* **Arithmetic:** Runtime evaluation uses the standard Twilight behavior (e.g., **revert-on-overflow** for `+`, `-`, `*`; panic on div/mod by zero).
* **`old()` Handling:** Requires runtime support to snapshot relevant state values at function entry and make them available for evaluation within `assert_post` checks just before function return.
* **Outcome:**
    * If `condition` evaluates to `true`, execution continues normally (with associated gas cost for evaluation).
    * If `condition` evaluates to `false`, execution halts immediately, the **transaction is reverted**, gas used up to the point of failure is consumed, and the optional `message` (or a default one) is returned as the revert reason. This behavior mirrors `require(false, message)`.
* **Disabled Mode:** If runtime assertion checks are disabled (default for release builds), `assert_*` statements generate no executable code and have zero runtime overhead.

*(Details of Runtime Assertion Check implementation are in `specs/verification/20-runtime-asserts.md`)*