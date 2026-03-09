# Modeling a University System in BESSER and Generating SQLAlchemy ORM

## Overview

BESSER uses a B-UML (BESSER UML) metamodel to define domain models in Python. You define classes with attributes, then link them with associations that specify multiplicities (one-to-many, many-to-many, etc.). Once the model is defined, you pass it to the `SQLAlchemyGenerator` to produce a fully functional SQLAlchemy ORM file.

## Step 1: Install BESSER

```bash
# Clone and install
git clone https://github.com/BESSER-PEARL/BESSER.git
cd BESSER
pip install -e .
```

## Step 2: Define the University Domain Model

Create a file called `university.py` with the following content:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType
)
from besser.generators.sql_alchemy import SQLAlchemyGenerator

# -------------------------------------------------------
# Define attributes for each class
# -------------------------------------------------------

# Student attributes
student_name: Property = Property(name="name", type=StringType)
student_email: Property = Property(name="email", type=StringType)

# Student class
student: Class = Class(name="Student", attributes={student_name, student_email})

# Course attributes
course_name: Property = Property(name="name", type=StringType)
course_credits: Property = Property(name="credits", type=IntegerType)

# Course class
course: Class = Class(name="Course", attributes={course_name, course_credits})

# Professor attributes
professor_name: Property = Property(name="name", type=StringType)
professor_department: Property = Property(name="department", type=StringType)

# Professor class
professor: Class = Class(name="Professor", attributes={professor_name, professor_department})

# -------------------------------------------------------
# Define associations between classes
# -------------------------------------------------------

# Student enrolls in many Courses (many-to-many)
# A student can enroll in 0 or more courses
# A course can have 0 or more students enrolled
enrolls_in: Property = Property(
    name="enrollsIn", type=course, multiplicity=Multiplicity(0, "*")
)
has_students: Property = Property(
    name="hasStudents", type=student, multiplicity=Multiplicity(0, "*")
)
student_course_association: BinaryAssociation = BinaryAssociation(
    name="enrollment", ends={enrolls_in, has_students}
)

# Professor teaches Courses (one-to-many)
# A professor can teach 0 or more courses
# Each course is taught by exactly 1 professor
teaches: Property = Property(
    name="teaches", type=course, multiplicity=Multiplicity(0, "*")
)
taught_by: Property = Property(
    name="taughtBy", type=professor, multiplicity=Multiplicity(1, 1)
)
professor_course_association: BinaryAssociation = BinaryAssociation(
    name="teaching", ends={teaches, taught_by}
)

# -------------------------------------------------------
# Create the Domain Model
# -------------------------------------------------------
university_model: DomainModel = DomainModel(
    name="University",
    types={student, course, professor},
    associations={student_course_association, professor_course_association}
)

# -------------------------------------------------------
# Generate SQLAlchemy ORM code
# -------------------------------------------------------
sql_alchemy_generator = SQLAlchemyGenerator(model=university_model)
sql_alchemy_generator.generate(dbms="sqlite")

print("SQLAlchemy ORM code generated successfully!")
```

## Step 3: Run the Script

```bash
python university.py
```

This will produce a file called `sql_alchemy.py` in the `output/` folder inside your current working directory.

## Step 4: Understanding the Generated Output

The generated `output/sql_alchemy.py` file will contain SQLAlchemy ORM classes that look similar to this:

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

# Tables definition for many-to-many relationships
enrollment = Table(
    "enrollment",
    Base.metadata,
    Column("hasStudents", ForeignKey("student.id"), primary_key=True),
    Column("enrollsIn", ForeignKey("course.id"), primary_key=True),
)

# Tables definition
class Course(Base):
    __tablename__ = "course"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    credits: Mapped[int] = mapped_column(Integer)
    taughtBy_id: Mapped[int] = mapped_column(ForeignKey("professor.id"))

class Professor(Base):
    __tablename__ = "professor"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    department: Mapped[str] = mapped_column(String(100))

class Student(Base):
    __tablename__ = "student"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(100))
    email: Mapped[str] = mapped_column(String(100))

# Relationships
Course.taughtBy: Mapped["Professor"] = relationship("Professor", back_populates="teaches", ...)
Professor.teaches: Mapped[List["Course"]] = relationship("Course", back_populates="taughtBy", ...)
Student.enrollsIn: Mapped[List["Course"]] = relationship("Course", secondary=enrollment, back_populates="hasStudents")
Course.hasStudents: Mapped[List["Student"]] = relationship("Student", secondary=enrollment, back_populates="enrollsIn")

# Database connection
DATABASE_URL = "sqlite:///University.db"
engine = create_engine(DATABASE_URL, echo=True)

# Create tables in the database
Base.metadata.create_all(engine, checkfirst=True)
```

Key aspects of the generated output:

- **Many-to-many (Student-Course)**: Because both ends have max multiplicity > 1, the generator creates an intermediate `enrollment` junction `Table` with composite primary keys referencing both `student.id` and `course.id`.
- **One-to-many (Professor-Course)**: Because the `taughtBy` end has max multiplicity of 1, the generator places a `taughtBy_id` foreign key column directly on the `Course` table.
- **Auto-generated `id`**: Since no attribute is explicitly marked with `is_id=True`, the generator automatically adds an `id: Mapped[int]` primary key to each class.
- **Relationships**: SQLAlchemy `relationship()` calls with `back_populates` are generated for bidirectional navigation.

