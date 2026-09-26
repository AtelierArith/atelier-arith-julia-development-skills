---
name: fixing-julia-exports
description: Use when a Julia package exports too much — internal helpers exported only so `Pkg.test` can reach them, unused or stale exports, or namespace pollution and name collisions — or when you need to audit and trim a package's public API. Covers finding `export` statements, classifying public vs internal names, replacing `export` with explicit test imports or the `public` keyword, and verifying with tests and Aqua.jl.
---

# Fixing excessive exports in a Julia package

`using MyPkg` brings every name in `export ...` into the caller's scope. When a
package exports an internal helper just because its own tests need it, that
helper becomes part of the de-facto public API: it shows up in `names(MyPkg)`,
in the docs namespace, and in name-collision warnings for every user of the
package.

The fix is almost always on the **test** side, not the source side: stop
exporting the helper, and import it explicitly in the tests. This skill finds
those exports and removes them safely.

See [[developing-julia-package]] for the general style rule ("Avoid excessive
`export`s"); this skill is the concrete audit-and-fix workflow.

## Step 1: Inventory what the package exports

On Julia 1.11 and later, `names(MyPkg)` lists public names marked with either
`public` or `export`; it is not an export-only inventory. Use
`Base.isexported` when you need to distinguish names imported by `using`:

```sh
$ julia --project -e 'using MyPkg; println(sort(filter(n -> Base.isexported(MyPkg, n), names(MyPkg; all=true))))'
```

```sh
$ rg -n '^\s*export\b' src/
```

`names(MyPkg; all=true)` includes all bindings for comparison. A single
`export` line can list many names, so read the statements rather than grepping
for one name.

## Step 2: Classify each exported name as public or internal

For every exported name, find out whether it is genuinely public:

- **Documented / intended API** — appears in `docs/` (`@docs` blocks), the
  README, or package examples. Keep it exported (or see Step 4 for `public`).
- **Referenced by another package / a user-facing entry point** — public; keep.
- **Used only inside `src/` or only by `test/`** — internal; remove the export.

Search each candidate across the whole repository:

```sh
$ rg -n --glob '!Manifest.toml' '\bhelper_name\b'
```

If the only hits outside `src/` are in `test/`, the export exists solely for
`Pkg.test`, which is exactly the case to fix. Treat docs as the source of
truth: a name that is documented is public even if nothing in `src/` calls it.

## Step 3: Remove the export and import the internal in tests

Before editing, tell the user which files you are about to change (especially
when modifying an existing package). Then delete the name from the `export`
list in `src/MyPkg.jl` — the function itself stays exactly where it is — and
update the test suite to reach it explicitly.

Before (documented only by an accidental export):

```julia
# src/MyPkg.jl
export fit_model, initial_guess   # initial_guess is internal
```

```julia
# test/runtests.jl
using Test
using MyPkg

@testset "initial_guess" begin
    @test initial_guess(x, y) isa NamedTuple   # works only because of the export
end
```

After — import only what the test needs, keeping the public API clean:

```julia
# src/MyPkg.jl
export fit_model
```

```julia
# test/runtests.jl
using Test
using MyPkg
using MyPkg: initial_guess   # explicit internal import

@testset "initial_guess" begin
    @test initial_guess(x, y) isa NamedTuple
end
```

Prefer testing through the public API first. Import an internal directly only
when the helper has meaningful behavior that cannot be exercised through the
public path (see [[developing-julia-package]]).

Notes that matter when editing tests:

- `using MyPkg: name` imports just that name; `import MyPkg` plus a fully
  qualified `MyPkg.name` is the other acceptable form. Avoid pulling in
  `MyPkg` wholesale in a test whose point is to exercise internals.
- Macros need the same treatment: `using MyPkg: @internal_macro`.
- If many tests reach into many internals, that is a signal the tests are
  testing implementation rather than behavior — consider rewriting them, or
  moving shared test fixtures into `test/` instead of exporting from `src/`.

## Step 4: Use `public` (not `export`) when a name is public but unexported

Sometimes a symbol is meant to be callable but should **not** be dumped into
every caller's namespace. On Julia 1.11+, mark it with `public`:

```julia
module MyPkg

public internal_api_for_advanced_users
internal_api_for_advanced_users(x) = ...

end
```

`public` records the name as public API (so doc tooling and users know it is
supported) without adding it to the names exported by `using MyPkg`; callers
still write `MyPkg.internal_api_for_advanced_users`. Do not use `public` as a
substitute for fixing tests that merely needed an import — Step 3 is the fix
for test-only access.

## Step 5: Verify

Run the full test suite; the explicit imports should keep every test passing:

```sh
$ julia --project -e 'using Pkg; Pkg.test()'
```

See [[testing-julia-package]] for test setup and optional focused or parallel
runs while iterating.

Confirm the exported surface shrank:

```sh
$ julia --project -e 'using MyPkg; println(sort(filter(n -> Base.isexported(MyPkg, n), names(MyPkg; all=true))))'
```

A successful cleanup looks like:

- `Pkg.test()` still passes.
- The export list no longer contains the internal name.
- Tests that need the internal name import it with `using MyPkg: name`.

If the package runs quality checks, let [Aqua.jl](https://github.com/JuliaTesting/Aqua.jl)
catch exported-but-undefined names and other API mistakes (see
[[generating-julia-package]] for adding Aqua to the test environment):

```julia
using Aqua
Aqua.test_all(MyPkg)   # includes undefined_exports and unbound_args checks
```

If you are preparing the package for registration, continue with the
pre-registration checklist in [[generating-julia-package]] (compat bounds,
version, full test suite).

## What counts as a breaking change

Removing an export is a **breaking change** for anyone who relied on
`using MyPkg; helper` — even if your intent was that the helper be internal. In
practice:

- If the name was truly internal (undocumented, only used by your own tests),
  removing the export is low-risk; ship it in a patch or minor release.
- If the name appears in docs, README, or examples, it is public: removing its
  export is a breaking change and should follow the package's SemVer policy
  (major bump), or be replaced with `public` if you only want to stop
  auto-injecting it into caller scopes.

## Pitfalls

- **Removing only some occurrences.** A name can be listed on several `export`
  lines or re-listed after a refactor; grep for the name, not just the first
  `export`.
- **Leaving the test import implicit.** After deleting the export, a test that
  still does `using MyPkg` and calls the bare name fails with
  `UndefVarError` — add `using MyPkg: name` (or qualify it).
- **Treating a documented symbol as internal.** Docs are the contract; check
  `docs/` and the README before cutting an export.
- **Using `public` to paper over test access.** `public` does not put the name
  in scope for `using MyPkg`, so tests would still not see it; import it
  explicitly instead.
- **Forgetting macros.** `export @m` and `public @m` have the same semantics;
  import macros the same way in tests.
- **Forgetting downstream consumers.** An export is a promise to every
  package that does `using MyPkg`; when in doubt, deprecate rather than delete.
