---
name: debugging-julia
description: Use when Julia code errors, throws an exception, produces wrong results, or crashes and you need to find the cause from the command line. Covers capturing and reading stack traces, locating the failing method with @which/@code_warntype/@code_typed, finding potential runtime errors with JET.jl, and narrowing to a minimal reproduction. Do not reach for interactive debuggers (Debugger.jl, Infiltrator.jl, Cthulhu.jl) — they need a human at a TTY.
---

# Debugging Julia code from the CLI

Coding agents run commands, not terminals: there is no TTY to drive an
interactive debugger. Everything here is non-interactive — it produces output on
stdout/stderr and an exit code, so an agent can read it, reason, and iterate.

Do **not** use `Debugger.jl` (`@enter` / the `debug>` REPL), `Infiltrator.jl`
(`@infiltrate` / the `infil>` REPL), or `Cthulhu.jl` (the `@descend` arrow-key
menu). They block on raw keyboard input and full-screen redraws, so they hang or
fail without a human. The information Cthulhu shows is still available
non-interactively through `@code_typed` etc. (see Step 3); use these instead.

## Step 1: Reproduce deterministically

Pin the exact invocation so the failure is repeatable and independent of the
user's shell setup:

```sh
julia --startup-file=no --project script.jl args...
```

Skip `--startup-file=no` only when the failure depends on the user's startup
file. Shrink `args...` and any input files until you have the smallest input
that still fails — a minimal reproduction makes every later step cheaper and the
final fix verifiable.

## Step 2: Capture the full stack trace

Julia prints a stack trace on an uncaught exception, but if your code swallows
it, log it explicitly instead of letting `catch` discard the backtrace:

```julia
try
    main()
catch e
    @error "uncaught" exception = (e, catch_backtrace())
    rethrow()
end
```

To print it directly, use `Base.showerror` with the captured backtrace:

```julia
catch e
    bt = catch_backtrace()
    showerror(stderr, e, bt)
    println(stderr)
end
```

When frames you expect are missing, they were likely inlined: rerun with
`julia --debug-info=2`, and add `Base.@noinline` to the suspect function to keep
its frame visible. Read the trace top-down, but the bug is usually at the first
frame that belongs to **your** code, not in the deepest Base/stdlib frame.

## Step 3: Locate the failing method / inspect inference

`InteractiveUtils` ships with Julia, so these need no dependency:

```julia
@which f(args...)        # which method the call resolves to; a MethodError means none matched
f                        # methods(f) list, or @methods f
@code_warntype f(args)   # inferred types, with unstable ones flagged
@code_typed f(args)      # the type-inferred IR (what Cthulhu shows interactively)
@code_lowered f(args)    # lowered code, no inference
```

For a `MethodError`, the message lists the argument types and the "closest
candidates". Compare the actual argument types against the method signatures to
see which dispatch failed — often a concrete type narrowed somewhere upstream.

`@code_typed` / `@code_warntype` are the non-interactive substitute for
descending in Cthulhu: they print the inferred types and the dispatch decisions
to stdout so you can see exactly where inference falls back to `Any`.

## Step 4: Find potential runtime errors with JET.jl

JET reuses the compiler's inference to report calls that *may* throw before you
hit them, which is useful for `MethodError`-style bugs that depend on input
types you have not tried yet:

```julia
using JET

@report_call target_function(1, 2.0)
```

Pass representative values; JET infers from their types. `report_package(MyPkg)`
covers a whole package. JET's full functionality only works on supported Julia
versions — check `JET.JET_AVAILABLE` after loading, and run it from a temporary
environment (`Pkg.activate(; temp=true)`) if dependency conflicts prevent a
working install. For performance (as opposed to correctness) analysis, see
[[profiling-julia-performance]].

## Step 5: Narrow with logging and assertions

When the trace points at a region but not the exact line, add cheap
instrumentation and rerun:

- `@debug "msg" x y` plus `JULIA_DEBUG=MyPkg` to enable it, so noise stays off by
  default.
- `@assert cond "message"` to fail fast at the point where a wrong value first
  appears.
- Run with `julia --check-bounds=yes` to turn out-of-bounds accesses into errors
  instead of undefined behavior.
- For silent numeric corruption, check `isfinite(x)` / `isnan(x)` at loop
  boundaries.

Remove the instrumentation once the cause is found; do not leave debug prints or
assertions in committed code unless they are genuine invariants.

## Step 6: Minimize and bisect

- Cut the reproduction down: fewer arguments, smaller input, one code path.
- If the failure is recent, `git bisect` between a known-good and known-bad
  commit finds the introducing change.
- Confirm the fix by re-running the exact minimal reproduction, and add a
  regression test (see [[creating-julia-test-env]]).

## Pitfalls

- **Catching and discarding the backtrace.** `catch; end` or `catch e; @warn e`
  loses the stack; always attach `catch_backtrace()`.
- **Reading only the top frame.** The deepest frame is often Base internals; the
  actionable frame is the first in your code.
- **Missing inline frames.** Add `--debug-info=2` and `Base.@noinline`.
- **Trusting a stale compile cache.** If behavior contradicts the source, clear
  the cache (`julia --project -e 'using Pkg; Pkg.precompile()'`) or invalidate it
  after edits.
- **Interactive debuggers in automation.** `Debugger`/`Infiltrator`/`Cthulhu`
  wait for a human and will hang a non-interactive run.
