# Twilight Language Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the core features, syntax, type system, and semantics of the `twilight-native` syntax mode for the Twilight programming language. Twilight is designed for developing safe, verifiable, and high-performance smart contracts, particularly for the NOVA blockchain environment which leverages conflict domains for parallel execution.

### 1.2 Scope

This specification defines the fundamental building blocks of the language available to developers. It covers primitive and compound types, core syntax for expressions and control flow, the memory and ownership model, error handling mechanisms, and essential blockchain integration features. Detailed grammar, standard library APIs, and advanced features are specified in separate documents.

### 1.3 Design Philosophy

* **Safety:** Provide strong static guarantees against common errors like null references, type mismatches, and memory safety issues (managed implicitly by the compiler). Defaults prioritize safety (e.g., revert on integer overflow).
* **Clarity & Readability:** Employ clean syntax with minimal boilerplate, leveraging type inference where appropriate.
* **Expressiveness:** Offer sufficient language constructs (types, control flow, generics, traits) for effective smart contract development.
* **Domain Integration:** Treat blockchain concepts (contracts, storage, events, context, conflict domains) as first-class language features.
* **Determinism:** Ensure all specified language operations are fully deterministic for consensus compatibility.

## 2. Lexical Structure

Twilight source code is composed of keywords, identifiers, literals, operators, and punctuation, defined precisely in the Grammar Specification. Code is case-sensitive. Whitespace is generally insignificant except for token separation. Comments (`//` single-line, `/* ... */` multi-line) are ignored, while doc comments (`///`) are associated with subsequent items for documentation generation.

## 3. Core Syntax Constructs

### 3.1 Variable Bindings & Constants

Twilight provides three ways to bind names to values:

* **Immutable Bindings (`val`):**
    * Syntax: `val binding_name: Type = expression;` or `val binding_name = expression;` (type inferred).
    * Semantics: Creates a binding that cannot be reassigned. For compound types, this enforces deep immutability by default through this binding – the contents cannot be mutated via this binding.
    * Example:
        ```twilight
        val count: u64 = 10;
        val inferred_name = "Alice"; // Type 'string' inferred
        ```

* **Mutable Bindings (`var`):**
    * Syntax: `var binding_name: Type = expression;` or `var binding_name = expression;` (type inferred).
    * Semantics: Creates a binding that can be reassigned (`binding_name = new_expression;`). If the binding holds a mutable compound type (like `vector` or a `struct` with public mutable fields), its contents can be mutated through this binding.
    * Example:
        ```twilight
        var mutable_counter = 100; // Type inferred
        mutable_counter = 101;
        ```

* **Constants (`const`):**
    * Syntax: `const CONSTANT_NAME: Type = constant_expression;`
    * Semantics: Defines true compile-time constants. The value must be evaluatable at compile time. Constants are implicitly static and have no fixed memory location. Requires an explicit type annotation.
    * Example:
        ```twilight
        const MAX_SUPPLY: u256 = 1_000_000_000u256;
        ```

### 3.2 Functions

* **Definition:** Defined using the `fn` keyword. Requires explicit type annotations for all parameters and the return type.
    ```twilight
    fn calculate_fee(amount: u128, rate_bps: u16) -> u128 {
        // Function body
        let fee = amount * (rate_bps as u128) / 10000u128;
        fee // Implicit return of the last expression's value
    }

    fn check_status(id: u64) -> bool {
        if id == 0 {
            return false; // Explicit return keyword
        }
        // ... more logic ...
        true
    }
    ```
* **Return Values:** Functions return the value of the last expression in their body unless it's terminated by a semicolon. The `return` keyword allows explicit returns from any point within the function. Functions declared with `-> ()` or no return type annotation implicitly return the unit type `()`.

### 3.3 Control Flow Structures

* **`if`/`else` Expression:** Conditional execution based on a boolean condition. `if` is an expression, meaning it evaluates to a value. Both branches must evaluate to the same type.
    ```twilight
    val is_active = true;
    val status_code = if is_active { 1 } else { 0 }; // status_code is 1
    ```
