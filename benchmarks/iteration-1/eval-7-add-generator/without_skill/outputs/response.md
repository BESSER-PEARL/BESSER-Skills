# Adding a New GraphQL Generator to BESSER: Step-by-Step Guide

This guide walks through every step required to add a new GraphQL schema generator to the BESSER platform, from creating the package structure to registering it in the web editor backend and writing thorough tests.

---

## 1. Understand the Generator Architecture

Every BESSER generator follows the same pattern:

1. **Extend `GeneratorInterface`** (defined in `besser/generators/generator_interface.py`) -- an abstract base class requiring an `__init__(model, output_dir)` constructor and a `generate()` method.
2. **Use Jinja2 templates** stored in a `templates/` subdirectory to render the output from the B-UML metamodel.
3. **Register** the generator in the web editor backend configuration at `besser/utilities/web_modeling_editor/backend/config/generators.py`.
4. **Write tests** in `tests/generators/<name>/`.

The `GeneratorInterface` base class (from `besser/generators/generator_interface.py`) provides:

```python
class GeneratorInterface(ABC):
    @abstractmethod
    def __init__(self, model: Model, output_dir: str = None):
        self.model = model
        self.output_dir = output_dir

    @abstractmethod
    def generate(self):
        pass

    def build_generation_path(self, file_name: str) -> str:
        """Returns the full path for a generated file."""
        ...

    def build_generation_dir(self) -> str:
        """Creates and returns the output directory."""
        ...
```

---

## 2. Create the Package Structure

Create the following directory and file structure:

```
besser/generators/graphql/
    __init__.py
    graphql_generator.py
    templates/
        __init__.py
        graphql_schema_template.graphql.j2
```

Run these commands to create the structure:

```bash
mkdir -p besser/generators/graphql/templates
touch besser/generators/graphql/__init__.py
touch besser/generators/graphql/templates/__init__.py
touch besser/generators/graphql/graphql_generator.py
touch besser/generators/graphql/templates/graphql_schema_template.graphql.j2
```

---

## 3. Implement the Generator Class

Create `besser/generators/graphql/graphql_generator.py`:

```python
import os
from jinja2 import Environment, FileSystemLoader
from besser.BUML.metamodel.structural import DomainModel
from besser.generators import GeneratorInterface
from besser.utilities import sort_by_timestamp


class GraphQLGenerator(GeneratorInterface):
    """
    GraphQLGenerator is a class that implements the GeneratorInterface and is responsible
    for generating a GraphQL schema definition based on the input B-UML model.

    The generator transforms B-UML classes into GraphQL types, enumerations into GraphQL
    enums, and associations into typed fields with appropriate relationships.

    Args:
        model (DomainModel): An instance of the DomainModel class representing the B-UML model.
        output_dir (str, optional): The output directory where the generated schema will be
            saved. Defaults to None.
    """

    # Mapping from B-UML primitive type names to GraphQL scalar types
    GRAPHQL_TYPES = {
        "str": "String",
        "int": "Int",
        "float": "Float",
        "bool": "Boolean",
        "date": "String",       # GraphQL has no built-in Date; use String or a custom scalar
        "datetime": "String",
        "time": "String",
    }

    def __init__(self, model: DomainModel, output_dir: str = None):
        super().__init__(model, output_dir)

    def get_graphql_type(self, buml_type) -> str:
        """
        Convert a B-UML type to its GraphQL equivalent.

        Args:
            buml_type: A B-UML type (PrimitiveDataType, Enumeration, or Class).

        Returns:
            str: The corresponding GraphQL type name.
        """
        from besser.BUML.metamodel.structural import Enumeration, Class
        if isinstance(buml_type, Enumeration):
            return buml_type.name
        if isinstance(buml_type, Class):
            return buml_type.name
        # Primitive type -- use name attribute for lookup
        return self.GRAPHQL_TYPES.get(buml_type.name, "String")

    def generate(self):
        """
        Generates a GraphQL schema file based on the provided B-UML model and saves it
        to the specified output directory.

        If the output directory was not specified, the code generated will be stored in the
        <current directory>/output folder.

        Returns:
            None, but stores the generated code as a file named schema.graphql
        """
        file_path = self.build_generation_path(file_name="schema.graphql")
        templates_path = os.path.join(
            os.path.dirname(os.path.abspath(__file__)), "templates"
        )
        env = Environment(
            loader=FileSystemLoader(templates_path),
            trim_blocks=True,
            lstrip_blocks=True,
        )
        # Register custom filter for type conversion
        env.filters["graphql_type"] = self.get_graphql_type
        env.globals["sort_by_timestamp"] = sort_by_timestamp

        template = env.get_template("graphql_schema_template.graphql.j2")
        with open(file_path, mode="w", encoding="utf-8") as f:
            generated_code = template.render(
                classes=self.model.classes_sorted_by_inheritance(),
                enumerations=self.model.get_enumerations(),
                associations=self.model.associations,
                generalizations=self.model.generalizations,
                sort_by_timestamp=sort_by_timestamp,
            )
            f.write(generated_code)
        print("Code generated in the location: " + file_path)
```

