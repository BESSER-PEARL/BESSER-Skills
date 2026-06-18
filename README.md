# BESSER Skills

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/agent--skills-compatible-brightgreen)](https://agentskills.io)
[![BESSER](https://img.shields.io/badge/BESSER-v7.8.3-orange)](https://github.com/BESSER-PEARL/BESSER)

[Agent Skills](https://agentskills.io) for [BESSER](https://github.com/BESSER-PEARL/BESSER), the low-code model-driven engineering platform. These skills give AI coding agents deep knowledge of BESSER's metamodel, code generators, troubleshooting patterns, and contributor workflows -- without needing the full BESSER codebase in context.

With these skills an agent can build **UML class models** -- classes, attributes, associations, inheritance -- plus state machines and chatbot agents, then generate working code (Python, SQL, REST APIs, Django, React, and more) straight from the model.

The same knowledge lets an agent **draw correct UML class diagrams in B-UML and embed them straight into your Markdown docs** -- B-UML is BESSER's own model representation, so the diagram is valid, runnable, and *is* the model rather than a throwaway picture. See [Drawing correct UML diagrams](#drawing-correct-uml-diagrams).

Works with any agent that supports the Agent Skills standard: **Claude Code**, **Cursor**, **Cline**, **Windsurf**, **GitHub Copilot**, and [40+ others](https://agentskills.io).

## Skills

| Skill | Description |
|-------|-------------|
| [`besser-user`](skills/besser-user/SKILL.md) | End-user guide: UML class modeling (classes, associations, inheritance), code generation, **drawing correct class diagrams for documentation**, PlantUML, web editor (import & render) |
| [`besser-generators`](skills/besser-generators/SKILL.md) | Per-generator operations, safe customization, template overrides |
| [`besser-troubleshooting`](skills/besser-troubleshooting/SKILL.md) | Diagnosis guide for install, import, runtime, and deployment issues |
| [`besser-dev`](skills/besser-dev/SKILL.md) | Contributor guide: adding generators, tests, docs, PR workflow |

## Drawing correct UML diagrams

AI agents are constantly asked to capture a system's structure while
documenting it -- and they routinely get the modeling wrong: invalid
multiplicities, associations pointing the wrong way, inheritance reversed.
With `besser-user` loaded, the agent expresses the design directly in
**B-UML, BESSER's own model representation**, and embeds it straight into
a Markdown file:

```python
from besser.BUML.metamodel.structural import (
    DomainModel, Class, Property, Multiplicity,
    BinaryAssociation, Generalization, StringType, IntegerType,
)

publication = Class(name="Publication", attributes={
    Property(name="title", type=StringType)})
book = Class(name="Book", attributes={
    Property(name="pages", type=IntegerType)})
author = Class(name="Author", attributes={
    Property(name="name", type=StringType)})

# Book is a Publication
book_is_a = Generalization(general=publication, specific=book)

# A Book is written by 1..* Authors; an Author writes 0..* Books
written_by = Property(name="writtenBy", type=author, multiplicity=Multiplicity(1, "*"))
writes     = Property(name="writes",    type=book,   multiplicity=Multiplicity(0, "*"))
book_author = BinaryAssociation(name="book_author", ends={written_by, writes})

model = DomainModel(
    name="Library",
    types={publication, book, author},
    associations={book_author},
    generalizations={book_is_a},
)
assert model.validate()["success"]
```

The same B-UML works **two ways**:

- **As documentation** -- a precise, readable structural model embedded in
  your `README`, design doc, or `.md` spec, to document **any** project --
  no code generation required, even one that will never use BESSER's
  generators. Drop it into
  [editor.besser-pearl.org](https://editor.besser-pearl.org) (Import →
  B-UML) whenever you want the rendered visual class diagram.
- **As a real model** -- it isn't a drawing *of* the system, it *is* the
  system: `validate()`-checked and fed straight to any generator (Python,
  SQL, FastAPI, Django, React, …). No separate, drifting diagram to keep
  in sync.

You don't have to generate anything: a correct B-UML diagram is a useful
**documentation artifact on its own**, in any repo, whether or not it ever
touches a BESSER generator. And when you *do* generate, the embedded B-UML
is the source of truth, so the documentation and the generated code never
diverge.

## Installation

### Using the skills CLI (any agent)

```bash
# Install all skills
npx skills add BESSER-PEARL/besser-skills --all

# Install a specific skill
npx skills add BESSER-PEARL/besser-skills --skill besser-user

# Target a specific agent
npx skills add BESSER-PEARL/besser-skills --skill besser-user -a claude-code
npx skills add BESSER-PEARL/besser-skills --skill besser-user -a cursor
npx skills add BESSER-PEARL/besser-skills --skill besser-user -a cline
```

### Manual installation (Claude Code)

Copy a skill directory into your project's `.claude/skills/` or your global `~/.claude/skills/`:

```bash
# Project-level
cp -r skills/besser-user .claude/skills/besser-user

# User-level (global)
cp -r skills/besser-user ~/.claude/skills/besser-user
```

### Manual installation (Cursor / other agents)

Copy the `SKILL.md` content into your agent's rules or instructions file (e.g., `.cursorrules`, `.github/copilot-instructions.md`).

## How It Works

Skills follow the [Agent Skills](https://agentskills.io) open standard. The agent loads only the skill name and description at startup (a few hundred tokens each). When your task matches a skill's description, the agent reads the full `SKILL.md` into context and applies its knowledge.

**When do skills activate?**

| Skill | Triggers on |
|-------|------------|
| `besser-user` | Imports from `besser.BUML` or `besser.generators`, questions about modeling with BESSER, mentions of B-UML, or drawing a correct class diagram for documentation (even without code generation) |
| `besser-generators` | Questions about generator output, safe customization, template overrides, generation errors |
| `besser-troubleshooting` | Error messages, import failures, installation problems, deployment issues related to BESSER |
| `besser-dev` | Contributing to BESSER, adding generators, writing tests, documentation structure |

## Benchmark Results

Skills evaluated across 8 test scenarios (2 per skill), with-skill vs
without-skill baseline. The baseline has no skills loaded but full read
access to the BESSER source tree. Run with Claude Code.

**Correctness** was re-run for v0.2.0 ([`benchmarks/iteration-2/`](benchmarks/iteration-2/benchmark.md),
graded against BESSER v7.8.3): the skill-equipped agent passes **38/38
(100%)**, matching the baseline — while reading only the relevant skill
files, no BESSER source. **Performance** figures are carried over from the
v0.1.0 benchmark ([`benchmarks/iteration-1/`](benchmarks/iteration-1/benchmark.md));
the v0.2.0 changes are skill-text only and do not alter the agent's work
pattern.

| Metric | With skills | Without skills (baseline) | Measured |
|--------|-------------|----------------------------|----------|
| **Pass rate** | **100% (38/38)** | 100% (38/38) | v0.2.0 (iteration-2) |
| **Mean response time** | **36.5s** | 138.4s | v0.1.0 (iteration-1) |
| **Mean token usage** | **25,181** | 48,007 | v0.1.0 (iteration-1) |

At the same correctness, skills are (per the v0.1.0 perf run) **74% faster**
and use **48% fewer tokens** — the agent gets the relevant patterns up-front
and skips the read-the-codebase phase entirely.

### Key findings

- **100% pass rate at v0.2.0** — every assertion across all 8 evals passes,
  with-skill and baseline alike.
- **The re-run earned its keep:** it caught a real, pre-existing API error in
  the skill — `SQLGenerator.generate(sql_dialect=...)`, which would
  `TypeError` (the dialect is a constructor argument) — fixed before release.
  See [`benchmarks/iteration-2/benchmark.md`](benchmarks/iteration-2/benchmark.md).
- **Performance (v0.1.0):** token usage is also far more consistent — stddev
  drops from ~19.8k (baseline) to ~1.4k (with skill).

### Per-eval timing (v0.1.0 carry-over)

```
Eval                       with skill    baseline    Δ
eval-1 (model+gen)            45.2s       139.8s    -68%
eval-2 (PlantUML)             35.8s       155.2s    -77%
eval-3 (customization)        44.3s       186.2s    -76%
eval-4 (DBMS error)           29.0s        59.4s    -51%
eval-5 (import error)         23.5s        98.0s    -76%
eval-6 (construction)         25.0s       102.5s    -76%
eval-7 (add generator)        53.9s       214.4s    -75%
eval-8 (test patterns)        35.6s       151.3s    -76%
```

Correctness data: [`benchmarks/iteration-2/benchmark.md`](benchmarks/iteration-2/benchmark.md).
Performance data and full methodology: [`benchmarks/iteration-1/benchmark.md`](benchmarks/iteration-1/benchmark.md)

## Evals

Eval definitions are in [`evals/evals.json`](evals/evals.json). Each eval specifies a prompt, the target skill, and expected output assertions.

**Methodology**: For each eval, two independent agents run in parallel -- one with the skill loaded (no codebase access), one without (full codebase access as baseline). Responses are graded against binary pass/fail assertions checking correctness, content presence, and negative checks.

See [`benchmarks/iteration-2/benchmark.md`](benchmarks/iteration-2/benchmark.md) (latest, v0.2.0 correctness) and [`benchmarks/iteration-1/benchmark.md`](benchmarks/iteration-1/benchmark.md) (v0.1.0 performance) for the full methodology and analysis.

## Project Structure

```
besser-skills/
  skills/                                 # installable skills
    besser-user/
      SKILL.md
      references/                         # progressive-disclosure references
        metamodel.md                      # full B-UML structural reference
        plantuml.md                       # PlantUML notation
        state-machines.md
        agents.md                         # chatbot/agent modeling
        gui-models.md                     # GUI for WebApp/Django
        delivering-models.md              # .py vs. Markdown delivery, code-vs-docs
      scripts/
        scaffold_model.py                 # prints a starter DomainModel
    besser-generators/
      SKILL.md
      references/
        python-and-data.md                # Python, Java, Pydantic, JSONSchema, RDF
        persistence.md                    # SQLAlchemy, SQL
        api-and-web.md                    # Backend, Django, WebApp, React, Flutter
        agents-and-other.md               # BAF, Qiskit, Terraform, Pytorch, TF
        debugging.md                      # generation-failure recipes
    besser-troubleshooting/SKILL.md
    besser-dev/
      SKILL.md
      scripts/
        scaffold_generator.py             # scaffolds a new generator package
  evals/                                  # eval definitions
    evals.json
  benchmarks/                             # benchmark results
    iteration-1/                          # v0.1.0: 8-eval correctness + performance
    iteration-2/                          # v0.2.0: correctness re-run (perf carried over)
```

The `besser-user` and `besser-generators` skills follow the
[skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
progressive-disclosure pattern: `SKILL.md` stays lean (a few hundred lines)
and points into `references/` for detail when it's needed.

## Related

- [BESSER](https://github.com/BESSER-PEARL/BESSER) -- the main platform
- [BESSER Documentation](https://besser.readthedocs.io/)
- [BESSER Web Editor](https://editor.besser-pearl.org/)
- [BESSER Examples](https://github.com/BESSER-PEARL/BESSER-examples)
- [Agent Skills Specification](https://agentskills.io/specification)

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on improving existing skills or adding new ones.

## License

Apache-2.0. See [LICENSE](LICENSE) for the full text.
