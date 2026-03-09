# BESSER Skills Benchmark -- Iteration 2

## Methodology

### Setup

Four skills evaluated across 8 test cases (2 per skill):
- `besser-user` -- end-user modeling and generation workflow
- `besser-generators` -- per-generator operations and safe customization
- `besser-troubleshooting` -- install, import, and runtime debugging
- `besser-dev` -- contributor workflows (adding generators, tests, docs)

### Test Cases

| Eval | Skill | Prompt Summary |
|------|-------|---------------|
| eval-1 | besser-user | Model a university system and generate SQLAlchemy ORM |
| eval-2 | besser-user | Convert PlantUML to FastAPI backend with GET/POST only |
| eval-3 | besser-generators | How to safely add custom endpoints without losing changes |
| eval-4 | besser-generators | Fix "Invalid DBMS" error: postgres vs postgresql |
| eval-5 | besser-troubleshooting | Fix ImportError: String vs StringType |
| eval-6 | besser-troubleshooting | Fix spaces in names and duplicate enum literals |
| eval-7 | besser-dev | Step-by-step guide to add a new GraphQL generator |
| eval-8 | besser-dev | Test patterns, fixtures, and tmp_path usage |

### Execution

For each test case, two independent agents were spawned in parallel:

1. **With-skill**: Agent reads the relevant SKILL.md first, then answers. No codebase access.
2. **Without-skill (baseline)**: Agent answers with full BESSER codebase read access. No skills loaded.

Total runs: 8 test cases x 2 configurations = **16 agent runs**.

### Grading

Each test case has 4-6 binary (pass/fail) assertions checking correctness, content presence, and negative checks. Total assertions: 38 per configuration.

---

## Results

### Pass Rate

| Config | eval-1 | eval-2 | eval-3 | eval-4 | eval-5 | eval-6 | eval-7 | eval-8 | Mean |
|--------|--------|--------|--------|--------|--------|--------|--------|--------|------|
| With skill | 6/6 | **4/5** | 4/4 | 4/4 | 4/4 | 4/4 | 6/6 | 5/5 | **97.4%** |
| Without skill | 6/6 | 5/5 | 4/4 | 4/4 | 4/4 | 4/4 | 6/6 | 5/5 | **100%** |

**One failure found**: eval-2 with-skill failed assertion "passes http_methods parameter" because the `besser-user` skill does not document `BackendGenerator`'s `http_methods` parameter. The agent suggested regex post-processing instead. The baseline agent found `http_methods` by reading the source code.

### Timing (seconds)

| Config | eval-1 | eval-2 | eval-3 | eval-4 | eval-5 | eval-6 | eval-7 | eval-8 | Mean |
|--------|--------|--------|--------|--------|--------|--------|--------|--------|------|
| With skill | 49.8 | 48.4 | 63.9 | 35.9 | 32.6 | 36.3 | 82.6 | 49.9 | **49.9** |
| Without skill | 139.8 | 155.2 | 186.2 | 59.4 | 98.0 | 102.5 | 214.4 | 151.3 | **138.4** |
| **Speedup** | 64% | 69% | 66% | 40% | 67% | 65% | 61% | 67% | **64%** |

### Token Usage

| Config | eval-1 | eval-2 | eval-3 | eval-4 | eval-5 | eval-6 | eval-7 | eval-8 | Mean |
|--------|--------|--------|--------|--------|--------|--------|--------|--------|------|
| With skill | 24,059 | 23,587 | 24,296 | 22,576 | 21,640 | 21,994 | 26,041 | 22,788 | **23,373** |
| Without skill | 40,021 | 47,086 | 82,537 | 27,982 | 27,551 | 29,205 | 57,324 | 72,346 | **48,007** |
| **Savings** | 40% | 50% | 71% | 19% | 21% | 25% | 55% | 69% | **51%** |

---

## Analysis

### Key Findings

1. **Skills deliver massive efficiency gains.** With-skill runs are **64% faster** and use **51% fewer tokens** on average. This is a substantial improvement over iteration-1 (35% faster, 25% fewer tokens) -- the full 8-eval suite reveals the true advantage more clearly.

2. **One correctness gap found.** The `besser-user` skill does not document `BackendGenerator`'s `http_methods` parameter, causing eval-2 to fail one assertion. The agent fell back to suggesting regex-based post-processing -- a fragile and incorrect approach. **Fix: add `http_methods` to the besser-user skill's Running a Generator section.**

3. **Token usage is remarkably consistent with skills.** With-skill runs range from 21,640 to 26,041 tokens (stddev 1,413). Without-skill runs range from 27,551 to 82,537 (stddev 19,849). Skills normalize agent effort by providing the knowledge upfront instead of requiring variable-cost exploration.

4. **Savings scale with task complexity.** The biggest token savings are on tasks requiring deep codebase exploration:
   - eval-3 (customization patterns): **71% savings** -- baseline read 33 files
   - eval-8 (test patterns): **69% savings** -- baseline read 20+ test files
   - eval-7 (add generator): **55% savings** -- baseline read 41 files
   - eval-4 (simple error): **19% savings** -- answer is quick to find either way

5. **Baseline discovers patterns skills don't cover.** The without-skill agent found:
   - `MethodImplementationType.BAL` and `MethodImplementationType.CODE` for generating custom REST endpoints (eval-3) -- not in `besser-generators` skill
   - A documentation bug in `docs/source/generators/sql.rst` that says `'postgres'` instead of `'postgresql'` (eval-4)
   - The `SUPPORTED_GENERATORS` dict name and `get_filename_for_generator()` registration details (eval-7)

### Comparison with Iteration 1

| Metric | Iteration 1 (4 evals) | Iteration 2 (8 evals) | Change |
|--------|----------------------|----------------------|--------|
| With-skill pass rate | 100% | 97.4% | -2.6pp (found a gap) |
| Without-skill pass rate | 100% | 100% | Same |
| Speed improvement | 35% | 64% | +29pp |
| Token savings | 25% | 51% | +26pp |

The larger eval set reveals both the true efficiency advantage and the first skill content gap.

---

## Concrete Skill Improvements Identified

| # | Skill | What to Add | Source |
|---|-------|-------------|--------|
| 1 | `besser-user` | `http_methods` parameter for `BackendGenerator` | eval-2 failure |
| 2 | `besser-generators` | `MethodImplementationType.BAL`/`CODE` for model-level custom endpoints | eval-3 baseline discovery |
| 3 | `besser-generators` | Note about `docs/source/generators/sql.rst` documentation bug | eval-4 baseline discovery |
| 4 | All skills | Rewrite descriptions in 3rd person per skills.sh spec | Format review |
| 5 | All skills | Add `license`, `compatibility`, `metadata` frontmatter fields | skills.sh publishing |
| 6 | `besser-dev` | Fix "main" to "master" branch reference | Factual error |

---

## Files

```
iteration-2/
  benchmark.json          -- machine-readable benchmark data
  benchmark.md            -- this document
  eval-{1..8}-*/
    eval_metadata.json    -- prompt + assertions
    with_skill/
      outputs/response.md -- agent output
      timing.json         -- tokens and duration
      grading.json        -- assertion pass/fail results
    without_skill/
      outputs/response.md
      timing.json
      grading.json
```
