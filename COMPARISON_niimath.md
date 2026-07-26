# ROMEO.jl vs. the `-romeo` C port in niimath

A comparison of the reference implementation ([ROMEO.jl](https://github.com/korbinian90/ROMEO.jl))
against the C port shipped in [niimath](https://github.com/rordenlab/niimath) `v1.0.20260725`
(`src/romeo.c`, 2463 lines).

**Bottom line.** This is not a re-implementation of the ROMEO *idea* — it is a line-by-line
transliteration of ROMEO.jl v1.4.0, pinned to commit `60d83fbb6966...`, that goes to considerable
lengths to reproduce Julia's floating-point semantics exactly. The unwrapping core is the same
algorithm producing the same bits. What is missing is the experimental periphery, and every
missing option fails loudly rather than silently doing something else.

Reference versions declared in `src/romeo.c`:

| upstream | version | what was ported |
| --- | --- | --- |
| ROMEO.jl | v1.4.0, `60d83fbb69669560d227c252dcd844afbb6648e1` | `utility.jl`, `priorityqueue.jl`, `weights.jl`, `seed.jl`, `algorithm.jl`, `unwrapping.jl`, `voxelquality.jl`, `ext/RomeoApp/` |
| MriResearchTools.jl | v3.5.0 | `robustmask`, `gaussiansmooth3d` box filter, `sample`/`quantile`, `readphase` rescaling, `calculateB0_unwrapped`, `get_B0_snr` |
| Julia stdlib | `base/math.jl`, `base/special/rem_pio2.jl` | `rem2pi(x, RoundNearest)` |

Licensing and attribution are handled correctly: MIT notices reproduced verbatim in
`src/romeo.LICENSE`, the Dymerska et al. 2020 citation is in the help text, the README, and the
license file, and the README states that the niimath binary contains an MIT component.

---

## 1. What is identical

### 1.1 The algorithm

Every function of the unwrapping core has a 1:1 counterpart:

| ROMEO.jl | romeo.c |
| --- | --- |
| `grow_region_unwrap!` | `rm_grow_region` |
| `getnewedge`, `getedgeindex`, `getdimfromedge`, `getfirstvoxfromedge` | `rm_getnewedge`, `rm_getedgeindex`, … |
| `unwrapedge!`, `unwrapvoxel` | `rm_unwrapedge`, `rm_unwrapvoxel_ff` / `_fd` |
| `PQueue` (bucket queue) | `rm_pq` |
| `getseedqueue` + `findseed!` | `rm_find_seed` (see §3.1) |
| `seedcorrection!` | `rm_seedcorrection` |
| `getweight` and the six weight terms | `rm_getweight` |
| `calculateweights` | `rm_calculateweights` |
| `voxelquality` | `rm_voxelquality` |
| `unwrap!` (3D / 4D), temporal propagation | `rm_unwrap3d`, `rm_core_run` |
| `temporal_uncertain_unwrapping!` | inline in `rm_core_run` |
| `unwrap_individual!` | inline in `rm_core_run` |
| `correctglobal`, `correct_multi_echo_wraps!` | `rm_correctglobal`, inline |
| `robustmask`, `gaussiansmooth3d`, `fill_holes` | `rm_robustmask`, `rm_boxsmooth3d`, `rm_fill_holes` |

The details that decide the spanning tree are preserved, not approximated:

- **Bucket-queue LIFO semantics.** `enqueue!` pushes, `dequeue!` pops from the *end* of the lowest
  non-empty bin. A heap or a FIFO would build a different (still valid) tree and assign different
  2π multiples in ambiguous regions. Ported literally, with a comment saying why.
- **Julia's linear-index `checkbounds`.** A neighbour lookup may cross a row or plane boundary and
  still count as in-bounds; ROMEO relies on this and then zeroes `weights[1,end,:,:]` etc. so the
  invalid directed edges never enter the queue. The C keeps 1-based voxel indices internally and
  reproduces the same formulas, with an explicit "do NOT fix these into Cartesian bounds checks".
- **Seed tie-breaking.** Smallest substituted weight sum, ties resolved toward the *highest* linear
  index — which is what popping from the end of the bin does.
- **`new_seed_thresh = NBINS - div(NBINS - sum(w)/3, 2)`** with `div` truncating toward zero.

### 1.2 Julia's numerics, reproduced deliberately

This is the part that distinguishes the port from a rewrite. `romeo.c` carries a numeric-type audit
table stating, per weight term, whether Julia's promotion rules make it `Float32` or `Float64`, and
implements each accordingly. Specifically:

- **`rem2pi(x, RoundNearest)` is ported in full** — Cody-Waite reduction, the extended branch, and
  the Payne-Hanek path with 128-bit arithmetic written portably (no `__int128`, so MSVC and wasm
  reduce identically). The reason is correct: Julia reduces against an infinitely precise 2π, so
  libc `remainder(x, 6.283185307179586)` is *not* equivalent — it differs by ~n·2.4e-16.
- **Round-half-to-even.** Julia's `round(Int, x)` is half-to-even; C's `round()`/`lround()` are
  half-away-from-zero. The port uses `nearbyint()`. This governs `rescale()` (the weight bin) and
  `unwrapvoxel()` (the number of wraps) — both load-bearing.
- **NaN propagation.** Julia's `max`/`min` propagate NaN where C's `fmax`/`fmin` return the non-NaN
  operand; `Base.minmax(5f0, NaN32)` returns `(NaN32, NaN32)`, i.e. NaN in *both* endpoints. Hand-
  written equivalents are used throughout, and the `minmax` case is called out as changing
  `magweight`/`magweight2` and potentially retaining an edge Julia drops.
- **`Statistics.quantile` type 7**, including the observation that the pinned registry package
  Statistics v1.11.1 computes `aleph = n*p + m` with a plain multiply-add while the newer bundled
  stdlib copy uses `fma(n, p, m)` — which moves `maxmag` in the 11th digit.
- **`Statistics.median` via `Base.middle`** (`x/2 + y/2`, not `(x+y)/2`).
- **`MriResearchTools.sample`** block geometry, computed with exact integer arithmetic and
  half-to-even rounding rather than inheriting Julia's `TwicePrecision` range spelling.
- **The two-width `unwrapedge!`.** `wrapped[oldvox] + d` is a Float32 addition when `d` is the
  pass-through difference and a Float64 addition when `d` is the `wrap_addition` threshold, and the
  two feed different `unwrapvoxel` methods. Both variants exist in the C and are not collapsed.

### 1.3 The build treats this as load-bearing

The rest of niimath is compiled whole-program `-ffast-math`. `romeo.o` is built
`-fno-fast-math -ffp-contract=off`, because a single reassociated expression can move a weight from
bin 137 to bin 138, change the spanning tree, and shift an entire connected region by exactly 2π.
The file documents this as measured, not assumed: under `-ffast-math`, reassociation pushes a weight
just past 1.0, the `0 <= w <= 1` guard in `rescale()` returns bin 0, and the edge *disappears* from
the graph — 66 voxels off by a full wrap, max |diff| 12.57 rad.

That is the correct diagnosis of the one real numerical hazard in ROMEO, and I have not seen another
port take it seriously.

---

## 2. What is missing

All of these are rejected with an explicit error message rather than silently ignored:

| ROMEO option | status in niimath |
| --- | --- |
| `-w bestpath` (Abdul-Rahman weights) | not implemented |
| `--max-seeds > 1` | not implemented (capped at 1) |
| `--merge-regions`, `--correct-regions` | not implemented — `region_handling.jl` is not ported at all |
| `--wrap-addition != 0` | not implemented (the code path exists but is rejected) |
| `--mask-unwrapped` (`-u`) | not implemented |
| `--unwrap-echoes` (`-e`) | not implemented |
| `--threshold` | not implemented |
| `--fix-ge-phase` | not implemented (`in_hdr` is already plumbed through for it) |
| `--phase-offset-correction`, `--phase-offset-smoothing-sigma-mm`, `--write-phase-offsets` | not implemented |
| 5D coil combination (MCPC-3D-S over channels) | out of scope — 5D input is rejected |
| `weights` as a pre-computed array | n/a (CLI only) |

The consequence worth flagging: **multi-echo `-B` is not equivalent to ROMEO's `-B`**, because
ROMEO's own `--compute-B0` silently enables MCPC-3D-S monopolar phase-offset correction. niimath
prints this on stderr and the README says the maps correspond to
`romeo --compute-B0 --phase-offset-correction off`. Correctly handled, but users comparing B0 maps
need to know.

MCPC-3D-S *does* exist in niimath, but inside `--medic` (`src/medic.c`), and only the reduced case
the MEDIC paper needs: two echoes, monopolar, single-channel, **no spatial smoothing** of the
offset. Bipolar acquisition and multi-channel combination are explicitly out of scope.

---

## 3. Deliberate divergences

Four, each documented in place:

1. **`robustmask`'s two `mean(::Vector{Float32})`** are accumulated in double and rounded once to
   float. Julia reduces through `sum`, i.e. pairwise blocks of 1024 whose base case is `@simd`, so
   the reduction order is a property of the oracle machine's codegen rather than a portable
   contract. The C version is *more* accurate; observed deviation ≤1 float ULP (~9e-8 relative).
2. **`correctglobal` on an all-non-finite mask** skips the correction instead of reaching Julia's
   `median([])` and throwing.
3. **`correct_multi_echo_wraps!`** requires *both* echoes to be finite at a voxel. Upstream filters
   the reference and current echoes independently before subtracting, which either throws on a
   length mismatch or silently pairs mismatched voxels when a NaN is present in only one echo.
4. **B0 SNR without a magnitude.** The substituted `exp(-TE/20)` decay is voxel-independent, so
   `sum(mag.*w)/sum(w)` has no spatial extent and ROMEO writes a 1×1×1 image; niimath writes the
   same constant across the working grid.

Divergences 2–4 are cases where upstream throws or misbehaves on degenerate input. On any image with
at least one finite in-mask voxel they are identical.

### 3.1 Engineering differences that do not change results

- **Seed search.** Upstream builds a `3*NBINS` bucket queue over `sum(weights; dims=1)` and dequeues
  it once. Since `maxseeds` is capped at 1, the port replaces it with a single O(n) scan — saving an
  `int64_t[n3]` plus bucket metadata (128 MiB at 256³). The code carries a warning that the bucket
  queue must come back if `--max-seeds > 1` is ever ported.
- **Lazy mask stages.** `robustmask`'s six intermediate buffers are allocated as reached rather than
  up front, cutting the peak from 12 bytes/voxel to ~5 when `-romeo-dump` is off.
- **Threading.** ROMEO.jl parallelises the weight kernel as `Threads.@threads for dim in 1:3` — at
  most three threads. niimath uses OpenMP over voxels *within* each dimension, which scales past 3
  cores. Results are unaffected: every output element is independent.
- **Fail-closed queue.** The bucket queue carries a sticky OOM flag, because a failed enqueue would
  silently drop an edge from the spanning tree and leave a region wrong with a zero exit status.
- **No comparator-based `qsort`** (a wasm constraint) — quickselect with a deterministic pivot.

---

## 4. Three findings in ROMEO.jl itself

Working through the port surfaced three things on our side that are worth a look independently of
niimath.

### 4.1 `unwrap_individual!` has a data race

```julia
function unwrap_individual!(wrapped::AbstractArray{T,4}; TEs, keyargs...) where T
    args = Dict{Symbol,Any}(keyargs)
    Threads.@threads for i in 1:length(TEs)
        e2 = if (i == 1) 2 else i-1 end
        if haskey(keyargs, :mag) args[:mag] = keyargs[:mag][:,:,:,i] end   # shared Dict, written from every thread
        unwrap!(view(wrapped,:,:,:,i); phase2=wrapped[:,:,:,e2], TEs=TEs[[i,e2]], args...)
    end
```

`args` is a single `Dict` shared across all threads and mutated inside the loop, then splatted. With
a magnitude and more than one thread, an echo can be unwrapped using **another echo's magnitude**.
Separately, `phase2=wrapped[:,:,:,e2]` copies a slice that another thread may be writing at that
moment, so whether the reference echo is already unwrapped is itself racy.

So `--individual-unwrapping` is not reproducible run-to-run under multithreading. The niimath port
is sequential and therefore deterministic; its comment notes that the oracle pins
`JULIA_NUM_THREADS=1`, which is presumably why this was never noticed.

**Scope, measured.** Four runs of each path on the 3-echo volume, `JULIA_NUM_THREADS=4`, counting
in-mask voxels that differ from run 1 (85 626 in-mask voxels):

| path | differing voxels across runs | max \|diff\| |
| --- | --- | --- |
| temporal (the default) | 0, 0, 0, 0 | 0 |
| `individual` **without** a magnitude | 0, 0, 0, 0 | 0 |
| `individual` with a magnitude | 0, 1088, 4018, 4862 | 37.7 rad (6 wraps) |
| `individual` + `correctglobal` | 0, 0, 11795, 12413 | 44.0 rad (7 wraps) |

That isolates the mechanism. The default path is unaffected — its only `Threads.@threads` is in
`calculateweights` over `dim in 1:3`, which writes disjoint slices of a preallocated array and is
benign. Dropping the magnitude also makes `individual` deterministic, which points at the
`args[:mag]` write specifically: it is the only statement that mutates the shared `Dict`, and it
only executes when a magnitude is present. The `phase2` slice read is a real race too, but it did
not manifest here — all three tasks take their copy before any of them has written much.

`--correct-global` does not rescue it. `correct_multi_echo_wraps!` estimates one 2π offset per echo
from the median, and it is computing that median from already-racily-unwrapped echoes.

Measured on the 76×76×46×2 validation volume with `JULIA_NUM_THREADS=4`, six identical calls to
`unwrap(phase; TEs, mag, mask, individual=true)`:

```
  run 1 vs run 1: differing voxels = 0        max|diff| = 0.0
  run 2 vs run 1: differing voxels = 0        max|diff| = 0.0
  run 3 vs run 1: differing voxels = 89481    max|diff| = 56.5487   (all >= 1 full wrap)
  run 4 vs run 1: differing voxels = 1399     max|diff| = 25.1327   (all >= 1 full wrap)
  run 5 vs run 1: differing voxels = 1399     max|diff| = 25.1327
  run 6 vs run 1: differing voxels = 1399     max|diff| = 25.1327
  all six runs identical: false
```

With `JULIA_NUM_THREADS=1` all six runs are identical. So this is the threading, not the data: up to
89 481 voxels — 17 % of the volume — land on a different 2π branch between two runs of the same
call, with excursions up to 56.5 rad (nine wraps).

**Does it cost quality, or is it only non-determinism?** Both, and the distinction matters:

- The differences are genuine wrap errors, not a cosmetic offset. Decomposing each run's difference
  from run 1 into a per-echo global 2π offset plus a remainder, the global offsets are `[0,0,0]` and
  the remainder is the entire difference — 12 801 / 12 565 / 11 795 in-mask voxels on the 3-echo
  volume. Nothing a global correction can absorb.
- But no run is systematically better. Scoring each run by the in-mask RMS residual of the
  magnitude-weighted `φ = a + ω·TE` fit gives median 2.5454–2.5468 rad and p95 2.9400–2.9446 across
  runs — indistinguishable. The tail moves a little (p99 4.22–4.46, max 12.0–13.6) with no
  consistent winner.

So the failure mode is not "threading corrupts an otherwise-correct answer". It is "the answer is
drawn arbitrarily from a set of roughly equally-flawed answers, and you get a different draw each
run". For a published pipeline that is still a real defect: the same command on the same data does
not reproduce, and ~14 % of in-mask voxels sit on a different branch between draws.

(The residual is high in absolute terms for every `individual` run because independent per-echo
unwrapping leaves an arbitrary 2π constant per echo, which the two-parameter fit cannot absorb. For
scale, the default temporal unwrapping on the same data gives a median residual of 0.0354 rad. That
is a property of `individual`, not of the race.)

**Fixed** in [ROMEO.jl#17](https://github.com/korbinian90/ROMEO.jl/pull/17): keyword arguments are
built per iteration, and the reference echoes are snapshotted before the threaded loop. All four
paths in the table above are deterministic afterwards over six runs at four threads, and the suite
passes 192/192.

The snapshot also makes `phase2` the wrapped phase for every echo — which is what `unwrap!` (4D) and
`voxelquality` already pass, and what `seedcorrection!`'s `off2 ∈ -1:1` search assumes. That is a
behaviour change: against the old single-threaded output, 830 of 256 878 in-mask voxel-echoes differ
(0.32 %), with the fit residual unchanged (median 2.5463 → 2.5462 rad, voxels above 1 rad 77 323 →
77 302). niimath's `-i` reproduces the old behaviour deliberately, so its individual-unwrapping
parity will need the same update.

### 4.2 `wrap_addition` changes arithmetic width between the library and the CLI

In `unwrapedge!`:

```julia
d = 0                              # Int
...
d = if v < -x  -x  elseif v > x  x  else  v  end
wrapped[newvox] = unwrapvoxel(wrapped[newvox], wrapped[oldvox] + d)
```

- `grow_region_unwrap!`'s signature default is `wrap_addition=0`, an **`Int`**. With `x::Int`, the
  threshold branches yield `Int` and `wrapped[oldvox] + d` is a **Float32** addition, so
  `unwrapvoxel` subtracts in Float32.
- The CLI app declares `--wrap-addition` as `arg_type = Float64, default = 0.0` and passes it
  through, so `x::Float64`, the branches yield `Float64`, and the subtraction happens in
  **Float64**.

The two paths round `new - old` differently, so `round((new-old)/2π)` could in principle land on a
different integer when the difference sits within ~1 Float32 ULP of an odd multiple of π.

**Measured: it does not bite here.** Comparing `unwrap(...)` against
`unwrap(...; wrap_addition=0.0)` on the validation volume gives **0 differing voxels out of
531 392**. So this is a latent inconsistency in the source, not an observed defect — the window is
narrow enough that real phase data does not hit it.

**Fixed** in `60d83fb..adb695a`: the signature default is now `0.0`. Verified test-neutral — the
data-backed testsets behave identically with and without the change, and the full suite
(`features`, `specialcases`, `dsp_tests`, `mri`, `voxelquality`) passes 192/192.

It does mean niimath is bit-parity with the **CLI** path specifically — its `wrap_addition` is a C
`double` fixed at `0.0` — which is the right target for a CLI port.

### 4.3 The "phase first without `-p`" convenience breaks on any hyphenated filename

`ext/RomeoApp/argparse.jl`:

```julia
if !('-' in args[1]) prepend!(args, Ref("-p")) end   # if phase is first without -p
```

`'-' in args[1]` tests whether the string *contains* a hyphen anywhere, not whether it *starts* with
one. So the convenience only fires for a path with no hyphen at all — and BIDS filenames are built
out of hyphens (`sub-01_echo-1_part-phase_bold.nii.gz`). The bare positional then reaches ArgParse,
which rejects it with "too many arguments" and a usage dump.

Demonstrated in one directory, same data, only the filename differing:

```
$ romeo.jl plain.nii.gz -m mag.nii.gz -t 16.8 -o outA                 # works, outA written
$ romeo.jl sub-01_part-phase.nii.gz -m mag.nii.gz -t 16.8 -o outB     # usage error, no output
```

The same `'-' in ...` test is used for the "phase is last without `-p`" branch on the line below.
I hit this by accident writing the benchmark above — the *directory* contained a hyphen, which is
enough. Worth fixing to `startswith(args[1], '-')`, especially now that MEDIC is bringing BIDS-named
data to ROMEO.

---

## 5. Verification

I re-ran the port's own parity suite against a freshly built oracle, on **Linux x86-64 with gcc** —
the platform `romeo.c` explicitly flags as never having been checked ("every measurement above was
made on arm64 macOS. Re-run `test/romeo_compare.py` on a Linux gcc release build before claiming
bit-identity there").

Setup:

- ROMEO.jl at the pinned commit `60d83fbb6966...`, MriResearchTools v3.5.0, Statistics v1.11.1,
  Julia 1.12.3 — all matching the pin. Only the resolved `Manifest.toml` hash differs (fresh
  resolution), so the oracle ran with `--skip-env-check`.
- niimath built from `v1.0.20260725` with plain `make` (gcc 14, OpenMP, `-flto`); `romeo.o` picks up
  the strict-FP flags from the Makefile as shipped.
- Validation data reconstructed from the `medic_bench` `echo2` BIDS dataset — the sbref volumes are
  76×76×46 with TEs 16.8 / 38.56 ms, i.e. exactly the volume `romeo.c` names.

### 5.1 Parity suite

```
602/602 checks passed
```

Covering 15 cases (4 real + 11 synthetic) × 10 weight selections, plus the primitives. On the real
single-echo volume `e0`:

| check | result |
| --- | --- |
| phase rescale | byte-identical (1 062 784 bytes) |
| weights (u8) | byte-identical (797 088 bytes) |
| mask stages s1–s4 | byte-identical (265 696 bytes each) |
| region labels (`visited`) | byte-identical |
| seed scalars | `seed=158584 new_seed_thresh=131` exact |
| quality maps (qmap, qmap_1..6) | max\|diff\| = 0.000e+00 |
| weights pre-rescale (f64) | max 0 ULP (limit 4) |
| **unwrapped phase** | **max\|diff\| = 0.000e+00**, wraps=0 |
| B0, all 6 weighting modes + SNR | byte-identical |
| `-g` correct-global | max\|diff\| = 0.000e+00 |

Multi-echo `me` adds `-i` individual, `--temporal-uncertain-unwrapping 0.5` and `--template 2`, all
at max\|diff\| = 0.000e+00. The unwrapped phase is not merely inside the 1e-4 tolerance — it is
bit-identical.

### 5.2 CLI against CLI

The suite compares raw dumps. As an independent end-to-end check I ran both command-line tools on
the 2-echo volume with default settings and compared the NIfTI outputs:

```
romeo.jl phase.nii -m mag.nii -t [16.8,38.56] -o jl_out
niimath  phase.nii -romeo mag.nii -t [16.8,38.56] t.nii
```

| output | max \|diff\| | differing voxels |
| --- | --- | --- |
| unwrapped phase (531 392 voxels) | 0.000000e+00 | 0 |
| `mask` (265 696 voxels) | 0.000000e+00 | 0 |

### 5.3 Three echoes

The parity suite's real cases are 1-echo (`e0`, `e1`, `e0n`) and 2-echo (`me`) — there is no 3-echo
case. Built one from the `medic_bench` `echo3` sbref volumes (76×76×46, TEs 14.8 / 34.38 / 53.94 ms)
and compared the two CLIs directly across ten option combinations:

| case | result |
| --- | --- |
| default, `-template 2`, `-temporal-uncertain-unwrapping 0.5`, `-i`, `-g`, `-B`, `-k qualitymask`, `-k nomask`, `-w romeo2`, `-w romeo6` | max\|diff\| = 0.000e+00, 0 of 797 088 voxels differ, in every case |

The B0 side outputs looked like a discrepancy at first (max\|diff\| 7.6e-6 Hz on the field map,
1.8e-3 on the SNR) but are not one: **ROMEO writes B0 and B0_snr as Float64, niimath as Float32.**
Rounding ROMEO's Float64 output to Float32 reproduces niimath's bytes exactly — 0 of 265 696 voxels
differ, on both maps. The relative deviation before rounding is 2e-8 to 6e-8, i.e. below one Float32
ULP. The parity suite never sees this because it compares Float32 raw dumps on both sides; only the
saved NIfTI differs, in datatype.

### 5.4 Cost, on this volume, with JIT excluded

Timing Julia against a compiled binary is only meaningful if compilation is kept out. Both sides
here run the **same full pipeline** — read, `readphase` rescale, `robustmask`, weights, unwrap,
write uncompressed NIfTI. On the Julia side `unwrapping_main` (the entry point `romeo.jl` calls) is
invoked once to force compilation and then timed over five further calls in the same session; on the
C side each run is a fresh process, so process startup is charged to niimath, not hidden. Four
cores, `JULIA_NUM_THREADS=4` / `OMP_NUM_THREADS=4`, `-gz 0` on both sides.

| case | ROMEO.jl warm, median (min) | niimath, median (min) | ratio (median) |
| --- | --- | --- | --- |
| single-echo | 121.3 ms (112.5) | 104.7 ms (101.1) | 1.16× |
| multi-echo, 2 echoes | 146.5 ms (120.9) | 99.0 ms (97.2) | 1.48× |
| multi-echo + `-B`, offset correction **off** | 130.1 ms (127.7) | 103.0 ms (100.4) | 1.26× |
| multi-echo + `-B`, MCPC-3D-S on (ROMEO default) | 547.3 ms (505.1) | — not implemented | — |

**Once JIT is excluded the two are within 1.2–1.5× of each other.** The C is faster, but modestly
so — this is a well-optimised Julia implementation, and the gap is what one expects from removing a
managed runtime, not an algorithmic difference.

The fourth row is the one to read carefully. ROMEO's `-B` silently switches phase-offset correction
to monopolar MCPC-3D-S (`caller.jl:82-84`), which niimath does not implement at all. Comparing that
row against niimath's `-B` would be comparing 4.2× more work against less work and calling it
slowness. The third row is the like-for-like `-B` comparison.

For reference, the cold `romeo.jl` CLI from source is **36.9 s** end to end on this box. That is
Julia startup and JIT, not the algorithm — the compiled `mritools` binary avoids it — but it is what
a user invoking the script once actually waits for, and it is the single largest practical
difference between the two tools.

### 5.4 Memory

| | peak RSS |
| --- | --- |
| `niimath -romeo` (largest child, `RUSAGE_CHILDREN`) | 0.015 GB |
| Julia process running `unwrapping_main` (`Sys.maxrss`) | 0.95 GB |

The ~60× gap is almost entirely the Julia runtime and the loaded package stack, not ROMEO's working
set — the data here is only a few MB. It is still a real cost to a user, and it is the one point
from the email's list that applies to us rather than only to warpkit. A `PackageCompiler` sysimage
reduces the startup cost but not this floor.

---

## 6. Assessment

The port is careful, honest about its limits, and correctly attributed. Concretely:

- The unwrapping core is ours, bit for bit, and the parity suite is a real one — byte-exact on
  weights, mask stages and region labels, ULP-bounded on the pre-rescale weights, wrap-count-checked
  on the unwrapped phase.
- What is absent is the experimental periphery (`bestpath`, multi-seed, region merging,
  `wrap_addition`, GE phase fix) and phase-offset correction. Each errors out; none is silently
  approximated.
- The one behavioural difference a user is likely to hit is multi-echo `-B` without MCPC-3D-S.

On cost: with JIT excluded, ROMEO.jl is within 1.2–1.5× of the C on the same pipeline. The honest
gaps are cold start (36.9 s from source) and the ~0.95 GB Julia runtime floor — both real for users,
neither algorithmic.

The framing for the wider claim in the email: the 32–40× figure is an end-to-end MEDIC number
dominated by the *apply* stage, where `wk-apply-warp` has no thread option. `medic_bench`'s own
README is clearer than the email — the estimate stage, the like-for-like comparison, is 4.2–4.6× at
roughly half the RAM, and the README states plainly that end to end the two do not yet match. None
of that is a criticism of the ROMEO port, which is a separate and much stronger claim: there,
"equivalent" means bit-identical, and it is checked.
