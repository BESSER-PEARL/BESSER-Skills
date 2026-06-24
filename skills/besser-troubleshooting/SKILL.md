---
name: besser-troubleshooting
description: >
  Diagnose and fix BESSER errors fast. Use this skill whenever the user is
  staring at a Python traceback, ImportError, ModuleNotFoundError, ValueError,
  TypeError, AttributeError, jinja2.TemplateNotFound,
  subprocess.CalledProcessError, or any other failure originating from
  BESSER (besser.BUML, besser.generators, besser.utilities). Covers
  installation failures (`pip install besser` errors, native dependency
  build failures for psycopg2/pyodbc/oracledb, Python version mismatches,
  Windows venv path quirks), import errors (`String` vs `StringType`,
  missing `bocl==1.0.1`, `antlr4-python3-runtime` version mismatch), model
  construction errors (spaces or hyphens in names, duplicate enum literals,
  invalid multiplicities, generalization-to-self, more than one is_id per
  class), generator crashes (Invalid DBMS, Django subprocess failures,
  missing GUIModel for WebApp, silent SQLGenerator failures, invalid Qiskit
  backend), Docker and deployment problems (port conflicts, docker-compose
  vs docker compose, missing system libs in slim images), OCL constraint
  parse failures, and model-to-output drift (changes not appearing in
  regenerated code). Trigger on any error message that mentions `besser`
  in the traceback, even if the user does not explicitly ask for
  troubleshooting. Provides a minimal-reproduction template and quick
  diagnostic commands; defer to besser-generators for non-error questions
  about specific generator behavior.
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

# BESSER Troubleshooting

Quick reference for diagnosing and fixing common issues. Organized by
symptom. This is an intentionally flat, single-file skill — a fast lookup
you scan top-to-bottom or search, with no `references/` subdirectory.

---

## Installation Issues

### `pip install besser` fails with native dependency errors

Several BESSER dependencies require system libraries:

| Dependency | Requires | Fix (Ubuntu/Debian) | Fix (macOS) |
|------------|----------|---------------------|-------------|
| `psycopg2-binary` | PostgreSQL client libs | `apt install libpq-dev` | `brew install postgresql` |
| `pyodbc` | ODBC driver manager | `apt install unixodbc-dev` | `brew install unixodbc` |
| `oracledb` | Oracle Instant Client | Download from Oracle | Download from Oracle |

If you don't need a specific database, these won't block core BESSER functionality —
most generators work fine with SQLite (no native deps needed).

### `pip install -e .` fails or installs nothing

- Must be run from the repository root (where `setup.cfg` and `pyproject.toml` live; `setup.cfg` holds the package metadata and `python_requires`).
- Requires `setuptools>=42` and `wheel`. Upgrade first: `pip install --upgrade setuptools wheel`.
- Check you're in the right virtual environment: `which python` (Unix) or `where python` (Windows).

### Python version incompatibility

BESSER requires **Python 3.11+** (enforced by `python_requires = >=3.11` in setup.cfg). Check with:

```bash
python --version
```

If you're on 3.10 or earlier, upgrade Python or use `pyenv`/`conda` to manage versions.

### Windows-specific: `venv\Scripts\activate` not found

The activation path is `venv\Scripts\activate` (capital S). A common mistake in
some docs is `venv\Script\activate` (missing the 's') — this will fail.

```cmd
:: Correct:
venv\Scripts\activate
```

---

## Import Errors

### `ModuleNotFoundError: No module named 'besser'`

1. **Not installed**: Run `pip install -e .` from the repo root, or `pip install besser`.
2. **Wrong environment**: Activate your venv first (`source venv/bin/activate` or `venv\Scripts\activate`).
3. **Multiple Python installs**: `pip` and `python` may point to different installations. Use `python -m pip install -e .` to ensure alignment.

### `ImportError: cannot import name 'X' from 'besser.BUML.metamodel.structural'`

