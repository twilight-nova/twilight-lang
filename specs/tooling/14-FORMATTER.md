# Twilight Formatter (`twilightfmt`) Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document specifies the behavior and style guide for `twilightfmt`, the official code formatting tool for the Twilight language. Its primary goal is to enforce a single, consistent code style for `twilight-native` source files across the entire ecosystem, improving code readability and maintainability while minimizing developer effort spent on stylistic debates.

### 1.2 Design Philosophy

* **Opinionated:** `twilightfmt` defines *the* standard Twilight style with minimal or no configuration options. Consistency across projects is paramount.
* **Idempotent:** Applying `twilightfmt` to already formatted code produces no changes. The operation `format(format(code))` is equivalent to `format(code)`.
* **Readability Optimized:** The enforced style rules prioritize clarity and adhere to established best practices for code layout.
* **Integrated:** Designed for both standalone command-line usage and seamless integration into development workflows via editor LSP integration or pre-commit hooks.
* **Semantics Preserving:** Formatting operations **MUST NOT** alter the semantic meaning or runtime behavior of the code.

## 2. Core Functionality & Implementation

### 2.1 Tool Name

The official formatter tool is named `twilightfmt`.

### 2.2 Input/Output

* **Input:** Accepts `twilight-native` source code files (`.twl`), directories containing such files, or source text from standard input.
* **Output:** Produces the formatted code to standard output or modifies files in-place (when requested via flags).

### 2.3 Parsing & Representation

* **Requirement:** `twilightfmt` must parse `twilight-native` source code into an internal representation that accurately preserves the syntactic structure, including the relative positions of comments and potentially significant whitespace, to enable correct reformatting.
* **Approach:** A Concrete Syntax Tree (CST) or a "lossless" Abstract Syntax Tree (AST) that retains token and whitespace information (e.g., using libraries like `rowan` or equivalent techniques) is the required approach for robust formatting, especially concerning comment placement.

### 2.4 Pretty-Printing Algorithm

* The formatter implements a pretty-printing algorithm that traverses its internal lossless syntax representation.
* It applies the deterministic rules defined in the Style Guide (Section 3) to emit the formatted text, managing indentation, line breaks (respecting the maximum line length), wrapping, and spacing consistently.

### 2.5 Idempotency Requirement

The formatting process must be idempotent. Re-formatting already formatted code must yield the identical text output.

### 2.6 Error Handling (Syntax Errors)

