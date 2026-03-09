# Fixing "Invalid DBMS" Error with SQLGenerator for PostgreSQL

## The Problem

You are passing `sql_dialect='postgres'` to the `SQLGenerator`, but the valid value for PostgreSQL is **`postgresql`** (not `postgres`).

The `SQLGenerator` delegates to the `SQLAlchemyGenerator` internally, and the `SQLAlchemyGenerator` validates the `dbms` parameter against a strict set of allowed values defined in `VALID_DBMS`:

```python
# From besser/generators/sql_alchemy/sql_alchemy_generator.py, line 28
VALID_DBMS = {"sqlite", "postgresql", "mysql", "mssql", "mariadb", "oracle"}
```

When you pass `sql_dialect='postgres'`, the flow is:

1. `SQLGenerator.__init__()` stores `self.sql_dialect = 'postgres'`
2. `SQLGenerator.generate()` calls `SQLAlchemyGenerator.generate(dbms=self.sql_dialect)`
3. `SQLAlchemyGenerator.generate()` checks `if dbms not in self.VALID_DBMS` and raises:
   ```
   ValueError: Invalid DBMS. Valid options are sqlite, postgresql, mysql, mssql, mariadb, oracle.
   ```

The string `'postgres'` does not match `'postgresql'`, hence the error.

**Note:** The BESSER documentation at `docs/source/generators/sql.rst` incorrectly lists `postgres` as a valid option. This is a documentation bug -- the actual code requires `postgresql`.

## The Fix

Change `sql_dialect='postgres'` to `sql_dialect='postgresql'`:

```python
from besser.generators.sql import SQLGenerator

# WRONG - causes "Invalid DBMS" error
# generator = SQLGenerator(model=my_model, sql_dialect='postgres')

# CORRECT - use 'postgresql' (the full name)
generator = SQLGenerator(model=my_model, sql_dialect='postgresql')
generator.generate()
```

## Complete Working Example

```python
from besser.BUML.metamodel.structural import (
    Class, DomainModel, StringType, IntegerType, DateType,
    Property, BinaryAssociation, Multiplicity
)
from besser.generators.sql import SQLGenerator

# Define classes
library_name = Property(name="name", type=StringType)
address = Property(name="address", type=StringType)
library = Class(name="Library", attributes={library_name, address})

title = Property(name="title", type=StringType)
pages = Property(name="pages", type=IntegerType)
release = Property(name="release", type=DateType)
book = Class(name="Book", attributes={title, pages, release})

author_name = Property(name="name", type=StringType)
email = Property(name="email", type=StringType)
author = Class(name="Author", attributes={author_name, email})

# Define associations
located_in = Property(name="locatedIn", type=library, multiplicity=Multiplicity(1, 1))
has = Property(name="has", type=book, multiplicity=Multiplicity(0, "*"))
lib_book_assoc = BinaryAssociation(name="lib_book_assoc", ends={located_in, has})

publishes = Property(name="publishes", type=book, multiplicity=Multiplicity(0, "*"))
written_by = Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, "*"))
book_author_assoc = BinaryAssociation(name="book_author", ends={written_by, publishes})

# Build domain model
model = DomainModel(
    name="Library_model",
    types={library, book, author},
    associations={lib_book_assoc, book_author_assoc}
)

# Generate SQL for PostgreSQL -- use 'postgresql', NOT 'postgres'
generator = SQLGenerator(model=model, output_dir="./output", sql_dialect="postgresql")
generator.generate()
# Output: ./output/tables_postgresql.sql
```

## All Valid `sql_dialect` Values

| Value          | Database                |
|----------------|-------------------------|
| `sqlite`       | SQLite (default)        |
| `postgresql`   | PostgreSQL              |
| `mysql`        | MySQL                   |
| `mariadb`      | MariaDB                 |
| `mssql`        | Microsoft SQL Server    |
| `oracle`       | Oracle Database         |

These same values apply to the `SQLAlchemyGenerator`'s `dbms` parameter as well:

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

alchemy_gen = SQLAlchemyGenerator(model=model, output_dir="./output")
alchemy_gen.generate(dbms="postgresql")
# Output: ./output/sql_alchemy.py (with PostgreSQL connection string)
```

## What the Generated SQL Looks Like

When using `sql_dialect="postgresql"`, the generated `.sql` file will contain PostgreSQL-specific syntax such as:

- `SERIAL` for auto-incrementing primary keys (instead of SQLite's `INTEGER`)
- `VARCHAR(100)` for string columns
- `CREATE TYPE ... AS ENUM (...)` for enumeration types
- Standard `CREATE TABLE` statements with proper PostgreSQL types

## Summary

The root cause is a mismatch between the documentation (which says `postgres`) and the actual code (which requires `postgresql`). Always use the full name `postgresql` when targeting PostgreSQL with either `SQLGenerator` or `SQLAlchemyGenerator`.
