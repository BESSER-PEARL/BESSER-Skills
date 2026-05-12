# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

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
