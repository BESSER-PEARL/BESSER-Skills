Both of your issues are name-validation rules in BESSER's B-UML metamodel. Here's how to fix each.

## Issue 1: Spaces in a property name

BESSER does not allow spaces in any B-UML name (classes, attributes/properties, associations, enumerations, etc.). The error you're seeing is:

```
ValueError: 'first name' is invalid. Name cannot contain spaces.
```

Fix: replace the space with an underscore.

```python
from besser.BUML.metamodel.structural import Property, StringType

# Wrong - raises ValueError
# name_prop = Property(name="first name", type=StringType)

# Correct
name_prop = Property(name="first_name", type=StringType)
```

Note the same rule applies to hyphens (`first-name` also fails with a "Hyphens are not allowed; use '_' instead" ValueError), so always use underscores as your separator.

## Issue 2: Two enum literals both named 'ACTIVE'

Each `EnumerationLiteral` inside an `Enumeration` must have a unique name. Two literals named `ACTIVE` triggers:

```
ValueError: An enumeration cannot have two literals with the same name
```

Fix: give each literal a distinct name. If they're genuinely meant to be different states, rename them; if it was an accidental duplicate, remove one.

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# Wrong - two literals named ACTIVE
# status = Enumeration(name="Status", literals={
#     EnumerationLiteral(name="ACTIVE"),
#     EnumerationLiteral(name="ACTIVE"),
#     EnumerationLiteral(name="INACTIVE"),
# })

# Correct - unique literal names
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
})
```

You can verify the literals on an enumeration with:

```python
print([lit.name for lit in status.literals])
```

## Putting it together

```python
from besser.BUML.metamodel.structural import (
    Class, Property, StringType,
    Enumeration, EnumerationLiteral,
)

# Fixed enumeration: unique literal names
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
})

# Fixed property: underscore instead of space
person = Class(name="Person", attributes={
    Property(name="first_name", type=StringType),
    Property(name="status", type=status),
})
```

Both errors come from the same family of metamodel naming/uniqueness rules, so as a general guideline: use underscores (no spaces, no hyphens) for every name, and keep names unique within their scope (literals unique within an enumeration; class/enumeration/datatype names unique within the `DomainModel`).