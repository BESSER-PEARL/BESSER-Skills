---
name: besser-generators
description: >
  Operational reference for BESSER code generators — output locations, overwrite
  behavior, per-generator configuration, template customization, and debugging
  generation failures. Covers specific generator behavior (Django, SQLAlchemy,
  Backend, WebApp, React, etc.), generated file structures, safe customization
  patterns, template overrides, and troubleshooting generation errors.
license: Apache-2.0
compatibility:
  - claude-code
  - cursor
  - cline
  - windsurf
  - copilot
metadata:
  author: BESSER-PEARL
  version: "0.2.0"
  repository: https://github.com/BESSER-PEARL/besser-skills
---

# BESSER Generator Operations

Every BESSER generator implements the same interface:

```python
class GeneratorInterface(ABC):
    def __init__(self, model, output_dir=None): ...
    def generate(self): ...
```

If `output_dir` is omitted, output goes to `<cwd>/output/`. The directory is
created automatically via `os.makedirs(..., exist_ok=True)`.

**Regeneration always overwrites.** Every call to `generate()` replaces the
output files entirely. This is by design — the model is the source of truth.

---

## Generator Quick Reference

### PythonGenerator

```python
from besser.generators.python_classes import PythonGenerator
gen = PythonGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `classes.py` |
| Template | `python_classes_template.py.j2` |
| Key behavior | Generates all classes with bidirectional relationship setters. Classes sorted by inheritance for correct definition order. |

---

### JavaGenerator

```python
from besser.generators.java_classes import JavaGenerator
gen = JavaGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | One `.java` file per class (e.g., `Book.java`, `Author.java`) |
| Template | `java_template.py.j2` |
| Key behavior | Package name auto-detected from output path. Files sorted by inheritance. |

---

### PydanticGenerator

```python
from besser.generators.pydantic_classes import PydanticGenerator
gen = PydanticGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `pydantic_classes.py` |
| Options | `backend=True` switches to backend API mode; `nested_creations=True` allows nested object creation in API payloads |
| Template | `pydantic_classes_template.py.j2` |
| Key behavior | Includes OCL constraint → Pydantic validator conversion. Unicode-safe via `ascii_identifier` filter. |

---

### SQLAlchemyGenerator

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator
gen = SQLAlchemyGenerator(model=domain_model, output_dir="./output")
gen.generate(dbms="sqlite")  # sqlite | postgresql | mysql | mssql | mariadb | oracle
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `sql_alchemy.py` |
| Template | `sql_alchemy_template.py.j2` |
| Key behavior | Handles `AssociationClass` as separate tables. Detects abstract parents for concrete table inheritance. Enumerations added as SQLAlchemy `Enum` types. |
| Error | `ValueError` if DBMS string is not one of the six valid options. |

---

### SQLGenerator

```python
from besser.generators.sql import SQLGenerator
gen = SQLGenerator(model=domain_model, output_dir="./output")
gen.generate(sql_dialect="sqlite")
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `tables_{dialect}.sql` |
| Key behavior | Two-stage: first generates SQLAlchemy in a temp directory, then runs it as a subprocess to dump DDL. Requires Python to be callable as a subprocess. |
| Gotcha | If the intermediate SQLAlchemy file has issues, the subprocess fails silently (error printed to stdout, not raised). |

---

### BackendGenerator

```python
from besser.generators.backend import BackendGenerator
gen = BackendGenerator(model=domain_model, output_dir="./output_backend")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Directory with 3 files: `main_api.py` (FastAPI), `pydantic_classes.py`, `sql_alchemy.py` |
| Options | `http_methods` (default: all of GET/POST/PUT/DELETE), `nested_creations`, `docker_image=True` to build/push Docker image |
| Key behavior | Orchestrates 3 sub-generators (RESTAPIGenerator, PydanticGenerator, SQLAlchemyGenerator). Optionally generates `Dockerfile` + `requirements.txt` + `.dockerignore`. |
| Default output_dir | `<cwd>/output_backend/` (not `output/`) |

---

### RESTAPIGenerator

```python
from besser.generators.rest_api import RESTAPIGenerator
gen = RESTAPIGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | `rest_api.py` + `pydantic_classes.py` + `requirements.txt` (standalone mode), or `main_api.py` + `requirements.txt` (backend mode) |
| Options | `http_methods`, `nested_creations`, `backend=True` (used internally by BackendGenerator), `port` |
| Key behavior | Invalid HTTP method names are silently filtered with a warning log — no error raised. |