* **`loop` Expression:** Creates an infinite loop that must be explicitly exited using `break`. The `continue` keyword skips the rest of the current iteration and proceeds to the next. A `loop` can evaluate to a value by providing it to `break`.
    ```twilight
    var counter = 0;
    val result = loop {
        counter = counter + 1;
        if counter >= 5 {
            break counter * 10; // Exits the loop, loop expression evaluates to 50
        }
    }; // result is 50
    ```
* **`while` Loop:** Executes the loop body as long as the condition expression evaluates to `true`.
    ```twilight
    var remaining = 3;
    while remaining > 0 {
        // do something
        remaining = remaining - 1;
    }
    ```
* **`for` Loop:** Iterates over values produced by an iterator expression (details of iterators are defined by the Standard Library and collection types).
    ```twilight
    // Conceptual example assumes 'my_vector' is iterable
    // var sum = 0;
    // for value in my_vector {
    //     sum = sum + value;
    // }
    ```
* **`match` Expression:** Performs pattern matching against a value. Branches are evaluated based on matching patterns. Matches must be exhaustive (cover all possible cases for the type being matched) unless a wildcard pattern (`_`) is used.
    ```twilight
    enum Action { Increment, Decrement(u64) }
    val action = Action::Decrement(5);
    match action {
        Action::Increment => print("Incrementing"),
        Action::Decrement(amount) => print("Decrementing by {}", amount),
        // No '_' needed if all variants are covered
    }
    ```

## 4. Type System

Twilight implements a static, strong type system designed for safety and expressiveness in the blockchain context.

### 4.1 Primitive Types

The language provides a set of built-in primitive types:

* **Boolean:** `bool` representing logical values `true` and `false`.
* **Unsigned Integers:** `u8`, `u16`, `u32`, `u64`, `u128`, `u256`. Represent non-negative whole numbers.
* **Signed Integers:** `i8`, `i16`, `i32`, `i64`, `i128`, `i256`. Represent positive and negative whole numbers using two's complement representation.
* **String:** `string`. Represents an immutable sequence of UTF-8 encoded characters.
* **Address:** `address`. An opaque type representing a blockchain address specific to the NOVA protocol. Its underlying size (e.g., 20 or 32 bytes) is determined by the protocol.
* **Hash256:** `hash256`. An opaque type representing a 32-byte cryptographic hash value.
* **Timestamp:** `timestamp`. Represents a point in time, implemented as an alias for `u64` (typically Unix timestamp seconds).
* **Block Number:** `blocknumber`. Represents a block height, implemented as an alias for `u64`.

### 4.2 Compound Types

Types composed from other types:

* **Structs:** User-defined aggregate types with named fields. Provide a way to group related data. Structs use nominal typing (type name matters, not just structure).
    ```twilight
    struct UserProfile {
        id: u64;
        name: string;
        is_verified: bool;
    }
    ```
* **Enums:** Algebraic data types (tagged unions) allowing a value to be one of several defined variants. Variants can optionally hold associated data. Enums use nominal typing.
    ```twilight
    enum ServerResponse {
        Success(string),
        Error { code: u32, message: string },
        Pending,
    }
    ```
* **Tuples:** Anonymous, ordered, fixed-size sequences of elements, potentially of different types. Accessed via zero-based numeric index (e.g., `my_tuple.0`).
    ```twilight
    val coordinates: (i64, i64, string) = (10, -5, "origin");
    let x = coordinates.0; // x is 10
    ```
* **Arrays:** Fixed-size, contiguous collections of elements of the same type. The size `N` must be a compile-time constant expression. Accessed via zero-based index `array[index]`.
    ```twilight
    // An array of 32 bytes, initialized to zero
    val key_material: array<u8, 32> = [0; 32];
    let first_byte = key_material[0];
    ```
* **Vectors:** Dynamically sized, growable list collections of elements of the same type. Provide methods like `push`, `pop`, `len`. Accessed via index `vector[index]`. Details provided by the Standard Library.
    ```twilight
    var user_ids: vector<u64> = []; // Creates an empty vector
    user_ids.push(101);
    ```
* **Maps:** Associative key-value collections (hash maps). Keys must implement required hashing and equality traits. Provide methods like `insert`, `get`, `remove`, `len`. Lookups for potentially non-existent keys return `Option<ValueType>`. Details provided by the Standard Library.
    ```twilight
    var balances: map<address, u128>; // Declares a map
    balances.insert(user_addr, 500);
    let maybe_balance = balances.get(another_addr); // Returns Option<u128>
    ```

