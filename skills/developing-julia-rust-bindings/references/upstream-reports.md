# Executable upstream reports

## Attribute the failure

1. Capture the exact dependency commit, crate version, Cargo features, Rust toolchain, platform, and invocation.
2. Reproduce the failing operation in a small Rust crate using the same dependency. Remove Julia, RustCall, random state, and unrelated wrapper machinery where possible. Check expected values independently and assert the discrepancy.
3. If standalone Rust succeeds, narrow the Julia/RustCall boundary: ownership, generated signature, checked references, scalar layout, or wrapper preconditions. Report to the responsible project only after obtaining evidence.
4. Inspect current upstream source or run against its current revision; distinguish a verified reproduction from a source-based inference. Search existing issues to avoid duplicate reports.

Never deliberately introduce undefined behavior to manufacture a reproducer. For alignment failures, allocate a known aligned owner and deliberately select an offset within it; keep its lifetime valid. For binary formats, include a tiny byte generator or committed fixture and explain its independently known value.

## Report contents

Use a concrete title describing the operation and wrong result/panic. Include:

- Expected behavior and actual behavior, including exit status or panic.
- Complete `Cargo.toml` and `src/main.rs` (or minimal Julia project when the boundary itself is faulty).
- Exact commands to fetch the pinned source and execute the MWE.
- Revision/toolchain/features/platform and whether latest upstream was executed or merely inspected.
- Relevant source location and a concrete proposed correction. Label untested proposals as proposals.
- Wrapper workaround and its practical limits, if relevant.

Actually execute the included MWE before posting. The report must be reproducible without the full wrapper repository or undocumented local files. Prefer deterministic values and include complex, order, or dtype details when they cause the fault. Preserve the reproducer in the wrapper repository so an upstream fix can later be verified.

For multiline issue text, use a structured API body or a temporary UTF-8 file with `gh issue create --body-file`; avoid shell interpolation of Markdown. Confirm the returned issue URL and retain it with the reproducer. If posting fails, keep the concrete draft and explain the failure rather than claim an issue exists.

A workaround does not close the upstream bug. Remove it only after the fixed revision is selected and the original MWE plus wrapper regression pass.
