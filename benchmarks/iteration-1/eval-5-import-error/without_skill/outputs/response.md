# Diagnosing the `ImportError` for `String` from `besser.BUML.metamodel.structural`

## The Problem

The import statement:

```python
from besser.BUML.metamodel.structural import String
```

fails with an `ImportError` because **there is no symbol called `String`** in the `besser.BUML.metamodel.structural` package. The correct name is **`StringType`**.

## Root Cause

In the BESSER codebase, primitive data types are defined as module-level instances of `PrimitiveDataType` in the file:

**`C:\Users\sulejmani\Desktop\BESSER\besser\BUML\metamodel\structural\structural.py`** (lines 291-302)

```python
# Define instances of PrimitiveDataType
StringType = PrimitiveDataType("str")
IntegerType = PrimitiveDataType("int")
FloatType = PrimitiveDataType("float")
BooleanType = PrimitiveDataType("bool")
TimeType = PrimitiveDataType("time")
DateType = PrimitiveDataType("date")
DateTimeType = PrimitiveDataType("datetime")
TimeDeltaType = PrimitiveDataType("timedelta")
AnyType = DataType("any")
```

The naming convention uses the suffix `Type` (e.g., `StringType`, `IntegerType`, `FloatType`), not bare names like `String`, `Integer`, or `Float`. There is no alias called `String` anywhere in the module.

The `__init__.py` at `besser/BUML/metamodel/structural/__init__.py` simply re-exports everything via:

```python
from .structural import *
```

So all the `*Type` names are available, but `String` is not -- because it was never defined.

## The Fix

Change your import to use `StringType` instead of `String`:

```python
from besser.BUML.metamodel.structural import StringType
```

## Complete List of Available Primitive Types

Here are all the primitive/data type instances you can import:

| Import Name      | Underlying Value | Description           |
|------------------|------------------|-----------------------|
| `StringType`     | `"str"`          | String data type      |
| `IntegerType`    | `"int"`          | Integer data type     |
| `FloatType`      | `"float"`        | Float data type       |
| `BooleanType`    | `"bool"`         | Boolean data type     |
| `TimeType`       | `"time"`         | Time data type        |
| `DateType`       | `"date"`         | Date data type        |
| `DateTimeType`   | `"datetime"`     | DateTime data type    |
| `TimeDeltaType`  | `"timedelta"`    | TimeDelta data type   |
| `AnyType`        | `"any"`          | Generic/any data type |

## Usage Example

Here is the canonical usage pattern, taken from the library example at `tests/BUML/metamodel/structural/library/library.py`:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property,
    Multiplicity, BinaryAssociation,
    StringType, IntegerType, DateType
)

# Define properties using the correct type names
library_name: Property = Property(name="name", type=StringType)
address: Property = Property(name="address", type=StringType)
title: Property = Property(name="title", type=StringType)
pages: Property = Property(name="pages", type=IntegerType)
release: Property = Property(name="release", type=DateType)
```

## Alternative: Pass Type as a String

As a convenience, you can also pass the type as a plain string when constructing a `Property`. The `TypedElement` class (the parent of `Property`) has a built-in `type_mapping` dictionary (defined at line 458 of `structural.py`) that resolves string names to the corresponding `PrimitiveDataType` instance:

```python
# This also works -- "str" is resolved to StringType automatically
name_prop = Property(name="name", type="str")
```

The supported string values are: `"str"`, `"string"`, `"int"`, `"float"`, `"bool"`, `"time"`, `"date"`, `"datetime"`, `"timedelta"`.

## Summary

The `ImportError` occurs because the symbol is named `StringType`, not `String`. This naming convention (`*Type` suffix) applies to all primitive data types in BESSER. Replace `String` with `StringType` in your import statement and the error will be resolved.
