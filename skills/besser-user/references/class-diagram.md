# B-UML Structural Metamodel Reference

Full reference for building B-UML domain models in Python. Read this when the
user is constructing classes, attributes, associations, enumerations,
generalizations, or methods, or when they hit a metamodel-related question
that the SKILL.md overview does not cover.

## Table of contents

- [Imports](#imports)
- [Naming rules](#naming-rules)
- [Classes and attributes](#classes-and-attributes)
- [Primitive types](#primitive-types)
- [Enumerations](#enumerations)
- [Associations](#associations)
- [Association classes](#association-classes)
- [Inheritance / generalizations](#inheritance--generalizations)
- [Methods](#methods)
- [Assembling the model](#assembling-the-model)
- [Validation](#validation)
- [Complete example](#complete-example)

## Imports

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, Generalization,
    Enumeration, EnumerationLiteral,
    Method, Parameter, AssociationClass,
    StringType, IntegerType, FloatType, BooleanType,
    DateType, DateTimeType, TimeType, TimeDeltaType,
    UNLIMITED_MAX_MULTIPLICITY,  # 9999, means "many"
)
```

## Naming rules

Everything in B-UML descends from `NamedElement`. Names must:

- not contain spaces (use `My_Name`, not `My Name`)
- not contain hyphens (use `my_name`, not `my-name`)
- be unique within their owning collection (no two classes with the same
  name in a `DomainModel`, no two literals with the same name in an
  `Enumeration`, etc.)

Conventions: `snake_case` for attributes/methods, `PascalCase` for classes
and enums, `UPPER_CASE` for enumeration literals.

## Classes and attributes

```python
title = Property(name="title", type=StringType)
pages = Property(name="pages", type=IntegerType)
book = Class(name="Book", attributes={title, pages})

# Mark a primary key — only one is_id per class
book_id = Property(name="id", type=IntegerType, is_id=True)
```

Attributes are `Property` objects whose `type` is a primitive type, an
`Enumeration`, or another `Class` (when used as an association end).

## Primitive types

The metamodel ships pre-built singleton instances:

| Singleton | Equivalent string |
|-----------|-------------------|
| `StringType` | `"str"` |
| `IntegerType` | `"int"` |
| `FloatType` | `"float"` |
| `BooleanType` | `"bool"` |
| `DateType` | `"date"` |
| `DateTimeType` | `"datetime"` |
| `TimeType` | `"time"` |
| `TimeDeltaType` | `"timedelta"` |
| `AnyType` | `"any"` |

Both forms work — `Property(name="x", type=StringType)` and
`Property(name="x", type="str")` produce the same result. Prefer the
singletons for IDE autocomplete.

## Enumerations

```python
genre = Enumeration(name="Genre", literals={
    EnumerationLiteral(name="FICTION"),
    EnumerationLiteral(name="SCIENCE"),
    EnumerationLiteral(name="HISTORY"),
})

# Use as an attribute type
book_genre = Property(name="genre", type=genre)

# Access literals by name: genre.FICTION, genre.SCIENCE
# Default values:
status = Property(name="genre", type=genre, default_value=genre.FICTION)
```

## Associations

A `BinaryAssociation` connects two classes via two `Property` ends. Each end
declares the *target* class and the cardinality from the other side.

```python
located_in = Property(name="locatedIn", type=library, multiplicity=Multiplicity(1, 1))
has_books  = Property(name="has",       type=book,    multiplicity=Multiplicity(0, "*"))

lib_book = BinaryAssociation(name="lib_book", ends={located_in, has_books})
```

`Multiplicity(min, max)` — `max` accepts `"*"` or `UNLIMITED_MAX_MULTIPLICITY`
(=9999) for "many".

Common cardinalities:

| Pattern | Multiplicity |
|---------|-------------|
| Exactly one | `Multiplicity(1, 1)` |
| Optional (0..1) | `Multiplicity(0, 1)` |
| One or more | `Multiplicity(1, "*")` |
| Zero or more | `Multiplicity(0, "*")` |

For composition (whole-part ownership where the part cannot exist without
the whole), set `is_composite=True` on the *whole* end.

## Association classes

An `AssociationClass` is both a class (with its own attributes) and a
binary association between two classes. Define the `BinaryAssociation`
first, then wrap it with `AssociationClass`.

```python
# Classes
person  = Class(name="Person",  attributes={...})
project = Class(name="Project", attributes={...})

# Underlying association
person_end  = Property(name="person",  type=person,  multiplicity=Multiplicity(0, "*"))
project_end = Property(name="project", type=project, multiplicity=Multiplicity(0, "*"))
assignment_assoc = BinaryAssociation(name="assignment_assoc", ends={person_end, project_end})

# Association class — adds attributes to the link itself
role       = Property(name="role",       type=StringType)
start_date = Property(name="startDate",  type=DateType)
Assignment = AssociationClass(
    name="Assignment",
    attributes={role, start_date},
    association=assignment_assoc,
)
```

Add both the `AssociationClass` and its underlying `BinaryAssociation` to
the model:

```python
model = DomainModel(
    name="MyModel",
    types={person, project, Assignment},
    associations={assignment_assoc},
)
```

## Inheritance / generalizations

```python
person = Class(name="Person", attributes={name_prop, email_prop})
employee = Class(name="Employee", attributes={salary_prop})
gen = Generalization(general=person, specific=employee)
# employee inherits person's attributes
```

`general` is the parent, `specific` is the child. A class cannot generalize
itself; circular inheritance is rejected at `model.validate()`.

## Methods

```python
from besser.BUML.metamodel.structural import Method, Parameter

greet = Method(
    name="greet",
    parameters=[Parameter(name="message", type=StringType)],
    type=StringType,    # return type
    is_abstract=False,
)
person.add_method(greet)
```

Methods can carry implementations — see the besser-generators skill for
`MethodImplementationType.BAL` / `MethodImplementationType.CODE`, which
auto-generate REST endpoints in `BackendGenerator`.

## Assembling the model

```python
model = DomainModel(
    name="Library_model",
    types={library, book, author, genre},   # classes + enumerations
    associations={lib_book, book_author},
    generalizations={gen},
)
```

Every class, enumeration, and data type referenced anywhere in the model
must appear in `types`. Associations and generalizations referencing
unlisted types fail validation.

## Validation

```python
result = model.validate()
# {"success": True/False, "errors": [...], "warnings": [...]}
```

Always call `validate()` before generating. Common errors caught here:

- generalization references class not in `types`
- association end references type not in `types`
- circular inheritance
- OCL constraint context not in `types`

## Complete example

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType, DateType
)

# Classes
library_name = Property(name="name", type=StringType)
address = Property(name="address", type=StringType)
library = Class(name="Library", attributes={library_name, address})

title   = Property(name="title",   type=StringType)
pages   = Property(name="pages",   type=IntegerType)
release = Property(name="release", type=DateType)
book    = Class(name="Book", attributes={title, pages, release})

author_name = Property(name="name",  type=StringType)
email       = Property(name="email", type=StringType)
author      = Class(name="Author", attributes={author_name, email})

# Associations
located_in = Property(name="locatedIn", type=library, multiplicity=Multiplicity(1, 1))
has        = Property(name="has",       type=book,    multiplicity=Multiplicity(0, "*"))
lib_book   = BinaryAssociation(name="lib_book", ends={located_in, has})

publishes  = Property(name="publishes", type=book,   multiplicity=Multiplicity(0, "*"))
written_by = Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, "*"))
book_author = BinaryAssociation(name="book_author", ends={written_by, publishes})

# Model
library_model = DomainModel(
    name="Library_model",
    types={library, book, author},
    associations={lib_book, book_author},
)

assert library_model.validate()["success"]
```
