# Twilight Debugger Support (DAP) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the architecture and requirements for enabling interactive, source-level debugging of Twilight smart contracts executing on the NOVA platform (in simulation or potentially on specialized nodes). It defines the necessary components: compiler-generated debug information, a required debugging interface from the NOVA Virtual Machine (VM)/Runtime, and a Debug Adapter Protocol (DAP) server (`twilight-dap`) to mediate between IDEs and the runtime.

### 1.2 Target Audience

* Implementation teams building the Twilight Debug Adapter (`twilight-dap`).
* NOVA Runtime/VM implementers (defining and exposing the debug interface).
* Twilight compiler (`twilightc`) implementers (generating debug information).
* IDE/Editor extension developers integrating Twilight debugging support.

### 1.3 Design Philosophy

* **Standard Integration:** Adhere strictly to the Debug Adapter Protocol (DAP) standard to ensure compatibility with a wide range of code editors and IDEs without requiring custom client-side logic.
* **Source-Level Debugging:** Enable developers to set breakpoints, step through execution, and inspect state directly within their original Twilight source code (`.twl` files), regardless of the Twilight Syntax Mode (TSM) used.
* **State Visibility:** Provide mechanisms to inspect the values of local variables, function arguments, and persistent contract `storage` variables during a debug session.
* **Runtime Control:** Offer standard debugging execution controls (continue, pause, step over, step into, step out).

## 2. Core Components & Architecture

Effective debugging requires coordination between three main components:

### 2.1 Compiler Debug Info Generation (`twilightc`)

* **Requirement:** When invoked with debug flags enabled (e.g., default builds or explicit `-g`), `twilightc` **MUST** generate comprehensive debug information alongside the executable bytecode (WASM) and standard metadata (`.meta`).
* **Content Requirements:**
    * **Source Maps:** High-fidelity, accurate mapping between executable bytecode instruction offsets and the corresponding original source code locations (file path, line, column). This mapping must be correct for all supported TSMs (e.g., `twilight-native`, `solidity`).
    * **Variable Information:** Mapping from runtime storage locations (e.g., WASM locals, stack slots, memory addresses representing variables) within specific lexical scopes (function bodies, blocks) back to the original source variable names (`val`/`var`) and their declared Twilight types.
    * **Type Information:** Structural definitions for `struct`s and `enum`s (field/variant names, types) to enable the debugger adapter to interpret raw memory or values and display them in a structured, user-friendly format.
    * **Function Boundaries:** Mapping of bytecode ranges corresponding to distinct source function definitions.
* **Format:** This debug information must be stored in a standardized, parseable format, either embedded within or linked from the `.meta` file (e.g., in the `debug_info` section, potentially using formats like DWARF extensions for WASM or a custom JSON-based format). The Debug Adapter (`twilight-dap`) needs to parse this information.

### 2.2 NOVA Runtime/VM Debugging Interface (Requirements)

* **Requirement:** The NOVA node executable, when run in a specific debugging mode or configured to allow debug attachments, **MUST** expose a stable, programmatic interface allowing an external controller (the `twilight-dap` server) to inspect and control the execution of a target smart contract instance (the "debuggee").
* **Interface Mechanism:** The interface could be implemented via Inter-Process Communication (IPC), a dedicated TCP socket using a custom binary/JSON-RPC protocol, or potentially an internal API if the DAP server runs embedded within the node process (less common).
* **Required Capabilities:** The interface must provide functions or messages equivalent to the following conceptual operations:
    * **Session Management:**
        * `Attach(target)`: Attach to a specific execution context (e.g., identified by transaction hash in simulation). Returns a Session ID.
        * `Detach(SessionID)`: Disconnect from the debug session.
        * `GetStatus(SessionID)`: Query the current execution status (e.g., Running, Paused, Stopped, Terminated).
    * **Execution Control:**
        * `Pause(SessionID)`: Request suspension of execution. Triggers a `StoppedEvent` notification from the VM.
        * `Continue(SessionID)`: Resume execution. Triggers a `ContinuedEvent`.
        * `StepInstruction(SessionID)`: Execute a single low-level VM instruction. Triggers `StoppedEvent(reason='step')`.
        * `StepInto(SessionID)`: Execute until the next source line/statement, stepping into function calls. Triggers `StoppedEvent(reason='step')`.
        * `StepOver(SessionID)`: Execute until the next source line/statement, stepping over function calls. Triggers `StoppedEvent(reason='step')`.
        * `StepOut(SessionID)`: Execute until the current function returns. Triggers `StoppedEvent(reason='step')`.
    * **Breakpoint Management:**
        * `SetBreakpoints(SessionID, bytecode_offsets)`: Set breakpoints at specified executable bytecode offsets. Returns success/failure per offset. Runtime pauses and sends `StoppedEvent(reason='breakpoint')` when hit.
        * `ClearBreakpoints(SessionID, bytecode_offsets)`: Remove specified breakpoints.
    * **State Inspection (at Paused state):**
        * `GetStackTrace(SessionID)`: Retrieve the current call stack (function identifiers, current bytecode offset per frame).
        * `GetScopes(SessionID, frameId)`: Identify logical scopes within a stack frame (e.g., "Locals", "Storage").
        * `GetVariables(SessionID, scopeReference)`: Retrieve variable information (name, type string, value string, expandable reference) within a specific scope, using compiler debug info to map runtime state (locals, stack, memory) to source variables.
        * `ReadWasmMemory(SessionID, offset, count)`: Read raw bytes from the WASM instance's linear memory.
        * `GetContractStorage(SessionID, storage_key_bytes)`: **Crucially**, provide a mechanism to securely read the current value associated with a specific persistent storage key for the contract instance being debugged.

