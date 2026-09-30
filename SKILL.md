---
name: bend-build
description: Guidelines for building efficient, maintainable software in Bend. Use when designing or implementing Bend libraries, algorithms, cryptography, serialization, or data structures; covers representation choices, language internals, generated code, performance, proofs when required, and keeping the edit-check loop fast.
---

# Build good software in Bend

Build the complete thing the user asked for. Start from the problem, the algorithm, and how the machine will execute it. Keep the design small, direct, fast, and understandable. These guidelines apply to implementation work; they do not turn every task into a formal-verification project.

## Understand before building

- Read the existing project and establish the intended public API, inputs, outputs, error behavior, workload, and constraints. Reuse working code and preserve already-proven properties.
- Check what Bend already provides before implementing a collection, primitive, or abstraction. Prefer suitable native facilities; inspect their actual semantics and costs.
- Study the installed language: its guide, Base definitions, compiler lowering, runtime, and representative working programs. Follow important operations into generated C. Do not import assumptions from another language or another Bend version.
- Record the relevant compiler version and target. Stock Bend means stock Bend: no hidden compiler fork, patched generated C, or unsupported extensions. Explore compiler modifications only when explicitly authorized.

## Design from first principles

Work out the algorithm's dataflow, access patterns, ownership, intermediate state, and asymptotic costs before choosing representations. Choose a representation because it fits those operations, not because it is familiar, convenient to recurse over, or easy to prove.

Aim for a small public API and a direct execution path. Avoid unnecessary frameworks, wrapper layers, duplicated implementations, compatibility representations, and generators that add more complexity than they remove. Helpers should clarify the algorithm or improve actual generated code. More abstraction is not automatically better; neither is blindly inlining everything.

Do not mistake a convenient proof model for the implementation architecture. When proofs are required, build the efficient algorithm and prove it. Do not weaken the algorithm, its API, or its performance target to obtain easier laws.

## Representation rules

- **No linked lists for array-shaped work.** Bytes, words, buffers, indexed state, vectors, lookup tables, and serialized data need packed arrays, supported slices, or suitable fixed records. Use linked structures only when the algorithm actually needs their operations or semantics.
- **No FFI for the implementation.** Do not hand the requested algorithm to C, Rust, Python, a crypto library, or another foreign implementation. External libraries may serve as independent test and benchmark references. Build scripts must not compute the result in place of Bend.
- Keep the entire public path efficient, including inputs and outputs. A fast internal operation followed by expansion into byte lists is not a fast API. Remove pointless conversions instead of hiding them outside the benchmark.
- Use fixed scalar records for small fixed state when they lower well. Use native arrays for indexed storage and suitable native maps for lookup. Inspect the lowering: names such as `Array` do not by themselves establish contiguous storage, constant-time indexing, or cheap updates.
- Follow affine ownership deliberately. Know what is consumed, shared, cloned, allocated, and retained. Do not repeatedly copy a large buffer to make a helper signature convenient.
- Specify byte order, logical length versus capacity, partial words, empty input, bounds, padding, and invalid-input behavior. Do not leave these as accidental implementation details.
- Treat memory consumption, copying, and allocation as design costs from the start. Define memory targets precisely: process RSS, incremental overhead, live heap, and retained output are different measurements.

Proof-only lists and trees can be useful models when verification is requested. They must not leak into the production representation or import graph; a theorem about a model needs a bridge to the implementation users actually call.

## Build the whole path

Implement the public API through to its actual result. Exercise it with real inputs early, including empty, boundary, partial, and invalid cases. Finish the required operations rather than polishing one helper indefinitely. Keep the project organized around the code's responsibilities, with implementation, tests, and benchmarks easy to find; add specification and proof directories only when they serve the task.

Do not stop at scaffolding, TODOs, checkpoint commits, a toy example, or a private fast path that callers cannot use. Track what remains against the user's full objective. Preserve useful work while simplifying; remove obsolete production paths once their replacements are established.

## Make performance real

Inspect the emitted C and optimized assembly for the hot path. Look for traversal, boxing, allocation, copying, helper transitions, generic arithmetic, failed inlining, large state transfers, register spills, and code size. Read the corresponding language source to understand why the compiler generated them.

Use a credible optimized reference and comparable inputs. Distinguish algorithm variants—Ethereum Keccak-256 is not SHA3-256—and software-only versus hardware-accelerated baselines. Pin versions, flags, and machine details. Do not slow the reference or change the workload to meet a ratio.

