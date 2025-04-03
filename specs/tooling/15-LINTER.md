# Twilight Linter (`twilight-lint`) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies `twilight-lint`, the official static analysis tool for the Twilight language. `twilight-lint` inspects Twilight source code or its intermediate representation to detect potential bugs, stylistic inconsistencies, anti-patterns, and deviations from best practices, with a particular focus on smart contract security and gas efficiency relevant to the NOVA platform. It complements the compiler's core type and memory safety checks by enforcing an additional layer of rules and heuristics.

### 1.2 Target Audience

* Implementation teams building `twilight-lint`.
* Twilight compiler (`twilightc`) implementers (for integration).
* Twilight Language Server (`twilight-lsp`) implementers (for integration).
* Twilight developers using the linter to improve code quality.

### 1.3 Design Philosophy

* **Proactive Quality Assurance:** Identify and flag potential issues early in the development cycle, before testing or deployment.
* **Security Focused:** Include specific lints targeting common smart contract vulnerabilities and potential misuse of NOVA-specific features (like Conflict Domains).
* **Configurable:** Allow users to enable/disable specific lint rules and configure their severity levels (error, warning, etc.).
* **Helpful Feedback:** Provide clear, actionable explanations for each reported lint violation, suggesting best practices or potential fixes.
* **Integrated:** Designed for seamless integration into the development workflow via the command line (standalone or integrated with `twilightc`) and within IDEs through the Language Server Protocol (LSP).

## 2. Core Functionality & Implementation Strategy

### 2.1 Tool Name

The official linter tool is named `twilight-lint`.

### 2.2 Analysis Approach

* **Input:** Operates on Twilight source code (`.twl` files or projects).
* **Compiler Infrastructure Reuse:** `twilight-lint` **MUST** reuse the core parsing and semantic analysis infrastructure (AST, type information, name resolution results, diagnostic reporting) provided by the shared `twilightc` compiler libraries. This ensures consistency and provides the necessary semantic context for accurate linting.
* **Analysis Target:** Lint rules primarily operate on the semantically enriched Abstract Syntax Tree (AST) after name resolution and type checking. Some rules might query lower-level representations like TIR for flow-sensitive information if necessary.

### 2.3 Lint Rule Framework

* **Implementation:** Lint rules are implemented individually, adhering to a defined structure or trait within the linter's codebase (e.g., a `LintRule` trait).
* **Rule Execution:** The lint engine traverses the AST (or other relevant IR), invoking the `check` methods of all registered and enabled lint rules for relevant nodes.
* **Extensibility:** The framework is designed with potential future extensibility in mind, allowing for custom, third-party lint rules (though the API stability for this is beyond v1.0).

### 2.4 Reporting

* **Output:** Reports detected violations including:
    * A unique lint rule identifier (e.g., `correctness::unused_variable`).
    * The configured severity level (Error, Warning, Info, Hint).
    * The precise source code location (`Span` mapped to `file:line:col`).
    * A clear explanation of the potential issue and why it's flagged.
    * Optionally, suggestions for fixing the code or links to relevant documentation.
* **Integration:** Diagnostics are presented on the command line or published via LSP to the editor.

## 3. Lint Categories & Initial Rule Set (v1.0)

This initial set provides examples across key categories. The specific rules and their default severity levels will evolve.

### 3.1 Group: `correctness` (Default: Warn or Deny)

* `unused_variable`: Detects `val` or `var` bindings that are never read.
* `unused_import`: Detects `use` statements importing items never used.
* `unreachable_code`: Detects code determined to be unreachable via static analysis.
* `variable_shadowing`: (Default: Allow) Warns when a variable declaration shadows one from an outer scope.

### 3.2 Group: `security` (Default: Warn or Deny)

