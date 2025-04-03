# Twilight Compiler WASM Backend Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the process, rules, and conventions for the Twilight compiler (`twilightc`) backend responsible for translating the optimized Twilight Intermediate Representation (TIR-SSA) into executable WebAssembly (WASM) bytecode. This WASM code is designed to run within the secure, deterministic, and gas-metered NOVA runtime environment.

### 1.2 Scope

This specification covers the target WASM environment assumptions, the mapping from TIR instructions and types to WASM constructs, Application Binary Interface (ABI) considerations, interaction with the Host Function Interface (HFI), and implementation guidance for the WASM code generation phase. It assumes the input is validated, optimized TIR-SSA.

### 1.3 Input/Output

* **Input:** Optimized, validated TIR-SSA representation (`specs/compiler/07-tir.md`). Host Function Interface (HFI) specification (`specs/interfaces/11-hfi.md`). Target WASM features and runtime configuration (`specs/interfaces/11-hfi.md` references).
* **Output:** A valid `.wasm` binary file adhering to the specified WASM standard and the NOVA HFI ABI, ready for deployment and execution on the NOVA blockchain.

## 2. Target WASM Environment

* **WASM Standard:** The backend targets the WebAssembly MVP (Minimum Viable Product) standard, plus any specific approved proposals required by the language or standard library implementation and supported deterministically by the target NOVA runtime (e.g., potentially `multi_value`). Features like `threads` and `simd` are explicitly disallowed.
* **Runtime Assumptions:** Code generated targets the NOVA runtime environment, which utilizes a specific WASM engine (e.g., Wasmtime) configured for:
    * Strict Determinism.
    * Gas Metering (Fuel Injection or Instruction Counting).
    * Strong Sandboxing (Memory Isolation, No direct OS/IO access).
    * Defined Host Function Interface (HFI) for all blockchain interaction.
* **Memory Model:** Assumes a single linear memory space per WASM instance, managed by the runtime. The backend is responsible for managing layout and allocations within this space for its stack, potentially large value passing, and HFI buffer interactions. Memory growth (`memory.grow`) is disallowed by default or strictly controlled and gas-costed by the runtime.

## 3. TIR-SSA to WASM Lowering

This section details the translation from canonical TIR-SSA instructions to WASM instructions and control flow.

### 3.1 Function Definition & Signature Mapping

* **TIR `define`:** Lowered to a WASM `func` section entry.
* **Visibility:** Functions marked `public` in TIR are exported from the WASM module using their mangled names.
* **Signature:** TIR function signatures (`<ReturnType> @Name(<ArgType1> %arg1, ...)` ) are translated into WASM function signatures (`(param $arg1 WasmType1 ...) (result WasmTypeN?)`). Type mapping follows Section 4. Arguments and return values are passed according to the ABI defined in Section 5.
* **WASM Locals:** Necessary WASM local variables (`local`) are declared for storing intermediate SSA values that cannot live solely on the WASM value stack, particularly across basic block boundaries or complex control flow.

### 3.2 Basic Block & Control Flow Mapping

* **TIR Basic Blocks (`%Label:`):** Map to sequences of WASM instructions. TIR labels (`%Label`) become targets for WASM branching instructions.
* **WASM Control Flow:** TIR's explicit CFG (Terminators, Labels) is translated into nested WASM control flow constructs: `block`, `loop`, `if`/`else`/`end`, `br`, `br_if`, `br_table`.
* **Terminators:**
    * `ret <Type> %value` / `ret void`: Load the TIR `%value` (if any) onto the WASM value stack according to ABI. Execute WASM `return`.
    * `br %DestLabel`: Execute WASM `br $WasmLabel`, where `$WasmLabel` corresponds to the WASM block/loop target for `%DestLabel`.
    * `br_if %condition, %TrueLabel, %FalseLabel`: Load the `bool` TIR `%condition` onto the WASM stack. Use WASM `if`/`else`/`end` blocks or `br_if $WasmTrueLabel` instructions to branch to the appropriate WASM labels corresponding to `%TrueLabel` and `%FalseLabel`. Stack management must ensure value consistency across branches.
    * `revert <Type> %msg`:
        1.  Generate WASM code to load the pointer (i32 offset) and length (i32) of the TIR `%msg` variable (which must reside in linear memory) onto the stack.
        2.  Generate `call $nova_contract_revert` (HFI function import).
        3.  Generate WASM `unreachable` immediately after the call (as `revert` does not return).
    * `panic <Type> %msg` / `unreachable`: Generate WASM `unreachable`. The NOVA runtime is configured to trap on `unreachable` and map this to the Panic transaction outcome (revert state, consume all gas).

