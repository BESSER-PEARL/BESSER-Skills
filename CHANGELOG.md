# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.2.1] - 2026-06-18

### Fixed
- Corrected **24 API/accuracy errors** across the skills, found by a
  systematic audit against BESSER v7.8.3 source:
  - `besser-generators`: the custom-endpoint example now uses
    `Method(code=..., implementation_type=MethodImplementationType.BAL)`
    (the previous `MethodImplementation` class/import did not exist); the
    SQLAlchemy customization example uses `sqlalchemy.orm.Session` + the
    generated `engine` (no `Session` is generated); `BAFGenerator` output is
    `config.yaml`, not `config.ini`.
  - `TerraformGenerator(deployment_model=...)`; the PyTorch/TF generators
    import from `besser.generators.nn.{pytorch,tf}.*` with `generation_type`
    as a constructor argument; `FlutterGenerator` requires `main_page` and
    is registered in the web editor; many-to-many join-table name is the
    association name only; `jinja2.TemplateNotFound`; `BackendGenerator`
    filters invalid HTTP methods silently.
  - Python requirement corrected to **3.11+** (setup.cfg `python_requires`);
    `bocl==1.0.1`; Docker base `python:3.11-slim`; state-machine transitions
    live on `State`, not `StateMachine`.
  - `besser-dev`: `SUPPORTED_GENERATORS` (not `GENERATORS`); documented that
    BESSER now ships CI (pytest + ruff).

### Added
- Two evals (documentation delivery, `SQLGenerator` dialect) — the suite is
  now 10 (`evals/evals.json`), making those behaviors permanent regression
  checks.
- README skills table and trigger table now surface "drawing UML for
  documentation".
- CI: `.github/workflows/release.yml` automatically creates a GitHub Release
  from the matching CHANGELOG section whenever a `v*` tag is pushed.

## [0.2.0] - 2026-06-17

### Added
- **Modeling for documentation, not only code.** `besser-user` now frames a
  B-UML model as having two first-class outcomes — generating code *and*
  documenting a system — and its trigger description was broadened so the
  skill activates on "draw/document a correct UML class diagram" requests
  even when BESSER is not named.
- **README "Drawing correct UML diagrams" section** showing how an agent
  draws a correct class diagram in B-UML and embeds it in Markdown docs,
  usable for any project whether or not code is generated.
- **Model-delivery guidance** in `besser-user`: deliver a model as a runnable
  `.py` file by default, or embed the same B-UML in Markdown for
  documentation; plus how to import a model into the web editor
  (Import → B-UML). Detail lives in the new
  `references/delivering-models.md`.

### Fixed
- **`SQLGenerator` API in `references/persistence.md`.** It previously showed
  `gen.generate(sql_dialect=...)`, which raises `TypeError` — `sql_dialect`
  is a constructor argument and `generate()` takes none (only
  `SQLAlchemyGenerator` takes `dbms` in `generate()`). Corrected, with a note
  contrasting the two SQL generators. Caught by the 0.2.0 benchmark re-run.

### Changed
- Refactored `besser-user/SKILL.md` toward the progressive-disclosure target
  by moving model-delivery detail into `references/delivering-models.md`.

### Benchmarks
- Correctness re-run for 0.2.0 in `benchmarks/iteration-2/`: **38/38 (100%)**
  with-skill, matching the no-skill baseline against BESSER v7.8.3.
  Performance figures are carried over from the v0.1.0 benchmark
  (changes are skill-text only).

## [0.1.0] - 2026-05-12

Initial public release of BESSER Skills.

### Added
- Four installable skills for AI coding agents:
  - `besser-user` — end-user guide: domain modeling, code generation, PlantUML, web editor
  - `besser-generators` — per-generator operations, safe customization, template overrides
  - `besser-troubleshooting` — diagnosis guide for install, import, runtime, and deployment issues
  - `besser-dev` — contributor guide: adding generators, tests, docs, PR workflow
- `besser-user` and `besser-generators` follow the progressive-disclosure
  pattern: a slim `SKILL.md` plus `references/*.md` loaded on demand.
- Scaffolding scripts:
  - `besser-user/scripts/scaffold_model.py` — prints a starter `DomainModel`
  - `besser-dev/scripts/scaffold_generator.py` — scaffolds a new generator package
- Eight eval definitions (2 per skill) in `evals/evals.json`.
- Benchmark results in `benchmarks/iteration-1/`: with-skill vs
  without-skill baseline across all 8 evals. 100% pass rate, 74% faster,
  48% fewer tokens than the baseline.
- README with installation instructions, benchmark results, and project structure.
- CONTRIBUTING guide for skill authors.
