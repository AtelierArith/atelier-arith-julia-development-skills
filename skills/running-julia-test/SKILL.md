---
name: running-julia-test
description: Use when you run tests, including running a specific testset or line range with TestRunner.jl (`testrunner`) and running a Julia package's test files in parallel with ParallelTestRunner.jl
---

# Running Julia tests

## Running all tests with `Pkg.test()`

Before running tests for a new package, make sure the test environment is already set up. If the package does not have tests yet, follow [[creating-julia-test-env]] first. Tests that use `using Test` must have `Test` available from the package or test project.

To run all of your package's tests, use the following command:

```sh
$ julia --project -e 'using Pkg; Pkg.test()'
```

`Pkg.test()` runs the package's test code in a fresh process, which executes arbitrary code; only run tests you trust.

## Running specific test sets only

To run only particular test sets (for example, those defined as `@testset "label" begin ... end`), use the `~/.julia/bin/testrunner` tool.

If `testrunner` is already installed, this command prints its path:

```sh
$ command -v testrunner
```

If you do not have the `testrunner` command, install it with:

```sh
$ julia -e 'using Pkg; Pkg.activate(); Pkg.Apps.add(url="https://github.com/aviatesk/TestRunner.jl#release")'
```

Install from the `release` branch: it vendors TestRunner's dependencies with rewritten UUIDs so the app does not conflict with the packages in your project. TestRunner requires Julia 1.12 or newer, and the executable lands in `~/.julia/bin`, which must be on your `PATH`.

`#release` is a moving ref, so the installed code can change without notice. For a reproducible install, pin an immutable commit instead: `rev="<commit-sha>"`. `testrunner` is a third-party app that executes your test code, so install and run it only on code you trust.

### Basic Usage

```bash
$ testrunner demo.jl "basic tests"                   # Run the testset named "basic tests"

$ testrunner demo.jl "basic tests" "struct tests"    # Run multiple named testsets

$ testrunner demo.jl '(:(@test startswith(inner_func2(), "inner")))'  # Run a standalone test case
```

See the [guide](https://raw.githubusercontent.com/aviatesk/TestRunner.jl/refs/heads/master/README.md) to learn more.

### Caveat: There are cases where name or regex filters do not work as expected

Sometimes, even after specifying a testset name or a regex, all tests may run instead of just the intended ones (this seems to be due to a pattern matching behavior in TestRunner.jl). Example:

```sh
# Expected: Only "calculate: basic operations" should run (4 tests), but actually all 20 tests run
$ testrunner --project=. test/runtests.jl "calculate: basic operations"
$ testrunner --project=. test/runtests.jl 'r"^calculate: basic operations$"'
$ testrunner --project=. test/runtests.jl L6:11   # Even if you include the @testset declaration line, all tests are executed
```

As a workaround, specify the line range including only the `@test` lines (do not include the `@testset ... begin` declaration line). This way, you can ensure that only the intended testset is executed:

```sh
# In the case where the target @test lines are in lines 7-10 of runtests.jl
$ testrunner --project=. test/runtests.jl L7:10
```

Always confirm whether your filter worked as intended by checking the Total count in the `Test Summary` or the `n_passed` field in the `--json` output.

## Running test files in parallel with ParallelTestRunner.jl

[ParallelTestRunner.jl](https://github.com/JuliaTesting/ParallelTestRunner.jl) runs each file in `test/` concurrently and in isolation, discovering them automatically. Reach for it when the suite is large and its files are independent, since wall-clock time then scales with the number of jobs instead of running serially.

Add it to the test environment:

```sh
$ julia --project=test -e 'using Pkg; Pkg.add("ParallelTestRunner")'
```

Then set up autodiscovery:

1. Remove the `include(...)` statements that manually assemble the suite — ParallelTestRunner discovers the files itself.
2. Replace `test/runtests.jl` with a call to the runner:

```julia
using MyPkg
using ParallelTestRunner

runtests(MyPkg, ARGS)
```

Each `test/*.jl` file becomes its own isolated test, so files must be self-contained (load their own dependencies and define their own testsets).

### Running

Pass the runner's arguments through `Pkg.test` with `test_args`:

```sh
# Use 4 worker processes
$ julia --project -e 'using Pkg; Pkg.test(; test_args=["--jobs=4"])'

# List the discovered tests
$ julia --project -e 'using Pkg; Pkg.test(; test_args=["--list"])'

# Run only tests whose names match the remaining arguments
$ julia --project -e 'using Pkg; Pkg.test(; test_args=["foobar", "widgets"])'
```

Useful options:

- `--jobs=N` — number of worker processes; also settable with the `PTR_NUM_JOBS` environment variable (`--jobs` wins).
- `--verbose` — print more detail while testing.
- `--quickfail` — abort the whole run as soon as one test errors.
- `--list` — list available tests alphabetically.
- Positional arguments filter the tests to run.
