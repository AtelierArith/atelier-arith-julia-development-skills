---
name: developing-julia-package
description: Use when you write a Julia package
---

# Creating a Julia package

Notes on developing Julia packages.

Assume the package is named `MyPkg`. Substitute your actual package name wherever `MyPkg` appears.

Put the core algorithms in `src/MyPkg.jl`.

```julia
module MyPkg

# Write code here

end
```

Guidelines below.

## Avoid excessive `export`s

- Do not `export` helpers that are only used internally just so tests can reach them. Prefer importing explicitly in tests:

```julia
# test/runtests.jl

using Test
using MyPkg: <internal-only helper>
```

## Julia-idiomatic style

### Multiple dispatch

Prefer splitting behavior across methods instead of a large `if`/`elseif` chain on `isa`, unlike typical Python style.

```julia
# Do not write if else end
function f(x)
    if x isa Integer
        return 2x
    else
        return x
    end
end
```

Instead, use multiple dispatch:

```julia
f(x) = x # generic implementation
f(x::Integer) = 2x # specialized implementation for x::Integer
```

### Type annotations

- On **public** APIs, narrow signatures when it prevents misuse or clarifies the contract.
- Inside the package, avoid over-constraining types everywhere; leave room for the compiler and for generic code.

### Type stability

- On hot paths, avoid return types that vary unpredictably across inputs (type instability hurts specialization).
- Confirm bottlenecks with profiling before micro-optimizing.

## Code formatting (JuliaFormatter)

- Format package sources with **[JuliaFormatter.jl](https://github.com/domluna/JuliaFormatter.jl)** so layout and whitespace stay consistent across contributors and CI.
- Optionally commit a **`.JuliaFormatter.toml`** at the repository root (or rely on defaults) so everyone applies the same rules.

From the package root:

```julia
using JuliaFormatter
format(".")  # formats src/, test/, etc. under the current directory
```

Run formatting before merging substantive edits; wire the same command into CI or pre-commit hooks if the team wants enforcement.

## Performance and allocations

- Measure with **`@benchmark` / `@btime`** from BenchmarkTools.jl rather than guessing.
- Watch unnecessary array copies from slicing and broadcasting; when an in-place API is needed, expose it explicitly (separate function name or keyword argument) so callers opt in.

## Errors and documentation

- Raise **`ArgumentError`**, **`DomainError`**, or other appropriate exceptions; messages should tell the caller what to fix.
- Give **docstrings** to exported/public functions—ordinary docstrings above definitions integrate cleanly with Documenter.jl; use `@doc` when you attach documentation programmatically.
