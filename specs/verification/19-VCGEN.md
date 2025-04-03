# Twilight Verification Condition Generation Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the Verification Condition Generation (VCGen) process within the Twilight compiler (`twilightc`). VCGen automatically translates Twilight source code and its embedded formal assertions (`assert_pre`, `assert_post`, `assert_invariant`) into logical formulas, known as Verification Conditions (VCs). These VCs serve as proof obligations for external formal verification tools, primarily SMT solvers, enabling mathematical proof of specified code properties relative to the language's semantics.

### 1.2 Target Audience

* Compiler Implementers (`twilightc` middle-end, specifically the VCGen pass).
* Verification Tool Developers (consuming VCs, typically in SMT-LIB 2 format).
* Developers interested in the theoretical underpinnings of Twilight's verification support.

### 1.3 Design Philosophy

* **Soundness:** The translation from Twilight code and assertions into logical VCs must be sound with respect to Twilight's defined semantics. Proving the validity of all generated VCs implies that the original assertions hold for all possible executions under the specified preconditions and invariants.
* **Automation:** VC generation is a fully automated pass within the compiler.
* **Standard Output:** VCs are primarily output in the standard SMT-LIB 2 format for compatibility with widely available SMT solvers (e.g., Z3, CVC5).
* **Traceability:** Generated VCs are linked back to the original source code assertions via unique identifiers and metadata.

## 2. Verification Condition Generation (VCGen) Pass

### 2.1 Overview

* **Placement:** A pass within the `twilightc` middle end, typically running after core semantic analyses (Type Checking, Borrow Checking) and potentially after some semantics-preserving optimizations (`specs/compiler/08-middle-end-passes.md`).
* **Input:** Type-checked, borrow-checked, annotated Twilight Intermediate Representation (TIR-SSA), including assertion intrinsics derived from source `assert_*` statements (`specs/verification/18-assertion-language.md`). Requires access to the axiomatic semantics rules for TIR (`specs/language/01-core-language.md` implies these).
* **Output:** A collection of Verification Conditions (VCs) represented as logical formulas, associated metadata linking VCs to source locations, and typically serialized to SMT-LIB 2 format (`specs/verification/19-vcgen.md` details SMT-LIB output).

### 2.2 Core Algorithm: Weakest Precondition (WP) Calculus

* **Foundation:** The VCGen pass implements a Weakest Precondition (WP) calculus based on the axiomatic semantics defined for Twilight/TIR.
* **Definition:** `wp(Code, PostconditionQ)` computes the weakest logical predicate `P` such that if `P` holds before executing the TIR `Code`, then `PostconditionQ` is guaranteed to hold upon normal termination of `Code`.
* **Traversal:** The algorithm typically traverses the Control Flow Graph (CFG) of the TIR-SSA *backwards* from assertion points or function exit points, applying WP transformation rules for each instruction.

### 2.3 Logical State Representation

To perform WP calculation, the state of the Twilight program is modeled logically:

* **Environment (SSA Registers):** Each TIR SSA virtual register (`%reg`) is represented as a distinct logical variable in the generated formulas.
* **Persistent Storage (`storage`):** Modeled abstractly, typically using the SMT theory of arrays (`Array KeySort ValueSort`).
    * `KeySort`: Represents the type of storage keys (e.g., `(_ BitVec N)` for hashed keys).
    * `ValueSort`: Represents the type of stored values (e.g., `(_ BitVec 256)` for `u256`).
    * State reads map to `(select StorageMap key)`.
    * State writes map to `(store StorageMap key value)`, which represents the *new* state map after the write.
* **`old()` State:** For `assert_post` conditions requiring comparison with the state at function entry, the WP calculation maintains two logical state variables for storage: `PStore_entry` (state at entry) and `PStore_current` (state at the current point of backward analysis). `old(storage.x)` translates logically to `(select PStore_entry key_x)`.

### 2.4 WP Transformer Rules for TIR-SSA (Selected Examples)

The WP calculation applies rules based on the semantics of each TIR instruction. `Q` represents the postcondition predicate being propagated backwards. `Q[E/x]` denotes substituting expression `E` for all free occurrences of variable `x` in `Q`.

* **`wp(assign %x = Literal, Q)`:** `Q[LiteralValue / %x]`
* **`wp(assign %x = %y op %z, Q)`:** `Q[ (%y op_logical %z) / %x ]`
    * `op_logical` is the SMT equivalent (e.g., `bvadd`, `bvult`). Arithmetic is typically treated mathematically (unbounded or wrapping based on SMT theory) for functional verification. Proving absence of runtime *reverts* due to overflow requires separate, specific checks or modeling the reverting behavior directly.
* **`wp(state_write key_reg, val_reg, Q)`:** `Q[store(PStore_current, key_reg, val_reg) / PStore_current]`
    * The postcondition `Q` must hold in a state where `PStore_current` has been updated.
* **`wp(assign %r = state_read key_reg, Q)`:** `(forall ((v ValueSort)) (=> (= (select PStore_current key_reg) v) Q[v / %r]))`
    * For whatever value `v` is read, the postcondition `Q` (with `v` substituted for `%r`) must hold. Requires quantifier support in the SMT logic.
* **`wp(Instr1; Instr2, Q)`:** `wp(Instr1, wp(Instr2, Q))` (Sequential composition)
* **`wp(revert msg, Q)`:** `true` (A postcondition holds vacuously if execution reverts before reaching it).
* **`wp(panic msg, Q)`:** `true` (Similar to revert).
* **`wp(unreachable, Q)`:** `true` (Or `false` depending on the property being proved, usually `true` for partial correctness).

### 2.5 WP Calculation for Control Flow & Loops

