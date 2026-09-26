---
name: using-juliacheck
description: Use when you want to run JuliaCheck.jl, a JuliaSyntax-based rule checker, against Julia source files, choose rules or output formats, or add static code checks to a workflow
---

# Check Julia code with JuliaCheck.jl

JuliaCheck.jl is a rule-based static code checker for Julia source. It parses
files with JuliaSyntax.jl and reports style or structural rule violations. It
complements tests: it does not execute the program or prove correctness.

## Obtain a pinned checkout

JuliaCheck.jl is distributed from its GitHub repository. Check the repository's
releases and select a tagged version compatible with the Julia version in use.
Pin a release tag or commit for repeatable runs; do not rely on a moving branch
in CI.

```sh
git clone --branch <release-tag> --depth 1 https://github.com/tiobe/JuliaCheck.jl.git "/tmp/JuliaCheck.jl"
julia --project="/tmp/JuliaCheck.jl" -e 'using Pkg; Pkg.instantiate()'
```

## Check source files

Run the checker against package source files. By default, JuliaCheck enables all
available rules:

```sh
julia --project="/tmp/JuliaCheck.jl" "/tmp/JuliaCheck.jl/src/JuliaCheck.jl" -- src/MyPkg.jl src/*.jl
```

To select rules, put their IDs after `--enable` and separate the rule list from
the file list with `--`:

```sh
julia --project="/tmp/JuliaCheck.jl" "/tmp/JuliaCheck.jl/src/JuliaCheck.jl" \
  --enable module-name-casing single-module-file -- src/MyPkg.jl
```

Ask the checker for its supported options and installed rules before building
automation around the CLI:

```sh
julia --project="/tmp/JuliaCheck.jl" "/tmp/JuliaCheck.jl/src/JuliaCheck.jl" --help
```

## Choose output for local and CI use

The CLI supports highlighted terminal output, plain text, and JSON. Keep the
human-readable default during local work; use JSON when another tool consumes
the results:

```sh
julia --project="/tmp/JuliaCheck.jl" "/tmp/JuliaCheck.jl/src/JuliaCheck.jl" \
  --output json --outputfile juliacheck.json -- src/*.jl
```

Review the enabled rules against the project's conventions. A checker can
produce findings that do not match a project's style or API policy; enable a
focused rule set when the defaults are too broad. Keep JuliaCheck in a
development or CI tool environment, not the package's runtime dependencies.

For Julia 1.13 language and parser changes, update the pinned checker release
and review its output before making the checker mandatory in CI.
