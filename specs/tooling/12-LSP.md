# Twilight Language Server Protocol (LSP) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the architecture, core features, and implementation strategy for the Twilight Language Server (`twilight-lsp`). The server implements the standard Language Server Protocol (LSP) to provide integrated development environment (IDE) features for Twilight, such as diagnostics, code completion, hover information, and code navigation, within compatible code editors (e.g., VS Code, Neovim).

### 1.2 Target Audience

* Implementation teams building the `twilight-lsp` server.
* Editor extension developers integrating with `twilight-lsp`.
* Twilight compiler (`twilightc`) developers (regarding shared library requirements).

### 1.3 Design Philosophy

* **Responsiveness & Accuracy:** Provide fast, incremental analysis feedback that accurately reflects Twilight language semantics and compiler behavior.
* **Consistency with Compiler:** Maximize consistency by leveraging the same core parsing and analysis libraries used by the `twilightc` compiler.
* **TSM Aware:** Fully support Twilight Syntax Modes (TSM), providing correct parsing, diagnostics, and features based on the active `#![syntax = "..."]` directive.
* **Feature Rich:** Implement standard, high-value LSP features expected for a modern development experience.
* **Robustness:** Handle concurrent requests and incremental document changes efficiently.

## 2. Architecture & Core Components

* **Execution Model:** `twilight-lsp` operates as a standalone background server process. It communicates with the editor/client using the standard Language Server Protocol's JSON-RPC messages, typically over standard input/output (stdio).
* **Implementation Language:** Rust.
* **LSP Framework:** Utilizes standard Rust libraries for handling LSP communication and protocol types (e.g., `tower-lsp`, `lsp-types`).
* **Core Internal Components:**
    * **LSP Server Endpoint:** Manages JSON-RPC communication, message serialization/deserialization, request routing, and server lifecycle.
    * **Document Manager:** Tracks the state of open text documents (URI, content, version), handling `textDocument/didOpen`, `didChange`, `didSave`, and `didClose` notifications. Provides access to current document text for analysis.
    * **Analysis Engine:** The central component responsible for parsing and semantic analysis.
        * *Shared Crates:* **Critically relies on shared library crates** from the `twilightc` compiler project (e.g., `twilightc_parser`, `twilightc_ast`, `twilightc_resolve`, `twilightc_typecheck`, `twilightc_borrowck`, `twilightc_diagnostics`).
        * *TSM Handling:* Detects the syntax mode for each document and invokes the appropriate shared parser.
        * *Semantic Analysis:* Performs name resolution, type checking, implicit borrow checking, and potentially linting using the shared compiler analysis passes.
        * *State Management:* Stores analysis results (ASTs, symbol tables, type information, diagnostics) associated with document versions, ideally supporting incremental updates.
        * *Query Interface:* Provides interfaces for LSP feature handlers to query analysis results (e.g., type at position, definition location, available completions).
    * **Indexer (Optional but Recommended):** A background component that builds and maintains a project-wide index of symbol definitions and potentially references. This enables faster responses for `textDocument/definition` and `textDocument/references` without requiring full project re-analysis on every request.

## 3. Implementation Strategy

