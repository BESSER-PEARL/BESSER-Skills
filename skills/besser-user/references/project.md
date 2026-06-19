# Project Metamodel Reference

`Project` is a thin top-level container that aggregates one or more B-UML
models (a `DomainModel`, `ObjectModel`, `GUIModel`, `StateMachine`, etc.)
under a single named, owned, metadata-carrying wrapper. It is the unit the
web modeling editor and the project code builder serialize to/from a
`Project(...)` block in a generated `model.py`/`project.py` file.

It is a thin container: it holds a list of models plus a few scalar fields and
adds no validation or generation logic of its own. Read this when the user
wants to bundle multiple models together or round-trip a project file.

## Imports

```python
from besser.BUML.metamodel.project import Project
from besser.BUML.metamodel.structural import Metadata  # optional, for project metadata
```

`Project` lives in its own subpackage (`besser.BUML.metamodel.project`), not in
`structural`. `Metadata` comes from `structural` (also importable as
`from besser.BUML.metamodel.structural.structural import Metadata`).

## Key classes

| Class | Description |
|-------|-------------|
| `Project` | Named container aggregating a `list[Model]`, an `owner` string, and optional `Metadata`. Subclass of `NamedElement`. |
| `Metadata` | Optional descriptive info attached to a project (or any named element): `description`, `uri`, `synonyms`, `icon`, `timestamp`. |

### `Project.__init__` signature

```python
Project(
    name: str,                      # required
    models: list[Model] = None,     # defaults to [] when None
    owner: str = "",                # defaults to empty string
    metadata: Metadata = None,      # defaults to None
)
```

Note the parameter order: `name, models, owner, metadata`. (The class
docstring is out of date — it lists `name, models, metadata` and omits
`owner`. Prefer keyword arguments to avoid positional mistakes.)

Attributes:

- `project.name` — inherited from `NamedElement` (plain attribute).
- `project.models` — `list[Model]`; the setter raises `TypeError` if you assign
  anything that is not a `list`.
- `project.owner` — `str`.
- `project.metadata` — `Metadata` or `None` (plain attribute, no setter
  validation).

There are **no** `add_model()`, `validate()`, or other helper methods on
`Project`; mutate `project.models` directly (e.g. `project.models.append(dm)`).

### `Metadata.__init__` signature

```python
Metadata(
    description: str = None,
    uri: str = None,
    synonyms: list[str] = None,
    icon: str = None,
    timestamp: datetime = None,     # defaults to creation time inside Element
)
```

## Minimal build example

```python
from besser.BUML.metamodel.project import Project
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, StringType, Metadata,
)

# A model to put in the project
person = Class(name="Person", attributes={Property(name="name", type=StringType)})
dm = DomainModel(name="PersonModel", types={person}, associations=set())

# Wrap it in a project
project = Project(
    name="TestProject",
    models=[dm],
    owner="tester",
    metadata=Metadata(description="Test project"),
)

# A project can hold several models of any kind
# project.models.append(other_model)
```

## Generation

`Project` objects round-trip to Python source via `project_to_code` in the
code builder:

```python
from besser.utilities.buml_code_builder.project_builder import project_to_code

project_to_code(project, "project.py")   # writes a runnable Project(...) file
```

Signature: `project_to_code(project: Project, file_path: str, sm: str = "")`.
The `sm` parameter is deprecated (legacy state-machine string); `StateMachine`
objects placed in `project.models` are emitted automatically. The generated
file emits one block per model plus a trailing
`project = Project(name=..., models=[...], owner=..., metadata=metadata)`.

## Gotchas

- `models` must be a **list** (`TypeError` otherwise), unlike `DomainModel`
  which takes a `set` of `types`. Order is preserved.
- Validation is per-model: call `model.validate()` on each `DomainModel`
  individually. `Project` itself has no `validate()`.
- The constructor's positional order is `name, models, owner, metadata` —
  the docstring is wrong about this. Use keyword arguments.
- `metadata` defaults to `None`, not an empty `Metadata()`. Generators that
  read `project.metadata.description` assume you supplied a `Metadata`; pass one
  if you intend to serialize metadata.
