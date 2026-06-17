The skill's SKILL.md fully covers generator testing (Step 5 plus the Testing section). Here is the complete answer based on it.

# Testing a New BESSER Generator

## Typical test pattern

Tests live under `tests/generators/<your_generator>/test_<your_generator>.py` (file name `test_*.py`, functions `test_*`). Each test builds a small in-memory domain model, runs the generator into a temp output dir, then asserts on both **file existence** and **file content**.

```python
import os
import shutil
import pytest
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)
from besser.generators.my_generator import MyGenerator


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


def test_generate_creates_output_file(domain_model, output_dir):
    gen = MyGenerator(model=domain_model, output_dir=output_dir)
    gen.generate()
    assert os.path.exists(os.path.join(output_dir, "my_output.py"))


def test_generate_contains_class_name(domain_model, output_dir):
    gen = MyGenerator(model=domain_model, output_dir=output_dir)
    gen.generate()
    with open(os.path.join(output_dir, "my_output.py")) as f:
        content = f.read()
    assert "Person" in content
    assert "name" in content
    assert "age" in content
```

## Setting up fixtures for domain models

Build **small inline models as `@pytest.fixture`** — do not depend on external `.py` model files or load anything from disk. The pattern is:

1. Create `Property` objects with a `name` and a `type` (e.g. `StringType`, `IntegerType`).
2. Group them into a `Class` via `attributes={...}` (a set).
3. Wrap your classes in a `DomainModel(name=..., types={...})`.

For relationships, add a `BinaryAssociation` with `Multiplicity` ends. Cover edge cases in additional fixtures/tests: empty model, deep inheritance, and many-to-many associations.

If your generator emits **importable Python**, use `importlib` to dynamically import the generated file and assert on the actual objects/behavior, not just the text.

## tmp_path vs. manual directories

**Use `tmp_path` (or `tmpdir`).** It is the preferred convention because pytest auto-creates and auto-cleans it — no leftover files between runs. Wrap it in an `output_dir` fixture returning `str(tmp_path)` so it plugs straight into the generator's `output_dir` argument (as shown above).

Only fall back to manually creating directories when you genuinely cannot use the pytest temp fixtures — and in that case clean up with `shutil.rmtree()` inside a `try/finally` block so temp dirs and file handles are always released.

## What to assert (generator-specific guidance)

- Output file(s) exist.
- Key content is present: class names, attribute names, relationship/multiplicity mappings, and other business logic — assert specific content, not just existence.
- Edge cases: empty model, deep inheritance, many-to-many.
- Error conditions: use `pytest.raises(ValueError)` for invalid input.
- **Determinism**: the same model must produce identical output every run (no timestamps/random values) — worth a test if your generator is at risk of nondeterminism.

## Running your tests

```bash
# Just your generator's directory
python -m pytest tests/generators/my_generator/

# By keyword
python -m pytest -k "my_generator"

# Verbose / stop on first failure
python -m pytest -v -x tests/generators/my_generator/
```

Note: BESSER has **no CI test workflow** — tests are expected to run locally before opening a PR, so run `python -m pytest` before pushing.

Source for all of the above: `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-dev/SKILL.md` (Step 5 "Write Tests", lines 173-223; and the "Testing" section, lines 328-369).