# uni_parser
Use EBNF to support any file format including re-output or JSON output.

File format may be:

* Block DSL or
* Non-block DSL

Block assumes the following DSL:

    block → statements → (assignment | block | list)

Non-block DSL may have:

* INI: infers DSL thru flat sections, key=value pair
* OpenSSL: infers DSL thru section, key=value pair
* YAML: infers DSL thru indentation-driven tree

This design will not support Python config due to arbitrary expressions such as `a = function(b)`.


Working example:

    ISC Bind9 (`/etc/bind/named.conf`)
    nftables (`/etc/nft/nftables.nft`)
    OpenSSL (`/etc/ssl/openssl.cnf`)
    YAML
    INI

Design Approach
====

EBNF (meta grammar, `ebnf-flexible.dhparser`)
        │
        ▼
[Stage 1] Parse EBNF
        │
        ▼
AST-EBNF (syntax-only IR)
        │
        ▼
S-Expression (normalized grammar IR)
        │
        ▼
[Stage 2] DSL Parse Engine (generated/interpreted)
        │
        ▼
AST-DSL (domain-neutral structure + tokens + comments)
        │
        ▼
DSL-SPEC (constraint-registry, `nftablesglobal.py`/DSL)
        │
        ▼
Shared Structural Validation Layer
        │
        ▼
Grammar Constraints Layer (context, multiplicity)
        │
        ▼
Semantic Mapping Layer (typed model)
        │
        ▼
Validation / Lint Layer
        │
        ▼
Formatter / Emitter
        │
        ▼
Output / Migration Engine



Stage One
====
Focus is on ingesting the syntax of the target using EBNF notations detailing its target (ie., nftables).

EBNF of EBNF
----
EBNF of all forms of known EBNF variants, in dhparse format.

Parse EBNF
----
Parse the EBNF-flexible.dhparse (we could hardcode those but this is a demonstrator of EBNF flexibility).

AST-EBNF
----
AST describing grammar only (no semantics).

S-Expression
----
Purpose

* Normalize grammar into execution-ready structure
* Remove DHParser-specific structure

```s-expression
(def <rule-name>
    (seq
        (lit "zone")
        (sym string)
        (sym block)
    )
)
```
S-Expression Schema
```python
class RuleIR:
    name: str
    expr: tuple  # ('seq', [...]), ('alt', [...]), ('lit', str), ('sym', name)
```


AST-EBNF Schema
```python
class EBNFNode:
    type: str  # definition | sequence | choice | literal | regex | symbol
    value: str | None
    children: list
```


Stage Two
====
Stage-2 AST must be: structure-first, syntax-agnostic, not block-centric.

Stage-2 AST cannot do uniform-model.

DSL-Parsing (Generic Engine)
---
Contract

* Input
 * Grammar IR (S-expr)
 * DSL text (named.conf / nftables / openssl.cnf)

* Output
 * AST-DSL (syntax tree + tokens + comments)

AST-DSL
----
Core Requirement

Must support:

* Bind9 (named.conf)
* nftables
* OpenSSL config
* YAML
* INI

Base Node would look like:
```python
class ASTNode:
    type: str
    value: str | None
    children: list["ASTNode"]

    # cross-cutting concerns
    comments: list["Comment"]
    meta: dict  # line, col, file

    # grammar-level annotations
    rule: str | None
    multiplicity: str | None  # "1", "?", "*", "+"
```

Specialized node Types would be:

* Block
```python
type = "block"

value = {
    "name": "options" | "zone" | "server" | ...
}
```
* Assignment
```python
type = "assignment"

value = {
    "key": str,
    "operator": "=" | None
}
children = [value_node]
```
* List
```python
type = "list"
children = [ASTNode...]
```
* Literal / Token
```python
type = "literal" | "identifier" | "number" | "string"
value = str
```

* Comment
```python
class Comment:
    text: str
    position: int
    inline: bool
```

Shared Structural Validation Layer
---
Purpose

* Grammar-independent checks

Examples

 * duplicate keys in same scope
 * empty blocks
 * invalid nesting depth

* Interface
```python
def validate_structure(ast: ASTNode) -> list[Issue]
```

Grammar Constraint Layer
---
Derived from Grammar IR

Adds:

 * allowed children per node
 * multiplicity constraints
 * context restrictions

Example Constraint Model
```python
class Constraint:
    parent: str
    child: str
    multiplicity: str  # "1", "?", "*", "+"
```

Example
```
zone:
    file → 1
    type → 1
    allow-transfer → *
```