## Step 5: Create and Use the Database

Run the generated file to create the SQLite database:

```bash
python output/sql_alchemy.py
```

This creates a `University.db` SQLite file. You can then use it with SQLAlchemy sessions:

```python
from sqlalchemy.orm import Session
from output.sql_alchemy import engine, Student, Course, Professor

with Session(engine) as session:
    # Create a professor
    prof = Professor(name="Dr. Smith", department="Computer Science")

    # Create courses
    cs101 = Course(name="Intro to CS", credits=3, taughtBy=prof)
    cs201 = Course(name="Data Structures", credits=4, taughtBy=prof)

    # Create students and enroll them
    alice = Student(name="Alice", email="alice@university.edu")
    bob = Student(name="Bob", email="bob@university.edu")

    alice.enrollsIn.append(cs101)
    alice.enrollsIn.append(cs201)
    bob.enrollsIn.append(cs101)

    session.add_all([prof, cs101, cs201, alice, bob])
    session.commit()

    # Query: find all students in CS101
    for s in cs101.hasStudents:
        print(f"{s.name} is enrolled in {cs101.name}")
```

## Customizing the Output Directory

You can specify a custom output directory when creating the generator:

```python
sql_alchemy_generator = SQLAlchemyGenerator(
    model=university_model,
    output_dir="./my_output"
)
sql_alchemy_generator.generate(dbms="sqlite")
```

## Choosing a Different Database

The `generate()` method accepts a `dbms` parameter. Supported values are:

| Value          | Database                |
|----------------|-------------------------|
| `"sqlite"`     | SQLite (default)        |
| `"postgresql"` | PostgreSQL              |
| `"mysql"`      | MySQL                   |
| `"mariadb"`    | MariaDB                 |
| `"mssql"`      | Microsoft SQL Server    |
| `"oracle"`     | Oracle Database         |

For databases other than SQLite, you will need to edit the generated `DATABASE_URL` connection string with your actual credentials (username, password, host, port, and database name).

```python
# Example: generate for PostgreSQL
sql_alchemy_generator.generate(dbms="postgresql")
```

## Additional Features

### Adding Enumerations

If you want to add a course level enumeration (e.g., Undergraduate, Graduate):

```python
from besser.BUML.metamodel.structural import Enumeration, EnumerationLiteral

# Define enumeration literals
undergrad = EnumerationLiteral(name="UNDERGRADUATE")
grad = EnumerationLiteral(name="GRADUATE")

# Create the enumeration type
course_level = Enumeration(name="CourseLevel", literals={undergrad, grad})

# Add an attribute using the enumeration type
level: Property = Property(name="level", type=course_level)

# Update the Course class to include the new attribute
course: Class = Class(name="Course", attributes={course_name, course_credits, level})

# Include the enumeration in the domain model types
university_model: DomainModel = DomainModel(
    name="University",
    types={student, course, professor, course_level},
    associations={student_course_association, professor_course_association}
)
```

### Marking a Primary Key Attribute

By default, the generator auto-creates an `id` primary key. If you want to define your own:

```python
student_id: Property = Property(name="student_id", type=IntegerType, is_id=True)
student: Class = Class(name="Student", attributes={student_id, student_name, student_email})
```

### Optional/Nullable Attributes

To make an attribute nullable in the generated schema:

```python
student_email: Property = Property(name="email", type=StringType, is_optional=True)
```

This produces `email: Mapped[Optional[str]] = mapped_column(String(100), nullable=True)` in the output.

### Generating Other Artifacts

BESSER can generate many other outputs from the same domain model:

```python
from besser.generators.python_classes import PythonGenerator
from besser.generators.sql import SQLGenerator
from besser.generators.rest_api import RESTAPIGenerator
from besser.generators.django import DjangoGenerator
from besser.generators.backend import BackendGenerator
from besser.generators.pydantic_classes import PydanticGenerator

# Plain Python classes
PythonGenerator(model=university_model).generate()

# Raw SQL DDL statements
SQLGenerator(model=university_model, sql_dialects="sqlite").generate()

# REST API (FastAPI)
RESTAPIGenerator(model=university_model).generate()

# Django models + views
DjangoGenerator(model=university_model).generate()

# FastAPI backend with CRUD endpoints
BackendGenerator(model=university_model, http_methods=["GET", "POST", "PUT", "DELETE"]).generate()

# Pydantic data models
PydanticGenerator(model=university_model).generate()
```

## Summary

The complete workflow is:

1. **Define classes** using `Class` and `Property` with primitive types (`StringType`, `IntegerType`, etc.)
2. **Define associations** using `BinaryAssociation` with `Property` ends that specify the target type and `Multiplicity`
3. **Bundle everything** into a `DomainModel` with all types and associations
4. **Generate code** by instantiating `SQLAlchemyGenerator` and calling `.generate()`
5. **Run the output** to create your database schema and use it with SQLAlchemy sessions
