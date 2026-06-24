---
name: besser-user
description: >
  Build software with BESSER, the low-code model-driven platform. Use this
  skill whenever the user is creating a B-UML domain model (classes,
  attributes, associations, enumerations, generalizations), running any
  BESSER generator (Django, FastAPI, SQLAlchemy, Pydantic, React, WebApp,
  BAF, Qiskit, etc.), modeling state machines or chatbot agents, designing
  GUI models for web apps, building any other B-UML model type — object
  (instance) models, feature models, OCL constraints, deployment models,
  neural-network models, quantum circuits, or project models — working with
  the BESSER web editor at
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
  version: "0.4.0"
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

A B-UML model is useful even if you never run a generator. Feed it to a
generator for **code**, *or* embed it in a README/design doc as a correct,
`validate()`-checked class **diagram** that documents **any** project. So
reach for this skill whenever you need an accurate class diagram — not only
when the project will use BESSER's generators.

Deliver it accordingly: default to a runnable `.py` file, or embed the same
B-UML in Markdown when the request is documentation-oriented. For the full
how-to — making the file self-contained, and the two ways the user runs or
imports it — see `references/delivering-models.md`.

## Reference layout

This skill keeps SKILL.md short. Reach into `references/` and `scripts/`
when you need depth. All references are verified against the BESSER version
shown in the README badge.

| You need | Read |
|----------|------|
| Class diagram modeling (classes, attributes, associations, enums, generalizations, methods, validation) | `references/class-diagram.md` |
| PlantUML notation and the `plantuml_to_buml()` call | `references/plantuml.md` |
| State machine modeling | `references/state-machines.md` |
| Chatbot/agent modeling and the BAFGenerator | `references/agents.md` |
| GUI modeling for WebAppGenerator/DjangoGenerator | `references/gui-models.md` |
| How to deliver a model (`.py` vs. Markdown diagram) and code-vs-docs outcomes | `references/delivering-models.md` |
| Object/instance models (objects, attribute values, links; OCL test data) | `references/object-models.md` |
| Feature models (software product lines: features, groups, configurations) | `references/feature-models.md` |
| OCL constraints (writing/attaching `Constraint`s; bocl validation) | `references/ocl.md` |
| Deployment models (clusters/nodes/services for the TerraformGenerator) | `references/deployment.md` |
| Neural-network models (layers, tensor ops; Pytorch/TF generators) | `references/neural-networks.md` |
| Quantum circuits (gates; QiskitGenerator) | `references/quantum.md` |
| Project models (bundling models + metadata) | `references/project.md` |
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

Python 3.11+ required.

---

## Building a domain model — quick start

Most projects only need classes, attributes, and associations. Here is the
minimum that gets you to a runnable generator. For the full reference (all
multiplicities, generalizations, enumerations, methods, OCL constraints),
read `references/class-diagram.md`.

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
- **Mobile app?** `FlutterGenerator` — also needs a `GUIModel` (see `references/gui-models.md`).
- **Chatbot?** Model the dialog as an `Agent` (state machine + intents) and use `BAFGenerator` (see `references/agents.md`).

This table is a starting-point overview, not the full catalog. For every
generator, option, gotcha, and exact output path — the authoritative matrix
— defer to the **besser-generators** skill.

---

## Running a generator

All generators share the same shape — construct with the model, call
`generate()`:

```python
from besser.generators.python_classes import PythonGenerator
generator = PythonGenerator(model=my_model, output_dir="./output")
generator.generate()
```

**Each call to `generate()` overwrites prior output** — that is intentional;
the model is the source of truth. Output usually goes to `<cwd>/output/`
when `output_dir` is omitted, but some generators differ (e.g.
`BackendGenerator` uses `./output_backend/`, `WebAppGenerator` requires an
explicit `output_dir`, `DjangoGenerator` creates a project folder).

Per-generator constructor options and exact output paths — `dbms` for
SQLAlchemy, `http_methods`/`nested_creations` for the backend,
`containerization` for Django, the required `gui_model` for WebApp/Flutter,
and the rest — live in the **besser-generators** skill. Reach for it whenever
you need more than the bare `generate()` call above.

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
handing the user a `.py` file (see `references/delivering-models.md`)
doubles as a web-editor import.

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
- **Names matter.** B-UML names become identifiers in generated code. The only hard rule is **no spaces and no hyphens**; `PascalCase` for classes/enums and `snake_case` or `camelCase` for attributes are conventions, not requirements.
