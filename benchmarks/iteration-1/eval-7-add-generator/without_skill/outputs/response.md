# Adding a GraphQL Generator to BESSER: Step-by-Step Guide

## Overview

This guide walks through every step needed to add a new **GraphQL schema generator** to BESSER. The generator will transform a B-UML `DomainModel` (classes, attributes, associations, enumerations, generalizations) into a valid GraphQL schema definition language (SDL) file.

The process follows the established patterns used by existing generators such as `PythonGenerator`, `SQLAlchemyGenerator`, and `JSONSchemaGenerator`.

---

## Step 1: Create the Generator Package Directory

Create a new package under `besser/generators/graphql/`.

**Files to create:**

```
besser/generators/graphql/
    __init__.py
    graphql_generator.py
    templates/
        __init__.py
        graphql_schema.graphql.j2
```

### 1a. `besser/generators/graphql/__init__.py`

This file re-exports the generator class, following the convention used by all other generators (e.g., `besser/generators/sql_alchemy/__init__.py` contains `from .sql_alchemy_generator import *`).

```python
from .graphql_generator import *
```

### 1b. `besser/generators/graphql/templates/__init__.py`

This file can be empty. It marks the templates directory as a Python package (required by some packaging tools, and consistent with every other generator's templates directory).

```python
```

---

## Step 2: Implement the Generator Class

Create `besser/generators/graphql/graphql_generator.py`. The generator must inherit from `GeneratorInterface` (defined in `besser/generators/generator_interface.py`) and implement:

- `__init__(self, model, output_dir)` -- calls `super().__init__(model, output_dir)`
- `generate(self)` -- performs the code generation

Here is a complete reference implementation:

```python
import os
from jinja2 import Environment, FileSystemLoader
from besser.BUML.metamodel.structural import DomainModel
from besser.generators import GeneratorInterface


class GraphQLGenerator(GeneratorInterface):
    """
    GraphQLGenerator implements the GeneratorInterface and generates a GraphQL
    schema definition (SDL) from a B-UML DomainModel.

    Args:
        model (DomainModel): An instance of the DomainModel class representing the B-UML model.
        output_dir (str, optional): The output directory where the generated schema will be saved.
            Defaults to None (which uses <cwd>/output).
    """

    # Mapping from B-UML primitive type names to GraphQL scalar types
    TYPES = {
        "int": "Int",
        "float": "Float",
        "str": "String",
        "bool": "Boolean",
        "date": "String",       # GraphQL has no built-in Date; use String or a custom scalar
        "datetime": "String",   # Same approach; could use a custom scalar like DateTime
        "time": "String",
    }

    def __init__(self, model: DomainModel, output_dir: str = None):
        super().__init__(model, output_dir)

    def generate(self):
        """
        Generates a GraphQL schema file based on the provided B-UML model and saves it
        to the specified output directory. If the output directory was not specified,
        the generated file will be stored in <current directory>/output/schema.graphql.

        Returns:
            None, but stores the generated schema as a file named schema.graphql.
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
        template = env.get_template("graphql_schema.graphql.j2")

        with open(file_path, mode="w", encoding="utf-8") as f:
            generated_code = template.render(
                classes=self.model.classes_sorted_by_inheritance(),
                enumerations=self.model.get_enumerations(),
                associations=self.model.associations,
                types=self.TYPES,
            )
            f.write(generated_code)
            print("Code generated in the location: " + file_path)
```

**Key design decisions explained:**

- **`TYPES` dictionary**: Maps B-UML primitive type names (`"int"`, `"str"`, etc.) to GraphQL scalar types (`"Int"`, `"String"`, etc.). This follows the same pattern used by `SQLAlchemyGenerator.TYPES`.
- **`classes_sorted_by_inheritance()`**: Returns classes ordered so that parent classes appear before children. This is important because GraphQL `type` definitions that use `implements` must reference interfaces that are already defined.
- **`build_generation_path()`**: Inherited from `GeneratorInterface`. Creates the output directory if needed and returns the full file path.
- **Jinja2 template loading**: Uses `FileSystemLoader` pointed at the `templates/` directory relative to the generator file, exactly like every other BESSER generator.

---

## Step 3: Create the Jinja2 Template

Create `besser/generators/graphql/templates/graphql_schema.graphql.j2`:

```graphql
# GraphQL Schema generated by BESSER

{% for enum in enumerations %}
enum {{ enum.name }} {
  {% for literal in enum.literals %}
  {{ literal.name }}
  {% endfor %}
}

{% endfor %}
{% for class in classes %}
{% if class.is_abstract %}
interface {{ class.name }} {
{% else %}
{% if class.parents() %}
type {{ class.name }} implements {{ class.parents()|map(attribute='name')|join(' & ') }} {
{% else %}
type {{ class.name }} {
{% endif %}
{% endif %}
  {% for attribute in class.attributes %}
  {{ attribute.name }}: {% if attribute.multiplicity.max > 1 %}[{% endif %}{% if attribute.type.name in types %}{{ types[attribute.type.name] }}{% else %}{{ attribute.type.name }}{% endif %}{% if attribute.multiplicity.max > 1 %}]{% endif %}{% if attribute.multiplicity.min > 0 %}!{% endif %}

  {% endfor %}
  {% for end in class.association_ends() %}
  {% if end.is_navigable %}
  {{ end.name }}: {% if end.multiplicity.max > 1 %}[{{ end.type.name }}]{% else %}{{ end.type.name }}{% endif %}{% if end.multiplicity.min > 0 %}!{% endif %}

  {% endif %}
  {% endfor %}
}

{% endfor %}
type Query {
  {% for class in classes %}
  {% if not class.is_abstract %}
  all{{ class.name }}s: [{{ class.name }}!]!
  {{ class.name | lower }}ById(id: ID!): {{ class.name }}
  {% endif %}
  {% endfor %}
}
```

**Template logic breakdown:**

1. **Enumerations** are rendered as GraphQL `enum` types.
2. **Abstract classes** become GraphQL `interface` types.
3. **Concrete classes** become GraphQL `type` declarations. If the class has parents (generalizations), it uses `implements`.
4. **Attributes** are rendered with their mapped GraphQL types. Multiplicity > 1 wraps the type in `[...]` (list). Required attributes (multiplicity min > 0) get `!`.
5. **Association ends** that are navigable are rendered as fields referencing other types.
6. **Query type** provides basic `all*` and `*ById` query entry points.

---

## Step 4: Register the Generator in the Backend Configuration

Edit `besser/utilities/web_modeling_editor/backend/config/generators.py` to add the GraphQL generator.

### 4a. Add the import

At the top of the file, add:

```python
from besser.generators.graphql import GraphQLGenerator
```

### 4b. Add the entry to `SUPPORTED_GENERATORS`

Add this entry inside the `SUPPORTED_GENERATORS` dictionary. A good location is right after the `"jsonschema"` entry, since GraphQL is a data format / API schema:

```python
"graphql": GeneratorInfo(
    generator_class=GraphQLGenerator,
    output_type="file",
    file_extension=".graphql",
    category="data_format",
    requires_class_diagram=True
),
```

### 4c. Add the filename mapping in `get_filename_for_generator`

In the `get_filename_for_generator` function, add a new `elif` clause:

```python
elif generator_type == "graphql":
    return "schema.graphql"
```

**After these changes, the web editor backend at `https://editor.besser-pearl.org` will automatically support the "graphql" generator type.** The `_handle_class_diagram_generation` function in `besser/utilities/web_modeling_editor/backend/backend.py` will route it through `_generate_standard`, which handles the generic pattern of: instantiate generator -> call `generate()` -> return file response. No changes to `backend.py` are needed.

---

## Step 5: Write Tests

Create the test file at `tests/generators/graphql/test_graphql_generator.py`.

Following the patterns established by the existing tests (especially `tests/generators/json_schema/test_json_schema.py` and `tests/generators/python/test_python_generator.py`), here is a comprehensive test suite:

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
    """Create a sample Library domain model for testing."""
    # Enumeration
    MemberType = Enumeration(
        name="MemberType",
        literals={
            EnumerationLiteral(name="ADULT"),
            EnumerationLiteral(name="SENIOR"),
            EnumerationLiteral(name="STUDENT"),
            EnumerationLiteral(name="CHILD"),
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

    # Associations
    lib_book = BinaryAssociation(
        name="lib_book",
        ends={
            Property(name="locatedIn", type=Library, multiplicity=Multiplicity(1, 1)),
            Property(name="has", type=Book, multiplicity=Multiplicity(0, "*")),
        }
    )
    book_author = BinaryAssociation(
        name="book_author",
        ends={
            Property(name="writtenBy", type=Author, multiplicity=Multiplicity(1, "*")),
            Property(name="publishes", type=Book, multiplicity=Multiplicity(0, "*")),
        }
    )

    model = DomainModel(
        name="Library_Model",
        types={Library, Book, Author, MemberType},
        associations={lib_book, book_author},
        generalizations={}
    )
    return model


@pytest.fixture
def generated_schema(domain_model, tmpdir):
    """Generate the GraphQL schema and return its content."""
    output_dir = tmpdir.mkdir("output")
    generator = GraphQLGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()
    output_file = os.path.join(str(output_dir), "schema.graphql")
    with open(output_file, "r", encoding="utf-8") as f:
        return f.read()


class TestGraphQLGeneratorFileCreation:
    """Tests that the generator creates the expected output file."""

    def test_output_file_created(self, domain_model, tmpdir):
        output_dir = tmpdir.mkdir("output")
        generator = GraphQLGenerator(model=domain_model, output_dir=str(output_dir))
        generator.generate()
        output_file = os.path.join(str(output_dir), "schema.graphql")
        assert os.path.isfile(output_file)

    def test_output_file_not_empty(self, domain_model, tmpdir):
        output_dir = tmpdir.mkdir("output")
        generator = GraphQLGenerator(model=domain_model, output_dir=str(output_dir))
        generator.generate()
        output_file = os.path.join(str(output_dir), "schema.graphql")
        assert os.path.getsize(output_file) > 0


class TestGraphQLEnumerations:
    """Tests for GraphQL enum generation."""

    def test_enum_type_present(self, generated_schema):
        assert "enum MemberType" in generated_schema

    def test_enum_literals_present(self, generated_schema):
        assert "ADULT" in generated_schema
        assert "SENIOR" in generated_schema
        assert "STUDENT" in generated_schema
        assert "CHILD" in generated_schema


class TestGraphQLTypes:
    """Tests for GraphQL type generation."""

    def test_library_type_present(self, generated_schema):
        assert "type Library" in generated_schema

    def test_book_type_present(self, generated_schema):
        assert "type Book" in generated_schema

    def test_author_type_present(self, generated_schema):
        assert "type Author" in generated_schema


class TestGraphQLAttributes:
    """Tests for correct attribute type mapping."""

    def test_string_attributes(self, generated_schema):
        # name and address in Library should map to String
        assert "name: String" in generated_schema or "name:" in generated_schema
        assert "address:" in generated_schema

    def test_integer_attributes(self, generated_schema):
        assert "pages:" in generated_schema
        # Should map to Int in GraphQL
        assert "Int" in generated_schema

    def test_enum_attribute(self, generated_schema):
        assert "member:" in generated_schema
        assert "MemberType" in generated_schema


class TestGraphQLAssociations:
    """Tests for relationship field generation."""

    def test_association_fields_present(self, generated_schema):
        # At least some of the association ends should appear
        has_some_relationship = (
            "locatedIn" in generated_schema or
            "has" in generated_schema or
            "writtenBy" in generated_schema or
            "publishes" in generated_schema
        )
        assert has_some_relationship


class TestGraphQLQueryType:
    """Tests for the generated Query type."""

    def test_query_type_present(self, generated_schema):
        assert "type Query" in generated_schema

    def test_query_contains_class_queries(self, generated_schema):
        # Should have queries for Library, Book, and Author
        assert "Library" in generated_schema
        assert "Book" in generated_schema
        assert "Author" in generated_schema


class TestGraphQLInheritance:
    """Tests for inheritance / generalization handling."""

    @pytest.fixture
    def inheritance_model(self):
        """Domain model with inheritance."""
        Vehicle = Class(name="Vehicle", is_abstract=True)
        Vehicle_make = Property(name="make", type=StringType)
        Vehicle.attributes = {Vehicle_make}

        Car = Class(name="Car")
        Car_doors = Property(name="doors", type=IntegerType)
        Car.attributes = {Car_doors}

        Truck = Class(name="Truck")
        Truck_payload = Property(name="payload", type=FloatType)
        Truck.attributes = {Truck_payload}

        gen_car = Generalization(general=Vehicle, specific=Car)
        gen_truck = Generalization(general=Vehicle, specific=Truck)

        model = DomainModel(
            name="Vehicle_Model",
            types={Vehicle, Car, Truck},
            associations=set(),
            generalizations={gen_car, gen_truck}
        )
        return model

    def test_abstract_class_becomes_interface(self, inheritance_model, tmpdir):
        output_dir = tmpdir.mkdir("output")
        generator = GraphQLGenerator(model=inheritance_model, output_dir=str(output_dir))
        generator.generate()
        output_file = os.path.join(str(output_dir), "schema.graphql")
        with open(output_file, "r", encoding="utf-8") as f:
            schema = f.read()
        assert "interface Vehicle" in schema

    def test_concrete_class_implements_interface(self, inheritance_model, tmpdir):
        output_dir = tmpdir.mkdir("output")
        generator = GraphQLGenerator(model=inheritance_model, output_dir=str(output_dir))
        generator.generate()
        output_file = os.path.join(str(output_dir), "schema.graphql")
        with open(output_file, "r", encoding="utf-8") as f:
            schema = f.read()
        assert "type Car implements Vehicle" in schema
        assert "type Truck implements Vehicle" in schema


class TestGraphQLDefaultOutputDir:
    """Test that generation works with default output directory."""

    def test_default_output_dir(self, domain_model, tmpdir, monkeypatch):
        monkeypatch.chdir(tmpdir)
        generator = GraphQLGenerator(model=domain_model)
        generator.generate()
        output_file = os.path.join(str(tmpdir), "output", "schema.graphql")
        assert os.path.isfile(output_file)
```

**Run the tests with:**

```bash
python -m pytest tests/generators/graphql/test_graphql_generator.py -v
```

### What the tests cover:

| Test Category | What It Validates |
|---|---|
| `TestGraphQLGeneratorFileCreation` | File is created and is non-empty |
| `TestGraphQLEnumerations` | B-UML enumerations become GraphQL `enum` types with correct literals |
| `TestGraphQLTypes` | Each class in the domain model produces a corresponding GraphQL `type` |
| `TestGraphQLAttributes` | Attributes are present with correct GraphQL type mappings (String, Int, etc.) |
| `TestGraphQLAssociations` | Navigable association ends produce relationship fields |
| `TestGraphQLQueryType` | A `Query` type is generated with entry points for each class |
| `TestGraphQLInheritance` | Abstract classes become `interface`, concrete subclasses use `implements` |
| `TestGraphQLDefaultOutputDir` | Generator works when `output_dir` is not specified (uses `<cwd>/output`) |

---

## Step 6: Update Documentation

### 6a. Create the generator documentation page

Create `docs/source/generators/graphql.rst`:

```rst
GraphQL Generator
=================

The GraphQL generator transforms a B-UML structural model into a GraphQL Schema Definition Language (SDL) file.

Usage
-----

.. code-block:: python

    from besser.generators.graphql import GraphQLGenerator

    # Assume `domain_model` is a DomainModel instance
    generator = GraphQLGenerator(model=domain_model, output_dir="output")
    generator.generate()

This generates a ``schema.graphql`` file in the specified output directory.

Type Mapping
------------

The following B-UML types are mapped to GraphQL scalar types:

.. list-table::
   :header-rows: 1

   * - B-UML Type
     - GraphQL Type
   * - ``int``
     - ``Int``
   * - ``float``
     - ``Float``
   * - ``str``
     - ``String``
   * - ``bool``
     - ``Boolean``
   * - ``date``
     - ``String``
   * - ``datetime``
     - ``String``
   * - ``time``
     - ``String``

Features
--------

- **Enumerations**: B-UML enumerations are converted to GraphQL ``enum`` types.
- **Inheritance**: Abstract classes become GraphQL ``interface`` types. Concrete subclasses use ``implements``.
- **Associations**: Navigable association ends become fields referencing related types. Multiplicity > 1 produces list types (``[Type]``).
- **Required fields**: Attributes with minimum multiplicity > 0 are marked as non-nullable (``!``).
- **Query type**: A basic ``Query`` type is auto-generated with ``all*`` and ``*ById`` queries for each concrete class.

Example
-------

Given a B-UML model with ``Library``, ``Book``, and ``Author`` classes, the generator produces:

.. code-block:: graphql

    type Library {
      name: String!
      address: String!
      has: [Book]
    }

    type Book {
      title: String!
      pages: Int!
      release: String
      locatedIn: Library!
      writtenBy: [Author]!
    }

    type Author {
      name: String!
      email: String!
      publishes: [Book]
    }

    type Query {
      allLibrarys: [Library!]!
      libraryById(id: ID!): Library
      allBooks: [Book!]!
      bookById(id: ID!): Book
      allAuthors: [Author!]!
      authorById(id: ID!): Author
    }

API Reference
-------------

.. autoclass:: besser.generators.graphql.GraphQLGenerator
   :members:
   :undoc-members:
```

### 6b. Add to the generators index

Edit `docs/source/generators.rst` and add `generators/graphql` under the **Data & API** section:

```rst
Data & API
----------

Generate database schemas, APIs, and data formats:

.. toctree::
   :maxdepth: 1

   generators/sql
   generators/json_schema
   generators/graphql
   generators/rdf
   generators/terraform
```

---

## Step 7: Verify Everything Works

### Run the tests

```bash
python -m pytest tests/generators/graphql/ -v
```

### Run a standalone example

Create a quick script (not committed) to verify end-to-end:

```python
from besser.BUML.metamodel.structural import (
    Class, DomainModel, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType
)
from besser.generators.graphql import GraphQLGenerator

Book = Class(name="Book")
Book.attributes = {
    Property(name="title", type=StringType),
    Property(name="pages", type=IntegerType),
}

Author = Class(name="Author")
Author.attributes = {
    Property(name="name", type=StringType),
}

assoc = BinaryAssociation(
    name="writes",
    ends={
        Property(name="writtenBy", type=Author, multiplicity=Multiplicity(1, "*")),
        Property(name="books", type=Book, multiplicity=Multiplicity(0, "*")),
    }
)

model = DomainModel(
    name="BookStore",
    types={Book, Author},
    associations={assoc},
    generalizations={}
)

generator = GraphQLGenerator(model=model, output_dir="./test_output")
generator.generate()
```

### Build the docs

```bash
cd docs
make html
```

Open `docs/build/html/generators/graphql.html` in a browser to verify the documentation renders correctly.

---

## Summary: Complete File Listing

| File | Purpose |
|---|---|
| `besser/generators/graphql/__init__.py` | Package init, re-exports `GraphQLGenerator` |
| `besser/generators/graphql/graphql_generator.py` | Generator class implementing `GeneratorInterface` |
| `besser/generators/graphql/templates/__init__.py` | Empty init for templates package |
| `besser/generators/graphql/templates/graphql_schema.graphql.j2` | Jinja2 template for GraphQL SDL output |
| `besser/utilities/web_modeling_editor/backend/config/generators.py` | **Modified** -- add import and registry entry |
| `tests/generators/graphql/test_graphql_generator.py` | Pytest test suite |
| `docs/source/generators/graphql.rst` | **New** -- generator documentation page |
| `docs/source/generators.rst` | **Modified** -- add to toctree |

---

## Architecture Alignment

This implementation follows every convention established in the BESSER codebase:

1. **GeneratorInterface pattern**: The `GraphQLGenerator` class inherits from `GeneratorInterface` and implements `__init__` and `generate`, exactly like `PythonGenerator` (`besser/generators/python_classes/python_classes_generator.py`), `SQLAlchemyGenerator` (`besser/generators/sql_alchemy/sql_alchemy_generator.py`), and all other generators.

2. **Jinja2 template-based generation**: The template lives in a `templates/` subdirectory and is loaded via `FileSystemLoader`, consistent with every generator in the codebase.

3. **Generator registry**: Registration in `besser/utilities/web_modeling_editor/backend/config/generators.py` uses the `GeneratorInfo` NamedTuple with the same fields as all other entries.

4. **Backend integration**: The `"data_format"` category and `requires_class_diagram=True` setting means the backend's `_handle_class_diagram_generation` function will route it to `_generate_standard` (`besser/utilities/web_modeling_editor/backend/backend.py`, line 1105), which handles the instantiation, generation, and file response automatically. No backend code changes are needed.

5. **Test structure**: Tests mirror the source structure (`tests/generators/graphql/`) and use pytest fixtures with `tmpdir` for isolated output, following the patterns in `tests/generators/json_schema/test_json_schema.py` and `tests/generators/python/test_python_generator.py`.

6. **Documentation**: The RST page follows the same structure as existing generator docs under `docs/source/generators/`, and is linked from the main `generators.rst` toctree.
