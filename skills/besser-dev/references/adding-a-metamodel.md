# Adding a New Metamodel / Sub-DSL

For adding a new model type under `besser/BUML/metamodel/` (e.g. a new DSL
alongside structural, state_machine, object, feature_model, etc.).

## Step 1: Design the metamodel

Create new classes in `besser/BUML/metamodel/your_dsl/`:

```
besser/BUML/metamodel/your_dsl/
  __init__.py
  your_dsl.py    # metamodel classes
```

Follow existing patterns:
- Extend `NamedElement` or `Element` as base classes.
- Use private properties with getter/setter validation.
- Keep naming consistent with existing metamodel packages.

## Step 2: Add converters (if web editor integration needed)

Converters translate between frontend JSON and B-UML objects:

```
besser/utilities/web_modeling_editor/backend/services/converters/
  json_to_buml/process_your_dsl.py    # JSON → BUML
  buml_to_json/your_dsl_to_json.py    # BUML → JSON
```

Converters must be symmetric: any feature supported in one direction must
work in the other. Test round-trips: `JSON → BUML → JSON` should produce the
original.

## Step 3: Add validation

If the metamodel has constraints beyond what setters enforce:

```python
def validate(self, raise_exception=True):
    errors = []
    warnings = []
    # Check structural rules
    if some_bad_condition:
        errors.append("Descriptive error message")
    result = {"success": len(errors) == 0, "errors": errors, "warnings": warnings}
    if raise_exception and errors:
        raise ValueError("\n".join(errors))
    return result
```

## Step 4: Wire into backend

- Add endpoint(s) in `besser/utilities/web_modeling_editor/backend/backend.py`.
- Add Pydantic request/response models in `backend/models/`.
- Add validation route under `/validate-diagram` if applicable.

## Step 5: Write tests

Place tests in `tests/BUML/metamodel/your_dsl/`:
- Valid model construction tests.
- Invalid model tests (expect `ValueError`, `TypeError`).
- Round-trip converter tests if applicable.

See `testing.md` for conventions.

## Step 6: Document

- Metamodel docs: `docs/source/buml_language/your_dsl.rst`
- Add to `docs/source/buml_language.rst` index.

For the user-facing shape your documentation should take, look at how
existing metamodels are presented in the **besser-user** skill's
`references/` (e.g. `class-diagram.md`, `feature-models.md`,
`object-models.md`) — those are the end-user reference your new DSL will
eventually want a counterpart to.
