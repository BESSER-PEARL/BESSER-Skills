# Testing Patterns for BESSER Generators

## Overview

The BESSER codebase has a well-established testing pattern for generators. All generator tests live under `tests/generators/<generator_name>/` and follow a consistent structure: build a domain model fixture, run the generator against a temporary directory, then assert on the generated output files. Below is a comprehensive guide drawn directly from the existing test suite.

---

## 1. The Standard Test Pattern

Every generator test in BESSER follows this three-phase pattern:

1. **Arrange** -- Build a `DomainModel` (and optionally other metamodel objects) as a pytest fixture or helper function.
2. **Act** -- Instantiate the generator with the model and a temporary output directory, then call `.generate()`.
3. **Assert** -- Verify that output files exist, then read their contents and check for expected code fragments, structures, or behaviors.

Here is a minimal working example modeled after the existing tests in `tests/generators/sql/test_sql_generator.py` and `tests/generators/python/test_python_generator.py`:

```python
import os
import pytest
from besser.generators.sql import SQLGenerator
from besser.BUML.metamodel.structural import (
    Class, DomainModel, StringType, IntegerType,
    Property, BinaryAssociation, Multiplicity
)

@pytest.fixture
def domain_model():
    # Define classes
    library = Class(name="Library", attributes={
        Property(name="name", type=StringType),
        Property(name="address", type=StringType)
    })
    book = Class(name="Book", attributes={
        Property(name="title", type=StringType),
        Property(name="pages", type=IntegerType)
    })

    # Define associations
    lib_book = BinaryAssociation(
        name="lib_book_assoc",
        ends={
            Property(name="locatedIn", type=library, multiplicity=Multiplicity(1, 1)),
            Property(name="has", type=book, multiplicity=Multiplicity(0, "*"))
        }
    )

    # Build the domain model
    model = DomainModel(
        name="Library_model",
        types={library, book},
        associations={lib_book}
    )
    return model

def test_sql_tables_generated(domain_model, tmpdir):
    # Arrange: create output directory inside tmpdir
    output_dir = tmpdir.mkdir("output")
    generator = SQLGenerator(model=domain_model, output_dir=str(output_dir), sql_dialect="postgresql")

    # Act
    generator.generate()

    # Assert: file exists
    output_file = os.path.join(str(output_dir), "tables_postgresql.sql")
    assert os.path.isfile(output_file)

    # Assert: content is correct
    with open(output_file, "r", encoding="utf-8") as f:
        generated_code = f.read()

    assert "CREATE TABLE library" in generated_code
    assert "CREATE TABLE book" in generated_code
```

---

## 2. Setting Up Domain Model Fixtures

### Use `@pytest.fixture` for Reusable Models

The recommended pattern (used in `tests/generators/sqlalchemy/test_sqlalchemy_generator.py`, `tests/generators/backend/test_backend.py`, `tests/generators/rdf/test_rdf_generator.py`, and others) is to define the domain model as a `@pytest.fixture`:

```python
@pytest.fixture
def domain_model():
    # Classes
    Author = Class(name="Author")
    Book = Class(name="Book")

    # Attributes
    Author_name = Property(name="name", type=StringType)
    Author_email = Property(name="email", type=StringType)
    Author.attributes = {Author_name, Author_email}

    Book_title = Property(name="title", type=StringType)
    Book_pages = Property(name="pages", type=IntegerType)
    Book_release = Property(name="release", type=DateType)
    Book.attributes = {Book_release, Book_pages, Book_title}

    # Associations
    Author_Book = BinaryAssociation(
        name="Author_Book",
        ends={
            Property(name="writtenBy", type=Author, multiplicity=Multiplicity(1, 9999)),
            Property(name="publishes", type=Book, multiplicity=Multiplicity(0, 9999))
        }
    )

    # Domain Model
    model = DomainModel(
        name="Class_Diagram",
        types={Author, Book},
        associations={Author_Book},
        generalizations={}
    )
    return model
```

Key points about fixture construction:

