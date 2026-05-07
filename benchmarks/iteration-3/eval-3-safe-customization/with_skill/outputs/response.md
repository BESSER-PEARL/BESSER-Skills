# Will my edits to `main_api.py` survive regeneration?

**No.** Every call to `BackendGenerator.generate()` overwrites the output files entirely — `main_api.py`, `pydantic_classes.py`, and `sql_alchemy.py` in `./output_backend/` are all replaced. This is by design: the BUML model is the source of truth, and direct edits to generated files are the one workflow you should *not* adopt.

> **Don't edit generated files directly as your primary workflow — they will be overwritten next time you generate.**
> **Don't mix generated and hand-written code in the same file.**

Here are the safe patterns, ordered from "try first" to "advanced".

---

## 1. Check the generator options first

Many "I need to tweak the output" requests are already covered by `BackendGenerator` parameters. Before reaching for custom code, see if one of these does what you want:

```python
BackendGenerator(
    model=domain_model,
    output_dir="./output_backend",
    http_methods=["GET", "POST"],   # restrict which endpoints are generated
    nested_creations=True,          # allow nested object creates
)
```

If you only need to limit verbs, change the dialect downstream, or toggle Docker, no custom code is needed.

---

## 2. Wrapper script (recommended for custom endpoints)

This is the cleanest answer to "how do I add custom endpoints?" Put your code in a **separate file** that imports the generated FastAPI `app` and attaches new routes to it. Run this wrapper instead of `main_api.py` directly.

```python
# app.py — YOUR CODE, lives outside output_backend/ (or alongside it)
from main_api import app
from fastapi import HTTPException
from sql_alchemy import Book, Session

@app.get("/custom/stats")
def custom_stats():
    with Session() as session:
        return {"book_count": session.query(Book).count()}

@app.post("/books/{book_id}/validate")
def validate_book(book_id: int):
    with Session() as session:
        book = session.query(Book).get(book_id)
        if not book:
            raise HTTPException(404, "not found")
        # your custom validation logic here
        if not book.title or len(book.title) < 2:
            raise HTTPException(422, "title too short")
        return {"valid": True}
```

Run with:

```bash
uvicorn app:app --reload
```

The generated `main_api.py` provides the baseline CRUD; your wrapper layers validation and custom routes on top. Regeneration overwrites `main_api.py` but never touches `app.py`.

A typical layout:

```
output_backend/
  main_api.py          # GENERATED — do not edit
  pydantic_classes.py  # GENERATED — do not edit
  sql_alchemy.py       # GENERATED — do not edit
  app.py               # YOUR CODE — imports from main_api
  custom_endpoints.py  # YOUR CODE — more routes, helpers, validators
```

For shared validation helpers, the same pattern works — put them in `custom_validators.py` and import them from `app.py`.

---

## 3. Push validation into the model (often cleanest)

If your "validation logic" is really a domain operation (e.g. "search books by title", "check ISBN"), the most regeneration-proof place for it is the **model itself**. Define a `Method` with a `MethodImplementation` of type `BAL` or `CODE`, and `BackendGenerator` will emit a matching REST endpoint automatically. The logic lives in the model, where regeneration cannot touch it because the model *drives* regeneration.

```python
from besser.BUML.metamodel.structural import Method, Parameter, StringType
from besser.BUML.metamodel.action_language import (
    MethodImplementationType, MethodImplementation,
)

search = Method(
    name="search_by_title",
    parameters=[Parameter(name="keyword", type=StringType)],
    type=StringType,
)
search.implementation = MethodImplementation(
    type=MethodImplementationType.BAL,
    body='return session.query(Book).filter(Book.title.contains(keyword)).all()',
)
book.add_method(search)
```

Use this when the custom behavior is *part of the domain*. Use the wrapper script (#2) when it's *plumbing* (auth, logging, request shaping, cross-cutting validation).

---

## 4. Template overrides (advanced, sparingly)

If you need to change *how* `main_api.py` itself is rendered — e.g. inject a global middleware into every regeneration — you can copy the generator's Jinja template from `besser/generators/{name}/templates/`, edit your copy, and point the generator at it. Powerful, but BESSER updates won't propagate to your overrides. Use only when #1–#3 don't fit.

---

## 5. Git patch workflow (small, stable edits)

For tiny, repeatable tweaks to generated output:

```bash
git diff output_backend/ > my_customizations.patch          # save once
cd output_backend && git apply ../my_customizations.patch   # reapply after each generate
```

Fine for one-line changes. Brittle for anything larger — prefer #2 or #3.

---

## TL;DR

| Need | Do this |
|------|---------|
| Limit which HTTP verbs are generated | Pass `http_methods=[...]` to `BackendGenerator` |
| Add a custom endpoint / validation route | **Wrapper script** that imports `app` from `main_api` |
| Domain-level operation that should be a real endpoint | Add a `Method` with `BAL`/`CODE` implementation to the model |
| Change how `main_api.py` is rendered globally | Template override |
| Tiny stable tweak to generated text | Git patch reapplied after each generate |

Whatever you do, keep generated files and hand-written files in **separate files**. That single rule keeps you safe across every regeneration.
