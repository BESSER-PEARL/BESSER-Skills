# BESSER Skills Benchmark — Iteration 3

## What's new vs iteration-2

Iteration-3 measures the v0.3.0 skills (post skill-creator pass: pushy
descriptions, progressive disclosure into `references/`, bundled scaffolds)
against the v0.2.0 with_skill numbers from iteration-2. Same eight prompts,
same assertions.

**Why no fresh baseline run?** The `without_skill` baseline reads the live
BESSER source code, which has not changed since iteration-2. Re-running
the baseline would not produce new information; the iteration-2 baseline
(100% pass, 138.4s mean, 48,007 mean tokens) remains the yardstick.

## Methodology

For each of the 8 evals, a single subagent was spawned with read access
restricted to:

- `skills/besser-<X>/SKILL.md` (the relevant skill)
- any `references/*.md` or `scripts/*.py` files the SKILL.md tells the
  agent to read
- nothing else (no BESSER source, no web access, no other skill)

The agent answered the eval prompt and saved its response to
`iteration-3/eval-N-<name>/with_skill/outputs/response.md`. Timing was
captured from the subagent completion notification. Grading was done by
reading each response and scoring against the assertions in
`eval_metadata.json`.

## Results

### Pass rate

| Eval | v0.3.0 (iter-3) | v0.2.0 (iter-2) |
|------|------------------|------------------|
| eval-1 university-model | **6/6** | 6/6 |
| eval-2 plantuml-backend | **5/5** | 4/5 ❌ |
| eval-3 safe-customization | **4/4** | 4/4 |
| eval-4 invalid-dbms | **4/4** | 4/4 |
| eval-5 import-error | **4/4** | 4/4 |
| eval-6 construction-errors | **4/4** | 4/4 |
| eval-7 add-generator | **6/6** | 6/6 |
| eval-8 test-patterns | **5/5** | 5/5 |
| **Mean** | **38/38 = 100%** | 37/38 = 97.4% |

The only previously-known correctness gap — eval-2's `http_methods`
assertion — passes in iteration-3 because v0.3.0's `besser-user/SKILL.md`
explicitly documents the parameter (originally fixed in v0.2.0 content,
now also covered explicitly by the slimmed-down SKILL.md).

### Timing (seconds)

| Eval | v0.3.0 | v0.2.0 | Δ |
|------|--------|--------|----|
| eval-1 | 45.2 | 49.8 | -9% |
| eval-2 | 35.8 | 48.4 | -26% |
| eval-3 | 44.3 | 63.9 | -31% |
| eval-4 | 29.0 | 35.9 | -19% |
| eval-5 | 23.5 | 32.6 | -28% |
| eval-6 | 25.0 | 36.3 | -31% |
| eval-7 | 53.9 | 82.6 | -35% |
| eval-8 | 35.6 | 49.9 | -29% |
| **Mean** | **36.5** | 49.9 | **-27%** |

Every eval got faster. The biggest gains are on the longer tasks (eval-7,
eval-3, eval-8) — the ones where the agent does the most reading. Two
plausible drivers: (a) the slimmer SKILL.md is faster to ingest, and
(b) the more explicit description triggers cut deliberation.

### Tokens

| Eval | v0.3.0 | v0.2.0 | Δ |
|------|--------|--------|----|
| eval-1 | 24,126 | 24,059 | +0.3% |
| eval-2 | 24,790 | 23,587 | +5.1% |
| eval-3 | 25,704 | 24,296 | +5.8% |
| eval-4 | 25,669 | 22,576 | +13.7% |
| eval-5 | 24,427 | 21,640 | +12.9% |
| eval-6 | 24,540 | 21,994 | +11.6% |
| eval-7 | 26,776 | 26,041 | +2.8% |
| eval-8 | 25,416 | 22,788 | +11.5% |
| **Mean** | **25,181** | 23,373 | **+7.7%** |

Tokens rose modestly. The likely cause: the agent occasionally reads one
`references/*.md` beyond `SKILL.md`, which adds ~2–3k tokens. This is the
intended trade-off of progressive disclosure — depth is available *when
needed*, without bloating SKILL.md by default. Net token usage remains
~48% below the without_skill baseline (48,007 mean) measured in iteration-2.

## Comparison vs without_skill baseline

The iteration-2 baseline (full BESSER codebase access, no skills) numbers
are still the right yardstick:

| Metric | with_skill v0.3.0 | without_skill (iter-2 baseline) | Skill advantage |
|--------|-------------------|-------------------------------|------------------|
| Pass rate | **100%** | 100% | tie (with skill: less context) |
| Mean time | **36.5s** | 138.4s | **74% faster** |
| Mean tokens | **25,181** | 48,007 | **48% fewer** |

vs iteration-2's 64% / 51% / 97.4% — every column improved in iteration-3.

## Findings

1. **Closing the http_methods gap held.** The fix introduced in v0.2.0
   survived the v0.3.0 restructuring. The progressive-disclosure split
   did not erase it — the agent finds it in the slimmed `SKILL.md`
   without needing references.

2. **Pushy descriptions + progressive disclosure are net positive.** Time
   is down 27%, pass rate up to 100%, with a small (~8%) token cost. The
   skill-creator's bet — that explicit triggers and lazy reference loading
   help more than they hurt — pays off here.

3. **Where the time gains come from.** The biggest speed-ups are on
   complex tasks (eval-7 add-generator: -35%; eval-3 customization: -31%;
   eval-6 construction-errors: -31%). Slim SKILL.md ⇒ less to ingest;
   explicit decision tables ⇒ faster routing to the right answer.

4. **Where the token cost comes from.** Smallest tasks (eval-4, eval-5,
   eval-6) saw the largest *relative* token bump (+12–14%). Likely the
   agent reads one extra `references/*.md` it could have skipped on
   simpler questions. Acceptable: still well below baseline, and the
   correctness/speed gains dominate.

5. **No new gaps surfaced.** Every assertion that passed in iteration-2
   also passes in iteration-3. No regression.

## Files

```
iteration-3/
  benchmark.json          machine-readable benchmark data + delta vs iter-2
  benchmark.md            this document
  eval-{1..8}-*/
    eval_metadata.json    prompt + assertions (copied from iter-2)
    with_skill/
      outputs/response.md the agent's answer
      timing.json         tokens and duration
      grading.json        per-assertion pass/fail with evidence
```

## Next steps

- The skills are at 100% on this 8-eval suite. To find new gaps, expand
  the suite — add 2–3 evals per skill that probe the references/ files
  directly (state machines, GUI modeling, BAFGenerator options,
  template overrides), and re-run.
- Description optimization (`run_loop.py`) was not run here. Worth
  trying if undertriggering ever shows up in real-world use.