If the input source code contains syntax errors preventing successful parsing, `twilightfmt` must report the parsing error (ideally using the compiler's standard diagnostic format) and exit with a non-zero status code without attempting to output formatted code.

## 3. Twilight Native Style Guide v1.0

These rules define the canonical style enforced by `twilightfmt` for `twilight-native` code.

### 3.1 General

* **Encoding:** Source files must use UTF-8 encoding.
* **Indentation:** Use 4 spaces for each indentation level. Tabs are not permitted.
* **Maximum Line Length:** Lines should not exceed 100 characters. Lines exceeding this limit will be wrapped according to the rules defined below.
* **Trailing Whitespace:** All trailing whitespace characters at the end of lines must be removed.
* **File Termination:** Files must end with exactly one newline character. No blank lines are permitted at the end of the file.

### 3.2 Brace Style

* Use a variant of K&R style:
    * Opening braces (`{`) appear on the same line as the preceding declaration or control flow keyword, separated by one space.
    * Closing braces (`}`) appear on their own line, aligned with the indentation level of the opening statement/declaration.
    * Exception: Empty blocks may be written as `{}` on a single line.
    ```twilight
    // Correct Brace Style
    fn my_function(arg: u64) -> bool {
        if arg > 10 {
            print("Large");
            return true;
        } else {
            print("Small");
            return false;
        }
    } // Aligned closing brace

    struct MyStruct { field: u64 } // OK for single line

    impl MyTrait for MyStruct {
        fn method() {
            // ...
        } // Aligned closing brace
    }
    ```

### 3.3 Spacing

* **Commas (`,`):** Follow with one space. No space before the comma. (Example: `fn foo(a: u64, b: u64)`)
* **Colons (`:`):**
    * Type annotations: No space before, one space after (`x: u64`).
    * Struct initializers: No space before, one space after (`field: value`).
    * Match arms (`=>`): See below.
    * Other uses (e.g., labels - not typically formatted): Follow standard conventions.
* **Semicolons (`;`):** No space before. Followed by a newline or the end of the file. Multiple statements on one line separated by semicolons are discouraged and may be reformatted onto separate lines.
* **Binary Operators (`+`, `-`, `*`, `/`, `%`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `&&`, `||`, `&`, `|`, `^`, `<<`, `>>`, `=`):** Surround with one space on each side (`a + b`, `x == 10`).
* **Unary Operators (`!`, `-` negation):** No space between the operator and its operand (`!is_valid`, `-5`).
* **Parentheses (`()`):** No space immediately following `(` or immediately preceding `)`. (Example: `foo(arg)`, `if (x > 0)` becomes `if x > 0`).
* **Square Brackets (`[]`):** No space immediately following `[` or immediately preceding `]`. (Example: `my_array[index]`).
* **Curly Braces (`{}`):**
    * One space after `{` and one space before `}` if the content is on the same line (e.g., `MyStruct { field: 0 }`).
    * If content spans multiple lines, place a newline after `{` and a newline before `}`.
    * No space before the opening brace if it follows a keyword or identifier (e.g., `if condition {`, `fn foo() {`).
* **Angle Brackets (`<>` for Generics/Types):** No space immediately following `<` or immediately preceding `>`. No space before `<` when used after a type or function name. (Example: `vector<u64>`, `MyStruct<T, U>`, `foo::<u32>()`).
* **Dot Operator (`.`):** No space on either side (`object.field`, `object.method()`).
* **Path Separator (`::`):** No space on either side (`crate::module::Item`).
* **Match Arm Arrow (`=>`):** Surround with one space on each side (`pattern => expression`).

### 3.4 Blank Lines

* **Between Top-Level Items:** Use **two** blank lines between item definitions (functions, structs, enums, contracts, impls, traits, consts, type aliases) at the module level.
* **Within Items:** Use **one** blank line to separate logical groups of code where appropriate (e.g., between methods in an `impl`, between struct fields for clarity). Avoid excessive blank lines.
* **File Boundaries:** No blank lines at the very beginning of the file. Exactly one newline character at the end.

### 3.5 Imports (`use` statements)

* One `use` statement per line.
* **Grouping & Ordering:** Group imports in the following order, with one blank line separating groups:
    1.  `std::` / `core::` imports (Standard library).
    2.  `nova::` imports (NOVA-specific stdlib).
    3.  External crate imports (Dependencies).
    4.  `crate::` imports (Items from the current crate).
    5.  `super::` imports.
    6.  `self::` imports.
* **Sorting:** Within each group, sort paths alphabetically. Identifiers within list imports (`{...}`) are also sorted alphabetically.
* **List Imports (`use path::{a, b, c};`):**
    * If the items fit on one line within the maximum line length, format as: `use path::{a, b, c};` (items sorted).
    * If the items exceed the line length, wrap vertically with one item per line, indented, and allow a trailing comma:
        ```twilight
        use some::very::long::module::path::{
            alpha_item,
            beta_item,
            gamma_item, // Trailing comma
        };
        ```
    * Nested trees (`use a::{b::{c, d}};`) follow similar wrapping and sorting rules.

### 3.6 Line Wrapping & Breaking

* Lines exceeding the maximum line length (100 characters) must be wrapped.
* **Priorities:** Prefer breaking after commas, opening parentheses/brackets/braces, and before or after binary operators (`&&`, `||`, arithmetic, etc.) where logical.
* **Indentation:** Indent wrapped lines consistently, typically by +4 spaces relative to the starting line.
* **Function Signatures:** Wrap long parameter lists or return types. Parameters are typically aligned vertically.
    ```twilight
    fn process_complex_data(
        config: ConfigurationObject,
        user_data: vector<UserDataEntry>,
        callback: fn(Result<Output, Error>) -> (),
    ) -> Result<FinalStatus, ProcessingError> {
        // ...
    }
    ```
* **Function Calls / Struct Initialization:** Wrap long argument/field lists, typically with one element per line, indented.
    ```twilight
    let result = some_very_long_function_name(
        first_argument,
        second_argument_which_is_also_long,
        calculate_third_argument(),
    );

    let my_struct = MyStruct {
        field_one: value_one,
        field_two_long_name: another_value,
        field_three: 42,
    };
    ```
* **Chained Method Calls:** Place each subsequent call starting with `.` on a new line, indented.
    ```twilight
    let processed_value = initial_data
        .step_one()
        .filter_items(|item| item.is_valid)
        .map_results(process_single_item)
        .collect_into_vector();
    ```
* **Binary Expressions:** Wrap long expressions, usually breaking *before* an operator (`+`, `&&`, etc.) and aligning subsequent operands under the first operand if logical.

### 3.7 Attributes (`#[...]`)

* Place attributes on the line immediately preceding the item they modify.
* Place each attribute on its own line.
    ```twilight
    #[payable]
    #[reads("balance:*")]
    fn deposit() {
        // ...
    }
    ```

### 3.8 Comments

* **Single-line (`//`):** Preserve both standalone line comments and inline comments following code. Align standalone comments with the surrounding code's indentation level. Attempt to keep inline comments aligned if feasible.
* **Doc Comments (`///`):** Preserve immediately before the item they document. Reformat content within doc comments minimally (e.g., fix indentation), but do not reflow markdown heavily.
* **Block Comments (`/* ... */`):** Preserve. Re-indent content relative to the opening `/*` line, but avoid significant reflowing or reformatting of the internal comment structure.

## 4. Twilight Syntax Mode (TSM) Handling

* **Target:** `twilightfmt` is designed exclusively for formatting `twilight-native` syntax.
* **Detection:** If a file contains `#![syntax = "solidity"]` or potentially another non-native directive, `twilightfmt` MUST detect this.
* **Action:** Upon detecting a non-native syntax mode, `twilightfmt` MUST NOT attempt to format the file. It should exit silently or print an informational message indicating the file was skipped because it is not `twilight-native` syntax. Developers should use formatters specific to the declared syntax (e.g., a Solidity formatter for `solidity` mode).

## 5. Configuration Policy

* **Policy:** `twilightfmt` is intentionally opinionated to enforce a single standard style.
* **Configuration:** No formatting style configuration options are provided in v1.0. The rules defined in Section 3 constitute the official Twilight style.

## 6. Integration

* **Command Line Interface (CLI):**
    * `twilightfmt <file/dir>`: Formats specified files/directories and prints the result to standard output.
    * `twilightfmt --write <file/dir>`: Formats files/directories, modifying them in-place.
    * `twilightfmt --check <file/dir>`: Checks if files/directories are correctly formatted. Exits with a non-zero status code if any file would be changed by formatting, printing diffs or filenames.
* **Language Server Protocol (LSP):**
    * The `twilight-lsp` server (`specs/tooling/12-lsp.md`) integrates with `twilightfmt` (likely by invoking it as a library or external process) to provide `textDocument/formatting` capabilities to compatible editors.