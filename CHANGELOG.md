# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.3.1] - 2026-09-11

Re-verified against BESSER 7.16.1. The backend generator's output shape and the
Java generator's options had both moved since the last pass, and two claims were
simply wrong.

### Fixed
- **`BackendGenerator` output is a package, not three files.** The API is split
  per class: `main_api.py` (app setup and `include_router` calls only),
  `database.py`, `bal_stdlib.py` and `routers/<class>.py`. Corrected in
  `besser-generators` SKILL.md, `references/api-and-web.md` (both
  `BackendGenerator` and `RESTAPIGenerator` backend mode) and the composite
  diagram in `references/debugging.md`. The guidance to "check `main_api.py` for
  what is in scope" now points at `routers/<class>.py` and `database.py`, where
  handler bodies and the session dependency actually live.
- **Generated output is not byte-identical between runs.** The skill promised
  determinism; generators iterate unordered collections, so members can come out
  in a different order each run. Regenerating the same model twice produced 216
  differing lines. The claim is replaced with what holds - equivalent, not
  identical - plus the two consequences: do not diff regenerations, and look
  table columns up by name rather than by position.
- **`JavaGenerator` no longer guesses a package from the output path.** That was
  removed; `package_name` is an explicit constructor parameter. Documented, along
  with the enum files, method stubs and navigability-filtered bidirectional
  associations the generator gained.
- **OCL consumers were understated.** `DjangoGenerator`, `BackendGenerator` /
  `RESTAPIGenerator` and `TestGenerator` all consume OCL, not only
  `PydanticGenerator`. More importantly, the split is not "single versus
  multi-property": a constraint over several properties of one object *is*
  enforced, as a `model_validator`; a constraint navigating a collection is
  *not*, and is emitted as a comment. `references/ocl.md` now states this in a
  table, with a warning not to assume a collection rule is applied.
- **`scaffold_model.py` emitted a non-ASCII character.** Redirected on a console
  that is not UTF-8 (cp1252 on Windows), the generated file would not parse:
  `SyntaxError: Non-UTF-8 code starting with ''`. The emitted text is ASCII
  now; same fix in `scaffold_generator.py`.

### Added
- `DomainModel.validate()` runs seven checks, not the four listed. The three
  missing ones are documented in `references/class-diagram.md`, including the
  rule added in BESSER 7.16.0: a method may not share its name with an attribute
  or association end of the same class, because generated code reaches both
  through that one name.
- Association-class link payloads (`<AssocClass>LinkCreate`) for
  `PydanticGenerator(backend=True)`, including that an optional link attribute
  stays optional in the payload.
- `WebAppGenerator`'s `agent_models` / `agent_configs` lists, and that the
  singular forms are a deprecated back-compat shim.

## [0.3.0] - 2026-06-26

Full coverage of every B-UML DSL and generator, deeper class-diagram /
state-machine / agent references, a complete editorial-review pass, a
marketing/positioning rework of the README, and field fixes surfaced by real
agent runs.

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
- **Much deeper class-diagram, state-machine, and agent references**
  (community contribution): `metamodel.md` renamed to `class-diagram.md` and
  expanded (`AssociationClass`, `GeneralizationSet`,
  `is_abstract`/`is_optional`/`is_read_only`, navigability, composition
  direction, void methods, end-name uniqueness); state machines gain final
  states, conditions, fallback bodies, config properties, and the
  `STATE_MACHINE` method-implementation pattern; agents gain LLM integration,
  entities/slot-filling, intent-classifier config, RAG, `DBReply`,
  `ReasoningState`, tools/skills/workspaces, and rich platform replies.
- The skills now cover all 19 BESSER generators and every B-UML
  metamodel/DSL; reference-layout tables, trigger descriptions, and the
  README were updated to surface them.
- **README "What your agent can build" capability block** — surfaces the full
  BESSER surface an agent can drive: every B-UML model type, all 19 generators,
  the deterministic-codegen-then-build-on-top workflow, troubleshooting, and
  extending BESSER itself. Cross-links the standalone
  [`uml-drawing`](https://github.com/BESSER-PEARL/uml-drawing) skill and adds an
  [Agent Skills directory](https://skills.sh/besser-pearl/besser-skills) link.
- **One-call `B-UML → SVG` rendering** in `besser-user` and the README — POST a
  model to BESSER v7.9.0's headless `/besser_api/get-svg` endpoint and embed the
  result, no browser required.
- **"Generate the baseline, then build on top"** guidance in `besser-generators`:
  regenerate the deterministic baseline freely (cheap and identical each run) and
  put customizations in *separate* files so a re-run never clobbers them.

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
- **README refreshed for v7.9.0**: BESSER badge bumped to v7.9.0, the drawing
  section now leads with the one-call headless SVG endpoint, and the codegen
  bullet is reframed around deterministic baselines you extend on top — spend
  tokens only on what is genuinely custom. The intro now links directly to the
  `uml-drawing` repo instead of an in-page anchor.

### Fixed
- Corrected a broken copy-paste example (an always-false chained-comparison
  assertion) in `object-models.md`, and the `chat_history` type in
  `agents.md`. Reconciled `CONTRIBUTING.md`'s description-voice rule with the
  trigger style all four skills actually use.
- **Stopped recommending `default_value` on enum-typed attributes** across
  `besser-user`, `besser-troubleshooting`, and `besser-generators`'
  `debugging.md`. It passes `validate()`, but the Pydantic/SQLAlchemy generators
  serialize the literal's raw `repr()` → invalid Python (e.g. `SyntaxError:
  leading zeros…`), and the headless SVG endpoint rejects it. Set the initial
  value in the app/auth layer or as a DB column default instead. (Primitive
  defaults like `True` or `0` round-trip fine.)
- **Class diagrams are delivered as B-UML, never Mermaid** (`besser-user`). The
  authoritative artifact is the `validate()`-checked B-UML model; a rendered
  SVG/PNG is the picture and a quick ASCII sketch is fine as a preview, but
  Mermaid (un-validated, can't be generated from) is no longer produced.

### Notes
- All new content was written against BESSER v7.8.3 source with verified API
  citations; several upstream documentation bugs were deliberately *not*
  reproduced (e.g. the wrong `besser.BUML.notations.od.*` import path and the
  `ObjectModel(instances=…)` kwarg). Known-broken paths are flagged in-place
  (e.g. the `objectPlantUML` parser is non-functional in v7.8.3).
- The headless `B-UML → SVG` flow and the v7.9.0 badge track BESSER v7.9.0
  (PR #554, ELK auto-layout); all other API citations remain verified against
  v7.8.3 source.

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