- **Primitive types** are imported singletons: `StringType`, `IntegerType`, `DateType`, `FloatType`, `DateTimeType`, `BooleanType`, `AnyType`.
- **Multiplicity** uses `Multiplicity(min, max)`. Use `"*"` or `9999` for unbounded maximums (the constant `UNLIMITED_MAX_MULTIPLICITY = 9999` is used throughout the codebase).
- **Enumerations** are created with `Enumeration(name=..., literals={EnumerationLiteral(name=...), ...})`.
- **Generalizations** (inheritance) are created with `Generalization(general=parent_class, specific=child_class)`.
- **Attributes are sets**, not lists: `Author.attributes = {Author_name, Author_email}`.

### Use Multiple Focused Fixtures

The backend tests (`tests/generators/backend/test_backend.py`) demonstrate how to use multiple fixtures that each test a different scenario:

```python
@pytest.fixture
def simple_model():
    """Simple N:M relationship test"""
    class1 = Class(name="name1", attributes={
        Property(name="attr1", type=IntegerType),
    })
    class2 = Class(name="name2", attributes={
        Property(name="attr2", type=IntegerType)
    })
    association = BinaryAssociation(
        name="name_assoc",
        ends={
            Property(name="assocs1", type=class1, multiplicity=Multiplicity(1, "*")),
            Property(name="assocs2", type=class2, multiplicity=Multiplicity(1, "*"))
        }
    )
    model = DomainModel(name="Name", types={class1, class2}, associations={association})
    return model

@pytest.fixture
def relationship_model():
    """Test model with 1:1, N:1, and 1:N relationships."""
    PhysicalAsset = Class(name="PhysicalAsset", attributes={
        Property(name="attribute", type=StringType),
    })
    DigitalTwin = Class(name="DigitalTwin", attributes={
        Property(name="attribute", type=StringType),
    })
    Sensor = Class(name="Sensor", attributes={
        Property(name="type", type=StringType),
        Property(name="timestamp", type=DateTimeType),
        Property(name="value", type=FloatType),
    })
    # ... associations and return model
```

### Use Plain Helper Functions for One-Off Models

The action language tests (`tests/generators/action_language/test_generation.py`) use a helper function instead of a fixture when the model is only needed by one test:

```python
def get_model() -> DomainModel:
    Book = Class(name="Book")
    Library = Class(name="Library")
    # ... build and return model

def test_REST_API_Generation():
    model = get_model()
    # ... run generation and assertions
```

### Composable Fixtures for Shared Components

The OCL integration tests (`tests/generators/backend/test_ocl_pydantic_integration.py`) show a pattern of composable fixtures where individual classes are fixtures that get composed into a domain model:

```python
@pytest.fixture
def player_class():
    """Create a Player class with various attributes."""
    return Class(name="Player", attributes={
        Property(name="age", type=IntegerType),
        Property(name="name", type=StringType),
        Property(name="salary", type=FloatType),
        Property(name="jerseyNumber", type=IntegerType),
    })

@pytest.fixture
def team_class():
    """Create a Team class."""
    return Class(name="Team", attributes={
        Property(name="name", type=StringType),
        Property(name="city", type=StringType),
    })

@pytest.fixture
def simple_domain_model(player_class, team_class):
    """Create a simple domain model with Player and Team."""
    association = BinaryAssociation(
        name="team_player",
        ends={
            Property(name="team", type=team_class, multiplicity=Multiplicity(1, 1)),
            Property(name="players", type=player_class, multiplicity=Multiplicity(0, 9999)),
        }
    )
    return DomainModel(
        name="TestModel",
        types={player_class, team_class},
        associations={association}
    )
```

---

## 3. Directory Management: `tmpdir` vs Manual Directories

### Use `tmpdir` (Recommended)

The majority of well-written generator tests in BESSER use pytest's `tmpdir` fixture. This is the recommended approach. The following generators all use this pattern:

- `tests/generators/sqlalchemy/test_sqlalchemy_generator.py`
- `tests/generators/sql/test_sql_generator.py`
- `tests/generators/python/test_python_generator.py`
- `tests/generators/backend/test_backend.py`
- `tests/generators/backend/test_backend_full_uml.py`
- `tests/generators/rdf/test_rdf_generator.py`
- `tests/generators/json_schema/test_json_schema.py`

The standard pattern is:

```python
def test_generator_output(domain_model, tmpdir):
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "expected_filename.py")
    assert os.path.isfile(output_file)
```

