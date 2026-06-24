# Code Style, Commits, PRs, and Release

## Code style

- PEP 8 with 4-space indentation.
- Type hints for public APIs; PEP 257 docstrings.
- Import order: standard library, third-party, local modules.
- `snake_case` functions/variables, `PascalCase` classes, `UPPER_CASE` constants.
- No wildcard imports, no implicit re-exports.
- 120-char line limit and pylint config live in `pyproject.toml`
  (`[tool.pylint.'FORMAT']`); CI also runs `ruff check` on `besser/` with a
  curated rule set. No black/pre-commit hooks are configured.

## Commit messages

Use Conventional Commits:

```
feat: add Terraform generator for AWS EKS
fix: correct multiplicity parsing for 0..* associations
refactor: extract shared template helpers to besser.utilities
docs: document QiskitGenerator usage
test: add round-trip tests for GUI model converter
```

## Pull request workflow

1. Create a topic branch: `git checkout -b feature/add-my-generator`.
2. Make focused, logically grouped commits.
3. Run tests locally: `python -m pytest` (see `testing.md`).
4. Build docs: `cd docs && make html` (see `docs-and-build.md`).
5. Rebase on latest `master`: `git fetch origin && git rebase origin/master`.
6. Push and open PR against `master`.
7. Fill in the PR template: description, tests executed, extra context.
8. Two maintainer approvals required for merge (one approval enough after 14 days).

## Cross-repo changes (BESSER + frontend)

When changes affect both BESSER and the web editor frontend:

1. Implement and commit WME changes in the [WME repo](https://github.com/BESSER-PEARL/BESSER-WEB-MODELING-EDITOR).
2. Implement BESSER changes in this repo.
3. Update the submodule pointer:
   ```bash
   cd besser/utilities/web_modeling_editor/frontend
   git fetch && git checkout <commit>
   cd ../../../..
   git add besser/utilities/web_modeling_editor/frontend
   ```
4. Link the two PRs so reviewers merge in correct order.

Don't touch the frontend submodule unless explicitly required — UI changes
go to the upstream WME repo.

## CI / Release

- **CI**: `.github/workflows/ci.yml` runs pytest on every PR to
  `master`/`development` (Python 3.11 and 3.12) plus a `ruff` lint job. Run
  tests locally before pushing too.
- **Release**: triggered by creating a GitHub Release. The
  `python-publish.yml` workflow builds and publishes to PyPI using
  `PYPI_API_TOKEN`.
- **Version**: defined in `setup.cfg` under `[metadata] version`.
- **Release notes**: added as RST files in `docs/source/releases/`.
