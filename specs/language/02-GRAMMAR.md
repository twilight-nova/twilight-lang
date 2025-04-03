# Twilight Grammar Specification v1.0

## 1. Introduction

### 1.1 Purpose

This document provides the formal grammar specification for the `twilight-native` syntax mode and the mapping specification for the `solidity` syntax mode. It defines the lexical structure (tokens) and the syntactic structure (grammar rules) that compliant Twilight compilers must adhere to when parsing source code.

### 1.2 Notation

* **EBNF:** The syntax for `twilight-native` is specified using Extended Backus-Naur Form (EBNF).
    * `::=` means "is defined as".
    * `|` separates alternatives.
    * `?` indicates optional occurrence (0 or 1).
    * `*` indicates zero or more occurrences.
    * `+` indicates one or more occurrences.
    * `( ... )` groups elements.
    * `'...'` denotes literal terminals.
    * `[abc]` denotes character classes.
    * `[^abc]` denotes negated character classes.
    * Non-terminals are typically `UpperCamelCase` or `snake_case`.
    * Terminals (tokens) are typically `UPPER_SNAKE_CASE`.
* **Mapping:** The `solidity` mode is specified via descriptive mapping rules from Solidity constructs to their intended Twilight equivalent semantics, assuming parsing is handled by a standard Solidity parser (e.g., Solang).

## 2. Lexical Structure (Tokens - twilight-native)

This section defines the valid tokens recognized by the `twilight-native` lexer. Source location (span) information must be associated with each token.

### 2.1 Whitespace and Comments

* **`WHITESPACE`**: Defined as one or more occurrences of space, tab, newline, or carriage return characters (`(' ' | '\t' | '\n' | '\r')+`).
    * *Action:* Ignored by the parser. Newlines may be tracked internally for potential future diagnostic or formatting use, but explicit semicolons are currently required to terminate most statements.
* **`COMMENT_SINGLE`**: Defined as `//` followed by any sequence of characters except newline (`'//' [^\n]*`).
    * *Action:* Ignored by the parser.
* **`COMMENT_MULTI`**: Defined as characters enclosed within `/*` and `*/` (`'/*' ( [^*] | '*'+ [^/] )* '*'+ '/'`). Nesting is not supported.
    * *Action:* Ignored by the parser.
* **`DOC_COMMENT`**: Defined as `///` followed by any sequence of characters except newline (`'///' [^\n]*`).
    * *Action:* Tokenized distinctly (e.g., `DOC_COMMENT(content)`). The parser associates this token with the subsequent item definition for documentation generation.

### 2.2 Identifiers and Keywords

* **`IDENTIFIER`**: Defined as starting with an uppercase letter, lowercase letter, or underscore (`[a-zA-Z_]`), followed by zero or more alphanumeric characters or underscores (`[a-zA-Z0-9_]*`).
    * *Action:* Tokenized as `IDENT(name)`. An identifier cannot be identical to a reserved keyword.
* **`KEYWORD`**: Reserved words that have special meaning in the language. They are tokenized as distinct types (e.g., `KW_CONTRACT`, `KW_FN`). The keywords are:
    * `contract`, `storage`, `fn`, `struct`, `enum`, `val`, `var`, `const`, `type`, `if`, `else`, `loop`, `while`, `for`, `in`, `match`, `return`, `break`, `continue`, `pub`, `crate`, `use`, `super`, `self`, `impl`, `trait`, `true`, `false`, `emit`, `event`, `require`, `old`, `mut`.

### 2.3 Literals

* **`INT_LITERAL`**: Represents integer values. Supports decimal (e.g., `123`), hexadecimal (prefixed with `0x`, e.g., `0xFF`), binary (prefixed with `0b`, e.g., `0b101`), and octal (prefixed with `0o`, e.g., `0o77`) notations. Underscores (`_`) are permitted as separators for readability (e.g., `1_000_000`). An optional type suffix (e.g., `u8`, `u64`, `i256`) can specify the exact integer type.
    * *Action:* Tokenized preserving the representation and suffix (e.g., `INT_LIT(value_string, suffix_string)`). Validation and conversion occur later.
* **`STRING_LITERAL`**: Defined as a sequence of characters enclosed in double quotes (`"`). Supports standard escape sequences (`\n`, `\t`, `\\`, `\"`) and Unicode escapes (`\u{...}`). (`'"' ( EscapeSequence | [^"\\] )* '"'`)
    * *Action:* Tokenized as `STR_LIT(value)`. The lexer processes escape sequences and stores the resulting UTF-8 string value.