Benefits of `tmpdir`:
- Automatic cleanup after the test -- no need for `shutil.rmtree()` or `os.remove()`.
- Each test gets an isolated temporary directory, preventing test interference.
- Unique per-test-run, avoiding collisions when running tests in parallel.

You can also use `tempfile.TemporaryDirectory()` as a context manager (as seen in `tests/generators/backend/test_ocl_pydantic_integration.py`):

```python
def test_constraint_validation(self, player_class, simple_domain_model):
    with tempfile.TemporaryDirectory() as tmpdir:
        classes = generate_and_load_pydantic_classes(simple_domain_model, tmpdir)
        # ... assertions
```

### Avoid Manual Directories (Anti-Pattern)

Some older tests in the codebase (e.g., `tests/generators/flutter/test_dart_code.py`, `tests/generators/backend/test_pydantic.py`) write to hardcoded paths like `output/` and then manually clean up with `shutil.rmtree("output")` or `os.remove(output_file)`. This is fragile:

```python
# AVOID this pattern in new tests:
def test_file_generation():
    pydantic_model = PydanticGenerator(model=domain_model, backend=False)
    pydantic_model.generate()

    output_file = 'output/pydantic_classes.py'
    assert os.path.exists(output_file)
    # ... assertions ...
    os.remove(output_file)  # fragile: skipped if test fails
```

Problems with manual directory management:
- If the test fails before cleanup, leftover files pollute the working directory.
- Tests become dependent on the current working directory.
- Parallel test execution can cause conflicts.

**Always use `tmpdir` for new tests.**

---

## 4. Assertion Patterns

### 4.1 File Existence Checks

Always verify that the expected output files were created:

```python
output_file = os.path.join(str(output_dir), "sql_alchemy.py")
assert os.path.isfile(output_file)
```

For generators that produce multiple files (like `BackendGenerator`):

```python
api_file = os.path.join(str(output_dir), "main_api.py")
pydantic_file = os.path.join(str(output_dir), "pydantic_classes.py")
sqlalchemy_file = os.path.join(str(output_dir), "sql_alchemy.py")

assert os.path.isfile(api_file)
assert os.path.isfile(pydantic_file)
assert os.path.isfile(sqlalchemy_file)
```

### 4.2 Content String Matching

The most common assertion pattern is reading the generated file and checking for expected strings:

```python
with open(output_file, "r", encoding="utf-8") as f:
    generated_code = f.read()

# Check for structural elements
assert "CREATE TABLE library" in generated_code
assert "CREATE TABLE book" in generated_code

# Check for specific columns/fields
assert "name VARCHAR(100) NOT NULL" in generated_code
assert "pages INTEGER NOT NULL" in generated_code
```

### 4.3 Marker-Based Batch Assertions

For checking many expected patterns at once, the backend tests use a list-based approach:

```python
pydantic_markers = [
    "class name1Create(BaseModel):",
    "attr1: int",
    "assocs2: List[int]",
    "class name2Create(BaseModel):",
    "attr2: int",
    "assocs1: List[int]"
]

for marker in pydantic_markers:
    assert marker in pydantic_code, f"Missing expected Pydantic code: {marker}"
```

### 4.4 Expected Output Block Matching

For generators where the exact output format matters, define expected blocks and check containment (from `tests/generators/python/test_python_generator.py`):

```python
library_output = """
class Library:

    def __init__(self, name: str, address: str, Book_end: set["Book"] = None):
        self.name = name
        self.address = address
        self.Book_end = Book_end if Book_end is not None else set()
"""

def test_generator(domain_model, tmpdir):
    output_dir = tmpdir.mkdir("output")
    generator = PythonGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "classes.py")
    with open(output_file, "r", encoding="utf-8") as f:
        generated_code = f.read()

    assert library_output in generated_code
```

### 4.5 Section-Based Assertions (Negative Checks)

For checking that something does NOT appear in a specific section of the output:

```python
# Extract only the PhysicalAsset class section
physicalasset_section = sqlalchemy_code.split("class PhysicalAsset(Base):")[1].split("class ")[0]
assert "ForeignKey" not in physicalasset_section, "PhysicalAsset should not have any ForeignKey"
assert "dt_id" not in physicalasset_section, "PhysicalAsset should not have dt_id FK"
```

