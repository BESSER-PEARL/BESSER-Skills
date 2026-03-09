# Adding Custom Validation and Endpoints to a BackendGenerator Output

## Short Answer: No, Direct Edits to `main_api.py` Will Not Survive Regeneration

Every call to `BackendGenerator.generate()` completely overwrites all three output files:

- `main_api.py` (the FastAPI application)
- `pydantic_classes.py` (Pydantic request/response schemas)
- `sql_alchemy.py` (SQLAlchemy ORM models)

This is by design -- the B-UML model is the single source of truth, and regeneration replaces these files entirely. Any custom validation logic, endpoints, or middleware you add directly to `main_api.py` will be lost the next time you run `generate()`.

The relevant code in `BackendGenerator.generate()` (at `besser/generators/backend/backend_generator.py`, line 85-86) shows how it delegates to sub-generators that each overwrite their output file unconditionally:

```python
rest_api = RESTAPIGenerator(model=self.model, http_methods=self.http_methods,
    nested_creations=self.nested_creations, output_dir=backend_folder_path, backend=True, port=docker_port)
rest_api.generate()  # Overwrites main_api.py

sql_alchemy = SQLAlchemyGenerator(model=self.model, output_dir=backend_folder_path)
sql_alchemy.generate()  # Overwrites sql_alchemy.py

pydantic_model = PydanticGenerator(model=self.model, output_dir=backend_folder_path, backend=True,
    nested_creations=self.nested_creations)
pydantic_model.generate()  # Overwrites pydantic_classes.py
```

---

## The Right Way: Post-Generation Extension Files

The recommended approach is to keep your custom code in **separate files** that import from the generated files. Generated files get overwritten; your extension files do not.

### Pattern 1: Wrapper App (Best for Custom Endpoints)

Create a file called `app.py` (or any name you choose) alongside the generated files. This file imports the generated FastAPI `app` instance and adds your custom routes to it. The generated `main_api.py` defines `app = FastAPI(...)` at module level and only runs `uvicorn` inside an `if __name__ == "__main__"` guard, which means importing the module will not start the server -- it will just give you the `app` object with all the generated CRUD endpoints already registered.

```
output_backend/
    main_api.py          # GENERATED -- do not edit
    pydantic_classes.py   # GENERATED -- do not edit
    sql_alchemy.py        # GENERATED -- do not edit
    app.py               # YOUR CODE -- safe from regeneration
    custom_validators.py  # YOUR CODE -- safe from regeneration
```

Here is a concrete example of `app.py`:

```python
# app.py -- YOUR CODE, not generated, safe from regeneration
from main_api import app, get_db  # Import the generated FastAPI app and DB dependency
from sql_alchemy import Book, Author, Session  # Import generated ORM models
from pydantic_classes import *  # Import generated Pydantic schemas
from fastapi import Depends, HTTPException, Body
from sqlalchemy.orm import Session as SASession
from pydantic import BaseModel, field_validator
from typing import Optional

# =============================================
# Custom Pydantic models with validation logic
# =============================================

class BookCreateValidated(BaseModel):
    """Custom schema with validation that goes beyond what the generator produces."""
    title: str
    isbn: str
    year: int
    author_id: int

    @field_validator("isbn")
    @classmethod
    def validate_isbn(cls, v):
        cleaned = v.replace("-", "").replace(" ", "")
        if len(cleaned) not in (10, 13):
            raise ValueError("ISBN must be 10 or 13 digits")
        if not cleaned.isdigit():
            raise ValueError("ISBN must contain only digits (and optional hyphens)")
        return v

    @field_validator("year")
    @classmethod
    def validate_year(cls, v):
        if v < 1450 or v > 2030:
            raise ValueError("Year must be between 1450 and 2030")
        return v

# =============================================
# Custom endpoints
# =============================================

@app.post("/custom/book/validated/", tags=["Custom"])
async def create_book_validated(
    book_data: BookCreateValidated,
    database: SASession = Depends(get_db)
):
    """Create a book with custom ISBN and year validation."""
    # Your custom validation already ran via Pydantic validators above.
    # Now check business rules:
    existing = database.query(Book).filter(Book.isbn == book_data.isbn).first()
    if existing:
        raise HTTPException(status_code=409, detail="A book with this ISBN already exists")

    author = database.query(Author).filter(Author.id == book_data.author_id).first()
    if author is None:
        raise HTTPException(status_code=404, detail="Author not found")

    db_book = Book(
        title=book_data.title,
        isbn=book_data.isbn,
        year=book_data.year,
        author_id=book_data.author_id,
    )
    database.add(db_book)
    database.commit()
    database.refresh(db_book)
    return db_book


@app.get("/custom/stats/", tags=["Custom"])
async def get_statistics(database: SASession = Depends(get_db)):
    """Custom analytics endpoint not available in the generated CRUD API."""
    book_count = database.query(Book).count()
    author_count = database.query(Author).count()
    return {
        "total_books": book_count,
        "total_authors": author_count,
        "avg_books_per_author": round(book_count / max(author_count, 1), 2),
    }


# =============================================
# Run the app (instead of running main_api.py directly)
# =============================================
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

**How to run**: Instead of `python main_api.py`, you run `python app.py`. The `app` object already contains all the generated CRUD routes. Your file simply adds more routes to the same application. After regeneration, the generated routes update automatically while your custom routes remain untouched.

### Pattern 2: Separate Custom Endpoints Module (For Larger Projects)

If you have many custom endpoints, put them in their own module and include them via a FastAPI router:

```python
# custom_endpoints.py -- YOUR CODE
from fastapi import APIRouter, Depends, HTTPException
from sql_alchemy import Book, Author
from main_api import get_db
from sqlalchemy.orm import Session

