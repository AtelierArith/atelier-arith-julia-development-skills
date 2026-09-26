---
name: documenting-julia-package
description: Use when you add, build, or publish documentation for a Julia package with Documenter.jl: the `docs/` layout, `docs/Project.toml`, `makedocs`/`deploydocs`, `@docs` and docstring blocks, doctests, and GitHub Pages deployment
---

# Documenting a Julia package with Documenter.jl

[Documenter.jl](https://documenter.juliadocs.org/stable/) builds a single,
cross-linked HTML site from Markdown files plus the package's docstrings, and
publishes it to GitHub Pages. This skill sets up the standard `docs/` layout,
wires doctests so examples cannot go stale, and deploys on CI.

Assume the package is named `MyPkg` and lives at the repository root (see
[[generating-julia-package]] for the package layout). Write docstrings as
described in [[developing-julia-package]].

## Step 1: Create the `docs/` layout

```
MyPkg/
├── docs/
│   ├── Project.toml   # doc-build environment
│   ├── make.jl        # build script
│   └── src/
│       └── index.md   # landing page (the name is mandatory for HTML output)
├── src/
│   └── MyPkg.jl
└── Project.toml
```

`DocumenterTools.generate` can create this skeleton, but hand-writing it is
simple enough for the default layout.

## Step 2: Set up the doc build environment

**Julia 1.13 development baseline:** add `docs` to the package's `[workspace]`
so the documentation environment resolves together with the package. Include
other development projects such as `test` and `benchmark` in the same list when
they exist:

```toml
[workspace]
projects = ["docs"] # Include "test" or "benchmark" too when those projects exist.
```

Declare Documenter and the local package in `docs/Project.toml`. Workspace
projects do not inherit the root package's dependencies:

```toml
[deps]
Documenter = "e30172f5-a6a5-5a46-863b-614d45cd2de4"
MyPkg = "<UUID from the root Project.toml>"

[sources]
MyPkg = {path = ".."}

[compat]
Documenter = "1"
```

Resolve the doc environment from the repository root:

```sh
julia --project=docs -e 'using Pkg; Pkg.instantiate(; workspace=true)'
```

Do **not** add Documenter to the package's own `[deps]` — it is a doc-build
dependency, not a runtime one.

## Step 3: Write `docs/make.jl`

```julia
using Documenter, MyPkg

makedocs(;
    sitename = "MyPkg.jl",
    modules = [MyPkg],
    authors = "Your Name",
    pages = [
        "Home" => "index.md",
        "API reference" => "api.md",
    ],
)

deploydocs(; repo = "github.com/USER_NAME/MyPkg.jl.git")
```

- `modules = [MyPkg]` tells Documenter which modules own the docstrings; that
  also prevents a `Base` method you extend from dragging in unrelated docs.
- If the package was added by local path (a dev dependency), also pass
  `remotes = nothing`, otherwise Documenter tries to build source links against
  a remote that does not exist.
- `deploydocs` needs no protocol prefix in `repo` (no `https://`, no `git@`).
- Keep `deploydocs` out of local builds only if you must; it is a no-op outside
  CI unless it decides to deploy.

## Step 4: Add docstrings and pages

`docs/src/index.md` is the landing page. Splice docstrings into Markdown with
`@docs` (one object per line), and use `@meta` to set the module for a page:

````markdown
# MyPkg.jl

```@meta
CurrentModule = MyPkg
```

## Reference

```@docs
fit_model
FitResult
```
````

- `@autodocs` includes every docstring in a module; `@contents` builds a
  table of contents; `@index` builds a flat docstring index.
- Cross-reference within the docs with `[`fit_model`](@ref)` and section
  headings with `[Home](@ref)`.
- Control the sidebar order and grouping with the `pages` argument (shown in
  Step 3) and hide pages with `Documenter.hide`.

## Step 5: Build locally

```sh
$ julia --project=docs docs/make.jl
```

Output lands in `docs/build/`. Documenter uses pretty URLs
(`src/foo.md` → `foo/index.html`), which browsers do not resolve for local
`file://` paths, so preview over HTTP instead:

```sh
$ julia -e 'using LiveServer; serve(dir="docs/build")'
# or
$ python3 -m http.server --bind localhost --directory docs/build
```

Add `docs/build/` to `.gitignore`; never commit build output to the main branch.

## Step 6: Doctests keep examples honest

A `jldoctest` block is executed and its output compared during `makedocs`
(failures are fatal by default). Use this for any example a reader can run:

````markdown
```jldoctest
julia> 1 + 1
2
```
````

- Definitions do not carry between blocks; give blocks the same label
  (`jldoctest mylabel`) to share scope, or use `setup = :(...)` for short
  definitions.
- Filter non-deterministic output (timings, addresses, versions) with
  `filter = r"..." => s"..."` or a page-level `DocTestFilters` in an `@meta`
  block.
- Doctests inside docstrings run only if their module is listed in `modules`.
- Update outdated doctests deliberately with `doctest(MyPkg; fix = true)` (or
  `makedocs(; doctest = :fix)`), then review the diff before committing.

To run doctests as part of the normal test suite, add Documenter to the test
project (see [[testing-julia-package]]) and call `doctest` in
`test/runtests.jl`; it behaves like a `@testset`:

```julia
using Test, Documenter, MyPkg

@testset "MyPkg" begin
    doctest(MyPkg; manual = false)  # manual = false checks docstrings only
end
```

Run it with the other tests (see [[testing-julia-package]]).

## Step 7: Deploy to GitHub Pages

Create `.github/workflows/documentation.yml`:

```yaml
name: Documentation

on:
  push:
    branches: [main]   # your development branch
    tags: '*'
  pull_request:
  workflow_dispatch:

jobs:
  build:
    permissions:
      contents: write   # required to push to gh-pages with GITHUB_TOKEN
      pull-requests: read
      statuses: write
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: julia-actions/setup-julia@v3
        with:
          version: '1'
      - uses: julia-actions/cache@v3
      - name: Install dependencies
        shell: julia --color=yes --project=docs {0}
        run: |
          using Pkg
          Pkg.instantiate(; workspace=true)
      - name: Build and deploy
        run: julia --color=yes --project=docs docs/make.jl
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- In the repository's **Settings → Pages**, set the source to the `gh-pages`
  branch (root). Documenter creates the branch on first deploy.
- **Same-repo deploy:** `GITHUB_TOKEN` works and needs `contents: write`. For
  tagged versions, TagBot must also be able to trigger the workflow — configure
  it with an SSH deploy key (see below).
- **Cross-repo or tag-triggered deploy:** generate an SSH deploy key with
  `DocumenterTools.genkeys(user="USER", repo="MyPkg.jl")`, add the public key as
  a write-enabled **Deploy key** on the target repository, and store the
  base64-encoded private key as the `DOCUMENTER_KEY` **repository secret** on
  the repository running the workflow.
- Add the docs badge to `README.md` once deployed:
  `[![](https://img.shields.io/badge/docs-stable-blue.svg)](https://USER.github.io/MyPkg.jl/stable)`.

## Security notes

- `DOCUMENTER_KEY` is an **unencrypted private key** granting write access to
  the target repository. Never print it in logs, paste it into issues, or
  commit it; store it only as an encrypted repository secret. Use the `Secrets`
  tab (not `Variables`) and *repository* secrets (not environment secrets).
- `makedocs` and `doctest` execute the package's code and any code in
  `jldoctest` blocks. Only build documentation for code you trust, and remember
  the doc CI runs that code on every push.

## Pitfalls

- **Missing `index.md`.** HTML output requires `docs/src/index.md`; other pages
  are optional.
- **Docstrings not found.** The owning module must be in `makedocs`'s `modules`,
  and `@docs` resolves names in `Main` unless a `CurrentModule` is set.
- **`remotes = nothing` forgotten** for a locally dev'd package, causing broken
  source links.
- **Committing `docs/build/`.** Gitignore it; generated files belong on
  `gh-pages`, which Documenter manages.
- **Editing doctest output by hand.** Let `doctest(MyPkg; fix = true)` regenerate
  it, then inspect the diff; the fixer can rarely rewrite the wrong snippet.
- **Unpinned Documenter.** A `[compat]` bound stops a new major release from
  breaking your build unexpectedly.
