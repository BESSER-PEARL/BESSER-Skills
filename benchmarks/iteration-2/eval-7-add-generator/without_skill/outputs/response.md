I have everything I need. Here is the complete answer.

---

# Adding a new GraphQL generator to BESSER

BESSER's generators all follow one pattern: a class that inherits `GeneratorInterface`, lives in its own folder under `besser/generators/`, renders output (usually via Jinja2) and writes a file. There is no plugin registry to satisfy for the Python API — exporting the class from the package `__init__.py` is enough to use it from code. The *only* extra "registration" is optional: a metadata entry so the generator shows up in the Web Modeling Editor. Below is the full process, grounded in how the existing generators are built.

## 1. The contract you must implement

Every generator subclasses `GeneratorInterface` (`besser/generators/generator_interface.py`). It is an `ABC` requiring `__init__(self, model, output_dir=None)` and `generate(self)`. It gives you two free helpers you should use for output paths:

- `self.build_generation_path(file_name)` -> full path to a file inside the output dir
- `self.build_generation_dir()` -> the output dir (creates `output_dir`, or `./output` if none given)

`self.model` is a `DomainModel` from `besser.BUML.metamodel.structural`. The useful traversal methods (confirmed in `besser/BUML/metamodel/structural/structural.py`) are:

- `model.get_classes()` -> `set[Class]`
- `model.classes_sorted_by_inheritance()` -> `list[Class]` (parents before children — important when you emit type extensions/interfaces)
- `model.get_enumerations()` -> `set[Enumeration]`
- per class: `class.attributes`, `class.all_attributes()` (includes inherited), `class.association_ends()` (each end has `.name`, `.type`, `.multiplicity` with `.min`/`.max`, `.is_navigable`, `.opposite_end()`), `class.parents()`, `class.is_abstract`
- an attribute's `attribute.type.name` gives the B-UML primitive name (`str`, `int`, `float`, `bool`, `date`, …) which you map to GraphQL scalars (`String`, `Int`, `Float`, `Boolean`, etc.).

A GraphQL SDL generator is structurally closest to the **SQL / JSON Schema / RDF** generators: single text-file output produced from the class diagram. Model it on those, not on the `zip`-producing web-framework ones.

## 2. Where the files go

Mirror the `python_classes` / `json` layout:

```
besser/generators/graphql/
├── __init__.py                       # re-exports GraphQLGenerator
├── graphql_generator.py              # the generator class
└── templates/
    ├── __init__.py                   # empty file
    └── graphql_schema.graphql.j2     # Jinja2 template
```

The `setup.cfg` already ships `*.j2` files (`[options.package_data] * = *.j2`), so your template is packaged automatically — no setup.cfg change needed. Add the empty `templates/__init__.py` to match the other generators.

## 3. The generator class

`besser/generators/graphql/graphql_generator.py` — this follows `PythonGenerator` almost verbatim (the canonical Jinja-based example), adapted to emit a single `.graphql` file:

```python
import os
from jinja2 import Environment, FileSystemLoader
from besser.BUML.metamodel.structural import DomainModel
from besser.generators import GeneratorInterface


# B-UML primitive type name -> GraphQL scalar
GRAPHQL_TYPES = {
    "str": "String",
    "string": "String",
    "int": "Int",
    "integer": "Int",
    "float": "Float",
    "bool": "Boolean",
    "boolean": "Boolean",
    "date": "String",
    "datetime": "String",
    "time": "String",
}


class GraphQLGenerator(GeneratorInterface):
    """Generates a GraphQL schema (SDL) from a B-UML domain model.

    Args:
        model (DomainModel): the B-UML model.
        output_dir (str, optional): where the schema is written.
            Defaults to ``<cwd>/output``.
    """

    def __init__(self, model: DomainModel, output_dir: str = None):
        super().__init__(model, output_dir)

    def generate(self):
        """Render the GraphQL schema and write it to ``schema.graphql``."""
        file_path = self.build_generation_path(file_name="schema.graphql")
        templates_path = os.path.join(
            os.path.dirname(os.path.abspath(__file__)), "templates"
        )
        env = Environment(
            loader=FileSystemLoader(templates_path),
            trim_blocks=True,
            lstrip_blocks=True,
            keep_trailing_newline=True,
        )
        env.filters["gql_type"] = self._gql_type
        template = env.get_template("graphql_schema.graphql.j2")
        with open(file_path, mode="w", newline="\n", encoding="utf-8") as f:
            f.write(
                template.render(
                    domain=self.model,
                    classes=self.model.classes_sorted_by_inheritance(),
                    enumerations=self.model.get_enumerations(),
                )
            )
            print("Code generated in the location: " + file_path)

    @staticmethod
    def _gql_type(end_or_attr):
        """Map a property's type + multiplicity to a GraphQL field type."""
        type_name = end_or_attr.type.name
        gql = GRAPHQL_TYPES.get(type_name.lower(), type_name)  # fall back to class/enum name
        mult = getattr(end_or_attr, "multiplicity", None)
        if mult is not None and mult.max != 1:        # to-many -> list
            gql = f"[{gql}!]"
        if mult is not None and mult.min and mult.min >= 1:  # required -> non-null
            gql = f"{gql}!"
        return gql
```