### 2.3 Twilight Debug Adapter (`twilight-dap`)

* **Role:** A standalone server implementing the Debug Adapter Protocol (DAP). It acts as the intermediary between the IDE (DAP Client) and the NOVA Runtime/VM Debugging Interface (Debuggee Controller).
* **Implementation:** Typically a Rust application using DAP libraries.
* **Core Logic:** Manages the debug session state, translates DAP requests into VM Debug Interface commands (using loaded debug info), and translates VM events/state back into DAP responses/events for the IDE.

## 3. Debug Adapter Protocol (DAP) Implementation Logic

`twilight-dap` implements handlers for standard DAP requests, interacting with the compiler debug info and the VM debug interface:

* **`initialize` / `launch` / `attach`:** Establish connection to the target NOVA process/VM using the `Attach` command of the VM Debug Interface. Load the compiler-generated debug information for the relevant contract code. Respond to the client with capabilities.
* **`setBreakpoints`:** Receive source file path and line numbers from the client. Use the loaded Source Maps (from debug info) to translate these source locations into corresponding executable bytecode offsets. Call the VM `SetBreakpoints` command with these offsets. Respond to the client with verified breakpoint locations.
* **`configurationDone`:** Prepare the VM to start execution if the session was launched (as opposed to attached).
* **`continue` / `next` / `stepIn` / `stepOut` / `pause`:** Translate directly to the corresponding commands in the VM Debug Interface (`Continue`, `StepOver`, `StepInto`, `StepOut`, `Pause`).
* **VM `StoppedEvent` Handling:** Upon receiving a notification from the VM that execution has stopped (e.g., breakpoint hit, step completed, pause requested):
    1.  Query the VM `GetStackTrace` command.
    2.  Send the DAP `StoppedEvent` to the client, including the reason (breakpoint, step, etc.) and thread ID.
* **`stackTrace`:** Receive DAP request. Use the stack information previously obtained (or query `GetStackTrace` again). Use Source Maps to map bytecode offsets in stack frames back to source file paths and line numbers. Return the DAP `StackTraceResponse`.
* **`scopes`:** Receive DAP request for a given stack frame ID. Call VM `GetScopes`. Return DAP `ScopesResponse` (typically including "Locals" and "Storage").
* **`variables`:** Receive DAP request with a `variablesReference` (identifying a scope from `scopes` or an expandable variable). Call VM `GetVariables` with this reference. The VM uses the compiler's debug info (Variable Info, Type Info) to map runtime locations (locals, stack, memory) to source variable names, types, and formatted values. For the "Storage" scope, the DAP server might initially list storage variable names (from debug info) and then trigger VM `GetContractStorage` calls when the user expands a specific storage variable in the IDE. Return the DAP `VariablesResponse`.
* **`evaluate` (Optional):** Handling expression evaluation during debugging requires significant support from the VM/Runtime Debug Interface and careful implementation in the DAP server. This is considered an advanced feature, likely deferred beyond v1.0.
* **`disconnect` / `terminate`:** Call VM `Detach`. Clean up internal session state.

## 4. Core Debugging Features (Target v1.0)

* Set/Clear Line Breakpoints
* Step Over / Step Into / Step Out (Source Line Level)
* Continue / Pause Execution
* View Call Stack (with source locations)
* Inspect Local Variables and Function Arguments (with source names, types, values)
* Inspect Contract `storage` Variables (values at paused state)
* View output from debug print statements

## 5. Twilight Syntax Mode (TSM) Handling

Source-level debugging across different TSMs relies entirely on the accuracy and completeness of the **Source Maps** generated by the `twilightc` compiler. The compiler backend MUST produce correct mappings from bytecode offsets back to the original `.twl` file lines/columns, regardless of whether the source used `twilight-native` or `solidity` syntax. The DAP server consumes these maps transparently.