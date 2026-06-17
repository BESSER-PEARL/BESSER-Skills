# Modeling a university system in BESSER and generating a SQLAlchemy ORM

BESSER is model-driven: you describe the domain once as a **B-UML DomainModel**, validate it, then run a generator to produce code. For your case, you'll build the model in Python (the recommended path), then run the `SQLAlchemyGenerator` to get a working ORM.

## 1. Install BESSER

```bash
python -m venv venv
# Windows:
venv\Scripts\activate
# macOS/Linux:
source venv/bin/activate

pip install besser
```

Python 3.10+ is required. Verify the install:

```bash
python -c "from besser.BUML.metamodel.structural import DomainModel; print('OK')"
```

## 2. Map your requirements to the model

Your requirements translate into three classes and two associations:

| Requirement | B-UML element |
|---|---|
| Course has a name and credits | `Course` class with `name: str`, `credits: int` attributes |
| Students enroll in many courses (and a course has many students) | many-to-many `BinaryAssociation` between `Student` and `Course` |
| Professors teach courses | association between `Professor` and `Course` (a professor teaches many courses; here each course is taught by one professor) |

A key modeling note: in B-UML each association **end** is a `Property` whose `type` is the class on the *other* side, and whose `Multiplicity` expresses the cardinality seen *from* that side. A `*..*` multiplicity on both ends produces a many-to-many relationship (the SQLAlchemy generator will create a join table for it), while `1..*` produces a foreign key.

## 3. The model file

Save this as `university_model.py`. It builds the model and validates it.

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, StringType, IntegerType,
)

# --- Classes -------------------------------------------------------------

# Student
student_name = Property(name="name", type=StringType)
student = Class(name="Student", attributes={student_name})

# Course: a name and credits
course_name    = Property(name="name", type=StringType)
course_credits = Property(name="credits", type=IntegerType)
course = Class(name="Course", attributes={course_name, course_credits})

# Professor
professor_name = Property(name="name", type=StringType)
professor = Class(name="Professor", attributes={professor_name})

# --- Associations --------------------------------------------------------

# Student <-> Course : many-to-many (a student enrolls in many courses;
# a course has many enrolled students). Both ends are "*".
enrolled_courses = Property(name="enrolledCourses", type=course,  multiplicity=Multiplicity(0, "*"))
enrolled_students = Property(name="enrolledStudents", type=student, multiplicity=Multiplicity(0, "*"))
enrollment = BinaryAssociation(name="enrollment", ends={enrolled_courses, enrolled_students})

# Professor <-> Course : a professor teaches many courses;
# each course is taught by exactly one professor (one-to-many -> FK on Course).
taught_courses = Property(name="taughtCourses", type=course,    multiplicity=Multiplicity(0, "*"))
taught_by      = Property(name="taughtBy",      type=professor, multiplicity=Multiplicity(1, 1))
teaching = BinaryAssociation(name="teaching", ends={taught_courses, taught_by})

# --- Model ---------------------------------------------------------------

university_model = DomainModel(
    name="University_model",
    types={student, course, professor},
    associations={enrollment, teaching},
)

assert university_model.validate()["success"]
```

A few things worth knowing:

- **Naming rules**: no spaces, no hyphens. Use `PascalCase` for classes, `snake_case` for attributes. B-UML names become identifiers in the generated code.
- **`types` must list every class** referenced anywhere in the model — otherwise `validate()` fails.
- **Multiplicity**: `Multiplicity(min, max)` where `max` is `"*"` (or the constant `UNLIMITED_MAX_MULTIPLICITY` = 9999) for "many". Common patterns: exactly one `Multiplicity(1, 1)`, optional `Multiplicity(0, 1)`, one-or-more `Multiplicity(1, "*")`, zero-or-more `Multiplicity(0, "*")`.
- **Always `validate()` before generating.** It returns `{"success": ..., "errors": [...], "warnings": [...]}`.

If you'd rather make each course taught by *one or more* professors (true many-to-many for teaching too), change `taught_by` to `Multiplicity(0, "*")` — the generator will then emit a second join table instead of a foreign key.

## 4. Generate the SQLAlchemy ORM

Add this to the bottom of the same file (or a separate script that imports `university_model`):

```python
from besser.generators.sql_alchemy import SQLAlchemyGenerator

gen = SQLAlchemyGenerator(model=university_model, output_dir="./output")
gen.generate()
# Writes ./output/sql_alchemy.py
```

This produces `./output/sql_alchemy.py` containing the ORM classes, the many-to-many join table for the student/course enrollment, and the foreign key for the professor→course teaching relationship.

### Targeting a specific database

By default the generator targets SQLite. To target another DBMS, pass `dbms` to `generate()`:

```python
gen = SQLAlchemyGenerator(model=university_model, output_dir="./output")
gen.generate(dbms="postgresql")
# Valid values: sqlite | postgresql | mysql | mssql | mariadb | oracle
# Note: it is "postgresql", NOT "postgres".
```

If you omit `output_dir`, output goes to `<cwd>/output/`. **Each call to `generate()` overwrites the prior output** — that is intentional: the model is the source of truth, so you regenerate rather than hand-edit.

## 5. Run it / verify the output

After `generate()` returns:

1. **File exists**: confirm `./output/sql_alchemy.py` is present.
2. **Syntax parses**:
   ```bash
   python -c "import ast; ast.parse(open('output/sql_alchemy.py').read())"
   ```
3. **Relationships translate correctly**: you should see a join table for the `*..*` student-course enrollment and a foreign key on `Course` for the `1..*` professor relationship.
4. **Edge cases**: check that optional fields and any enumerations came through as expected.

## Iterating

When requirements change — say you add an `email` to `Student`, or make courses co-taught by several professors — **edit the model, not the generated file**, then re-run `generate()`. The model stays the single source of truth, and one model can feed many generators (e.g. `PydanticGenerator`, `SQLGenerator` for raw DDL, or `BackendGenerator` to get a full FastAPI backend with SQLAlchemy + Pydantic in one shot).

## Alternative: no-code via the web editor

If you'd prefer not to write Python, the visual editor at https://editor.besser-pearl.org lets you draw the three classes and two associations, then click "Generate" and pick SQLAlchemy. You can also **Import** a `.py` model file (B-UML format) into the editor, so the model above can be opened, edited visually, and re-exported. It uses the same generators under the hood.

---

Relevant skill files I based this on: `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-user/SKILL.md` and `C:/Users/sulejmani/Desktop/besser-skills/skills/besser-user/references/metamodel.md`.