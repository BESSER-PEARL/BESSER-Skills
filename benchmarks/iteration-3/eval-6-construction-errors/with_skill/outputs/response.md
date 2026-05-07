Both errors are standard B-UML naming/uniqueness rules. Here is what's happening and how to fix each one.

## Issue 1: Spaces in names

BESSER raises:

```
ValueError: 'first name' is invalid. Name cannot contain spaces.
```

All B-UML names — classes, attributes, associations, enumerations, literals — must not contain spaces. Replace the space with an underscore.

```python
from besser.BUML.metamodel.structural import Class, Property, StringType

# Wrong: space in attribute name
# Property(name="first name", type=StringType)

# Correct:
Property(name="first_name", type=StringType)
```

Note: hyphens are also rejected with a similar error (`Hyphens are not allowed; use '_' instead.`), so stick to underscores for multi-word names.

## Issue 2: Duplicate enum literals

BESSER raises:

```
ValueError: An enumeration cannot have two literals with the same name
```

Each `EnumerationLiteral` inside a single `Enumeration` must have a unique name. You can't have two literals both called `ACTIVE` in the same enum — pick distinct names.

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# Wrong: two literals named "ACTIVE"
# status = Enumeration(name="Status", literals={
#     EnumerationLiteral(name="ACTIVE"),
#     EnumerationLiteral(name="ACTIVE"),
# })

# Correct: give each literal a unique name
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
})
```

If the duplicate was a copy-paste mistake, just remove the second one. If you genuinely meant two distinct active states, distinguish them (`ACTIVE_PENDING`, `ACTIVE_VERIFIED`, etc.).

## Putting it together

```python
from besser.BUML.metamodel.structural import (
    Class, Property, StringType,
    Enumeration, EnumerationLiteral,
)

# Fixed attribute name (underscore, not space)
person = Class(name="Person", attributes={
    Property(name="first_name", type=StringType),
})

# Fixed enum (unique literal names)
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
})
```

## Related gotchas to watch for

While you're cleaning up names, the same rules apply to other duplicate-name errors you may hit next:

- `ValueError: The model cannot have types with duplicate names` — every class, enumeration, and data type in your `DomainModel` must have a unique name.
- `ValueError: A class cannot have more than one attribute marked as 'id'` — only one `Property` per class can have `is_id=True`.

Calling `model.validate()` after building your model is a good habit to catch these structural issues before you hand it to a generator.
