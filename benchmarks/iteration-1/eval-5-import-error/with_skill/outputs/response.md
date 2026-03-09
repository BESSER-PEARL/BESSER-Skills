# Diagnosis: `ImportError` When Importing `String` from `besser.BUML.metamodel.structural`

## The Problem

The name `String` does not exist in the `besser.BUML.metamodel.structural` module. That is why you get an `ImportError` when you run:

```python
from besser.BUML.metamodel.structural import String
```

## Root Cause

In BESSER, the primitive data types are **pre-built singleton instances** of `PrimitiveDataType`, and they use the naming convention `<Type>Type` -- not bare names like `String`, `Integer`, etc.

The correct name for the string type is **`StringType`**, not `String`.

Here is the relevant code from `besser/BUML/metamodel/structural/structural.py` (around line 292):

```python
StringType = PrimitiveDataType("str")
IntegerType = PrimitiveDataType("int")
FloatType = PrimitiveDataType("float")
BooleanType = PrimitiveDataType("bool")
TimeType = PrimitiveDataType("time")
DateType = PrimitiveDataType("date")
DateTimeType = PrimitiveDataType("datetime")
```

## The Fix

Change your import to use the correct name:

```python
from besser.BUML.metamodel.structural import StringType
```

## Full Reference: Common Naming Mistakes

| Wrong (will cause ImportError) | Correct |
|-------------------------------|---------|
| `String` | `StringType` |
| `Integer` | `IntegerType` |
| `Float` | `FloatType` |
| `Boolean` | `BooleanType` |
| `Date` | `DateType` |
| `DateTime` | `DateTimeType` |
| `Time` | `TimeType` |
| `Association` | `BinaryAssociation` (for two-ended associations) |

## Example: Correct Usage

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, BinaryAssociation,
    Multiplicity, StringType, IntegerType,
)

# Use StringType (not String) when defining attribute types
name_attr = Property(name="name", type=StringType)
age_attr = Property(name="age", type=IntegerType)
person = Class(name="Person", attributes={name_attr, age_attr})
model = DomainModel(name="example", types={person})
```

## Quick Verification

After fixing the import, you can verify that BESSER is properly installed and importable by running:

```bash
python -c "from besser.BUML.metamodel.structural import StringType; print('OK')"
```

If that prints `OK`, your installation is working correctly.

## Additional Notes

- If you are also getting `ModuleNotFoundError: No module named 'besser'` (i.e., the entire `besser` package is not found, not just `String`), then the issue is with your installation or virtual environment. Make sure you activated your venv (`venv\Scripts\activate` on Windows, `source venv/bin/activate` on Unix) before running `pip install -e .`, and that you are running your script with the same Python interpreter.
- BESSER requires **Python 3.10 or later**. Verify with `python --version`.
