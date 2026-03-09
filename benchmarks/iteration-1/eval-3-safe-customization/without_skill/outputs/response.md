# Will My Changes to `main_api.py` Survive the Next Generation?

**No. Any manual edits to `main_api.py` will be overwritten the next time you run `BackendGenerator.generate()`.** The same applies to `pydantic_classes.py` and `sql_alchemy.py` -- all three files are regenerated from scratch every time.

Here is exactly why: the `BackendGenerator` (at `besser/generators/backend/backend_generator.py`) delegates to three sub-generators that each write their output files unconditionally:

```python
# From BackendGenerator.generate() (lines 85-92)
rest_api = RESTAPIGenerator(model=self.model, ..., output_dir=backend_folder_path, backend=True, ...)
rest_api.generate()  # Overwrites main_api.py

sql_alchemy = SQLAlchemyGenerator(model=self.model, output_dir=backend_folder_path)
sql_alchemy.generate()  # Overwrites sql_alchemy.py

pydantic_model = PydanticGenerator(model=self.model, output_dir=backend_folder_path, backend=True, ...)
pydantic_model.generate()  # Overwrites pydantic_classes.py
```

The `RESTAPIGenerator` renders `main_api.py` from the Jinja2 template `backend_fast_api_template.py.j2` using `mode="w"` (truncate and write), so the entire file is replaced on every generation.

---

## The Right Way to Add Custom Endpoints and Validation Logic

Since BESSER currently does not include a built-in mechanism for merging custom code with generated code (there is no `include_router`, plugin hook, or protected region pattern in the templates), you have several practical strategies available, ordered from best to most invasive.

---

### Strategy 1: Use a Separate FastAPI Router File (Recommended)

Create a **separate file** for your custom endpoints and validation, then import and mount it in `main_api.py` after generation. Since `main_api.py` will be overwritten, you automate the import injection with a small post-generation script.

**Step 1: Create your custom router file (never overwritten)**

Create a file called `custom_routes.py` alongside the generated files:

```python
# custom_routes.py -- YOUR custom code, not generated, never overwritten
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from pydantic_classes import *
from sql_alchemy import *

router = APIRouter(prefix="/custom", tags=["Custom"])

@router.post("/validate-order/")
def validate_order(order_data: dict, database: Session = Depends(get_db)):
    """Custom validation logic that is not part of the generated CRUD."""
    errors = []
    if order_data.get("quantity", 0) <= 0:
        errors.append("Quantity must be positive")
    if order_data.get("total_price", 0) < 0:
        errors.append("Total price cannot be negative")
    if errors:
        raise HTTPException(status_code=422, detail={"validation_errors": errors})
    return {"status": "valid"}

@router.get("/dashboard-stats/")
def get_dashboard_stats(database: Session = Depends(get_db)):
    """Custom endpoint for aggregated statistics."""
    # Your custom query logic here
    return {"total_orders": database.query(Order).count()}
```

Note: `get_db` is defined in the generated `main_api.py`. You will need to either import it from there or duplicate the session dependency in your custom file.

**Step 2: Inject the router import after generation**

Write a small post-processing script that appends the router include to `main_api.py`:

```python
# post_generate.py
import re

def inject_custom_router(api_file_path: str):
    with open(api_file_path, "r") as f:
        content = f.read()

    # Add import at the top (after existing imports)
    import_line = "from custom_routes import router as custom_router\n"
    if import_line not in content:
        # Insert after the last 'from' import line
        content = content.replace(
            "from sql_alchemy import *\n",
            "from sql_alchemy import *\n" + import_line
        )

    # Add router include before the __main__ block
    include_line = "app.include_router(custom_router)\n"
    if include_line not in content:
        content = content.replace(
            'if __name__ == "__main__":',
            include_line + '\nif __name__ == "__main__":'
        )

    with open(api_file_path, "w") as f:
        f.write(content)

if __name__ == "__main__":
    inject_custom_router("output_backend/main_api.py")
```

**Step 3: Run generation + post-processing together**

```python
from besser.generators.backend import BackendGenerator

generator = BackendGenerator(model=my_model, output_dir="output_backend")
generator.generate()

# Then run the post-processing
from post_generate import inject_custom_router
inject_custom_router("output_backend/main_api.py")
```

This keeps your custom logic safe in `custom_routes.py` while letting BESSER regenerate the CRUD layer freely.

---

### Strategy 2: Use OCL Constraints for Validation (Model-Level Approach)

If your validation logic is about **constraining attribute values** (e.g., "age must be >= 18", "price must be positive"), BESSER already supports this through OCL constraints on the domain model. These constraints get rendered as Pydantic `@field_validator` decorators in `pydantic_classes.py`, which means they are applied automatically during POST and PUT operations.

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, IntegerType, StringType,
    Constraint
)

# Define your class
order = Class(name="Order", attributes={
    Property(name="quantity", type=IntegerType),
    Property(name="total_price", type=IntegerType),
})

# Add OCL constraints for validation
quantity_constraint = Constraint(
    name="positive_quantity",
    context=order,
    expression="self.quantity > 0",
    language="OCL"
)

