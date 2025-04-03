# Twilight Host Function Interface (HFI) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies version 1.0 of the Host Function Interface (HFI) provided by the NOVA runtime environment to executed WebAssembly (WASM) smart contracts compiled from Twilight. The HFI defines the set of functions that WASM contracts can import and call to interact with the blockchain state, transaction context, cryptographic primitives, and other host-provided functionalities. It forms the critical, secure boundary between the untrusted WASM guest and the trusted NOVA host.

### 1.2 Target Audience

* Twilight Compiler (`twilightc`) Implementers (Backend targeting WASM and generating HFI calls)
* NOVA Runtime/VM Implementers (Providing the host-side implementation of HFI functions)
* Twilight Standard Library Implementers (Creating safe wrappers around HFI calls)
* Smart Contract Auditors and Developers (Understanding the contract execution environment)

### 1.3 Design Principles

* **Minimal & Explicit:** The HFI exposes only functionality essential for smart contract execution, minimizing the attack surface.
* **Deterministic:** All HFI functions MUST produce identical outputs given identical blockchain state and inputs. They must not rely on non-deterministic host features.
* **Gas Metered:** Every HFI function call has a defined gas cost (specified separately in the protocol's Gas Cost Table). The host MUST deduct this cost *before* executing the function's logic.
* **Secure Boundary:** The host implementation MUST treat all inputs from the WASM guest (pointers, lengths, data) as untrusted and rigorously validate them before use.
* **Efficient Marshalling:** Data transfer across the WASM/host boundary primarily uses pointers and lengths for efficiency.
* **Clear Error Handling:** HFI functions use standardized return codes to signal success or specific error conditions.
* **Versioning:** The HFI interface is versioned to allow for future evolution.

## 2. Versioning

* **Current Version:** `v1`
* **WASM Import Naming:** Contracts import HFI functions using versioned module names (e.g., `"nova_state_v1"`, `"nova_tx_v1"`).
* **Runtime Enforcement:** The NOVA runtime MUST verify that it supports the HFI version(s) imported by a contract during instantiation and reject incompatible contracts.

## 3. Data Marshalling

* **Primary Method:** Pointers (represented as `i32` byte offsets into the WASM instance's linear memory) and Lengths (represented as `i32` byte counts) are used to specify input and output buffers for variable-sized data (keys, values, addresses, hashes, strings).
* **Primitive Values:** Small, fixed-size values (`i32`, `i64`) may be passed directly as WASM function arguments or return values. Note that larger Twilight primitives (like `u128`, `u256`) are handled via pointers/lengths to their memory representation.
* **Return Values:**
    * Simple status or small values are returned directly (typically `i32` for status codes, `i64` for `u64` values).
    * Variable-sized data results are written back into an output buffer in WASM memory provided by the caller via pointer/length arguments. The function may return the number of bytes written.
* **Endianness:** All multi-byte integer values exchanged via memory buffers use Little Endian encoding.
* **Memory Safety (Host Responsibility):** The host implementation of *every* HFI function MUST validate all pointer and length arguments received from WASM before accessing WASM linear memory. This includes:
    1.  Checking for arithmetic overflow when calculating `end_offset = pointer + length`.
    2.  Checking that `pointer >= 0` and `end_offset <= current_wasm_memory_size`.
    3.  Failure of these checks MUST result in an immediate return with an appropriate error code (e.g., `NOVA_ERR_INVALID_POINTER`, `NOVA_ERR_INVALID_LENGTH`) without attempting the memory access.

## 4. Error Handling & Return Codes

* **Return Convention:** Most HFI functions return an `i32` status code.
    * `NOVA_SUCCESS` (0): Indicates successful execution.
    * Positive Value (> 0): For functions that read data into a buffer (e.g., `nova_state_v1::read`), indicates the total size of the requested data in bytes. The caller compares this with the provided buffer size to check for truncation.
    * Negative Value (< 0): Indicates an error condition, corresponding to a standard error code.
* **Standard Error Codes (`i32`):**
    * `NOVA_SUCCESS = 0`
    * `NOVA_ERR_INVALID_POINTER = -1` (Pointer out of bounds, null, or potentially misaligned)
    * `NOVA_ERR_INVALID_LENGTH = -2` (Length causes out-of-bounds access, exceeds limits, or is otherwise invalid)
    * `NOVA_ERR_BUFFER_TOO_SMALL = -3` (Output buffer provided by WASM is too small for the result)
    * `NOVA_ERR_KEY_NOT_FOUND = -4` (Specified key does not exist in state)
    * `NOVA_ERR_INVALID_ARGUMENT = -5` (Other invalid argument value, e.g., invalid domain hash format, bad UTF-8 encoding, invalid enum variant)
    * `NOVA_ERR_STATE_ACCESS_VIOLATION = -6` (State write attempt denied by protocol rules/permissions)
    * `NOVA_ERR_LIMIT_EXCEEDED = -7` (Protocol limit reached, e.g., max event size, max state writes)
    * `NOVA_ERR_SERIALIZATION = -8` (Internal host error during data serialization/deserialization for HFI)
    * `NOVA_ERR_INTERNAL_HOST_ERROR = -9` (Generic unexpected host-side failure)
    * `NOVA_ERR_GAS_EXHAUSTED = -10` (Gas ran out during HFI execution - less common, usually trapped earlier)
* **Contract Handling:** The Twilight standard library wrappers translate these `i32` codes into idiomatic Twilight `Result` types (e.g., `Result<u64, NovaError>`). Direct callers must check return codes.

## 5. Gas Costing

* **Principle:** Every successful HFI function call consumes gas.
* **Cost Definition:** Gas costs (base cost + potential dynamic cost based on arguments like buffer lengths) are defined externally in the protocol's canonical Gas Cost Table.
* **Deduction:** The NOVA runtime host MUST deduct the calculated gas cost for the HFI call *before* executing the host-side logic of the function. If insufficient gas remains, the call MUST fail immediately (e.g., return `NOVA_ERR_GAS_EXHAUSTED` or trigger OOG trap) without executing the logic.

## 6. HFI Module & Function Specifications (v1.0)

This section defines the functions available under version `v1` of the HFI, grouped by import module name. WASM signatures are shown in text format.

### 6.1 Module: `nova_state_v1`

Functions for interacting with persistent contract storage.

* **`read`**
    * WASM Signature: `(import "nova_state_v1" "read" (func $state_read (param i32 i32 i32 i32 i32 i32) (result i32)))`
    * Definition: `fn read(domain_hash_ptr: i32, key_ptr: i32, key_len: i32, value_out_ptr: i32, value_out_offset: i32, value_out_len: i32) -> i32`
    * Description: Reads the state value associated with `key` within the specified `domain_hash`. Writes the value bytes into the buffer at `value_out_ptr` starting at `value_out_offset`, up to `value_out_len` bytes. Host validates all pointers, lengths, and domain hash format.
    * Returns: Total size of the value in bytes if found (caller checks against `value_out_len` for truncation), `NOVA_ERR_KEY_NOT_FOUND` if key does not exist, or other negative error code on failure.
* **`write`**
    * WASM Signature: `(import "nova_state_v1" "write" (func $state_write (param i32 i32 i32 i32 i32) (result i32)))`
    * Definition: `fn write(domain_hash_ptr: i32, key_ptr: i32, key_len: i32, value_ptr: i32, value_len: i32) -> i32`
    * Description: Writes the `value` (read from `value_ptr` with `value_len`) to the specified `key` under the `domain_hash`. Host validates all inputs.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.
* **`exists`**
    * WASM Signature: `(import "nova_state_v1" "exists" (func $state_exists (param i32 i32 i32) (result i32)))`
    * Definition: `fn exists(domain_hash_ptr: i32, key_ptr: i32, key_len: i32) -> i32`
    * Description: Checks if a value exists for the specified `key` under `domain_hash`. Host validates inputs.
    * Returns: `1` if exists, `0` if not exists, negative error code otherwise.

### 6.2 Module: `nova_tx_v1`

Functions for accessing current transaction context.

* **`get_sender`**
    * WASM Signature: `(import "nova_tx_v1" "get_sender" (func $tx_get_sender (param i32) (result i32)))`
    * Definition: `fn get_sender(addr_out_ptr: i32)`
    * Description: Writes the immediate caller's address (fixed size, e.g., 20 bytes) into the buffer pointed to by `addr_out_ptr`. Host validates pointer.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.
* **`get_value`**
    * WASM Signature: `(import "nova_tx_v1" "get_value" (func $tx_get_value (param i32) (result i32)))`
    * Definition: `fn get_value(value_out_ptr: i32)`
    * Description: Writes the native token amount (`u128`, 16 bytes) sent with the transaction into the buffer pointed to by `value_out_ptr`. Returns zero value if the called contract function is not payable. Host validates pointer.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.
* **`get_origin`**
    * WASM Signature: `(import "nova_tx_v1" "get_origin" (func $tx_get_origin (param i32) (result i32)))`
    * Definition: `fn get_origin(addr_out_ptr: i32)`
    * Description: Writes the original transaction signer's address (fixed size) into the buffer pointed to by `addr_out_ptr`. Host validates pointer.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.

### 6.3 Module: `nova_env_v1`

Functions for accessing blockchain environment information.

* **`get_block_number`**
    * WASM Signature: `(import "nova_env_v1" "get_block_number" (func $env_get_block_number (result i64)))`
    * Definition: `fn get_block_number() -> i64`
    * Description: Returns the current block number (height) as a `u64`.
    * Returns: `u64` block number.
* **`get_timestamp`**
    * WASM Signature: `(import "nova_env_v1" "get_timestamp" (func $env_get_timestamp (result i64)))`
    * Definition: `fn get_timestamp() -> i64`
    * Description: Returns the current block timestamp (Unix seconds) as a `u64`.
    * Returns: `u64` timestamp.
* **`get_self_address`**
    * WASM Signature: `(import "nova_env_v1" "get_self_address" (func $env_get_self_address (param i32) (result i32)))`
    * Definition: `fn get_self_address(addr_out_ptr: i32)`
    * Description: Writes the current contract's own address (fixed size) into the buffer pointed to by `addr_out_ptr`. Host validates pointer.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.
* **`get_gas_left`**
    * WASM Signature: `(import "nova_env_v1" "get_gas_left" (func $env_get_gas_left (result i64)))`
    * Definition: `fn get_gas_left() -> i64`
    * Description: Returns the approximate amount of remaining gas (`u64`) before the cost of this HFI call itself is deducted.
    * Returns: `u64` gas remaining.

### 6.4 Module: `nova_contract_v1`

Functions for contract-specific actions.

* **`emit_event`**
    * WASM Signature: `(import "nova_contract_v1" "emit_event" (func $contract_emit_event (param i32 i32 i32 i32) (result i32)))`
    * Definition: `fn emit_event(topics_ptr: i32, topics_count: i32, data_ptr: i32, data_len: i32) -> i32`
    * Description: Emits a log entry. `topics_ptr` points to an array of `topics_count` consecutive `hash256` values (32 bytes each) in memory. `data_ptr` points to the ABI-encoded data payload of `data_len` bytes. Host reads topics and data, validates inputs against limits, and constructs/stores the log entry.
    * Returns: `NOVA_SUCCESS` (0) or negative error code (e.g., `NOVA_ERR_LIMIT_EXCEEDED`).
* **`revert`**
    * WASM Signature: `(import "nova_contract_v1" "revert" (func $contract_revert (param i32 i32) (result i32)))`
    * Definition: `fn revert(msg_ptr: i32, msg_len: i32) -> i32` *(Does not actually return to WASM caller)*
    * Description: Immediately halts contract execution and signals a transaction revert. The host reads the UTF-8 encoded revert message from WASM memory specified by `msg_ptr` and `msg_len`. Host validates pointer/length.
    * Returns: This function effectively does not return control flow to the WASM caller. The signature includes a result for linkage purposes, but execution stops.

### 6.5 Module: `nova_crypto_v1`

Functions for core cryptographic operations.

* **`keccak256`**
    * WASM Signature: `(import "nova_crypto_v1" "keccak256" (func $crypto_keccak256 (param i32 i32 i32) (result i32)))`
    * Definition: `fn keccak256(input_ptr: i32, input_len: i32, output_ptr: i32) -> i32`
    * Description: Computes the Keccak-256 hash of the input data read from WASM memory (`input_ptr`, `input_len`). Writes the 32-byte hash result to the buffer pointed to by `output_ptr`. Host validates pointers/lengths.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.
* **`blake3`**
    * WASM Signature: `(import "nova_crypto_v1" "blake3" (func $crypto_blake3 (param i32 i32 i32) (result i32)))`
    * Definition: `fn blake3(input_ptr: i32, input_len: i32, output_ptr: i32) -> i32`
    * Description: Computes the Blake3 hash (32 bytes) of the input data read from WASM memory. Writes the result to the buffer at `output_ptr`. Host validates pointers/lengths.
    * Returns: `NOVA_SUCCESS` (0) or negative error code.

### 6.6 Module: `nova_debug_v1` (Optional/Debug Only)

Functions intended only for debugging purposes. These functions MUST be disabled or implemented as no-ops in production mainnet environments due to non-determinism and potential resource usage.

* **`print`**
    * WASM Signature: `(import "nova_debug_v1" "print" (func $debug_print (param i32 i32)))`
    * Definition: `fn print(msg_ptr: i32, msg_len: i32)`
    * Description: The host reads the UTF-8 encoded message from WASM memory specified by `msg_ptr` and `msg_len` and prints it to the node's debug log output. Host validates pointers/lengths.
    * Returns: Void (implicitly success if enabled). May trap or return an error if called when disabled in the runtime environment.

## 7. Security Boundary Implementation Requirement

As stated in Principle 1.3.4 and Section 3, the host implementation of all HFI functions MUST treat the WASM guest as untrusted. Rigorous validation of all pointer, offset, length, and data arguments received from WASM is mandatory before they are used by the host. Failure to validate correctly can lead to critical security vulnerabilities (e.g., VM escapes, host crashes, state corruption).