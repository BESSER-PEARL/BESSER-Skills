# ImportError on `String` from `besser.BUML.metamodel.structural`

Good news: your install is almost certainly fine — you just hit one of the most common naming gotchas in BESSER. The primitive type is named **`StringType`**, not `String`.

## The fix

Change your import to use the `*Type` suffix:

```python
# Wrong (this is what's failing):
from besser.BUML.metamodel.structural import String

# Correct:
from besser.BUML.metamodel.structural import StringType
```

The same applies to all the other primitive types. Here's the full mapping in case you have related imports nearby:

| Wrong | Correct |
|-------|---------|
| `String` | `StringType` |
| `Integer` | `IntegerType` |
| `Float` | `FloatType` |
| `Boolean` | `BooleanType` |
| `Date` | `DateType` |
| `DateTime` | `DateTimeType` |
| `Association` | `BinaryAssociation` (for 2-end associations) |

## Why it's named that way

The primitive types in BESSER are **singleton instances**, not classes. You import them and use them directly — for example, when defining a `Property`:

```python
from besser.BUML.metamodel.structural import Class, Property, StringType, IntegerType

User = Class(name="User", attributes={
    Property(name="name", type=StringType),
    Property(name="age", type=IntegerType),
})
```

No `StringType()` call — just pass the singleton itself as the `type=`.

## Verifying your install is otherwise healthy

Once you've fixed the name, sanity-check the install with this one-liner:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel, StringType; print('OK')"
```

If that prints `OK`, you're good to go.

## If you still get an ImportError after the rename

A real install problem would more typically show up as `ModuleNotFoundError: No module named 'besser'` rather than a name-level `ImportError`. But if `StringType` itself fails to import, double-check:

1. **You're in the right virtual environment** — `where python` (Windows) / `which python` (Unix) should point inside your venv. `pip install -e .` only installs into the active environment.
2. **You ran `pip install -e .` from the repo root** (where `pyproject.toml` lives), not from a subdirectory.
3. **`pip` and `python` agree** — if you have multiple Python installs, run `python -m pip install -e .` instead of bare `pip install -e .` to guarantee they target the same interpreter.
4. **Python version is 3.10+** — BESSER requires it (Django 5.x dependency). Check with `python --version`.

But for the symptom you described (`from besser.BUML.metamodel.structural import String` → ImportError), the rename to `StringType` is virtually guaranteed to be the fix.
