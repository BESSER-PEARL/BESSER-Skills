# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

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
