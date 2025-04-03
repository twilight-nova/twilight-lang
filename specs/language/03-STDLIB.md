# Twilight Standard Library Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the scope, structure, and essential Application Programming Interfaces (APIs) provided by the Twilight Standard Library. This library offers fundamental types, traits, functions, and modules available to all Twilight smart contracts, ensuring a baseline capability for safe, deterministic, and effective development on the NOVA blockchain platform.

### 1.2 Design Philosophy

* **Core Essentials:** Provide broadly useful, fundamental building blocks. Avoid highly specialized features better suited for external packages.
* **Determinism & Safety:** Exclude any functionality that could introduce non-determinism (e.g., system time beyond `env::timestamp`, random numbers, floating points, external I/O like filesystem/network). Prioritize safety via types like `Option<T>`/`Result<T, E>` and safe APIs.
* **Blockchain Focused:** Provide convenient, safe access to NOVA-specific environment/transaction context and essential cryptographic primitives via wrappers around the Host Function Interface (HFI).
* **Minimal & Composable:** Maintain a relatively small and focused API surface, encouraging a rich ecosystem of external libraries for advanced features.
* **Consistency:** Ensure API design is consistent with the `twilight-native` language style.

### 1.3 Structure

The standard library is organized into two primary top-level namespaces:

* **`core::`**: Contains fundamental language items, types, traits, and general utilities mostly independent of the blockchain context (though implemented with blockchain constraints).
* **`nova::`**: Contains items specifically for interacting with the NOVA blockchain environment, transaction context, and consensus mechanisms. These often map directly to Host Function Interface (HFI) calls provided by the runtime.

## 2. Implementation Strategy

The standard library is an intrinsic part of the Twilight toolchain and runtime environment, not a typical user dependency.

* **Implementation Language:** Primarily implemented in Rust for performance, safety, and low-level control required for HFI interactions and memory management.
* **Compiler Intrinsics:** Certain fundamental items (e.g., `Option<T>`, `Result<T, E>`, core traits like `Copy`, basic numeric operations) are implemented directly by the `twilightc` compiler for optimal code generation. The standard library API provides the interface definition for these intrinsics.
* **HFI Wrappers:** Most functions within the `nova::` namespace are safe Rust wrappers around the low-level HFI calls defined in the HFI Specification (`specs/interfaces/11-hfi.md`). These wrappers handle buffer management, pointer validation, error code translation, and data serialization/deserialization across the WASM/host boundary.
* **Pure Logic:** Standard library components not involving intrinsics or HFI calls (e.g., certain collection helpers, formatting logic) are implemented in pure Rust logic within the library's source.
* **Memory Management:** Dynamically sized types (`vector<T>`, `map<K, V>`, `string`) manage memory via runtime-provided allocator functions exposed through the HFI (details in HFI spec).

## 3. `core::prelude`

Items in the prelude are automatically imported into every Twilight module's scope.

* **Types:** `bool`, `u8`, `u16`, `u32`, `u64`, `u128`, `u256`, `i8`, `i16`, `i32`, `i64`, `i128`, `i256`, `string`, `address`, `hash256`, `timestamp`, `blocknumber`, `vector<T>`, `map<K, V>`, `array<T, N>`, `Option<T>` (including variants `Some` and `None`), `Result<T, E>` (including variants `Ok` and `Err`).
* **Traits (Core):** `Copy`, `Clone` (implicitly derived for `Copy` types). Other core traits like `Eq`, `Ord`, `Hash` are defined but may require explicit imports. `Drop` is handled implicitly by the compiler.
* **Functions / Keywords:** `require(bool, string)` (language keyword), `panic!(string)` (language intrinsic/keyword). A `print(...)` function for debugging (via HFI) is typically included.

## 4. `core::*` Modules

### 4.1 Module: `core::option`

Provides methods for working with the `Option<T>` enum.

* `is_some(self) -> bool`: Returns `true` if the option is `Some`.
* `is_none(self) -> bool`: Returns `true` if the option is `None`.
* `expect(self, msg: string) -> T`: Returns the contained `Some` value. Panics with `msg` if the option is `None`.
* `unwrap(self) -> T`: Returns the contained `Some` value. Panics if the option is `None`.
* `unwrap_or(self, default: T) -> T`: Returns the contained `Some` value or `default` if `None`. `T` must be `Copy` or handled implicitly.
* `unwrap_or_else<F>(self, f: F) -> T where F: FnOnce() -> T`: Returns the contained `Some` value or computes it from `f` if `None`.
* `map<U, F>(self, f: F) -> Option<U> where F: FnOnce(T) -> U`: Maps an `Option<T>` to `Option<U>` by applying `f` to a contained value.
* `ok_or<E>(self, err: E) -> Result<T, E>`: Transforms `Some(v)` to `Ok(v)` and `None` to `Err(err)`.
* `ok_or_else<E, F>(self, f: F) -> Result<T, E> where F: FnOnce() -> E`: Transforms `Some(v)` to `Ok(v)` and `None` to `Err(f())`.

