# ReBuzz render nondeterminism — finding (2026-07-08)

## Summary

`det_grid_08x04` does **not** render bit-identically every time on `wasteddesign/main`
= `355deba`, even under the controlled gate protocol (one offline float32 render per
fresh app launch, AudioThreads=8, RME Fireface ASIO). Two distinct intermittent effects
were observed. This undermines byte-exact render gating and is logged as its own
engine-robustness item, separate from any perf PR.

## Evidence

- **Base fresh-launch, 8 renders** (`355deba`, clean main, no patch): 7/8 byte-identical
  (`data` md5 `fe9b9252…`, zero garbage); 1/8 (`base_fresh_04`) had a silent lead-in.
- Earlier warm 3-in-session runs (`355deba`) showed the garbage spike in 2/3; `cd77415`
  warm 3-in-session showed 0/3 garbage but grossly divergent whole-render output
  (within-session drift — first render after launch is the outlier, later ones converge).

## Effect 1 — silent lead-in (~12%, milder)

- One render in eight: first ~114 ms (0.006–0.119 s, ~5441 frames) is all zeros where
  the other renders carry signal; identical to the reference from 0.119 s onward.
- Values are exactly 0.0 (silence), not garbage. Looks like an occasional render-start
  priming/latency-ring warm-up that drops the first buffers.

## Effect 2 — native-scale garbage sub-chunk (rarer, severe)

- One 128-frame Work sub-chunk (256 interleaved samples) occasionally emits a near-
  constant railed value ≈ ±7.86e6. Note **7,864,320 = 240 × 32768**: the garbage is
  native-domain (pre-`1/32768` output scale) data, i.e. an unscaled/stale buffer reaching
  the output.
- Window **wanders** run-to-run (observed at 1.70 s, 2.19 s, 2.33 s, 2.44 s, 2.65 s,
  3.81 s in different renders) — intermittent, not a fixed spot.
- Seen on `355deba` (warm 2/3; fresh-launch 1× on the arraycopy build). **Not** observed
  on `cd77415` yet, but only under warm 3-in-session there (not a comparable control), so
  trio-attribution is **unconfirmed**.

## Ruled out

- **#122 (MasterTapSamples double-buffer)** — inspected (`ReBuzzCore.MasterTapSamples`,
  ~line 1663). It is a **read-only GUI tap**: reads `resSamples` into a reuse buffer and
  `BeginInvoke`s to the meter/scope. It never writes `resSamples` or the audio output, so
  a race there garbles the meter, not the render. Exonerated for the render garbage
  despite the tempting native-scale coincidence (MasterTap is native-domain).
- **#123 (output-scale vectorise)** — deterministic, single-threaded in
  `MainAudioFillBuffer`; cannot intermittently skip a buffer. Bit-exact by construction.
- **arraycopy patch (`ManagedMachineHost.Work`)** — bit-exact `Array.Copy`; copies whatever
  bytes are in the buffer. Cannot create a garbage value. (A bit-exact change *can* shift
  thread timing and modulate a latent race, but it is not a source.)

## Open leads

- Source of the garbage is in the `resSamples` / output path itself. Remaining trio
  suspect is **#121** (`AudioEngine.cs` + `ReBuzzCore.cs`) — diff not yet analyzed for
  this. Or a pre-existing race exposed by the current rig.
- The unscaled (×32768) magnitude points at the boundary between native-domain mixing and
  the `1/32768` output scale — a buffer occasionally reaching output before/without the
  scale, or a stale native buffer.

## Consequence for methodology

Single-render byte-exact gating is **unreliable on `355deba`**: the base disagrees with
itself ~1/8 (silent lead-in) plus rarer garbage. Bit-exact-by-construction PRs (SIMD
scale, Array.Copy) should be validated by the construction argument plus a *clean-subset*
render match (anomaly-free renders must equal `fe9b9252…`), not by a single pair.

## Suggested next steps (when this item is picked up)

1. Reproduce each effect deliberately; get per-effect rates on `355deba` fresh-launch.
2. Proper pre-trio control: `cd77415` **fresh-launch** ×N (not warm 3-in-session) to
   confirm/deny trio attribution of the garbage.
3. If trio: bisect #121 vs #123 (fresh-launch renders on each parent commit).
4. Inspect #121's diff and the `resSamples`/output-scale path for the unscaled-buffer and
   priming-dropout races.
