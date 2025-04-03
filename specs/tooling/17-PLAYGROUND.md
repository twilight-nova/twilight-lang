# Twilight Playground Backend Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the architecture, requirements, and core logic for the backend service powering the Twilight Web Playground. The playground provides a zero-installation environment for users to write, compile, execute, and share simple Twilight code snippets directly in their browser. This backend is responsible for securely handling code compilation and simulated execution.

### 1.2 Target Audience

* Implementation teams building and deploying the playground backend service.
* Frontend developers building the playground web interface interacting with this backend.

### 1.3 Design Philosophy

* **Accessibility:** Enable easy experimentation with Twilight without local setup.
* **Safety & Security:** Prioritize secure execution of untrusted user code through robust sandboxing and resource limitation. Security overrides simulation fidelity.
* **Fast Feedback:** Provide reasonably quick responses for compiling and running small code snippets.
* **Simplicity:** Focus on core functionality (compile, run, share) for single-file or small multi-file examples.
* **TSM Support:** Allow users to select and execute code using different supported Twilight Syntax Modes (e.g., `native`, `solidity`).

## 2. Backend Architecture & API

### 2.1 Service Model

The backend operates as a stateless or near-stateless web service, accessible via a simple HTTP API. It is designed to handle individual, independent requests for code processing. Deploying using containerized services or serverless functions is recommended for isolation and scalability.

### 2.2 API Endpoints

The backend exposes the following primary HTTP endpoints:

* **`POST /run`**: Compiles and executes a provided Twilight code snippet within a simulated environment.
* **`POST /compile` (Optional):** Compiles a code snippet and returns diagnostics and potentially compiled artifacts (e.g., WASM byte hash, TIR). Primarily for diagnostics or informational purposes.
* **`POST /share` (Optional):** Accepts a code snippet, stores it persistently, and returns a unique identifier or URL for sharing.
* **`GET /load?id=<ID>` (Optional):** Retrieves a previously shared code snippet based on its unique identifier.

### 2.3 Request/Response Formats

Requests and responses use JSON format.

* **Request Body (`/run`, `/compile`)**
    ```json
    {
      "code": "/* Twilight source code */",
      "syntax_mode": "native" | "solidity", // Default: "native"
      "entry_point": "function_name_to_call", // For /run, e.g., "main"
      "context": { // Optional simulation context for /run
        "sender": "0x...", // Simulated sender address
        "value": "0",      // Simulated native token value sent (stringified u128)
        "block_number": "1", // Simulated block number (stringified u64)
        "timestamp": "...",  // Simulated timestamp (stringified u64)
        // Initial storage state can be omitted for v1.0 simplicity
      },
      "gas_limit": "10000000" // Simulated gas limit (stringified u64)
    }
    ```

* **Success Response (`/run`)**
    ```json
    {
      "success": true,
      "diagnostics": [ /* List of compiler warnings */ ],
      "output": "...", // Concatenated string output from debug print calls
      "events": [        // List of simulated emitted events
        {
          "address": "0xContractAddress...", // Simulated contract address
          "topics": ["0xTopic0...", "0xTopic1..."], // Hex-encoded topic hashes
          "data": "0x..." // Hex-encoded ABI data payload
        }
      ],
      "return_value": "...", // String representation of the entry point's return value
      "gas_used": "..."      // Simulated gas consumed (stringified u64)
    }
    ```

* **Failure Response (`/run`, `/compile`)**
    ```json
    {
      "success": false,
      "diagnostics": [ // List of compiler or runtime errors/warnings
        {
          "severity": "error" | "warning",
          "message": "...",
          "span": { /* Source location info */ }
        }
      ],
      "error_type": "Compilation" | "RuntimeRevert" | "RuntimePanic" | "OutOfGas" | "Timeout" | "SandboxError" | "InternalError",
      "revert_message": "..." // Included if error_type is "RuntimeRevert"
      // Potentially include partial output/events before failure
    }
    ```

