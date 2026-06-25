# Debugging generation failures

Read this when a generator runs but the output is empty, wrong, or fails
silently. For installation and import errors, the **besser-troubleshooting**
skill is the right place; use this reference when generation itself
misbehaves.

## Generator produces no output (no file, no exception)

1. Validate the model first: `model.validate()`. Fix any errors before
   generating — many generators happily produce nothing for an invalid
   model.
2. Confirm `output_dir` exists or is writable. Generators auto-create it
   with `os.makedirs(..., exist_ok=True)`, but unusual permission setups
   can still fail.
3. For `SQLGenerator`: output is produced by an internal subprocess. If
   that subprocess fails, the error is *printed to stdout, not raised*.
   Check the console — you may see a Python traceback there with no
   exception bubbled up.

## Generated `pydantic_classes.py` / `sql_alchemy.py` won't parse

- **Symptom**: `main_api.py` is fine but the Pydantic/SQLAlchemy files raise a
  `SyntaxError` — often `leading zeros in decimal integer literals are not
  permitted` — and won't import.
- **Cause**: an **enum-typed attribute with a `default_value`**, e.g.
  `Property(name="role", type=role, default_value=role.MEMBER)`. It's valid
  B-UML (`validate()` passes), but these generators can't serialize an
  enum-literal default — they dump the literal's raw `repr()` (the whole
  enumeration object graph, including a timestamp like `13:37:05.00…`, which
  parses as a leading-zero number).
- **Fix**: remove the `default_value` from enum attributes (keep the enums
  themselves). Set the initial value in the app/auth layer or as a DB column
  default instead. Primitive defaults (`is_active: bool = True`, `0`, …)
  round-trip fine; only enum-literal defaults break.

## Generator produces wrong content

- **Class names**: names with spaces or hyphens raise `ValueError` at
  *model construction* time, not generation time. If you see odd output,
  print `model.get_classes()` and check.
- **Associations**: each `BinaryAssociation` needs exactly two ends. Each
  end's `type` must point to a class included in `model.types`.
- **Inheritance**: circular inheritance (`A → B → A`) is caught by
  `model.validate()`.

## SQLAlchemy / SQL specifics

- **`ValueError: Invalid DBMS`**: only `sqlite`, `postgresql`, `mysql`,
  `mssql`, `mariadb`, `oracle` are valid. Note: **`postgresql`, not
  `postgres`** — even the BESSER docs occasionally got this wrong.
- **Missing primary keys**: without an `is_id=True` attribute, SQLAlchemy
  auto-generates an `id` column. Mark one attribute as `is_id=True` if
  you need control.
- **Association tables**: many-to-many produces a join table whose name
  is the association name, lowercased (not the class names).

## Django specifics

- **Django not installed**: the generator runs `django-admin` as a
  subprocess. Without `django` in the environment it fails.
- **`"Warning: No main page found"`**: the GUIModel has no screen with
  `is_main_page=True`. Generation continues but views may be incomplete.
- **Project name conflicts**: if a project with the same name already
  exists in the output directory, `django-admin startproject` fails.
  Delete it or pick a different name.
- **Subprocess errors are caught but not re-raised** — read console output.

## WebApp specifics

- **`gui_model=None`** raises `AttributeError` when `ReactGenerator` tries
  to access GUI elements. Always pass a `GUIModel`.
- **Port conflicts**: defaults are 3000 / 8000 / 8765. Edit the generated
  `docker-compose.yml` if those are taken.
- **Docker daemon not running**: `docker-compose up --build` requires
  Docker to be active.

## Backend specifics

- **Invalid HTTP method names** are silently filtered (no warning is
  emitted by `BackendGenerator`). Spell-check method strings.
- **Docker image push fails**: `docker_image=True` requires the `docker`
  Python package and Docker Hub credentials in a config INI file.

## Template rendering errors

`jinja2.TemplateNotFound` or `jinja2.UndefinedError` usually mean:

- the generator's template directory is missing or corrupted.
- BESSER was installed without templates (rare).
- Fix: `pip install --force-reinstall besser`, or `pip install -e .` from
  the repo root, then re-run.

## React generator: visible `[[ ]]` markers

The React generator uses non-standard Jinja delimiters (`[[` / `]]`) to
avoid conflicts with JSX. Raw `[[ ]]` in generated files means the
template render failed silently — check console for Jinja errors.

## Composite generator failures

Several generators are composites — a failure inside a sub-generator can
look like a failure of the wrapper:

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

When debugging composite failures, trace which sub-generator the error
originates in — the error rarely lives in the orchestrator.
