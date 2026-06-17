---
name: besser-user
description: >
  Build software with BESSER, the low-code model-driven platform. Use this
  skill whenever the user is creating a B-UML domain model (classes,
  attributes, associations, enumerations, generalizations), running any
  BESSER generator (Django, FastAPI, SQLAlchemy, Pydantic, React, WebApp,
  BAF, Qiskit, etc.), modeling state machines or chatbot agents, designing
  GUI models for web apps, working with the BESSER web editor at
  editor.besser-pearl.org, or drawing a correct UML class diagram to
  document a system (classes, attributes, associations, inheritance) — even
  when no code will be generated, e.g. adding a class diagram to a README,
  design doc, or `.md` spec. Trigger on imports from
  `besser.BUML` or `besser.generators`, mentions of B-UML, DomainModel,
  BinaryAssociation, GUIModel, or any BESSER generator class — even if the
  user does not say "BESSER" by name. Also trigger when the user wants to
  draw, sketch, or document a UML class diagram or data model and wants it
  to be correct, even if BESSER is never mentioned. Prefer this skill over generic Python,
  Django, or FastAPI guidance whenever the project uses BESSER for modeling.
  For per-generator deep dives (output paths, options, customization
  patterns), defer to the besser-generators skill; for errors and
  diagnostics, defer to besser-troubleshooting.
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

# Working with BESSER

BESSER is a model-driven platform: describe your domain as a model, then
generators turn that model into running code. The model is the source of
truth — when requirements change, update the model and regenerate. Never
hand-edit generated code as your primary change.

## Core workflow

```
1. Define requirements
2. Build a B-UML model (Python API, PlantUML, or web editor)
3. Validate the model: model.validate()
4. Pick a generator for your target platform
5. Generate code
6. Verify the output (run, test, inspect)
7. Iterate: update the model, regenerate
```

## Two outcomes: code *or* documentation

A B-UML model is useful even if you never run a generator. There are two
first-class things you can do with one:

1. **Generate code** — feed the model to a generator (Python, SQL, FastAPI,
   Django, React, …). This is BESSER's headline feature.
2. **Document a system** — a B-UML model *is* a correct UML class diagram:
   right multiplicities, associations, and inheritance, checked by
   `model.validate()`. Embed it in a README, design doc, or `.md` spec to
   document **any** project — no code generation required. Import it into
   editor.besser-pearl.org for the rendered visual diagram.

So reach for this skill whenever you need an accurate class diagram, not
only when the project will use BESSER's generators. For how to deliver it
(a `.py` file vs. a Markdown-embedded diagram), see "Delivering the model
to the user" below.

## Reference layout

This skill keeps SKILL.md short. Reach into `references/` and `scripts/`
when you need depth:

| You need | Read |
|----------|------|
| Full B-UML metamodel (classes, attributes, associations, enums, generalizations, methods, validation) | `references/metamodel.md` |
| PlantUML notation and the `plantuml_to_buml()` call | `references/plantuml.md` |
| State machine modeling | `references/state-machines.md` |
| Chatbot/agent modeling and the BAFGenerator | `references/agents.md` |
| GUI modeling for WebAppGenerator/DjangoGenerator | `references/gui-models.md` |
| Per-generator options, output paths, customization | the **besser-generators** skill |
| Errors and diagnostics | the **besser-troubleshooting** skill |

To bootstrap a new model quickly:

```bash
python scripts/scaffold_model.py Library Book Author
# prints ready-to-edit Python that builds a DomainModel with those classes
```

---

## Installation

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

pip install besser
# OR for the latest development version:
git clone https://github.com/BESSER-PEARL/BESSER.git
cd BESSER
pip install -e .
```

Verify:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"
```

Python 3.10+ required.

---

## Building a domain model — quick start

Most projects only need classes, attributes, and associations. Here is the
minimum that gets you to a runnable generator. For the full reference (all
multiplicities, generalizations, enumerations, methods, OCL constraints),
read `references/metamodel.md`.

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)

# Classes
title = Property(name="title", type=StringType)
pages = Property(name="pages", type=IntegerType)
book = Class(name="Book", attributes={title, pages})

author_name = Property(name="name", type=StringType)
author = Class(name="Author", attributes={author_name})

