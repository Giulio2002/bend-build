---
name: bend-performance-proofs
description: Design, optimize, benchmark, and formally verify Bend libraries using efficient native representations and generated-code inspection. Use for performance-sensitive Bend algorithms, cryptography, serialization, or data structures with correctness laws.
---

# Fast, proved Bend libraries

Deliver an efficient implementation and laws that describe the actual public implementation. Do not choose a slow representation merely because it makes induction or definitional equality easy. Design from the algorithm's dataflow, memory requirements, and machine operations; then build the proof around that design. Respect the user's narrower scope and explicit overrides.

## Learn the language that is actually installed

Record the Bend version, compiler revision when available, target architecture, and backend. Read the installed guide, Base definitions, compiler lowering, and relevant runtime source before assuming how a datatype behaves. Distinguish Bend versions and forks: available integer widths, arrays, effects, ownership, and native lowering can differ.

Follow a representative operation through source, generated C, optimized assembly, and timing. Establish which operations are intrinsic, how fixed records are flattened, whether arrays are contiguous on the target backend, how affine values move or clone, and where calls allocate. Never infer constant-time indexing or packed storage merely from an API named `Array`.

If compiler source is unavailable locally, locate the matching upstream revision. Keep compiler modifications separate and disclose them; do not quietly depend on a local fork. Use the installed CLI's help instead of guessing commands.

## Representation and API rules

- No FFI implementation of the target algorithm. External implementations are allowed as independent test or benchmark references. Host scripts may orchestrate builds and validation; they must not compute the result in place of Bend.
- Do not use linked lists for bytes, words, vectors, indexed state, tables, or serialization buffers. Use packed arrays, slices where supported, or fixed records. Use a linked structure only where the chosen algorithm actually needs its operations or semantics; explain that need.
- Native maps and other suitable built-ins take precedence over unnecessary replacements, after checking their semantics and performance. A native map does not automatically make every structure built on it efficient.
- Keep public input and output compact. A fast internal decoder followed by expansion into byte lists is still a slow, allocation-heavy public API. Remove unnecessary conversions rather than excluding them from the measured path.
- Define byte order, logical length, capacity, padding, ownership, empty input, partial words, and invalid-input behavior explicitly. Ignore unused capacity only where specified; test dirty unused bytes.
- Use fixed scalar state for small fixed-width algorithms when it lowers well. A fixed record is not a linked-list substitute: inspect whether it becomes registers, stack storage, or repeated allocations.
- Track allocation, peak memory, and copying as algorithmic costs. State precisely whether a memory budget means process RSS, incremental overhead, live heap, or retained output.

Proof-only lists or trees can be useful mathematical models. Keep them out of the production import graph, and prove the representation bridge to the actual runtime API. A theorem about a historical list implementation does not establish the correctness of a new packed implementation.

## Optimize from evidence

Start with a pinned correct baseline and an independent, credible optimized reference. Establish the input sizes, operations, execution target, memory budget, and ratio definition. “At most 2x slower” should be recorded unambiguously as `candidate_time / reference_time <= 2` for the agreed workload set.

Inspect the generated hot path for:

- Repeated traversal, boxing, copying, allocation, and conversion.
- Generic arithmetic or rotation helpers that failed to become native operations.
- Large state passed between helper functions, missed inlining, and repeated round dispatch.
- Register pressure, spills, live temporary ranges, cache traffic, and code size.
- Native word-width mismatches and instructions used by the reference.

Make one attributable change at a time. Fixed rotations, loop specialization, loop fusion, row-wise scheduling, rolling windows, and alternate bit layouts are candidates, not universal improvements. Unrolling can increase spills or instruction-cache pressure. Fewer source lines, helpers, or arithmetic operations do not guarantee faster native code.