### 4.2 Module: `core::result`

Provides methods for working with the `Result<T, E>` enum.

* `is_ok(self) -> bool`: Returns `true` if the result is `Ok`.
* `is_err(self) -> bool`: Returns `true` if the result is `Err`.
* `ok(self) -> Option<T>`: Converts `Result<T, E>` to `Option<T>`, discarding the error.
* `err(self) -> Option<E>`: Converts `Result<T, E>` to `Option<E>`, discarding the success value.
* `expect(self, msg: string) -> T`: Returns the contained `Ok` value. Panics with `msg` if the result is `Err`.
* `unwrap(self) -> T`: Returns the contained `Ok` value. Panics if the result is `Err`.
* `expect_err(self, msg: string) -> E`: Returns the contained `Err` value. Panics with `msg` if the result is `Ok`.
* `unwrap_err(self) -> E`: Returns the contained `Err` value. Panics if the result is `Ok`.
* `map<U, F>(self, op: F) -> Result<U, E> where F: FnOnce(T) -> U`: Maps `Ok(T)` to `Ok(op(T))`, leaving `Err` untouched.
* `map_err<F, O>(self, op: O) -> Result<T, F> where O: FnOnce(E) -> F`: Maps `Err(E)` to `Err(op(E))`, leaving `Ok` untouched.

### 4.3 Module: `core::num`

Provides explicit arithmetic operations and utilities for primitive integer types (`u8`..`u256`, `i8`..`i256`). These supplement the standard operators which revert on overflow by default.