### Key implementation details:

- **Type mapping**: The `GRAPHQL_TYPES` dictionary maps B-UML primitive types (`str`, `int`, `float`, `bool`, `date`, `datetime`, `time`) to GraphQL scalar types (`String`, `Int`, `Float`, `Boolean`). GraphQL lacks built-in `Date`/`DateTime` scalars, so we map those to `String` (a real project might define custom scalars).
- **`get_graphql_type()`**: Handles enumerations, classes (rendered as references to other GraphQL types), and primitives.
- **Template loading**: Uses `FileSystemLoader` pointed at the `templates/` directory relative to the generator file, which is the standard BESSER convention.
- **Custom Jinja2 filter**: The `graphql_type` filter is registered so the template can call `{{ attribute.type | graphql_type }}`.

---

## 4. Create the `__init__.py` Export

Create `besser/generators/graphql/__init__.py`:

```python
from .graphql_generator import *
```

This follows the exact same pattern as every other generator package (e.g., `besser/generators/rest_api/__init__.py`, `besser/generators/python_classes/__init__.py`).

---

## 5. Create the Jinja2 Template

Create `besser/generators/graphql/templates/graphql_schema_template.graphql.j2`:

```jinja2
# GraphQL Schema
# Auto-generated by BESSER GraphQL Generator

{% if enumerations %}
# ─────────────────────────────────────
# Enumerations
# ─────────────────────────────────────

{% for enum in sort_by_timestamp(enumerations) %}
enum {{ enum.name }} {
  {% for literal in sort_by_timestamp(enum.literals) %}
  {{ literal.name }}
  {% endfor %}
}

{% endfor %}
{% endif %}
# ─────────────────────────────────────
# Types
# ─────────────────────────────────────

{% for class in classes %}
{% if class.is_abstract %}
interface {{ class.name }} {
{% else %}
{% set parent_names = [] %}
{% for parent in class.parents() %}
{% if parent_names.append(parent.name) %}{% endif %}
{% endfor %}
{% if parent_names %}
type {{ class.name }} implements {{ parent_names | join(" & ") }} {
{% else %}
type {{ class.name }} {
{% endif %}
{% endif %}
  id: ID!
  {% for attribute in sort_by_timestamp(class.attributes) %}
  {{ attribute.name }}: {{ attribute.type | graphql_type }}{% if attribute.is_id %}!{% endif %}

  {% endfor %}
  {% for end in class.association_ends() if end.is_navigable and end.owner != class %}
  {% if end.multiplicity.max > 1 %}
  {{ end.name }}: [{{ end.type | graphql_type }}!]
  {% else %}
  {{ end.name }}: {{ end.type | graphql_type }}{% if end.multiplicity.min >= 1 %}!{% endif %}

  {% endif %}
  {% endfor %}
}

{% endfor %}
# ─────────────────────────────────────
# Queries
# ─────────────────────────────────────

type Query {
  {% for class in classes %}
  {% if not class.is_abstract %}
  all{{ class.name }}s: [{{ class.name }}!]!
  {{ class.name | lower }}ById(id: ID!): {{ class.name }}
  {% endif %}
  {% endfor %}
}

# ─────────────────────────────────────
# Mutations
# ─────────────────────────────────────

type Mutation {
  {% for class in classes %}
  {% if not class.is_abstract %}
  create{{ class.name }}({% for attribute in sort_by_timestamp(class.attributes) %}{{ attribute.name }}: {{ attribute.type | graphql_type }}{% if not loop.last %}, {% endif %}{% endfor %}): {{ class.name }}!
  update{{ class.name }}(id: ID!, {% for attribute in sort_by_timestamp(class.attributes) %}{{ attribute.name }}: {{ attribute.type | graphql_type }}{% if not loop.last %}, {% endif %}{% endfor %}): {{ class.name }}
  delete{{ class.name }}(id: ID!): Boolean!
  {% endif %}
  {% endfor %}
}
```

