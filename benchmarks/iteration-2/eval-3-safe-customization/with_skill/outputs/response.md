# Adding Custom Validation and Endpoints to a BackendGenerator Project

## The Short Answer: No, Your Changes Will Not Survive

**Regeneration always overwrites.** Every call to `generate()` replaces the output files entirely. This is by design -- the model is the source of truth. If you edit `main_api.py` directly and then regenerate, your custom validation logic and endpoints will be lost.

The `BackendGenerator` produces three files in its output directory (default `./output_backend/`):

| File | Contents |
|------|----------|
| `main_api.py` | FastAPI application with CRUD endpoints |
| `pydantic_classes.py` | Pydantic models for request/response validation |
| `sql_alchemy.py` | SQLAlchemy ORM models and database setup |

All three are overwritten on every generation cycle. You should **never** edit them directly as your primary workflow.

---

## The Right Way: Safe Customization Patterns

There are several strategies for adding custom validation and endpoints that survive regeneration. Here they are, ordered from simplest to most advanced.

### 1. Use Generator Options First

Before writing any custom code, check whether the `BackendGenerator` already supports what you need via its configuration options:

```python
from besser.generators.backend import BackendGenerator

gen = BackendGenerator(
    model=domain_model,
    output_dir="./output_backend",
    http_methods=["GET", "POST", "PUT", "DELETE"],  # control which CRUD operations are generated
    nested_creations=True,  # allow nested object creation in API payloads
)
gen.generate()
```

- `http_methods` -- limit or expand which HTTP methods are generated (e.g., `["GET", "POST"]` for read-only + create).
- `nested_creations=True` -- enables creating related objects in a single API call.
- `docker_image=True` -- generates `Dockerfile`, `requirements.txt`, and `.dockerignore`.

If generator options cover your needs, you avoid custom code entirely.

### 2. Wrapper Script Pattern (Recommended for Custom Endpoints)

Create a separate `app.py` file that imports the generated FastAPI `app` and adds your custom routes. This file lives alongside the generated code but is **never overwritten** by the generator.

```
output_backend/
  main_api.py           # GENERATED -- do not edit
  pydantic_classes.py   # GENERATED -- do not edit
  sql_alchemy.py        # GENERATED -- do not edit
  app.py                # YOUR CODE -- safe from regeneration
```

Here is a complete working example:

```python
# app.py -- YOUR CODE, not generated, safe from regeneration
from main_api import app
from pydantic_classes import Book, BookBase  # import generated Pydantic models
from sql_alchemy import Book as BookORM, Session  # import generated ORM models
from fastapi import HTTPException
from pydantic import BaseModel

# ─────────────────────────────────────────────
# Custom Validation Endpoint
# ─────────────────────────────────────────────

class BookValidationRequest(BaseModel):
    title: str
    isbn: str
    pages: int

class ValidationResult(BaseModel):
    valid: bool
    errors: list[str]

@app.post("/custom/validate-book", response_model=ValidationResult)
def validate_book(book: BookValidationRequest):
    errors = []

    # Custom validation: ISBN format check
    if len(book.isbn.replace("-", "")) not in (10, 13):
        errors.append("ISBN must be 10 or 13 digits")

    # Custom validation: Page count range
    if book.pages < 1 or book.pages > 10000:
        errors.append("Page count must be between 1 and 10,000")

    # Custom validation: Title length
    if len(book.title.strip()) < 2:
        errors.append("Title must be at least 2 characters")

    return ValidationResult(valid=len(errors) == 0, errors=errors)


# ─────────────────────────────────────────────
# Custom Query Endpoint
# ─────────────────────────────────────────────

@app.get("/custom/books/search")
def search_books(query: str, min_pages: int = 0):
    with Session() as session:
        results = (
            session.query(BookORM)
            .filter(BookORM.title.ilike(f"%{query}%"))
            .filter(BookORM.pages >= min_pages)
            .all()
        )
        return [{"title": b.title, "pages": b.pages} for b in results]


# ─────────────────────────────────────────────
# Custom Business Logic Endpoint
# ─────────────────────────────────────────────

@app.get("/custom/stats")
def get_stats():
    with Session() as session:
        total_books = session.query(BookORM).count()
        return {
            "total_books": total_books,
            "status": "ok",
        }
```

**To run this**, start the wrapper file instead of the generated one:

```bash
cd output_backend
uvicorn app:app --reload --port 8000
```

This gives you the full generated CRUD API **plus** your custom endpoints. After regenerating, your `app.py` remains untouched -- it just imports from the freshly regenerated `main_api.py`.

### 3. Post-Generation Extension Pattern (Recommended for Custom Queries/Services)

For custom validation logic and database queries that are not endpoints themselves, put them in separate service files:

```
output_backend/
  main_api.py            # GENERATED -- do not edit
  pydantic_classes.py    # GENERATED -- do not edit
  sql_alchemy.py         # GENERATED -- do not edit
  custom_validators.py   # YOUR CODE -- safe from regeneration
  custom_queries.py      # YOUR CODE -- safe from regeneration
  app.py                 # YOUR CODE -- safe from regeneration
```

```python
# custom_validators.py -- YOUR CODE, safe from regeneration
from pydantic import BaseModel, field_validator
from pydantic_classes import BookBase  # import generated Pydantic base

class BookWithValidation(BookBase):
    """Extends the generated Pydantic model with custom validation rules."""

    @field_validator("title")
    @classmethod
    def title_must_not_be_empty(cls, v):
        if not v or not v.strip():
            raise ValueError("Title cannot be empty or whitespace")
        return v.strip()

    @field_validator("pages")
    @classmethod
    def pages_must_be_positive(cls, v):
        if v is not None and v < 1:
            raise ValueError("Page count must be at least 1")
        return v
```

```python
# custom_queries.py -- YOUR CODE, safe from regeneration
from sql_alchemy import Book, Author, Session

def get_books_by_author(author_name: str):
    with Session() as session:
        return (
            session.query(Book)
            .join(Author)
            .filter(Author.name == author_name)
            .all()
        )

def get_recent_books(limit: int = 10):
    with Session() as session:
        return session.query(Book).order_by(Book.id.desc()).limit(limit).all()
```

Then wire them into your `app.py`:

```python
# app.py -- YOUR CODE, safe from regeneration
from main_api import app
from custom_validators import BookWithValidation
from custom_queries import get_books_by_author

@app.post("/custom/books/validated")
def create_book_validated(book: BookWithValidation):
    # BookWithValidation runs your custom validators automatically
    # then you can pass it to the generated ORM logic
    from sql_alchemy import Book as BookORM, Session
    with Session() as session:
        db_book = BookORM(title=book.title, pages=book.pages)
        session.add(db_book)
        session.commit()
        return {"id": db_book.id, "title": db_book.title}

@app.get("/custom/books/by-author/{author_name}")
def books_by_author(author_name: str):
    books = get_books_by_author(author_name)
    return [{"title": b.title} for b in books]
```

### 4. Git Patch Workflow (For Small, Repeatable Edits)

If you absolutely must modify the generated files (e.g., adding middleware to `main_api.py`), you can use a patch-based approach:

```bash
# After first generation, make your edits to main_api.py and save a patch
git diff output_backend/ > my_customizations.patch

# After regeneration, reapply your changes
cd output_backend && git apply ../my_customizations.patch
```

This works for small, stable changes but is fragile -- if the generated code changes significantly between model updates, the patch may fail to apply.

### 5. Template Overrides (Advanced)

For maximum control, you can copy and modify the generator's Jinja2 templates. The `BackendGenerator` orchestrates three sub-generators, each with its own template:

- `besser/generators/rest_api/templates/` -- the REST API template (produces `main_api.py`)
- `besser/generators/pydantic_classes/templates/` -- the Pydantic classes template
- `besser/generators/sql_alchemy/templates/` -- the SQLAlchemy template

You can copy a template, modify it to include your custom validation logic directly in the generated output, and point the generator at your custom template. This is the most powerful approach but also the most fragile -- template changes in BESSER updates will not propagate to your overrides. Use sparingly.

---

## Summary: What to Do

| Approach | When to Use | Survives Regeneration? |
|----------|-------------|----------------------|
| Generator options (`http_methods`, `nested_creations`) | Behavior is already configurable | Yes (it is the generation) |
| Wrapper script (`app.py` importing `main_api`) | Adding custom endpoints | Yes |
| Extension files (`custom_validators.py`, `custom_queries.py`) | Custom validation, queries, services | Yes |
| Git patches | Small, repeatable edits to generated files | Fragile, may break |
| Template overrides | Changing the structure of generated code itself | Yes, but must be maintained manually |

**The recommended approach for your case** (custom validation + custom endpoints) is to combine Pattern 2 and Pattern 3: put your validators in `custom_validators.py`, your queries in `custom_queries.py`, and wire everything together in `app.py`. Run `uvicorn app:app` instead of `uvicorn main_api:app`. Your custom code lives in files the generator never touches, and you get the full generated API for free via the import.

## What NOT to Do

- **Do not edit `main_api.py`, `pydantic_classes.py`, or `sql_alchemy.py` directly** as your primary workflow. They will be overwritten.
- **Do not fork the generator** to add one small feature. Use generator options or the wrapper pattern instead.
- **Do not mix generated and hand-written code in the same file.** Separation keeps your code safe across regeneration cycles.