# Association: a Book has 1..* Authors; an Author writes 0..* Books
written_by = Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, "*"))
publishes  = Property(name="publishes", type=book,   multiplicity=Multiplicity(0, "*"))
book_author = BinaryAssociation(name="book_author", ends={written_by, publishes})

model = DomainModel(name="Library", types={book, author}, associations={book_author})
assert model.validate()["success"]
```

**Naming rules**: no spaces, no hyphens. `My_Class` and `my_attribute`,
not `My Class` or `my-attribute`.

PlantUML imports are also supported via `plantuml_to_buml()` — see
`references/plantuml.md` if needed. The Python API is the recommended
path.

---

## Delivering the model to the user

**Choose the format from context — don't ask every time:**

- **Default → a runnable `.py` file** (e.g. `library_model.py`). For almost
  any "build/model X" request, this is the deliverable: a real artifact the
  user can run and import into the web editor.
- **Documentation context → embed the same B-UML in Markdown.** When the
  user is writing docs (a README or design doc, "add the model to the
  docs", "show me the diagram"), put the B-UML in a fenced Python code
  block inside the `.md` instead — same B-UML, for reading rather than
  running.
- **Ambiguous? Don't block on a question.** Write the `.py` file and add
  one line noting the same B-UML can be dropped into their docs.

Whichever you produce, don't just paste code into chat with no artifact.
Make the model self-contained: imports, class and association definitions,
the `DomainModel` assembly, a `model.validate()` check, and a
commented-out generator call the user can uncomment.
(`scripts/scaffold_model.py` already prints code in this shape.)

When you deliver the `.py` file, tell the user they have **two ways to use
it**:

1. **Run it directly** to validate and generate code:

   ```bash
   python library_model.py
   ```

2. **Import it into the web editor** at https://editor.besser-pearl.org —
   use the editor's **Import** option and choose the **B-UML** format to
   load the `.py` model and edit it visually. (The editor imports and
   exports models in B-UML and JSON formats, so a model you write in code
   round-trips with the visual editor.)

---

## Picking a generator

| Goal | Generator | Input | Output |
|------|-----------|-------|--------|
| Python classes | `PythonGenerator` | DomainModel | `classes.py` |
| Java classes | `JavaGenerator` | DomainModel | `.java` files |
| Pydantic models | `PydanticGenerator` | DomainModel | `pydantic_classes.py` |
| SQLAlchemy ORM | `SQLAlchemyGenerator` | DomainModel | `sql_alchemy.py` |
| Raw SQL DDL | `SQLGenerator` | DomainModel | `tables_<dialect>.sql` |
| JSON Schema | `JSONSchemaGenerator` | DomainModel | `json_schema.json` |
| FastAPI backend | `BackendGenerator` | DomainModel | API + ORM + Pydantic |
| Django app | `DjangoGenerator` | DomainModel + optional GUIModel | Django project |
| Full-stack web app | `WebAppGenerator` | DomainModel + GUIModel | React + FastAPI + Docker |
| Conversational agent | `BAFGenerator` | Agent | Agent script + config |
| Quantum circuit | `QiskitGenerator` | QuantumCircuit | `qiskit_circuit.py` |
| RDF vocabulary | `RDFGenerator` | DomainModel | `vocabulary.ttl` |
| Terraform infra | `TerraformGenerator` | DeploymentModel | `.tf` files |
| Neural network | `PytorchGenerator` / `TFGenerator` | NN model | PyTorch/TF script |
| Flutter app | `FlutterGenerator` | DomainModel + GUIModel | Dart files |

### Decision guide

- **Just data classes?** `PythonGenerator` or `PydanticGenerator`.
- **Persistent storage?** `SQLAlchemyGenerator` (ORM) or `SQLGenerator` (raw DDL).
- **REST API?** `BackendGenerator` — gives you FastAPI + SQLAlchemy + Pydantic in one shot.
- **Full web app?** `WebAppGenerator` — React + FastAPI + Docker Compose, but you must also build a `GUIModel` (see `references/gui-models.md`).
- **Django specifically?** `DjangoGenerator`.
- **Chatbot?** Model the dialog as an `Agent` (state machine + intents) and use `BAFGenerator` (see `references/agents.md`).

For per-generator details — every option, every gotcha, every output path
— defer to the **besser-generators** skill.

---

## Running a generator

All generators share the same shape:

```python
from besser.generators.python_classes import PythonGenerator
generator = PythonGenerator(model=my_model, output_dir="./output")
generator.generate()
```

If `output_dir` is omitted, output goes to `<cwd>/output/`. **Each call to
`generate()` overwrites prior output** — that is intentional, the model is
the source of truth.

### SQLAlchemy with a specific DBMS

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator
gen = SQLAlchemyGenerator(model=my_model, output_dir="./output")
gen.generate(dbms="postgresql")
# Valid: sqlite | postgresql | mysql | mssql | mariadb | oracle
# Note: it is "postgresql" not "postgres".
```