### 4.3 Safety Types

These enum types are fundamental for writing safe code by making potential absence or failure explicit in the type system. They are typically available via the prelude.

* **`Option<T>`:** Represents a value that may or may not be present.
    * Variants: `Some(T)` (contains a value of type `T`) and `None` (represents absence).
    * Usage: Returned by operations that might not yield a result (e.g., map lookups, safe indexing) to avoid null-like errors. Requires explicit handling (e.g., `match`, standard library methods).
* **`Result<T, E>`:** Represents a result that can either be success (`Ok(T)`) or failure (`Err(E)`).
    * Variants: `Ok(T)` (contains the success value of type `T`) and `Err(E)` (contains the error value of type `E`).
    * Usage: Standard mechanism for functions to signal recoverable errors. Forces callers to acknowledge and handle potential failures.

### 4.4 Type Inference & Aliasing

* **Inference:** The compiler infers the types of `val` and `var` bindings from their initializer expressions when possible.
* **Annotations Required:** Explicit type annotations are mandatory for function parameters, function return types, `const` items, struct fields, and enum variant payloads.
* **Type Aliases:** The `type` keyword creates a new name (alias) for an existing type, which can improve readability. Aliases do not create distinct types; they are synonyms.
    ```twilight
    type Wei = u256;
    type Balances = map<address, Wei>;
    ```

### 4.5 Type Conversion Rules

* **No Implicit Coercion:** The language does not perform automatic type conversions between different types (e.g., numeric types of different sizes, integers to strings).
* **Explicit Conversion:** Conversion must be invoked explicitly using methods provided by the standard library. Common patterns include:
    * `.into()`: For infallible conversions (implemented via traits).
    * `T::from(value)`: Constructor pattern, often infallible.
    * `T::try_from(value) -> Result<T, Error>`: For fallible conversions that might fail (e.g., value out of range).
    * `.to_string()`: For converting values to string representation (requires specific trait support).
* **Casting (`as`):** The `as` keyword is generally not used for standard type conversions to avoid ambiguity and potential unsafety associated with its behavior in other languages. Its use, if any, would be highly restricted to specific low-level scenarios (likely disallowed in contract code).

### 4.6 Literal Representation

* **Integers:** Standard decimal (`123`), hexadecimal (`0x7B`), binary (`0b0111_1011`), and octal (`0o173`) notations are supported. Underscores (`_`) can be used as separators for readability (e.g., `1_000_000`). Type suffixes (e.g., `100u8`, `500i256`) are necessary when the type cannot be inferred or differs from the default integer type (TBD, likely `i64`).
* **Strings:** Double quotes (`"..."`) enclose `string` literals. Standard escape sequences like `\n`, `\t`, `\\`, `\"`, and Unicode escapes `\u{...}` are supported.
* **Booleans:** `true` and `false`.
* **Address/Hash:** Direct literals for `address` and `hash256` are **disallowed**. These types must be constructed using validated functions from the standard library (e.g., `Address::from_hex(str)?`, `Hash256::from_bytes(bytes)?`) that typically return `Result` or `Option` to handle parsing/validation errors.

### 4.7 Integer Arithmetic Behavior

* **Default Overflow:** Standard arithmetic operators (`+`, `-`, `*`) applied to integer types **revert** the transaction if the mathematical result exceeds the minimum or maximum value representable by the type.
* **Explicit Operations:** The standard library provides methods for explicit control over overflow behavior:
    * `wrapping_add`, `wrapping_sub`, etc.: Perform two's complement wrapping arithmetic.
    * `checked_add`, `checked_sub`, etc.: Return `Option<IntegerType>`, yielding `None` on overflow/underflow.
    * `saturating_add`, `saturating_sub`, etc.: Clamp the result to the type's minimum or maximum value on overflow/underflow.
* **Division/Modulo by Zero:** Performing integer division (`/`) or modulo (`%`) with a zero divisor triggers a **panic**.

### 4.8 Floating Point

* **Disallowed:** Binary floating-point types (`f32`, `f64`) and their associated arithmetic are not supported in Twilight due to potential non-determinism across different execution environments, which is incompatible with blockchain consensus requirements. Fixed-point arithmetic using integer types or dedicated libraries must be used for fractional values.