### 3.3 Instruction Lowering

#### 3.3.1 Arithmetic/Bitwise/Comparison (Incl. Overflow Checks)

* **Mapping:** TIR arithmetic (`add`, `sub`, `mul`, `div_[u|s]`, `rem_[u|s]`), bitwise (`and`, `or`, `xor`, `shl`, `shr_[l|a]`, `not`), and comparison (`icmp`) instructions operating on `i32`/`i64`-compatible TIR types map directly to their corresponding WASM instructions (e.g., `i64.add`, `i32.xor`, `i64.div_u`, `i64.lt_s`).
* **Overflow/Underflow Checks (Critical):** For standard TIR arithmetic ops (`add`, `sub`, `mul`, `neg`) that default to revert-on-overflow:
    1.  Generate WASM code to perform the operation speculatively or check operands beforehand.
    2.  Generate WASM code to explicitly check the result for overflow/underflow conditions according to two's complement arithmetic.
    3.  If an overflow/underflow is detected, generate a branch to a sequence that calls `nova_contract::revert` with a standard overflow message (similar to `revert` lowering above).
    4.  If no overflow, proceed with the result.
* **Checked/Wrapping/Saturating Ops:** Lower `checked_*` ops to sequences using standard WASM ops followed by checks that produce the `Option` representation. Lower `wrapping_*` and `saturating_*` ops to sequences using standard WASM ops that implement the defined wrapping or clamping logic. These do not include the implicit revert checks.
* **Division/Modulo by Zero:** Map `div_[u|s]` and `rem_[u|s]` directly. WASM standards cause a trap on division/remainder by zero. The runtime maps this trap to the Panic outcome.

#### 3.3.2 SSA Phi Node Elimination

* `phi` instructions are not directly represented in WASM. The backend eliminates them during CFG lowering.
* The code generation for predecessor blocks ensures that the correct value (corresponding to that predecessor path) is loaded onto the WASM value stack or into the correct WASM local variable before branching to the block that originally contained the `phi`.

#### 3.3.3 Select Instruction

* `%result = select bool %cond, <Type> %true_val, <Type> %false_val`: Load `%cond` (i32), `%true_val` (WasmType), and `%false_val` (WasmType) onto the WASM stack. Execute the WASM `select` instruction.

#### 3.3.4 State Access Instructions (HFI Call Generation)

* `state_read <Type> %key_reg` / `state_write <Type> %key_reg, %value_reg`:
    1.  Generate code to prepare arguments in WASM linear memory according to HFI specification (`specs/interfaces/11-hfi.md`), including domain hash, key, and value/output buffers.
    2.  Obtain pointers (i32 offsets) and lengths (i32 counts) for these memory buffers.
    3.  Load necessary pointer/length arguments onto the WASM stack.
    4.  Generate `call $nova_state_read` or `call $nova_state_write` targeting the corresponding HFI import.
    5.  Generate code to handle the `i32` return code from the HFI call (check for errors, potentially retrieve bytes read count for `state_read`).

#### 3.3.5 Function Call Instructions (Internal & HFI)

* `call <RetType> @MangledFunctionName(...)` (Internal Static Call):
    1.  Load arguments onto the WASM stack according to the ABI (Section 5).
    2.  Generate WASM `call $MangledFunctionName` targeting the WASM function corresponding to the TIR function.
    3.  If function returns a value, it will be left on the WASM stack; store it to a local or use it as needed.
* HFI Calls (General):
    1.  Prepare arguments, potentially marshalling data into linear memory and obtaining pointers/lengths.
    2.  Load direct pointer/length/value arguments onto the WASM stack.
    3.  Generate `call $hfi_module::function_name` targeting the specific HFI import.
    4.  Handle return values/codes according to the HFI specification.

#### 3.3.6 Type Conversion Instructions

* `zext`, `sext`, `trunc`: Map directly to corresponding WASM integer conversion instructions (`i64.extend_i32_u`, `i32.wrap_i64`, `i64.extend32_s`, etc.) based on source and target TIR types.
* `bitcast`: Lowered to WASM `iXX.reinterpret_fXX` or `fXX.reinterpret_iXX` if floating point types were involved (disallowed) or memory reinterpretation if sizes match (use highly restricted).

#### 3.3.7 Aggregate Type Operations

* `extractvalue <AggType> %agg_val, <index>` / `insertvalue <AggType> %agg_val, %elem_val, <index>`: Lowering depends on the ABI for aggregate types (Section 5).
    * If aggregates are passed/represented as packed values in WASM locals or on the stack (for small aggregates): Use sequences of WASM bit shifts, masks, or stack manipulation instructions.
    * If aggregates are passed/represented by reference (pointer `i32`): Lower to WASM memory instructions (`i32.load`, `i64.load`, `i32.store`, etc.) using base pointers and calculated offsets based on field `index` and type layout definitions. `insertvalue` requires copying the aggregate before storing the modified element to maintain immutability semantics if the source `%agg_val` is still live.

