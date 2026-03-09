# ImportError When Importing `String` from BESSER

## The Problem

The import `from besser.BUML.metamodel.structural import String` fails because there is no export called `String` in BESSER's structural metamodel. This is a common naming mistake.

## The Fix

In BESSER, the primitive data types use a `Type` suffix. The correct name is **`StringType`**, not `String`. This applies to all primitive types:

| Wrong (will fail)  | Correct              |
|--------------------|----------------------|
| `String`           | `StringType`         |
| `Integer`          | `IntegerType`        |
| `Float`            | `FloatType`          |
| `Boolean`          | `BooleanType`        |
| `Date`             | `DateType`           |
| `DateTime`         | `DateTimeType`       |

These are **pre-built singleton instances**, not classes. You import them directly and use them as-is when defining attributes.

## Corrected Import

```python
from besser.BUML.metamodel.structural import StringType
```

Or, if you need multiple primitive types:

```python
from besser.BUML.metamodel.structural import (
    StringType, IntegerType, FloatType, BooleanType, DateType, DateTimeType
)
```

## Full Working Example

Here is a complete example showing correct usage of primitive types when building a domain model:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)

# Define classes with attributes using the correct type names
person = Class(name="Person", attributes={
    Property(name="name", type=StringType),
    Property(name="age", type=IntegerType),
})

address = Class(name="Address", attributes={
    Property(name="street", type=StringType),
    Property(name="city", type=StringType),
})

# Define an association between Person and Address
lives_at = BinaryAssociation(name="lives_at", ends={
    Property(name="person_end", type=person, multiplicity=Multiplicity(1, 1)),
    Property(name="address_end", type=address, multiplicity=Multiplicity(0, "*")),
})

# Build the domain model
model = DomainModel(name="example", types={person, address}, associations={lives_at})

print("Model created successfully!")
print("Classes:", [c.name for c in model.get_classes()])
```

## Quick Verification

After fixing your import, you can verify everything is working with this one-liner:

```bash
python -c "from besser.BUML.metamodel.structural import StringType; print('Import OK:', StringType)"
```

## Other Common Import Pitfall

If you are working with associations, note that the correct name for a two-ended association is `BinaryAssociation`, not `Association`:

```python
# Wrong:
from besser.BUML.metamodel.structural import Association

# Correct:
from besser.BUML.metamodel.structural import BinaryAssociation
```

## Still Getting `ModuleNotFoundError: No module named 'besser'`?

If the error is about the `besser` module itself not being found (rather than a specific name within it), check these:

1. **Ensure installation succeeded**: Run `pip install -e .` from the repo root where `pyproject.toml` lives.
2. **Activate your virtual environment first**: On Windows use `venv\Scripts\activate` (note the capital S in `Scripts`).
3. **Match pip and python**: Use `python -m pip install -e .` to ensure pip installs into the same Python that you are running.