* **Shared Compiler Crates (Mandatory):** The `twilightc` compiler's core logic (parsing, AST definition, semantic analysis passes, type system implementation, diagnostic reporting) **MUST** be factored into reusable library crates. `twilight-lsp` **MUST** use these shared crates for its analysis capabilities to ensure consistency between editor feedback and actual compilation results. Direct invocation of the `twilightc` binary for analysis is typically too slow for a responsive LSP experience.
* **Incremental Analysis:** The Analysis Engine should be designed to perform incremental parsing and analysis where possible. On a `textDocument/didChange` notification, it should ideally re-parse only the affected file and re-run semantic analysis passes only on the parts of the project directly or indirectly affected by the change, leveraging cached results for unchanged parts. Frameworks like `salsa` or custom dependency tracking can facilitate this.
* **Asynchronous Processing:** Utilize asynchronous programming (e.g., Rust's `async`/`await` with `tokio`) to handle concurrent LSP requests. Analysis tasks, especially project-wide operations like initial indexing or full checks, should run in background tasks to avoid blocking time-sensitive requests like completion or hover.
* **TSM Handling:**
    * The Document Manager or Analysis Engine detects the `#![syntax = "..."]` directive per file.
    * The Analysis Engine uses the appropriate parser (Native or Solidity->Twilight AST Transformation) from the shared compiler crates.
    * Semantic analysis operates on the canonical Twilight AST.
    * LSP feature implementations (e.g., completion) provide context-aware results appropriate for the detected syntax mode.

## 4. Core LSP Feature Implementation

### 4.1 Diagnostics (`textDocument/publishDiagnostics`)

* **Trigger:** Generated upon `textDocument/didOpen`, `didSave`, or after a debounced `textDocument/didChange`.
* **Process:**
    1.  Retrieve the current document content.
    2.  Invoke the Analysis Engine to perform a full analysis chain (Lex, Parse, [AST Transform], Resolve, Type Check, Borrow Check, Lint Check) on the document and potentially affected dependencies, using incremental analysis and caching where possible.
    3.  Collect all `Diagnostic` objects (errors, warnings) produced by the shared compiler/linter crates.
    4.  Convert these internal diagnostics into the LSP `Diagnostic` format (mapping source `Span` to LSP `Range`, severity levels, messages).
    5.  Publish the complete set of diagnostics for the document URI using the `textDocument/publishDiagnostics` notification, replacing any previous diagnostics for that file.

### 4.2 Code Completion (`textDocument/completion`)

* **Trigger:** LSP request from the client, usually triggered by typing, providing cursor position.
* **Process:**
    1.  Analyze the code context surrounding the cursor position to determine the expected kind of completion (e.g., variable, function name, type name, keyword, struct field, method).
    2.  Query the Analysis Engine for symbols visible in the current scope (local bindings, function parameters, module items via `use`, prelude items).
    3.  If completing after a `.` (e.g., `my_struct.`), determine the type of the expression before the dot and query the Analysis Engine for its fields or implemented trait methods.
    4.  If completing after `::` (e.g., `MyEnum::`), query for associated items or variants.
    5.  Filter candidates based on context, visibility rules, and the characters already typed.
    6.  Generate a list of LSP `CompletionItem` objects, including label, kind (function, variable, struct, etc.), detail (e.g., type signature), and documentation (from doc comments).
    7.  Provide TSM-aware keyword suggestions.

### 4.3 Hover Information (`textDocument/hover`)

* **Trigger:** LSP request when the user hovers the cursor over a symbol.
* **Process:**
    1.  Identify the symbol (identifier, keyword) at the cursor position.
    2.  Query the Analysis Engine to resolve the symbol to its definition and retrieve its kind, type signature, and associated documentation comments (`///`).
    3.  Format this information (signature, type constraints, documentation) into Markdown.
    4.  Return the formatted content in an LSP `Hover` response.

### 4.4 Go To Definition (`textDocument/definition`)

* **Trigger:** LSP request, typically via a keyboard shortcut or context menu action on a symbol.
* **Process:**
    1.  Identify the symbol under the cursor.
    2.  Query the Analysis Engine (using name resolution results or the indexer) to find the source `Span` of the symbol's definition.
    3.  Convert the definition's `Span` into an LSP `Location` (URI + `Range`).
    4.  Return the `Location`.

### 4.5 Find References (`textDocument/references`)

* **Trigger:** LSP request on a symbol.
* **Process:**
    1.  Identify the symbol under the cursor and resolve it to its definition.
    2.  Query the Indexer (if available) or perform a project-wide analysis to find all source code locations (`Span`s) where this definition is referenced.
    3.  Exclude/include the definition location based on the request's `includeDeclaration` parameter.
    4.  Convert the `Span` of each reference into an LSP `Location`.
    5.  Return a list of `Location`s. *(Note: Full project analysis can be slow without an index).*

### 4.6 Document Symbols (`textDocument/documentSymbol`)

* **Trigger:** LSP request to provide a structural outline of a document.
* **Process:**
    1.  Retrieve the AST for the requested document from the Analysis Engine.
    2.  Perform a top-down traversal of the AST's `Item` nodes (contracts, functions, structs, enums, impls, traits, etc.).
    3.  Construct a hierarchical list of LSP `DocumentSymbol` objects, including name, kind, full `Span`, selection `Span`, and any nested children (e.g., methods within an `impl`).
    4.  Return the list.

### 4.7 Formatting (`textDocument/formatting`)

* **Trigger:** LSP request to format the entire document.
* **Process:**
    1.  Retrieve the full document content.
    2.  Invoke the standard Twilight code formatter (`twilightfmt`, specified in `specs/tooling/14-formatter.md`) as an external process or library. Pass the document content to it.
    3.  Receive the fully formatted text from the formatter.
    4.  Compute the difference between the original and formatted text, representing it as an LSP `TextEdit` (or potentially a list of edits, though a single replace-all edit is often simplest).
    5.  Return the `TextEdit`(s).
    6.  **TSM Handling:** Only attempt formatting if the document's syntax mode is `twilight-native`. For other modes, return null or an empty edit list.

## 5. Integration with `twilightc`

As emphasized in Section 3, `twilight-lsp` **must** reuse the core analysis components (parsing, AST, semantic analysis, type system, diagnostics) implemented as libraries within the `twilightc` codebase. This architectural dependency is essential for ensuring consistency and accuracy of the language server.