### Template design notes:

- **Enumerations** are rendered as GraphQL `enum` types with each literal as a value.
- **Abstract classes** are rendered as GraphQL `interface` types.
- **Concrete classes** are rendered as `type` declarations. If a class has parent classes (via generalization), it uses `implements` to reference the parent interfaces.
- **Attributes** are mapped using the `graphql_type` custom filter.
- **Association ends** that are navigable and belong to a different class become typed fields. A `max > 1` multiplicity renders as a list type `[TypeName!]`, while a `min >= 1` multiplicity adds the `!` (non-null) marker.
- **Query type** is auto-generated with `allXs` (list) and `xById` (single lookup) for each concrete class.
- **Mutation type** provides `create`, `update`, and `delete` operations for each concrete class.

---

## 6. Register the Generator in the Web Editor Backend

Edit `besser/utilities/web_modeling_editor/backend/config/generators.py` to register the new generator.

### 6a. Add the import

Add this line near the top of the file, alongside the other generator imports:

```python
from besser.generators.graphql import GraphQLGenerator
```

### 6b. Add the registry entry

Add this entry to the `SUPPORTED_GENERATORS` dictionary (for example, under the "Data & API" section after `"sql"`):

```python
    "graphql": GeneratorInfo(
        generator_class=GraphQLGenerator,
        output_type="file",
        file_extension=".graphql",
        category="data_format",
        requires_class_diagram=True
    ),
```

### 6c. Add the filename mapping

In the `get_filename_for_generator()` function, add a case for the GraphQL generator:

```python
    elif generator_type == "graphql":
        return "schema.graphql"
```

### Full diff for `generators.py`:

```python
# At the top, add the import:
from besser.generators.graphql import GraphQLGenerator

# In SUPPORTED_GENERATORS dict, add after "sql" entry:
    "graphql": GeneratorInfo(
        generator_class=GraphQLGenerator,
        output_type="file",
        file_extension=".graphql",
        category="data_format",
        requires_class_diagram=True
    ),

# In get_filename_for_generator(), add a branch:
    elif generator_type == "graphql":
        return "schema.graphql"
```

Because the GraphQL generator uses the standard constructor signature (`__init__(model, output_dir)`) and produces a single file, it will be handled automatically by the `_generate_standard()` function in `backend.py`. No changes to `backend.py` are needed.

---

## 7. Write Tests

Create the test file at `tests/generators/graphql/test_graphql_generator.py`:

```python
import os
import pytest
from besser.generators.graphql import GraphQLGenerator
from besser.BUML.metamodel.structural import (
    Class, DomainModel, Enumeration, EnumerationLiteral,
    DateType, StringType, IntegerType, FloatType, BooleanType,
    Property, BinaryAssociation, Multiplicity, Generalization
)


@pytest.fixture
def domain_model():
    """Create a library domain model for testing the GraphQL generator."""
    # Enumeration
    MemberType = Enumeration(
        name="MemberType",
        literals={
            EnumerationLiteral(name="ADULT"),
            EnumerationLiteral(name="STUDENT"),
            EnumerationLiteral(name="SENIOR"),
        }
    )

    # Classes
    Library = Class(name="Library")
    Book = Class(name="Book")
    Author = Class(name="Author")

    # Library attributes
    Library_name = Property(name="name", type=StringType)
    Library_address = Property(name="address", type=StringType)
    Library.attributes = {Library_name, Library_address}

    # Book attributes
    Book_title = Property(name="title", type=StringType)
    Book_pages = Property(name="pages", type=IntegerType)
    Book_release = Property(name="release", type=DateType)
    Book.attributes = {Book_title, Book_pages, Book_release}

    # Author attributes
    Author_name = Property(name="name", type=StringType)
    Author_email = Property(name="email", type=StringType)
    Author_member = Property(name="member", type=MemberType)
    Author.attributes = {Author_name, Author_email, Author_member}

    # Library-Book association (1-to-many)
    lib_book = BinaryAssociation(
        name="LibraryBooks",
        ends={
            Property(name="locatedIn", type=Library, multiplicity=Multiplicity(1, 1)),
            Property(name="has", type=Book, multiplicity=Multiplicity(0, "*")),
        }
    )

    # Book-Author association (many-to-many)
    book_author = BinaryAssociation(
        name="BookAuthors",
        ends={
            Property(name="writtenBy", type=Author, multiplicity=Multiplicity(1, "*")),
            Property(name="publishes", type=Book, multiplicity=Multiplicity(0, "*")),
        }
    )

    model = DomainModel(
        name="Library_Model",
        types={Library, Book, Author, MemberType},
        associations={lib_book, book_author},
    )
    return model


@pytest.fixture
def generated_schema(domain_model, tmpdir):
    """Generate a GraphQL schema and return its contents."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    with open(output_file, "r", encoding="utf-8") as f:
        return f.read()


def test_output_file_created(domain_model, tmpdir):
    """Test that the generator creates the output file."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    assert os.path.isfile(output_file)
    assert os.path.getsize(output_file) > 0


def test_types_exist(generated_schema):
    """Test that all B-UML classes are rendered as GraphQL types."""
    assert "type Library" in generated_schema
    assert "type Book" in generated_schema
    assert "type Author" in generated_schema


def test_enum_exists(generated_schema):
    """Test that B-UML enumerations are rendered as GraphQL enums."""
    assert "enum MemberType" in generated_schema
    assert "ADULT" in generated_schema
    assert "STUDENT" in generated_schema
    assert "SENIOR" in generated_schema


def test_attributes_exist(generated_schema):
    """Test that class attributes appear in the generated schema."""
    # Library attributes
    assert "name: String" in generated_schema
    assert "address: String" in generated_schema
    # Book attributes
    assert "title: String" in generated_schema
    assert "pages: Int" in generated_schema
    # Author attributes
    assert "email: String" in generated_schema
    assert "member: MemberType" in generated_schema


def test_id_field_exists(generated_schema):
    """Test that an id field is generated for each type."""
    assert "id: ID!" in generated_schema


def test_query_type_exists(generated_schema):
    """Test that Query type is generated with list and by-id queries."""
    assert "type Query" in generated_schema
    assert "allLibrarys:" in generated_schema or "allLibraries:" in generated_schema
    assert "allBooks:" in generated_schema or "allBook" in generated_schema
    assert "allAuthors:" in generated_schema or "allAuthor" in generated_schema


def test_mutation_type_exists(generated_schema):
    """Test that Mutation type is generated with create/update/delete operations."""
    assert "type Mutation" in generated_schema
    assert "createLibrary" in generated_schema
    assert "createBook" in generated_schema
    assert "createAuthor" in generated_schema
    assert "deleteLibrary" in generated_schema
    assert "deleteBook" in generated_schema
    assert "deleteAuthor" in generated_schema


def test_query_by_id(generated_schema):
    """Test that by-id queries accept an ID argument."""
    assert "id: ID!" in generated_schema
    assert "ById(id: ID!)" in generated_schema


def test_output_dir_default(domain_model, tmpdir, monkeypatch):
    """Test that the generator works when no output_dir is specified."""
    monkeypatch.chdir(tmpdir)
    generator = GraphQLGenerator(model=domain_model)
    generator.generate()
    output_file = os.path.join(str(tmpdir), "output", "schema.graphql")
    assert os.path.isfile(output_file)


@pytest.fixture
def domain_model_with_inheritance():
    """Create a model with inheritance (generalization) for testing."""
    # Base class (abstract)
    Vehicle = Class(name="Vehicle", is_abstract=True)
    Vehicle_make = Property(name="make", type=StringType)
    Vehicle_year = Property(name="year", type=IntegerType)
    Vehicle.attributes = {Vehicle_make, Vehicle_year}

    # Subclasses
    Car = Class(name="Car")
    Car_doors = Property(name="doors", type=IntegerType)
    Car.attributes = {Car_doors}

    Truck = Class(name="Truck")
    Truck_payload = Property(name="payload", type=FloatType)
    Truck.attributes = {Truck_payload}

    # Generalizations
    gen_car = Generalization(general=Vehicle, specific=Car)
    gen_truck = Generalization(general=Vehicle, specific=Truck)

    model = DomainModel(
        name="Vehicle_Model",
        types={Vehicle, Car, Truck},
        associations=set(),
        generalizations={gen_car, gen_truck},
    )
    return model


def test_abstract_class_as_interface(domain_model_with_inheritance, tmpdir):
    """Test that abstract classes are rendered as GraphQL interfaces."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(
        model=domain_model_with_inheritance, output_dir=str(output_dir)
    )
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    with open(output_file, "r", encoding="utf-8") as f:
        schema = f.read()

    assert "interface Vehicle" in schema
    assert "type Car" in schema
    assert "type Truck" in schema


def test_implements_interface(domain_model_with_inheritance, tmpdir):
    """Test that subclasses use 'implements' for their parent interface."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(
        model=domain_model_with_inheritance, output_dir=str(output_dir)
    )
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    with open(output_file, "r", encoding="utf-8") as f:
        schema = f.read()

    assert "type Car implements Vehicle" in schema
    assert "type Truck implements Vehicle" in schema


def test_abstract_class_excluded_from_mutations(domain_model_with_inheritance, tmpdir):
    """Test that abstract classes do not get create/update/delete mutations."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(
        model=domain_model_with_inheritance, output_dir=str(output_dir)
    )
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    with open(output_file, "r", encoding="utf-8") as f:
        schema = f.read()

    assert "createVehicle" not in schema
    assert "deleteVehicle" not in schema
    assert "createCar" in schema
    assert "createTruck" in schema


def test_generator_interface_compliance(domain_model):
    """Test that GraphQLGenerator properly implements GeneratorInterface."""
    from besser.generators import GeneratorInterface
    generator = GraphQLGenerator(model=domain_model)
    assert isinstance(generator, GeneratorInterface)
    assert hasattr(generator, "generate")
    assert hasattr(generator, "model")
    assert hasattr(generator, "output_dir")
    assert hasattr(generator, "build_generation_path")
    assert hasattr(generator, "build_generation_dir")
```

