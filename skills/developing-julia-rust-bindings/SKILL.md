---
name: developing-julia-rust-bindings
description: Use when implementing or debugging Julia bindings to a Rust library with RustCall.jl or a native ABI. Covers memory ownership, Julia dispatch, reproducible builds, and isolating upstream failures with executable reproductions.
---

# Julia–Rust bindings

Build idiomatic Julia APIs backed by Rust operations, with explicit ownership and evidence that failures belong to the correct layer. Follow the package's existing Julia development and test conventions.

## Establish the contract

Inspect the Rust public API, actual dependency revision and enabled Cargo features, Julia wrapper, build script, and binding mechanism before selecting an implementation. For RustCall.jl, check the installed version and its generated API. For a native ABI, verify symbol names, integer widths, scalar representations, and buffer contracts.

Map Rust capabilities to Julia/Base/LinearAlgebra methods by category. Distinguish native implementation, Julia fallback, partial support, and missing support. Keep reproducible inventory snapshots separate from the evolving implementation mapping. Record limitations and verified revisions rather than calling a category complete because one entry point works.

Read [references/colmat.md](references/colmat.md) when working on Colmat.jl. It describes Colmat-specific examples and how to check them against the current checkout.

## Preserve native ownership

- Retain the owning Rust handle for every native view. Do not retain a borrowed pointer beyond the call unless its owner and lifetime are demonstrably protected.
- Guard use-after-close and bounds before native access. Keep close idempotent and synchronize operations that can invalidate owners or views.
- Validate dtype, shape, buffer length, strides, byte order, and alignment before pointer-based reads or writes. Cached Julia metadata is insufficient if a native owner can be replaced. Validate against the current owner at the boundary.
- Treat C-order versus column-major storage, complex component layout, empty arrays, noncontiguous views, and scalar/rank conversions as explicit contracts.
- When using RustCall, use the installed version's checked reference or handle mechanism. Do not copy raw-owner exceptions between versions without checking why they are needed.
- Define copy, deepcopy, aliasing, and read-only behavior separately. Julia deepcopy must preserve the object's concrete type; an ordinary copy may intentionally materialize another type when documented.

Prefer native operations and standard Julia dispatch on package-owned types. Avoid method piracy. Define `+`, `*`, `\`, `svd`, and similar interfaces only with clear return types, mutation rules, promotion, and errors. Keep decomposition reconstruction conventions explicit, especially complex adjoints.

## Verify the boundary

Use small independent fixtures or a standalone Rust program to avoid comparing a wrapper against itself. Cover all advertised scalar types and representative empty, rectangular, complex, aliasing, and closed-owner cases. For binary loaders, exercise source mutation, garbage collection, nonnative byte order, unaligned input, malformed lengths, and preservation of NaN payloads/signed zeros where promised.

Run focused tests while developing, then the package's canonical `julia --project -e 'using Pkg; Pkg.test()'` and relevant executable examples. Reuse an active test session and read its failure log before restarting. Report actual assertions/files and runtime versions only from completed runs. Run the repository's formatting and native checks appropriate to the change.

## Isolate and report failures

Read [references/upstream-reports.md](references/upstream-reports.md) when a dependency may be faulty. First reproduce the failure without Julia to distinguish Rust library defects from RustCall and wrapper defects. Check whether the caller satisfied the Rust API's preconditions before attributing an upstream panic.

For a confirmed RustCall binding defect, isolate the generated code or call signature in an executable Julia/Rust reproducer. Include a concrete proposed correction if the evidence supports one.

Keep workarounds in the wrapper or its native adapter; preserve dependency checkouts for reproducibility. Recheck whether the current upstream revision still contains the fault. Retain an executable reproducer and link any report in the project's implementation notes.

Post an upstream report only when the user has authorized it.