Check the exact name. Common mistakes:

| Wrong | Correct |
|-------|---------|
| `String` | `StringType` |
| `Integer` | `IntegerType` |
| `Float` | `FloatType` |
| `Boolean` | `BooleanType` |
| `Date` | `DateType` |
| `DateTime` | `DateTimeType` |
| `Association` | `BinaryAssociation` (for 2-end associations) |

The primitive types are **singleton instances**, not classes. You import them
directly: `from besser.BUML.metamodel.structural import StringType`.

### `ImportError` related to `bocl`

The OCL constraint checker requires `bocl==1.0.1` (exact version). If you installed
a different version or it's missing:

```bash
pip install bocl==1.0.1
```

This only affects OCL validation — generators work without it.

### `ImportError` related to `antlr4`

The PlantUML parser requires `antlr4-python3-runtime>=4.13.1`. The runtime version
must be compatible with the grammar files shipped with BESSER. Install:

```bash
pip install antlr4-python3-runtime>=4.13.1
```

If you get runtime parse errors after upgrading, the ANTLR runtime version may be
too new for the shipped grammar. Pin to `4.13.1` or `4.13.2`.

### `plantuml_to_buml()` parses but the model is empty or wrong

The converter only handles a subset of PlantUML class-diagram syntax. If a
`.plantuml` file converts without error but yields missing classes,
associations, or attributes, the input likely used unsupported notation
(n-ary associations, non-class diagrams, stereotypes/OCL) rather than a
BESSER bug. See the besser-user skill's `references/plantuml.md` for the
exact supported subset, then adjust the diagram or add the missing pieces in
Python after conversion.

### `ImportError` for optional packages

These packages are only needed for specific features:

| Package | Needed for | Install |
|---------|-----------|---------|
| `deep_translator` | Agent personalization (translation) | `pip install deep-translator` |
| `docker` | Docker image building in BackendGenerator | `pip install docker` |
| `qiskit` | Running generated quantum circuits | `pip install qiskit` |
| `torch` / `tensorflow` | Running generated NN code | `pip install torch` / `pip install tensorflow` |

---

## Model Construction Errors

### `ValueError: 'My Name' is invalid. Name cannot contain spaces.`

All B-UML names (classes, attributes, associations, etc.) must not contain spaces.
Use underscores: `My_Name`.

### `ValueError: 'my-name' is invalid. Hyphens are not allowed; use '_' instead.`

Replace hyphens with underscores: `my_name`.

### `ValueError: Invalid primitive data type`

Valid primitive type names for `PrimitiveDataType(name)`: `int`, `float`, `str`,
`bool`, `time`, `date`, `datetime`, `timedelta`, `any`.

More commonly, use the pre-built singletons: `StringType`, `IntegerType`, etc.

### `ValueError: A class cannot have more than one attribute marked as 'id'`

Only one `Property` per class can have `is_id=True`. Pick one primary key attribute.

### `ValueError: A binary association must have exactly two ends`

`BinaryAssociation` requires a set of exactly 2 `Property` objects as ends.
Check that you're passing `ends={end1, end2}` with both ends.

### `ValueError: A class cannot be a generalization of itself`

`Generalization(general=A, specific=A)` is not allowed. Check that `general` and
`specific` are different classes.

### `ValueError: The model cannot have types with duplicate names`

Two types in your `DomainModel` share the same name. Each class, enumeration, and
data type must have a unique name within the model.

### `ValueError: Invalid min/max multiplicity`

- `min_multiplicity` must be >= 0.
- `max_multiplicity` must be > 0 and >= min.
- Use `"*"` or `UNLIMITED_MAX_MULTIPLICITY` (9999) for "many".

### `ValueError: An enumeration cannot have two literals with the same name`

Each `EnumerationLiteral` within an `Enumeration` must have a unique name.

### `TypeError: Expected Property instance, but got X`

