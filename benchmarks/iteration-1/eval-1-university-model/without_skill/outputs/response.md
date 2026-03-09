# Modeling a University System in BESSER and Generating SQLAlchemy ORM

## Overview

BESSER lets you define a domain model in Python using its B-UML metamodel, and then use code generators to produce executable artifacts such as SQLAlchemy ORM code. Below is a complete, step-by-step guide for modeling a university system with **Students**, **Courses**, and **Professors**, including the relationships "students enroll in many courses" and "professors teach courses."

---

## Step 1: Install BESSER

Make sure BESSER is installed in your Python environment:

```bash
# Clone the repository (if you haven't already)
git clone https://github.com/BESSER-PEARL/BESSER.git
cd BESSER

# Create a virtual environment and install
python -m venv venv
venv/Scripts/activate        # On Windows
# source venv/bin/activate   # On Linux/macOS
pip install -e .
```

---

## Step 2: Define the Domain Model

Create a Python file (e.g., `university_model.py`) with the following content:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# -------------------------------------------------------
# 1. Define Classes and Their Attributes
# -------------------------------------------------------

# Student class
student_name: Property = Property(name="name", type=StringType)
student_email: Property = Property(name="email", type=StringType)

student: Class = Class(name="Student", attributes={student_name, student_email})

# Course class
course_name: Property = Property(name="name", type=StringType)
credits: Property = Property(name="credits", type=IntegerType)

course: Class = Class(name="Course", attributes={course_name, credits})

# Professor class
professor_name: Property = Property(name="name", type=StringType)
department: Property = Property(name="department", type=StringType)

professor: Class = Class(name="Professor", attributes={professor_name, department})

# -------------------------------------------------------
# 2. Define Associations (Relationships)
# -------------------------------------------------------

# Student <-> Course: many-to-many ("enrolls in")
# A student enrolls in 0 or more courses; a course has 0 or more students.
enrolls_student_end: Property = Property(
    name="students",
    type=student,
    multiplicity=Multiplicity(0, "*")
)
enrolls_course_end: Property = Property(
    name="courses",
    type=course,
    multiplicity=Multiplicity(0, "*")
)
enrollment_association: BinaryAssociation = BinaryAssociation(
    name="Enrollment",
    ends={enrolls_student_end, enrolls_course_end}
)

# Professor <-> Course: one-to-many ("teaches")
# A professor teaches 0 or more courses; each course is taught by exactly 1 professor.
teaches_prof_end: Property = Property(
    name="professor",
    type=professor,
    multiplicity=Multiplicity(1, 1)
)
teaches_course_end: Property = Property(
    name="courses",
    type=course,
    multiplicity=Multiplicity(0, "*")
)
teaches_association: BinaryAssociation = BinaryAssociation(
    name="Teaches",
    ends={teaches_prof_end, teaches_course_end}
)

# -------------------------------------------------------
# 3. Create the Domain Model
# -------------------------------------------------------
university_model: DomainModel = DomainModel(
    name="University",
    types={student, course, professor},
    associations={enrollment_association, teaches_association}
)

# -------------------------------------------------------
# 4. Generate SQLAlchemy ORM Code
# -------------------------------------------------------
sqlalchemy_gen = SQLAlchemyGenerator(model=university_model)
sqlalchemy_gen.generate()

print("SQLAlchemy code generated successfully!")
```

---

## Step 3: Run the Script

```bash
python university_model.py
```

This will create a file named `sql_alchemy.py` inside an `output/` directory (relative to your current working directory). If you want to control the output location, pass an `output_dir` argument:

```python
sqlalchemy_gen = SQLAlchemyGenerator(model=university_model, output_dir="./generated")
sqlalchemy_gen.generate()
```

---

## Step 4: Understanding the Generated Output

The generated `sql_alchemy.py` file will contain:

1. **A `Base` class** for SQLAlchemy's declarative base.
2. **An association table** for the many-to-many relationship between `Student` and `Course` (the `Enrollment` association). Since both ends have a max multiplicity greater than 1, BESSER automatically generates an intermediary table.
3. **Table classes** for `Student`, `Course`, and `Professor`, each with:
   - An auto-generated `id` primary key (since we did not mark any attribute with `is_id=True`).
   - Mapped columns for each attribute (e.g., `name`, `email`, `credits`, `department`).
   - A foreign key on `Course` pointing to `Professor` (for the one-to-many "teaches" relationship).
4. **Relationship definitions** using SQLAlchemy's `relationship()` function to enable ORM-level navigation between related objects.
5. **A database connection** setup (SQLite by default).

The generated code will look approximately like this:

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
from datetime import datetime as dt_datetime, time as dt_time, date as dt_date

class Base(DeclarativeBase):
    pass

# Association table for many-to-many (Student <-> Course)
enrollment = Table(
    "enrollment",
    Base.metadata,
    Column("students", ForeignKey("student.id"), primary_key=True),
    Column("courses", ForeignKey("course.id"), primary_key=True),
)

# Tables definition
class Student(Base):
    __tablename__ = "student"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(100))

class Professor(Base):
    __tablename__ = "professor"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    department: Mapped[str] = mapped_column(String(100))

class Course(Base):
    __tablename__ = "course"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    credits: Mapped[int] = mapped_column(Integer)
    professor_id: Mapped[int] = mapped_column(ForeignKey("professor.id"))

# Relationships
Student.courses: Mapped[List["Course"]] = relationship(
    "Course", secondary=enrollment, back_populates="students"
)
Course.students: Mapped[List["Student"]] = relationship(
    "Student", secondary=enrollment, back_populates="courses"
)
Course.professor: Mapped["Professor"] = relationship("Professor", back_populates="courses")
Professor.courses: Mapped[List["Course"]] = relationship("Course", back_populates="professor")

# Database connection
DATABASE_URL = "sqlite:///University.db"
engine = create_engine(DATABASE_URL, echo=True)
Base.metadata.create_all(engine, checkfirst=True)
```