Measure the real public operation, stating which preparation, cloning, output allocation, conversion, and startup costs are included. Retain outputs so the compiler cannot eliminate the work. Use long enough batches for timer resolution, warmups, repeated alternating measurements, and raw samples. Verify full outputs independently; a benchmark checksum alone is not a correctness test.

Optimize one attributable change at a time. Fixed rotations, specialization, fusion, rolling windows, scheduling, and alternative layouts are hypotheses to measure. Unrolling can worsen spills and code size. Keep supported improvements, reject regressions, and do not assume fewer source-level operations mean faster execution.

Record performance targets as explicit ratios over agreed workloads, for example `Bend time / reference time <= 2`. Report failing rows and the worst case. If the target remains unmet, say so; do not claim a proof, a passing test, or a favorable average satisfies it.

## When proofs are required

Keep the user's laws and semantics stable. Prove the actual exported implementation against a separate, reviewable specification. Compose representation, primitive, loop, and public-API theorems instead of expanding an entire computation unnecessarily. Efficient proof structure should support the chosen implementation, not dictate a slower one.

Do not add axioms, unsafe declarations, admitted holes, or a specification that calls the implementation it is supposed to validate. Run the kernel gate and use targeted mutations to check that claimed properties detect real mistakes. A timeout is not a logical rejection.

Be exact about what is established: universal public-API refinement, a component/model theorem, an unproved representation bridge, or finite differential tests. Keep the trusted compiler, runtime, toolchain, and hardware explicit. Functional correctness does not establish cryptographic security, constant-time execution, or compiler correctness.

## Be impatient: keep the loop fast

Slow iteration is the most expensive bug in a Bend project. Proof checking, regeneration and queueing can quietly turn a one-line fix into an hour and a half. Treat that as a defect to fix, not weather to wait out.

- **Measure the loop first, then again whenever it feels slow.** Time every stage of one real iteration: edit, regenerate, check, failure localization, queueing. Write the numbers down. If checking (or anything around it) is more than half of the loop, or one iteration takes over about five minutes, stop the feature work and optimize the tooling before continuing. Report the before and after in seconds.
- **Iterate on the smallest thing that can fail.** Check the single file you touched before any full run. Regenerate only the generators whose inputs changed, in one pass. Run the full check only when the single files pass, and batch several fixes into one full run.
- **Do not localize failures while iterating.** Bisecting a failed group can cost more than the run itself. Read the failing log directly; localize only for the final gate.
- **Group files so shared modules are checked once.** Import-only umbrella files that pull in many roots make each shared module check once per group instead of once per importing file. This gave a 10x or better wall-time cut on a large proof tree. Keep every file covered, and test soundness with a planted wrong proof.
- **Shorten the critical path.** A full run takes as long as its slowest group. Find that file and split it into independent parts, or move its shared setup into a lemma checked once. Look at per-file seconds and peak memory, not only the total.
- **Make generators idempotent.** If a regenerate step needs more than two passes to reach a fixed point, two generators are undoing each other's output. Fix the cycle. Never pay for a fixpoint you can remove.
- **Cache for the dev loop only.** A module cache or incremental checker is fine for iteration if it is kept out of every gate, stamp, and claim, and the docs say so. A result that depends on a cache is not evidence.
- **Do not queue for what does not need a lock.** Reserve one-at-a-time locks for heavy full runs; let single-file checks run freely within memory limits. Cap memory and parallelism so a crowded machine does not stall everyone, and run heavy checks on the machine set aside for them.
- **Write proofs that check fast.** Bend's `Nat` is unary, so comparing two spellings of a large number makes the checker recurse once per unit, and deep recursion overflows the stack, sometimes only under load. State each large constant once, keep sizes symbolic, reach big literals through an equality test instead of a conversion, put the small operand first in additions, and split big proofs into lemmas. Keep every conversion far below the stack limit, and gate with a run at a reduced stack budget so nothing sits near the edge.
- **Never trade soundness for speed.** Shortcuts change how you iterate, not what counts as checked. The final gate stays the full, uncached, fully localized check with every pinned tool.

## Deliver something usable

Provide a tested minimal usage example, straightforward build/test commands, and reproducible benchmarks where performance matters. Avoid machine-specific paths in instructions. Ensure evidence corresponds to the shipped source, not a previous binary or abandoned experiment.

Publish only within the user's authorization and chosen visibility. Check the actual uploaded commit and a clean consumer build/import when packaging. Report what works, measured performance, any requested proof coverage, and unfinished requirements plainly. Do not label an incomplete objective complete.
