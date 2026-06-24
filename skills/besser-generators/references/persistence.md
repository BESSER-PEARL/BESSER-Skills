# Persistence generators

Reference for generators that produce database artifacts — ORM code, raw
DDL, or a Supabase schema.

**Covers:** [`SQLAlchemyGenerator`](#sqlalchemygenerator) ·
[`SQLGenerator`](#sqlgenerator) · [`SupabaseGenerator`](#supabasegenerator)

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

## SupabaseGenerator

Generates Supabase-flavored Postgres DDL (a single migration-style `.sql` file) from a `DomainModel`: enum types, tables, association foreign keys, an `auth.users` mirror trigger, grants, and Row Level Security policies.

```python
from besser.generators.supabase import SupabaseGenerator

gen = SupabaseGenerator(
    model=domain_model,      # a BUML DomainModel
    output_dir="./output",   # optional; defaults to <cwd>/output
    user_root="User",        # optional; class mirrored onto auth.users; pass None to skip auth
)
gen.generate()               # writes <YYYYMMDDHHMMSS>_<model_slug>.sql; prints the path
```

| Aspect | Detail |
|--------|--------|
| Input | `DomainModel` (reads classes, attributes' `is_id`/`is_optional`/type, enumerations, binary associations) |
| Output | One file `<YYYYMMDDHHMMSS>_<model_slug>.sql` in `output_dir`. **Timestamped — every run creates a NEW file, never overwrites** (unlike other generators) |
| Options | `user_root: str = "User"` — class mirrored onto `auth.users`; pass `None` to disable auth/triggers/RLS |
| Key behavior | Emits enum types, tables, FK `ALTER`s, an `auth.users` mirror trigger, grants, and Row Level Security policies (see Details) |
| Gotcha | Identifier names are **lowercased**; `AssociationClass` is skipped; only **binary** associations are processed; inheritance is flattened (no table-per-hierarchy) |

### Details

- **Type map:** `str→TEXT`, `int→INTEGER`, `float→DOUBLE PRECISION`, `bool→BOOLEAN`, `date→DATE`, `time→TIME`, `datetime→TIMESTAMPTZ`, `timedelta→INTERVAL`, `any→JSONB`; unknown→`TEXT`; enum-typed attrs use the enum type name.
- **Keys & relations:** `is_id` attrs → `UUID PRIMARY KEY DEFAULT gen_random_uuid()` (or `REFERENCES auth.users(id)` on the user-root); M:N associations → junction tables.
- **Auth & RLS:** per-user classes get a `user_id` column + RLS policies keyed on `(SELECT auth.uid()) = "user_id"`. The user-root id is force-emitted as a reference to `auth.users` even if you omit `is_id` on that class.

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
