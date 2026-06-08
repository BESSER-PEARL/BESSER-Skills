# BESSER Skills

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/agent--skills-compatible-brightgreen)](https://agentskills.io)
[![BESSER](https://img.shields.io/badge/BESSER-v7.8.3-orange)](https://github.com/BESSER-PEARL/BESSER)

[Agent Skills](https://agentskills.io) for [BESSER](https://github.com/BESSER-PEARL/BESSER), the low-code model-driven engineering platform. These skills give AI coding agents deep knowledge of BESSER's metamodel, code generators, troubleshooting patterns, and contributor workflows -- without needing the full BESSER codebase in context.

Works with any agent that supports the Agent Skills standard: **Claude Code**, **Cursor**, **Cline**, **Windsurf**, **GitHub Copilot**, and [40+ others](https://agentskills.io).

## Skills

| Skill | Description |
|-------|-------------|
| [`besser-user`](skills/besser-user/SKILL.md) | End-user guide: domain modeling, code generation, PlantUML, web editor |
| [`besser-generators`](skills/besser-generators/SKILL.md) | Per-generator operations, safe customization, template overrides |
| [`besser-troubleshooting`](skills/besser-troubleshooting/SKILL.md) | Diagnosis guide for install, import, runtime, and deployment issues |
| [`besser-dev`](skills/besser-dev/SKILL.md) | Contributor guide: adding generators, tests, docs, PR workflow |

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
| `besser-user` | Imports from `besser.BUML` or `besser.generators`, questions about modeling with BESSER, mentions of B-UML |
| `besser-generators` | Questions about generator output, safe customization, template overrides, generation errors |
| `besser-troubleshooting` | Error messages, import failures, installation problems, deployment issues related to BESSER |
| `besser-dev` | Contributing to BESSER, adding generators, writing tests, documentation structure |

## Benchmark Results

Skills evaluated across 8 test scenarios (2 per skill), with-skill vs
without-skill baseline. The baseline has no skills loaded but full read
access to the BESSER source tree. Run with Claude Code (Claude Sonnet 4).

| Metric | With skills (v0.1.0) | Without skills (baseline) |
|--------|----------------------|----------------------------|
| **Pass rate** | **100%** | 100% |
| **Mean response time** | **36.5s** | 138.4s |
| **Mean token usage** | **25,181** | 48,007 |

vs the baseline, skills are **74% faster** and use **48% fewer tokens** at
the same correctness. Skills give the agent the relevant patterns
up-front and skip the read-the-codebase phase entirely.

### Key Findings

- **100% pass rate** — every assertion across all 8 evals passes.
- **74% faster** on average; the largest gains are on complex tasks (eval-7 add-generator: -75%; eval-3 safe-customization: -76%; eval-2 PlantUML backend: -77%).
- **Token usage is normalised** — stddev drops from ~19.8k (baseline) to ~1.4k (with skill). Effort becomes predictable.

### Per-eval timing

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

Full benchmark data and methodology: [`benchmarks/iteration-1/benchmark.md`](benchmarks/iteration-1/benchmark.md)

## Evals

Eval definitions are in [`evals/evals.json`](evals/evals.json). Each eval specifies a prompt, the target skill, and expected output assertions.

**Methodology**: For each eval, two independent agents run in parallel -- one with the skill loaded (no codebase access), one without (full codebase access as baseline). Responses are graded against binary pass/fail assertions checking correctness, content presence, and negative checks.

See [`benchmarks/iteration-1/benchmark.md`](benchmarks/iteration-1/benchmark.md) for the full methodology and analysis.

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
    iteration-1/                          # 8-eval with-skill vs baseline
```

The `besser-user` and `besser-generators` skills follow the
[skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
progressive-disclosure pattern: `SKILL.md` stays under ~300 lines and
points into `references/` for detail when it's needed.

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