### What these tests cover:

| Test | Purpose |
|------|---------|
| `test_output_file_created` | Verifies the file is created and non-empty |
| `test_types_exist` | Checks all B-UML classes become GraphQL types |
| `test_enum_exists` | Checks enumerations render as GraphQL enums with all literals |
| `test_attributes_exist` | Verifies class attributes appear with correct GraphQL types |
| `test_id_field_exists` | Ensures each type gets an `id: ID!` field |
| `test_query_type_exists` | Validates the Query type is generated |
| `test_mutation_type_exists` | Validates CRUD mutations are generated |
| `test_query_by_id` | Checks that lookup-by-id queries exist |
| `test_output_dir_default` | Tests the default output directory behavior |
| `test_abstract_class_as_interface` | Abstract B-UML classes become GraphQL interfaces |
| `test_implements_interface` | Subclasses use `implements` keyword |
| `test_abstract_class_excluded_from_mutations` | Abstract classes are excluded from mutations |
| `test_generator_interface_compliance` | Confirms the class properly extends `GeneratorInterface` |

---

## 8. Run the Tests

```bash
# Run only the GraphQL generator tests
python -m pytest tests/generators/graphql/test_graphql_generator.py -v

# Run all generator tests to make sure nothing is broken
python -m pytest tests/generators/ -v
```

---

## 9. Update Documentation (Optional but Recommended)

### 9a. Create a documentation page

Create `docs/source/generators/graphql.rst`:

```rst
GraphQL Generator
=================

The GraphQL generator transforms a B-UML structural model into a GraphQL schema definition
(``schema.graphql``).

Features
--------

- Converts B-UML classes to GraphQL ``type`` definitions
- Converts abstract classes to GraphQL ``interface`` definitions
- Maps B-UML enumerations to GraphQL ``enum`` types
- Generates ``Query`` type with list and by-ID lookups
- Generates ``Mutation`` type with create, update, and delete operations
- Maps associations to typed relationship fields (single or list)

Usage
-----

.. code-block:: python

    from besser.BUML.metamodel.structural import (
        DomainModel, Class, Property, StringType, IntegerType,
        BinaryAssociation, Multiplicity
    )
    from besser.generators.graphql import GraphQLGenerator

    # Define your B-UML domain model
    Book = Class(name="Book")
    Book.attributes = {
        Property(name="title", type=StringType),
        Property(name="pages", type=IntegerType),
    }

    Author = Class(name="Author")
    Author.attributes = {
        Property(name="name", type=StringType),
    }

    writes = BinaryAssociation(
        name="Writes",
        ends={
            Property(name="author", type=Author, multiplicity=Multiplicity(1, 1)),
            Property(name="books", type=Book, multiplicity=Multiplicity(0, "*")),
        }
    )

    model = DomainModel(
        name="Bookstore",
        types={Book, Author},
        associations={writes},
    )

    # Generate the GraphQL schema
    generator = GraphQLGenerator(model=model, output_dir="./output")
    generator.generate()

Output
------

The generator produces a file named ``schema.graphql`` in the specified output directory
(or ``./output/`` by default).
```