Notes on choices: I used a Jinja2 environment exactly like `PydanticGenerator` (`trim_blocks`/`lstrip_blocks`/`keep_trailing_newline`) and a custom filter for the type mapping. Keeping the scalar map and multiplicity logic in Python (a small filter) rather than in the template keeps the template readable and the logic unit-testable — the doc `create_generator.rst` explicitly recommends "small, composable functions to simplify testing."

`besser/generators/graphql/__init__.py`:

```python
from .graphql_generator import GraphQLGenerator as GraphQLGenerator
```

`besser/generators/graphql/templates/graphql_schema.graphql.j2`:

```jinja
{% for enum in enumerations %}
enum {{ enum.name }} {
{% for literal in enum.literals %}
  {{ literal.name }}
{% endfor %}
}

{% endfor %}
{% for class in classes %}
type {{ class.name }} {
{% for attribute in class.all_attributes() %}
  {{ attribute.name }}: {{ attribute | gql_type }}
{% endfor %}
{% for end in class.association_ends() if end.is_navigable %}
  {{ end.name }}: {{ end | gql_type }}
{% endfor %}
}

{% endfor %}
```

(If you want queries/mutations too, extend the template with a `type Query { ... }` block — but the structural type emission above is the core deliverable.)

## 4. Registration

There are two distinct things people mean by "register," and BESSER treats them separately:

**(a) Make it importable (required).** Already done by the `__init__.py` re-export. After that, anyone can do:

```python
from besser.generators.graphql import GraphQLGenerator
GraphQLGenerator(model=my_model, output_dir="out").generate()
```

There is no central Python registry of generators — the package export is the whole story for the programmatic API.

**(b) Expose it in the Web Modeling Editor (optional).** This is the only place a "registry" exists. Edit `besser/utilities/web_modeling_editor/backend/config/generators.py`:

1. Import it at the top with the others:
   ```python
   from besser.generators.graphql import GraphQLGenerator
   ```
2. Add an entry to the `SUPPORTED_GENERATORS` dict using the `GeneratorInfo` NamedTuple:
   ```python
   "graphql": GeneratorInfo(
       generator_class=GraphQLGenerator,
       output_type="file",          # single file, not a zip
       file_extension=".graphql",
       category="data_format",      # same bucket as jsonschema / rdf
       requires_class_diagram=True, # it consumes the class diagram
   ),
   ```
3. Add a filename mapping in `get_filename_for_generator()` so downloads get a sensible name:
   ```python
   elif generator_type == "graphql":
       return "schema.graphql"
   ```

`requires_class_diagram=True` is correct here because GraphQL types come from the class diagram (set it `False`, with `required_diagram_type=...`, only for generators driven by other diagram types like NN/quantum/agent). This matches what `CONTRIBUTING.md` (lines 244-248) and `docs/source/generators/build_generator.rst` describe.

## 5. Tests

Tests live under `tests/generators/<name>/`, i.e. create `tests/generators/graphql/test_graphql_generator.py`. The suite uses `pytest` with `--import-mode=importlib` (set in `pyproject.toml`), pytest's `tmpdir`, and shared model fixtures from `tests/conftest.py` (notably `library_book_author_model`, plus `employee_self_assoc_model` for self-associations). Reuse those fixtures rather than building models inline — that is the established convention (`test_json_schema.py`, `test_python_generator.py` both alias a shared fixture).

`tests/generators/graphql/test_graphql_generator.py`:

```python
import os
import pytest
from besser.generators.graphql import GraphQLGenerator


@pytest.fixture
def domain_model(library_book_author_model):
    """Reuse the canonical Library/Book/Author fixture from tests/conftest.py."""
    return library_book_author_model


def test_graphql_generator_writes_file(domain_model, tmpdir):
    output_dir = tmpdir.mkdir("output")
    GraphQLGenerator(model=domain_model, output_dir=str(output_dir)).generate()

    output_file = os.path.join(str(output_dir), "schema.graphql")
    assert os.path.isfile(output_file)


def test_graphql_types_and_fields(domain_model, tmpdir):
    output_dir = tmpdir.mkdir("output")
    GraphQLGenerator(model=domain_model, output_dir=str(output_dir)).generate()
    with open(os.path.join(str(output_dir), "schema.graphql"), encoding="utf-8") as f:
        schema = f.read()

    # one GraphQL `type` per class
    assert "type Library {" in schema
    assert "type Book {" in schema
    assert "type Author {" in schema

    # scalar mapping (str -> String, int -> Int)
    assert "title: String" in schema
    assert "pages: Int" in schema

    # to-many association end becomes a list
    assert "has: [Book!]" in schema


def test_default_output_dir(domain_model, monkeypatch, tmpdir):
    # When no output_dir is given, GeneratorInterface writes to <cwd>/output
    monkeypatch.chdir(tmpdir)
    GraphQLGenerator(model=domain_model).generate()
    assert os.path.isfile(os.path.join(str(tmpdir), "output", "schema.graphql"))
```

Test the things that actually matter for a generator:
- **File is produced** at the expected path (and the default `./output` fallback works).
- **Structure**: a `type` per class, an `enum` per enumeration, fields per attribute and per navigable association end.
- **Mapping logic**: scalar type translation and multiplicity handling (lists for to-many, `!` for required). It's worth adding a focused unit test on the `_gql_type` filter and an enum-bearing fixture too (`library_model_with_enum`, used by the Python generator test).
- For self-references / opposite ends, add a case using `employee_self_assoc_model`.

If you want to go further (as the JSON/python tests do), parse or compile the output to prove validity — e.g. feed the SDL into `graphql-core`'s `build_schema()` inside the test and assert it parses. That's the GraphQL equivalent of the Python generator test that actually `exec`s the generated module.

Run just your tests while iterating:
```bash
python -m pytest tests/generators -k graphql
```

## 6. Documentation (expected for a PR)

The contributing guide treats docs as part of "adding a generator." To match the repo conventions:

- **User guide**: add `docs/source/generators/graphql.rst` and list it in the generators `toctree` in `docs/source/generators.rst`.
- **API reference**: add `docs/source/api/generators/api_graphql.rst` with an `.. automodule:: besser.generators.graphql.graphql_generator` block (copy `api_python.rst`) and add it to the `toctree` in `docs/source/api/api_generators.rst`.

## Checklist

1. Create `besser/generators/graphql/{__init__.py, graphql_generator.py, templates/...}`; class inherits `GeneratorInterface`, implements `__init__(model, output_dir=None)` and `generate()`, writes via `build_generation_path()`.
2. Re-export `GraphQLGenerator` from the package `__init__.py` (this is all that's needed for programmatic use).
3. (Optional, for the web editor) add a `GeneratorInfo` entry + filename mapping in `besser/utilities/web_modeling_editor/backend/config/generators.py`.
4. Add `tests/generators/graphql/test_graphql_generator.py` using shared `conftest.py` fixtures and `tmpdir`; assert file creation, type/field emission, and scalar/multiplicity mapping. Run `python -m pytest tests/generators -k graphql`.
5. Add user + API docs and wire them into the toctrees.
6. `ruff check .` before opening the PR (CI lints).

### Key reference files

- `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/generators/generator_interface.py` — the base class / contract
- `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/generators/python_classes/python_classes_generator.py` — closest scaffold to copy
- `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/utilities/web_modeling_editor/backend/config/generators.py` — the `SUPPORTED_GENERATORS` registry + `GeneratorInfo`
- `C:/Users/sulejmani/Desktop/BESSER-upstream/tests/conftest.py` — shared model fixtures
- `C:/Users/sulejmani/Desktop/BESSER-upstream/tests/generators/json_schema/test_json_schema.py` and `.../python/test_python_generator.py` — test patterns to mirror
- `C:/Users/sulejmani/Desktop/BESSER-upstream/docs/source/contributing/create_generator.rst` and `docs/source/generators/build_generator.rst` — the official playbook this answer follows