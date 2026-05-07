# Modeling a University in BESSER and Generating a SQLAlchemy ORM

In BESSER, you describe your domain as a **B-UML model** (the source of truth) and then run a **generator** that produces the code for your target platform. For your university system, you'll define three classes (`Student`, `Course`, `Professor`), wire them up with two associations (enrollment and teaching), and then hand the model to the `SQLAlchemyGenerator`.

## 1. Install BESSER

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

pip install besser
```

Python 3.10+ is required. Quick sanity check:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"
```

## 2. Build the B-UML model

Create a file `university_model.py`:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)

# --- Student ---
student_name = Property(name="name", type=StringType)
student_id   = Property(name="student_id", type=IntegerType)
student = Class(name="Student", attributes={student_name, student_id})

# --- Course ---
course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course = Class(name="Course", attributes={course_name, course_credits})

# --- Professor ---
prof_name = Property(name="name", type=StringType)
prof_id   = Property(name="prof_id", type=IntegerType)
professor = Class(name="Professor", attributes={prof_name, prof_id})

# --- Association: Student *..* Course (enrollment) ---
enrolled_in = Property(name="enrolledIn", type=course,
                       multiplicity=Multiplicity(0, "*"))
students    = Property(name="students",   type=student,
                       multiplicity=Multiplicity(0, "*"))
enrollment = BinaryAssociation(name="enrollment",
                               ends={enrolled_in, students})

# --- Association: Professor 1 -- 0..* Course (teaching) ---
teaches      = Property(name="teaches",   type=course,
                        multiplicity=Multiplicity(0, "*"))
taught_by    = Property(name="taughtBy",  type=professor,
                        multiplicity=Multiplicity(1, 1))
teaching = BinaryAssociation(name="teaching",
                             ends={teaches, taught_by})

# --- Assemble the domain model ---
model = DomainModel(
    name="University",
    types={student, course, professor},
    associations={enrollment, teaching},
)

assert model.validate()["success"], "Model failed validation"
```

A few things to note:

- **Naming rules**: `PascalCase` for classes, `snake_case` (or camelCase) for attributes/roles, no spaces or hyphens.
- **Multiplicities**: `Multiplicity(0, "*")` for many, `Multiplicity(1, 1)` for exactly one. Use `"*"` (a string) for the unbounded upper end.
- **Associations are symmetric in declaration**: each `BinaryAssociation` has two `ends`, one Property per end, and each end's `type` is the *opposite* class (the role-name property points to the other side).
- The "many-to-many" between `Student` and `Course` produces a join table automatically when you generate the ORM. The "one-to-many" between `Professor` and `Course` produces a foreign key on the `Course` table.
- Always call `model.validate()` before generating.

> If you'd rather sketch the model in PlantUML first, BESSER can convert it: `from besser.BUML.notations.structuralPlantUML import plantuml_to_buml`. Or use the visual editor at https://editor.besser-pearl.org.

## 3. Generate the SQLAlchemy ORM

Add this to the same file (or a separate runner script):

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

gen = SQLAlchemyGenerator(model=model, output_dir="./output")
gen.generate(dbms="postgresql")
# Valid dbms values: sqlite | postgresql | mysql | mssql | mariadb | oracle
# (note: it is "postgresql", not "postgres")
```

Run it:

```bash
python university_model.py
```

You'll get `./output/sql_alchemy.py` containing SQLAlchemy `Mapped`/`Column` declarations for `Student`, `Course`, `Professor`, plus the join table for the student-course many-to-many and a `professor_id` foreign key on `Course`.

If you don't yet have a database picked, use `dbms="sqlite"` to try it locally with zero setup.

## 4. Verify

After generation:

1. **File exists**: confirm `./output/sql_alchemy.py` is there.
2. **Syntax parses**:
   ```bash
   python -c "import ast; ast.parse(open('output/sql_alchemy.py').read())"
   ```
3. **Relationships look right**: open the file and check that
   - `Course` has a `professor_id` FK and a `professor` relationship,
   - There's an association table (something like `enrollment`) joining `student` and `course`,
   - The many-to-many on `Student` and `Course` uses `secondary=`.

## 5. Iterating

The model is the source of truth. When requirements change — say you want to add a `Department` class, or give `Course` a `semester` attribute — **edit `university_model.py` and regenerate**. Don't hand-edit `sql_alchemy.py`; every call to `generate()` overwrites the output directory.

## Bonus: also want a REST API?

The same `model` object feeds any generator. To get a full FastAPI backend (Pydantic + SQLAlchemy + endpoints) in one shot:

```python
from besser.generators.backend import BackendGenerator

BackendGenerator(
    model=model,
    output_dir="./output_backend",
    http_methods=["GET", "POST", "PUT", "DELETE"],
).generate()
# Produces: main_api.py, pydantic_classes.py, sql_alchemy.py
```

That's the BESSER way: one model, many targets.
