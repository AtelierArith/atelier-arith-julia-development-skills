---
name: profiling-julia-performance
description: Use when Julia code is slower than expected at steady-state runtime (not just on its first call), or when you suspect type instability or unnecessary memory allocations, or when the user mentions profiling, Profile, JET.jl, AllocCheck.jl, @code_warntype, allocations, type stability, flamegraphs, or optimizing Julia performance. Measure and diagnose with Profile, JET.jl, and AllocCheck.jl before guessing at optimizations.
---

# Profiling Julia performance

When Julia code has insufficient runtime performance, do not start by rewriting
it. Two root causes account for most slow Julia: **type instability** (the
compiler cannot infer concrete types, so it falls back to dynamic dispatch) and
**unnecessary memory allocations** (in hot loops, they trigger garbage
collection and dominate the profile). Both are measurable, and fixing them
usually matters far more than micro-tuning arithmetic.

This skill covers three tools:

- **`Profile`** — Julia's built-in statistical CPU and allocation profiler
  (standard library; there is no separate `Profile.jl` to install).
- **`JET.jl`** — static analysis of type instability and potential errors.
- **`AllocCheck.jl`** — static check that a function *may* allocate.

First confirm the problem is runtime cost, not compilation latency. If the
first call is slow but later calls are fast, the fix belongs in
[[precompiling-julia-package]] instead.

## Step 1: Profile to find the hot code

Profile a **function**, not global scope, and feed it a representative
workload. Code at global scope is not optimized the same way, and profiling it
gives misleading results.

```julia
using Profile

function main()
    # representative workload
end

main()                 # let compilation finish first
Profile.clear()
@profile main()
Profile.print(format=:flat, sortedby=:count)
```

Run it from a script so the workload is reproducible:

```sh
julia --startup-file=no --project -e '
using Profile
include("bench.jl")
main()  # warm up
Profile.clear()
@profile main()
Profile.print(format=:flat, sortedby=:count)
'
```

Read the output by sample count. A large share in `jl_apply_generic`,
`jl_gc_*`, `jl_gc_pool_alloc`, or `macro expansion`/`typeinf` is a strong hint
of dynamic dispatch or allocation pressure. Add `maxdepth=12` and
`sortedby=:overhead` when the tree is noisy, and `noisefloor=2.0` to hide
sampling noise.

For a graphical flamegraph, `ProfileView.jl` provides
`ProfileView.@profview main()`; `PProf.jl` exports `pprof`-compatible profiles
(`PProf.@profile main()` then `pprof -http=:8081 profile.pb.gz`). These are
optional convenience layers over the same samples.

### Allocation profiling

The built-in allocation profiler records the call stacks that allocate:

```julia
Profile.Allocs.clear()
Profile.Allocs.@profile sample_rate=1.0 main()
Profile.Allocs.print()
```

For a quick total, `@time` and `@allocated` report bytes and allocation count:

```julia
@time main()
```

```julia
@allocated main()
```

`@time`/`@allocated` tell you *how much* allocates; the allocation profiler
tells you *where*. Use them together.

## Step 2: Check type instability with JET.jl

JET reuses the compiler's inference to report where dispatch cannot be resolved
statically. Point it at the hot function you found in Step 1, passing
representative **values** (JET infers from their types):

```julia
using JET

@report_opt target_function(1, 2.0)
```

Look for lines like `runtime dispatch detected: f(x::Any)::Any` — those are the
dynamic dispatches to remove. `@report_opt` is noisy by design (it reports all
non-inferable calls, including benign ones in dependencies); focus on the
frames inside your own code. `report_package(MyPkg; target_modules=(MyPkg,))`
analyzes a whole package, and `@report_call` finds potential runtime errors
rather than performance issues.

Three practical caveats:

- JET loads and analyzes the target code, so it executes top-level statements
  and `__init__`. Only analyze code you trust.
