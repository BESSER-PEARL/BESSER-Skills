Both `Property` (attribute) and `EnumerationLiteral` ultimately extend `NamedElement`, so the name validation in the `name.setter` applies to both. I have everything I need.

---

## Both errors come from BESSER's `NamedElement` name validation rules

In BESSER's structural metamodel, every named thing — classes, attributes (`Property`), enumerations, and enumeration literals — inherits from `NamedElement`. Its `name` setter enforces strict rules, and enumerations additionally reject duplicate literal names. Here is the exact source.

### Issue 1: "Name cannot contain spaces"

From `besser/BUML/metamodel/structural/structural.py` (the `NamedElement.name` setter, lines 182-192):

```python
if name is None:
    raise ValueError("Name cannot be None.")
if not isinstance(name, str) or name.strip() == "":
    raise ValueError("Name cannot be empty or whitespace-only.")
if ' ' in name:
    raise ValueError(f"'{name}' is invalid. Name cannot contain spaces.")
if '-' in name:
    raise ValueError(f"'{name}' is invalid. Hyphens are not allowed; use '_' instead.")
if keyword.iskeyword(name):
    logging.warning(f"'{name}' is a Python keyword. ...")
```

So `'first name'` is rejected purely because of the space. There is **no flag to disable this** — names must be valid identifiers (no spaces, no hyphens). Note hyphens are also banned, and Python keywords only produce a warning (not an error).

**Fix:** use an underscore (or camelCase) instead of a space:

```python
from besser.BUML.metamodel.structural import Property, StringType

first_name = Property(name="first_name", type=StringType)   # OK
# or: Property(name="firstName", type=StringType)
```

If you want a human-readable label like "first name" preserved somewhere, BESSER doesn't store it on the name itself — you'd attach it via `Metadata` (e.g. the `description` field) while keeping the machine name underscore-separated.

### Issue 2: "An enumeration cannot have two literals with the same name"

From the same file, the `Enumeration.literals` setter (lines 412-415) and `add_literal` (lines 429-431):

```python
names = [literal.name for literal in literals]
if len(names) != len(set(names)):
    raise ValueError("An enumeration cannot have two literals with the same name")
...
# add_literal:
if literal.name in [literal.name for literal in self.literals]:
    raise ValueError(f"An enumeration cannot have two literals with the same name: '{literal.name}'")
```

Literal names within one enumeration must be unique. Two literals both named `'ACTIVE'` collide. (Also note `literals` is a `set`, so even if the names differed but two `EnumerationLiteral` objects were considered equal, only one would survive — but the explicit check above is what raises your error.)

**Fix:** give each literal a distinct name:

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

status = Enumeration(
    name="Status",
    literals={
        EnumerationLiteral(name="ACTIVE"),
        EnumerationLiteral(name="INACTIVE"),   # was the duplicate 'ACTIVE'
    },
)
```

If the second `ACTIVE` was a copy-paste mistake, just remove it. If you genuinely need two different states that you were both calling `ACTIVE`, rename one (e.g. `ACTIVE` and `RE_ACTIVATED`).

### Putting it together — a valid class

```python
from besser.BUML.metamodel.structural import (
    Class, Property, StringType,
    Enumeration, EnumerationLiteral,
)

status = Enumeration(
    name="Status",
    literals={EnumerationLiteral(name="ACTIVE"), EnumerationLiteral(name="INACTIVE")},
)

person = Class(
    name="Person",
    attributes={
        Property(name="first_name", type=StringType),   # no space
        Property(name="status", type=status),
    },
)
```

### Summary
- **Spaces (and hyphens):** not configurable — rename `'first name'` to `first_name` / `firstName`. Source: `NamedElement.name` setter, `structural.py` lines 186-189.
- **Duplicate enum literals:** rename or remove the second `'ACTIVE'`; literal names must be unique within an enumeration. Source: `Enumeration.literals` setter / `add_literal`, `structural.py` lines 414-415 and 430-431.

File reference: `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/BUML/metamodel/structural/structural.py`