* `checked_add(self, rhs: Self) -> Option<Self>`: Checked addition. Returns `None` on overflow.
* `checked_sub(self, rhs: Self) -> Option<Self>`: Checked subtraction. Returns `None` on overflow/underflow.
* `checked_mul(self, rhs: Self) -> Option<Self>`: Checked multiplication. Returns `None` on overflow.
* `checked_div(self, rhs: Self) -> Option<Self>`: Checked division. Returns `None` if `rhs` is zero or on signed overflow (`MIN / -1`).
* `wrapping_add(self, rhs: Self) -> Self`: Wrapping (two's complement) addition.
* `wrapping_sub(self, rhs: Self) -> Self`: Wrapping subtraction.
* `wrapping_mul(self, rhs: Self) -> Self`: Wrapping multiplication.
* `saturating_add(self, rhs: Self) -> Self`: Saturating addition (clamps to type's min/max value).
* `saturating_sub(self, rhs: Self) -> Self`: Saturating subtraction.
* `pow(self, exp: u32) -> Self`: Exponentiation. Default behavior reverts on overflow. Checked/wrapping/saturating variants may be provided.
* `from_str_radix(s: string, radix: u32) -> Result<Self, ParseIntError>`: Parses an integer from a string with the given radix.
* *(Additional methods like `count_ones()`, `leading_zeros()`, etc. may be included).*

### 4.4 Module: `core::string`

Provides operations for the immutable `string` type. String creation/manipulation involves memory allocation managed via runtime HFI calls.

* `len(self) -> u64`: Returns the length of the string in bytes.
* `is_empty(self) -> bool`: Returns `true` if the string length is 0.
* `slice(self, start_byte: u64, end_byte: u64) -> Result<string, SliceError>`: Returns a new string containing a byte slice of the original. Returns `Err` if indices are out of bounds or do not fall on UTF-8 character boundaries.
* `find(self, pattern: string) -> Option<u64>`: Returns the byte index of the first occurrence of `pattern`, or `None`.
* `contains(self, pattern: string) -> bool`: Checks if the string contains `pattern`.
* `split(self, separator: string) -> vector<string>`: Splits the string by `separator` into a vector of new strings.
* `concat(s1: string, s2: string) -> string`: Concatenates two strings. (The `+` operator may be overloaded for this).
* *(Other methods like `starts_with`, `ends_with`, `replace`, case conversion may be added based on evaluation of needs vs. complexity/gas cost).*

### 4.5 Module: `core::collections`

Provides APIs for standard collection types. Implementations manage memory via the runtime allocator HFI.

* **`vector<T>`**: Dynamically sized array.
    * `new() -> Self`
    * `with_capacity(capacity: u64) -> Self`
    * `len(self) -> u64`
    * `is_empty(self) -> bool`
    * `capacity(self) -> u64`
    * `push(mut self, value: T)` (May reallocate)
    * `pop(mut self) -> Option<T>`
    * `get(self, index: u64) -> Option<T>` (Requires `T: Copy` or `T: Clone`. Returns `None` on OOB)
    * `set(mut self, index: u64, value: T) -> Result<(), BoundsError>` (Requires `T`. Returns `Err` on OOB)
    * `clear(mut self)`
    * *(Iteration methods TBD)*
* **`map<K, V>`**: Key-value hash map. Requires `K: Hash + Eq` traits.
    * `new() -> Self`
    * `len(self) -> u64`
    * `is_empty(self) -> bool`
    * `insert(mut self, key: K, value: V) -> Option<V>` (Returns previous value if key existed)
    * `get(self, key: K) -> Option<V>` (Requires `V: Copy` or `V: Clone`. Returns `None` if key not found)
    * `remove(mut self, key: K) -> Option<V>` (Returns value if key existed)
    * `contains_key(self, key: K) -> bool`
    * *(Iteration methods for keys/values/pairs TBD)*

### 4.6 Module: `core::array`

Provides methods for fixed-size `array<T, N>`.

* `len(self) -> u64`: Returns the compile-time constant size `N`.
* `get(self, index: u64) -> Option<T>`: Safe indexed access. Returns `None` if `index` >= `N`. Requires `T: Copy` or `T: Clone`.
* *(Iteration methods TBD)*
* *(Note: Direct indexing `array[index]` is provided by the language but panics on out-of-bounds access).*

### 4.7 Module: `core::fmt`

Provides basic string formatting capabilities. The exact mechanism involves compiler support and traits.

* **Trait `Displayable` (Conceptual):** A standard trait enabling types to be formatted. `trait Displayable { fn fmt(self, f: &mut Formatter) -> Result<(), FmtError>; }`. Implementations for primitives are provided. Users can implement for custom types.
* **Function `print(fmt_string: string, args...)`:** Available for debugging. Formats arguments implementing `Displayable` according to `fmt_string` specifiers (e.g., `{}`) and outputs via the `nova_debug::debug_print` HFI. This function is non-deterministic regarding external output and has significant gas cost; disabled in production builds.

## 5. `nova::*` Modules (Blockchain Interaction via HFI)

These modules provide safe, high-level Twilight interfaces to the NOVA Host Function Interface (HFI).

### 5.1 Module: `nova::env`

Accesses blockchain environment information.

* `block_number() -> u64`: Returns the current block number/height. (Wraps `env_get_block_number` HFI).
* `timestamp() -> timestamp`: Returns the current block timestamp (`u64`). (Wraps `env_get_timestamp` HFI).
* `self_address() -> address`: Returns the address of the currently executing contract. (Wraps `env_get_self_address` HFI).
* `gas_left() -> u64`: Returns the approximate remaining gas available for execution. (Wraps `env_get_gas_left` HFI).

### 5.2 Module: `nova::tx`

Accesses information about the current transaction context.

* `sender() -> address`: Returns the address of the immediate message caller. (Wraps `tx_get_sender` HFI).
* `origin() -> address`: Returns the address of the original transaction signer. (Wraps `tx_get_origin` HFI).
* `value() -> u128`: Returns the amount of native NOVA token sent with the call. Returns `0` if called in a non-`#[payable]` function. (Wraps `tx_get_value` HFI).

### 5.3 Module: `nova::crypto`

Provides access to core cryptographic primitives via HFI calls.

* `keccak256(input: vector<u8>) -> hash256`: Computes the Keccak-256 hash of the input bytes. (Wraps `crypto_keccak256` HFI).
* `blake3(input: vector<u8>) -> hash256`: Computes the Blake3 hash of the input bytes. (Wraps `crypto_blake3` HFI).
* *(Note: Signature verification functions and other cryptographic algorithms are deferred to external libraries or future stdlib extensions).*

### 5.4 Module: `nova::contract`

Provides contract-specific operations.

* *(Note: `emit` is a language keyword. Low-level calls like `call` or `delegatecall` are excluded for safety in v1.0. `self.address()` is available directly or via `nova::env`).*
* This module is expected to be minimal in v1.0.

## 6. Excluded Functionality

The standard library explicitly excludes features that could introduce non-determinism or require unsafe host environment access:

* File system I/O.
* Network I/O.
* Direct operating system interactions (time beyond `env::timestamp`, process management, environment variables).
* Standard concurrency primitives (threads, mutexes); parallelism is managed by the NOVA runtime.
* Random number generation (requires protocol-level solutions like VRFs).
* Access to precise wall-clock time.
* Binary floating-point types and arithmetic.