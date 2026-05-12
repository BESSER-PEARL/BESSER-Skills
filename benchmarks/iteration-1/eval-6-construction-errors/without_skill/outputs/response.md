# Fixing Construction Validation Errors: Spaces in Names and Duplicate Enum Literals

Both errors you are encountering are **construction-time validation errors** enforced by BESSER's metamodel setters. They are intentional safeguards that prevent invalid models from being created. Here is what is happening and how to fix each one.

---

## Issue 1: Property Name with Spaces (`'first name'`)

### Why It Fails

BESSER's `NamedElement` class (the base class for `Property`, `Class`, `Enumeration`, and virtually every named construct) validates names in its setter. The relevant code is in `besser/BUML/metamodel/structural/structural.py`:

```python
@name.setter
def name(self, name: str):
    if ' ' in name:
        raise ValueError(f"'{name}' is invalid. Name cannot contain spaces.")
    if '-' in name:
        raise ValueError(f"'{name}' is invalid. Hyphens are not allowed; use '_' instead.")
    self.__name = name
```

Any name that contains a space or a hyphen will be rejected immediately at object construction time. This validation exists because names are used directly in code generation (Python variables, SQL column names, REST endpoints, etc.), and spaces would produce invalid identifiers in those target languages.

### The Fix: Use Underscores Instead of Spaces

Replace spaces with underscores (`_`) in all element names:

```python
from besser.BUML.metamodel.structural import Property, StringType

# WRONG - will raise ValueError
# first_name_attr = Property(name="first name", type=StringType)

# CORRECT - use underscores
first_name_attr = Property(name="first_name", type=StringType)
```

The same rule applies to class names, method names, enumeration names, and any other `NamedElement`:

```python
from besser.BUML.metamodel.structural import Class, Property, StringType

# Property names: use underscores
first_name = Property(name="first_name", type=StringType)
last_name = Property(name="last_name", type=StringType)
email = Property(name="email", type=StringType)

# Class names: use PascalCase (no spaces, no hyphens)
person = Class(name="Person", attributes={first_name, last_name, email})
```

Similarly, hyphens are also forbidden. Use underscores instead:

```python
# WRONG - will raise ValueError
# prop = Property(name="phone-number", type=StringType)

# CORRECT
prop = Property(name="phone_number", type=StringType)
```

---

## Issue 2: Duplicate Enumeration Literals (`'ACTIVE'` appearing twice)

### Why It Fails

The `Enumeration` class validates that all literal names are unique, both when you construct an enumeration with a set of literals and when you add literals later with `add_literal()`.

From `besser/BUML/metamodel/structural/structural.py`:

```python
# In the literals setter:
@literals.setter
def literals(self, literals: set[EnumerationLiteral]):
    if literals is not None:
        names = [literal.name for literal in literals]
        if len(names) != len(set(names)):
            raise ValueError("An enumeration cannot have two literals with the same name")
        for literal in literals:
            literal.owner = self
        self.__literals = literals
    else:
        self.__literals = set()

# In the add_literal method:
def add_literal(self, literal: EnumerationLiteral):
    if self.literals is not None:
        if literal.name in [literal.name for literal in self.literals]:
            raise ValueError(
                f"An enumeration cannot have two literals with the same name: '{literal.name}'"
            )
    self.literals.add(literal)
```

This validation exists because enumeration literals map directly to unique constants in generated code (Python enums, SQL CHECK constraints, etc.). Having two literals with the same name would be ambiguous and cause errors in every target language.

### The Fix: Give Each Literal a Unique Name

Each `EnumerationLiteral` within the same `Enumeration` must have a distinct name. If you intended to have two separate statuses, give them different names that reflect their distinct meanings:

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# WRONG - duplicate literal name 'ACTIVE' will raise ValueError
# status = Enumeration(name="Status", literals={
#     EnumerationLiteral(name="ACTIVE"),
#     EnumerationLiteral(name="ACTIVE"),
#     EnumerationLiteral(name="INACTIVE"),
# })

# CORRECT - each literal has a unique name
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
    EnumerationLiteral(name="SUSPENDED"),
})
```

If you originally had two `ACTIVE` literals because you need to distinguish between different kinds of "active" states, encode that distinction in the name:

```python
status = Enumeration(name="Status", literals={
    EnumerationLiteral(name="ACTIVE_FULL"),
    EnumerationLiteral(name="ACTIVE_LIMITED"),
    EnumerationLiteral(name="INACTIVE"),
})
```

You can also build enumerations incrementally using `add_literal()`, which performs the same uniqueness check on each addition:

```python
status = Enumeration(name="Status")
status.add_literal(EnumerationLiteral(name="ACTIVE"))
status.add_literal(EnumerationLiteral(name="INACTIVE"))
status.add_literal(EnumerationLiteral(name="SUSPENDED"))

# This would raise ValueError because 'ACTIVE' already exists:
# status.add_literal(EnumerationLiteral(name="ACTIVE"))
```

---

## Complete Working Example

Here is a complete example that combines both fixes into a valid domain model:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, Enumeration, EnumerationLiteral,
    StringType, IntegerType
)

# Define an enumeration with unique literal names
account_status = Enumeration(name="AccountStatus", literals={
    EnumerationLiteral(name="ACTIVE"),
    EnumerationLiteral(name="INACTIVE"),
    EnumerationLiteral(name="SUSPENDED"),
})

# Define properties using underscores (no spaces, no hyphens)
first_name = Property(name="first_name", type=StringType)
last_name = Property(name="last_name", type=StringType)
email = Property(name="email", type=StringType, is_id=True)
status = Property(name="status", type=account_status)

# Define the class
user = Class(name="User", attributes={first_name, last_name, email, status})

# Build the domain model
model = DomainModel(name="UserModel", types={user, account_status})

# Verify it works
for attr in user.attributes:
    print(f"  {attr.name}: {attr.type.name}")
```

---

## Summary of BESSER Naming Rules

| Rule | What Is Checked | Where Enforced |
|------|----------------|----------------|
| No spaces in names | `' ' in name` | `NamedElement.name` setter |
| No hyphens in names | `'-' in name` | `NamedElement.name` setter |
| Unique enum literal names | Duplicate name detection | `Enumeration.literals` setter and `Enumeration.add_literal()` |
| Unique attribute names per class | Duplicate name detection | `Class.attributes` setter |
| Unique method names per class | Duplicate name detection | `Class.methods` setter |
| Unique type names per model | Duplicate name detection | `DomainModel.types` setter |

These are all **construction-time validations** (Level 1 in BESSER's three-level validation strategy). They fire immediately when you set a property value, ensuring that invalid models cannot be created in the first place. Use underscores for multi-word names and ensure all literals within an enumeration have distinct names.
