# Twilight Formal Semantics Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document outlines the vision, importance, scope, and intended approach for developing formal semantics for the Twilight programming language. Formal semantics provide a mathematically precise and unambiguous definition of the language's behavior. They serve as the essential foundation for ensuring the soundness of the Twilight compiler (`twilightc`), the NOVA runtime/VM execution, and any formal verification tools used within the ecosystem (such as the Verification Condition Generator).

### 1.2 Target Audience

* Language Designers
* Compiler and Runtime Implementers (`twilightc`, NOVA node)
* Verification Tool Developers
* Formal Methods Researchers
* Auditors requiring precise language understanding
* Educators teaching Twilight

## 2. Rationale and Importance

Formal semantics are critical for a high-assurance language and platform like Twilight/NOVA for several reasons:

* **Unambiguity:** Eliminates ambiguities inherent in natural language specifications, providing a single, definitive source of truth for language behavior.
* **Soundness of Verification:** Formal verification tools (like VCGen targeting SMT solvers) can only be proven sound (i.e., if they claim a property holds, it actually does hold) relative to a formal definition of the language's semantics. Formal semantics provide this necessary foundation.
* **Language Design Clarity:** The process of formalization rigorously examines language features, often uncovering subtle edge cases, inconsistencies, or undefined behaviors, leading to a more robust and well-designed language.
* **Implementation Guidance:** Provides a precise specification for compiler and runtime implementers, helping ensure that different components and future implementations behave identically according to the language rules.
* **Standardization & Interoperability:** Forms the basis for potential future standardization efforts and ensures ecosystem tools operate on a shared, precise understanding of Twilight.

## 3. Scope and Phased Approach

Formalizing an entire language is a significant undertaking. The effort will be approached iteratively:

1.  **Core Language Semantics:** Initial focus on formalizing the execution rules for core expressions, statements, standard control flow (`if`, `loop`, `while`, `match`), basic function calls, and the static type system rules.
2.  **Memory & State Model:** Crucially, formalize the implicit memory and ownership model (`specs/language/01-core-language.md`, Section 5), including the abstract machine state, ownership transfer (move), copy semantics, and the rules governing implicit borrow validity and lifetimes. Formalize the semantics of persistent `storage` access (`state_read`/`state_write`) and the state delta model.
3.  **Blockchain Specifics:** Formalize the behavior of integrated blockchain features like `require`, `panic`, event emission (`emit`), context access (`ctx.*`, `env.*`), and the semantics of interactions with core Host Function Interface (HFI) calls.
4.  **Advanced Features:** Extend the formalization incrementally to cover generics, traits, advanced error handling patterns, and other features as they are fully specified and implemented.

The documentation MUST clearly indicate which parts of the language have their semantics formally specified in any given version.

## 4. Chosen Formalisms

A combination of formal methods is recommended:

### 4.1 Operational Semantics (Primary)

* **Method:** Define the meaning of programs by specifying their execution step-by-step, typically as transitions on an abstract state machine (Small-Step Semantics preferred for detailed modeling).
* **Rationale:** Well-suited for specifying execution flow, state changes, determinism, and potentially gas consumption per step. Often more intuitive for implementers. Can be used to prove safety properties (e.g., type preservation).
* **Abstract Machine:** Defines the components representing the machine state (Instruction Pointer/Term, Environment/Scope, Store/Memory, Persistent Storage Map, Context, Gas Counter, Logs, Status, etc.).

### 4.2 Axiomatic Semantics (Complementary)

* **Method:** Define the meaning of program constructs via logical assertions (preconditions, postconditions, invariants), typically using Hoare Logic or Weakest Precondition (WP) calculus.
* **Rationale:** Directly supports the Verification Condition Generation process (`specs/verification/19-vcgen.md`). Provides a specification-oriented view useful for reasoning about program correctness relative to assertions.
* **Relationship:** Axiomatic rules must be proven sound with respect to the operational semantics. WP definitions derive directly from the axiomatic rules.

## 5. Process, Evolution, and Documentation

### 5.1 Iterative Development

Formal semantics will be developed iteratively, ideally in parallel with or slightly lagging the design and implementation of the corresponding language features. The process starts with a core subset and expands coverage over time.

### 5.2 Mechanization (Optional Recommendation)

* **Approach:** The use of a proof assistant (e.g., Coq, Isabelle/HOL, Lean) to mechanize the formal semantics definitions and potentially prove key properties (like type safety) about the language specification itself is recommended for achieving the highest level of assurance.
* **Benefit:** Provides machine-checked proofs of correctness and consistency for the formal semantics, aiding in uncovering subtle issues.
* **Effort:** This represents a significant research and engineering commitment requiring specialized expertise.

### 5.3 Documentation Integration

* **Location:** The formal semantics specification will reside in a dedicated section of the official Twilight Language Specification document.
* **Content:** Must include definitions of the chosen formalism(s), notation, the abstract machine model, state representations, and the detailed transition/axiomatic rules. Each formal rule must be accompanied by a clear natural language explanation and justification.
* **Versioning:** The formal semantics specification MUST be versioned alongside the language specification. Any change to language semantics requires a corresponding update here.
* **Cross-Referencing:** Link formal definitions to relevant sections in the language reference manual and tutorials where appropriate.

## 6. Relationship to Verification Tools

The formal semantics serve as the ground truth upon which the soundness of verification tools, particularly the VCGen pass within `twilightc`, depends. The VCGen pass implements the axiomatic semantics (via WP) to translate program assertions into logical VCs; these VCs are only meaningful if the axiomatic semantics accurately reflect the true operational behavior of the language as defined by the operational semantics.