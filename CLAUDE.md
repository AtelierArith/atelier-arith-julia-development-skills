# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a collection of Claude Code **skill definitions** for Julia development workflows. Each skill lives under `skills/<skill-name>/SKILL.md` and follows a standard frontmatter format consumed by the Claude Code skill system.

## Skill File Format

Every `SKILL.md` must begin with YAML frontmatter:

```markdown
---
name: <skill-name>        # matches the directory name
description: Use when ... # one-line trigger description shown to Claude
---

# Skill Title
...content...
```

The `description` field is critical — it is what Claude reads to decide whether to invoke the skill.

## Julia Conventions Encoded in These Skills

When writing or updating skills that involve Julia, follow the patterns already established:

**Package layout (default: in-place)**
- Default layout writes `./Project.toml` and `./src/MyPkg.jl` directly in the working directory — do NOT create a subdirectory unless the user explicitly asks.
- Do NOT use `Pkg.generate()` for the in-place layout; hand-write the files.
- Generate a UUID with: `julia -E 'using UUIDs; string(uuid4())'`

**Test environment**
- Use the `[workspace]` section in the root `Project.toml` (Julia 1.12+). For Julia 1.11 and older, use `[extras]` + `[targets]` instead.
- The `test/Project.toml` must `Pkg.develop(path="../")` to reference the parent package.

**Running tests**
- Full suite: `julia -e 'using Pkg; Pkg.test()'` (use for CI / pre-merge)
- Fast iteration: `julia --project -e 'using WarmTestRunner; runtests()'` (daemon-backed, reuses a warm worker)
- After `Pkg.build()` or any workspace-structure change, always `stop()` the WarmTestRunner daemon before re-running.
- Targeted test sets: `testrunner --project=. test/runtests.jl L<start>:<end>` — use line ranges targeting only `@test` lines (not the `@testset` declaration line) to avoid the pattern-matching caveat.