* **`BOOL_LITERAL`**: The keywords `true` and `false`.
    * *Action:* Tokenized distinctly (e.g., `BOOL_LIT(true)`).

### 2.4 Operators

Tokens representing operations. Each operator typically corresponds to a distinct token type.

* **Arithmetic:** `+`, `-`, `*`, `/`, `%`
* **Comparison:** `==`, `!=`, `<`, `>`, `<=`, `>=`
* **Logical:** `&&`, `||`, `!`
* **Bitwise:** `&`, `|`, `^`, `<<`, `>>`
* **Assignment:** `=`
* **Member Access / Path:** `.`, `::`
* **Misc:** `=>` (used in match arms), `?` (the try/propagation operator)

### 2.5 Punctuation

Tokens representing syntactic structure or separators. Each typically corresponds to a distinct token type.

* **Braces/Parens/Brackets:** `{`, `}`, `(`, `)`, `[`, `]`
* **Separators/Terminators:** `:`, `,`, `;`
* **Annotation/Attribute Marker:** `#`
* **Pattern Wildcard:** `_`

## 3. Syntactic Grammar (EBNF - twilight-native)

This EBNF defines the valid structure of `twilight-native` source code. Parser implementations must handle operator precedence and associativity rules, which are not explicitly detailed in this grammar format but follow standard conventions (e.g., multiplication before addition, left-associativity for arithmetic).