Association ends must be `Property` instances, not classes or strings. Each end
should be: `Property(name="...", type=SomeClass, multiplicity=Multiplicity(...))`.

---

## Validation Errors

### `model.validate()` reports errors

`DomainModel.validate()` checks structural consistency:

| Error message | Meaning | Fix |
|---------------|---------|-----|
| Generalization references class not in model | A `Generalization` uses a class that wasn't added to `types` | Add the missing class to the model's `types` set |
| Association end references type not in model | An association end's `type` class isn't in `types` | Add the class to `types` |
| Constraint context not in model | An OCL `Constraint` references a class not in `types` | Add the context class or remove the constraint |
| Circular inheritance detected | A → B → A inheritance loop | Fix the generalization chain |

Always call `model.validate()` before running generators to catch these early.

### OCL constraint validation failures

OCL constraints must:
1. Start with the `context` keyword (case-insensitive).
2. Contain at least one OCL keyword: `inv`, `pre`, `post`, `self`, `collect`, `select`, `exists`, `forall`, `size`.
3. Have balanced parentheses.

If the `bocl` library can't parse a constraint, it falls back to a regex-based
parser. Complex OCL that neither parser handles will be reported as invalid but
won't block generation.

**Two distinct OCL failure classes — don't conflate them:**

- *Structural* (blocking): a `Constraint` whose context class isn't in
  `model.types` is a `model.validate()` error — it fails validation and you
  should fix it before generating (see the table above).
- *Expression* (non-blocking): a constraint that is attached correctly but
  whose OCL text neither parser can evaluate is reported as invalid but does
  **not** block generation — generators that don't consume OCL run fine.

---

## Generator Failures (quick reference)

This skill is the fast lookup for a generator error *message*: the table
maps symptom → likely cause → first fix. For *wrong-output* debugging (a
generator that runs but produces nothing or the wrong content) and
per-generator deep dives, defer to the **besser-generators** skill's
`references/debugging.md`.