router = APIRouter(prefix="/custom", tags=["Custom"])

@router.get("/books/search/")
async def search_books(q: str, database: Session = Depends(get_db)):
    results = database.query(Book).filter(Book.title.ilike(f"%{q}%")).all()
    return results

@router.get("/authors/{author_id}/bibliography/")
async def get_bibliography(author_id: int, database: Session = Depends(get_db)):
    author = database.query(Author).filter(Author.id == author_id).first()
    if not author:
        raise HTTPException(status_code=404, detail="Author not found")
    books = database.query(Book).filter(Book.author_id == author_id).order_by(Book.year).all()
    return {"author": author.name, "books": books}
```

Then in your `app.py`:

```python
# app.py -- YOUR CODE
from main_api import app
from custom_endpoints import router

app.include_router(router)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

### Pattern 3: Custom Validators as Middleware

If you want validation that applies to all requests (not just specific endpoints), add middleware in your wrapper:

```python
# app.py -- YOUR CODE
from main_api import app
from fastapi import Request
from fastapi.responses import JSONResponse

@app.middleware("http")
async def validate_content_type(request: Request, call_next):
    """Require JSON content type on all POST/PUT requests."""
    if request.method in ("POST", "PUT"):
        content_type = request.headers.get("content-type", "")
        if "application/json" not in content_type:
            return JSONResponse(
                status_code=415,
                content={"detail": "Content-Type must be application/json"}
            )
    return await call_next(request)

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Before You Write Custom Code: Check Generator Options First

The `BackendGenerator` has built-in options that may already cover your needs, potentially making custom code unnecessary:

| Option | What it does |
|--------|-------------|
| `http_methods=["GET", "POST"]` | Limit which CRUD endpoints are generated. For example, omit `DELETE` to prevent deletions via API. |
| `nested_creations=True` | Changes POST behavior so that related entities can be created inline (nested in the request body) instead of requiring pre-existing IDs. |
| `docker_image=True` | Generates a `Dockerfile` and `requirements.txt` for containerized deployment. |

Example:

```python
from besser.generators.backend import BackendGenerator

gen = BackendGenerator(
    model=domain_model,
    http_methods=["GET", "POST"],      # No PUT or DELETE endpoints
    nested_creations=True,             # Allow nested object creation
    output_dir="./output_backend",
)
gen.generate()
```

---

## What Not To Do

1. **Do not edit `main_api.py`, `pydantic_classes.py`, or `sql_alchemy.py` directly.** They will be overwritten on the next `generate()` call.

2. **Do not mix generated and hand-written code in the same file.** The file boundary is what protects your custom code from regeneration.

3. **Do not fork the BackendGenerator** just to add a few endpoints. The wrapper pattern described above is simpler and survives BESSER updates.

4. **If you must make small, repeatable edits to generated files** (e.g., adding a single import line), consider a git-based patch workflow as a last resort:

   ```bash
   # After generation, make your edits and save a patch
   git diff output_backend/ > my_customizations.patch

   # After regeneration, reapply
   cd output_backend && git apply ../my_customizations.patch
   ```

   This is fragile and should only be used for small, stable changes. The extension file pattern is strongly preferred.

---

## Summary

| Approach | When to use | Survives regeneration? |
|----------|------------|----------------------|
| Generator options (`http_methods`, `nested_creations`) | Controlling which CRUD endpoints are generated | Yes (configuration, not code) |
| Wrapper `app.py` that imports `main_api.app` | Adding custom endpoints, validation, middleware | Yes |
| Separate router modules included via `app.include_router()` | Many custom endpoints, clean separation | Yes |
| Editing `main_api.py` directly | Never recommended | **No** |
| Git patch workflow | Tiny, stable edits to generated files | Fragile, last resort |

The key insight is that the generated `main_api.py` exposes a standard FastAPI `app` object that you can import and extend from your own files. Run your wrapper file instead of the generated file, and your custom logic will coexist with the generated CRUD endpoints across any number of regeneration cycles.
