---
name: precompiling-julia-package
description: Use when a Julia package takes 1 second or more to load (TTFL) or a function takes 3 seconds or more on its first call (TTFX) because of JIT compilation, or when the user mentions precompilation, time-to-first-execution, time-to-first-load, slow package load, slow first call, startup latency, or PrecompileTools.jl. Add a PrecompileTools.jl workload before reaching for micro-optimizations.
---

# Reducing Julia TTFX and TTFL with PrecompileTools.jl

Julia compiles methods on demand, so the first time a package is loaded
(**TTFL**, time to first load) or a function is called (**TTFX**, time to first
execution) can dominate interactive latency. `PrecompileTools.jl` lets a package
declare a small, representative workload that Julia runs during precompilation
and caches, so that work is paid once when the package is installed or updated
instead of every time a fresh process starts.

Use this skill when either threshold is exceeded:

- **TTFL ≥ 1 s** — `using MyPkg` is slow.
- **TTFX ≥ 3 s** — the first call to a function is slow.

Treat these as starting signals, not hard rules. A 0.5 s first call in a REPL
loop is still worth fixing even if it is under the threshold; conversely, a
one-shot batch script that runs for hours does not need precompilation of its
own code.

## Step 1: Measure before changing anything

Never add a workload based on a guess. Record the current numbers first so you
can prove the change helped. Always use a clean, up-to-date precompile cache and
skip the user's startup file:

```sh
julia --project -e 'using Pkg; Pkg.precompile()'
```

### Measuring TTFL

Run the load several times in fresh processes and take the minimum — the first
run may just be cold disk cache:

```sh
for i in 1 2 3; do
  julia --startup-file=no --project -e 't=@elapsed using MyPkg; println(round(t, digits=3), " s")'
done
```

### Measuring TTFX

Time the **first** call of each representative function, in a fresh process.
Do not call the function before the timed call, or you will measure the already
compiled second call:

```sh
julia --startup-file=no --project -e '
using MyPkg
t = @elapsed MyPkg.foo(1, 2)
println("TTFX foo: ", round(t, digits=3), " s")
t = @elapsed MyPkg.bar([1.0, 2.0])
println("TTFX bar: ", round(t, digits=3), " s")
'
```

Each call in the same process can share compilation with earlier calls, so when
isolating one function, measure it alone in its own process.

To see *what* is being compiled, add `--trace-compile=stderr`; each line is a
method compiled at runtime and is a candidate for the workload:

```sh
julia --startup-file=no --project --trace-compile=stderr -e 'using MyPkg'
```

## Step 2: Add a PrecompileTools workload

`PrecompileTools.jl` must be a normal dependency of the package (add it to
`[deps]`). Put the workload in the package module, after the code it exercises:

```julia
module MyPkg

using PrecompileTools: @setup_workload, @compile_workload

# ... your package code ...

@setup_workload begin
    # Setup runs ONLY during precompilation, never at load time.
    # Build minimal but representative inputs here.
    x = [1.0, 2.0, 3.0]
    y = 2.0

    @compile_workload begin
        # Exercise the hot paths you want cached.
        result = compute(x, y)
        summarize(result)
    end
end

end # module
```

The workload is package code that runs automatically during precompilation
(`Pkg.precompile()`, or when the package is installed or updated), so
precompiling a package executes that code. Only precompile code you trust.

Key properties that make this work:

- `@setup_workload` wraps the whole block so it runs during precompilation and
  is skipped at runtime. It also catches errors, so a broken workload cannot
  make the package unloadable — but a silently failing workload also does
  nothing, so check that precompilation actually exercises your code.
- `@compile_workload` marks the methods hit inside it for caching (bytecode
  from Julia 1.9+, and native code via pkgimages on 1.9+).
- The cached work is keyed to the Julia version, dependency versions, and the
  package's own source, so it is recomputed automatically when any of those
  change. After editing the workload, re-measure.

### Choosing a good workload

The goal is to cover the methods a user will realistically hit, at the lowest
precompilation cost:

- Use the **real input types** callers use, not `Any` or toy types. Dispatch on
  concrete types is what gets compiled, so `foo(1, 2)` does nothing for a later
  `foo(1.0, 2.0)`.
- Cover the entry points and the functions they transitively call. Exercising a
  top-level API usually compiles most of its callees for free.
- Keep it small. Every line here slows `Pkg.precompile()` at install/update time
  and enlarges the package image. Precompile the top TTFX offenders, not
  everything.
- Avoid side effects: no writing files, starting servers, or mutating global
  state. The block may run in unusual environments (CI, read-only caches).
- If a function is expensive to actually run, call it on tiny inputs — you are
  paying for compilation, not for the numeric result.

## Step 3: Fix invalidations when TTFL does not improve

Sometimes load time is dominated not by missing precompiled code but by
**invalidations**: loading your package makes previously compiled methods from
other packages stale, so they must be recompiled. If TTFL stays high after
adding a workload, add `@recompile_invalidations` around your package code:

```julia
using PrecompileTools
PrecompileTools.@recompile_invalidations begin
    include("api.jl")
    include("backend.jl")
end
```

This detects methods invalidated by loading the package and re-precompiles them
during the package's own precompilation. It is most useful for packages that
extend or interact heavily with other packages' methods. Place it so it wraps
the code responsible for the invalidations.

## Step 4: Verify the improvement

Re-run the Step 1 measurements with a fresh cache:

```sh
julia --project -e 'using Pkg; Pkg.precompile()'
```

Then confirm TTFL dropped below 1 s and the targeted TTFX calls dropped below
3 s. Report the before/after numbers. If nothing improved, check that the
workload actually ran (a swallowed error in `@setup_workload` will leave you
with a no-op) and that the types you exercised match the types callers use.

## Optional: find exact methods with SnoopCompile.jl

When you cannot tell which methods are worth precompiling, `SnoopCompile.jl`
records the full inference and can emit code you can fold into a workload:

```julia
using SnoopCompile
tinf = @snoopi_deep begin
    MyPkg.foo(1, 2)
    MyPkg.bar([1.0, 2.0])
end
```

Use the results to discover hot methods, then write a hand-tuned
`@setup_workload` — a curated workload is usually smaller and faster to
precompile than the raw output. `@snoopr` similarly reports runtime
invalidations. Reach for these only when measurement plus `--trace-compile`
is not enough to locate the cost.

## Pitfalls

- **Over-precompiling.** A workload that pulls in a large dependency graph can
  make installation slow and fail CI precompile timeouts. Precompile the paths
  that actually show up in TTFL/TTFX measurements.
- **Stale numbers.** Cached precompilation is invalidated by any source edit,
  dependency change, or Julia upgrade. Re-measure after each change; do not
  trust a number from before an edit.
- **Testing only one type.** A workload on `Float64` inputs does not help a
  user whose first call uses `Int` or `Matrix{Float64}`.
- **Mistaking compilation for algorithm cost.** If a function is slow on the
  *second* call too, the problem is the algorithm, not TTFX. Precompilation
  will not fix it.