```ebnf
// --- Top Level ---
CompilationUnit ::= SyntaxDirective? Imports Items EOF;
SyntaxDirective ::= '#' '!' '[' 'syntax' '=' STRING_LITERAL ']' ';';
Imports ::= UseDeclaration*;
Items ::= ItemDefinition*;

// --- Imports and Paths ---
UseDeclaration ::= 'use' Path UseSpecifier? ';';
Path ::= PathSegment ( '::' PathSegment )*;
PathSegment ::= IDENTIFIER | 'super' | 'crate' | 'self';
UseSpecifier ::= '::' '*'                                 // Glob import
               | '::' '{' UseTree ( ',' UseTree )* ','? '}' // Tree import
               | 'as' IDENTIFIER;                          // Alias import
UseTree ::= Path UseSpecifier?;

// --- Item Definitions ---
ItemDefinition ::= Visibility? ItemKind;
Visibility ::= 'pub' ('(' 'crate' ')')?; // pub or pub(crate)

ItemKind ::= ContractDefinition
           | StructDefinition
           | EnumDefinition
           | FunctionDefinition
           | TraitDefinition
           | ImplDefinition
           | ConstDefinition
           | TypeAliasDefinition;

// --- Contract Related ---
ContractDefinition ::= Attribute* 'contract' IDENTIFIER '{' ContractItem* '}';
ContractItem ::= FunctionDefinition
               | EventDefinition
               | StructDefinition  // Nested types
               | EnumDefinition    // Nested types
               | TypeAliasDefinition // Nested types
               | ConstDefinition
               | StorageBlock
               | InvariantAssertion;

StorageBlock ::= 'storage' '{' StorageItem* '}';
StorageItem ::= IDENTIFIER ':' Type ( '=' Expression )? ';'; // Optional initializer

EventDefinition ::= Attribute* 'event' IDENTIFIER '(' EventParameterList? ')' ';';
EventParameterList ::= EventParameter ( ',' EventParameter )*;
EventParameter ::= Attribute* '#'? '[' 'topic' ']' IDENTIFIER ':' Type // #[topic] marker
                 | Attribute* IDENTIFIER ':' Type;

InvariantAssertion ::= Attribute* 'assert_invariant' '(' Expression (',' STRING_LITERAL)? ')' ';';

// --- Function ---
FunctionDefinition ::= Attribute* 'fn' IDENTIFIER GenericParams? '(' ParameterList? ')' ('->' Type)? WhereClause? BlockExpression;
ParameterList ::= Parameter ( ',' Parameter )*;
Parameter ::= Attribute* 'mut'? IDENTIFIER ':' Type;

// --- Struct ---
StructDefinition ::= Attribute* 'struct' IDENTIFIER GenericParams? StructBody WhereClause?;
StructBody ::= '{' FieldList? '}'          // Named fields
             | '(' TypeList? ')' ';'       // Tuple struct
             | ';';                        // Unit struct
FieldList ::= StructField ( ',' StructField )* ','?;
StructField ::= Attribute* Visibility? IDENTIFIER ':' Type;
TypeList ::= Type ( ',' Type )*;

// --- Enum ---
EnumDefinition ::= Attribute* 'enum' IDENTIFIER GenericParams? '{' EnumVariantList? '}' WhereClause?;
EnumVariantList ::= EnumVariant ( ',' EnumVariant )* ','?;
EnumVariant ::= Attribute* IDENTIFIER StructBody?; // Allows variants with/without fields

// --- Trait ---
TraitDefinition ::= Attribute* 'trait' IDENTIFIER GenericParams? WhereClause? '{' TraitItem* '}';
TraitItem ::= FunctionSignature ';'
            | TypeAliasDefinition ';'; // Associated types eventually

FunctionSignature ::= 'fn' IDENTIFIER GenericParams? '(' ParameterList? ')' ('->' Type)? WhereClause?;

// --- Implementation ---
ImplDefinition ::= Attribute* 'impl' GenericParams? TraitName? 'for' Type WhereClause? '{' ImplItem* '}'; // TraitName optional for inherent impls
TraitName ::= Path; // Path to the trait
ImplItem ::= FunctionDefinition
           | ConstDefinition
           | TypeAliasDefinition; // Associated items

// --- Other Items ---
ConstDefinition ::= 'const' IDENTIFIER ':' Type '=' Expression ';';
TypeAliasDefinition ::= 'type' IDENTIFIER GenericParams? '=' Type ';';

// --- Generics & Attributes ---
GenericParams ::= '<' GenericParam ( ',' GenericParam )* '>'?;
GenericParam ::= IDENTIFIER (':' TraitBoundList)?; // Optional trait bounds
TraitBoundList ::= TraitBound ('+' TraitBound)*;
TraitBound ::= Path; // Path to a trait

WhereClause ::= 'where' WherePredicate ( ',' WherePredicate )*;
WherePredicate ::= IDENTIFIER ':' TraitBoundList; // e.g., where T: Debug + Clone

Attribute ::= '#' '[' Path ('(' AttributeArgs? ')')? ']';
AttributeArgs ::= Expression (',' Expression)*; // Allow expressions, not just literals

// --- Statements ---
Statement ::= LetStatement
            | ExpressionStatement
            | ItemDefinition // Allow nested item definitions
            // | MacroInvocationStmt (If macros added)
            | ';'; // Empty statement

LetStatement ::= ('val' | 'var') Pattern (':' Type)? '=' Expression ';'; // Type annotation optional if initializer present
ExpressionStatement ::= Expression ';'; // Restrictions apply, e.g., expression must have side effects or not end in block/semicolon

// --- Expressions (High-Level, Precedence Defined Separately) ---
Expression ::= BinaryExpression | UnaryExpression | PrimaryExpression;

BinaryExpression ::= Expression BinOp Expression;
UnaryExpression ::= UnOp Expression;

PrimaryExpression ::= Literal
                    | PathExpression // Variables, constants, enum variants, type names
                    | GroupedExpression // ( Expression )
                    | ArrayExpression // [ elem1, elem2 ] or [ init; N ]
                    | VectorExpression // vector![...] syntax sugar? Or use stdlib constructor
                    | TupleExpression // ( elem1, elem2, )
                    | StructInitialization // MyStruct { field: value }
                    | FunctionCall // func(arg1)
                    | MethodCall // value.method(arg1)
                    | IndexExpression // array[index]
                    | FieldAccess // struct.field or tuple.0
                    | BlockExpression // { statements... expr? }
                    | IfExpression // if cond { ... } else { ... }
                    | LoopExpression // loop { ... }
                    | WhileExpression // while cond { ... }
                    | ForExpression // for pattern in iterable { ... }
                    | MatchExpression // match value { pattern => expr, ... }
                    | ReturnExpression // return expr?
                    | BreakExpression // break expr?
                    | ContinueExpression // continue
                    | EmitStatement // emit Event(...) - treated as expression?
                    | RequireCall // require(cond, msg) - treated as expression?
                    | OldCall; // old(expr) - only in specific verification contexts

// Selected Specific Forms:
PathExpression ::= Path ( '::' GenericArgs )?; // Optional generic args for types/functions
ArrayExpression ::= '[' Expression (',' Expression)* ','? ']' // List form
                  | '[' Expression ';' Expression ']'; // Repeat form [value; count]
TupleExpression ::= '(' ( Expression ',' )+ Expression? ')'; // Need refinement for single element (x,)
StructInitialization ::= Path '{' FieldInitList? '}';
FieldInitList ::= FieldInit ( ',' FieldInit )* ','?;
FieldInit ::= IDENTIFIER ':' Expression | IDENTIFIER; // Shorthand if variable name matches field
FunctionCall ::= PrimaryExpression '(' ArgumentList? ')';
MethodCall ::= PrimaryExpression '.' IDENTIFIER GenericArgs? '(' ArgumentList? ')';
IndexExpression ::= PrimaryExpression '[' Expression ']';
FieldAccess ::= PrimaryExpression '.' ( IDENTIFIER | INT_LITERAL /*Tuple index*/ );
BlockExpression ::= '{' Statement* Expression? '}'; // Last expression is block's value
IfExpression ::= 'if' Expression BlockExpression ('else' (IfExpression | BlockExpression))?;
ForExpression ::= 'for' Pattern 'in' Expression BlockExpression;
MatchExpression ::= 'match' Expression '{' MatchArm* '}';
MatchArm ::= Attribute* Pattern ('if' Expression /*Guard*/)? '=>' Expression ','?;
ReturnExpression ::= 'return' Expression?;
BreakExpression ::= 'break' Expression?;
EmitStatement ::= 'emit' Path '(' ArgumentList? ')';
RequireCall ::= 'require' '(' Expression ',' STRING_LITERAL ')';
OldCall ::= 'old' '(' Expression ')'; // Validate context
ArgumentList ::= Expression (',' Expression)*;
GenericArgs ::= '<' Type (',' Type)* '>'?; // Type arguments for function/method calls or type paths

// --- Types ---
Type ::= PathType // e.g., u64, string, MyStruct<T>
       | TupleType // (T1, T2)
       | ArrayType // array<T, N>
       | VectorType // vector<T>
       | MapType // map<K, V>
       | ImplTraitType; // impl TraitName
       // Deferred: dyn Trait, references &, pointers *

PathType ::= Path GenericArgs?;
TupleType ::= '(' Type (',' Type)* ','? ')';
ArrayType ::= 'array' '<' Type ',' Expression '>'; // Expression must be constant
VectorType ::= 'vector' '<' Type '>';
MapType ::= 'map' '<' Type ',' Type '>';
ImplTraitType ::= 'impl' TraitBoundList;

// --- Patterns (for `let`, `match`, `for`) ---
Pattern ::= Literal
          | IDENTIFIER ('@' Pattern)? // Bind variable `x @ SubPattern`
          | '_' // Wildcard
          | 'mut'? IDENTIFIER // Binding
          | Path '{' FieldPatternList? '}' // Struct pattern
          | Path '(' PatternList? ')' // Tuple struct / Enum variant pattern
          | '(' Pattern (',' Pattern)* ','? ')' // Tuple pattern
          | '[' Pattern (',' Pattern)* ','? ']' // Array/Slice pattern
          // Deferred: Range patterns ( Start '..' End )

FieldPatternList ::= FieldPattern (',' FieldPattern)* ','?;
FieldPattern ::= IDENTIFIER ':' Pattern | IDENTIFIER; // Shorthand binding `field` same as `field: field`
PatternList ::= Pattern (',' Pattern)*;

4. Solidity-Mode (solidity) Mapping

This section defines the mapping from supported Solidity syntax constructs (when #![syntax = "solidity"] is used) to their equivalent Twilight semantics and internal representation (Twilight AST / TIR). Parsing is assumed to be handled by a mature Solidity parser (e.g., Solang). The semantics executed are always Twilight's semantics.
4.1 Goal and Implementation Strategy

    Goal: Provide a familiar syntax entrypoint for Solidity developers while leveraging Twilight's safety features, compiler backend, and NOVA integration. The mapping aims for semantic equivalence where features overlap, clearly erroring on unsupported constructs.
    Implementation Approach: Utilize an existing, robust Solidity parser library (e.g., Solang is recommended). Implement a dedicated SolidityAST -> TwilightAST conversion pass within the twilightc compiler. This pass translates supported Solidity AST nodes into the equivalent Twilight AST nodes (defined in specs/compiler/06-frontend.md), applying the mapping rules below.

4.2 Mapping Rules

The following outlines how key Solidity constructs are interpreted:

    Contracts (contract C is A, B { ... }):
        Maps to a Twilight contract C { ... }.
        Inheritance (is A, B) presents significant challenges due to differences in state layout and function overriding semantics. The mapping strategy is TBD and may involve:
            Restricting complex inheritance patterns (e.g., disallowing multiple inheritance with overlapping state/functions).
            Code generation/inlining logic from base contracts.
            Mapping Solidity interfaces/abstract contracts to Twilight trait requirements (impl Trait for C).
    State Variables:
        uint256 public constant X = 10; -> Maps to Twilight pub const X: u256 = 10;.
        uint256 public balance; -> Maps to Twilight storage { pub balance: u256 = 0; } (Default zero-initialization).
        mapping(address => uint) public balances; -> Maps to Twilight storage { pub balances: map<address, u256>; }.
        Solidity visibility (public, internal, private) maps to Twilight visibility (pub, pub(crate), private default). Note that Solidity public state variables generate implicit getter functions, which the converter must also generate if required.
    Functions:
        function f(uint x) external pure returns (uint) -> Maps to Twilight pub fn f(x: u64) -> u64 { ... } (assuming uint maps to u64).
        Solidity visibility (external, public, internal, private) maps to Twilight (pub, pub, pub(crate), private).
        Solidity state mutability (payable, pure, view) maps to Twilight annotations (#[payable], inferred/annotated #[reads] or #[conflict_free]).
        Parameter and return types are mapped according to the Type Mapping rules.
    Modifiers:
        A modifier like modifier onlyOwner { require(msg.sender == owner); _; } applied to function doThing() is handled by inlining the modifier's code (excluding the _) at the beginning of the corresponding Twilight function fn doThing(). The original function body follows the inlined modifier code.
        Example: fn doThing() { require(ctx.sender() == storage.owner, "Caller is not the owner"); /* original body */ }.
    Events:
        event Transfer(address indexed from, address indexed to, uint256 value); -> Maps to Twilight event Transfer(#[topic] from: address, #[topic] to: address, value: u256);. The indexed keyword maps to the #[topic] annotation.
        emit Transfer(a, b, c); -> Maps directly to emit Transfer(a, b, c);.
    Custom Errors:
        error InsufficientBalance(uint requested, uint available); ... revert InsufficientBalance(100, 50); -> Maps to a require(false, "EncodedErrorString") call in Twilight. The error name and arguments are ABI-encoded into the string message according to a defined scheme, allowing off-chain tools to decode the reason. Direct mapping to custom Twilight enum errors is deferred.
    Types:
        uintN/intN -> uN/iN (e.g., uint256 -> u256, int128 -> i128).
        address/address payable -> address (Payability is handled by #[payable] on functions).
        bool -> bool.
        string memory/string calldata -> string.
        bytes memory/bytes calldata -> vector<u8>.
        bytesN -> array<u8, N>.
        mapping(K => V) -> map<MappedK, MappedV>.
        T[N] (fixed array) -> array<MappedT, N>.
        T[] (dynamic array) -> vector<MappedT>.
        Structs map field-by-field to equivalent Twilight structs.
    Globals & Context:
        msg.sender -> ctx.sender()
        msg.value -> ctx.value()
        block.timestamp -> env.timestamp()
        block.number -> env.block_number()
        tx.origin -> tx.origin()
        Gas-related globals (block.coinbase, block.difficulty, block.gaslimit, gasleft(), tx.gasprice) are generally unsupported as gas mechanics differ.

4.3 Unsupported Solidity Features

The following Solidity features are explicitly not supported in solidity mode. Usage will result in a compile-time error during the AST conversion phase:

    Inline Assembly: assembly { ... } blocks.
    Contract Destruction: selfdestruct.
    Low-Level Calls (Unsafe Variants): delegatecall, .call{value: v, gas: g}(), .staticcall() used without robust, verifiable safety wrappers.
    Explicit Memory Management Keywords: memory, storage, calldata keywords on variables (memory areas are managed implicitly in Twilight).
    Function Overloading: Defining multiple functions with the same name but different parameter types within the same scope.
    Error Handling: try/catch blocks. (Use require or Result-based checks).
    Libraries: Solidity library contracts (linking and delegatecall usage are complex and often unsafe). Simple patterns might be mappable to standard contract calls in the future, but disallowed initially.
    Complex Inheritance: Patterns involving multiple inheritance (is A, B, ...) especially with conflicting state variables or function signatures. super keyword usage may be restricted.
    Special Functions (Default Mapping TBD): The default receive() external payable { ... } and fallback() external payable { ... } functions. A specific mapping (e.g., to designated function names like fn receive() #[payable] / fn fallback()) may be defined, or they may be disallowed initially.
    Gas-Related Globals: See Section 4.2.