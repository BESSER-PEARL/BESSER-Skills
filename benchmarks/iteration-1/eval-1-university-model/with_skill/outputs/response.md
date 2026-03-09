# Modeling a University System in BESSER and Generating SQLAlchemy ORM

BESSER follows a model-first workflow: you define your domain as a B-UML model in Python, then run a generator to produce code. Here is how to model your university system with Students, Courses, and Professors, and then generate a SQLAlchemy ORM from it.

---

## Step 1: Define the B-UML Domain Model

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)

# -----------------------------------------------------------
# Classes
# -----------------------------------------------------------

# Student
student_name = Property(name="name", type=StringType)
student_id   = Property(name="student_id", type=IntegerType, is_id=True)
student_email = Property(name="email", type=StringType)
student = Class(name="Student", attributes={student_id, student_name, student_email})

# Course
course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course_code    = Property(name="code", type=StringType, is_id=True)
course = Class(name="Course", attributes={course_code, course_name, course_credits})

# Professor
prof_name   = Property(name="name", type=StringType)
prof_id     = Property(name="professor_id", type=IntegerType, is_id=True)
prof_department = Property(name="department", type=StringType)
professor = Class(name="Professor", attributes={prof_id, prof_name, prof_department})

# -----------------------------------------------------------
# Associations
# -----------------------------------------------------------

# Student enrolls in many Courses (many-to-many)
enrolls_in   = Property(name="enrollsIn",   type=course,  multiplicity=Multiplicity(0, "*"))
has_students = Property(name="hasStudents",  type=student, multiplicity=Multiplicity(0, "*"))
enrollment = BinaryAssociation(name="Enrollment", ends={enrolls_in, has_students})

# Professor teaches Courses (one-to-many: one professor teaches many courses)
teaches      = Property(name="teaches",  type=course,    multiplicity=Multiplicity(1, "*"))
taught_by    = Property(name="taughtBy", type=professor,  multiplicity=Multiplicity(1, 1))
teaching = BinaryAssociation(name="Teaching", ends={teaches, taught_by})

# -----------------------------------------------------------
# Assemble the Domain Model
# -----------------------------------------------------------
university_model = DomainModel(
    name="University",
    types={student, course, professor},
    associations={enrollment, teaching},
)
```

### What this model captures

| Relationship | Meaning | Multiplicity |
|---|---|---|
| Student -- Course (Enrollment) | A student can enroll in zero or more courses; a course can have zero or more students | many-to-many (`0..*` on both sides) |
| Professor -- Course (Teaching) | A professor teaches one or more courses; each course is taught by exactly one professor | one-to-many (`1..*` on the professor side, `1..1` on the course side) |

### Validate the model (optional but recommended)

```python
result = university_model.validate()
print(result)
# Expected: {"success": True, "errors": [], "warnings": [...]}
```

---

## Step 2: Generate SQLAlchemy ORM Code

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

generator = SQLAlchemyGenerator(model=university_model, output_dir="./output")
generator.generate()
# Output: ./output/sql_alchemy.py
```

This produces a file called `sql_alchemy.py` in the `./output` directory containing:

- SQLAlchemy `Base` declarative model classes for `Student`, `Course`, and `Professor`
- Columns mapped from your B-UML attributes (e.g., `name` becomes `Column(String(100))`, `credits` becomes `Column(Integer)`)
- A many-to-many association table for the `Enrollment` relationship between `Student` and `Course`
- A foreign key on `Course` pointing to `Professor` for the `Teaching` one-to-many relationship
- An SQLite engine and session setup by default

### Targeting a different database

Pass the `dbms` parameter to `generate()` to target a different database system:

```python
generator.generate(dbms="postgresql")   # PostgreSQL
generator.generate(dbms="mysql")        # MySQL
generator.generate(dbms="sqlite")       # SQLite (default)
```

Valid options are: `sqlite`, `postgresql`, `mysql`, `mssql`, `mariadb`, `oracle`.

---

## Complete Script

Here is the full script you can save as `university.py` and run:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# -----------------------------------------------------------
# Classes
# -----------------------------------------------------------

student_name  = Property(name="name", type=StringType)
student_id    = Property(name="student_id", type=IntegerType, is_id=True)
student_email = Property(name="email", type=StringType)
student = Class(name="Student", attributes={student_id, student_name, student_email})

course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course_code    = Property(name="code", type=StringType, is_id=True)
course = Class(name="Course", attributes={course_code, course_name, course_credits})

prof_name       = Property(name="name", type=StringType)
prof_id         = Property(name="professor_id", type=IntegerType, is_id=True)
prof_department = Property(name="department", type=StringType)
professor = Class(name="Professor", attributes={prof_id, prof_name, prof_department})

# -----------------------------------------------------------
# Associations
# -----------------------------------------------------------

# Many-to-many: Student <-> Course
enrolls_in   = Property(name="enrollsIn",  type=course,  multiplicity=Multiplicity(0, "*"))
has_students = Property(name="hasStudents", type=student, multiplicity=Multiplicity(0, "*"))
enrollment = BinaryAssociation(name="Enrollment", ends={enrolls_in, has_students})

# One-to-many: Professor -> Course
teaches   = Property(name="teaches",  type=course,    multiplicity=Multiplicity(1, "*"))
taught_by = Property(name="taughtBy", type=professor, multiplicity=Multiplicity(1, 1))
teaching = BinaryAssociation(name="Teaching", ends={teaches, taught_by})

# -----------------------------------------------------------
# Domain Model
# -----------------------------------------------------------

university_model = DomainModel(
    name="University",
    types={student, course, professor},
    associations={enrollment, teaching},
)

# Validate
result = university_model.validate()
print("Validation:", result)

# -----------------------------------------------------------
# Generate SQLAlchemy ORM
# -----------------------------------------------------------

generator = SQLAlchemyGenerator(model=university_model, output_dir="./output")
generator.generate()
print("SQLAlchemy ORM generated at ./output/sql_alchemy.py")
```

---

## What to Expect in the Generated Output

The generated `sql_alchemy.py` file will contain:

1. **Imports**: SQLAlchemy's `Column`, `Integer`, `String`, `ForeignKey`, `Table`, `create_engine`, `sessionmaker`, etc.
2. **Association table**: An `enrollment` table for the many-to-many relationship between `Student` and `Course`.
3. **ORM classes**:
   - `Student` with columns `student_id` (primary key), `name`, and `email`, plus a `relationship()` to `Course` via the enrollment table.
   - `Course` with columns `code` (primary key), `name`, and `credits`, plus a foreign key to `Professor` and relationships to both `Student` and `Professor`.
   - `Professor` with columns `professor_id` (primary key), `name`, and `department`, plus a `relationship()` to `Course`.
4. **Engine and session**: Pre-configured database engine and session factory.

---

## Next Steps

- **Need a full REST API?** Use `BackendGenerator` instead -- it generates a FastAPI application with SQLAlchemy ORM and Pydantic schemas in one shot.
- **Need raw SQL?** Use `SQLGenerator` to get a `tables.sql` DDL file.
- **Want to iterate?** Update the model (add attributes, change multiplicities, add new classes) and re-run the generator. The output file is overwritten each time -- never hand-edit generated code as your primary change.
