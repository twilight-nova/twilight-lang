# Twilight Compiler Architecture Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document defines the high-level architecture and processing stages of the Twilight compiler (`twilightc`). It outlines the flow from Twilight source code input to the final emission of executable bytecode (primarily WebAssembly) and associated metadata (`.meta` file). This specification establishes the major components and their core responsibilities.

### 1.2 Scope

This specification covers the overall structure and data flow within the compiler. Detailed specifications for individual stages (Frontend, Middle End Passes, Backend Code Generation) and internal representations (AST, TIR) are provided in separate documents.

## 2. Design Philosophy & Goals

The architecture of `twilightc` is guided by the following principles:

* **Correctness & Safety:** The primary goal is to accurately translate Twilight source code according to its defined semantics, rigorously enforcing the language's type system and implicit memory safety rules.
* **Modularity:** The compiler is designed with distinct pipeline stages communicating through well-defined interfaces (e.g., Abstract Syntax Tree (AST), Twilight Intermediate Representation (TIR)). This facilitates testing, maintenance, potential parallelism, and future extensibility.
* **NOVA Integration:** Analysis passes required for NOVA's features (Conflict Domains, Gas Estimation, Pipeline Information, Verification Hooks) are integrated throughout the compilation pipeline.
* **Developer Experience (DX):** The architecture supports clear, actionable error messages, especially for complex issues related to memory safety or conflict domains. Rich metadata generation (`.meta` file, source maps) enables effective downstream tooling (LSP, debuggers).
* **Performance:** Design choices facilitate effective optimization passes operating on the TIR and support efficient code generation by the backend.
* **Extensibility:** The modular structure and intermediate representations are designed with the potential for future extensions, such as alternative backends or analysis plugins.

## 3. Compiler Pipeline Overview

The compilation process proceeds through three major stages: Frontend, Middle End, and Backend.

[Twilight Source Code (.twl, Multiple Syntax Modes)]
|
V
+--------------+
|   Frontend   | ---> [AST] ---> [TIR-HL]
+--------------+
| (Lexing, Parsing, Basic Checks, TIR-HL Lowering)
V
+--------------+
|  Middle End  | ---> [Optimized TIR-SSA]
+--------------+
| (TIR-SSA Lowering, Analysis, Optimization)
V
+--------------+
|   Backend    | ---> [Target Bytecode (WASM) + .meta File]
+--------------+
| (Code Generation, Metadata Emission)

### 3.1 Frontend Stage

* **Input:** Twilight source code files (`.twl`). Reads the `#![syntax = "..."]` directive to handle different Twilight Syntax Modes (TSM), such as `twilight-native` and `solidity`.
* **Responsibilities:**
    * **Lexing:** Converts source text into a stream of tokens.
    * **Parsing:** Constructs an Abstract Syntax Tree (AST) based on the grammar of the selected syntax mode. Reports syntax errors. For non-native modes like `solidity`, this stage involves parsing the source using a relevant parser (e.g., Solang) and transforming the resulting AST into the canonical Twilight AST.
    * **Initial Semantic Analysis:** Performs name resolution (linking identifiers to definitions) and basic type checking/inference.
    * **TIR-HL Generation:** Lowers the AST into a High-Level Twilight Intermediate Representation (TIR-HL), preserving source location information (`srcloc`) and initial metadata from annotations.
* **Output:** TIR-HL representation.

*(Details in `specs/compiler/06-frontend.md`)*

### 3.2 Middle End Stage (TIR Processing)

* **Input:** TIR-HL representation from the Frontend.
* **Responsibilities:**
    * **TIR-SSA Lowering:** Transforms TIR-HL into the canonical Static Single Assignment (SSA) form (TIR-SSA), using virtual registers, basic blocks, terminators, and phi nodes. Resolves higher-level constructs into lower-level TIR instructions.
    * **Core Analyses:** Performs in-depth analysis passes on TIR-SSA, including:
        * Final Type Checking.
        * Implicit Borrow Checking & Lifetime Analysis (Enforces memory safety).
        * Conflict Domain Analysis (Computes aggregated read/write sets).
        * Gas Estimation (Annotates TIR with costs).
        * Pipeline Analysis (Validates pipeline hints).
        * Verification Condition Generation (Extracts proof obligations from assertions).
    * **Optimization:** Executes sequences of optimization passes to improve performance, reduce gas costs, and minimize code size, while preserving program semantics. Includes standard compiler optimizations (DCE, Inlining, etc.) and potentially NOVA-specific optimizations.
* **Output:** Optimized, validated TIR-SSA, annotated with comprehensive metadata.

*(Details in `specs/compiler/07-tir.md` and `specs/compiler/08-middle-end-passes.md`)*

### 3.3 Backend Stage

* **Input:** Optimized TIR-SSA representation from the Middle End.
* **Responsibilities:**
    * **Code Generation:** Translates TIR-SSA into executable bytecode for the selected target (e.g., `--target wasm`). This includes instruction selection, mapping TIR operations to target opcodes (including HFI calls for WASM), and handling register allocation or stack management according to the target's ABI.
    * **Metadata Emission:** Collects metadata generated throughout the pipeline (Conflict Domains, gas estimates, source maps, function ABIs, verification info, etc.) and serializes it into the standardized `.meta` file format.
* **Output:** Target executable bytecode (e.g., `.wasm` file) and the corresponding metadata file (e.g., `.meta`).

*(Details in `specs/compiler/09-backend-wasm.md` and `specs/interfaces/10-meta-schema.md`)*