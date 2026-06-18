I have everything verified against the actual v7.8.3 source. Here is the complete answer.

---

# Modeling a University System in BESSER and Generating a SQLAlchemy ORM

BESSER lets you build a structural model (a UML class model, called B-UML) in plain Python, then run code generators against it. Below is the full workflow for your university domain.

## 1. The model in BESSER

In BESSER, you build the model from these building blocks (all importable from `besser.BUML.metamodel.structural`):

- **`Class`** — your entities (Student, Course, Professor).
- **`Property`** — used for two things: a class **attribute** (e.g. `name`, `credits`), and an **association end** (the role a class plays in a relationship).
- **`Multiplicity(min, max)`** — cardinality of an association end. Use `"*"` for "many" (unlimited).
- **`BinaryAssociation`** — a relationship between exactly two classes, defined by its two ends.
- **`DomainModel`** — the container holding all classes (`types=`) and associations (`associations=`).

Attribute types can be passed either as the type objects (`StringType`, `IntegerType`, ...) or as the string shortcuts (`"str"`, `"int"`, `"float"`, `"bool"`, `"date"`, `"datetime"`, `"time"`). Both work.

Note on **many-to-many** (Students enroll in many courses, and a course has many students): you model it as a single `BinaryAssociation` where **both** ends have a `*` max multiplicity. The SQLAlchemy generator automatically emits a join table for that case — you do not create a join class yourself.

### `university.py`

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# --- Student ---
student_name: Property = Property(name="name", type=StringType)
student: Class = Class(name="Student", attributes={student_name})

# --- Course ---
course_name: Property = Property(name="name", type=StringType)
credits: Property = Property(name="credits", type=IntegerType)
course: Class = Class(name="Course", attributes={course_name, credits})

# --- Professor ---
professor_name: Property = Property(name="name", type=StringType)
professor: Class = Class(name="Professor", attributes={professor_name})

# --- Association: Student *--* Course (many-to-many enrollment) ---
enrolled_courses: Property = Property(
    name="enrolledCourses", type=course, multiplicity=Multiplicity(0, "*")
)
students: Property = Property(
    name="students", type=student, multiplicity=Multiplicity(0, "*")
)
enrollment: BinaryAssociation = BinaryAssociation(
    name="enrollment", ends={enrolled_courses, students}
)

# --- Association: Professor 1--* Course (a professor teaches many courses) ---
taught_courses: Property = Property(
    name="taughtCourses", type=course, multiplicity=Multiplicity(0, "*")
)
taught_by: Property = Property(
    name="taughtBy", type=professor, multiplicity=Multiplicity(1, 1)
)
teaching: BinaryAssociation = BinaryAssociation(
    name="teaching", ends={taught_by, taught_courses}
)

# --- Domain model ---
university_model: DomainModel = DomainModel(
    name="University",
    types={student, course, professor},
    associations={enrollment, teaching},
)

# --- Generate the SQLAlchemy ORM ---
generator = SQLAlchemyGenerator(model=university_model, output_dir="output")
generator.generate()   # default DBMS is "sqlite"
```

### Important conventions (from the actual generator code)

- **Primary keys are automatic.** You do not need to declare an `id`. The generator gives every class an `id: Mapped[int] = mapped_column(primary_key=True)` unless you explicitly mark one of your own attributes with `Property(..., is_id=True)`. So you can leave `Student`/`Course`/`Professor` without an explicit id and you'll get one for free.
- **Reading association ends.** In a `BinaryAssociation`, an end's `type` is the class it *points to*, and its `multiplicity` is how many of that target are reachable. So for "a professor teaches many courses": the `taughtCourses` end points to `Course` with `(0, *)`, and the `taughtBy` end points to `Professor` with `(1, 1)` (each course has exactly one professor). This produces a one-to-many.
- **Reserved names.** Do not name classes/attributes/associations any of: `Base`, `Enum`, `enum`, or the underscore-suffixed aliases (`String_`, `Integer_`, `Table_`, `List_`, etc.). The generator calls `validate_model()` and raises `ValueError` if you do.
- **`str` maps to `String(100)`** by default in the generated columns.

## 2. Running it

BESSER is a Python library, so just run the script:

```bash
pip install besser            # if not already installed
python university.py
```

You'll see `Code generated successfully!` printed. The output is written to a file named **`sql_alchemy.py`**. Because the example passes `output_dir="output"`, it lands at `output/sql_alchemy.py`. If you omit `output_dir`, the generator writes to an `output/` folder under the current directory by default.

## 3. What gets generated

For your model you'll get something equivalent to this (mirroring the real generated structure — note the auto-generated `id` PKs, the auto join table for the many-to-many, the FK + `relationship()` for the one-to-many, and a ready-to-use SQLite engine):

```python
import enum
from typing import List, Optional
from sqlalchemy import (
    create_engine, Column, ForeignKey, Table, Text, Boolean, String, Date,
    Time, DateTime, Float, Integer, Enum
)
from sqlalchemy.orm import (
    column_property, DeclarativeBase, Mapped, mapped_column, relationship
)
from datetime import datetime, time, date

