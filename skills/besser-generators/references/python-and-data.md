# Data-shape generators

Reference for generators that produce data classes, schemas, or
serialization formats — no I/O layer, no UI.

**Covers:** [`PythonGenerator`](#pythongenerator) ·
[`JavaGenerator`](#javagenerator) · [`PydanticGenerator`](#pydanticgenerator) ·
[`JSONSchemaGenerator`](#jsonschemagenerator) ·
[`JSONObjectGenerator`](#jsonobjectgenerator) · [`RDFGenerator`](#rdfgenerator)

## PythonGenerator

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
| Key behavior | All classes generated with bidirectional relationship setters. Classes sorted by inheritance for correct definition order. |

## JavaGenerator

```python
from besser.generators.java_classes import JavaGenerator
gen = JavaGenerator(model=domain_model, output_dir="./output", package_name="com.example.model")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | One `.java` file per class (e.g., `Book.java`, `Author.java`) and one per enumeration (`MemberType.java`) |
| Options | `package_name` — emitted as the `package` line. Omit it and the files carry no package declaration |
| Template | `java_template.py.j2`, `java_enum_template.py.j2` |
| Key behavior | Files sorted by inheritance. An enumeration becomes a Java `enum` and is used as the type of every attribute, parameter and return value referencing it. Both ends of a bidirectional association get a field, and a non-navigable end gets none. Methods declared on the model are emitted as stubs that throw `UnsupportedOperationException`, so the sources compile whatever a method returns. |

`package_name` is a constructor parameter, not something inferred: earlier
versions guessed a package from the output path, which made the same model
generate differently depending on where you ran it. If you need a package line,
pass one.

## PydanticGenerator

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
| Key behavior | OCL constraints translate to Pydantic validators — see `besser-user`'s `references/ocl.md` for which ones. Unicode-safe via `ascii_identifier` filter. |

**Association classes** (`backend=True`). When the model has an
`AssociationClass`, the generator emits a `<AssocClass>LinkCreate` payload
alongside the entity models, and the two related classes take
`Union["<AssocClass>LinkCreate", int]` in place of a plain id — so a request can
create the link and its attributes in one go:

```python
{"rooms": [{"target": 101, "agreed_price": 90.0}]}
```

An attribute the model marks optional stays optional in that payload. If you
are hand-writing a client, send the link object rather than a bare id whenever
the link itself carries data.

## JSONSchemaGenerator

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

## JSONObjectGenerator

Serializes a populated `ObjectModel` (concrete object instances + their links) into a single JSON document. This is the *instance-level* counterpart to `JSONSchemaGenerator`, which emits a *schema* from a `DomainModel`. (See the besser-user skill's `references/object-models.md` for building an `ObjectModel`.)

```python
from besser.generators.json import JSONObjectGenerator
gen = JSONObjectGenerator(model=object_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `ObjectModel` (NOT a `DomainModel`); constructor raises `TypeError` otherwise |
| Output | One JSON file `<model.name>.json` (spaces → `_`; empty name → `object_model.json`); defaults to `./output/` |
| Options | None (`__init__(self, model: ObjectModel, output_dir=None)`) — no `mode` parameter; that belongs to `JSONSchemaGenerator` |
| Key behavior | Emits a concrete data document `{ "name", "objects": [ { "id", "class", "attributes", "relationships" } ] }` (see Details) |
| Gotcha | Malformed `ObjectModel`s degrade silently — unresolvable slots/links are skipped, yielding a partial document, not an error |

### Details

- **Document shape:** top-level `"description"` appears only if `model.metadata.description` is set; empty `attributes`/`relationships` keys are omitted.
- **Value handling:** `datetime`/`date`/`time` → ISO 8601; `timedelta` → total seconds; `set`/`list`/`tuple` → JSON array (sets sorted by `str()`); enum literals → literal name; written with `indent=2, ensure_ascii=False`.
- **vs `JSONSchemaGenerator`:** this generator takes an `ObjectModel` and emits concrete *data*; `JSONSchemaGenerator` takes a `DomainModel` and emits a JSON *Schema* (plus optional FIWARE/NGSI-LD Smart Data Models).

## RDFGenerator

```python
from besser.generators.rdf import RDFGenerator
gen = RDFGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | `vocabulary.ttl` (RDF Turtle) |
| Use case | Publishing domain vocabularies for linked-data applications. |
