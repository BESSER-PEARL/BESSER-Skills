# Will Editing `main_api.py` Survive the Next Generation? No.

**Short answer: No.** Every time you call `BackendGenerator.generate()`, it completely overwrites `main_api.py`, `sql_alchemy.py`, and `pydantic_classes.py`. Any manual edits you make to those files will be lost. This is by design -- the generator uses Jinja2 templates to produce fresh output each time (see `besser/generators/rest_api/templates/backend_fast_api_template.py.j2`).

The docs themselves note this in `docs/source/examples/backend_example.rst`:

> "It is important to note that the generated code is a starting point and can be further customized to meet the specific requirements of the backend service."

This means once you are done iterating on generation, you treat the output as a starting point. But there are better approaches than editing `main_api.py` directly if you want to keep regenerating.

Below are the recommended strategies, ordered from most model-driven (preferred) to most manual.

---

## Strategy 1: Define Methods in Your B-UML Model (Recommended)

BESSER supports defining **method implementations** directly on your domain classes. The `BackendGenerator` will automatically generate REST API endpoints for those methods. This is the cleanest approach because your custom logic lives in the model, not in generated code.

### Option A: Using BESSER Action Language (BAL)

BAL is BESSER's own action language that gets translated into REST API code. It supports field assignments, conditionals, loops, and calls to other model methods.

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Method, Parameter,
    MethodImplementationType, Multiplicity, BinaryAssociation,
    StringType, IntegerType, FloatType
)

# Define your class
product = Class(
    name="Product",
    attributes={
        Property(name="name", type=StringType),
        Property(name="price", type=FloatType),
        Property(name="stock", type=IntegerType),
    }
)

# Define a method with BAL implementation
apply_discount = Method(
    name="apply_discount",
    parameters=[Parameter(name="percent", type=FloatType)],
    type=None,
    implementation_type=MethodImplementationType.BAL
)
apply_discount.code = """def apply_discount(percent: real) -> nothing {
    this.price = this.price * (1.0 - percent / 100.0)
}"""
product.add_method(apply_discount)

# Define a class-level validation method (no 'self' / 'this')
check_stock = Method(
    name="check_low_stock",
    parameters=[Parameter(name="threshold", type=IntegerType)],
    type=None,
    implementation_type=MethodImplementationType.BAL
)
check_stock.code = """def check_low_stock(threshold: int) -> nothing {
    // Class-level methods don't use 'this'
    // They operate on the whole collection
}"""
product.add_method(check_stock)

model = DomainModel(name="Shop", types={product}, associations=set())
```

The generator will produce:

- **Instance method endpoint**: `POST /product/{product_id}/methods/apply_discount/`
  - Request body: `{"params": {"percent": 10.0}}`
- **Class-level method endpoint**: `POST /product/methods/check_low_stock/`
  - Request body: `{"params": {"threshold": 5}}`

### Option B: Using Raw Python Code (CODE Implementation Type)

If BAL is too restrictive, you can embed raw Python code strings. The generator will inject them directly into the endpoint body.

```python
validate_order = Method(
    name="validate_order",
    parameters=[Parameter(name="min_quantity", type=IntegerType)],
    type=None,
    implementation_type=MethodImplementationType.CODE
)
validate_order.code = """def validate_order(self, min_quantity: int):
    if self.stock < min_quantity:
        raise Exception(f"Insufficient stock: {self.stock} < {min_quantity}")
    return {"valid": True, "available_stock": self.stock}
"""
product.add_method(validate_order)
```

This generates an endpoint at `POST /product/{product_id}/methods/validate_order/` that:
1. Fetches the `Product` entity from the database by ID
2. Executes your code with `self` replaced by the entity instance
3. Commits changes and returns the result

The key difference between BAL and CODE:

| Feature | BAL | CODE |
|---------|-----|------|
| Type-checked | Yes (by `BALTypeChecker`) | No |
| Generates DB update calls | Automatically via `update_{entity}()` | You must handle DB mutations manually |
| Syntax | BESSER Action Language (`this.field = value;`) | Raw Python (`self.field = value`) |
| Best for | Simple field mutations, assignments | Complex logic, external calls, queries |

---

## Strategy 2: Use FastAPI's `include_router` Pattern (Post-Generation)

Since the generated `main_api.py` exposes the `app` FastAPI instance, you can create a **separate file** for your custom endpoints and include it via FastAPI's router mechanism. This way, you only edit a file the generator does not touch.

### Step 1: Generate your backend as usual

```python
from besser.generators.backend import BackendGenerator