## 5. Memory Model & Ownership

Twilight guarantees memory safety statically through a compiler-managed ownership and borrowing system, inspired by Rust, but without requiring explicit lifetime or borrow syntax from the developer.

### 5.1 Ownership

* Every value in Twilight has a variable binding (`val` or `var`) that is considered its owner.
* There is only one owner at a time.
* When the owner goes out of scope, the value is dropped (memory resources are potentially reclaimed).

### 5.2 Move & Copy Semantics

The behavior of assignments, function argument passing, and function returns depends on whether a type implements the `Copy` trait.

* **`Copy` Types:**
    * Includes `bool`, all integer types (`u8`-`u256`, `i8`-`i256`), `address`, `hash256`, `timestamp`, `blocknumber`. Tuples and fixed-size `array`s composed solely of `Copy` types are also `Copy`.
    * When a `Copy` type value is used in an assignment or passed/returned from a function, its bits are copied. The original binding remains valid and usable.
    ```twilight
    val x: u64 = 100;
    val y = x; // y is a copy of 100. x is still valid.
    ```
* **Move Types:**
    * All other types that are not `Copy` (including `string`, `vector`, `map`, most `struct`s and `enum`s) follow move semantics.
    * When a Move type value is assigned, passed as an argument, or returned from a function, ownership of the underlying resource is transferred.
    * The original binding becomes invalidated and cannot be accessed further. This is enforced by the compiler at compile time.
    ```twilight
    val v1: vector<u64> = [1].to_vector();
    val v2 = v1; // Ownership of vector data moves to v2.
    // v1 cannot be used here anymore (compile error).
    ```

### 5.3 Implicit Borrowing (Compiler Managed)

The compiler infers temporary non-owning references (borrows) to enable accessing data without transferring ownership, while enforcing safety rules internally.

* **Borrowing Rules (Enforced by Compiler):**
    1.  At any time, you can have either one mutable borrow OR any number of shared borrows.
    2.  Borrows must not outlive the data they refer to.
* **Shared Borrows (`&T` Equivalent):** Allow read-only access. Multiple shared borrows can coexist. Inferred typically when passing Move types to functions expecting `fn process(item: T)`.
* **Mutable Borrows (`&mut T` Equivalent):** Allow read-write access. Only one mutable borrow can exist at a time. Inferred when passing a `var`-bound Move type to functions expecting `fn mutate(mut item: T)` or calling methods taking `mut self`. The original `var` binding is considered exclusively borrowed during the mutable borrow's lifetime.
* **No Explicit Syntax:** Developers do not write `&` or `&mut`. The compiler manages this based on function signatures (`mut` keyword on parameters/`self`) and usage context. Compile-time errors occur if rules are violated.

### 5.4 Lifetime Management (Compiler Managed)

* The compiler performs lifetime analysis implicitly to ensure inferred borrows do not outlive their owners.
* **No Explicit Syntax:** Developers do not write explicit lifetime annotations (`'a`).
* **Possible Restrictions:** To ensure safety without explicit annotations, initial versions of the language may restrict patterns like returning borrows derived from multiple inputs or storing borrows within structs (favoring owned data).

## 6. Modules & Visibility (Basic Concepts)

Code is organized into modules, and visibility controls access between them.

* **Modules:** Each `.twl` file corresponds to a module. The directory structure creates a hierarchy.
* **Visibility:**
    * **Private (Default):** Items are only usable within the module (.twl file) they are defined in.
    * **Crate-Visible (`pub(crate)`):** Items are usable anywhere within the same crate (compilation unit defined by `Twilight.toml`), but not by external crates.
    * **Public (`pub`):** Items are usable anywhere, including external crates that depend on this one. Defines the public API.
* **Struct Field Visibility:** Fields within a struct are private by default and must be marked `pub` or `pub(crate)` to be accessed from outside the module.
* **Enum Variant Visibility:** Enum variants inherit the visibility of the enum itself. Fields within enum variants follow struct field visibility rules.
* **Imports (`use`):** The `use` keyword brings items from other modules into scope (`use path::to::Item;`, `use crate::module;`).
* **Re-exporting (`pub use`):** Allows making an imported item part of the current module's public API.