---

### DjangoGenerator

```python
from besser.generators.django import DjangoGenerator
gen = DjangoGenerator(
    model=domain_model,
    project_name="myproject",
    app_name="myapp",
    output_dir="./output",
    gui_model=None,          # optional GUIModel for HTML templates
    containerization=False,  # True adds docker-compose.yml + Dockerfile
)
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` (required) + `GUIModel` (optional) |
| Output | Full Django project tree: `models.py`, `urls.py`, `views.py`, `forms.py`, `admin.py`, HTML templates, `requirements.txt` |
| Templates | 12+ Jinja2 templates |
| Key behavior | Runs `django-admin startproject` + `manage.py startapp` as subprocess, then populates files. Modifies `settings.py` in-place. |
| Warning | Prints `"Warning: No main page found."` if GUIModel has no main page — this is not an error, the generator continues. |
| Gotcha | Requires `django` installed in the environment. Subprocess failures are caught but not re-raised. |

---

### WebAppGenerator

```python
from besser.generators.web_app import WebAppGenerator
gen = WebAppGenerator(
    model=domain_model,
    gui_model=gui_model,       # required
    output_dir="./webapp",
    agent_model=None,          # optional Agent for chatbot
)
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` + `GUIModel` (both required), optional `Agent` |
| Output | `frontend/` (React/Vite), `backend/` (FastAPI), optional `agent/`, `docker-compose.yml`, Dockerfiles |
| Ports | Frontend: 3000, Backend: 8000, Agent WS: 8765, Agent HTTP: 5000 |
| Key behavior | Orchestrates ReactGenerator + BackendGenerator + optional BAFGenerator, then generates Docker orchestration. |
| Run | `cd webapp && docker-compose up --build` |

---

### BAFGenerator (Agent)

