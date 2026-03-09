# Fixing "Invalid DBMS" Error with SQLGenerator

## The Problem

You are passing `sql_dialect='postgres'` to `SQLGenerator`, but `'postgres'` is not a recognized dialect string. The valid values are:

- `sqlite`
- `postgresql`
- `mysql`
- `mssql`
- `mariadb`
- `oracle`

The correct value is **`postgresql`**, not `postgres`. This is a common mistake since many tools (e.g., `psql`, connection URIs like `postgres://...`) use the shorter form, but BESSER requires the full `postgresql` spelling.

## How It Works Under the Hood

`SQLGenerator` is a composite generator. It does not generate SQL directly. Instead, it:

1. Internally invokes `SQLAlchemyGenerator` to produce an intermediate `sql_alchemy.py` file in a temp directory.
2. Runs that file as a subprocess to dump the DDL (the raw SQL statements).
3. Writes the result to `tables_{dialect}.sql`.

The dialect string you pass to `SQLGenerator.generate(sql_dialect=...)` is forwarded to `SQLAlchemyGenerator.generate(dbms=...)`, which validates it against the six allowed values. If the string does not match, you get a `ValueError: Invalid DBMS`.

## The Fix

Change `'postgres'` to `'postgresql'`:

```python
from besser.generators.sql import SQLGenerator

gen = SQLGenerator(model=domain_model, output_dir="./output")
gen.generate(sql_dialect="postgresql")  # NOT 'postgres'
```

This will produce a file named `tables_postgresql.sql` in your output directory.

## Complete Working Example

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, PrimitiveDataType,
    BinaryAssociation, Multiplicity
)

# Define primitive types
string_type = PrimitiveDataType("str")
int_type = PrimitiveDataType("int")

# Define classes
book = Class(name="Book", attributes={
    Property(name="title", type=string_type),
    Property(name="isbn", type=string_type, is_id=True),
    Property(name="pages", type=int_type),
})

author = Class(name="Author", attributes={
    Property(name="name", type=string_type),
    Property(name="author_id", type=int_type, is_id=True),
})

# Define association
writes = BinaryAssociation(
    name="writes",
    ends={
        Property(name="books", type=book, multiplicity=Multiplicity(0, "*")),
        Property(name="authors", type=author, multiplicity=Multiplicity(1, "*")),
    }
)

# Build domain model
domain_model = DomainModel(
    name="Library",
    types={book, author},
    associations={writes},
)

# Generate SQL for PostgreSQL
from besser.generators.sql import SQLGenerator

gen = SQLGenerator(model=domain_model, output_dir="./output")
gen.generate(sql_dialect="postgresql")
# Output: ./output/tables_postgresql.sql
```

## If You Need SQLAlchemy Code Instead

If you want the SQLAlchemy ORM Python file (not raw SQL), use `SQLAlchemyGenerator` directly with the `dbms` parameter:

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

gen = SQLAlchemyGenerator(model=domain_model, output_dir="./output")
gen.generate(dbms="postgresql")
# Output: ./output/sql_alchemy.py (configured for PostgreSQL)
```

## Valid Dialect Reference

| Value          | Database              |
|----------------|-----------------------|
| `sqlite`       | SQLite                |
| `postgresql`   | PostgreSQL            |
| `mysql`        | MySQL                 |
| `mssql`        | Microsoft SQL Server  |
| `mariadb`      | MariaDB               |
| `oracle`       | Oracle Database       |

## Additional Gotcha with SQLGenerator

Be aware that `SQLGenerator` runs a subprocess internally. If the intermediate SQLAlchemy file has issues (e.g., model problems that only surface at execution time), the subprocess may fail silently -- the error is printed to stdout but no exception is raised, and you end up with no output file. If you run the corrected dialect and still see no output, check the console for printed error messages.
