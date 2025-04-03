# Twilight Runtime Assertion Checking Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the implementation mechanism for the optional Runtime Assertion Checking mode in Twilight. When enabled, this mode compiles formal assertions (`assert_pre`, `assert_post`, `assert_invariant`) present in the source code into executable runtime checks within the target bytecode (e.g., WASM). This provides a means for dynamic validation of specifications during testing and debugging.

### 1.2 Target Audience

* Compiler Implementers (`twilightc` backend)
* NOVA Runtime/VM Implementers (supporting `old()` state snapshotting)
* Twilight Developers (understanding the feature for testing/debugging)

### 1.3 Design Philosophy

* **Dynamic Validation:** Offer a mechanism to detect violations of specified properties during concrete program executions, complementing static verification techniques.
* **Testing & Debugging Aid:** Provide immediate feedback by halting execution upon assertion failure, helping developers pinpoint the cause of specification violations.
* **Explicit Activation:** Runtime checks impose performance and gas overhead and MUST be explicitly enabled via compiler flags; they are disabled by default in production builds.
* **Consistent Failure Behavior:** Failed runtime assertion checks MUST behave identically to a standard `require(false, ...)` failure, resulting in a transaction revert.

## 2. Activation Mechanism

* **Compiler Flag:** Runtime assertion checking is disabled by default. Compilation of assertions into runtime checks is activated by passing a specific flag to `twilightc` during compilation (e.g., `--enable-runtime-asserts` or an equivalent configuration setting like `--cfg debug_assertions=true`).
* **Build Profiles:** Standard release build configurations (e.g., activated by `twilightc build --release`) MUST have runtime assertion checking disabled by default to minimize gas costs and overhead in production environments. Debug build configurations may enable it by default or require the explicit flag.

## 3. Compiler Backend Implementation

### 3.1 Conditional Code Generation

The compiler backend (e.g., TIR-to-WASM) MUST check if the runtime assertion checking flag is enabled before generating code for assertion statements found in the TIR (which correspond to `assert_*` in the source). If the flag is *not* set, these assertion points generate no executable code.

### 3.2 Lowering Assertion Statements

When runtime assertion checking is enabled, assertion points in TIR are lowered into executable code as follows:

1.  **Evaluate Condition:** Generate target bytecode (e.g., WASM) to evaluate the assertion's boolean `condition` expression at runtime. This evaluation uses standard Twilight runtime semantics, including default revert-on-overflow for arithmetic operations. Gas is consumed for this evaluation.
2.  **Conditional Check:** Generate code to check the boolean result of the condition evaluation.
3.  **Trigger Revert (on False):** If the condition evaluates to `false`:
    * Generate code to immediately halt execution and trigger a transaction revert.
    * This is typically achieved by lowering to a TIR `revert` instruction or directly generating the backend code equivalent (e.g., preparing the message buffer and generating a `call` to the `nova_contract::revert` HFI function).
    * The revert message used is the optional `message` string provided in the source assertion or a default message indicating the assertion type and source location (e.g., `"Assertion failed: Precondition at counter.twl:10:5"`) if no message was provided.

### 3.3 Placement of Runtime Checks

The generated runtime check code MUST be inserted at the semantically correct point in the executable code's control flow graph:

* **`assert_pre`:** Check inserted at the beginning of the function's entry block, before any other user-written code executes.
* **`assert_post`:** Check inserted immediately before *every* `ret` (return) instruction within the function's body. Requires special handling for `old()` expressions (See Section 4.2).
* **`assert_invariant` (Loop):** Check inserted at the loop header (conceptually, before the loop condition is checked for the first and subsequent iterations) AND immediately before the loop's back-edge branch (i.e., after the loop body finishes execution for an iteration, before branching back to the header).
* **`assert_invariant` (Contract):** Check inserted at the very end of the `init` function body (before `ret`). Check also inserted at the beginning of *every* public function body and immediately before *every* `ret` instruction within those public functions.

## 4. Runtime Support Requirements

### 4.1 Revert Handling Consistency

The NOVA runtime environment MUST handle reverts triggered by failed runtime assertion checks identically to how it handles reverts triggered by `require(false, ...)`:

* Execution halts immediately.
* All state changes made by the transaction are discarded (reverted).
* Gas is consumed only up to the point of the failed assertion check.
* The transaction outcome is recorded as `Reverted`, storing the provided assertion failure message.

### 4.2 `old()` State Snapshotting (Conditional)

Evaluating `assert_post` conditions containing `old(expression)` requires specific runtime support, which MUST ONLY be active when runtime assertion checking is enabled for the contract code being executed.

* **Trigger:** The runtime needs a mechanism to know when it enters a function that contains `assert_post` conditions using `old()`. This might involve the compiler embedding specific metadata in the bytecode or requiring a specific (conditionally compiled) HFI call at the function's entry.
* **Snapshotting:** Upon entering such a function (and only if runtime checks are enabled), the runtime environment MUST snapshot the current values of the specific persistent storage locations referenced by `old()` expressions within *that function's* `assert_post` conditions. This snapshot is stored temporarily, associated with the current execution context.
* **Accessibility:** When the generated runtime check code for an `assert_post` executes (just before a `ret`), it needs access to the corresponding snapshotted value(s). This likely requires a dedicated HFI function (e.g., `nova_debug::get_old_value(key_ptr, key_len, value_out_ptr) -> i32`) which is only functional when runtime assertion checking is enabled. This HFI retrieves the previously snapshotted value for the given storage key.
* **Overhead:** Snapshotting (reading state at entry) and subsequent retrieval (HFI call) introduces runtime overhead and gas costs *only* when this feature is active.
* **Cleanup:** The temporary snapshot associated with a function call must be discarded when the function execution completes (returns or reverts).

## 5. Gas Costs

When runtime assertion checking is enabled:

* **Condition Evaluation:** Evaluating the `condition` expression consumes gas according to standard execution costs.
* **Check Overhead:** A small base gas cost might be associated with the conditional check instruction itself.
* **Revert Cost:** If an assertion fails, the standard gas cost associated with processing a transaction revert applies (in addition to gas consumed up to that point).
* **`old()` Snapshotting:** Reading state for snapshotting at function entry consumes gas. The dedicated HFI call (e.g., `nova_debug::get_old_value`) used to retrieve snapshotted values during `assert_post` checks also consumes gas.

## 6. Usage Context and Limitations

* **Intended Use:** Runtime assertion checking is primarily intended for development, testing (unit, integration, fuzzing), debugging, and potentially deployments to non-production testnets or staging environments. It helps dynamically catch violations of specified properties.
* **Production Deployments:** This mode **MUST be disabled** for production mainnet deployments. Production code should rely on static verification results, thorough testing, audits, and the core language safety guarantees. Enabling runtime checks in production introduces unnecessary gas overhead and the risk of legitimate transactions reverting due to assertion failures (which might represent specification bugs or rare edge cases rather than critical protocol violations suitable for `require`).