## 3. Sandboxing Requirements (Critical)

Executing arbitrary user code necessitates extremely strict sandboxing to prevent abuse and secure the backend infrastructure.

### 3.1 Isolation Techniques

* **Mandatory:** Each incoming `/compile` or `/run` request **MUST** be processed within a dedicated, ephemeral, and strongly isolated sandbox environment. Suitable techniques include:
    * **Containers:** Using Docker or similar, run each request in a new container instance that is destroyed immediately after processing.
    * **MicroVMs:** Utilizing technologies like Firecracker to provide lightweight VM isolation per request.
* **Rationale:** Prevents interference between requests and limits the blast radius of potential exploits or resource exhaustion from any single request.

### 3.2 Resource Limits

The sandbox environment for each request **MUST** have strict resource limits enforced at the infrastructure level (container runtime, VM manager, or OS):

* **CPU Time:** A short, non-negotiable execution time limit (e.g., 2-5 seconds) covering the entire request processing (compilation + execution). Requests exceeding this limit must be forcibly terminated.
* **Memory:** A low maximum memory allocation limit (e.g., 128-256 MiB). Exceeding this limit must terminate the process.
* **Networking:** Network access from within the sandbox **MUST** be completely disabled.
* **Filesystem:** The filesystem accessible within the sandbox must be primarily read-only. Limited write access should only be granted to necessary temporary directories (e.g., for `twilightc` output), which must be cleaned up after the request. No access to the host filesystem is permitted.
* **Process Limits:** Restrict the number of processes that can be spawned within the sandbox (e.g., maximum 1 or 2 PIDs).

### 3.3 Process Privileges

Processes running inside the sandbox (e.g., `twilightc`, the WASM runtime) **MUST** execute as a non-root user with the minimum privileges required. System call filtering (e.g., via `seccomp-bpf`) should be applied to further restrict capabilities.

## 4. Compilation Logic (within Sandbox)

For each `/run` or `/compile` request, the backend performs these steps within a dedicated sandbox:

1.  **Prepare Files:** Write the received `code` to a temporary source file (e.g., `src/main.twl` or `src/contract.twl`) within the sandbox. Create a minimal `Twilight.toml` if necessary. Ensure the `#![syntax = "..."]` directive matches the request's `syntax_mode`.
2.  **Invoke Compiler:** Execute the `twilightc build --target wasm` (or `twilightc check`) command as a subprocess, targeting the temporary source file(s).
3.  **Capture Output:** Redirect and capture the `stdout`, `stderr`, and exit code from the `twilightc` process.
4.  **Process Results:**
    * If `twilightc` exits non-zero or produces error diagnostics: Parse the diagnostics from `stderr`, format them according to the API response schema, and return a failure response with `error_type: "Compilation"`.
    * If `twilightc` succeeds: Parse any warning diagnostics. Locate the generated `.wasm` file (and potentially `.meta` file) in the compiler's output directory (e.g., `target/debug/`).
    * For `/compile` requests, return success with diagnostics.
    * For `/run` requests, proceed to execution simulation.
5.  **Handle Sandbox Errors:** If the compilation process exceeds the sandbox resource limits (time, memory), terminate it and return a failure response (`error_type: "Timeout"` or `"SandboxError"`).

## 5. Execution Simulation Logic (within Sandbox)

For `/run` requests after successful compilation:

### 5.1 Simulated Runtime Environment

* Instantiate the chosen WASM runtime engine (e.g., Wasmtime) within the sandbox.
* Create a runtime `Store` configured with the simulated `gas_limit` from the request as the initial fuel amount.

### 5.2 HFI Stub Implementation

Implement host functions (stubs) that mimic the behavior of the real HFI (`specs/interfaces/11-hfi.md`) within the simulated environment. These stubs are linked to the WASM instance:

* **State (`nova_state_v1::*`):** Use an in-memory `HashMap<Vec<u8>, Vec<u8>>` within the simulation context to store key-value pairs. Ignore `domain_hash`. Implement `read`, `write`, `exists` logic against this map. Handle `NOVA_ERR_KEY_NOT_FOUND`.
* **Context (`nova_tx_v1::*`, `nova_env_v1::*`):** Return values provided in the request's `context` object (sender, value, block_number, timestamp) or use reasonable defaults. `get_self_address` can return a fixed simulated address. `get_gas_left` returns the current fuel level from the WASM runtime `Store`.
* **Events (`nova_contract_v1::emit_event`):** Append the received topics (list of hex strings) and data (hex string) to a list of event objects within the simulation context. Check against reasonable size limits.
* **Revert (`nova_contract_v1::revert`):** Read the revert message from WASM memory. Signal a `RuntimeRevert` condition to the simulation host, storing the message. Halt WASM execution.
* **Crypto (`nova_crypto_v1::*`):** Use standard cryptographic libraries available in the backend environment (e.g., Rust crates for Keccak-256, Blake3) to perform the actual hashing.
* **Debug (`nova_debug_v1::print`):** Append the message read from WASM memory to a string buffer representing simulation output.

### 5.3 Gas Simulation

* The WASM runtime's fuel mechanism handles gas consumption for WASM instructions.
* The HFI stubs MUST deduct approximate gas costs for their operations (e.g., fixed cost for reads, higher cost + per-byte cost for writes/emits) from the WASM `Store`'s fuel counter before executing their logic. These costs are simplified estimates for playground feedback, not canonical protocol costs.

### 5.4 Execution Flow & Outcome Handling

1.  **Instantiate:** Load and instantiate the compiled `.wasm` module, linking the HFI stubs.
2.  **Lookup Entry Point:** Find the exported WASM function corresponding to the `entry_point` specified in the request.
3.  **Execute:** Call the entry point function. Handle potential ABI encoding/decoding for simple arguments/return values if the entry point requires them (initially focus on parameterless entry points).
4.  **Process Outcome:**
    * **Success:** Execution completes normally. Capture the return value (if any), the accumulated debug print output, and the list of emitted events. Calculate `gas_used` (`gas_limit` - fuel consumed). Format and return the success response.
    * **Failure (Trap):** Catch WASM traps:
        * `OutOfFuel`: Format and return an `OutOfGas` error response. `gas_used` equals `gas_limit`.
        * `Unreachable`: Format and return a `RuntimePanic` error response. `gas_used` equals `gas_limit`.
        * Other Traps (e.g., Memory OOB): Treat as `RuntimePanic`. `gas_used` equals `gas_limit`.
    * **Failure (HFI Revert):** If `nova_contract::revert` was called, format and return a `RuntimeRevert` error response, including the captured message. Calculate `gas_used` based on fuel consumed before the revert.
    * **Failure (Sandbox Limit):** If execution exceeds sandbox resource limits (time, memory), terminate and return `Timeout` or `SandboxError`.

## 6. Sharing Service Logic (Optional)

If sharing functionality is implemented:

* **`/share` Endpoint:** Receives code snippet. Generates a unique ID. Stores the `(ID, code)` mapping in a persistent key-value store (e.g., database, Redis with TTL). Returns the unique ID or a full URL containing it.
* **`/load` Endpoint:** Receives an ID via query parameter. Retrieves the corresponding code snippet from the store. Returns the code to the frontend.

## 7. Limitations

The playground backend simulation has inherent limitations compared to a real blockchain environment:

* **Project Size:** Suitable only for small, single-file snippets or very simple multi-file structures manageable via the API. Cannot handle complex projects with external dependencies defined in `Twilight.toml`.
* **Runtime Fidelity:** The HFI stubs and state simulation are simplified. Behavior related to complex consensus, P2P networking, accurate gas costs, historical state, and intricate transaction interactions cannot be replicated.
* **Dependencies:** Does not support external package dependencies.