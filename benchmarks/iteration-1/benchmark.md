# BESSER Skills Benchmark — v0.1.0

This benchmark measures the four BESSER skills against a baseline agent
with no skill but full read access to the BESSER source code. Same eight
prompts, same assertions, two configurations.

## Methodology

For each of the 8 evals, two subagents run in parallel:

- **with_skill** — read access restricted to `skills/besser-<X>/SKILL.md`
  (the relevant skill) and any `references/*.md` or `scripts/*.py` it
  points to. No BESSER source, no web, no other skill.
- **without_skill (baseline)** — full read access to the BESSER source
  tree, no skills loaded.

Each agent answers the eval prompt; the response is graded against the
binary assertions in `eval_metadata.json`. Timing is captured from the
subagent completion notification.

## Results

### Pass rate

| Eval | with skill | without skill |
|------|------------|----------------|
| eval-1 university-model      | **6/6** | 6/6 |
| eval-2 plantuml-backend      | **5/5** | 5/5 |
| eval-3 safe-customization    | **4/4** | 4/4 |
| eval-4 invalid-dbms          | **4/4** | 4/4 |
| eval-5 import-error          | **4/4** | 4/4 |
| eval-6 construction-errors   | **4/4** | 4/4 |
| eval-7 add-generator         | **6/6** | 6/6 |
| eval-8 test-patterns         | **5/5** | 5/5 |
| **Mean**                     | **38/38 = 100%** | 38/38 = 100% |

Equal correctness at 38/38. The skill-equipped agent reaches the same
answers with much less context.

### Timing (seconds)

| Eval | with skill | without skill | Δ |
|------|-----------|----------------|----|
| eval-1 | 45.2 | 139.8 | -68% |
| eval-2 | 35.8 | 155.2 | -77% |
| eval-3 | 44.3 | 186.2 | -76% |
| eval-4 | 29.0 |  59.4 | -51% |
| eval-5 | 23.5 |  98.0 | -76% |
| eval-6 | 25.0 | 102.5 | -76% |
| eval-7 | 53.9 | 214.4 | -75% |
| eval-8 | 35.6 | 151.3 | -76% |
| **Mean** | **36.5** | **138.4** | **-74%** |

Every eval is faster with the skill. The largest absolute gains are on
complex tasks (eval-7 add-generator, eval-3 safe-customization, eval-2
plantuml-backend) — the ones where the baseline does the most reading.

### Tokens

| Eval | with skill | without skill | Δ |
|------|-----------|----------------|----|
| eval-1 | 24,126 | 40,021 | -40% |
| eval-2 | 24,790 | 47,086 | -47% |
| eval-3 | 25,704 | 82,537 | -69% |
| eval-4 | 25,669 | 27,982 |  -8% |
| eval-5 | 24,427 | 27,551 | -11% |
| eval-6 | 24,540 | 29,205 | -16% |
| eval-7 | 26,776 | 57,324 | -53% |
| eval-8 | 25,416 | 72,346 | -65% |
| **Mean** | **25,181** | **48,007** | **-48%** |

Token usage is also notably more *consistent* with the skill (range
24.1k–26.8k) than without (range 28.0k–82.5k). The skill removes the
read-the-codebase phase of the agent's work, normalising effort.

## Summary

| Metric | with skill | without skill | Skill advantage |
|--------|-----------|----------------|------------------|
| Pass rate   | **100%**  | 100%   | tie (with skill: less context) |
| Mean time   | **36.5s** | 138.4s | **74% faster** |
| Mean tokens | **25,181** | 48,007 | **48% fewer** |

## Findings

1. **Equivalent correctness with one quarter of the time.** Every
   assertion that the baseline gets right, the skill-equipped agent also
   gets right — at 26% of the wall-clock cost.

2. **Largest gains where reading dominates.** eval-7 (add a new
   generator), eval-3 (safe customization), eval-8 (test patterns) all
   require deep navigation of the codebase in the baseline path. With
   the skill, the agent has the relevant patterns already in hand and
   skips the exploration phase.

3. **Token usage is normalised.** Standard deviation of token usage
   drops by an order of magnitude (≈1.4k with skill vs ≈19.8k without).
   This is the "no surprise" property — once skills are loaded, the
   agent's effort is bounded and predictable.

4. **Smallest gains on simple diagnostics.** eval-4 (invalid DBMS) and
   eval-5 (import error) show smaller token deltas because both the
   skill and the baseline can answer them quickly — the baseline finds
   the offending file fast, the skill has the recipe directly.

## Files

```
iteration-1/
  benchmark.json            machine-readable benchmark data
  benchmark.md              this document
  eval-{1..8}-*/
    eval_metadata.json      prompt + assertions
    with_skill/
      outputs/response.md   the agent's answer
      timing.json           tokens and duration
      grading.json          per-assertion pass/fail with evidence
    without_skill/
      outputs/response.md   baseline agent's answer
      timing.json           tokens and duration
      grading.json          per-assertion pass/fail with evidence
```
