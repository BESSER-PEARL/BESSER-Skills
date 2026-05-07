# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.3.0] - 2026-05-07

### Skill-creator pass

Reviewed all four skills against the
[skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
guidelines and applied a structural pass.

### Changed
- **All skills**: rewrote `description` frontmatter to be more "pushy" with
  explicit trigger phrases ("Use this skill whenever…", concrete class
  names, file-path patterns, error message keywords), per skill-creator's
  guidance against undertriggering. Each description now also tells Claude
  which skill to defer to for adjacent topics.
- **All skills**: bumped `metadata.version` to `0.3.0` and corrected the
  `metadata.repository` URL casing from `besser-skills` to `BESSER-Skills`.
- **`besser-user` SKILL.md**: shrank from 595 → 314 lines by extracting
  the metamodel reference, PlantUML notation, state machines, agents, and
  GUI modeling into `references/*.md`. SKILL.md now keeps the workflow,
  decision tables, quick-start examples, and the most common generator
  invocations.
- **`besser-generators` SKILL.md**: shrank from 525 → 263 lines by
  splitting per-generator detail into `references/{python-and-data,
  persistence, api-and-web, agents-and-other, debugging}.md`. SKILL.md
  keeps the interface, generator picker, output-directory summary, safe
  customization patterns, and composite architecture.
- **`besser-troubleshooting` SKILL.md**: trimmed the "Generator failures"
  section to a symptom→cause→fix table that defers per-generator detail
  to the besser-generators skill, removing duplication.

### Added
- **`besser-user/scripts/scaffold_model.py`**: prints a ready-to-edit
  `DomainModel` skeleton from a list of class names. Saves agents from
  rewriting boilerplate on every model task.
- **`besser-dev/scripts/scaffold_generator.py`**: scaffolds a new
  generator package (class, Jinja template, smoke-test) inside a BESSER
  checkout. Cuts the "add a generator" workflow down to filling in the
  real logic and registering the result.
- `references/` directories under `besser-user/` and `besser-generators/`
  for progressive disclosure (SKILL.md → references → scripts).

### Notes
- Iteration-2 benchmarks were measured against v0.2.0 SKILL.md content;
  they are retained for historical comparison. A re-run on v0.3.0 is
  warranted to confirm the restructured skills do not regress on the
  97.4% pass rate or the 64% / 51% time/token savings.

## [0.2.0] - 2026-03-03

### Added
- Standalone repository for BESSER Claude Code skills (migrated from `skills/` in the main BESSER repo)
- 4 skills: `besser-user`, `besser-generators`, `besser-troubleshooting`, `besser-dev`
- 8 eval definitions (2 per skill) in `evals/evals.json`
- Benchmark results for iteration-1 (4 evals) and iteration-2 (8 evals)
- README with installation instructions, benchmark results, and project structure
- CONTRIBUTING guide for skill authors

### Content improvements (v0.2.0 vs v0.1.0)
- `besser-user`: Added `http_methods` parameter documentation for `BackendGenerator`
- `besser-generators`: Added `MethodImplementationType.BAL/CODE` pattern for model-level custom REST endpoints
- `besser-dev`: Fixed branch reference from `main` to `master`
- All skills: Rewrote descriptions in 3rd person per Agent Skills spec
- All skills: Added `license`, `compatibility`, and `metadata` frontmatter fields

## [0.1.0] - 2026-02-15

### Added
- Initial skill drafts created inside the main BESSER repository
- 4 skills covering user workflows, generator operations, troubleshooting, and contributor workflows
- 4-eval benchmark (iteration-1): 100% pass rate, 35% faster, 25% fewer tokens with skills
