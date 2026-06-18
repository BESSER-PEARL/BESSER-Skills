I have everything I need from the skill. Here is the answer.

---

## Short answer

**No — your edits will not survive.** `BackendGenerator.generate()` overwrites everything every time. Each call replaces all three output files (`main_api.py`, `pydantic_classes.py`, `sql_alchemy.py`) entirely. This is by design in BESSER: the model is the source of truth, so anything you hand-edit in `main_api.py` is lost the next time you regenerate.

The right approach is to keep your custom logic in **separate files** (or in the model itself), never in the generated file. Below are the recommended patterns, in roughly the order you should try them.

---

## Option A (cleanest for "custom endpoints"): wrapper script that mounts the generated app

Don't touch `main_api.py`. Instead, create your own file that imports the generated FastAPI `app` and adds routes on top of it. The generated file gets overwritten freely; your wrapper is untouched.

```python
# app.py  — YOUR CODE, not generated. Lives next to main_api.py in ./output_backend/
from main_api import app
from sql_alchemy import Book, Author, Session   # reuse generated ORM models + Session

@app.get("/custom/stats")
def custom_stats():
    with Session() as session:
        return {"book_count": session.query(Book).count()}

# Add your custom validation logic here as a dependency or inside the route
from fastapi import HTTPException

@app.post("/custom/books/validated")
def create_book_validated(title: str):
    if len(title.strip()) < 3:
        raise HTTPException(status_code=422, detail="Title must be at least 3 characters")
    with Session() as session:
        book = Book(title=title)
        session.add(book)
        session.commit()
        return {"id": book.id}
```

Then run **your** file instead of the generated one:

```bash
uvicorn app:app --reload     # not main_api:app
```

You get every generated endpoint plus your custom ones, and regeneration of `main_api.py` never disturbs `app.py`.

For pure helper/query logic (no new HTTP route), the same separation applies — put it in e.g. `custom_queries.py` that imports from `sql_alchemy.py`:

```python
# custom_queries.py — safe from regeneration
from sql_alchemy import Book, Author, Session

def get_books_by_author(author_name: str):
    with Session() as session:
        return (session.query(Book).join(Author)
                .filter(Author.name == author_name).all())
```

---

## Option B (best when the logic is really "part of the API"): put it in the model

Often the cleanest fix is to define a method on your model class with `MethodImplementationType.BAL` (or `CODE`). `BackendGenerator` will then *emit* a matching REST endpoint for it automatically. Your custom logic lives in the model — which is the source of truth — so regeneration can't touch it; it actually regenerates it back.

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

Use this when the endpoint conceptually belongs to a domain class. Use Option A when it's app-level glue (cross-cutting stats, auth shims, etc.).

---

## Before either: check whether a generator option already does what you want

Some "I need to change the API" needs are just constructor parameters — no custom code at all:

- `BackendGenerator(model=m, http_methods=["GET", "POST"])` — limit which HTTP verbs get generated (default is GET/POST/PUT/DELETE). Note: invalid method names are silently filtered with a warning log, so check spelling.
- `BackendGenerator(model=m, nested_creations=True)` — allow nested object creation.
- `BackendGenerator(model=m, docker_image=True)` — build/push a Docker image.

---

## Last resort: git patch workflow

If you have a small, stable hand-edit you truly cannot express otherwise, you can capture it as a patch and reapply it after each generation:

```bash
git diff output_backend/ > my_customizations.patch          # save once
cd output_backend && git apply ../my_customizations.patch    # reapply after each generate()
```

This is brittle (any template change can break the patch) — prefer Option A or B.

---

## What to avoid

- Don't edit `main_api.py` directly as your workflow — it's overwritten on every `generate()`.
- Don't mix generated and hand-written code in the same file — keep your code in separate files so it survives.
- Don't fork the generator just to add an endpoint — the wrapper/model patterns above cover it.

---

### One practical note on paths

`BackendGenerator`'s default `output_dir` is **`./output_backend/`**, not the usual `./output/`. Put your `app.py` / `custom_*.py` files alongside the generated files there (so the `from main_api import app` import resolves), or adjust your import paths / `PYTHONPATH` accordingly.

Relevant skill files: `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-generators/SKILL.md` and `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-generators/references/api-and-web.md`.