*(Detailed rules are provided in the Module System and Grammar specifications).*

## 7. Generics

Generics allow writing code that operates over multiple types.

### 7.1 Generic Functions

* Defined using type parameters in angle brackets: `<T>`. Types can be constrained using trait bounds (See Section 8.3).
    ```twilight
    // Generic function swap (requires T to be Copy or handle moves appropriately)
    fn swap<T>(a: T, b: T) -> (T, T) {
        (b, a)
    }
    ```
* The compiler usually infers the concrete types (`u64`, `string`, etc.) at the call site.

### 7.2 Generic Data Structures

* Structs and enums can be defined with generic type parameters.
    ```twilight
    struct Pair<A, B> { first: A, second: B }
    enum Option<T> { Some(T), None } // Already introduced as safety type
    ```
* Concrete types are provided during instantiation. `Option<u64>`, `Pair<address, string>`.

*(Implementation Note: Generics are typically monomorphized at compile time, generating specialized code for each concrete instantiation, ensuring zero runtime overhead compared to non-generic code at the cost of potential code size increase).*

## 8. Traits

Traits define shared interfaces and abstract behavior.

### 8.1 Defining Traits

* Use the `trait` keyword followed by the trait name and a block `{}` containing method signatures.
    ```twilight
    trait EventProcessor {
        // Method takes shared access to self
        fn process_event(self, event_data: vector<u8>);

        // Method takes mutable access to self
        fn reset_state(mut self);

        // Method takes ownership of self
        // fn finalize(self) -> Summary; // 'self' usage TBD further
    }
    ```
* Method signatures specify the required functions, including the receiver (`self` or `mut self`).

### 8.2 Implementing Traits

* Use `impl TraitName for TypeName { ... }` to provide concrete implementations of trait methods for a specific type.
    ```twilight
    struct SimpleProcessor { /* fields */ }

    impl EventProcessor for SimpleProcessor {
        fn process_event(self, event_data: vector<u8>) { /* ... */ }
        fn reset_state(mut self) { /* ... */ }
    }
    ```
* All methods defined in the trait must be implemented. Method calls use dot notation (`my_processor.process_event(...)`).

### 8.3 Trait Bounds (Generic Constraints)

* Traits are used to constrain generic parameters, ensuring they provide required functionality.
* Syntax:
    * Single bound: `<T: TraitName>`
    * Multiple bounds: `<T: Trait1 + Trait2>`
    * `where` clause (for clarity): `fn foo<K, V>(k: K, v: V) where K: Hashable + Eq, V: Serializable { ... }`
* Allows calling trait methods (e.g., `k.hash()`) on generic types within the generic function body. The compiler verifies that instantiated types satisfy the bounds.

### 8.4 Abstract Types (`impl Trait`)

* **Argument Position:** Using `impl TraitName` as an argument type is syntactic sugar for a generic function with a trait bound. Static dispatch is used.
    ```twilight
    // Equivalent to fn print_generic<T: Displayable>(item: T)
    fn print_item(item: impl Displayable) { /* ... */ }
    ```
* **Return Position:** Using `impl TraitName` as a return type allows returning an opaque type that implements the trait, hiding the concrete implementation type.
    * **Constraint:** All code paths within the function MUST return the *same* concrete type.
    * Static dispatch is used.
    ```twilight
    struct MyWidget { /* ... */ }
    impl Widget for MyWidget { /* ... */ }

    fn create_widget() -> impl Widget {
        MyWidget { /* ... */ } // Always returns a MyWidget
    }
    ```

*(Note: Dynamic dispatch using `dyn Trait` trait objects is not supported in v1.0).*

## 9. Blockchain Integration

Twilight integrates blockchain concepts as first-class language features.

### 9.1 Contracts (`contract`)

* The `contract` keyword defines a smart contract, the primary unit of deployment and state on NOVA.
    ```twilight
    contract MyToken {
        // Storage, events, functions defined inside
    }
    ```

### 9.2 Storage (`storage`)

* An explicit `storage { ... }` block within a `contract` defines its persistent state variables.
* Variables declared here reside in the contract's storage on the blockchain.
* They are accessed directly by name within the contract's functions.
* Variables are zero-initialized unless an explicit initializer is provided.
    ```twilight
    contract SimpleCounter {
        storage {
            count: u64 = 0; // Initialized
            owner: address; // Zero-initialized (zero address)
        }
        // ... functions accessing storage.count, storage.owner ...
    }
    ```

