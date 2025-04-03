# Twilight Compiler CLI (`twilightc`) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the command-line interface (CLI) for the core Twilight compiler, `twilightc`. This tool serves as the primary interface for developers to invoke the compiler's functions for checking, building, and potentially testing Twilight projects defined by a `Twilight.toml` manifest file.

### 1.2 Scope

This specification defines the core commands, common flags, expected behavior, and integration points for `twilightc`. It focuses on the compiler's direct interface. Higher-level package management operations (like adding dependencies or publishing) are typically handled by a separate package manager tool that orchestrates `twilightc`.

## 2. Design Philosophy

* **Integrated Compiler Interface:** Provides direct access to core compiler actions (checking, building).
* **Configuration Driven:** Relies on the `Twilight.toml` manifest file for project structure, dependencies, and build target definitions.
* **Clear Output:** Aims to provide informative progress messages, warnings, and actionable error messages with precise source locations.
* **Standard Conventions:** Follows common practices for command-line build tools regarding commands and flags.

## 3. Core Commands

*(Note: Assumes execution from within a directory containing `Twilight.toml` or a subdirectory thereof).*

### 3.1 `twilightc check`

* **Action:** Performs a rapid analysis of the current package/crate, including parsing, name resolution, type checking, implicit borrow checking, and other core semantic analyses defined in the middle end. It stops *before* backend code generation and extensive optimization.
* **Purpose:** To provide fast feedback on compilation errors and warnings during development without the overhead of a full build. Essential for iterative coding cycles.
* **Output:** Prints warnings and errors to the standard error stream, referencing source locations (`file:line:col`). Exits with a non-zero status code if any errors are detected.

### 3.2 `twilightc build`

* **Action:** Compiles the current package/crate, including resolving and potentially building dependencies, generating executable bytecode (e.g., WASM) and the associated metadata (`.meta`) file.
* **Purpose:** To create the final deployable or usable artifacts from the Twilight source code.
* **Common Flags:**
    * `--target <TARGET>`: Specifies the compilation target architecture/format (e.g., `wasm32-nova-v1`). Defaults to the primary supported target (likely WASM).
    * `--release`: Builds with optimizations enabled and potentially fewer debug checks/info. Results in potentially faster but larger/longer-compiling code compared to the default debug build.
    * `--emit <list>`: Specifies a comma-separated list of artifacts to output (e.g., `bytecode`, `meta`, `tir`, `ast`). Default typically includes `bytecode` and `meta`.
    * `--out-dir <directory>` or `-o <directory>`: Specifies the output directory for generated artifacts. Defaults to a standard location (e.g., `target/debug/` or `target/release/`).
    * `--contract <name>`: Specifies which contract target(s) defined in `Twilight.toml`'s `[[contract]]` sections to build, if multiple exist. Default behavior might be to build all defined contract targets.
* **Output:** Writes the specified artifact files (e.g., `.wasm`, `.meta`) to the output directory. Prints progress information, warnings, and errors to the standard error stream. Exits with a non-zero status code on error.

### 3.3 `twilightc test`

* **Action:** Compiles the package's code in a test configuration (enabling `#[cfg(test)]` code) and executes functions annotated as tests (`#[test]`) using an integrated test runner and simulated execution environment.
* **Purpose:** To execute unit and integration tests defined within the Twilight project.
* **Framework Integration:** Relies on a standard Twilight testing framework (API defined in `specs/tooling/xx-test-framework.md`) for test discovery (`#[test]`), execution context (simulated runtime, mock HFI), and assertion macros (`assert!`, `assert_eq!`). `twilightc test` orchestrates the compilation and invocation of this framework.
* **Output:** Reports test discovery, execution progress, and pass/fail results to the standard output/error stream. Provides details for failed tests (assertion failures, panics, unhandled errors). Exits with a non-zero status code if any tests fail.
* **Filtering:** May accept filter arguments to run only specific tests (e.g., `twilightc test my_test_name`).

### 3.4 `twilightc new <package_name>`

* **Action:** Creates a new directory named `<package_name>` containing a default `Twilight.toml` manifest file and a basic `src/` directory structure (e.g., `src/lib.twl` or `src/contract.twl`). Flags like `--lib` or `--contract` may control the initial template.
* **Purpose:** Provides a standard way to quickly bootstrap new Twilight projects.

### 3.5 `twilightc init`

* **Action:** Initializes a Twilight package structure (default `Twilight.toml`, `src/`) within the *current* directory, assuming it is not already part of a Twilight package. Flags like `--lib` or `--contract` may control the initial template.
* **Purpose:** To initialize a Twilight package in an existing directory.

## 4. Integration with `Twilight.toml`

All `twilightc` commands operate within the context of a Twilight package defined by a `Twilight.toml` manifest file found in the current or a parent directory. The compiler reads this file to determine:

* Package metadata (name, version).
* Dependency information (required for name resolution and linking during builds).
* Compilation targets (library path, contract paths and names defined in `[lib]` or `[[contract]]` sections).

## 5. Dependency Management Role (High Level)

While a separate package manager tool typically handles user-facing dependency operations (adding, removing, updating), `twilightc build`, `check`, and `test` rely on dependency information being available. They expect dependencies (source code and potentially compiled metadata) listed in `Twilight.toml` and resolved (e.g., via a lockfile) to be accessible during compilation for analysis and linking. The build process ensures dependencies are compiled before the current package.

## 6. Output and User Experience

* **Diagnostics:** Errors and warnings generated by the compiler MUST reference specific source locations (`file:line:col`) and provide clear, actionable messages.
* **Progress:** Commands involving significant work (like `build`) should provide informative progress updates.
* **Consistency:** Commands and flags should follow standard CLI conventions for usability.