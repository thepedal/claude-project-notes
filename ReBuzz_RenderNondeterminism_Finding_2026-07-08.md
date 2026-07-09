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

---

## Code-level analysis (2026-07-08, cont.) — trio fully exonerated, mechanisms localized

### Entire trio cleared by inspection (garbage is PRE-EXISTING)
- **#121** (`13b97d1`): only adds `Interlocked.Increment(ref DriverResets)` inside the
  ASIO `DriverResetRequest` / WASAPI `PlaybackStopped` fault handlers, plus a static
  counter + accessors. Touches no sample buffer / no render path. Cannot corrupt output.
- **#122** (`cc52f7a`): read-only GUI meter tap (`MasterTapSamples`), already cleared.
- **#123** (`7b227fb`): deterministic single-threaded output scale; bit-exact.
- The "355deba only, not cd77415" evidence was never valid (cd77415 was only warm
  3-in-session; garbage rate is low enough that 0/N there proves nothing). Treat the
  nondeterminism as pre-existing, surfaced by rigorous byte-comparison.

### Effect 1 — silent lead-in: mechanism found
`WorkManager.MainAudioFillBuffer`: at a tick/subtick boundary, if `samplesToProcess`
computes negative, the method does `return 0` **mid-fill, before the post-loop scale
pass** (lines ~173–174 and ~183–184). A transient tick-counter inconsistency during
warm-up yields a short/zero buffer → silent lead-in. Return value 0 (not `count`) likely
read by the render as silence for that region.

### Effect 2 — native-scale garbage: localized to the MT work-dispatch join
- `MainAudioFillBuffer` fills native-scale sub-chunks in a `while` loop, then applies ONE
  `*= 1/32768` pass over `[offset, offset+count)` AFTER the loop (WorkManager.cs ~246–261).
- `ReadWork` writes output from `master.GetStereoSamples()` (lines 636–642) — correct only
  if all worker writes are joined first.
- Non-cached / groups-cached paths join with `Task.WaitAll`. The **algorithm-2 cached**
  path (`HandleWorkAlgorithm2Cached`, ~963) instead uses a custom barrier in
  `WorkThreadEngine`: `AddWork` (counter++, list.Add, allDoneHandle.Reset) →
  `AllJobsAdded()` (workWaitHandle.Set) → `AllDoneEvent().WaitOne()`; workers drain via
  `GetWorkItem`, run `TickAndWork`, then `WorkDone` (counter--, Set allDoneHandle at 0).
- **Hazard:** `workWaitHandle` (start gate, a `ManualResetEvent`) is only reset inside
  `GetWorkItem` when the list is found empty (line 139). When jobs ≈ workers, that reset
  can lag past the *next* wave's `AllJobsAdded()`, letting a worker pick up next-wave jobs
  early and thrashing the counter to 0 transiently mid-dispatch. workLock serialization
  keeps the common case from hanging, but this is the classic barrier shape where a worker
  can still be inside `TickAndWork` when `WaitOne()` releases → `GetStereoSamples` mixes a
  half-written native-scale machine buffer → one sub-chunk of ≈(value×32768) garbage.
  Matches: one sub-chunk wide, native scale, wandering, intermittent, MT-only.
- This is the §6.2 / #107 barrier — the `dev-barrier-instr` stash already instruments it.

### Decisive cheap experiment (do first)
Toggle **Multithreading OFF** (or force single-thread work order) on the current
`355deba` build and repeat the 8× fresh-launch `det_grid_08x04` renders. Garbage (and
ideally the lead-in) vanishing ⟹ MT work-dispatch barrier confirmed as the source.
Persisting ⟹ look elsewhere.

### Needed to finish
- Live engine settings: `Multithreading`, work `algorithm` (0/1/2), `UseCachedWorkOrder`
  — determines which dispatch/join path is live (custom barrier vs `Task.WaitAll`).
- If barrier-confirmed: add an assert/log in `ReadWork` right before `GetStereoSamples`
  that `workEngine` reports zero in-flight (counter==0 AND no worker inside `TickAndWork`);
  the `dev-barrier-instr` probes are the vehicle.

---

## CONFIRMED: both effects are multithreading-only (2026-07-08)

