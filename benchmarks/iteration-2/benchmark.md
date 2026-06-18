# BESSER Skills Benchmark — v0.2.0 (iteration-2)

Correctness re-run of the four BESSER skills for the v0.2.0 release. Same
eight prompts and assertions as iteration-1, scored **with-skill** vs a
**no-skill baseline** that reads the released BESSER source.

## Methodology

For each of the 8 evals, two agents answer the prompt:

- **with_skill** — may read only the relevant `skills/besser-<X>/` directory
  (SKILL.md + the references/ and scripts/ it points to). No BESSER source,
  no web, no other skill.
- **without_skill (baseline)** — full read access to the released **BESSER
  v7.8.3** source tree (`BESSER-upstream`, branch `master`). No skills.

Each answer is then graded by an independent grader agent against the binary
assertions in that eval's `eval_metadata.json`.

**Scope of this iteration:** correctness (pass rate) only. Timing and token
figures are **carried over from the v0.1.0 benchmark** (see
`../iteration-1/`) and are not re-measured here — the 0.2.0 changes are
skill-text only and do not change the agent's work pattern. See each eval's
`timing.json` (marked with a carry-over note).

## Results — pass rate

| Eval | with skill | baseline |
|------|------------|----------|
| eval-1 university-model      | **6/6** | 6/6 |
| eval-2 plantuml-backend      | **5/5** | 5/5 |
| eval-3 safe-customization    | **4/4** | 4/4 |
| eval-4 invalid-dbms          | **4/4** | 4/4 |
| eval-5 import-error          | **4/4** | 4/4 |
| eval-6 construction-errors   | **4/4** | 4/4 |
| eval-7 add-generator         | **6/6** | 6/6 |
| eval-8 test-patterns         | **5/5** | 5/5 |
| **Total**                    | **38/38 = 100%** | 38/38 = 100% |

Full per-eval answers and per-assertion verdicts are under each
`eval-*/with_skill/` and `eval-*/without_skill/` directory.

## A bug the re-benchmark caught

The first 0.2.0 run scored **with-skill 37/38**. eval-4 (invalid-dbms)
failed one assertion: the answer called `SQLGenerator(...).generate(sql_dialect="postgresql")`,
which raises `TypeError` — `generate()` takes no arguments.

Root cause was a **pre-existing error in the skill** (shipped since v0.1.0):
`skills/besser-generators/references/persistence.md` documented
`gen.generate(sql_dialect="sqlite")`. Verified against the BESSER source,
`SQLGenerator` takes `sql_dialect` in its **constructor** and `generate()`
takes no arguments (only `SQLAlchemyGenerator` takes `dbms` in `generate()`).

`persistence.md` was corrected (constructor usage + an explicit note
contrasting the two SQL generators), and eval-4 with-skill was re-run
against the fixed skill — now **4/4**. The table above reflects the shipped
v0.2.0 skill.

## Takeaway

At v0.2.0 the skills reach the same 100% correctness as the no-skill
baseline while reading only the relevant skill files (no BESSER source). The
re-benchmark also did its job as a regression check: it surfaced a concrete
API error in the skill text, which was fixed before release.