### 9.3 Events (`event`, `emit`, `#[topic]`)

* Define events using the `event` keyword and typed parameters.
    ```twilight
    event Transfer(#[topic] from: address, #[topic] to: address, amount: u256);
    ```
* The `#[topic]` attribute marks parameters (up to ~3) for indexing, facilitating off-chain filtering. The event signature hash serves as an implicit first topic.
* Emit events using the `emit` keyword followed by the event name and arguments. Logs are recorded in the transaction receipt.
    ```twilight
    emit Transfer(sender, recipient, transfer_amount);
    ```

### 9.4 Context Access

* Access blockchain transaction and environment context via predefined namespaces (e.g., `ctx`, `env`).
    * `ctx.sender()`: Address of the immediate caller.
    * `ctx.value()`: Native token (`NOVA`) amount sent with the call (only non-zero in `#[payable]` functions).
    * `env.block_number()`: Current block number/height.
    * `env.timestamp()`: Current block timestamp.
    * `self.address()`: The contract's own address.

### 9.5 Payability (`#[payable]`)

* Functions must be explicitly annotated with `#[payable]` to accept incoming native token (`NOVA`) transfers via `ctx.value()`. Calls sending value to non-payable functions will revert.

### 9.6 Lifecycle (`init`, No `selfdestruct`)

* **Constructor (`init`):** A contract can define at most one `fn init(...) { ... }`. This function is executed exactly once upon contract deployment for initialization purposes. It can accept deployment arguments. Failures within `init` (using `require`) revert the deployment.
* **No Destruction:** Twilight **does not support** a `selfdestruct` mechanism. Contracts and their state persist permanently unless decommissioned via application-level logic (e.g., pausing, migrating state).

### 9.7 Annotations

* Annotations (`#[...]`) provide metadata for the compiler and runtime. Key examples:
    * `#[reads("domain")]`, `#[writes("domain")]`: Declare conflict domains accessed by a function.
    * `#[pipeline("hint")]`: Suggests scheduling preference for the runtime.
    * `#[payable]`: Allows function to receive native tokens.
    * `#[verify]`: Marks code for formal verification analysis.
    * `#[test]`: Marks a function as a test case.

## 10. Error Handling Mechanisms

Twilight provides distinct mechanisms for handling different failure conditions.

### 10.1 Recoverable Errors (`Result<T, E>` / `?`)

* **`Result<T, E>`:** The standard type for functions that can fail in expected ways. `Ok(T)` represents success, `Err(E)` represents failure. `E` is typically `string` or a custom error `enum`.
* **Handling:** Callers *must* handle `Result` values (e.g., using `match`) or propagate errors. Unhandled `Result` values are a compile-time error.
* **`?` Operator:** Provides ergonomic error propagation for `Result`. If `expr` evaluates to `Err(e)`, `expr?` immediately returns `Err(e)` from the enclosing function (which must itself return a compatible `Result`). If `expr` is `Ok(v)`, `expr?` evaluates to `v`.

### 10.2 Assertions / Preconditions (`require`)

* **Syntax:** `require(condition: bool, message: string);`
* **Purpose:** Checks essential conditions (preconditions, invariants, access control) that *must* be true for valid execution. Failure indicates an invalid state or action by the caller.
* **Behavior:** If `condition` is false, execution halts, the **transaction is reverted** (state changes discarded), gas up to the failure point is consumed, and the `message` is returned as the reason.

### 10.3 Unrecoverable Errors (`panic!`)

* **Purpose:** Signals critical, unrecoverable errors, typically indicating a bug in the contract logic or runtime. Should not be used for expected failures.
* **Explicit Trigger:** `panic!(message: string);`
* **Implicit Triggers (Default):** Integer division/modulo by zero; Array/vector index out of bounds. *(Integer overflow defaults to revert, not panic).*
* **Behavior:** If a panic occurs, execution halts, the **transaction is reverted** (state changes discarded), and **all gas** provided for the transaction (up to the gas limit) is consumed. This high penalty strongly discourages panics for non-fatal errors.