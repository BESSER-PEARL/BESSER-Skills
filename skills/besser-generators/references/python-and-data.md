# Data-shape generators

Reference for generators that produce data classes, schemas, or
serialization formats — no I/O layer, no UI. Read this when the user is
running `PythonGenerator`, `JavaGenerator`, `PydanticGenerator`,
`JSONSchemaGenerator`, or `RDFGenerator`.

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
gen = JavaGenerator(model=domain_model, output_dir="./output")
gen.generate()
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | One `.java` file per class (e.g., `Book.java`, `Author.java`) |
| Template | `java_template.py.j2` |
| Key behavior | Package name auto-detected from output path. Files sorted by inheritance. |

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
| Key behavior | OCL constraints translate to Pydantic validators. Unicode-safe via `ascii_identifier` filter. |

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
