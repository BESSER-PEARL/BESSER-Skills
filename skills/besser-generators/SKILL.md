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
  JSONSchemaGenerator, RDFGenerator, TerraformGenerator, PytorchGenerator,
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
  version: "0.1.0"
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

---

## Reference layout

To keep this skill scannable, per-generator detail lives in `references/`.
Read the relevant file when the user is working with a specific generator:

| If the user is running… | Read |
|--------------------------|------|
| `PythonGenerator`, `JavaGenerator`, `PydanticGenerator`, `JSONSchemaGenerator`, `RDFGenerator` | `references/python-and-data.md` |
| `SQLAlchemyGenerator`, `SQLGenerator` | `references/persistence.md` |
| `BackendGenerator`, `RESTAPIGenerator`, `DjangoGenerator`, `WebAppGenerator`, `ReactGenerator`, `FlutterGenerator` | `references/api-and-web.md` |
| `BAFGenerator`, `QiskitGenerator`, `TerraformGenerator`, `PytorchGenerator`, `TFGenerator` | `references/agents-and-other.md` |
| Anything failing or producing wrong output | `references/debugging.md` |

Each reference file documents inputs, outputs, options, templates, and
known gotchas for the generators it covers.

---

## Generator picker

| Want | Generator | Where it lives |
|------|-----------|----------------|
| Python data classes | `PythonGenerator` | `python-and-data.md` |
| Java classes | `JavaGenerator` | `python-and-data.md` |
| Pydantic models (with optional OCL→validator) | `PydanticGenerator` | `python-and-data.md` |
| JSON Schema (regular or NGSI-LD) | `JSONSchemaGenerator` | `python-and-data.md` |
| RDF Turtle vocabulary | `RDFGenerator` | `python-and-data.md` |
| SQLAlchemy ORM | `SQLAlchemyGenerator` | `persistence.md` |
| Raw SQL DDL | `SQLGenerator` | `persistence.md` |
| FastAPI + ORM + Pydantic stack | `BackendGenerator` | `api-and-web.md` |
| Standalone REST API | `RESTAPIGenerator` | `api-and-web.md` |
| Django project | `DjangoGenerator` | `api-and-web.md` |
| Full-stack web app (React + FastAPI + Docker) | `WebAppGenerator` | `api-and-web.md` |
| Flutter mobile app | `FlutterGenerator` | `api-and-web.md` |
| Conversational agent / chatbot | `BAFGenerator` | `agents-and-other.md` |
| Quantum circuit | `QiskitGenerator` | `agents-and-other.md` |
| Terraform infrastructure | `TerraformGenerator` | `agents-and-other.md` |
| PyTorch / TensorFlow neural net | `PytorchGenerator` / `TFGenerator` | `agents-and-other.md` |

## Output directory summary

| Generator | Default output_dir | Output structure |
|-----------|--------------------|------------------|
| PythonGenerator | `./output/` | `classes.py` |
| JavaGenerator | `./output/` | `Book.java`, `Author.java`, … |
| PydanticGenerator | `./output/` | `pydantic_classes.py` |
| SQLAlchemyGenerator | `./output/` | `sql_alchemy.py` |
| SQLGenerator | `./output/` | `tables_<dialect>.sql` |
| BackendGenerator | `./output_backend/` | `main_api.py`, `pydantic_classes.py`, `sql_alchemy.py` |
| DjangoGenerator | CWD (creates project folder) | `myproject/myapp/models.py`, `views.py`, … |
| WebAppGenerator | must be specified | `frontend/`, `backend/`, `docker-compose.yml` |
| BAFGenerator | `./output/` | `{agent_name}.py`, `config.ini`, `readme.txt` |

`BackendGenerator`'s default `./output_backend/` is the odd one out — easy
to miss if you assume `./output/` like everyone else.

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
from sql_alchemy import Book, Author, Session

def get_books_by_author(author_name: str):
    with Session() as session:
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
from besser.BUML.metamodel.structural import Method, Parameter, StringType
from besser.BUML.metamodel.action_language import (
    MethodImplementationType, MethodImplementation,
)

search = Method(
    name="search_by_title",
    parameters=[Parameter(name="keyword", type=StringType)],
    type=StringType,
)
search.implementation = MethodImplementation(
    type=MethodImplementationType.BAL,
    body='return session.query(Book).filter(Book.title.contains(keyword)).all()',
)
book.add_method(search)
```

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

Several generators orchestrate sub-generators. When debugging, trace which
sub-generator is actually failing — the error rarely lives in the wrapper.

```
WebAppGenerator
  ├── ReactGenerator          → frontend/
  ├── BackendGenerator        → backend/
  │     ├── RESTAPIGenerator  → main_api.py
  │     ├── PydanticGenerator → pydantic_classes.py
  │     └── SQLAlchemyGenerator → sql_alchemy.py
  └── BAFGenerator (optional) → agent/

SQLGenerator
  └── SQLAlchemyGenerator (temp) → subprocess → .sql file
```

For debugging recipes per symptom, see `references/debugging.md`.

---

## Web editor registry

Generators that should appear in the web editor's "Generate" dropdown must
be registered in
`besser/utilities/web_modeling_editor/backend/config/generators.py`. Two
hooks:

```python
# 1) Add to SUPPORTED_GENERATORS:
SUPPORTED_GENERATORS = {
    "my_generator": GeneratorInfo(
        generator_class=MyGenerator,
        output_type="file",           # or "zip" for multi-file
        file_extension=".py",
        category="my_category",
        requires_class_diagram=True,
    ),
}

# 2) Add to get_filename_for_generator():
filenames = {
    "my_generator": "my_output.py",
    ...
}
```

Both registrations are required for the dropdown to populate. For the full
authoring workflow (package layout, tests, docs, PR), see the **besser-dev**
skill.
