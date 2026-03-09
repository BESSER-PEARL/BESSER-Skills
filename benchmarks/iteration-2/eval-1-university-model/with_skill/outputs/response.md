# Modeling a University System in BESSER and Generating SQLAlchemy ORM

BESSER follows a model-first workflow: you define your domain as a B-UML model in Python, then run a generator to produce executable code. Here is how to model your university system (Students, Courses, Professors) and generate a SQLAlchemy ORM from it.

---

## Step 1: Install BESSER

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

pip install besser
```

Verify the installation:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"
```

---

## Step 2: Define the Domain Model

Create a file called `university_model.py` with the following content:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)

# -------------------------------------------------------------------
# 1. Define Classes and Their Attributes
# -------------------------------------------------------------------

# Student class
student_name = Property(name="name", type=StringType)
student_id   = Property(name="student_id", type=IntegerType, is_id=True)
student_email = Property(name="email", type=StringType)
student = Class(name="Student", attributes={student_id, student_name, student_email})

# Course class
course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course_code    = Property(name="code", type=StringType, is_id=True)
course = Class(name="Course", attributes={course_code, course_name, course_credits})

# Professor class
prof_name   = Property(name="name", type=StringType)
prof_id     = Property(name="professor_id", type=IntegerType, is_id=True)
prof_email  = Property(name="email", type=StringType)
professor = Class(name="Professor", attributes={prof_id, prof_name, prof_email})

# -------------------------------------------------------------------
# 2. Define Associations
# -------------------------------------------------------------------

# Student enrolls in many Courses (many-to-many)
# A student can enroll in 0 or more courses.
# A course can have 0 or more students enrolled.
enrolls_in    = Property(name="enrolls_in",    type=course,  multiplicity=Multiplicity(0, "*"))
enrolled_by   = Property(name="enrolled_by",   type=student, multiplicity=Multiplicity(0, "*"))
enrollment    = BinaryAssociation(name="Enrollment", ends={enrolls_in, enrolled_by})

# Professor teaches Courses (one-to-many)
# A professor can teach 0 or more courses.
# Each course is taught by exactly 1 professor.
teaches      = Property(name="teaches",    type=course,    multiplicity=Multiplicity(0, "*"))
taught_by    = Property(name="taught_by",  type=professor, multiplicity=Multiplicity(1, 1))
teaching     = BinaryAssociation(name="Teaching", ends={teaches, taught_by})

# -------------------------------------------------------------------
# 3. Assemble the Model
# -------------------------------------------------------------------

university_model = DomainModel(
    name="University_model",
    types={student, course, professor},
    associations={enrollment, teaching},
)

# -------------------------------------------------------------------
# 4. Validate the Model
# -------------------------------------------------------------------

result = university_model.validate()
print("Validation result:", result)
```

### What this model captures

- **Student** has a `student_id` (primary key), `name`, and `email`.
- **Course** has a `code` (primary key), `name`, and `credits`.
- **Professor** has a `professor_id` (primary key), `name`, and `email`.
- **Enrollment** is a many-to-many association between Student and Course (`0..*` on both sides).
- **Teaching** is a one-to-many association: one Professor teaches many Courses, each Course is taught by exactly one Professor (`1..1` on the professor side, `0..*` on the course side).

---

## Step 3: Generate the SQLAlchemy ORM

Add the following lines at the bottom of `university_model.py` (or create a separate script that imports the model):

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# Generate SQLAlchemy ORM code
generator = SQLAlchemyGenerator(model=university_model, output_dir="./output")
generator.generate()
print("SQLAlchemy ORM code generated in ./output/sql_alchemy.py")
```

You can also specify a target database management system:

```python
# For PostgreSQL:
generator.generate(dbms="postgresql")

# For MySQL:
generator.generate(dbms="mysql")

# For SQLite (default):
generator.generate(dbms="sqlite")
```

---

## Step 4: Run It

```bash
python university_model.py
```

This will:

1. Validate the domain model and print the validation result.
2. Generate a file at `./output/sql_alchemy.py` containing SQLAlchemy ORM classes for `Student`, `Course`, and `Professor`, including the many-to-many join table for Enrollment and the foreign key for Teaching.

---

## Step 5: Verify the Output

After generation, check that the output file exists and is syntactically valid:

```bash
# Check the file exists
ls ./output/sql_alchemy.py

# Verify valid Python syntax
python -c "import ast; ast.parse(open('./output/sql_alchemy.py').read()); print('Syntax OK')"
```

You can inspect the generated file to verify:

- A `Student` table with columns `student_id`, `name`, `email`.
- A `Course` table with columns `code`, `name`, `credits`.
- A `Professor` table with columns `professor_id`, `name`, `email`.
- A many-to-many association table linking `Student` and `Course` (for enrollment).
- A foreign key on `Course` pointing to `Professor` (for teaching).

---

## Complete Script

Here is the full, self-contained script you can copy and run:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# ---- Classes ----
student_name  = Property(name="name", type=StringType)
student_id    = Property(name="student_id", type=IntegerType, is_id=True)
student_email = Property(name="email", type=StringType)
student = Class(name="Student", attributes={student_id, student_name, student_email})

course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course_code    = Property(name="code", type=StringType, is_id=True)
course = Class(name="Course", attributes={course_code, course_name, course_credits})

prof_name  = Property(name="name", type=StringType)
prof_id    = Property(name="professor_id", type=IntegerType, is_id=True)
prof_email = Property(name="email", type=StringType)
professor = Class(name="Professor", attributes={prof_id, prof_name, prof_email})

# ---- Associations ----
# Many-to-many: Student <-> Course (enrollment)
enrolls_in  = Property(name="enrolls_in",  type=course,  multiplicity=Multiplicity(0, "*"))
enrolled_by = Property(name="enrolled_by", type=student, multiplicity=Multiplicity(0, "*"))
enrollment  = BinaryAssociation(name="Enrollment", ends={enrolls_in, enrolled_by})

# One-to-many: Professor -> Course (teaching)
teaches  = Property(name="teaches",  type=course,    multiplicity=Multiplicity(0, "*"))
taught_by = Property(name="taught_by", type=professor, multiplicity=Multiplicity(1, 1))
teaching = BinaryAssociation(name="Teaching", ends={teaches, taught_by})

# ---- Model ----
university_model = DomainModel(
    name="University_model",
    types={student, course, professor},
    associations={enrollment, teaching},
)

# ---- Validate ----
result = university_model.validate()
print("Validation result:", result)

# ---- Generate SQLAlchemy ORM ----
generator = SQLAlchemyGenerator(model=university_model, output_dir="./output")
generator.generate()
print("SQLAlchemy ORM generated at ./output/sql_alchemy.py")
```

---

## Going Further

The same `university_model` can be used with other generators without any changes:

| Goal | Generator | Command |
|------|-----------|---------|
| Plain Python classes | `PythonGenerator` | `PythonGenerator(model=university_model, output_dir="./output").generate()` |
| Pydantic models | `PydanticGenerator` | `PydanticGenerator(model=university_model, output_dir="./output").generate()` |
| Raw SQL DDL | `SQLGenerator` | `SQLGenerator(model=university_model, output_dir="./output").generate()` |
| FastAPI REST API | `BackendGenerator` | `BackendGenerator(model=university_model, output_dir="./output").generate()` |
| Django project | `DjangoGenerator` | `DjangoGenerator(model=university_model, project_name="university", app_name="core", output_dir="./output").generate()` |

This is the core principle of BESSER: **one model, many targets**. When your requirements change, update the model and regenerate -- never hand-edit generated files as your primary change.
