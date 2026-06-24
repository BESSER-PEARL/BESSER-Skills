---
name: besser-dev
description: >
  Contributor guide for developing BESSER itself
  (https://github.com/BESSER-PEARL/BESSER). Use this skill whenever the user
  is working *inside* the BESSER source tree — adding a new generator (the
  most common contribution), adding a new metamodel or sub-DSL under
  `besser/BUML/metamodel/`, writing pytest tests for generators or
  metamodels, writing JSON↔BUML converters for the web editor, building
  Sphinx documentation under `docs/source/`, registering a generator in
  `SUPPORTED_GENERATORS`, or preparing a pull request to BESSER. Trigger on
  phrases like "add a new generator", "register in the web editor",
  "GeneratorInterface", "json_to_buml", "buml_to_json", "write tests for my
  generator", "build the docs", "open a PR to BESSER", or any work that
  touches `besser/generators/`, `besser/BUML/metamodel/`,
  `besser/utilities/web_modeling_editor/`, or `tests/`. Prefer this skill
  over besser-user when the user is contributing *to* BESSER rather than
  *using* BESSER to build something else.
license: Apache-2.0
compatibility:
  - claude-code
  - cursor
  - cline
  - windsurf
  - copilot
metadata:
  author: BESSER-PEARL
  version: "0.3.0"
  repository: https://github.com/BESSER-PEARL/BESSER-Skills
---

# Contributing to BESSER

This skill covers the procedural workflows for contributing to the BESSER
codebase. For architecture details and code conventions, also consult the
BESSER repo's `CLAUDE.md` (auto-loaded in Claude Code; with other agents,
read `CLAUDE.md` at the repo root directly).

---

## Development Setup

```bash
git clone https://github.com/<your-username>/BESSER.git
cd BESSER
python -m venv venv
# Windows: venv\Scripts\activate
# macOS/Linux: source venv/bin/activate
pip install -r requirements.txt
pip install -r docs/requirements.txt  # for building docs
pip install -e .                       # editable install
```

Verify: `python tests/BUML/metamodel/structural/library/library.py`

Python 3.11+ is required (enforced by `python_requires = >=3.11` in
`setup.cfg`; CI tests 3.11 and 3.12).

---

## Reference layout

This skill keeps SKILL.md short. Reach into `references/` for the procedure
you need:

| Task | Read |
|------|------|
| Add a new generator (the most common contribution) — 6 steps + scaffold | `references/adding-a-generator.md` |
| Add a new metamodel / sub-DSL, plus JSON↔BUML converters | `references/adding-a-metamodel.md` |
| Write pytest tests (fixtures, `tmp_path`, what to assert) | `references/testing.md` |
| Build the Sphinx docs and cross-reference them | `references/docs-and-build.md` |
| Code style, commit/PR conventions, cross-repo, CI/release | `references/contributing-workflow.md` |

Related skills for the *using* side (not contributing):
- Running or customizing an existing generator → the **besser-generators** skill.
- Diagnosing install/import/generation errors → the **besser-troubleshooting** skill.
- The user-facing shape of a model type (doc templates) → the **besser-user** skill's `references/`.

---

## Common Contribution Pitfalls

1. **Don't duplicate logic** — shared helpers go in `besser/utilities/`, not in individual generators.
2. **Maintain determinism** — generators must produce identical output for identical input.
3. **Keep converters symmetric** — if `JSON → BUML` supports a feature, `BUML → JSON` must too.
4. **Update docs** — any backend change likely needs `docs/source/` updates.
5. **Don't touch the frontend submodule** unless explicitly required. UI changes go to the upstream WME repo.
6. **Clean up resources** — always use `try/finally` for temp directories and file handles.
7. **Test round-trips** — especially for converters (`JSON → BUML → JSON` should be identity).
