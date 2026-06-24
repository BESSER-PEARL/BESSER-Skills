# Testing

BESSER uses pytest. Run tests locally before opening a PR (CI also runs them
— see `contributing-workflow.md`).

## Run tests

```bash
python -m pytest                              # all tests

python -m pytest tests/generators/sqlalchemy/ # by directory
python -m pytest -k "test_django"             # by keyword
python -m pytest -v tests/BUML/metamodel/structural/  # verbose
python -m pytest -x                           # stop on first failure
```

## Conventions

- Test files: `test_*.py`; test functions: `test_*`.
- Use `@pytest.fixture` for reusable models and directories.
- Use `tmp_path` for output directories (auto-cleaned by pytest) rather than
  creating/removing directories by hand.
- Assert specific content, not just file existence.
- Test error conditions with `pytest.raises(ValueError)`.
- For generators that produce importable Python, use `importlib` to load and
  test the generated code.
- If you must manage a directory yourself, clean up with `shutil.rmtree()` in
  a `try/finally`.

## What to test

For **metamodel changes**: construction validation, setter constraints,
expected errors for invalid input, `validate()` results.

For **generators**: output file existence, key content in output (class
names, relationship mappings), edge cases (empty model, deep inheritance,
many-to-many).

For **converters**: round-trip fidelity (`JSON → BUML → JSON` should be
equivalent).