backend = BackendGenerator(
    model=my_model,
    http_methods=["GET", "POST", "PUT", "DELETE"],
    output_dir="./my_backend"
)
backend.generate()
```

This produces:
- `my_backend/main_api.py`
- `my_backend/sql_alchemy.py`
- `my_backend/pydantic_classes.py`

### Step 2: Create a custom router file (never overwritten)

Create `my_backend/custom_routes.py`:

```python
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic import BaseModel

# Import your generated models
from sql_alchemy import *
from pydantic_classes import *

router = APIRouter(prefix="/custom", tags=["Custom Endpoints"])


# --- Custom Pydantic models for your validation logic ---

class OrderValidationRequest(BaseModel):
    product_id: int
    quantity: int


class OrderValidationResponse(BaseModel):
    valid: bool
    message: str
    available_stock: int


# --- Custom endpoints ---

@router.post("/validate-order", response_model=OrderValidationResponse)
def validate_order(request: OrderValidationRequest, database: Session = Depends()):
    """Custom validation: check stock before allowing an order."""
    product = database.query(Product).filter(Product.id == request.product_id).first()
    if not product:
        raise HTTPException(status_code=404, detail="Product not found")

    if product.stock < request.quantity:
        return OrderValidationResponse(
            valid=False,
            message=f"Insufficient stock for product '{product.name}'",
            available_stock=product.stock
        )

    return OrderValidationResponse(
        valid=True,
        message="Order is valid",
        available_stock=product.stock
    )


@router.post("/apply-bulk-discount")
def apply_bulk_discount(
    category: str,
    discount_percent: float,
    database: Session = Depends()
):
    """Custom business logic: apply discount to all products in a category."""
    products = database.query(Product).filter(
        Product.category == category
    ).all()

    if not products:
        raise HTTPException(status_code=404, detail=f"No products in category '{category}'")

    for product in products:
        product.price = product.price * (1 - discount_percent / 100)

    database.commit()
    return {
        "updated_count": len(products),
        "discount_applied": discount_percent
    }
```

### Step 3: Add one line to `main_api.py` (after generation)

After each regeneration, add this single line near the top of `main_api.py`, right after the `app` is created:

```python
# Add this after the line:  app = FastAPI(...)
from custom_routes import router as custom_router
app.include_router(custom_router)
```

To make this more robust, you can automate this with a post-generation script:

```python
import os

def patch_main_api(output_dir: str):
    """Inject custom router include into generated main_api.py."""
    api_file = os.path.join(output_dir, "main_api.py")

    with open(api_file, "r") as f:
        content = f.read()

    # Only patch if not already patched
    if "custom_routes" not in content:
        # Insert after the CORS middleware block
        marker = "app.add_middleware("
        if marker in content:
            idx = content.index(marker)
            # Find the end of the middleware block
            closing = content.index(")", idx)
            next_newline = content.index("\n", closing)
            content = (
                content[:next_newline + 1]
                + "\n# Include custom routes (not overwritten by generator)\n"
                + "from custom_routes import router as custom_router\n"
                + "app.include_router(custom_router)\n"
                + content[next_newline + 1:]
            )
        else:
            # Fallback: insert after app creation
            content = content.replace(
                "app = FastAPI(",
                "# Custom routes will be included after app creation\napp = FastAPI("
            )

        with open(api_file, "w") as f:
            f.write(content)


# Usage: call after generation
from besser.generators.backend import BackendGenerator

backend = BackendGenerator(model=my_model, output_dir="./my_backend")
backend.generate()
patch_main_api("./my_backend")
```

---

## Strategy 3: Create a Wrapper Application (Fully Safe)

The most robust approach for complex projects is to create a wrapper application that imports the generated code and extends it. This approach survives regeneration with **zero patching required**.

Create `my_backend/app.py` (your custom entry point, never overwritten):

```python
"""
Custom application entry point.
Imports the generated FastAPI app and extends it with custom logic.
This file is NOT generated and is safe to edit.
"""
import uvicorn
from main_api import app, get_db  # Import the generated app and DB session
from fastapi import Depends, HTTPException
from sqlalchemy.orm import Session
from sql_alchemy import *
from pydantic import BaseModel, validator
from typing import Optional


# ===========================================================
#   Custom Pydantic models for validation
# ===========================================================

class ProductValidation(BaseModel):
    name: str
    price: float
    stock: int

    @validator("price")
    def price_must_be_positive(cls, v):
        if v <= 0:
            raise ValueError("Price must be positive")
        return round(v, 2)

    @validator("stock")
    def stock_must_be_non_negative(cls, v):
        if v < 0:
            raise ValueError("Stock cannot be negative")
        return v


