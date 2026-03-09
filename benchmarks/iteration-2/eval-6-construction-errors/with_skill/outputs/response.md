# Fixing Construction Errors: Spaces in Names and Duplicate Enumeration Literals

You are hitting two distinct model construction validation errors. Here is how to fix both.

---

## Issue 1: Spaces in Property Names

**Error:** `ValueError: 'first name' is invalid. Name cannot contain spaces.`

All B-UML names -- classes, attributes (properties), associations, and so on -- must not contain spaces. The fix is to replace spaces with underscores.

**Broken code:**

```python
from besser.BUML.metamodel.structural import Class, Property, StringType

# This will raise a ValueError because of the space in "first name"
person = Class(name="Person", attributes={
    Property(name="first name", type=StringType)
})
```

**Fixed code:**

```python
from besser.BUML.metamodel.structural import Class, Property, StringType

# Use underscores instead of spaces
person = Class(name="Person", attributes={
    Property(name="first_name", type=StringType)
})
```

The same rule applies to all named elements. For example, a class named `"My Class"` must become `"My_Class"`, and an association named `"has member"` must become `"has_member"`.

Note: Hyphens are also not allowed (e.g., `"first-name"` will raise `ValueError: 'first-name' is invalid. Hyphens are not allowed; use '_' instead.`). Always use underscores as the word separator.

---

## Issue 2: Duplicate Enumeration Literals

**Error:** `ValueError: An enumeration cannot have two literals with the same name`

Each `EnumerationLiteral` within an `Enumeration` must have a unique name. If you have two literals both called `"ACTIVE"`, you need to rename one of them so they are distinct.

**Broken code:**

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# This will raise a ValueError because both literals are named "ACTIVE"
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE")
})
```

**Fixed code:**

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# Give each literal a unique name
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
    EnumerationLiteral(name="SUSPENDED")
})
```

If you truly need to distinguish between two kinds of "active" states, use more descriptive names:

```python
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE_REGULAR"),
    EnumerationLiteral(name="ACTIVE_TRIAL"),
    EnumerationLiteral(name="INACTIVE")
})
```

---

## Full Working Example with Both Fixes Applied

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Enumeration, EnumerationLiteral,
    BinaryAssociation, Multiplicity, StringType, IntegerType,
)

# Enumeration with unique literal names
status_enum = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
    EnumerationLiteral(name="SUSPENDED"),
})

# Class with underscore-separated property names (no spaces)
person = Class(name="Person", attributes={
    Property(name="first_name", type=StringType),
    Property(name="last_name", type=StringType),
    Property(name="age", type=IntegerType),
    Property(name="status", type=status_enum),
})

account = Class(name="Account", attributes={
    Property(name="account_id", type=IntegerType, is_id=True),
    Property(name="account_name", type=StringType),
})

owns = BinaryAssociation(name="owns", ends={
    Property(name="owner", type=person, multiplicity=Multiplicity(1, 1)),
    Property(name="accounts", type=account, multiplicity=Multiplicity(0, "*")),
})

model = DomainModel(name="MyModel", types={person, account, status_enum}, associations={owns})

# Validate the model to catch any remaining structural issues
model.validate()
print("Model is valid!")
```

---

## Summary of Rules

| Rule | Invalid Example | Valid Example |
|------|----------------|---------------|
| No spaces in names | `"first name"` | `"first_name"` |
| No hyphens in names | `"first-name"` | `"first_name"` |
| Unique enumeration literal names | Two literals both named `"ACTIVE"` | `"ACTIVE"` and `"ACTIVE_TRIAL"` |

These checks happen at construction time (when you create the object), so you will see the error immediately rather than later during generation.