* **Conditionals (`br_if %c, %L_true, %L_false`):**
    * Let `Q_after_if` be the predicate required after the conditional structure merges.
    * Let `wp_true = wp(BasicBlock_at_L_true, Q_after_if)`
    * Let `wp_false = wp(BasicBlock_at_L_false, Q_after_if)`
    * `wp(br_if..., Q_after_if) = (ite %c wp_true wp_false)` or equivalently `(%c => wp_true) && ((not %c) => wp_false)`.
* **Loops (`loop`, `while`):** Require Loop Invariants specified via `assert_invariant` inside the loop.
    * The WP calculation *assumes* the invariant `I` holds upon exiting the loop.
    * `wp(LoopConstruct, Q_after_loop) = I`.
    * Separate VCs are generated to prove the invariant itself (see Section 2.6).
* **Backward Traversal:** WP calculation propagates backwards through the CFG from assertion points or function exits, computing the weakest precondition required at the start of each basic block to ensure the desired postcondition holds at its end.

### 2.6 Generating VCs from Source Assertions

VCs are generated as logical implications based on the assertion type and the computed WP:

* **`assert_pre(P)` at function entry:**
    * VC: `FunctionPrecondition => P_logical`
    * Where `FunctionPrecondition` might be `true` or derived from contract invariants. `P_logical` is the logical translation of `P`.
* **`assert_post(Q)` at function exit:**
    * VC: `FunctionPrecondition => wp(FunctionBody, Q_logical)`
    * `Q_logical` translates `Q`, handling `old()` by referring to `PStore_entry`.
* **`assert_invariant(I)` (Loop Invariant):**
    * VC 1 (Establishment): `wp(Code_before_loop, I_logical)` (Must hold on loop entry). Assumes function precondition.
    * VC 2 (Preservation): `(I_logical && LoopCondition_logical) => wp(LoopBody, I_logical)` (Must be preserved by one iteration).
* **`assert_invariant(I)` (Contract Invariant):**
    * VC 1 (Establishment): `Precondition_init => wp(Body_init, I_logical)`.
    * VC 2 (Preservation): For each public function `F`, `(I_logical && Precondition_F) => wp(Body_F, I_logical)`.
* **Inline `assert(P)` (if supported):** At location `L`, `FunctionPrecondition => wp(Code_from_entry_to_L, P_logical)`.
* **VC Identifiers:** Each generated VC is assigned a unique ID (`verification_condition_id`) which is linked via metadata back to the source assertion and included in the `.meta` file.

## 3. SMT-LIB 2 Output Generation

The generated VCs are primarily formatted for consumption by SMT solvers using the SMT-LIB 2 standard.

### 3.1 Target Format & Configuration

* **Standard:** SMT-LIB Version 2.6.
* **Logic:** Typically `AUFBV` (Arrays, Uninterpreted Functions, BitVectors) is suitable. If quantifiers are generated (e.g., from `state_read`), a logic supporting them is required. The emitter selects the appropriate logic.
* **Options:** Standard options like `(set-option :produce-models true)` are included.

### 3.2 File Structure

A typical `.smt2` file includes:
1.  Header comments (generator version, source file).
2.  `(set-logic ...)` command.
3.  `(set-option ...)` commands.
4.  Type declarations (`declare-sort`) if needed for abstract types.
5.  Variable declarations (`declare-fun`) for all logical variables (SSA regs, state maps, parameters).
6.  Assertion commands (`(assert VC_Formula)`) for each generated VC.
7.  Final commands (`(check-sat)`, optionally `(get-model)`, `(exit)`).

### 3.3 Type Mapping (TIR -> SMT Sorts)

* `bool` -> `Bool`
* `uN` / `iN` -> `(_ BitVec N)` (e.g., `u64` -> `(_ BitVec 64)`)
* `address` -> `(_ BitVec 160)` (assuming 20 bytes)
* `hash256` -> `(_ BitVec 256)`
* Storage (`map<K,V>`) -> `(Array KeySort ValueSort)` (e.g., `(Array (_ BitVec 160) (_ BitVec 256))`)
* Other types (structs, opaque types) may be bitvectorized, modeled using uninterpreted sorts, or abstracted depending on the verification context.

### 3.4 Variable Declaration

* Logical variables are declared using `(declare-fun <variable_name> () <Sort>)`.

### 3.5 Expression & Operation Mapping

Logical formulas derived from WP calculation are mapped to SMT-LIB 2 functions:

* Boolean ops: `and`, `or`, `not`, `=>`, `ite`
* Comparisons: `=`, `distinct`, `bvult`, `bvslt`, etc.
* BitVector Arithmetic: `bvadd`, `bvsub`, `bvmul`, `bvudiv`, `bvsdiv`, etc.
* BitVector Bitwise: `bvand`, `bvor`, `bvxor`, `bvnot`, `bvshl`, `bvlshr`, `bvashr`
* Array Theory: `select`, `store`
* Quantifiers: `forall`, `exists` (require appropriate logic)

### 3.6 Assertion Command

* Each VC formula `F` is emitted as `(assert F)`.

### 3.7 Provenance Comments

* Each `(assert ...)` command is preceded by SMT comments (` ;; `) indicating the VC ID, source location, and kind of the original assertion.

## 4. Tooling Workflow

1.  Developer adds assertions (`assert_*`) to Twilight code.
2.  `twilightc` (with verification enabled) invokes the VCGen pass.
3.  VCGen calculates VCs using WP and emits them to an `.smt2` file(s).
4.  Developer or CI process feeds the `.smt2` file to an SMT solver (e.g., `z3 file.smt2`).
5.  Solver output (`sat`, `unsat`, `unknown`) indicates whether assertions *might* fail (`sat` - counterexample potentially available), provably hold (`unsat`), or could not be determined.
6.  Tooling interprets solver output, reporting results back to the developer, linked to original source assertions via metadata.