# Delivering a B-UML model

How to hand a model to a user: what to produce, in what format, and what to
tell them. Read this once you've built (or are about to build) a model and
need to decide between a runnable `.py` file and a Markdown-embedded diagram.

## Two outcomes: code or documentation

A B-UML model is useful even if you never run a generator. There are two
first-class things you can do with one:

1. **Generate code** — feed the model to a generator (Python, SQL, FastAPI,
   Django, React, …). This is BESSER's headline feature.
2. **Document a system** — a B-UML model *is* a correct UML class diagram:
   right multiplicities, associations, and inheritance, checked by
   `model.validate()`. Embed it in a README, design doc, or `.md` spec to
   document **any** project — no code generation required. Import it into
   editor.besser-pearl.org for the rendered visual diagram.

So reach for this skill whenever you need an accurate class diagram, not
only when the project will use BESSER's generators.

## Choosing the format — don't ask every time

- **Default → a runnable `.py` file** (e.g. `library_model.py`). For almost
  any "build/model X" request, this is the deliverable: a real artifact the
  user can run and import into the web editor.
- **Documentation context → embed the same B-UML in Markdown.** When the
  user is writing docs (a README or design doc, "add the model to the
  docs", "show me the diagram"), put the B-UML in a fenced Python code block
  inside the `.md` instead — same B-UML, for reading rather than running.
- **Ambiguous? Don't block on a question.** Write the `.py` file and add one
  line noting the same B-UML can be dropped into their docs.

Whichever you produce, don't just paste code into chat with no artifact.
Make the model self-contained: imports, class and association definitions,
the `DomainModel` assembly, a `model.validate()` check, and a commented-out
generator call the user can uncomment. (`scripts/scaffold_model.py` already
prints code in this shape.)

## The two ways to use the `.py` file

When you deliver a `.py` file, tell the user:

1. **Run it directly** to validate and generate code:

   ```bash
   python library_model.py
   ```

2. **Import it into the web editor** at https://editor.besser-pearl.org —
   use the editor's **Import** option and choose the **B-UML** format to
   load the `.py` model and edit it visually. (The editor imports and
   exports models in B-UML and JSON formats, so a model you write in code
   round-trips with the visual editor.)