Do not edit generated C to obtain a result advertised as a Bend implementation. Experimental C edits may diagnose compiler behavior, but the accepted optimization must be reproducible from checked Bend source and the declared toolchain. Do not silently introduce hardware acceleration; identify the instructions, dispatch, fallback, and additional trust boundary when it is explicitly in scope.

Prefer repeated paired measurements of old and new binaries on the same host. Keep an optimization only when the gain survives measurement noise, required correctness checks, proof checking, and the relevant workloads. Archive useful rejected experiments as evidence without leaving them on the production path.

## Prove the efficient implementation

Write a separate, reviewable specification from the algorithm definition. Share neutral datatypes where helpful; avoid a specification that calls the production implementation or duplicates its optimized scheduling verbatim.

State universal laws over the actual exported functions, including rejection behavior and all output bytes or words. Compose proofs through meaningful boundaries: representation, primitive operations, a round, repeated rounds, absorption or traversal, padding or termination, and public output. Instantiate symbolic component theorems at the real public parameters without expanding every round in the kernel when a composition proof suffices.

Keep the specification and required laws stable during optimization. Changing an internal representation calls for a bridge theorem, not weaker input quantification or a new specification tailored to make the candidate trivially correct. Do not introduce axioms, admitted holes, unsafe proof declarations, test-derived “proofs,” or opaque external correctness assumptions.

Run the actual kernel proof gate. Check that it reaches the public API theorem and the production files shipped to consumers. Re-run after source changes. Use targeted mutations—wrong rotation, round constant, padding bit, output order, index, length check, or round count—to test whether the claimed proof boundary detects real errors. A timeout is not evidence of logical rejection.

Report the exact proof boundary. Distinguish:

- Universal refinement of the public API to an independent specification.
- A component theorem or proof about a model only.
- A missing representation bridge.
- Differential tests and finite examples.
- Trusted kernel, standard library, compiler/runtime, native toolchain, and CPU.

Do not call source-level refinement a proof of the compiler, constant-time execution, or cryptographic security. Formal functional correctness and performance measurements support different claims.

## Benchmark honestly

Use the same algorithm variant, byte contents, workload sizes, and observable outputs. Ethereum Keccak-256 and SHA3-256 are different algorithms. Pin reference revisions and record compiler flags, versions, CPU, architecture, and any hardware acceleration.

Measure the real public operation. Declare whether input preparation, cloning, output allocation, format conversion, initialization, and process startup are included. If the two implementations have different ownership or allocation contracts, disclose the difference rather than calling the workloads identical. Offer isolated-kernel measurements only as separately labeled diagnostics.

Use sufficiently long batches for the timer resolution, warmups, alternating run order, multiple samples, and a robust statistic. Retain raw samples. Validate outputs outside the timed region where possible, and ensure the compiler cannot discard the work. A retained checksum helps prevent dead-code elimination; full-output differential tests are still necessary.

Do not weaken the comparator to meet a ratio. A portable scalar C reference, a hardware-accelerated library, and a standard-library default are distinct baselines. Label them. Do not infer an ARM-versus-x86 result without measurements on both hosts.

Store source and binary identities with benchmark evidence so stale results cannot be mistaken for a changed implementation. Report worst-case ratios and failing rows, not just favorable averages. If a required target is unmet, say so and preserve the remaining work; do not rename the target into a claimed success.

## Ship reproducibly

Keep implementation, independent specification, proofs, tests, benchmarks, and build tools clearly separated. Include a tested minimal usage example, exact proof and test commands, benchmark reproduction instructions, and a concise account of known limits.

Before an authorized publication, verify a clean consumer can build/import the shipped source, check the actual release commit, and ensure proof and benchmark evidence correspond to it. Respect repository visibility and publish only within the user's authorization. For content-addressed packages, verify the downloaded source and record the immutable package identifier; updating documentation does not change an already published package.

Finish with the measured improvement, target status, proof scope, remaining limitations, and links to the published artifacts. Do not substitute a checkpoint or partial theorem for the user's complete objective.