### 4.6 Dynamic Import and Runtime Validation

For the most thorough testing, you can dynamically import the generated code and test it at runtime. This is used in `tests/generators/sqlalchemy/test_sqlalchemy_generator.py` and `tests/generators/python/test_python_generator.py`:

```python
import importlib.util
import sys

@pytest.fixture
def generated_module(domain_model, tmpdir):
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=domain_model, output_dir=str(output_dir))
    generator.generate()

    file_path = os.path.join(str(output_dir), "generated_file.py")

    # Dynamically import the generated module
    spec = importlib.util.spec_from_file_location("generated_module", file_path)
    module = importlib.util.module_from_spec(spec)
    sys.modules["generated_module"] = module
    spec.loader.exec_module(module)
    return module

def test_runtime_behavior(generated_module):
    MyClass = getattr(generated_module, "MyClass")
    instance = MyClass(name="test", value=42)
    assert instance.name == "test"
```

The OCL integration tests use `exec()` for the same purpose:

```python
def generate_and_load_pydantic_classes(domain_model, output_dir):
    generator = PydanticGenerator(model=domain_model, backend=True, output_dir=output_dir)
    generator.generate()

    file_path = os.path.join(output_dir, "pydantic_classes.py")
    with open(file_path, 'r') as f:
        code = f.read()

    namespace = {}
    exec(code, namespace)
    return namespace
```

---

## 5. Complete Working Example: Testing a New Generator

Here is a full, copy-paste-ready example for testing a hypothetical new generator:

```python
"""
tests/generators/my_generator/test_my_generator.py
"""
import os
import pytest
from besser.generators.my_generator import MyGenerator
from besser.BUML.metamodel.structural import (
    Class, DomainModel, StringType, IntegerType, DateType, FloatType,
    Property, BinaryAssociation, Multiplicity, Enumeration,
    EnumerationLiteral, Generalization
)


# ---------------------------------------------------------------------------
# Fixtures
# ---------------------------------------------------------------------------

@pytest.fixture
def simple_model():
    """A minimal domain model with two classes and one association."""
    author = Class(name="Author", attributes={
        Property(name="name", type=StringType),
        Property(name="email", type=StringType),
    })
    book = Class(name="Book", attributes={
        Property(name="title", type=StringType),
        Property(name="pages", type=IntegerType),
        Property(name="release", type=DateType),
    })

    author_book = BinaryAssociation(
        name="AuthorBook",
        ends={
            Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, 9999)),
            Property(name="publishes", type=book, multiplicity=Multiplicity(0, 9999)),
        }
    )

    return DomainModel(
        name="SimpleModel",
        types={author, book},
        associations={author_book},
    )


@pytest.fixture
def model_with_inheritance():
    """A domain model that includes enumerations and inheritance."""
    role_enum = Enumeration(
        name="Role",
        literals={
            EnumerationLiteral(name="ADMIN"),
            EnumerationLiteral(name="USER"),
        }
    )
    person = Class(name="Person", attributes={
        Property(name="name", type=StringType),
        Property(name="role", type=role_enum),
    })
    employee = Class(name="Employee", attributes={
        Property(name="salary", type=FloatType),
    })

    gen = Generalization(general=person, specific=employee)

    return DomainModel(
        name="InheritanceModel",
        types={person, employee, role_enum},
        associations=set(),
        generalizations={gen},
    )


# ---------------------------------------------------------------------------
# Tests: File Generation
# ---------------------------------------------------------------------------

def test_output_file_created(simple_model, tmpdir):
    """Verify that the generator creates the expected output file."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=simple_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    assert os.path.isfile(output_file), "Expected output file was not created"
    assert os.path.getsize(output_file) > 0, "Output file is empty"


def test_output_not_empty(simple_model, tmpdir):
    """Verify the generated file is non-empty."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=simple_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    assert os.path.getsize(output_file) > 0


# ---------------------------------------------------------------------------
# Tests: Content Verification
# ---------------------------------------------------------------------------

def test_classes_present(simple_model, tmpdir):
    """Verify that all domain classes appear in the generated output."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=simple_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    with open(output_file, "r", encoding="utf-8") as f:
        code = f.read()

    assert "Author" in code, "Author class not found in generated output"
    assert "Book" in code, "Book class not found in generated output"


def test_attributes_present(simple_model, tmpdir):
    """Verify that class attributes appear in the generated output."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=simple_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    with open(output_file, "r", encoding="utf-8") as f:
        code = f.read()

    expected_markers = ["name", "email", "title", "pages", "release"]
    for marker in expected_markers:
        assert marker in code, f"Missing attribute '{marker}' in generated output"


def test_associations_present(simple_model, tmpdir):
    """Verify that associations are represented in the generated output."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=simple_model, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    with open(output_file, "r", encoding="utf-8") as f:
        code = f.read()

    assert "writtenBy" in code or "publishes" in code, \
        "Association ends not found in generated output"


def test_inheritance_structure(model_with_inheritance, tmpdir):
    """Verify that inheritance relationships are properly generated."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=model_with_inheritance, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    with open(output_file, "r", encoding="utf-8") as f:
        code = f.read()

    assert "Person" in code
    assert "Employee" in code
    # Check that Employee inherits from or references Person
    assert "Person" in code and "Employee" in code


def test_enumerations_present(model_with_inheritance, tmpdir):
    """Verify that enumerations are generated correctly."""
    output_dir = tmpdir.mkdir("output")
    generator = MyGenerator(model=model_with_inheritance, output_dir=str(output_dir))
    generator.generate()

    output_file = os.path.join(str(output_dir), "my_output.py")
    with open(output_file, "r", encoding="utf-8") as f:
        code = f.read()

    assert "Role" in code
    assert "ADMIN" in code
    assert "USER" in code


# ---------------------------------------------------------------------------
# Tests: Negative / Edge Cases
# ---------------------------------------------------------------------------

def test_no_cross_contamination(simple_model, tmpdir):
    """Verify that generating twice into different directories produces independent output."""
    dir1 = tmpdir.mkdir("output1")
    dir2 = tmpdir.mkdir("output2")

    gen1 = MyGenerator(model=simple_model, output_dir=str(dir1))
    gen1.generate()

    gen2 = MyGenerator(model=simple_model, output_dir=str(dir2))
    gen2.generate()

    file1 = os.path.join(str(dir1), "my_output.py")
    file2 = os.path.join(str(dir2), "my_output.py")

    with open(file1, "r") as f1, open(file2, "r") as f2:
        assert f1.read() == f2.read(), "Identical models should produce identical output"
```

