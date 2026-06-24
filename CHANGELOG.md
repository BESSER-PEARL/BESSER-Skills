# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.4.0] - 2026-06-24

Expanded modeling references (contributed) plus a full editorial-review pass
across all four skills.

### Added
- **`besser-user` — much deeper class-diagram, state-machine, and agent
  references** (community contribution): `metamodel.md` renamed to
  `class-diagram.md` and expanded (`AssociationClass`, `GeneralizationSet`,
  `is_abstract`/`is_optional`/`is_read_only`, navigability, composition
  direction, void methods, end-name uniqueness); state machines gain final
  states, conditions, fallback bodies, config properties, and the
  `STATE_MACHINE` method-implementation pattern; agents gain LLM integration,
  entities/slot-filling, intent-classifier config, RAG, `DBReply`,
  `ReasoningState`, tools/skills/workspaces, and rich platform replies — all
  verified against BESSER v7.8.3 source.

### Changed
- **`besser-dev` restructured** from a 487-line monolith into a lean overview
  + routing table plus five `references/` (adding-a-generator,
  adding-a-metamodel, testing, docs-and-build, contributing-workflow),
  matching the progressive-disclosure pattern of the other skills.
- **De-duplicated drift-prone content across skills**: the web-editor
  registry snippet is now single-sourced in `besser-dev`; the
  composite-generator tree lives only in `debugging.md`; the
  `besser-troubleshooting` generator-failure table is scoped to error-message
  lookups with a clear hand-off to `besser-generators`.
- **`besser-generators` editorial fixes**: merged the redundant
  reference-layout/generator-picker tables; replaced the partial
  output-directory table with an exhaustive-by-rule "Output locations"
  call-out; normalized the Supabase/JSON-object entries to the house table
  style; clarified that BAL/CODE method bodies are inserted verbatim.
- **Consistency pass**: standardized reference "Gotchas" headings, added
  tables of contents to the longer references, and moved per-file version
  stamps to a single note; relaxed the attribute-naming guidance to match
  what BESSER actually enforces (no spaces/hyphens; snake_case or camelCase).

### Fixed
- Corrected a broken copy-paste example (an always-false chained-comparison
  assertion) in `object-models.md`, and the `chat_history` type in
  `agents.md`. Reconciled `CONTRIBUTING.md`'s description-voice rule with the
  trigger style all four skills actually use.

## [0.3.0] - 2026-06-18

### Added
- **Coverage of all remaining B-UML DSLs** in `besser-user` — seven new
  source-verified references:
  - `references/object-models.md` — object/instance models (objects,
    attribute values, links), incl. the fluent builder and OCL test data.
  - `references/feature-models.md` — software product lines (features,
    groups, configurations).
  - `references/ocl.md` — writing and attaching OCL `Constraint`s; `bocl`
    evaluation; which generators consume OCL.
  - `references/deployment.md` — deployment models for the
    `TerraformGenerator`.
  - `references/neural-networks.md` — NN models for the PyTorch/TF
    generators.
  - `references/quantum.md` — quantum circuits for the `QiskitGenerator`.
  - `references/project.md` — bundling models + metadata.
- **Two previously undocumented generators** in `besser-generators`:
  - `SupabaseGenerator` (Postgres DDL + RLS) in `references/persistence.md`.
  - `JSONObjectGenerator` (JSON data from an `ObjectModel`) in
    `references/python-and-data.md`.
- Reference-layout/generator-picker tables, trigger descriptions, and the
  README updated to surface the new model types and generators. The skills
  now cover all 19 BESSER generators and every B-UML metamodel/DSL.

### Notes
- All new content was written against BESSER v7.8.3 source with verified API
  citations; several upstream documentation bugs were deliberately *not*
  reproduced (e.g. the wrong `besser.BUML.notations.od.*` import path and the
  `ObjectModel(instances=…)` kwarg). Known-broken paths are flagged in-place
  (e.g. the `objectPlantUML` parser is non-functional in v7.8.3).

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
