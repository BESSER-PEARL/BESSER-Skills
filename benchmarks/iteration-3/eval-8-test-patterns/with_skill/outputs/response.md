# Testing a New Generator in BESSER

Here's the typical test pattern used across the BESSER codebase, plus answers to your fixture questions.

## Where tests live

Place generator tests under:

```
tests/generators/my_generator/test_my_generator.py
```

Test discovery uses pytest defaults:
- Test files: `test_*.py`
- Test functions: `test_*`

## The standard pattern

The codebase uses two fixtures (one for the model, one for the output directory) and then small focused tests that assert both file existence and content:

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

## Domain model fixtures

Build the model **inline as a `@pytest.fixture`** — don't load from external files. This keeps tests:

- Self-contained (no fixture file drift)
- Fast
- Easy to read (the test asserts content you can see in the same file)

If you need different shapes (empty model, deep inheritance, many-to-many), make additional fixtures rather than parametrizing one giant model. Common shapes worth covering:

- A minimal model (one class, a couple of attributes)
- An association (try a `BinaryAssociation` with various `Multiplicity` settings)
- Inheritance
- Edge cases that are specific to your generator (e.g., enums, abstract classes)

## `tmp_path` vs manual directories

**Use `tmp_path` (or `tmpdir`).** That's the project-wide convention. Reasons:

- Auto-cleaned by pytest after the test, even on failure
- Isolated per test — no cross-test pollution
- No need for `setup`/`teardown` boilerplate

The pattern is exactly the wrapper above:

```python
@pytest.fixture
def output_dir(tmp_path):
    return str(tmp_path)
```

(`tmp_path` is a `pathlib.Path`; the `str()` cast matches what generators expect for `output_dir`.)

Only fall back to manual directories + `shutil.rmtree()` if you genuinely cannot use the pytest tmp fixtures (rare). When you do, wrap cleanup in `try/finally` — leaking temp dirs is one of the listed contribution pitfalls.

## What to assert

Per the project's testing conventions, cover both:

1. **Structure** — output files exist at expected paths, with expected names.
2. **Content** — class names, attribute names, relationship mappings, and any business logic specific to your generator. Don't stop at "file exists."

Other useful tactics:

- **Importable Python output?** Use `importlib` to dynamically import the generated module and exercise it. This catches syntax errors and runtime issues the templates might produce.
- **Determinism check** — generate twice into two different `tmp_path` dirs and assert the outputs are byte-equal. Generators must be deterministic (no timestamps, no random ordering).
- **Error paths** — use `pytest.raises(ValueError)` for invalid inputs.
- **Edge cases** — empty model, deep inheritance, many-to-many, recursive associations.

## Running your tests

```bash
# Just your generator
python -m pytest tests/generators/my_generator/

# By keyword
python -m pytest -k "my_generator"

# Verbose, stop on first failure
python -m pytest -v -x tests/generators/my_generator/
```

Note: BESSER has no CI workflow for tests — you're expected to run `python -m pytest` locally before opening the PR. Don't skip it.

## Quick checklist

- [ ] Tests under `tests/generators/<your_generator>/test_*.py`
- [ ] `domain_model` fixture built inline (no external files)
- [ ] `output_dir` fixture wrapping `tmp_path`
- [ ] At least one "file exists" test
- [ ] At least one "content is correct" test
- [ ] Edge case tests (empty model, associations, inheritance as relevant)
- [ ] Determinism asserted if your generator has any ordering risk
- [ ] `pytest.raises(...)` for any documented error conditions
