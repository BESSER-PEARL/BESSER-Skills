# Contributing to BESSER Skills

Thank you for your interest in improving BESSER skills! This guide covers how to improve existing skills, add new ones, and run evaluations.

## Improving an Existing Skill

1. Edit the `SKILL.md` in the relevant `skills/<name>/` directory
2. Keep the body under 500 lines -- move detailed reference material to a `references/` subdirectory if needed
3. Ensure the YAML frontmatter is valid (name, description, license, compatibility, metadata)
4. Write a trigger-oriented description: open with what the skill provides, then an explicit "Use this skill whenever the user is …" clause listing concrete triggers (symbols, file paths, phrasings). This is the effective activation style used by all four BESSER skills.
5. Run the relevant evals to verify the change doesn't break correctness
6. Submit a pull request with a clear description of what changed and why

## Adding a New Skill

### 1. Create the skill directory

```
skills/my-skill/
  SKILL.md
```

### 2. Write the SKILL.md

Follow the [Agent Skills specification](https://agentskills.io/specification):

```yaml
---
name: my-skill
description: >
  Concise 3rd-person description of what this skill provides and when it
  activates. Max 1024 characters.
license: Apache-2.0
compatibility:
  - claude-code
metadata:
  author: BESSER-PEARL
  version: "0.1.0"
  repository: https://github.com/BESSER-PEARL/besser-skills
---

# Skill Title

Instructions, examples, and reference material here.
```

**Naming rules** for the `name` field:
- 1-64 characters, lowercase letters, numbers, and hyphens only
- Must not start or end with a hyphen
- Must not contain consecutive hyphens
- Must match the parent directory name

### 3. Add eval definitions

Add at least 2 eval entries to `evals/evals.json`:

```json
{
  "id": 9,
  "skill": "my-skill",
  "prompt": "A realistic user prompt that should trigger this skill...",
  "expected_output": "Description of what a correct response should contain"
}
```

### 4. Run a benchmark

Follow the methodology in [`benchmarks/iteration-1/benchmark.md`](benchmarks/iteration-1/benchmark.md):

1. For each eval, spawn two independent agents:
   - **With-skill**: Agent reads the SKILL.md, then answers (no codebase access)
   - **Without-skill**: Agent answers with full BESSER codebase access (no skills)
2. Grade responses against binary pass/fail assertions
3. Record timing and token usage
4. Save results under `benchmarks/iteration-N/`

### 5. Submit a pull request

Include the new skill, eval definitions, and benchmark results.

## Skill Authoring Guidelines

- **Keep it actionable**: Skills should provide step-by-step instructions, not just conceptual overviews
- **Include code examples**: Show complete, runnable code snippets
- **Cover error cases**: Document common pitfalls and their solutions
- **Stay focused**: Each skill should have a clear scope. Don't overlap with other skills
- **Write trigger-oriented descriptions**: open with what the skill provides, then "Use this skill whenever the user is …" with concrete triggers (symbols, file paths, phrasings) — the activation style all four skills use
- **Test with Claude Code**: Verify the skill activates on the right prompts and provides useful responses

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add new skill for BESSER deployment workflows
fix: correct BackendGenerator http_methods documentation in besser-user
docs: update benchmark methodology in README
test: add eval-9 for deployment skill
```

## Code of Conduct

This project follows the [BESSER governance and community guidelines](https://github.com/BESSER-PEARL/BESSER/blob/master/GOVERNANCE.md).

## License

By contributing, you agree to license your contribution under the [Apache-2.0 License](LICENSE).