MT-off control: 8 fresh-launch `det_grid_08x04` renders on `355deba` with **Multithreading
OFF** → **all 8 byte-identical**, md5 `fe9b9252…` (== the clean MT reference), **zero
garbage, zero silent lead-ins**.

Conclusions:
- Effect 1 (silent lead-in) and Effect 2 (native-scale garbage) are **both MT-only**.
  Single-threaded, the engine renders `det_grid_08x04` deterministically and correctly.
- MT-off output equals the MT *clean* output exactly ⟹ MT is not computing different
  audio, only intermittently corrupting it ⟹ a **synchronization defect**, not a logic
  difference. Confirms the work-dispatch join hypothesis (worker still in `TickAndWork`
  when the barrier releases → `master.GetStereoSamples` mixes a half-written native-scale
  buffer).

Methodology escape hatch: **render bit-exact gates with Multithreading OFF** — fully
reproducible, so byte-comparison is valid again for future bit-exact PRs.

Still open: which MT path is live (custom `WorkThreadEngine` barrier on algorithm-2 cached
vs `Task.WaitAll` on the group paths). Need engine settings: `algorithm` (0/1/2) and
`UseCachedWorkOrder`. Tie-breaker: MT on + `UseCachedWorkOrder` OFF ×8; garbage returning
only with cached-order ON pins it to the cached-barrier path.

---

## Live path CONFIRMED: algorithm 2 ("Threads") + UseCachedWorkOrder On (2026-07-08)

User settings: work algorithm dropdown = **"Threads"** (index 2), UseCachedWorkOrder =
**On**. cbAlgorithms order (PreferencesWindow.xaml.cs ~152-164): 0 "Recursive Task
Groups", 1 "Recursive Tasks", 2 "Threads". So the live dispatch is
**`HandleWorkAlgorithm2Cached`** → the custom `WorkThreadEngine` barrier
(`AddWork`/`AllJobsAdded`/`AllDoneEvent().WaitOne()`), NOT `Task.WaitAll`. Matches the
prime suspect; MT-off elimination confirms the join is the fault site.

### Real defect found (worth fixing regardless)
Start gate `workWaitHandle` is reset by **workers** (`GetWorkItem`, line 139), not the
dispatcher → cross-wave set/reset race: a lagging worker's `Reset()` can clobber the next
wave's `AllJobsAdded()` `Set()`, and workers can pick up jobs mid-dispatch.

### Honest caveat on mechanism
Static analysis of the **counter** logic (workLock-guarded; AddWork Resets allDone,
WorkDone-to-0 Sets it) suggests `WaitOne()` returning does imply `counter==0` implies all
`TickAndWork` returned — i.e. the counter itself looks robust. So the garbage is more
likely a **skipped/stale wave** (gate clobber → a machine never runs → its buffer left
stale/uninitialized, a better fit for native-scale garbage than a half-write) or a
buffer-ownership issue in the cached order. Exact interleaving NOT provable from source
alone — needs runtime instrumentation.

### Decisive next step (instrument, don't theorize)
1. Fresh branch off `355deba`; apply barrier instrumentation from `dev-barrier-instr`
   stash (`stash@{0}` / backup patch `wip_stash_backup.patch`).
2. Add an "active workers" `Interlocked` count (inc before `TickAndWork`, dec after) and
   assert, at the top of `ReadWork`'s mix (before `master.GetStereoSamples()`), that both
   `workCounter == 0` AND active-workers == 0.
3. Reproduce MT-on 8× fresh renders; capture which invariant trips on a garbage render:
   - counter==0 but active>0 ⟹ early barrier release.
   - a machine never ran ⟹ skipped/stale wave from the gate clobber.
4. Fix accordingly — likely rework the start gate so the DISPATCHER owns it (generation /
   countdown design), retiring the worker-reset clobber race.

---

## CORRECTION: not the barrier — bug is common to ALL MT dispatch paths (2026-07-08)

Differential test: algorithm **0 ("Recursive Task Groups")**, which joins with
`Task.WaitAll` (not the custom barrier), MT on, 8× fresh renders:
- 6/8 clean (`fe9b9252`); run6 = **garbage sub-chunk** (256 samples, native-scale,
  2.370 s); run1 = lead-in-class anomaly. Garbage rate ~1/8 — same as algorithm 2.

