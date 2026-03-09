# Test Patterns for BESSER Generators

## Typical Test Pattern

Generator tests in BESSER follow a consistent pattern: build small, inline domain models as pytest fixtures, use `tmp_path` for output directories, instantiate the generator, call `generate()`, then assert on both file existence and file content. Here is a complete working example.

## File Location

Place your test file at:

```
tests/generators/my_generator/test_my_generator.py
```

This mirrors the source structure under `besser/generators/my_generator/`.

## Setting Up Domain Model Fixtures

Build small, focused domain models directly in `@pytest.fixture` functions. Do not depend on external files -- inline models are self-contained and make tests easy to understand and maintain.

```python
import os
import pytest
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)
from besser.generators.my_generator import MyGenerator


@pytest.fixture
def domain_model():
    """Build a small test model with a single class."""
    name_prop = Property(name="name", type=StringType)
    age_prop = Property(name="age", type=IntegerType)
    person = Class(name="Person", attributes={name_prop, age_prop})
    model = DomainModel(name="TestModel", types={person})
    return model
```

You can create additional fixtures for more complex scenarios (associations, inheritance, empty models):

```python
@pytest.fixture
def model_with_association():
    """Build a model with two classes and a binary association."""
    name_prop = Property(name="name", type=StringType)
    title_prop = Property(name="title", type=StringType)

    author = Class(name="Author", attributes={name_prop})
    book = Class(name="Book", attributes={title_prop})

    writes = BinaryAssociation(
        name="writes",
        ends={
            Property(name="author", type=author, multiplicity=Multiplicity(1, 1)),
            Property(name="books", type=book, multiplicity=Multiplicity(0, 9999)),
        },
    )

    model = DomainModel(
        name="LibraryModel",
        types={author, book},
        associations={writes},
    )
    return model


@pytest.fixture
def empty_model():
    """Edge case: a domain model with no classes."""
    return DomainModel(name="EmptyModel", types=set())
```

Note: `9999` is the `UNLIMITED_MAX_MULTIPLICITY` constant used throughout the codebase to represent "many" (`*`).

## Use `tmp_path`, Not Manual Directories

Always use pytest's built-in `tmp_path` fixture for output directories. It is automatically created before each test and cleaned up after, so you never have to worry about leftover files or manual `shutil.rmtree()` calls.

```python
@pytest.fixture
def output_dir(tmp_path):
    return str(tmp_path)
```

If for some reason you cannot use `tmp_path` (rare), clean up manually with `shutil.rmtree()` in a `try/finally` block. But `tmp_path` is the recommended and standard approach.

## Complete Test File

```python
import os
import pytest
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)
from besser.generators.my_generator import MyGenerator


# ---------- Fixtures ----------

@pytest.fixture
def domain_model():
    """Build a small test model."""
    name_prop = Property(name="name", type=StringType)
    age_prop = Property(name="age", type=IntegerType)
    person = Class(name="Person", attributes={name_prop, age_prop})
    model = DomainModel(name="TestModel", types={person})
    return model


@pytest.fixture
def output_dir(tmp_path):
    return str(tmp_path)


# ---------- Tests: File Existence ----------

def test_generate_creates_output_file(domain_model, output_dir):
    """Generator should produce the expected output file."""
    gen = MyGenerator(model=domain_model, output_dir=output_dir)
    gen.generate()
    assert os.path.exists(os.path.join(output_dir, "my_output.py"))


# ---------- Tests: Content Verification ----------

def test_generate_contains_class_name(domain_model, output_dir):
    """Generated file should contain the class name and attribute names."""
    gen = MyGenerator(model=domain_model, output_dir=output_dir)
    gen.generate()
    with open(os.path.join(output_dir, "my_output.py")) as f:
        content = f.read()
    assert "Person" in content
    assert "name" in content
    assert "age" in content


# ---------- Tests: Edge Cases ----------

def test_generate_empty_model(output_dir):
    """Generator should handle an empty model without crashing."""
    empty = DomainModel(name="EmptyModel", types=set())
    gen = MyGenerator(model=empty, output_dir=output_dir)
    gen.generate()
    assert os.path.exists(os.path.join(output_dir, "my_output.py"))


# ---------- Tests: Associations / Relationships ----------

def test_generate_with_association(output_dir):
    """Generator should render associations correctly."""
    name_prop = Property(name="name", type=StringType)
    title_prop = Property(name="title", type=StringType)
    author = Class(name="Author", attributes={name_prop})
    book = Class(name="Book", attributes={title_prop})
    writes = BinaryAssociation(
        name="writes",
        ends={
            Property(name="author", type=author, multiplicity=Multiplicity(1, 1)),
            Property(name="books", type=book, multiplicity=Multiplicity(0, 9999)),
        },
    )
    model = DomainModel(
        name="LibraryModel",
        types={author, book},
        associations={writes},
    )
    gen = MyGenerator(model=model, output_dir=output_dir)
    gen.generate()
    with open(os.path.join(output_dir, "my_output.py")) as f:
        content = f.read()
    assert "Author" in content
    assert "Book" in content
```

## What to Test

For generators, the BESSER project expects you to cover:

1. **Output file existence** -- verify the expected files are actually created.
2. **Key content in output** -- check that class names, attribute names, and relationship mappings appear in the generated code.
3. **Edge cases** -- empty models, deep inheritance hierarchies, many-to-many associations.
4. **Importable output (if applicable)** -- if your generator produces Python code, use `importlib` to dynamically import the generated module and verify it is syntactically valid.

## Running Your Tests

```bash
# Run just your generator's tests
python -m pytest tests/generators/my_generator/

# Run with verbose output
python -m pytest -v tests/generators/my_generator/

# Stop on first failure
python -m pytest -x tests/generators/my_generator/

# Run by keyword match
python -m pytest -k "test_my_generator"
```

## Summary

| Question | Answer |
|----------|--------|
| Where do tests go? | `tests/generators/<name>/test_<name>.py` |
| How to set up fixtures? | Inline `@pytest.fixture` functions that build small `DomainModel` instances directly |
| `tmp_path` or manual dirs? | Use `tmp_path` -- it is auto-cleaned by pytest and is the standard across the codebase |
| What to assert? | Both file existence and specific content (class names, attributes, relationships) |
| Naming conventions? | Files: `test_*.py`, functions: `test_*` |