---

## 6. Summary of Best Practices

| Practice | Recommendation | Example Test File |
|---|---|---|
| Fixture style | Use `@pytest.fixture` for domain models | `test_sqlalchemy_generator.py` |
| Composable fixtures | Separate class fixtures from model fixtures | `test_ocl_pydantic_integration.py` |
| Directory management | Use `tmpdir` (not manual `output/`) | `test_backend.py`, `test_sql_generator.py` |
| File verification | Always check `os.path.isfile()` first | All test files |
| Content assertions | Use `in` checks with descriptive messages | `test_backend.py` |
| Batch markers | Use a list of markers with a loop | `test_backend.py` |
| Block matching | Define expected output blocks as strings | `test_python_generator.py` |
| Runtime validation | Use `importlib.util` or `exec()` for deep validation | `test_sqlalchemy_generator.py`, `test_ocl_pydantic_integration.py` |
| Cleanup | Let `tmpdir` handle it automatically | All modern tests |
| Docstrings | Add docstrings describing the scenario being tested | `test_backend.py`, `test_backend_full_uml.py` |

### Imports You Will Need

```python
import os
import pytest
from besser.BUML.metamodel.structural import (
    Class, DomainModel, Property, Multiplicity,
    BinaryAssociation, Generalization,
    Enumeration, EnumerationLiteral,
    StringType, IntegerType, FloatType, DateType, DateTimeType, BooleanType, AnyType
)
from besser.generators.your_generator import YourGenerator
```

### Running Your Tests

```bash
# Run all tests for your generator
python -m pytest tests/generators/my_generator/ -v

# Run a specific test
python -m pytest tests/generators/my_generator/test_my_generator.py::test_output_file_created -v

# Run with keyword matching
python -m pytest tests/generators/my_generator/ -k "inheritance"
```
