The root cause is the dialect string. `'postgres'` is not a valid value — the correct value is `'postgresql'`.

## The fix

The "Invalid DBMS" error is coming from the dialect string, not from how you're calling the generator. BESSER's `SQLGenerator` does **not** accept `'postgres'`. The valid value is `'postgresql'`.

(This is a well-known trap: BESSER's own docs historically said `'postgres'`, but that value is wrong.)

```python
from besser.generators.sql import SQLGenerator

# sql_dialect goes in the CONSTRUCTOR; generate() takes no arguments.
gen = SQLGenerator(
    model=domain_model,
    output_dir="./output",
    sql_dialect="postgresql",   # NOT "postgres"
)
gen.generate()                  # no arguments here
```

This writes `tables_postgresql.sql` into `./output/`.

## Two things to get right

1. **Use `'postgresql'`, not `'postgres'`.** Valid `sql_dialect` values are exactly:
   `sqlite | postgresql | mysql | mssql | mariadb | oracle`

2. **Pass the dialect in the constructor, and call `generate()` with no args.** This is the part people most often get backwards. For `SQLGenerator`:
   - `sql_dialect` → **constructor** (defaults to `"sqlite"` if omitted)
   - `generate()` → **no arguments**

   Do NOT write `SQLGenerator(...).generate(sql_dialect="postgresql")` — that raises `TypeError: generate() got an unexpected keyword argument`.

   Note this differs from `SQLAlchemyGenerator`, which is the opposite: it takes the dialect as `dbms` inside `generate()`, e.g. `SQLAlchemyGenerator(model=...).generate(dbms="postgresql")`. Don't mix the two patterns up.

## If it still fails (no .sql file, no exception)

`SQLGenerator` works in two stages: it first generates a SQLAlchemy file in a temp directory, then runs it as a **subprocess** to dump the DDL. Consequences:

- Python must be callable as a subprocess in your environment.
- If the intermediate SQLAlchemy step has a problem, the subprocess can fail **silently** — the error is printed to **stdout**, not raised as an exception. So if `tables_postgresql.sql` is missing but you got no traceback, check the stdout/console output for the real error.

Relevant skill files:
- `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-generators/references/persistence.md`
- `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-generators/SKILL.md`