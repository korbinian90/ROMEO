# mritools stack - cross-cutting plan

Working notes from the architecture review of 2026-08-19..21. Deliberately
uncommitted: this is a plan, not a deliverable, and it will go stale as items
land. The full review, with the evidence behind each item, is at
https://claude.ai/code/artifact/1dcb7a4a-4523-46fc-af23-7c5b0ecc3688

Per-repository lists live in each repo's own `issues.md`. This file holds only
the work that spans repositories, plus the ordering constraints between them.

Everything here was measured on this machine unless marked *unverified*.

---

## X0. Release ordering - blocks several other items

The provenance work created a strict order. Getting it wrong breaks CI, as it
already did once on PR #20.

1. **ROMEO 1.5.0** - new exported API (`write_provenance`, `register_citation!`).
   Requires nothing new from anyone; compat on MriResearchTools stays `"2,3"`.
2. **MriResearchTools 3.6.0** - requires ROMEO `1.5` for `register_citation!`.
3. **CLEARSWI 1.6.2** - requires MriResearchTools `3.6`.
4. **CompileMRI / mritools binaries** - `App/Project.toml` now pins
   `=1.5.0` / `=3.6.0` / `=1.6.2`. Re-release once 1-3 are registered.

Between steps 1 and 2, a `romeo` run against MriResearchTools 3.5 warns that the
ASPIRE citation is unavailable rather than silently omitting it. That window is
expected; it closes at step 2.

**The binary release matters more than usual this time.** Every mritools release
since ROMEO 1.2.0 (2024-10-03) has shipped the `unwrap_individual!` data race:
`romeo -i` and `clearswi --qsm` on a multi-core machine could differ by a full
2pi between identical runs. Step 4 is the fix reaching users.

---

## X1. Shared CLI layer

Not one CLI - five commands, shared plumbing. `romeo`, `clearswi`, `mcpc3ds`,
`makehomogeneous` and `romeo_mask` keep their names and flags.

Measured duplication across the five implementations:

| Helper | Copies | State |
|---|---|---|
| `exception_handler` | 5 | byte-identical (same md5) |
| `getTEs` | 4 | four different implementations that disagree |
| `getechoes` | 3 | |
| `parseweights` | 2 | |
| `load_data_and_resolve_args!`, `get_keyargs`, `select_echoes!`, `write_qualitymap` | 2 | same names in romeo and romeo_mask, all drifted |

The `getTEs` disagreements are real, not cosmetic: romeo returns the scalar `1`
for single-echo where CLEARSWI returns `[1]`; CLEARSWI tests `isa Matrix` where
romeo tests `isa AbstractMatrix`; romeo guards `1 < length(TEs) == neco` where
CLEARSWI drops the `1 <`; mcpc3ds has no `epi` support.

Scope: option struct -> ArgParse adapter, echo-time and echo-selection parsing,
output-path splitting (`.nii` vs directory), memory-mapping decisions, mask
resolution, the error handler. `write_provenance` is already the first piece and
worked - it removed five copies of the citation text and exposed two
wrong-citation bugs.

This is also where `RomeoApp` should live, which removes the ROMEO <-> MriResearchTools
cycle as a side effect rather than as a package created for that one purpose.

**Do the safe echo-time parser first** - see X2.

---

## X2. Safe echo-time parser (also a security fix)

All four parsers run the `-t` argument through `eval(Meta.parse(...))`.
Demonstrated on the shipped CLI:

    romeo -t '(write("PROOF.txt","executed"); [4,8,12])'    -> file written

Not an escalation for someone at their own shell. It matters where `-t` is
machine-supplied: a pipeline reading echo times from a BIDS sidecar or job
config, a Neurodesk or QSMxT container, a MATLAB `system()` call, or any product
path where the value comes from data rather than a person.

`mcpc3ds`'s `parse_array` is shorter but evals too - there is no safe
implementation among the four to promote.

Needs to accept `[4,8,12]`, `3:3:15`, `4 8 12` and `epi [TE]`. Roughly 20 lines.
Write it once, in X1.

Sites: `ROMEO.jl/ext/RomeoApp/argparse.jl:175,197,212,222`,
`CLEARSWI.jl/ext/ClearswiApp/{argparse.jl:111,133, caller.jl:100}`,
`CompileMRI.jl/App/src/Mcpc3ds.jl:133`.

---

## X3. Conformance suite - one numerical truth

The single highest-leverage item. The CLEAR-SWI back half is implemented four
times - Julia, the ICE C++ functor, Rust, TypeScript - with no shared definition
of correct, and the measured spread is not small.

Build: a frozen reference dataset plus the golden intermediate volume after every
named pipeline stage, published once as a versioned artefact (a Julia Artifact or
a small data repo) and consumed by the Julia tests, the Rust `compare` harness,
the viewer's parity suite and the ICE functor's validation.

Also collapses the four vendored copies of `test/data/small` (byte-identical in
MriResearchTools, ROMEO, CompileMRI and mritools-binaries; CLEARSWI's copy has
*different* checksums under the same filenames - reconcile that while doing it).