class Base(DeclarativeBase):
    pass

# Join table for the Student <-> Course many-to-many
enrollment = Table(
    "enrollment",
    Base.metadata,
    Column("students", ForeignKey("student.id"), primary_key=True),
    Column("enrolledCourses", ForeignKey("course.id"), primary_key=True),
)

class Student(Base):
    __tablename__ = "student"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

class Course(Base):
    __tablename__ = "course"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    credits: Mapped[int] = mapped_column(Integer)
    taughtBy_id: Mapped[int] = mapped_column(ForeignKey("professor.id"), nullable=False)

class Professor(Base):
    __tablename__ = "professor"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))

# --- Relationships ---
Student.enrolledCourses: Mapped[List["Course"]] = relationship(
    "Course", secondary=enrollment, back_populates="students")
Course.students: Mapped[List["Student"]] = relationship(
    "Student", secondary=enrollment, back_populates="enrolledCourses")
Course.taughtBy: Mapped["Professor"] = relationship(
    "Professor", back_populates="taughtCourses", foreign_keys=[Course.taughtBy_id])
Professor.taughtCourses: Mapped[List["Course"]] = relationship(
    "Course", back_populates="taughtBy", foreign_keys=[Course.taughtBy_id])

# Database connection
DATABASE_URL = "sqlite:///University.db"
engine = create_engine(DATABASE_URL, echo=True)
Base.metadata.create_all(engine)
```

You can then use it directly with a SQLAlchemy `Session`.

## 4. Choosing a different database

`generate()` takes a `dbms` argument. Valid values are `"sqlite"` (default), `"postgresql"`, `"mysql"`, `"mssql"`, `"mariadb"`, `"oracle"`:

```python
generator.generate(dbms="postgresql")
```

## Optional extras

- The `SQLAlchemyGenerator` is one of several generators sharing the exact same `DomainModel`. From the same model you can also produce plain SQL DDL (`from besser.generators.sql import SQLGenerator`), Python classes (`from besser.generators.python_classes import PythonGenerator`), a REST API, a full backend, RDF, etc. — see the canonical example at `C:/Users/sulejmani/Desktop/BESSER-upstream/tests/BUML/metamodel/structural/library/library.py`.

### Key source references (v7.8.3)
- Metamodel / API (Class, Property, Multiplicity, BinaryAssociation, DomainModel, type shortcuts): `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/BUML/metamodel/structural/structural.py`
- SQLAlchemy generator (default `dbms="sqlite"`, valid DBMS list, reserved names, auto-id logic, `str`→`String(100)`): `C:/Users/sulejmani/Desktop/BESSER-upstream/besser/generators/sql_alchemy/sql_alchemy_generator.py`
- A complete, runnable end-to-end example: `C:/Users/sulejmani/Desktop/BESSER-upstream/tests/BUML/metamodel/structural/library/library.py`
- Reference generated output for comparison: `C:/Users/sulejmani/Desktop/BESSER-upstream/tests/BUML/metamodel/structural/library/output/sql_alchemy.py`