| Symptom | Likely cause | First fix |
|---------|--------------|-----------|
| `ValueError: Invalid DBMS` | DBMS string is not one of `sqlite`, `postgresql`, `mysql`, `mssql`, `mariadb`, `oracle` | Use exactly `postgresql` (not `postgres`) |
| `ValueError: Invalid backend` (Qiskit) | `backend_type` is not `aer_simulator`, `fake_backend`, or `ibm_quantum` | Pick a valid backend |
| `subprocess.CalledProcessError` (Django) | Django not installed, project name conflict, or invalid Python identifier | `python -m django --version`; delete stale project; rename |
| `AttributeError` inside `WebAppGenerator` | `gui_model=None` was passed | Build a `GUIModel` first (see the besser-user skill's `references/gui-models.md`) |
| `jinja2.TemplateNotFound` | BESSER package corrupted or templates missing | `pip install --force-reinstall besser` |
| Raw `[[` `]]` in generated React files | ReactGenerator template rendering failed silently | Check console for Jinja errors; React uses `[[ ]]` to avoid JSX clashes |

The Django generator catches subprocess errors and prints them rather than
re-raising — if the generator "succeeded" but produced no project, scroll
back through the console. (A generator that produces *no file and no error*
is a debugging case → `references/debugging.md` in besser-generators.)

---

## Docker and Deployment Issues

### `docker-compose up` fails immediately

1. **Docker daemon not running**: Start Docker Desktop or the Docker service.
2. **Port conflict**: Default ports are 3000 (frontend), 8000 (backend), 8765 (agent). Check if something else is using them: `lsof -i :8000` (Unix) or `netstat -an | findstr 8000` (Windows).
3. **Frontend submodule not initialized**: If using the full BESSER repo with Docker Compose, the frontend submodule must be cloned:
   ```bash
   git submodule update --init --recursive
   ```
4. **Missing `.env` file**: The frontend build expects a `.env` file at `besser/utilities/web_modeling_editor/frontend/packages/webapp/webpack/.env`. Create one if missing.

### `docker-compose` vs `docker compose`

BESSER's `docker_deployment.py` calls `docker-compose` (v1 with hyphen). If you
only have Docker Compose v2 (plugin), either:
- Install `docker-compose` separately: `pip install docker-compose`
- Create an alias: `alias docker-compose='docker compose'`

### Backend container can't find modules

The `PYTHONPATH` must be set to `/app` in the container. Check that `docker-compose.yml`
has `PYTHONPATH: /app` in the backend service's environment.

### `ValueError: A different container is already running on port 8000`

The deployment service detected a port conflict. Stop the existing container:

```bash
docker ps  # find the container using port 8000
docker stop <container_id>
```

### Native dependencies fail in Docker

The base image `python:3.11-slim` doesn't include system libraries for `psycopg2`,
`pyodbc`, etc. Add to the Dockerfile:

```dockerfile
RUN apt-get update && apt-get install -y libpq-dev gcc
```

---

## Model/Output Drift

When the generated code doesn't match what you expect from your model:

### New attribute not appearing in generated code

1. Is the attribute actually in the class? `print(my_class.attributes)`.
2. Is the class in the model? `print(model.get_classes())`.
3. Did you regenerate after adding the attribute?

### Association not generating a foreign key

1. Check multiplicities — the generator uses multiplicity to decide which side
   gets the foreign key.
2. Check that both classes are in `model.types` and the association is in
   `model.associations`.
3. For many-to-many, a separate join table is generated.

### Inheritance not reflected in generated code

1. Is the `Generalization` in `model.generalizations`?
2. Check the order: `Generalization(general=Parent, specific=Child)` — `general`
   is the parent, `specific` is the child.

### Enumeration values wrong in generated code

1. Check literal names: `print([lit.name for lit in my_enum.literals])`.
2. Default values: `Property(name="status", type=my_enum, default_value=my_enum.ACTIVE)` — make sure to reference the literal via the enum object.

---

## Capturing Logs for Bug Reports

When filing a bug report, include:

1. **Python version**: `python --version`
2. **BESSER version**: `pip show besser` or check `setup.cfg`
3. **Full traceback**: Copy the complete error output
4. **Minimal model**: Reduce your model to the smallest example that reproduces the issue
5. **Generator used**: Which generator, with what options
6. **Environment**: OS, virtual environment type, Docker version if applicable

### Creating a Minimal Reproduction

```python
# minimal_repro.py
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# Smallest model that triggers the bug
a = Class(name="A", attributes={Property(name="x", type=StringType)})
b = Class(name="B", attributes={Property(name="y", type=IntegerType)})
assoc = BinaryAssociation(name="a_b", ends={
    Property(name="a_end", type=a, multiplicity=Multiplicity(1, 1)),
    Property(name="b_end", type=b, multiplicity=Multiplicity(0, "*")),
})
model = DomainModel(name="repro", types={a, b}, associations={assoc})

# Validate first — validate() returns {"success": bool, "errors": [...], "warnings": [...]}
result = model.validate()
assert result["success"], result["errors"]

# Swap in whichever generator reproduces your bug
gen = SQLAlchemyGenerator(model=model, output_dir="./repro_output")
gen.generate(dbms="sqlite")
```

Run it: `python minimal_repro.py`. If it reproduces the bug, file it as an issue
at https://github.com/BESSER-PEARL/BESSER/issues with the script and the full
error output.

---

## Quick Diagnostic Commands

```bash
# Check BESSER is importable
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"

# Check specific generator
python -c "from besser.generators.sql_alchemy import SQLAlchemyGenerator; print('OK')"

# Check OCL support
python -c "import bocl; print(f'bocl {bocl.__version__}')"

# Check Django
python -m django --version

# Check Docker
docker --version && docker compose version

# Run the library example to verify full stack
python tests/BUML/metamodel/structural/library/library.py

# Run tests
python -m pytest tests/ -x --tb=short
```
