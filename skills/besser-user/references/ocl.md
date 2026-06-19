# B-UML OCL Constraints Reference

How to write OCL (Object Constraint Language) constraints, attach them to a
`DomainModel`, parse them into an AST, validate them, and which generators
consume them. Read this when the user adds invariants/constraints to a model,
parses OCL text, or asks how constraints are checked or generated.

The simplest and most common path is a plain string-based `Constraint` added to
the model. A second, AST-based path (`parse_ocl` → `OCLConstraint`) exists for
tooling that needs to inspect or rewrite the parsed expression.

## Table of contents

- [Imports](#imports)
- [The Constraint class (string-based)](#the-constraint-class-string-based)
- [Attaching constraints to a DomainModel](#attaching-constraints-to-a-domainmodel)
- [Supported OCL syntax](#supported-ocl-syntax)
- [Minimal runnable example](#minimal-runnable-example)
- [Parsing into an AST (OCLConstraint)](#parsing-into-an-ast-oclconstraint)
- [Validation / evaluation](#validation--evaluation)
- [AST helpers](#ast-helpers)
- [Generators that consume constraints](#generators-that-consume-constraints)
- [Gotchas](#gotchas)

## Imports

```python
# The string-based path — this is all most users need.
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Constraint,
    IntegerType, StringType,
)

# The AST path (parsing, pretty-printing, AST inspection).
from besser.BUML.notations.ocl import (
    parse_ocl,        # parse OCL text -> OCLConstraint (AST)
    pretty_print,     # render an AST back to OCL source text
    WrappingVisitor,  # ANTLR visitor used internally by parse_ocl
    BOCLSyntaxError,  # raised on lex/parse errors
    normalize,        # rewrite an OCLConstraint to B-OCL normal form
)
from besser.BUML.metamodel.ocl.ocl import OCLConstraint, OCLExpression
```

## The Constraint class (string-based)

`Constraint` lives in `structural.py` and holds OCL as a **string**. It does not
parse anything by itself.

```python
Constraint(
    name: str,
    context: Class,          # the class the invariant is defined on
    expression: str,         # the OCL source text
    language: str,           # free-form tag; use "OCL"
    description: str = None,  # optional plain-language message on violation
    timestamp=None, metadata=None, is_derived=False, uncertainty=0.0,
)
```

- `language` is a **required positional** argument and is a free-form string;
  by convention it is `"OCL"`. Generators filter on `constraint.language == "OCL"`.
- `expression` must be a `str`. Assigning a non-string raises `TypeError`
  (to attach a parsed AST, use `OCLConstraint` — see below).
- `description` is optional natural-language text surfaced to end users when the
  constraint is violated (e.g. in the graphical editor).
- Attribute accessors: `constraint.context`, `constraint.expression`,
  `constraint.language`, `constraint.description`.

The expression text typically takes the form
`context <ClassName> inv[ <name>]: <boolean OCL expression>`.

## Attaching constraints to a DomainModel

`DomainModel` keeps constraints in a `set[Constraint]`:

```python
DomainModel(
    name, types=None, associations=None, generalizations=None,
    packages=None, constraints=None,  # set[Constraint]
    timestamp=None, metadata=None, is_derived=False, elements=None, uncertainty=0.0,
)
```

Three ways to attach:

```python
model = DomainModel(name="University", types={...}, constraints={c1, c2})
# or after construction:
model.add_constraint(c1)        # adds one constraint
model.constraints = {c1, c2}    # replace the whole set
model.remove_constraint(c1)     # raises ValueError if not present
```

- The `constraints` setter rejects duplicates: two constraints with the same
  `name` raise `ValueError`; a non-`Constraint` element raises `TypeError`.
- Constraints are folded into the model's derived `elements` set automatically.

## Supported OCL syntax

`expression` is parsed by the B-OCL (`BOCL`) ANTLR grammar. The top-level rule
accepts an invariant, an `init`, a `pre`, or a `post` declaration. Supported
constructs include:

- Headers: `context X inv[ name]: …`, `context X::op(...) pre:/post: …`,
  `context X::attr : T init: …`.
- Boolean operators: `and`, `or`, `xor`, `not`, `implies`.
- Comparisons: `=`, `<>`, `<`, `>`, `<=`, `>=`.
- Navigation: `self.attr`, association navigation, and `->` collection calls
  (the arrow may be written `->` or the Unicode `→`).
- Collection / iterator ops: `forAll`, `exists`, `select`, `collect`, `size`,
  `isEmpty`, `includes`, and `ClassName::allInstances()`.
- Literals: integer, real, string (single quotes), boolean (`true`/`false`),
  and dates.
- Primitive/collection type refs: `Set<T>`, `Bag<T>`, `Sequence<T>`,
  `OrderedSet<T>`.

## Minimal runnable example

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Constraint, IntegerType,
)

# --- domain ---
age = Property(name="age", type=IntegerType)
employee = Class(name="Employee", attributes={age})

# --- constraint (string-based) ---
adult = Constraint(
    name="employee_is_adult",
    context=employee,
    expression="context Employee inv: self.age > 16",
    language="OCL",
    description="Employees must be older than 16.",
)

# --- assemble ---
model = DomainModel(
    name="Company",
    types={employee},
    constraints={adult},
)
```

## Parsing into an AST (OCLConstraint)

`parse_ocl` turns OCL text into an `OCLConstraint` whose `ast` is the parsed
`OCLExpression` tree and whose `expression` is the pretty-printed source text.

```python
from besser.BUML.notations.ocl import parse_ocl

c = parse_ocl("context Employee inv: self.age > 16", model)
# context_class resolved from the "context X inv|pre|post|init" header
# against model.types, unless context_class=<Class> is passed explicitly.

c.context.name   # "Employee"
c.language        # "OCL"
c.ast             # the parsed OCLExpression AST
c.expression      # pretty-printed source text, e.g. "self.age > 16"
```

Signature:

```python
parse_ocl(text: str, model: DomainModel, context_class: Class = None) -> OCLConstraint
```

- Raises `BOCLSyntaxError` on lex/parse failure (unbalanced parens, unknown
  property, etc.).
- Raises `ValueError` if no `context_class` is given and the text has no
  `context X inv|pre|post|init` header, or the named class is not in `model.types`.

`OCLConstraint(name, context, expression, language="OCL")` is a subclass of
`Constraint`. Its `expression` argument must be an **`OCLExpression` AST**
(not a string) — passing a string raises `TypeError`. The constructor
pretty-prints the AST to populate the inherited string `expression`. Reassigning
`.ast` refreshes `.expression` automatically.

## Validation / evaluation

There is no `model.validate()` for OCL. Two separate mechanisms exist:

- **Syntax-only**: drive the ANTLR lexer/parser (`BOCLLexer`, `BOCLParser`) with
  a `BOCLErrorListener`; `parse_ocl` already does this and raises
  `BOCLSyntaxError` on failure.
- **Evaluation against an object model**: the external `bocl` package
  (`bocl==1.0.1`, the B-OCL-Interpreter) provides `OCLWrapper(dm, om)` whose
  `evaluate(constraint)` returns the boolean result of the constraint against an
  object model `om`. It reads `constraint.expression` and `constraint.context`,
  re-parses, and evaluates.

```python
from bocl.OCLWrapper import OCLWrapper
wrapper = OCLWrapper(domain_model, object_model)
result = wrapper.evaluate(adult)   # True / False; BOCLSyntaxError on bad OCL
```

## AST helpers

Re-exported from both `besser.BUML.metamodel.ocl` and
`besser.BUML.notations.ocl`, for working on a parsed `OCLExpression` tree:

- `walk(node)` — generator yielding every node in post-order (children first).
- `clone(node)` — deep-copy an AST subtree.
- `substitute(expr, var_name, replacement, scope=None)` — capture-avoiding
  replacement of a free variable; returns a fresh tree (input not mutated).
  `ScopeStack` tracks bound names.
- `normalize(constraint, model, max_iterations=1000)` — return a fresh
  `OCLConstraint` rewritten to B-OCL normal form (input not mutated).
- Predicates: `is_op, is_and, is_or, is_xor, is_implies, is_not, is_size,`
  `is_isempty, is_allinstances, is_comparison, is_atomic_type_test, is_loop,`
  `is_loop_with_n_iterators, is_self, is_bool_const`.
- Chain helpers: `is_chain_from_self, walk_chain_from_self,`
  `chain_min_multiplicity, chain_max_multiplicity`.

## Generators that consume constraints

OCL support across generators is **limited**. The notable consumer is the
**Pydantic generator** (`besser.generators.pydantic_classes`), which converts
simple OCL invariants into Pydantic field validators via
`besser/generators/pydantic_classes/ocl_utils.py`:

- It only picks up constraints where `constraint.language == "OCL"`.
- It handles single-property comparisons (`self.x > 5`) and simple compound
  forms (`self.age > 10 and self.age < 20`); falls back to regex parsing when
  the ANTLR parse yields nothing usable. Multi-property and complex iterator
  constraints are skipped for field validation.

The SQL and Supabase generators emit their own `CHECK` constraints from
enums/multiplicities — they do **not** translate the OCL `Constraint`
expressions. Other generators ignore OCL constraints.

## Gotchas

- `language` is positional and required on `Constraint`; constraints are only
  recognized as OCL when `language == "OCL"`.
- `Constraint.expression` must be a `str`; to hold a parsed tree use
  `OCLConstraint` and its `.ast` field.
- `parse_ocl` resolves the context class against `model.types` (classes only).
  An unknown class or a missing header raises `ValueError`, not
  `BOCLSyntaxError`.
- Referencing a property the context class does not have raises
  `BOCLSyntaxError` (message contains "not found") during parsing.
- Evaluation requires the separate `bocl` PyPI package (`bocl==1.0.1`) and an
  object model — it is not part of `besser.BUML`.
- Most generators ignore OCL; do not assume a constraint is enforced unless you
  use the Pydantic generator or evaluate with `OCLWrapper`.