- JET integrates tightly with the compiler and only offers full functionality
  on a limited set of Julia versions. Check `JET.JET_AVAILABLE` after loading;
  if it is `false`, JET is a no-op on that Julia version.
- Install and run JET in a temporary environment
  (`Pkg.activate(; temp=true)`) when dependency conflicts (notably with
  `JuliaInterpreter.jl` via `Revise.jl`) prevent a working version. Do not make
  JET a runtime dependency of the package under study.

`@code_warntype target_function(1, 2.0)` remains a useful zero-dependency
spot check: red `Any`/`Union` types in the body mark instability.

## Step 3: Verify allocation-freedom with AllocCheck.jl

Where JET finds *type* problems, `AllocCheck.jl` statically proves whether a
function *may* allocate by inspecting its LLVM IR. Annotate the function you
want to keep allocation-free:

```julia
using AllocCheck

@check_allocs hot_loop(x) = sum(x)
hot_loop(rand(100))     # passes: no allocation proven
```

A violating call raises `AllocCheckFailure`, whose `errors` field points to the
allocating site:

```julia
try
    hot_loop(rand(3, 3))
catch err
    err.errors[1]
end
```

Key details:

- By default `@check_allocs` assumes no exception is thrown and ignores the
  allocations involved in throwing. Add `ignore_throw=false` if throwing paths
  must also be allocation-free.
- Every call into a `@check_allocs` function behaves like a dynamic dispatch
  and incurs small entry allocations. Wrap a **top-level entry point or main
  loop**, not a function called per iteration, and treat allocations at the
  entry as expected; the guarantee applies once the body starts running.
- Only annotate performance-critical functions. Wrapping everything turns the
  check into noise and slows execution.

Unlike `@allocated` (runtime bytes actually allocated) and the allocation
profiler (runtime stacks), AllocCheck is a static guarantee: it catches
allocation on paths your test inputs never exercised.

## Step 4: Fix the root cause

Typical fixes, in rough order of impact:

- **Constrain container element types.** `Any[]` or `[]` forces boxing; prefer
  `Float64[]` or `Vector{Float64}(undef, n)`.
- **Avoid non-`const` globals.** Non-const globals are type-unstable by nature;
  make them `const` or pass them as arguments.
- **Use function barriers.** Wrap unstable setup in a function and call a
  type-stable inner function with concretely typed arguments.
- **Stabilize return types.** A function returning different types on different
  branches forces callers into dynamic dispatch; make all branches return the
  same concrete type.
- **Reduce allocations in hot loops.** Preallocate outputs, use in-place
  (`mul!`, `@views`) operations, and avoid intermediate temporaries.
- **Annotate only where needed.** Add `::T` assertions to document and enforce
  the type at key boundaries, not everywhere.

## Step 5: Re-measure

After each change, redo the measurement that exposed the problem:

```julia
@time main()
```

```julia
Profile.clear(); @profile main(); Profile.print(format=:flat, sortedby=:count)
```

Re-run `@report_opt` and `@check_allocs` on the changed functions and confirm
the reported dispatches/allocations are gone. Report before/after numbers
rather than claiming an improvement. If the hot spot merely moved, follow it —
optimization is a loop, not a single pass.

## Pitfalls

- **Profiling global scope or a cold call.** Always profile a function running
  a representative workload after warm-up, or you will measure compilation.
- **Trusting `@time` without a warm-up.** The first execution includes
  compilation; run the workload once before timing.
- **Chasing JET noise.** `@report_opt` reports dispatch in Base and
  dependencies too. Fix the frames in your code first.
- **Over-annotating.** Too many `@check_allocs`/`::T` add complexity and can
  even slow execution; optimize the measured hot path only.
- **Benchmarking without BenchmarkTools.jl.** For micro-benchmarks, use
  `BenchmarkTools.jl` (`@benchmark`) rather than a single `@time`, which is
  dominated by variance and one-time effects.
