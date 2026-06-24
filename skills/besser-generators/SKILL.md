---
name: besser-generators
description: >
  Operational reference for BESSER code generators — covers per-generator
  options, generated file layout, regeneration/overwrite behavior, safe
  customization patterns, template overrides, and debugging generation
  failures. Use this skill whenever the user is configuring or running a
  BESSER generator (PythonGenerator, PydanticGenerator, SQLAlchemyGenerator,
  SQLGenerator, BackendGenerator, RESTAPIGenerator, DjangoGenerator,
  WebAppGenerator, ReactGenerator, BAFGenerator, QiskitGenerator,
  JSONSchemaGenerator, JSONObjectGenerator, RDFGenerator, SupabaseGenerator,
  TerraformGenerator, PytorchGenerator,
  TFGenerator, FlutterGenerator, JavaGenerator), wondering "where does the
  output go", "will my edits survive regeneration", "how do I add custom
  endpoints to a generated FastAPI app", or "how do I switch the database
  dialect". Trigger on questions about generator parameters (`http_methods`,
  `nested_creations`, `dbms`, `containerization`, `backend_type`, `shots`,
  `generation_mode`), generated file paths, template overrides, or how to
  extend generated code without losing edits across regeneration cycles.
  Prefer this skill over besser-user when the question is about a specific
  generator's behavior rather than how to build a model.
license: Apache-2.0
compatibility:
  - claude-code
  - cursor
  - cline
  - windsurf
  - copilot
metadata:
  author: BESSER-PEARL
  version: "0.3.0"
  repository: https://github.com/BESSER-PEARL/BESSER-Skills
---

# BESSER Generator Operations

Every BESSER generator implements the same shape:

```python
class GeneratorInterface(ABC):
    def __init__(self, model, output_dir=None): ...
    def generate(self): ...
```

If `output_dir` is omitted, output goes to `<cwd>/output/`. The directory
is created automatically (`os.makedirs(..., exist_ok=True)`).

**Regeneration always overwrites.** Every call to `generate()` replaces the
output files entirely. This is by design — the model is the source of truth.
Customizations belong in *separate files* (see "Safe customization" below).
Once you have generated, the **besser-user** skill's "Verification checklist"
covers what to check next (files exist, syntax parses, it runs).

---

## Generator picker

Per-generator detail (inputs, outputs, options, templates, gotchas) lives in
`references/`. Find your target, then read the listed reference:

| Want | Generator | Reference |
|------|-----------|-----------|
| Python data classes | `PythonGenerator` | `python-and-data.md` |
| Java classes | `JavaGenerator` | `python-and-data.md` |
| Pydantic models (with optional OCL→validator) | `PydanticGenerator` | `python-and-data.md` |
| JSON Schema (regular or NGSI-LD) | `JSONSchemaGenerator` | `python-and-data.md` |
| JSON data from an ObjectModel (instances) | `JSONObjectGenerator` | `python-and-data.md` |
| RDF Turtle vocabulary | `RDFGenerator` | `python-and-data.md` |
| SQLAlchemy ORM | `SQLAlchemyGenerator` | `persistence.md` |
| Raw SQL DDL | `SQLGenerator` | `persistence.md` |
| Supabase (Postgres + RLS) schema | `SupabaseGenerator` | `persistence.md` |
| FastAPI + ORM + Pydantic stack | `BackendGenerator` | `api-and-web.md` |
| Standalone REST API | `RESTAPIGenerator` | `api-and-web.md` |
| Django project | `DjangoGenerator` | `api-and-web.md` |
| Full-stack web app (React + FastAPI + Docker) | `WebAppGenerator` | `api-and-web.md` |
| Flutter mobile app | `FlutterGenerator` | `api-and-web.md` |
| Conversational agent / chatbot | `BAFGenerator` | `agents-and-other.md` |
| Quantum circuit | `QiskitGenerator` | `agents-and-other.md` |
| Terraform infrastructure | `TerraformGenerator` | `agents-and-other.md` |
| PyTorch / TensorFlow neural net | `PytorchGenerator` / `TFGenerator` | `agents-and-other.md` |

Not generating, but stuck?
- A generator runs but produces wrong or empty output → `references/debugging.md`.
- Installation / import errors (not generation itself) → the **besser-troubleshooting** skill.

## Output locations

By default a generator writes to `<cwd>/output/` and produces the single
file documented in its reference. The exceptions worth memorizing:

- **`BackendGenerator`** → defaults to `./output_backend/` (not `./output/`); emits `main_api.py`, `pydantic_classes.py`, `sql_alchemy.py`.
- **`WebAppGenerator`** → `output_dir` must be specified; emits `frontend/`, `backend/`, `docker-compose.yml`.
- **`DjangoGenerator`** → creates a project folder in the CWD (`myproject/myapp/models.py`, …).
- **`SupabaseGenerator`** → timestamped filename, so every run creates a *new* file and never overwrites.