```python
from besser.generators.agents import BAFGenerator
gen = BAFGenerator(model=agent, output_dir="./agent_output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `Agent` (from `besser.BUML.metamodel.state_machine.agent`) |
| Output | `{agent_name}.py`, `config.ini`, `readme.txt`, optional RAG subdirectories |
| Options | `generation_mode` (FULL, PERSONALIZED_ONLY, CODE_ONLY), `config_path`/`config` for personalization, `openai_api_key` for LLM personalization |
| Known limits | Telegram handler not fully implemented. Platform names hardcoded in template. |

---

### QiskitGenerator

```python
from besser.generators.qiskit import QiskitGenerator
gen = QiskitGenerator(model=circuit, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `QuantumCircuit` (from `besser.BUML.metamodel.quantum`) |
| Output | Single file: `qiskit_circuit.py` |
| Options | `backend_type` (`aer_simulator`, `fake_backend`, `ibm_quantum`), `shots` (default 1024) |
| Error | `ValueError` if backend_type is invalid. |
| Note | Requires `qiskit` installed to *run* the generated code (not needed for generation). |

---

### JSONSchemaGenerator

```python
from besser.generators.json import JSONSchemaGenerator
gen = JSONSchemaGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | `json_schema.json` (regular mode), or per-class directories with schema + examples (smart_data mode) |
| Options | `mode="regular"` or `mode="smart_data"` (FIWARE/NGSI-LD compliant) |

---

### Other Generators (not in web editor registry)

| Generator | Input | Output | Notes |
|-----------|-------|--------|-------|
| `RDFGenerator` | DomainModel | `vocabulary.ttl` | RDF Turtle format |
| `TerraformGenerator` | DeploymentModel | `.tf` files per cluster | GCP and AWS supported |
| `PytorchGenerator` | NN model | PyTorch script | `generation_type`: `"subclassing"` or `"sequential"` |
| `TFGenerator` | NN model | TensorFlow script | Same options as PyTorch |
| `FlutterGenerator` | DomainModel + GUIModel | `main.dart`, `sql_helper.dart`, `pubspec.yaml` | Not in web registry |
| `ReactGenerator` | DomainModel + GUIModel | Full React/Vite project | Used internally by WebAppGenerator |

---

## Output Directory Summary

| Generator | Default output_dir | Output structure |
|-----------|--------------------|------------------|
| PythonGenerator | `./output/` | `classes.py` |
| JavaGenerator | `./output/` | `Book.java`, `Author.java`, ... |
| PydanticGenerator | `./output/` | `pydantic_classes.py` |
| SQLAlchemyGenerator | `./output/` | `sql_alchemy.py` |
| SQLGenerator | `./output/` | `tables_sqlite.sql` |
| BackendGenerator | `./output_backend/` | `main_api.py`, `pydantic_classes.py`, `sql_alchemy.py` |
| DjangoGenerator | CWD (creates project folder) | `myproject/myapp/models.py`, `views.py`, ... |
| WebAppGenerator | must be specified | `frontend/`, `backend/`, `docker-compose.yml` |
| BAFGenerator | `./output/` | `{name}.py`, `config.ini`, `readme.txt` |

---

## Safe Customization Patterns

Since regeneration overwrites all generated files, you need strategies to preserve
customizations across generation cycles.

### Pattern 1: Generator Options

Many generators accept configuration that controls output. Use these before
resorting to manual edits:

- `SQLAlchemyGenerator.generate(dbms="postgresql")` — switch database dialect
- `BackendGenerator(http_methods=["GET", "POST"])` — limit API endpoints
- `BackendGenerator(nested_creations=True)` — allow nested object creation
- `DjangoGenerator(containerization=True)` — add Docker support
- `QiskitGenerator(backend_type="ibm_quantum", shots=2048)`
- `BAFGenerator(generation_mode=GenerationMode.CODE_ONLY)`

### Pattern 2: Post-Generation Extension

Put your custom code in **separate files** that import from generated code.
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

### Pattern 3: Wrapper Scripts

For backends, create a wrapper that mounts the generated API and adds your routes:

```python
# app.py — YOUR CODE, not generated
from main_api import app  # import the generated FastAPI app

@app.get("/custom/stats")
def custom_stats():
    return {"status": "ok"}
```

### Pattern 4: Model-Level Custom Endpoints (BAL/CODE)

For the `BackendGenerator`, you can define methods on your model classes with
`MethodImplementationType.BAL` or `MethodImplementationType.CODE` to auto-generate
custom REST endpoints — no wrapper scripts needed.

```python
from besser.BUML.metamodel.structural import (
    Method, Parameter, StringType, IntegerType,
)
from besser.BUML.metamodel.action_language import (
    MethodImplementationType, MethodImplementation,
)

# Define a method with BAL implementation
search_method = Method(
    name="search_by_title",
    parameters=[Parameter(name="keyword", type=StringType)],
    type=StringType,
)
# Attach a BAL implementation
search_method.implementation = MethodImplementation(
    type=MethodImplementationType.BAL,
    body='return session.query(Book).filter(Book.title.contains(keyword)).all()',
)
book.add_method(search_method)
```

When `BackendGenerator` encounters methods with implementations, it generates
corresponding endpoints in `main_api.py` automatically. This keeps custom logic
in the model rather than in fragile post-generation edits.

### Pattern 5: Template Overrides

For advanced cases, you can copy a generator's template, modify it, and point
the generator at your version. Templates live in `besser/generators/{name}/templates/`.

This is the most powerful approach but also the most fragile — template changes
in BESSER updates won't propagate to your overrides. Use sparingly.

### Pattern 6: Git-Based Patch Workflow

For small, repeatable edits to generated files:

```bash
# After first generation, make your edits and save a patch
git diff output/ > my_customizations.patch

# After regeneration, reapply
cd output && git apply ../my_customizations.patch
```

This works for small, stable changes. For larger customizations, prefer the
extension pattern above.

### What NOT to Do

- **Don't edit generated files directly** as your primary workflow. They'll be
  overwritten next time you generate.
- **Don't fork a generator** to add one small feature. Use generator options or
  template overrides instead.
- **Don't mix generated and hand-written code in the same file.** Separation
  keeps your code safe across regeneration cycles.

---

## Debugging Generation Failures

### Generator produces empty or wrong output

1. **Validate the model first**: `model.validate()` — if it returns errors,
   fix those before generating.
2. **Check class names**: Names with spaces or hyphens cause `ValueError` at
   model construction time, not at generation time.
3. **Check associations**: Each `BinaryAssociation` needs exactly 2 ends. Each
   end's `type` must point to a class that's in the model's `types` set.
4. **Check inheritance**: Circular inheritance (`A → B → A`) will be caught by
   `model.validate()`.

### SQLAlchemy generation issues

- **"Invalid DBMS"**: Only `sqlite`, `postgresql`, `mysql`, `mssql`, `mariadb`, `oracle` are valid. **Note:** the BESSER documentation at `docs/source/generators/sql.rst` incorrectly says `postgres` — the correct value is `postgresql`.
- **Missing primary keys**: If no attribute has `is_id=True`, SQLAlchemy auto-generates an `id` column. If you need control, explicitly mark one attribute as `is_id=True`.
- **Association tables**: Many-to-many relationships generate a separate association table. The name is derived from the two class names.

### Django generation issues

- **Django not installed**: The generator runs `django-admin` as a subprocess. If Django isn't in your environment, it fails.
- **"Warning: No main page found"**: The GUIModel has no screen with `is_main_page=True`. The generator continues but may produce incomplete views.
- **Subprocess errors**: Django admin failures are caught but not re-raised — check the console output for details.

### SQL generation issues

- **Silent failure**: `SQLGenerator` runs a subprocess internally. If it fails, the error is printed but not raised — you get no output file with no exception. Check stdout.
- **Missing tables in output**: If associations reference classes not in the model, the intermediate SQLAlchemy file may be broken.

### WebApp generation issues

- **Missing GUIModel**: Both `DomainModel` and `GUIModel` are required. Passing `None` for `gui_model` will cause an `AttributeError`.
- **Port conflicts**: Default ports are 3000 (frontend), 8000 (backend), 8765 (agent WS). If these are in use, edit the generated `docker-compose.yml`.
- **Docker not running**: `docker-compose up` requires Docker daemon to be running.

### Backend generation issues

- **Invalid HTTP methods**: Unknown method names are silently filtered — the generator logs a warning but doesn't raise an error. Check your spelling.
- **Docker image push fails**: Requires `docker` Python package and Docker Hub credentials in a config INI file.

### Template rendering errors

If you see `jinja2.TemplateNotFoundError` or `jinja2.UndefinedError`:
- The generator's template directory may be missing or corrupted.
- Reinstall BESSER: `pip install -e .`
- Check that `besser/generators/{name}/templates/` contains the expected `.j2` files.

---

## Composite Generator Architecture

Several generators are composites that orchestrate sub-generators:

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

When debugging composite generators, trace which sub-generator is failing. The
error often originates in a specific sub-generator, not in the orchestrator.

---

## Web Editor Generator Registry

When a generator is used through the web editor, it must be registered in
`besser/utilities/web_modeling_editor/backend/config/generators.py`.

### SUPPORTED_GENERATORS

The `SUPPORTED_GENERATORS` dict maps generator keys to `GeneratorInfo` metadata:

```python
from besser.generators.my_generator import MyGenerator

SUPPORTED_GENERATORS = {
    # ... existing generators ...
    "my_generator": GeneratorInfo(
        generator_class=MyGenerator,
        output_type="file",           # "file" for single file, "zip" for multi-file
        file_extension=".py",
        category="my_category",
        requires_class_diagram=True,
    ),
}
```

### get_filename_for_generator()

This function maps generator keys to output filenames. If your generator
produces a specific filename, register it here:

```python
def get_filename_for_generator(generator_key: str) -> str:
    filenames = {
        "python": "classes.py",
        "sql_alchemy": "sql_alchemy.py",
        "pydantic": "pydantic_classes.py",
        # ... add your generator's output filename ...
        "my_generator": "my_output.py",
    }
    return filenames.get(generator_key, "output")
```

Both registrations are required for a generator to appear in the web editor's
"Generate" dropdown.
