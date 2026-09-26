---
name: creating-julia-test-env
description: Use when you add tests for Julia packages
---

# Creating a Julia test environment

A minimal Julia package layout looks like this:

```sh
$ tree
.
├── Project.toml
└── src
    └── MyPkg.jl # Assume the package is named MyPkg for this example.

```

To add software tests for this package, follow the steps below.

Step 1: Create the `test` directory, `runtests.jl`, and a `Project.toml` for the test environment.

```sh
mkdir test
touch test/runtests.jl
touch test/Project.toml
```

Step 2: Add a `[workspace]` section to `./Project.toml` (not `test/Project.toml`):

```toml
name = "MyPkg"

...
...<omitted>
...

# Add this section
[workspace]
projects = ["test"]
```

---

**Note:** If you are on Julia 1.11 or older, `[workspace]` is unavailable. You have two options:

- **Keep `test/Project.toml` without `[workspace]`** (supported since Julia 1.2). Pkg merges the package and test projects automatically, so this is the closest to the workspace layout — just omit the `[workspace]` section and continue with Step 3 unchanged.
- **Use the legacy `[extras]`/`[targets]` in the root `./Project.toml` instead of `test/Project.toml`.** When a `test/Project.toml` exists, `Pkg.test` ignores `[targets]`, so this option replaces Steps 1 and 3 rather than adding to them:

```toml
[extras]
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[targets]
test = ["Test"]
```

Step 3: Set up `./test/Project.toml` as follows:

```sh
$ cd test
$ ls
runtests.jl Project.toml
$ julia --project -e 'using Pkg; Pkg.add("Test")'
$ julia --project -e 'using Pkg; Pkg.develop(path="../")'
$ cat Project.toml
[deps]
MyPkg = <UUID for MyPkg goes here>
Test = "8dfed614-e22c-5e08-85e1-65c5234f0b40"

[sources]
MyPkg = {path = ".."}
```

Step 4: Put a simple sample test in `test/runtests.jl` (replace `MyPkg` with your actual package name).

```julia
using Test

using MyPkg

@testset "sample" begin
    @test 1 + 1 == 2
end
```

Step 5: Run the following to verify:

```sh
$ julia --project -e 'using Pkg; Pkg.test()'
```

## Optional: parallel test execution with ParallelTestRunner.jl

The single-file setup above is fine to start. Once the suite grows into several
independent test files under `test/`, [ParallelTestRunner.jl](https://github.com/JuliaTesting/ParallelTestRunner.jl)
runs each file concurrently and in isolation, with autodiscovery. Add it to the
test environment from inside `test/`:

```sh
$ cd test
$ julia --project -e 'using Pkg; Pkg.add("ParallelTestRunner")'
```

Then turn `test/runtests.jl` into a dispatcher and **remove any `include(...)`
calls** that manually assemble the suite — ParallelTestRunner discovers the
files itself:

```julia
using MyPkg
using ParallelTestRunner

runtests(MyPkg, ARGS)
```

Each `test/*.jl` file becomes its own isolated test, so files must be
self-contained: load their own dependencies and define their own testsets. Run
it through `Pkg.test` with `test_args`:

```sh
$ julia --project -e 'using Pkg; Pkg.test(; test_args=["--jobs=4"])'
$ julia --project -e 'using Pkg; Pkg.test(; test_args=["--list"])'
```

See [[running-julia-test]] for the full set of options (`--verbose`,
`--quickfail`, name filtering, and `PTR_NUM_JOBS`).
