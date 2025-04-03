# Twilight Language & NOVA Project

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

**Twilight** is a next-generation smart contract language designed for safety, expressiveness, and performance on the **NOVA Layer 1 blockchain**. NOVA leverages a unique Conflict-Resolved Commit Set (CRCS) DAG consensus and a Conflict Domain-aware runtime to achieve high throughput via parallel execution. Twilight is built to harness these capabilities.

Inspired by Rust's safety principles but aiming for simpler developer ergonomics, Twilight enforces memory safety implicitly through its compiler (`twilightc`) and targets the **Twilight Intermediate Representation (TIR)**.

## Key Features

* **Memory Safety by Default:** Implicit ownership and borrow checking managed by the compiler aim to prevent common memory errors (dangling pointers, use-after-free, data races) without requiring explicit lifetime annotations.
* **Strong, Static Typing:** Features a robust type system with primitive types (integers, bool, address, hash, etc.), compound types (structs, enums, tuples, vectors, maps), and explicit `Option<T>` / `Result<T, E>` types to prevent null errors and force error handling. Integer overflows/underflows revert by default.
* **Conflict Domain Awareness:** Language and compiler support for static analysis of state access patterns (`#[reads]`, `#[writes]`). This generates metadata used by the NOVA runtime for safe parallel transaction execution.
* **Formal Verification Focus:** Designed with verifiability in mind, including language constructs (`assert_pre`, `assert_post`, `assert_invariant`) and compiler support (VCGen) for generating proof obligations suitable for SMT solvers.
* **Twilight Syntax Modes (TSM):** Supports multiple frontend syntaxes compiling to the same TIR, starting with `twilight-native` (Rust-inspired, domain-focused) and `solidity-mode` (offering familiarity).
* **Intermediate Representation (TIR):** Compiles to TIR-HL and then canonical TIR-SSA, a static single assignment form designed for optimization, analysis (gas, domains), and targeting multiple backends.
* **Targeting WASM:** Primarily compiles to WebAssembly (WASM) for execution within a secure, gas-metered, and deterministic runtime environment on NOVA nodes.
* **Integrated Tooling:** Aims for a comprehensive developer experience with tools like LSP (`twilight-lsp`), package manager (`gloam`), formatter (`twilightfmt`), linter (`twilight-lint`), and debugger support (DAP).

## Architecture Overview

The Twilight compiler (`twilightc`) follows a typical multi-stage pipeline[cite: 1559]:

1.  **Frontend:** Parses source code (handling TSM), performs name resolution and basic type checking, generating High-Level TIR (TIR-HL).
2.  **Middle End:** Lowers TIR-HL to canonical TIR-SSA, performs core analyses (type checking, implicit borrow checking, conflict domains, gas estimation, verification condition generation), runs optimizations.
3.  **Backend:** Translates optimized TIR-SSA into target bytecode (WASM primary) and emits associated metadata (`.meta` file).

## Core Concepts

* **Implicit Memory Safety:** Safety enforced by the compiler without explicit `&`, `&mut`, or `'a` syntax.
* **Conflict Domains:** Static identifiers for state access scopes, enabling parallel execution.
* **TIR (Twilight Intermediate Representation):** The compiler's internal language for analysis and optimization.
* **TSM (Twilight Syntax Modes):** Support for multiple input language syntaxes.
* **CRCS (Conflict-Resolved Commit Sets):** NOVA's DAG-based consensus mechanism.
* **HFI (Host Function Interface):** The secure boundary between WASM contracts and the NOVA runtime.

## Project Structure (Conceptual Monorepo)

This repository contains the core components of the Twilight language and potentially related NOVA elements. A logical structure might resemble:

├── compiler/       # twilightc compiler crates (parser, ast, resolve, typecheck, borrowck, tir, codegen, etc.)
├── crates/         # Core libraries (e.g., core stdlib, nova stdlib, package manager)
├── runtime/        # Components related to the NOVA execution environment (VM interface, HFI stubs, scheduler)
├── specs/          # Detailed blueprint documents
├── examples/       # Example Twilight contracts and code snippets
├── tools/          # Standalone tooling (LSP, Formatter, Linter, DAP server, Playground backend)
├── book/           # Source for "The Twilight Book" documentation
├── .github/        # CI/CD workflows, issue templates
├── LICENSE_APACHE
├── LICENSE_MIT
└── README.md

## Getting Started

Toolchain setup and detailed instructions will be provided closer to the first alpha release.

## Status

**Actively Under Development:** The language specifications and protocol blueprints are largely complete. Implementation of the compiler (`twilightc`), runtime components, standard library, and core tooling is in progress.

**Expect breaking changes** during the initial development phases. This project is pre-alpha.

## License

This project is dual-licensed under the terms of both the MIT license and the Apache License (Version 2.0).

See `LICENSE_MIT` and `LICENSE_APACHE` for details.

## Contact & Community

* **GitHub:** <https://github.com/reinazone>
* **X/Twitter:** <https://x.com/twilightreina>
* **Email:** reina@disroot.org
    
## Contributing

Contributions are welcome! Please see `CONTRIBUTING.md` (when available) for guidelines on how to contribute code, report issues, or suggest improvements.