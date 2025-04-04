# Twilight Compiler Metadata (`.meta`) File Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the standard schema for the `.meta` file generated by the Twilight compiler (`twilightc`). This file contains verified, aggregated metadata derived from compiler analysis of the source code and intermediate representations (TIR). It serves as a crucial interface for various components of the NOVA ecosystem, including the runtime/consensus layer, debuggers, indexers, block explorers, verification tools, and off-chain interaction libraries.

### 1.2 Target Audience

* Twilight Compiler Implementers (`twilightc` backend)
* NOVA Runtime/Consensus Implementers
* Developer Tooling Implementers (Debuggers, Explorers, Indexers, Analysis Tools)
* Smart Contract Auditors and Developers (for understanding compiled artifact metadata)

### 1.3 Philosophy

The `.meta` file provides a stable, machine-readable, comprehensive, and versioned format containing essential metadata required for safe execution, consensus validation, tooling integration, and developer understanding. Clarity, completeness, and standardization are key goals.

## 2. Format and Versioning

* **Primary Format:** JSON (JavaScript Object Notation). Chosen for its human-readability, widespread library support, and ease of parsing.
* **File Extension:** `.meta`
* **Optional Binary Format:** MessagePack (`.meta.mpk`). May be offered as a more compact alternative for performance-sensitive contexts, providing an equivalent schema. This specification focuses on the JSON structure.
* **Schema Versioning:** The root object of the `.meta` file **MUST** contain a `schema_version` field indicating the version of the `.meta` schema itself. This allows consumers to handle potential future changes.
    ```json
    {
      "schema_version": "1.0",
      // ... rest of metadata ...
    }
    ```

## 3. Top-Level Schema Structure

The root JSON object contains the following top-level keys:

```json
{
  "schema_version": "1.0",
  "source_info": { /* See Section 4.1 */ },
  "bytecode_info": { /* See Section 4.2 */ },
  "abi": [ /* See Section 4.3 */ ],
  "functions": { /* See Section 4.4 */ },
  "events": { /* See Section 4.5 */ },
  "verification_info": { /* See Section 4.6 */ } | null,
  "debug_info": { /* See Section 4.7 */ } | null
}

4. Detailed Section Schemas
4.1 source_info

Contains information about the original source code compiled.

    Schema:

        "source_info": {
      "files": [
        {
          "path": "relative/path/to/source.twl", // Path relative to package root
          "hash_type": "sha256",                 // Hashing algorithm used
          "hash": "0x..."                        // Hex-encoded hash of the source file content
        }
        // ... more entries for other primary source files ...
      ],
      "compiler_version": "twilightc x.y.z" // Version string of the compiler used
    }

4.2 bytecode_info

Contains information about the compiled executable bytecode artifact (e.g., WASM).

    Schema:

    "bytecode_info": {
  "file": "contract_name.wasm",       // Relative path to the generated bytecode file
  "target": "wasm32-nova-v1",        // Target architecture/platform identifier
  "hash_type": "sha256",             // Hashing algorithm used
  "hash": "0x..."                     // Hex-encoded hash of the bytecode file content
}

4.3 abi (Application Binary Interface)

Provides a consolidated ABI definition, useful for off-chain interaction libraries. Follows conventions similar to Ethereum ABI JSON format where applicable.

    Schema: An array [] containing objects describing constructors, functions, and events.
        Constructor Entry:

        {
  "type": "constructor",
  "inputs": [
    // Array of ABI Input objects, e.g.:
    { "name": "initialAdmin", "type": "address", "internalType": "address" }
  ],
  "stateMutability": "payable" | "nonpayable" // Based on init function's #[payable]
}

Function Entry:

{
  "type": "function",
  "name": "humanReadableName",         // Optional: Original function name
  "mangled_name": "mangled::name::...", // Key used in "functions" map
  "inputs": [
     // Array of ABI Input objects
     { "name": "recipient", "type": "address", "internalType": "address" }
  ],
  "outputs": [
     // Array of ABI Output objects
     { "name": "success", "type": "bool", "internalType": "bool" }
  ],
  "stateMutability": "payable" | "nonpayable" // Based on function's #[payable]
}

Event Entry:

{
  "type": "event",
  "name": "Transfer",                        // Event name identifier
  "signature_string": "Transfer(address,address,u256)", // Key used in "events" map
  "inputs": [
    // Array of ABI Event Input objects
    { "indexed": true, "name": "from", "type": "address", "internalType": "address" },
    { "indexed": false, "name": "amount", "type": "uint256", "internalType": "u256" }
  ],
  "anonymous": false
}

ABI Types: Standard ABI type strings (uint256, address, bool, bytes, string, tuple, bytes32, arrays T[] or T[N], etc.). internalType specifies the original Twilight type.

4.4 functions (Per-Function Metadata)

Contains detailed metadata for each compiled function (public and internal), keyed by the function's unique mangled name. Required by runtime and consensus layers.

    Schema: An object {} where keys are mangled function names.
        Per-Function Value:

        "<mangled_func_name>": {
  "signature": "humanReadableName(argType1,argType2)", // Optional: For display/debugging
  "is_init": true | false,                       // Indicates if this is the contract constructor
  "is_payable": true | false,                      // Derived from #[payable] annotation
  "gas_estimate_static": 12345,                    // Aggregated static gas estimate (base cost)
  "aggregate_read_domains": [                      // Set of domain hashes read (transitively)
    { "hash": "0xHash1...", "domain": "optional.human.readable.domain:key" } // Domain string is optional
  ],
  "aggregate_write_domains": [                     // Set of domain hashes written (transitively)
    { "hash": "0xHash3...", "domain": "optional.domain:*" }
  ],
  "pipeline_hint": "optional_hint_string" | null, // Developer hint from #[pipeline]
  "is_recursive": true | false | null,            // Flag indicating recursion (null if not analyzed/applicable)
  "verification_condition_ids": [ "vc_id_1", "vc_id_2" ] | null // IDs linking to verification_info section
}

4.5 events (Per-Event Metadata)

Contains detailed metadata for each defined event, keyed by the event's canonical signature string. Required for off-chain decoding and indexing.

    Schema: An object {} where keys are canonical event signature strings (e.g., "Transfer(address,address,u256)").
        Per-Event Value:

        "<EventName(Type1,Type2,...)>": {
  "signature_hash": "0xTopic0Hash...",             // Keccak-256 hash of the signature string (implicit first topic)
  "human_name": "EventName",                    // Event identifier for display
  "parameters": [
    {
      "name": "param1",
      "type": "address",                          // ABI Type
      "internal_type": "address",                 // Original Twilight Type
      "indexed": true                             // Derived from #[topic]
    },
    {
      "name": "param2",
      "type": "uint256",
      "internal_type": "u256",
      "indexed": false
    }
    // ... parameters in order matching signature string ...
  ]
}

4.6 verification_info

Stores generated Verification Conditions (VCs) if the verification pass was enabled. null if verification info was not generated.

    Schema:

    "verification_info": {
  "format": "SMT-LIB2", // Identifier for the VC format used (e.g., SMT-LIB 2)
  "conditions": {
    // Map where keys are verification_condition_ids referenced in function metadata
    "vc_id_1": {
      "source_location": "file:line:col",           // Location of original assert_* statement
      "description": "Optional text description",   // E.g., "Precondition for function X"
      "formula": "(assert (=> path_condition condition))" // The VC formula itself, or reference
    }
    // ... other VCs ...
  }
} | null

4.7 debug_info (Optional)

Contains information required by debuggers. Can be large and may reference external files. null if not compiled with debug information.

    Schema:

    "debug_info": {
  "source_map": { // Mapping bytecode offsets to source locations
    "format": "standard_source_map_v3_extended", // Identifier for the source map format used
    "data": { /* Embedded sourcemap data */ } | "path/to/external.sourcemap" // Data or reference
  },
  "variable_info": { // Mapping runtime locations/scopes to source variables
    // Detailed schema TBD based on debugger requirements (e.g., DWARF-like info)
    // Example: "scopes": { "scope_id": { "variables": { "var_name": { "type": "u64", "location": ... } } } }
  }
} | null

5. Implementation and Consumption Guidance

    Compiler (twilightc Backend): Responsible for collecting all required metadata during compilation (analysis results, source info, bytecode hash) and serializing it into the .meta file according to this schema. Consistency across sections (e.g., ABI vs. function signatures) is crucial.

    Runtime/Consensus: Must parse the .meta file (primarily JSON format). Critically uses bytecode_info.hash for verification and functions.<mangled_name>.aggregate_read_domains / aggregate_write_domains for validating transaction metadata and controlling parallel execution scheduling/locking.
    
    Tooling: Off-chain libraries use abi and events sections for encoding calls and decoding logs. Debuggers rely heavily on debug_info (especially source_map). Verification tools consume verification_info. Explorers utilize various sections for display purposes.