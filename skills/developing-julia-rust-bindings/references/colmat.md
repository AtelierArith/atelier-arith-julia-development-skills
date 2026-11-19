# Colmat.jl verification examples

Inspect the current Colmat.jl checkout to identify the interface and the pinned Rust dependency. Earlier bindings used RustCall handles, while the BLAS/LAPACK backend uses a native shared library and libblastrampoline. The build script, package metadata, and tests in the checkout determine which checks apply.

## RustCall-owned matrices and views

Historical Colmat wrappers exposed owned matrices and borrowed views. Apply the following checks when working on code with that design.

- Validate the native owner's type, shape, and stride under the same synchronization used to access its data. Test views after both garbage collection and explicit owner closure.
- For binary loaders that promise lossless copies, construct fixtures whose expected values and bit patterns are known. Include an unaligned payload, nonnative endianness, C-order data, NaN payloads, and signed zeros where the format supports them.
- Run file-loading reproductions both on the main task and within a Julia task when the Rust parser uses an OS-stack guard. Task stacks can produce a failure that a standalone Rust process does not reproduce.

## BLAS/LAPACK forwarding

- Verify the actual libblastrampoline forwarding target before attributing a result or benchmark to Colmat. A loaded library listing can also include the reference backend.
- For ABI changes, exercise every advertised integer width and scalar layout. Check logical shapes, padded leading dimensions, in-place outputs, and argument error reporting. For factorizations, reconstruct independently and verify interoperability with another backend's factor consumer where available.
- Pin the upstream Rust revision and retain its Cargo features in standalone reproductions. Compare against the currently supported upstream revision before applying or retiring a workaround.