For the exact output filename of any other generator, see its row in the
relevant `references/` file.

---

## Safe customization patterns

Since `generate()` overwrites everything, customizations need a strategy
that survives regeneration.

### 1. Generator options (try this first)

Many "I need to tweak the output" requests are already covered by
parameters:

- `SQLAlchemyGenerator.generate(dbms="postgresql")` — switch dialect
- `BackendGenerator(http_methods=["GET", "POST"])` — limit endpoints
- `BackendGenerator(nested_creations=True)` — allow nested creates
- `DjangoGenerator(containerization=True)` — add Docker support
- `QiskitGenerator(backend_type="ibm_quantum", shots=2048)`
- `BAFGenerator(generation_mode=GenerationMode.CODE_ONLY)`

### 2. Post-generation extension files

Put custom code in **separate files** that import from generated code.
Generated files are overwritten; your extension files are not.

```
output/
  sql_alchemy.py       # GENERATED — do not edit
  custom_queries.py    # YOUR CODE — imports from sql_alchemy.py
  custom_endpoints.py  # YOUR CODE — imports from main_api.py
```

```python
# custom_queries.py — safe from regeneration
# The generated sql_alchemy.py exposes `Base` and `engine` (not a Session);
# create the Session yourself from SQLAlchemy.
from sqlalchemy.orm import Session
from sql_alchemy import Book, Author, engine

def get_books_by_author(author_name: str):
    with Session(engine) as session:
        return session.query(Book).join(Author).filter(Author.name == author_name).all()
```

### 3. Wrapper scripts for backends

Mount the generated FastAPI app and add your routes on top:

```python
# app.py — YOUR CODE, not generated
from main_api import app

@app.get("/custom/stats")
def custom_stats():
    return {"status": "ok"}
```

### 4. Model-level custom endpoints (BAL / CODE)

Often the cleanest fix: define methods on your model classes with
`MethodImplementationType.BAL` or `MethodImplementationType.CODE`, and
`BackendGenerator` emits matching REST endpoints. Custom logic stays in
the model, where regeneration cannot touch it.

```python
from besser.BUML.metamodel.structural import (
    Method, Parameter, StringType, MethodImplementationType,
)

# Pass the implementation via the Method constructor: `code` holds the body
# and `implementation_type` selects BAL or CODE. (There is no separate
# MethodImplementation class, and the body is not assigned afterwards.)
search = Method(
    name="search_by_title",
    parameters=[Parameter(name="keyword", type=StringType)],
    type=StringType,
    code="return session.query(Book).filter(Book.title.contains(keyword)).all()",
    implementation_type=MethodImplementationType.BAL,
)
book.add_method(search)
```

The method body is inserted **verbatim** into the generated FastAPI handler
— it is not parsed or scoped by the metamodel. So any name it references
(here `session`) must actually exist in the generated handler at that point.
Treat the body as code you are responsible for: check the generated
`main_api.py` to confirm what is in scope (e.g. the SQLAlchemy session
variable name the handler uses) and reference exactly those names.

### 5. Template overrides (advanced)

Copy a generator's template, edit it, and point the generator at your
copy. Templates live in `besser/generators/{name}/templates/`. Powerful,
but BESSER updates won't propagate to your overrides — use sparingly.

### 6. Git-based patch workflow (small repeatable edits)

```bash
git diff output/ > my_customizations.patch          # save once
cd output && git apply ../my_customizations.patch   # reapply after each generate
```

Works for small, stable changes. For larger customizations, prefer #2 or #4.

### What NOT to do

- **Don't edit generated files directly** as your primary workflow — they
  will be overwritten next time you generate.
- **Don't fork a generator** to add one small feature. Generator options
  or template overrides almost always suffice.
- **Don't mix generated and hand-written code in the same file.**
  Separation keeps your code safe across regeneration cycles.

---

## Composite generator architecture

Several generators orchestrate sub-generators (e.g. `WebAppGenerator` drives
`ReactGenerator` + `BackendGenerator` + optional `BAFGenerator`;
`BackendGenerator` drives `RESTAPIGenerator` + `PydanticGenerator` +
`SQLAlchemyGenerator`; `SQLGenerator` drives `SQLAlchemyGenerator` in a
subprocess). When debugging, trace which sub-generator is actually failing —
the error rarely lives in the wrapper. The full tree and per-symptom recipes
are in `references/debugging.md`.

---

## Web editor registry

A generator only appears in the web editor's "Generate" dropdown once it is
registered in
`besser/utilities/web_modeling_editor/backend/config/generators.py` via two
hooks: an entry in `SUPPORTED_GENERATORS` and its filename in
`get_filename_for_generator()`. Both are required for the dropdown to
populate. For the annotated `GeneratorInfo` snippet and the full authoring
workflow (package layout, tests, docs, PR), see the **besser-dev** skill's
`references/adding-a-generator.md`.