* `unchecked_external_call_result`: (Default: Deny) Detects external contract calls where the `Result` return value is not checked.
* `reentrancy_read_after_external_call`: (Default: Warn) Detects patterns where state is read after an external call within the same function, potentially indicating a Checks-Effects-Interactions violation risk. (Heuristic).
* `timestamp_dependence`: (Default: Warn) Warns on direct use of `env.timestamp()` in conditional logic affecting state or value transfer.
* `panic_in_function`: (Default: Warn) Warns on explicit `panic!` calls, suggesting `require` or `Result` for application errors.
* `missing_access_control`: (Default: Warn) Heuristically warns if `pub` functions modifying potentially sensitive state lack apparent `require` checks on `ctx.sender()` or similar.
* `integer_wrapping_math`: (Default: Warn) Warns on use of `wrapping_*` arithmetic, suggesting a comment justifying the intended wrapping behavior.

### 3.3 Group: `style` (Default: Warn or Allow)

* `non_camel_case_types`: Checks `struct`, `enum`, `trait`, `type` names use `UpperCamelCase`.
* `non_snake_case_functions`: Checks `fn` names use `snake_case`.
* `non_snake_case_variables`: Checks `val`/`var` binding names use `snake_case`.
* `constant_naming`: Checks `const` item names use `SCREAMING_SNAKE_CASE`.
* `todo_comment`: (Default: Allow) Detects `// TODO` or `// FIXME` comments.
* `unnecessary_mut`: Detects `var` bindings that are never reassigned.

### 3.4 Group: `performance` / `gas` (Default: Warn or Allow)

* `storage_read_in_loop`: Detects reads from `storage` inside loops. Suggests caching the value in a local `val` if it doesn't change within the loop.
* `large_struct_copy`: (Default: Allow) Heuristically warns if large non-`Copy` structs are frequently passed/returned by value. (Less critical due to implicit borrow model).

### 3.5 Group: `nova` (Nova-specific Best Practices)

* `overly_broad_domain`: (Default: Warn) Warns on explicit wildcard annotations (`#[writes("map:*")]`) where more specific domains might improve parallelism.

## 4. Twilight Syntax Mode (TSM) Handling

* `twilight-lint` operates on the canonical Twilight AST generated by the frontend, making most correctness and security rules applicable regardless of the original source syntax (`native` or `solidity`).
* Style-related rules (e.g., naming conventions) primarily apply only when the source TSM is `twilight-native`. Rules can internally check the source mode if necessary.

## 5. Configuration

* **Configuration File:** Lint rules and their severity levels are configured via the project's `Twilight.toml` file within a dedicated `[lints]` section (or potentially a separate `.twilight-lint.toml` file). `Twilight.toml` is the preferred location initially.
* **Severity Levels:** Users can configure the reporting level for individual rules or groups using `allow`, `warn`, or `deny`. `deny` typically causes builds/checks to fail.
    ```toml
    [lints]
    # Example configuration
    allow = ["style::todo_comment", "correctness::variable_shadowing"]
    deny = ["security::unchecked_external_call_result"]
    # Other rules default to 'warn' or their group default
    ```
* **Rule Groups:** Rules are categorized into groups (`correctness`, `security`, `style`, `performance`, `nova`). Default levels can be set per group. Specific rule settings override group settings.
    ```toml
    [lints.groups]
    # Example group defaults
    security = "deny"
    style = "allow"
    # correctness, performance, nova default to 'warn' implicitly or explicitly
    ```

## 6. Integration

* **Command Line Interface (CLI):**
    * Standalone: `twilight-lint <path/to/project>` can be run independently.
    * Compiler Integration: Invoked as part of `twilightc check` or `twilightc build` using a flag (e.g., `--lints` or implicitly enabled). Compiler output includes lint diagnostics alongside compilation errors/warnings.
* **Language Server Protocol (LSP):**
    * `twilight-lsp` (`specs/tooling/12-lsp.md`) integrates the linter engine as a library.
    * Lints are run analysis is performed (e.g., on save or change).
    * Lint diagnostics are published to the editor alongside compiler diagnostics via `textDocument/publishDiagnostics`.