> **Note**: The exact output may vary slightly depending on attribute ordering (which is based on creation timestamps in BESSER). The code above is a representative illustration.

---

## Step 5: Using the Generated Code

You can immediately use the generated ORM with SQLAlchemy sessions:

```python
from sqlalchemy.orm import Session

# Import from the generated file
from output.sql_alchemy import engine, Student, Course, Professor

with Session(engine) as session:
    # Create a professor
    prof = Professor(name="Dr_Smith", department="Computer_Science")
    session.add(prof)

    # Create courses
    course1 = Course(name="Databases", credits=5, professor=prof)
    course2 = Course(name="Algorithms", credits=4, professor=prof)
    session.add_all([course1, course2])

    # Create a student and enroll in courses
    student = Student(name="Alice", email="alice@university.edu")
    student.courses.append(course1)
    student.courses.append(course2)
    session.add(student)

    session.commit()
```

---

## Choosing a Different Database

By default, the generator targets SQLite. You can specify a different DBMS when calling `generate()`:

```python
# For PostgreSQL
sqlalchemy_gen.generate(dbms="postgresql")

# For MySQL / MariaDB
sqlalchemy_gen.generate(dbms="mysql")

# For Microsoft SQL Server
sqlalchemy_gen.generate(dbms="mssql")

# For Oracle
sqlalchemy_gen.generate(dbms="oracle")
```

The supported options are: `sqlite`, `postgresql`, `mysql`, `mssql`, `mariadb`, and `oracle`.

---

## Additional Options

### Marking a Primary Key Explicitly

If you want a specific attribute to serve as the primary key instead of the auto-generated `id`, set `is_id=True`:

```python
student_id: Property = Property(name="student_id", type=IntegerType, is_id=True)
student: Class = Class(name="Student", attributes={student_id, student_name, student_email})
```

### Optional / Nullable Attributes

Mark an attribute as optional (nullable in the database):

```python
phone: Property = Property(name="phone", type=StringType, is_optional=True)
```

### Other Generators

BESSER can generate much more from the same domain model. Just swap the generator:

```python
from besser.generators.python_classes import PythonGenerator
from besser.generators.sql import SQLGenerator
from besser.generators.django import DjangoGenerator
from besser.generators.backend import BackendGenerator

# Plain Python classes
PythonGenerator(model=university_model).generate()

# Raw SQL DDL
SQLGenerator(model=university_model).generate()

# Django models + project scaffold
DjangoGenerator(model=university_model).generate()

# FastAPI backend with CRUD endpoints
BackendGenerator(model=university_model, http_methods=["GET", "POST", "PUT", "DELETE"]).generate()
```

---

## Key Concepts Summary

| BESSER Concept | What It Represents | Example |
|---|---|---|
| `Class` | A domain entity | `Student`, `Course`, `Professor` |
| `Property` (as attribute) | An attribute of a class | `name`, `credits`, `email` |
| `Property` (as association end) | One end of a relationship | `students`, `courses` |
| `Multiplicity(min, max)` | Cardinality constraint | `Multiplicity(0, "*")` = zero-to-many |
| `BinaryAssociation` | A relationship between two classes | `Enrollment`, `Teaches` |
| `DomainModel` | The complete model container | Groups all classes and associations |
| `SQLAlchemyGenerator` | Code generator | Transforms the model into ORM code |

---

## File References

The key source files in the BESSER codebase related to this example:

- **Metamodel definition**: `besser/BUML/metamodel/structural/structural.py` -- defines `Class`, `Property`, `Multiplicity`, `BinaryAssociation`, `DomainModel`, and primitive types like `StringType` and `IntegerType`.
- **SQLAlchemy generator**: `besser/generators/sql_alchemy/sql_alchemy_generator.py` -- the `SQLAlchemyGenerator` class that transforms a `DomainModel` into SQLAlchemy code.
- **SQLAlchemy Jinja2 template**: `besser/generators/sql_alchemy/templates/sql_alchemy_template.py.j2` -- the template used to render the generated Python file.
- **Library example** (good reference): `tests/BUML/metamodel/structural/library/library.py` -- a similar example using Library, Book, and Author classes.