**Blocked on X4** for the CLEAR-SWI half: the Rust and Julia `steps/` dumps
disagree on the shape of `phase_unwrapped`, so the largest divergence cannot
currently be drilled into at all.

---

## X4. writesteps as a specified contract

`combinedphase` and `filteredphase` mean different things depending on the
unwrapping path. In `laplacian_combine`, `filteredphase` is the 4D per-echo
high-passed phase and `combinedphase` is the field combined *from* it; in
`romeo_combine`, `combinedphase` is the unfiltered combination and
`filteredphase` is the 3D filtered result. `laplacianslice_combine` saves the
already-filtered array under the name `unwrappedphase` and never writes
`filteredphase` at all.

Write down what each name means, independent of path, and make CLEARSWI obey it.
This retires most of the viewer's `JULIA_STEM_ROLE` compensation, gives the ICE
export a schema to target, and unblocks the Rust parity work.

---

## X5. Monorepo

Julia supports this properly now, and it is the right shape for packages that are
released together and change together.

- `[workspace]` in a root Project.toml gives the sub-projects one shared manifest.
- `[sources]` (Julia 1.11+) points them at each other in-tree.
- The General registry registers subdirectory packages via a `subdir` field, and
  documents the migration for already-registered packages: *"Move the files with
  normal git commands, without changing the revision history. Make a manual pull
  request to this repository in which you add a `subdir` field."*

Users are unaffected: a subdirectory package is still an independent registered
package. `Pkg.add("ROMEO")` is identical.

    mritools/
      Project.toml    [workspace] projects = [...]
      ROMEO/  MriResearchTools/  CLEARSWI/  QuantitativeSusceptibilityMappingTGV/
      CompileMRI/
      MriToolsCLI/          <- X1
      test/conformance/     <- X3

**In:** the five Julia packages, the new CLI layer, the conformance data.
**Out:** `mritools-binaries` and `CLEARSWI-Viewer` (different toolchains and
release cadences), the `ROMEO` docs repo (the citable front door and the support
desk), `ASPIRE` (frozen), and **`KorecSWI` - which must stay separate or the SOUP
boundary argument stops meaning anything.**

Gotchas the registry calls out: each subdirectory package needs its own LICENSE
copy, and the move must preserve git history (`git mv`, not copy-delete).
`[workspace]` is recommended for Julia 1.13+, but that affects only the developer
environment - the packages keep their `julia = "1.9"` floor.

**Do this after X0 lands.** Migrating with four branches in flight and three
pending releases would be miserable.

---

## X6. Static compilation - keep in mind, do not chase yet

Measured on this machine with Julia 1.12.7's `juliac`, compiling the real ROMEO
kernel:

| | Artefact | Startup |
|---|---|---|
| Today's `mritools` bundle (Linux) | 160 MB | - |
| Through the Julia runtime, JIT | - | 3899 ms |
| `juliac --trim=safe` | **2.29 MB** | **39 ms** |

Bit-identical checksum. For scale the Rust `romeo` is 1.5 MB.

Type stability is necessary but not the requirement: `--trim` needs every call
site *provably reachable and statically resolvable*.

What blocked it, and what it teaches:

- The first attempt gave 14 verifier errors and **all fourteen traced to one
  file**, `region_handling.jl` - the two experimental off-by-default features.
  They are off at *runtime*, so static reachability must compile the branch,
  which drags in `Statistics.median`, whose error path calls `repr`, which pulls
  in the entire array-display machinery. Making them a compile-time choice took
  it to zero errors.
- `Threads.@threads` is not yet handled; the binary crashed in `threading_run`
  until the two threaded loops were serialised. A real cost, not a cleanup.
- Adding NIfTI reading and `robustmask` takes it from 0 to **68 errors**, all
  ordinary dynamic dispatch. 32 of them are `niread`/`readmag`: a NIfTI's element
  type comes from the file header at run time. Standard fix is a function barrier.
  About half that work is inside NIfTI.jl, which is not ours.

**The point for now:** the refactors static compilation needs are the ones
already on this list. X2 (no `eval`) is an absolute blocker for `--trim` and a
security bug. Typed options instead of `Dict{String,Any}` are what make dispatch
resolvable. The five `invokelatest` calls in `App.jl` are three lines each. None
of it is work done *only* for compilation - so do those with trimming in mind,
and revisit when `juliac` is no longer `--experimental`.

It also sharpens the Rust question: if Julia reaches 2.3 MB and 39 ms, the case
for a second implementation currently at r = 0.63 on SWI weakens considerably.

---

## X7. Decisions needed - not engineering

1. **Public/private split for the new CLEAR-SWI work.** Gates pushing the four
   local branches -> `Hooks` -> CLEARSWI 1.7 -> KorecSWI running at all, and the
   viewer's licence. Every week it waits, the merge cost rises and the
   differentiating work stays in exactly one place.
2. **The ASPIRE patent (US10605885B2).** Whether the product path touches
   MCPC-3D-S, and if so whether rights exist through the original assignment or a
   licence is needed. Longest lead time of anything here; gates launch, not
   development.
3. **Governance.** An organisation and a second maintainer with commit rights.
4. **What the Rust port is for** - retire, keep explicitly experimental, or
   promote to the product runtime. Only answerable after X3.