### FastAPI backend, with control over endpoints

```python
from besser.generators.backend import BackendGenerator

gen = BackendGenerator(
    model=my_model,
    output_dir="./output_backend",
    http_methods=["GET", "POST", "PUT", "DELETE"],  # subset to limit endpoints
    nested_creations=False,                           # True allows nested creates in payloads
)
gen.generate()
# Produces: main_api.py, pydantic_classes.py, sql_alchemy.py
```

To generate only read+create endpoints:

```python
gen = BackendGenerator(
    model=my_model,
    output_dir="./output_backend",
    http_methods=["GET", "POST"],
)
gen.generate()
```

Invalid method names are silently filtered with a warning log — double-check
spelling.

### Django

```python
from besser.generators.django import DjangoGenerator

gen = DjangoGenerator(
    model=my_model,
    project_name="myproject",
    app_name="myapp",
    output_dir="./output",
    containerization=False,  # True adds docker-compose.yml + Dockerfile
)
gen.generate()
```

Requires `django` installed in the environment (the generator runs
`django-admin startproject` as a subprocess).

### Full-stack web app

Requires both a `DomainModel` and a `GUIModel`:

```python
from besser.generators.web_app import WebAppGenerator

gen = WebAppGenerator(
    model=domain_model,
    gui_model=gui_model,
    output_dir="./webapp_output",
    agent_model=None,        # optional Agent for an embedded chatbot
)
gen.generate()
# Produces: frontend/ (React/Vite), backend/ (FastAPI),
#           optional agent/, docker-compose.yml, Dockerfiles
# Run with: cd webapp_output && docker-compose up --build
```

Default ports: frontend 3000, backend 8000, agent WS 8765.

---

## Using the web editor

The visual editor at https://editor.besser-pearl.org lets users build
models graphically and generate code without writing Python:

1. Create a new diagram (Class, State Machine, GUI, Deployment, …).
2. Add classes, attributes, and associations visually.
3. Click "Generate" and pick a target generator.
4. Download the generated code.

You can also **import an existing model** instead of starting from a
blank diagram: use **Import** and select the **B-UML** format to load a
`.py` model file (or JSON). This means a model you build with the Python
API can be opened in the editor, edited visually, and exported again — so
handing the user a `.py` file (see "Delivering the model" above) doubles
as a web-editor import.

The editor uses the same generators as the Python API — the backend
converts the visual diagram to a B-UML model, runs the generator, and
streams the result. Any generator that registers in `SUPPORTED_GENERATORS`
(see the **besser-dev** skill) shows up in the dropdown.

---

## Verification checklist

After `generate()` returns:

1. **Files exist** in the output directory.
2. **Syntax parses** — for Python: `python -c "import ast; ast.parse(open('output/classes.py').read())"`.
3. **It runs** — for backends: `cd output && pip install -r requirements.txt && uvicorn main_api:app`.
4. **Relationships translate correctly** — foreign keys for 1..*, join tables for *..*, inheritance reflected.
5. **Edge cases** — enumerations, optional fields (`Multiplicity(0, 1)`), inheritance hierarchies.
6. **Docker** — for WebApp/Django containerized: `docker-compose up --build`.

If something is missing or wrong, the **besser-troubleshooting** skill
maps symptoms to fixes.

---

## Key principles

- **Model is the source of truth.** All changes flow model → code, never the reverse.
- **Regeneration overwrites.** Every `generate()` replaces output files. Customizations live in *separate* files (see the besser-generators skill for safe customization patterns).
- **Validate early.** Call `model.validate()` before generating.
- **One model, many targets.** The same `DomainModel` feeds multiple generators — Python classes, SQL, REST API, etc. — so it is rarely worth maintaining target-specific models.
- **Names matter.** B-UML names become identifiers in generated code: `snake_case` for attributes, `PascalCase` for classes, no spaces or hyphens.
