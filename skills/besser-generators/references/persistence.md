# Persistence generators

Reference for generators that produce database artifacts — ORM code or raw
DDL. Read this when the user is running `SQLAlchemyGenerator` or
`SQLGenerator`.

## SQLAlchemyGenerator

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator
gen = SQLAlchemyGenerator(model=domain_model, output_dir="./output")
gen.generate(dbms="sqlite")
# Valid: sqlite | postgresql | mysql | mssql | mariadb | oracle
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `sql_alchemy.py` |
| Template | `sql_alchemy_template.py.j2` |
| Key behavior | Handles `AssociationClass` as separate tables. Detects abstract parents for concrete table inheritance. Enumerations become SQLAlchemy `Enum` types. |
| Error | `ValueError` if DBMS string is not one of the six valid options. |

**Watch out**: the BESSER documentation at `docs/source/generators/sql.rst`
historically said `'postgres'` — that is wrong, the correct value is
`'postgresql'`. Several baseline agents have hit this in evals.

## SQLGenerator

```python
from besser.generators.sql import SQLGenerator
# sql_dialect goes in the CONSTRUCTOR; generate() takes no arguments.
gen = SQLGenerator(model=domain_model, output_dir="./output", sql_dialect="postgresql")
gen.generate()
# Valid sql_dialect: sqlite | postgresql | mysql | mssql | mariadb | oracle
# (it is "postgresql", not "postgres")
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` |
| Output | Single file: `tables_{dialect}.sql` |
| Dialect | Pass `sql_dialect` to the **constructor** (defaults to `"sqlite"`); `generate()` takes **no** arguments. |
| Key behavior | Two-stage: first generates SQLAlchemy in a temp directory, then runs it as a subprocess to dump DDL. Requires Python to be callable as a subprocess. |
| Gotcha | If the intermediate SQLAlchemy file has issues, the subprocess fails silently — error is printed to stdout, not raised. If the output file is missing with no exception, check stdout. |

**Watch out — the two SQL generators take the dialect in different places.**
`SQLGenerator` takes `sql_dialect` in its **constructor** and `generate()`
takes no args; `SQLAlchemyGenerator` takes `dbms` in **`generate()`**.
Calling `SQLGenerator(...).generate(sql_dialect="postgresql")` raises
`TypeError: generate() got an unexpected keyword argument`.

## Modeling tips that affect persistence

- **Primary key**: if no attribute has `is_id=True`, an auto-incrementing
  `id` column is added. To control it, mark exactly one attribute as
  `is_id=True`.
- **Many-to-many**: a join table is generated automatically — its name is
  the association name, lowercased (the class names are not part of it; e.g.
  an association named `Enrollment` produces a table `enrollment`).
- **Inheritance**: BESSER uses concrete table inheritance for SQLAlchemy
  (each concrete class gets its own table; abstract parents are not
  emitted as tables).
