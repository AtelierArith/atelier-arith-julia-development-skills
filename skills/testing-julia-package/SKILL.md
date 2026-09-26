---
name: testing-julia-package
description: Use when you set up or run tests for a Julia package, including workspace-based test dependencies, Pkg.test(), focused local runs, and optional parallel test files
---

# Testing a Julia package

Use Julia 1.13 as the development baseline. Keep test-only dependencies in a
workspace project so they are resolved together with the package and share the
root manifest.

## Set up the test project

Add `test` to the root package's workspace:

```toml
[workspace]
projects = ["test"]
```

If you are migrating from the older `[extras]`/`[targets]` style, remove those
sections from the root `Project.toml` once `test/Project.toml` exists —
`Pkg.test()` ignores `[targets]` when a `test/Project.toml` is present.

Create `test/Project.toml` and declare the package, `Test`, and every other
package that tests import. Workspace projects do not inherit the root
project's dependencies.

When filling this in, substitute `MyPkg` with the actual package name and UUID
from the root `Project.toml`, and write tests against the package's real public
API.

```toml
[deps]
MyPkg = "<UUID from the root Project.toml>"
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[sources]
MyPkg = {path = ".."}
```

From the package root, resolve the test environment:

```sh
julia --project=test -e 'using Pkg; Pkg.resolve()'
```

Alternatively, add dependencies with Pkg while the test project is active:

```sh
julia --project=test -e 'using Pkg; Pkg.develop(path="."); Pkg.add("Test")'
```

Create `test/runtests.jl` and exercise the public API:

```julia
using Test
using MyPkg

@testset "MyPkg" begin
    @test my_function(2) == 4
end
```

## Run tests

For fast local iteration, run the test entry point directly in the test
environment:

```sh
julia --project=test test/runtests.jl
```

Run the full package test workflow before merging and in CI:

```sh
julia --project -e 'using Pkg; Pkg.test()'
```

`Pkg.test()` runs `test/runtests.jl` in a fresh process with the test-specific
dependencies. It can resolve compatible versions for the test environment;
to require the existing resolution, pass `allow_reresolve=false`.

Commit `Manifest.toml` and `test/Manifest.toml` for reproducible CI, or add
them to `.gitignore` if you prefer CI to resolve fresh each run.

For a focused check, make test files independently runnable and invoke the
relevant file with `--project=test`. If selecting individual testsets or source
line ranges is important, [TestRunner.jl](https://github.com/aviatesk/TestRunner.jl)
can select named testsets or line ranges. Its filters may select more than
intended, so confirm the test summary reports the expected count.

## Optional: run independent test files in parallel

For a large suite of independent files, add ParallelTestRunner to the test
project:

```sh
julia --project=test -e 'using Pkg; Pkg.add("ParallelTestRunner")'
```

Use its file autodiscovery in `test/runtests.jl`; remove manual `include` calls
that assemble the suite:

```julia
using MyPkg
using ParallelTestRunner

runtests(MyPkg, ARGS)
```

Each discovered test file runs in isolation and must load its own dependencies
and define its own testsets. Pass runner options through `Pkg.test`:

```sh
julia --project -e 'using Pkg; Pkg.test(; test_args=["--jobs=4"])'
julia --project -e 'using Pkg; Pkg.test(; test_args=["--list"])'
```

Use parallel execution only when the test files are independent and the added
runner is useful to the project. Keep `Pkg.test()` as the canonical full-suite
check.