**Implication:** the garbage appears on the `Task.WaitAll` path too. `Task.WaitAll` is a
correct join, so this is **not** a premature-release/barrier defect. The
`WorkThreadEngine` start-gate reset/set race I found is real but is **NOT** the garbage
root cause. Barrier is exonerated as the source.

**Refocused hypothesis:** a race **common to all MT paths**, downstream of the join — the
shared mix/accumulation. Prime suspects now:
- `master.GetStereoSamples()` (the "input accumulation" B2 candidate) and the per-machine
  output buffers it reads — concurrent access to a buffer the parallelization treats as
  independent but isn't, or a shared scratch in the accumulation.
- The **cached work order** (`perf/cached-work-order`, merged) occasionally mis-grouping
  *dependent* machines to run in parallel → read/write race on a connection buffer.

### Next differential (decides between the two)
MT on, algorithm 2 or 0, **UseCachedWorkOrder OFF**, 8× fresh:
- garbage vanishes ⟹ cached-order mis-parallelization; fix in the ordering.
- garbage persists ⟹ fundamental shared-buffer/mix race (`GetStereoSamples` / machine
  buffer ownership); instrument there.

Both effects (garbage + lead-in) confirmed on algorithm 0 as well, consistent with Effect
1 living in `MainAudioFillBuffer` (common to all paths).

---

## ISOLATED to UseCachedWorkOrder ON; concrete defect: missing Ready-gate (2026-07-08)

Differential (algorithm 2 "Threads", MT ON, **UseCachedWorkOrder OFF**), 8× fresh:
**8/8 byte-identical `fe9b9252`, zero garbage, zero lead-in.**

Full table:
| MT  | cached | result        |
|-----|--------|---------------|
| off | —      | 8/8 clean     |
| on  | on (a2)| garbage ~1/8  |
| on  | on (a0)| garbage ~1/8  |
| on  | off(a2)| 8/8 clean     |

Bug tracks **UseCachedWorkOrder = ON + MT** (both algorithms 0 and 2), i.e. the merged
`perf/cached-work-order`. Join mechanism is NOT it (a0 uses Task.WaitAll, a2 the barrier;
both fail with cached on, both clean with cached off / MT off).

### Corrected: wave grouping is CORRECT (not a mis-grouping)
`CachedWorkOrder` builds `AudioWaves` by Kahn topological layering over the master audio
cone (CachedWorkOrder.cs 114-160). A machine's in-degree only reaches 0 after its inputs'
wave completes, so a machine cannot share a wave with a transitive input. Earlier
"mis-groups dependent machines" guess is disconfirmed.

### Concrete defect found: audio-waves dispatch skips the Ready-gate
- `FillAndSortDescFrom` (WorkManager.cs 922) copies ALL wave machines and dispatches them
  unconditionally — **no `Ready` check**.
- `DispatchPrefixCached` (949) DOES check `m.Ready && !m.workDone`; its comment: "Ready is
  re-tested here (a native crash can clear it mid-buffer)."
- Non-cached path gates on Ready via `CollectMachinesThatCanWork`.
- So the cached AUDIO-WAVES dispatch is the only path that works a machine without
  checking `Ready` (runtime-mutable). Working a not-Ready machine → reads unfilled/stale
  input or mid-reset state → native-scale garbage sub-chunk. Intermittent (Ready
  transitions are timing-dependent). Matches the cached-ON-only isolation.

### Fix candidate (mirror existing code)
Gate the audio-waves dispatch on `Ready` in BOTH cached paths
(`HandleWorkAlgorithm2Cached` ~999-1005 and `HandleWorkAlgorithmGroupsCached` ~1039-1045):
skip machines where `!Ready` (they keep last-chunk stale-but-valid output, as the other
paths already do). Confirm first with instrumentation (log if a not-Ready machine is
dispatched on a garbage render), then validate: MT on + cached on + fix, ~16× fresh
renders all clean.

### Workaround available now
`UseCachedWorkOrder = Off` — proven 8/8 clean; reliable renders + valid byte-gates today,
at the cost of the cached-order perf gain.