#### 3.3.8 NOVA Intrinsic Instructions (HFI Call Generation)

* `emit_event`, `get_ctx`, `crypto_hash`, assertion intrinsics: Translate directly into the corresponding WASM `call $hfi_module::function_name` instructions. Prepare arguments (pointers/lengths for buffers, direct values for simple args) and handle results as defined in the HFI specification (`specs/interfaces/11-hfi.md`).

### 3.4 Register Allocation / Stackification

* **Goal:** Map the infinite virtual registers (`%reg`) of TIR-SSA onto WASM's combination of a value stack and a finite set of typed local variables (`local`).
* **Process:** Employ standard register allocation techniques adapted for a stack machine target.
    * Values are primarily computed and consumed on the value stack.
    * Use `local.get`, `local.set`, `local.tee` to store/retrieve values in WASM local variables when they need to persist across stack operations, basic block boundaries, or complex control flow constructs.
    * Algorithms like linear scan or graph coloring (applied conceptually) can inform which TIR registers are best mapped to WASM locals versus kept transiently on the stack.

## 4. Type Mapping (TIR -> WASM)

This defines the mapping from TIR types to WASM value types (`i32`, `i64`) or how they are handled via memory.

* `bool`, `u8`..`u32`, `i8`..`i32`: -> `i32`
* `u64`, `i64`, `timestamp`, `blocknumber`: -> `i64`
* `u128`, `i128`, `u256`, `i256`: Not mapped directly to WASM value types. Handled via pointers (`i32`) to representations in linear memory. Calculations occur via standard library or HFI calls.
* `address`, `hash256`: Handled via pointers (`i32`) to representations in linear memory.
* `string`, `vector<T>`, `map<K,V>`: Handled via opaque handles/pointers (`i32`) managed by the standard library and runtime allocator.
* `array<T, N>`, `struct`, `tuple`: Depends on ABI (Section 5). Small/simple aggregates might be passed/returned as multiple `i32`/`i64` values. Larger aggregates are handled via pointers (`i32`) to representations in linear memory.
* `ptr<T>`: -> `i32` (representing a byte offset in linear memory).

## 5. Memory Management & Application Binary Interface (ABI)

* **Stack Pointer:** The backend manages allocation within the WASM linear memory for the execution stack (function call frames, argument passing buffers, potential register spills), typically using a designated WASM global variable or a reserved memory region.
* **ABI Conventions:** A stable ABI must be defined for:
    * **Argument Passing:** How function arguments (especially aggregates, `string`s, collections, large integers) are passed - by value (multiple `i32`/`i64`s on stack/locals if small) or by pointer (`i32`) to linear memory.
    * **Return Values:** How function results (especially aggregates) are returned - by value on stack or via a caller-provided output pointer (`i32`) passed as an implicit argument.
    * **Memory Layout:** Define the memory layout (size, alignment, field offsets) for structs and tuples when passed or stored indirectly via pointers.
* **Allocation:** WASM code does not directly call `malloc`/`free`. Dynamic memory required by collections (`vector`, `map`) or `string` operations is managed via calls to the standard library, which in turn uses runtime-provided allocator HFI functions.

## 6. Implementation Guidance & Testing

* **WASM Generation Library:** Utilize a robust library for programmatically generating WASM bytecode (e.g., `wasm-encoder`, `walrus`, `cranelift-wasm`).
* **TIR Traversal:** Implement lowering logic using visitors or linear scans over the optimized TIR-SSA.
* **Stackification:** Implement logic to manage the mapping from SSA registers to the WASM value stack and local variables.
* **Control Flow:** Carefully translate TIR CFG constructs into valid nested WASM `block`/`loop`/`if` structures.
* **HFI Integration:** Ensure correct marshalling of arguments and handling of return codes for all HFI calls according to the HFI specification.
* **Safety Checks:** Reliably inject required runtime checks (e.g., overflow checks, potentially bounds checks if not handled by WASM traps).
* **Testing:**
    * Generate WASM for a wide range of Twilight/TIR constructs.
    * Validate generated WASM using standard tools (`wasm-validate`).
    * Execute generated WASM in the target NOVA runtime environment (initially with mock HFI) and verify semantic equivalence with the source TIR/Twilight code using differential testing.
    * Utilize WASM spec tests where applicable. Test edge cases for ABI, HFI interactions, and safety checks.