### 9b. Add to the table of contents

Edit `docs/source/generators.rst` and add the new generator under the "Data & API" section:

```rst
Data & API
----------

Generate database schemas, APIs, and data formats:

.. toctree::
   :maxdepth: 1

   generators/sql
   generators/json_schema
   generators/rdf
   generators/terraform
   generators/graphql
```

---

## 10. Summary of All Files

Here is the complete list of files to create or modify:

### New files to create:

| File | Purpose |
|------|---------|
| `besser/generators/graphql/__init__.py` | Package init, exports `GraphQLGenerator` |
| `besser/generators/graphql/graphql_generator.py` | Generator implementation |
| `besser/generators/graphql/templates/__init__.py` | Empty init for templates package |
| `besser/generators/graphql/templates/graphql_schema_template.graphql.j2` | Jinja2 template |
| `tests/generators/graphql/test_graphql_generator.py` | Test suite |
| `docs/source/generators/graphql.rst` | Documentation page (optional) |

### Existing files to modify:

| File | Change |
|------|--------|
| `besser/utilities/web_modeling_editor/backend/config/generators.py` | Add import, registry entry, and filename mapping |
| `docs/source/generators.rst` | Add `generators/graphql` to the toctree (optional) |

### Files that require NO changes:

| File | Reason |
|------|--------|
| `besser/utilities/web_modeling_editor/backend/backend.py` | The `_generate_standard()` function handles any generator with a standard `__init__(model, output_dir)` signature automatically |
| `besser/generators/__init__.py` | Already exports `GeneratorInterface`; no changes needed |

---

## 11. How It All Connects at Runtime

When a user selects "GraphQL" in the BESSER web editor and clicks generate, the following flow executes:

1. The frontend sends a `POST /besser_api/generate-output` request with `generator: "graphql"` and the class diagram JSON.
2. `backend.py` looks up `"graphql"` in `SUPPORTED_GENERATORS` via `get_generator_info("graphql")`.
3. Since `requires_class_diagram=True` and `category="data_format"`, the request is routed to `_handle_class_diagram_generation()`.
4. The JSON data is converted to a B-UML `DomainModel` via `process_class_diagram(json_data)`.
5. Since `"graphql"` does not match any special-cased generator type (django, sql, sqlalchemy, jsonschema), it falls through to `_generate_standard()`.
6. `_generate_standard()` instantiates `GraphQLGenerator(buml_model, output_dir=temp_dir)` and calls `generate()`.
7. The generator loads its Jinja2 template, renders the schema, and writes `schema.graphql` to the temp directory.
8. Since `output_type="file"`, `_create_file_response()` returns the file content directly (not zipped).
9. The filename is determined by `get_filename_for_generator("graphql")`, which returns `"schema.graphql"`.

---

## 12. Additional Enhancements to Consider

Once the basic generator is working, you might consider these improvements:

- **Custom scalar types**: Add GraphQL `scalar Date` and `scalar DateTime` declarations instead of mapping to `String`.
- **Input types**: Generate `input CreateBookInput { ... }` types for cleaner mutation signatures.
- **Pagination**: Add `first`, `after`, `last`, `before` arguments to list queries for Relay-style cursor pagination.
- **Filtering**: Generate filter input types like `BookFilter` with field-level predicates.
- **Subscriptions**: Add a `Subscription` type for real-time updates.
- **Association class support**: Handle `AssociationClass` from the B-UML metamodel (similar to how `SQLAlchemyGenerator` does with `separate_classes()`).
- **OCL constraint mapping**: Translate OCL constraints into GraphQL directive annotations (e.g., `@constraint(min: 0, max: 100)`).
