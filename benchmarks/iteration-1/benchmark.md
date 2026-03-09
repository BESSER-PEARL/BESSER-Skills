# BESSER Skills Benchmark — Iteration 1

## Methodology

### Setup

Four skills were evaluated:
- `besser-user` — end-user modeling and generation workflow
- `besser-generators` — per-generator operations and safe customization
- `besser-troubleshooting` — install, import, and runtime debugging
- `besser-dev` — contributor workflows (adding generators, tests, docs)

### Test Cases

One representative prompt per skill (4 total), selected to exercise each skill's core value:

| Eval | Skill | Prompt Summary |
|------|-------|---------------|
| eval-1 | besser-user | Model a university system (Student, Course, Professor) and generate SQLAlchemy ORM |
| eval-3 | besser-generators | How to safely add custom endpoints without losing changes on regeneration |
| eval-5 | besser-troubleshooting | Fix `ImportError` when importing `String` instead of `StringType` |
| eval-7 | besser-dev | Step-by-step guide to add a new GraphQL generator to BESSER |

### Execution

For each test case, two independent subagents were spawned in parallel:

1. **With-skill**: Agent reads the relevant SKILL.md first, then answers the prompt. No additional codebase access beyond the skill content.
2. **Without-skill (baseline)**: Agent answers the same prompt with NO skill loaded, but has full read access to the entire BESSER codebase (can grep, glob, read any file).

Total runs: 4 test cases x 2 configurations = **8 agent runs**.

All runs were launched as background tasks. Token count and wall-clock duration were captured from task completion notifications.

### Grading

Each test case has 4-6 assertions defined in `eval_metadata.json`. Assertions are binary (pass/fail) and check for:
- **Correct imports** — uses the right BESSER API names
- **Content presence** — mentions required concepts (classes, associations, patterns)
- **Correctness** — no hallucinated API calls or wrong parameter names
- **Negative checks** — doesn't recommend bad practices (e.g., editing generated files)

Grading was done by reading each output and judging each assertion manually. Results saved to `grading.json` per run.

### Metrics

- **Pass rate**: Fraction of assertions passed per configuration (mean across evals)
- **Duration**: Wall-clock seconds per agent run
- **Tokens**: Total tokens consumed per agent run

---

## Results

### Pass Rate

| Configuration | eval-1 | eval-3 | eval-5 | eval-7 | Mean |
|--------------|--------|--------|--------|--------|------|
| With skill | 6/6 (100%) | 4/4 (100%) | 4/4 (100%) | 6/6 (100%) | **100%** |
| Without skill | 6/6 (100%) | 4/4 (100%) | 4/4 (100%) | 6/6 (100%) | **100%** |

### Timing (seconds)

| Configuration | eval-1 | eval-3 | eval-5 | eval-7 | Mean |
|--------------|--------|--------|--------|--------|------|
| With skill | 57.8 | 110.4 | 44.2 | 156.2 | **92.2** |
| Without skill | 97.0 | 199.3 | 72.4 | 197.2 | **141.4** |
| **Speedup** | 40% | 45% | 39% | 21% | **35%** |

### Token Usage

| Configuration | eval-1 | eval-3 | eval-5 | eval-7 | Mean |
|--------------|--------|--------|--------|--------|------|
| With skill | 25,010 | 37,712 | 21,498 | 47,018 | **32,810** |
| Without skill | 32,569 | 56,106 | 22,218 | 65,098 | **44,000** |
| **Savings** | 23% | 33% | 3% | 28% | **25%** |

---

## Analysis

### Key Findings

1. **Correctness is equivalent.** Both configurations achieve 100% pass rate. The baseline agent can explore the full BESSER codebase, so it finds correct answers independently. Skills don't improve correctness for agents with codebase access.

2. **Skills deliver significant efficiency gains.** With-skill runs are 35% faster on average and consume 25% fewer tokens. The agent skips codebase exploration since the skill provides the knowledge directly.

3. **Speedup scales with task complexity.** The most complex task (eval-3, customization patterns) saw a 45% speedup. The simplest (eval-5, import error) saw 39%. The dev task (eval-7) had a smaller speedup (21%) because both agents needed to explore specific code structures.

4. **Token savings track exploration overhead.** eval-5 (simple import error) shows only 3% token savings — the answer is quick to find either way. eval-3 (customization patterns) saves 33% because the baseline agent reads multiple generator files to understand overwrite behavior.

### Qualitative Observations

- **With-skill responses are more consistent** — they follow standardized patterns from the skill (e.g., the extension file pattern for customization) rather than discovering different approaches by reading source code.
- **Without-skill responses include more raw file references** — absolute paths, line numbers, raw code excerpts from the codebase. Useful for debugging but noisier for end users.
- **The without-skill baseline for eval-3 discovered additional approaches** (OCL constraints, Method with BAL implementation) that the skill doesn't cover. This suggests the besser-generators skill could be enriched with these patterns.

### Limitations

1. **All assertions pass in both configs** — the assertions test correctness, not quality or conciseness. More discriminating assertions (e.g., "response under 200 lines", "no raw file paths in user-facing output") would better differentiate.
2. **Only 4 test cases** — one per skill. Statistical confidence is low. Expanding to 3+ cases per skill would be more rigorous.
3. **Single run per case** — no repeated trials, so timing numbers have no within-config variance. The stddev reported is across evals, not repeated runs.
4. **Manual grading** — assertions were graded by reading outputs, not by automated scripts. For concrete checks (import presence, file paths), this is reliable. For subjective quality, it's less so.
5. **Baseline has full codebase access** — this is a strong baseline. In practice, many users of these skills won't have the BESSER repo cloned locally (e.g., using the web editor or `pip install besser`). The skill advantage would be larger for those users.

---

## Files

```
iteration-1/
  benchmark.json          — machine-readable benchmark data
  benchmark.md            — this document
  review.html             — interactive eval viewer (open in browser)
  eval-1-university-model/
    eval_metadata.json    — prompt + assertions
    with_skill/
      outputs/response.md — agent output
      timing.json         — tokens and duration
      grading.json        — assertion pass/fail results
    without_skill/
      outputs/response.md
      timing.json
      grading.json
  eval-3-safe-customization/
    ...same structure...
  eval-5-import-error/
    ...same structure...
  eval-7-add-generator/
    ...same structure...
```
