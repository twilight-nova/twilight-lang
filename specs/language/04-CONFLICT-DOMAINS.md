# Twilight Language Conflict Domains Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies how Conflict Domains, the mechanism enabling safe parallel execution in the NOVA runtime, are represented and handled within the Twilight language and compiler. It defines the structure of domains, how they are identified, and the language features (primarily annotations) developers use to declare or refine domain information.

### 1.2 Definition

A **Conflict Domain** represents a statically identifiable scope of persistent blockchain state (e.g., contract storage, account balances) accessed by a transaction or contract function. The NOVA runtime uses Conflict Domains to determine whether operations can execute in parallel or conflict, requiring sequential ordering.

### 1.3 Core Principles

* **Static Identification:** Domains are primarily determined statically by the compiler (`twilightc`) through analysis of state access patterns.
* **Conflict Rules:** Access types (Read/Write) determine conflicts. Two operations conflict if they access the same domain (or one accesses a parent domain of the other) and at least one operation is a Write (Write/Write conflict, Read/Write conflict). Read/Read operations do not conflict.
* **Hashing:** For efficient processing, domains are represented by deterministic, fixed-size cryptographic hashes derived from a canonical string representation.

## 2. Conflict Domain Identification

### 2.1 Canonical Domain String Format

To ensure deterministic hashing, Conflict Domains are first represented as a canonical UTF-8 string before hashing:

`<ContractID>.<StateNamespace>[:<KeyExpression>]`

* **`<ContractID>`:** The canonical identifier for the contract instance, typically the lowercase hexadecimal address (e.g., `0xabc...def`). For native protocol components, a predefined identifier is used (e.g., `native.token`).
* **`<StateNamespace>`:** A case-sensitive identifier matching the declared name of the state variable, map, or logical storage area within the contract (e.g., `balance`, `allowance`, `config`).
* **`[:<KeyExpression>]` (Optional):** Specifies a particular key within a namespace, typically for maps or arrays.
    * The KeyExpression is represented canonically (e.g., lowercase address `0x123...`, integer literal `12345`, UTF-8 string `"key"`).
    * The compiler attempts to statically resolve variable keys used in expressions (e.g., map access using a function argument) where possible through alias analysis.
* **Wildcard (`*`):** A literal asterisk (`*`) in the `<KeyExpression>` position (e.g., `0xabc...def.user_data:*`) signifies access potentially affecting the entire namespace, conflicting with all specific accesses within it. Used when precise keys cannot be determined statically or for operations affecting the whole namespace.
* **Separators:** A period (`.`) separates ContractID and StateNamespace. A colon (`:`) separates StateNamespace and KeyExpression. No extraneous whitespace is included.

### 2.2 Domain Hashing

* **Algorithm:** Blake3 (default).
* **Output:** 32-byte (256-bit) hash.
* **Purpose:** The hash serves as the compact, efficiently comparable identifier used by the compiler's intermediate representation (TIR), the runtime scheduler, and the consensus mechanism.

*(Implementation details of hashing and TIR representation are in `specs/compiler/07-tir.md`)*.

## 3. Language Integration

Twilight employs a hybrid approach, combining compiler inference with optional developer annotations to determine the Conflict Domains accessed by functions.

### 3.1 Compiler Inference (Default Behavior)

* **Mechanism:** By default, the `twilightc` compiler performs static analysis (intra- and inter-procedural) on the code (TIR) to automatically infer the set of domains read and written by each function.
* **Process:**
    * Identifies direct persistent state accesses (e.g., `storage.variable`, map/vector accesses on storage variables).
    * Resolves keys used in accesses where possible using static analysis.
    * Calculates the canonical domain string and corresponding hash for each resolved access.
    * Aggregates these local domains with the domains inferred or declared for all called functions (transitively across the call graph).
* **Output:** Inferred `read_domains` and `write_domains` sets (of hashes) associated with each function.

### 3.2 Explicit Annotations (`#[reads]`, `#[writes]`, `#[conflict_free]`)

Developers can use annotations to explicitly declare domain usage, override inference, or resolve ambiguity.

* **Syntax:**
    ```twilight
    // Read access to specific user balance and entire allowance namespace
    #[reads("self.balance:user", "self.allowance:*")]
    fn check_allowance(user: address) { /* ... */ }

    // Write access to a specific log entry and config value
    #[writes("self.log:entry_id", "self.config")]
    fn update_log_and_config(entry_id: u64) { /* ... */ }

    // Declare both reads and writes
    #[reads("config.rate")]
    #[writes("balance:sender", "balance:recipient")]
    fn transfer_with_fee(sender: address, recipient: address, amount: u128) { /* ... */ }

    // Declare function has no persistent state side-effects
    #[conflict_free]
    fn pure_calculation(a: u64, b: u64) -> u64 { /* ... */ }
    ```
    * Annotations are placed directly before the `fn` definition.
    * String literals within `#[reads(...)]` and `#[writes(...)]` contain comma-separated canonical domain strings (or representations that the compiler resolves to canonical strings, e.g., using `self` to represent the current contract's ID). The exact syntax for specifying keys (like addresses or variables) within the annotation string needs precise definition in the grammar spec (`02-grammar.md`).

* **Semantics & Interaction with Inference:**
    * **Override:** If `#[reads]` or `#[writes]` annotations are present, they **override** the compiler's inference for that function. The declared domains are used as the authoritative set for that function when performing inter-procedural aggregation for its callers.
    * **Validation:** The compiler *should* still perform inference on annotated functions and issue warnings if the actual accesses found in the function body significantly diverge from the declared annotations (e.g., writing to a domain not listed in `#[writes]`, or declaring write access to a domain never touched). This helps catch inaccurate annotations.
    * **`#[conflict_free]`:** This annotation asserts that the function performs **no reads or writes** to persistent state domains. The compiler treats its domain sets as empty, skipping analysis for this function. This requires trust or should ideally be verified by other means (like static analysis proving purity).

### 3.3 Handling Dynamic State Access (Coarse Graining)

* **Challenge:** When state is accessed using a key expression that the compiler cannot statically resolve to a specific value or a limited set of values (e.g., `my_map[calculate_key_from_input(data)]`).
* **Compiler Behavior:**
    1.  **Detection:** The compiler identifies such dynamic key expressions during analysis.
    2.  **Fallback:** It falls back to inferring access to the entire namespace associated with the access, using the wildcard (`*`). For example, the access above would infer a read/write (depending on the operation) to the domain `self.my_map:*`.
    3.  **Warning:** The compiler MUST issue a warning to the developer indicating that a coarse-grained domain was inferred due to dynamic key access. The warning should recommend using explicit annotations (`#[reads]` / `#[writes]`) if a more precise domain is known and applicable, or if the coarse domain significantly impacts potential parallelism.

## 4. Metadata & Verification

### 4.1 Role of Compiler-Generated Metadata

* The primary output of the domain analysis (whether inferred or from annotations) is the aggregated `read_domains` and `write_domains` sets (of hashes) stored as metadata associated with each function in the compiled artifact's `.meta` file (Specification `specs/interfaces/10-meta-schema.md`).
* This metadata is consumed by the NOVA runtime scheduler and CRCS consensus layer.

### 4.2 Metadata Verification Requirement

* To prevent manipulation, the NOVA protocol (validators during consensus/mempool processing) MUST verify that the domain metadata provided with a transaction is consistent with the authoritative `.meta` file associated with the specific version (`code_hash`) of the contract code being executed.
* Transactions with incorrect or mismatched domain declarations MUST be rejected. The compiler is responsible for generating this correct metadata.