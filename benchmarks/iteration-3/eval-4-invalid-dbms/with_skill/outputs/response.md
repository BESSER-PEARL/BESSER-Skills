# `Invalid DBMS` error from `SQLGenerator` — two things to fix

You've actually got two separate issues stacked on top of each other. Let's untangle them.

## Issue 1: The dialect string is wrong

BESSER does not accept `'postgres'`. The valid set is exactly:

```
sqlite | postgresql | mysql | mssql | mariadb | oracle
```

The correct string for PostgreSQL is **`postgresql`** (with the `ql`). This is a well-known papercut — even the BESSER docs historically said `'postgres'`, and several baseline runs trip over it. The generator raises `ValueError: Invalid DBMS` for anything outside the six values above.

## Issue 2: `SQLGenerator` doesn't take a `sql_dialect` kwarg

`SQLGenerator` (raw DDL) and `SQLAlchemyGenerator` (ORM) have slightly different APIs. The dialect parameter is named **`sql_dialect`** on `SQLGenerator` and **`dbms`** on `SQLAlchemyGenerator`, but the *valid values* are the same set, and `SQLGenerator` internally delegates to `SQLAlchemyGenerator` in a subprocess — so the `Invalid DBMS` error you're seeing is bubbling up from the inner SQLAlchemy step, validating the value you passed.

## Fix

```python
from besser.generators.sql import SQLGenerator

gen = SQLGenerator(model=domain_model, output_dir="./output")
gen.generate(sql_dialect="postgresql")   # not "postgres"
```

This will produce `./output/tables_postgresql.sql`.

If you'd rather generate the ORM directly (no subprocess), use:

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

gen = SQLAlchemyGenerator(model=domain_model, output_dir="./output")
gen.generate(dbms="postgresql")          # not "postgres"
```

That writes `./output/sql_alchemy.py`.

## If swapping `postgres` → `postgresql` doesn't fix it

`SQLGenerator` is a composite: it generates a temporary SQLAlchemy file and then runs it as a Python subprocess to dump the DDL. **If that subprocess fails, the error is printed to stdout, not raised** — so the file may be missing with no traceback in your Python process. Check your console output for a Python traceback that didn't bubble up. Common culprits in that subprocess:

- Model validation issues (run `model.validate()` first).
- A class name with spaces or hyphens (these raise at model-construction time, but can also show up here).
- Missing primary keys are fine — SQLAlchemy will auto-add an `id` column unless you set `is_id=True` on an attribute to take control.

## Quick checklist

- [ ] Replace `'postgres'` with `'postgresql'`.
- [ ] Confirm you're calling `gen.generate(sql_dialect=...)` on `SQLGenerator` (or `dbms=...` on `SQLAlchemyGenerator`).
- [ ] If still no output and no exception, check stdout for a swallowed subprocess traceback.
- [ ] Run `model.validate()` before generating.
