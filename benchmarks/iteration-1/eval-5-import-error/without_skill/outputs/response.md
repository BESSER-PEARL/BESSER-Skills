# ImportError: Cannot Import `String` from `besser.BUML.metamodel.structural`

## The Problem

The import fails because there is **no symbol named `String`** in the BESSER structural metamodel. The correct name is **`StringType`**.

```python
# This FAILS with ImportError
from besser.BUML.metamodel.structural import String

# This is CORRECT
from besser.BUML.metamodel.structural import StringType
```

## Why This Happens

In BESSER, primitive data types are pre-instantiated objects of the `PrimitiveDataType` class, and they all follow the naming convention `<Name>Type`. They are defined in `besser/BUML/metamodel/structural/structural.py` (lines 291-299):

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
```

There is also a general-purpose type:

```python
AnyType = DataType("any")
```

These are **singleton instances**, not classes. When you use `StringType` as a type for a `Property`, you are passing this pre-built object directly.

## Complete List of Available Primitive Data Types

| Import Name      | Underlying Value | Use Case                     |
|------------------|------------------|------------------------------|
| `StringType`     | `"str"`          | Text / string attributes     |
| `IntegerType`    | `"int"`          | Whole number attributes      |
| `FloatType`      | `"float"`        | Decimal number attributes    |
| `BooleanType`    | `"bool"`         | True/false attributes        |
| `DateType`       | `"date"`         | Date-only attributes         |
| `TimeType`       | `"time"`         | Time-only attributes         |
| `DateTimeType`   | `"datetime"`     | Date and time attributes     |
| `TimeDeltaType`  | `"timedelta"`    | Duration / time delta values |
| `AnyType`        | `"any"`          | Generic / untyped attributes |

## Corrected Working Example

Here is a complete working example based on the classic Library model from the BESSER documentation:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType, DateType
)

# Book attributes and class
title: Property = Property(name="title", type=StringType)
pages: Property = Property(name="pages", type=IntegerType)
release: Property = Property(name="release", type=DateType)
book: Class = Class(name="Book", attributes={title, pages, release})

# Author attributes and class
author_name: Property = Property(name="name", type=StringType)
email: Property = Property(name="email", type=StringType)
author: Class = Class(name="Author", attributes={author_name, email})

# Library attributes and class
library_name: Property = Property(name="name", type=StringType)
address: Property = Property(name="address", type=StringType)
library: Class = Class(name="Library", attributes={library_name, address})

# Associations
located_in: Property = Property(name="locatedIn", type=library, multiplicity=Multiplicity(1, 1))
has: Property = Property(name="has", type=book, multiplicity=Multiplicity(0, "*"))
lib_book_association: BinaryAssociation = BinaryAssociation(
    name="lib_book_assoc", ends={located_in, has}
)

written_by: Property = Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, "*"))
publishes: Property = Property(name="publishes", type=book, multiplicity=Multiplicity(0, "*"))
book_author_association: BinaryAssociation = BinaryAssociation(
    name="book_author_assoc", ends={written_by, publishes}
)

# Domain model
library_model: DomainModel = DomainModel(
    name="Library_model",
    types={library, book, author},
    associations={lib_book_association, book_author_association}
)

print(f"Model '{library_model.name}' created with {len(library_model.types)} types.")
```

## Quick Verification

You can verify your installation is working correctly by running:

```bash
python -c "from besser.BUML.metamodel.structural import StringType, IntegerType; print('Import successful:', StringType, IntegerType)"
```

Expected output:

```
Import successful: PrimitiveDataType(str, ...) PrimitiveDataType(int, ...)
```

## Summary

The fix is straightforward: replace `String` with `StringType` in your import statement. All primitive data types in BESSER use the `<Name>Type` naming convention (`StringType`, `IntegerType`, `FloatType`, `BooleanType`, `DateType`, `TimeType`, `DateTimeType`, `TimeDeltaType`). These are pre-instantiated `PrimitiveDataType` objects, not classes, so you use them directly as type arguments when defining `Property` instances.
