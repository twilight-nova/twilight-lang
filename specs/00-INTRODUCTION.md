# Twilight Specifications: Introduction

## Overview

This collection of documents provides the official technical specifications for the Twilight programming language, its compiler (`twilightc`), the Twilight Intermediate Representation (TIR), associated developer tooling, verification features, and relevant interface definitions.

These specifications serve as the definitive reference for implementers of the language, compiler, runtime components, and tooling, as well as for developers seeking a precise understanding of Twilight's features and behavior.

## Structure

The specifications are organized into the following main categories within this directory:

* **`/language`**: Defines the core Twilight language itself, including syntax, type system, semantics, memory model, standard library outline, and blockchain feature integration (`twilight-native` focus).
* **`/compiler`**: Details the architecture and implementation strategy for the `twilightc` compiler, covering its frontend, intermediate representation (TIR), middle-end analysis and optimization passes, and backend code generation (primarily targeting WASM).
* **`/interfaces`**: Specifies stable interfaces required for interoperability, such as the compiler metadata (`.meta`) file schema and the Host Function Interface (HFI) expected by compiled contracts.
* **`/tooling`**: Defines the specifications for standard developer tools within the Twilight ecosystem, including the Language Server (LSP), command-line interface, code formatter, linter, debugger support (DAP), and web playground backend.
* **`/verification`**: Covers features supporting formal verification and static analysis, including the assertion language, Verification Condition Generation (VCGen), SMT-LIB output, runtime assertion checking mode, and the vision for formal semantics.

Each numbered document within these directories covers a specific component or aspect of the Twilight system. Refer to the individual files for detailed technical information.

## Goals

The goals of these specifications are to ensure:

* **Clarity:** Provide unambiguous definitions for language features and system components.
* **Consistency:** Ensure all components (compiler, runtime, tools) operate based on a shared, consistent understanding.
* **Correctness:** Serve as the basis for implementing and verifying the language and its toolchain.
* **Stability:** Define stable interfaces for developers and tool builders.

These specifications reflect the finalized design decisions for the current version (v1.0) of the Twilight language and tooling.