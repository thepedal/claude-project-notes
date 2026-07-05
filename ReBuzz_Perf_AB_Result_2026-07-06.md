# ReBuzz perf sweep — PRE/POST A/B result (2026-07-06)

Standalone result record for the allocation/UpdateBuffer optimisation sweep. Companion to
`ReBuzz_Perf_Sweep_Findings_2026-07-05.md` (the findings) — this doc records the *measured
outcome*. It is a fixed finding, not a living status doc; do not overwrite it on the next
handover rewrite (reference it from the handover instead).

## Bottom line

The sweep produced a **clean, definitive ~8% reduction in Gen-0 allocations** and a
**corroborating ~8% improvement in best-case per-call render time**, measured PRE vs POST
across 5 runs each on `det_grid_08x04`. Both signals agree in direction and magnitude and
match what the changes were designed to do. The effect is modest in absolute terms and, as
predicted, does **not** touch the coordination/barrier wall (handoff §6.2) — it lands in the
engine allocation + per-sample term, exactly where intended.

## What was compared

- **PRE** = `c841786` (merge of #111; pre-sweep baseline).
- **POST** = `e9837a7` (merge of #116; includes #113 + #114 + #116) **plus** the connbuffer
  fast-path patch (B1) applied on top, since B1 was not yet merged upstream at the time.
- So POST = #113 (O(1) work-queue pop) + #114 (I/O facade + SongTime alloc removal) +
  #116 (WorkManager facade/LINQ removal) + connbuffer B1 (zero-latency fast path +
  branchless ring wrap).
- Both built **Release/x64** in Visual Studio (Clean Solution + Build Solution), same
  machine, back to back. `ReBuzzPatternXP` unloaded (dead C++ `std::tr1` project, irrelevant).

## Instrument

`RenderBenchmark` probe (perf/instrument branch, `perf_instrument_render_benchmark.patch`),
triggered by Ctrl+Shift+B in-app. It drives the driver-free override render path
(`AudioEngine.GetAudioProvider().ReadOverride` → `WorkManager.MainAudioFillBuffer`), so no
ASIO cadence and no fill-thread ring enter the timed window — this is what made the signal
readable where live PP2 dumps could not (the live dumps' per-machine EMA spans ~4x on
identical machines; ratio>3 also cadence-inflated the dropout family). Each run = 8192 render
calls × 512, 256 warm-up, GC counted across the pass.

## Data (5 runs each, det_grid_08x04, Release)

Trustworthy metrics only. `min` = cleanest timing sample (the pass that dodged OS/GC
interference); `G0`/`G1` = Gen-0/1 collections across the pass (low variance, build-dependent).

| metric        | PRE runs                  | PRE median | POST runs                 | POST median | Δ (median) |
|---------------|---------------------------|-----------:|---------------------------|------------:|-----------:|
| min µs        | 422, 632, 502, 478, 413   | **478**    | 439, 406, 448, 495, 433   | **439**     | **−8.2%**  |
| G0            | 750, 769, 752, 751, 769   | **752**    | 693, 711, 691, 700, 693   | **693**     | **−7.8%**  |
| G1            | 100, 144, 99, 105, 133    | 105        | 101, 126, 107, 98, 101    | 101         | −3.8%      |

Median/mean/p90 per-call are **not** reported as signal: they swing ~2x between identical
runs (PRE median spanned 1272–2689 µs), i.e. desktop scheduling/thermal noise dominates those
statistics. G2 = 0 throughout (no full collections either build).

## Why the result is trustworthy

- **G0 ranges do not overlap.** PRE G0 ∈ [750, 769], POST G0 ∈ [691, 711]. Every POST run
  allocated less than every PRE run — this is not noise; it is a clean separation. This is
  the strongest evidence and it directly confirms the #114/#116 allocation removal landed.
- **min timing (−8.2%) corroborates** the GC result in the same direction and similar
  magnitude, consistent with allocation removal + the connbuffer modulo/​copy elimination.
- Both agree with the mechanism; neither was expected to (nor did) move `BarrierWaitUs`.

## Caveats

- Live-desktop measurement: only `min`/`G0` cut through the noise; the central per-call
  statistics are unreliable at n=5 and should be ignored.
- Reports were distinguished by the `[PRE]`/`[POST]` label + build timestamp; the probe's
  `build` line read `0.0.0.0` (unset assembly version). Fixed post-hoc — see below.

## Harness fix applied (for future A/Bs)

`RenderBenchmark.BuildStamp()` updated to emit the assembly **MVID** (module version id, a
GUID baked into every compiled binary that differs whenever source differs) alongside the
version and file timestamp. Future reports self-identify the exact binary even when the
assembly version is unset or two builds share a timestamp — removing reliance on the manually
set label. Change is in the updated `perf_instrument_render_benchmark.patch`.

## Interpretation / where this leaves the perf work

- The sweep did what it set out to do: measurably fewer allocations, slightly faster
  best-case engine time, zero risk to correctness (all changes bit-exact, render-gated).
- Magnitude (~8%) quantifies this *class* of work: facade/allocation/per-sample PRs each
  yield low-single-digit-to-~8% on the engine term. They do not, and cannot, address the
  structural walls — the per-wave barrier (§6.2, wide graphs) and the deep-chain per-chunk
  host floor (§12/§34) — which dominate on the deep/serial shapes (01x32 was ~90%
  unaccounted). Those remain the only levers with order-of-magnitude headroom, and both are
  structural/parked.
- Remaining unshipped sweep items (own PRs, host-engine, bit-exact): deferred **A5**
  (`GetEvents` alloc-kill, needs save/playback concurrency handling), **Tier-C** correctness
  fixes (note-off pruning; VU meter offset/scale), **SIMD** mix loops. A5 + note-off are the
  higher-value of these.