# ===========================================================
#   Custom Middleware
# ===========================================================

@app.middleware("http")
async def custom_auth_check(request, call_next):
    """Example: add API key validation."""
    if request.url.path.startswith("/admin/"):
        api_key = request.headers.get("X-API-Key")
        if api_key != "your-secret-key":
            from fastapi.responses import JSONResponse
            return JSONResponse(
                status_code=401,
                content={"error": "Unauthorized", "detail": "Invalid API key"}
            )
    response = await call_next(request)
    return response


# ===========================================================
#   Custom Endpoints
# ===========================================================

@app.post("/admin/validate-product", tags=["Admin"])
def validate_product(data: ProductValidation):
    """Validate product data before creation with custom rules."""
    return {"valid": True, "data": data.dict()}


@app.get("/reports/inventory-summary", tags=["Reports"])
def inventory_summary(database: Session = Depends(get_db)):
    """Custom report: summarize inventory across all products."""
    products = database.query(Product).all()
    total_value = sum(p.price * p.stock for p in products)
    low_stock = [p for p in products if p.stock < 10]
    return {
        "total_products": len(products),
        "total_inventory_value": round(total_value, 2),
        "low_stock_items": [{"id": p.id, "name": p.name, "stock": p.stock} for p in low_stock]
    }


# ===========================================================
#   Override the entry point
# ===========================================================

if __name__ == "__main__":
    # Run YOUR app.py instead of main_api.py
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

Then run `python app.py` instead of `python main_api.py`. Since `app.py` imports the `app` object from `main_api.py`, all generated endpoints are available, plus your custom ones. When you regenerate, `main_api.py` gets overwritten but `app.py` remains untouched.

---

## Strategy 4: Subclass BackendGenerator (Advanced)

For teams that need to consistently customize the generated output, you can subclass `BackendGenerator` and override behavior. This is the approach for organizations building on top of BESSER as a platform.

```python
import os
from besser.generators.backend import BackendGenerator
from besser.BUML.metamodel.structural import DomainModel


class CustomBackendGenerator(BackendGenerator):
    """Extended backend generator that appends custom routes."""

    def __init__(self, model: DomainModel, custom_routes_file: str = None, **kwargs):
        super().__init__(model, **kwargs)
        self.custom_routes_file = custom_routes_file

    def generate(self):
        # Run the standard generation
        super().generate()

        output_dir = self.output_dir or os.path.join(os.path.abspath(''), "output_backend")

        # Append custom router include to main_api.py
        if self.custom_routes_file and os.path.exists(self.custom_routes_file):
            api_file = os.path.join(output_dir, "main_api.py")

            with open(api_file, "r") as f:
                content = f.read()

            if "custom_routes" not in content:
                # Insert the router include before the __main__ block
                content = content.replace(
                    'if __name__ == "__main__":',
                    '# Include custom routes\n'
                    'from custom_routes import router as custom_router\n'
                    'app.include_router(custom_router)\n\n'
                    'if __name__ == "__main__":'
                )
                with open(api_file, "w") as f:
                    f.write(content)

            # Copy custom routes file to output
            import shutil
            shutil.copy2(
                self.custom_routes_file,
                os.path.join(output_dir, "custom_routes.py")
            )

        print(f"Custom backend generated at {output_dir}")


# Usage
generator = CustomBackendGenerator(
    model=my_model,
    custom_routes_file="./my_custom_routes.py",
    http_methods=["GET", "POST", "PUT", "DELETE"],
    output_dir="./my_backend"
)
generator.generate()
```

---

## Summary: Which Approach Should You Use?

| Approach | Survives Regen? | Complexity | Best For |
|----------|----------------|------------|----------|
| **BAL/CODE methods in model** | Yes (it IS the model) | Low | Validation, field updates, simple business logic |
| **FastAPI `include_router`** | Custom file: yes; needs 1-line patch | Medium | Complex custom endpoints, external integrations |
| **Wrapper `app.py`** | Fully safe, zero patching | Medium | Production deployments with custom middleware/auth |
| **Subclass `BackendGenerator`** | Fully automated | High | Teams building reusable generation pipelines |

**The recommended path** is to start with Strategy 1 (model-level methods) for logic that is intrinsic to your domain. For anything beyond what the model can express -- custom authentication, third-party integrations, complex reports -- use Strategy 3 (wrapper application). This gives you the best balance of model-driven generation and practical extensibility.
