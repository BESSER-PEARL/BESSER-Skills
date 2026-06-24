# API and Web generators

Reference for generators that produce HTTP services or full UIs. Read this
when the user is running `BackendGenerator`, `RESTAPIGenerator`,
`DjangoGenerator`, `WebAppGenerator`, `ReactGenerator`, or
`FlutterGenerator`.

## BackendGenerator

```python
from besser.generators.backend import BackendGenerator
gen = BackendGenerator(model=domain_model, output_dir="./output_backend")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Three files: `main_api.py` (FastAPI), `pydantic_classes.py`, `sql_alchemy.py` |
| Default `output_dir` | `./output_backend/` (note: not the usual `./output/`) |
| Options | `http_methods` (default: GET/POST/PUT/DELETE), `nested_creations`, `docker_image=True` to build/push a Docker image |
| Key behavior | Composite generator — orchestrates `RESTAPIGenerator` + `PydanticGenerator` + `SQLAlchemyGenerator`. Optionally emits `Dockerfile` + `requirements.txt` + `.dockerignore`. |

To limit endpoints:

```python
BackendGenerator(model=m, output_dir="./out", http_methods=["GET", "POST"])
```

Invalid method names are silently filtered (no warning is emitted) —
double-check spelling. (Only `RESTAPIGenerator` logs a warning; the
`BackendGenerator` wrapper filters silently.)

## RESTAPIGenerator

```python
from besser.generators.rest_api import RESTAPIGenerator
gen = RESTAPIGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Standalone mode: `rest_api.py` + `pydantic_classes.py` + `requirements.txt`. Backend mode: `main_api.py` + `requirements.txt` |
| Options | `http_methods`, `nested_creations`, `backend=True` (used internally by `BackendGenerator`), `port` |

Use `BackendGenerator` for the typical "FastAPI + ORM + Pydantic" stack;
use `RESTAPIGenerator` directly only if you want a thinner output.

## DjangoGenerator

```python
from besser.generators.django import DjangoGenerator
gen = DjangoGenerator(
    model=domain_model,
    project_name="myproject",
    app_name="myapp",
    output_dir="./output",
    gui_model=None,           # optional GUIModel for HTML templates
    containerization=False,   # True adds docker-compose.yml + Dockerfile
)
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` (required) + `GUIModel` (optional) |
| Output | Full Django project tree: `models.py`, `urls.py`, `views.py`, `forms.py`, `admin.py`, HTML templates, `requirements.txt` |
| Templates | 12+ Jinja2 templates |
| Key behavior | Runs `django-admin startproject` + `manage.py startapp` as subprocess, then populates files. Modifies `settings.py` in-place. |
| Warning | Prints `"Warning: No main page found."` if the GUIModel has no main page — not an error, generation continues but views may be incomplete. |
| Gotcha | Requires `django` installed. Subprocess failures are caught but not re-raised — check console output. |

## WebAppGenerator

```python
from besser.generators.web_app import WebAppGenerator
gen = WebAppGenerator(
    model=domain_model,
    gui_model=gui_model,    # required
    output_dir="./webapp",
    agent_model=None,       # optional Agent for embedded chatbot
)
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` + `GUIModel` (both required), optional `Agent` |
| Output | `frontend/` (React/Vite), `backend/` (FastAPI), optional `agent/`, `docker-compose.yml`, Dockerfiles |
| Default ports | Frontend 3000, backend 8000, agent WS 8765, agent HTTP 5000 |
| Key behavior | Composite — orchestrates `ReactGenerator` + `BackendGenerator` + optional `BAFGenerator`, then emits Docker orchestration. |
| Run | `cd webapp && docker-compose up --build` |
| Pitfall | Passing `gui_model=None` raises `AttributeError` when the React generator tries to read GUI elements. Always supply a `GUIModel`. |

## ReactGenerator

*Used internally by `WebAppGenerator`; rarely invoked directly.*

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` + `GUIModel` |
| Output | A full React/Vite project (`frontend/`) |
| Key behavior | Uses non-standard Jinja delimiters (`[[`/`]]` instead of `{{`/`}}`) to avoid clashing with JSX braces |
| Gotcha | Raw `[[ ]]` markers in generated files mean template rendering failed silently — check the console for Jinja errors |

## FlutterGenerator

```python
from besser.generators.flutter import FlutterGenerator
# main_page is REQUIRED — a Screen from your GUIModel.
gen = FlutterGenerator(
    model=domain_model, gui_model=gui, main_page=main_screen, output_dir="./mobile",
)
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` + `GUIModel` + `main_page` (a required `Screen` from the GUI model); optional `module` |
| Output | `main.dart`, `sql_helper.dart`, `pubspec.yaml` |
| Note | Registered in the web editor's `SUPPORTED_GENERATORS` (key `flutter`); also usable via the Python API. |
