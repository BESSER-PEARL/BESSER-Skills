Confirmed. The `ImportError` has nothing to do with your install — `pip install -e .` worked fine. The problem is that **`String` simply does not exist** in BESSER's structural metamodel. The name is wrong.

## What's actually wrong

In BESSER (v7.8.3), primitive types are not exposed as `String`, `Integer`, etc. They are predefined **instances** of the `PrimitiveDataType` class, and each one has a `Type` suffix. From `besser/BUML/metamodel/structural/structural.py` (lines 299-307):

```python
StringType   = PrimitiveDataType("str")
IntegerType  = PrimitiveDataType("int")
FloatType    = PrimitiveDataType("float")
BooleanType  = PrimitiveDataType("bool")
TimeType     = PrimitiveDataType("time")
DateType     = PrimitiveDataType("date")
DateTimeType = PrimitiveDataType("datetime")
TimeDeltaType = PrimitiveDataType("timedelta")
AnyType      = PrimitiveDataType("any")
```

So there is no symbol named `String` to import — hence the `ImportError`. The correct name is `StringType`.

## The fix

```python
from besser.BUML.metamodel.structural import StringType
```

Or import several at once:

```python
from besser.BUML.metamodel.structural import (
    StringType, IntegerType, FloatType, BooleanType,
    DateType, DateTimeType, TimeType, TimeDeltaType,
)
```

These imports work because `besser/BUML/metamodel/structural/__init__.py` does `from .structural import *`, re-exporting everything (there is no `__all__` restricting it).

## How to actually use it

`StringType` is an *instance* you pass as the `type` of a `Property` (attribute), not a class you instantiate:

```python
from besser.BUML.metamodel.structural import (
    Class, Property, StringType, IntegerType,
)

# Define attributes with their types
name_attr = Property(name="name", type=StringType)
age_attr  = Property(name="age",  type=IntegerType)

# Attach them to a class
person = Class(name="Person", attributes={name_attr, age_attr})
```

## Bonus: you can also just use the string name

There's a convenience shortcut. `TypedElement` has a `type_mapping` (lines 469-479) that resolves plain strings to the right primitive instance, so this also works without importing the type at all:

```python
from besser.BUML.metamodel.structural import Property

# "str" / "string" are mapped to StringType automatically
name_attr = Property(name="name", type="str")
```

Accepted string keys: `"str"`/`"string"`, `"int"`, `"float"`, `"bool"`, `"time"`, `"date"`, `"datetime"`, `"timedelta"`.

## Summary

- Your install is fine; the import name is wrong.
- There is no `String` — use `StringType` (and `IntegerType`, `FloatType`, `BooleanType`, `DateType`, `DateTimeType`, `TimeType`, `TimeDeltaType`, `AnyType`).
- They are ready-made instances of `PrimitiveDataType`, used as the `type=` argument of a `Property`, or you can pass the string `"str"` directly.

Source verified in: `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/BUML/metamodel/structural/structural.py` (lines 264-308, 469-484).