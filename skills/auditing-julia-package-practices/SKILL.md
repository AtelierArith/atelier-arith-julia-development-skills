---
name: auditing-julia-package-practices
description: Use when you review an existing or in-progress Julia package against this repository's development practices and want prioritized, evidence-based findings and concrete remediation suggestions
---

# Audit Julia package practices

Review the package against the practices in this skill collection. The task is
an audit: inspect the package, cite evidence, and suggest changes. Do not edit
the package under review unless the user separately asks for fixes.

## Establish the package and its Julia target

Start at the package root and inspect its instructions, project metadata, and
layout:

```sh
rg --files --hidden -g '!.git/**' -g '!Manifest.toml' -g '!**/.DS_Store'
```

Read `AGENTS.md` and other applicable project instructions before evaluating
conventions. Read `Project.toml` to identify the package name, dependencies,
compatibility bounds, workspace projects, and declared Julia support. Evaluate
the package against its declared compatibility range as well as Julia 1.13
practices; do not recommend syntax that exceeds its minimum supported Julia
version without calling out the compatibility impact.

## Audit areas

Check only the areas that apply to this package. Use the corresponding skill as
the detailed standard; avoid reporting the same issue under multiple headings.

### Package structure and dependencies

- Confirm the root `Project.toml` and `src/<PackageName>.jl` form a loadable
  package, with focused source files and a clear module entry point.
- Check that runtime, test, documentation, and tooling dependencies are
  declared in the appropriate projects rather than mixed into `[deps]`.
- For Julia 1.13 development, look for a `[workspace]` that includes applicable
  `test`, `docs`, or `benchmark` projects. Each project must declare the
  packages it imports; workspace projects do not inherit root dependencies.
- Check dependency and Julia compatibility bounds, especially before
  registration or release.

### API and implementation

- Check whether exports and `public` declarations match the documented API.
  Tests should primarily exercise public behavior; import internal helpers
  explicitly when direct tests are justified.
- Look for Julia-idiomatic multiple dispatch, clear configuration and result
  types, useful docstrings, and actionable exceptions.
- Check that performance-sensitive code is inside functions and that measured
  bottlenecks—not guesses—drive optimization. Separate steady-state runtime
  issues from first-load and first-call latency.
- If the package has slow first use, look for measurements before precompile
  workloads and for representative, low-cost workloads without side effects.

### Tests, docs, and tooling

- Check that `test/runtests.jl` and its test environment are reproducible, and
  that a full `Pkg.test()` path exists. Treat parallel runners and focused test
  selection as optional tooling, not substitutes for the full suite.
- Check that user-facing APIs have documentation and that Documenter builds
  and doctests are connected to the package's workflow where appropriate.
- Run or recommend JuliaCheck.jl for supported syntax and rule checks when it
  is available. Review the configured rules and distinguish its style or
  structural findings from correctness, runtime, and API-design findings.
- Use JET.jl, Aqua.jl, JuliaFormatter.jl, or profiling tools only where they
  answer a relevant audit question; do not turn every optional tool into a
  universal requirement.

### Apps and release readiness

- If the package provides a CLI app, inspect its `@main` entry point, `[apps]`
  declarations, argument handling, and stdout/stderr behavior. Mention that
  Pkg app support remains experimental.
- For a package intended for registration, check versioning, compatibility,
  tests, documentation, and registration readiness. Do not recommend a release
  or registration as a default fix for unrelated findings.

## Report findings with evidence

For each finding, include:

1. **Category** — one of:
   - `REGISTRY BLOCKER` — prevents registration or breaks CI/users.
   - `USER-FACING BUG` — affects runtime behavior or correctness.
   - `RECOMMENDED BEFORE ANNOUNCEMENT` — worth fixing before public release.
   - `POLISH` — style, documentation, or optional tooling; not a blocker.
2. **Priority** — high, medium, or low, based on user impact and release risk.
3. **Evidence** — file and line(s), command output, or a concise explanation of
   what is missing. Do not infer a defect from an absent convention alone.
4. **Why it matters** — the concrete consequence for users, maintainers,
   compatibility, or reproducibility.
5. **Suggested change** — a specific implementation or workflow the maintainer
   can review.

Group related observations and avoid repeating a single root cause as several
findings. Separate confirmed issues from questions that need maintainer intent
or additional runtime evidence. End with a short summary of strengths,
highest-priority fixes, and the checks that were or were not run.

If the user asks you to apply fixes, do a brief re-audit afterward. Fixes can
enable checks that were previously hidden (for example, a missing test
environment may have been masking a failing test), so re-run the relevant
commands before giving a final go/no-go.

Do not label optional preferences as violations. In particular, a package does
not need every skill in this repository: assess applicability from its purpose,
declared Julia support, and intended users.

For release-readiness audits, end with a clear go/no-go statement that separates
registry blockers from announcement polish. For example: "GO for registration
once the two REGISTRY BLOCKER items are fixed; add a README before announcing
publicly."
