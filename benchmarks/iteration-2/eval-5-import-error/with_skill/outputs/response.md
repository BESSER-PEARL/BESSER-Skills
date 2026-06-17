The error is exactly what the skill flags as the single most common BESSER import mistake. Here is the complete answer.

## What's wrong

`String` does not exist in `besser.BUML.metamodel.structural`. The primitive type is named `StringType`. BESSER's primitive types are **singleton instances** (not classes), and every one of them ends in `Type`.

So this line is the bug:

```python
from besser.BUML.metamodel.structural import String   # ImportError
```

## The fix

```python
from besser.BUML.metamodel.structural import StringType
```

This is the same pattern for every primitive type. The common wrong-vs-correct names:

| Wrong | Correct |
|-------|---------|
| `String` | `StringType` |
| `Integer` | `IntegerType` |
| `Float` | `FloatType` |
| `Boolean` | `BooleanType` |
| `Date` | `DateType` |
| `DateTime` | `DateTimeType` |

(Also note: `Association` for a two-end association should be `BinaryAssociation`.)

Because these are singleton instances, you import and use them directly — you do not instantiate them. For example:

```python
from besser.BUML.metamodel.structural import Class, Property, StringType, IntegerType

person = Class(name="Person", attributes={
    Property(name="name", type=StringType),
    Property(name="age", type=IntegerType),
})
```

## Important: your install is almost certainly fine

The phrasing ("I just cloned and ran `pip install -e .`") suggests you might suspect a broken installation, but this is purely a wrong-name import error, not an install problem. You can confirm BESSER itself imported correctly:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"
```

If that prints `OK`, your `pip install -e .` worked and only the symbol name in your import was wrong. (If instead you got `ModuleNotFoundError: No module named 'besser'`, that would be a different problem — typically not in the repo root when running `pip install -e .`, or a wrong/unactivated virtual environment; in that case run `python -m pip install -e .` from the repo root with your venv active.)

## If you genuinely need to construct a primitive type by name

The pre-built singletons (`StringType`, `IntegerType`, etc.) are the recommended approach. If you ever build one explicitly via `PrimitiveDataType(name)`, the valid name strings are: `int`, `float`, `str`, `bool`, `time`, `date`, `datetime`, `timedelta`, `any`.