model = DomainModel(
    name="MyApp",
    types={order},
    constraints={quantity_constraint}
)
```

The generated `pydantic_classes.py` will include:

```python
class OrderCreate(BaseModel):
    quantity: int
    total_price: int

    @field_validator('quantity')
    @classmethod
    def validate_quantity_1(cls, v):
        """OCL Constraint: positive_quantity"""
        if not (v > 0):
            raise ValueError('quantity must be > 0')
        return v
```

This approach survives regeneration because the validation rules are embedded in the model itself. However, it is limited to field-level value constraints and does not support cross-field validation or complex business logic.

---

### Strategy 3: Use Methods with Code Implementation (BAL or Python)

BESSER supports defining **methods with executable code** on your domain model classes. These methods are rendered as API endpoints (POST) in the generated `main_api.py`. You can use this to embed validation or business logic that gets regenerated correctly each time.

```python
from besser.BUML.metamodel.structural import (
    Class, Method, Parameter, StringType, IntegerType,
    MethodImplementationType
)

validate_method = Method(
    name="validate_order",
    parameters=set(),
    type=StringType,
    code="""def validate_order(self):
    errors = []
    if self.quantity <= 0:
        errors.append("Quantity must be positive")
    if self.total_price < 0:
        errors.append("Price cannot be negative")
    if errors:
        raise ValueError("; ".join(errors))
    return "valid"
""",
    implementation_type=MethodImplementationType.CODE
)

order = Class(name="Order", attributes={...}, methods={validate_method})
```

This generates a `POST /order/{order_id}/methods/validate_order/` endpoint in `main_api.py`. The method code is embedded in the Jinja2 template output and will persist across regenerations since it comes from the model.

The limitation here is that methods operate on existing entity instances (they require an entity ID), so they are not suitable for pre-creation validation. They are better suited for business operations on already-persisted entities.

---

### Strategy 4: Subclass BackendGenerator (Advanced)

If you need to systematically customize the generated output across multiple projects, you can subclass `BackendGenerator` and override the `generate()` method to add post-processing or use a custom template:

```python
import os
from besser.generators.backend import BackendGenerator

class CustomBackendGenerator(BackendGenerator):
    def __init__(self, model, custom_routes_file=None, **kwargs):
        super().__init__(model, **kwargs)
        self.custom_routes_file = custom_routes_file

    def generate(self):
        # Run standard generation
        super().generate()

        # Post-process: inject custom router
        output_dir = self.output_dir or "output_backend"
        api_path = os.path.join(output_dir, "main_api.py")

        if self.custom_routes_file and os.path.exists(self.custom_routes_file):
            # Copy custom routes file to output directory
            import shutil
            shutil.copy2(self.custom_routes_file, os.path.join(output_dir, "custom_routes.py"))

            # Inject router include into main_api.py
            with open(api_path, "r") as f:
                content = f.read()

            import_line = "from custom_routes import router as custom_router\n"
            include_line = "app.include_router(custom_router)\n"

            if import_line not in content:
                content = content.replace(
                    "from sql_alchemy import *\n",
                    "from sql_alchemy import *\n" + import_line
                )
            if include_line not in content:
                content = content.replace(
                    'if __name__ == "__main__":',
                    include_line + '\nif __name__ == "__main__":'
                )

            with open(api_path, "w") as f:
                f.write(content)
```

Usage:

```python
gen = CustomBackendGenerator(
    model=my_model,
    output_dir="output_backend",
    custom_routes_file="my_custom_routes.py"
)
gen.generate()
```

---

### Strategy 5: Modify the Jinja2 Template (Most Invasive)

If you want all generated backends to include a hook for custom routes by default, you can modify the template at:

```
besser/generators/rest_api/templates/backend_fast_api_template.py.j2
```

For example, you could add near the end of the template (before the `if __name__` block):

```jinja2
############################################
# Custom extensions
############################################
import importlib, os
if os.path.exists(os.path.join(os.path.dirname(__file__), "custom_routes.py")):
    from custom_routes import router as custom_router
    app.include_router(custom_router)
```

**Warning**: Modifying the template affects all users of the generator and would need to be maintained through BESSER updates. This approach is only recommended if you are contributing the feature upstream to the BESSER project.

---

## Summary and Recommendation

| Strategy | Survives Regeneration | Complexity | Best For |
|---|---|---|---|
| Separate router file + post-processing | Yes | Low | Custom endpoints, complex validation |
| OCL Constraints | Yes (in model) | Low | Simple field-level value validation |
| Methods with code | Yes (in model) | Medium | Business operations on existing entities |
| Subclass BackendGenerator | Yes | Medium | Repeated customization across projects |
| Modify Jinja2 template | Yes | High | Contributing to BESSER itself |

**The recommended approach is Strategy 1** (separate router file with post-generation injection). It cleanly separates generated code from custom code, survives regeneration with minimal scripting, and follows FastAPI best practices with `APIRouter`. For simple attribute constraints, combine it with Strategy 2 (OCL Constraints) to get the best of both worlds.

The key principle is: **never edit generated files by hand**. Either put your logic in the model (so it gets regenerated correctly) or keep it in separate files that the generator does not touch.
