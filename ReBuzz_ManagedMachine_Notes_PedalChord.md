# ReBuzz Managed Machine Development — Pedal Chord Addendum

Source: reconstructed 2026-06-05 from the original development thread
("Pedal Chord control machine chat 01", 2026-05-23) plus a read of the
**pedal-chord v1.5.3** source (`PedalChord.cs`, `PedalChord.csproj`).
No state file survived that chat; this addendum is the recovery. Where a
detail comes from memory of the chat rather than the shipping source it is
marked *(from chat)*; everything else was verified against v1.5.3.

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`; `Build §N` to
`ReBuzz_ManagedMachine_Notes_Build.md`. Internal cross-references use plain
`§N`.

Pedal Chord is a **control machine** (`public void Work()`, no audio I/O) that
reads root notes from its own pattern and writes chord/arpeggio notes onto a
target generator. As of v1.5 it is **single-voice / single-target**: one
instance drives one generator on one base track. For multiple targets, run
multiple instances.

**Depends on:** Build §1.2 (csproj — fully compliant, see §1), §1.3 (deploy →
`Gear\Generators`), §2 (`.NET` AssemblyName → "Pedal Chord"), §4 (`NoWarn
MSB3277`); Core §2 (control classification via `void Work()`), §4
(`SendControlChanges()` after writing to a target), §7 (note encoding), §8
(control machines tick first), §15 (lazy init of `host.Machine`). Core §14
(multi-track note recovery) **no longer applies** — see §6.

---

## 1. Build / deployment status

`PedalChord.csproj` is **fully compliant** with the project deployment rules:

```xml
<DebugType>none</DebugType>
<DebugSymbols>false</DebugSymbols>
<GenerateDependencyFile>false</GenerateDependencyFile>
```

Plus `NoWarn=MSB3277` (Build §4), `AssemblyName = "Pedal Chord.NET"` (Build §2),
`net10.0-windows`, `UseWPF=true`, `Platforms/PlatformTarget=x64`,
`OutputType=Library`. Output path is the gear folder directly:
`$(BuzzDir)\Gear\Generators`, default `C:\Program Files\ReBuzz`, overridable
via `/p:BuzzDir=...`. References (ReBuzz, BuzzGUI.Interfaces, BuzzGUI.Common)
are all `Private=false`. Settings UI is built in code, not XAML (`<Page
Remove>` / `<EmbeddedResource Remove>` strip all `.xaml`) to dodge BAML issues.

Built/tested against **1827-preview** as of v1.5.4 (§9). Verified clean — no
source changes were needed for the 1827 rebuild. Current version **v1.6.0**:
the step clock is now driven off the musical fraction `PosInTick/SamplesPerTick`
(§11), which supersedes the v1.5.5 sub-tick-edge mechanism (§10) and fixes a
tempo/sub-tick timing drift. v1.5.6 added the re-seed off-by-one guard (§10.5).

---

## 2. Timing model — the core finding

The first non-obvious thing learned building this machine, and the foundation
everything else rests on:

**`Work()` is called many times per pattern tick — once per audio buffer — not
once per tick.** A control machine that wants to advance state "per row" must
detect the tick boundary itself. Pedal Chord does it by watching
`IBuzzMachineHost.MasterInfo.PosInTick` reset toward 0:

```csharp
int  pit     = host?.MasterInfo?.PosInTick ?? 0;
bool newTick = pit < _prevPit;     // PosInTick counts up 0..SamplesPerTick-1, resets each tick
_prevPit     = pit;
```

All per-tick logic (slot note-offs, the arp countdown) is gated on `newTick`.
`MasterInfo` lives on `IBuzzMachineHost`, not `IBuzz`, and is only valid inside
`Work()` and parameter setters. `IBuzzMachine.Tick()` is **not called** for
managed machines — putting per-tick logic there is a silent no-op; it must all
live in `Work()`. (This finding generalised into the Core notes; it applies to
every managed control machine.)

Parameter/note delivery from `IBuzzMachineHost.Tick()` arrives before the first
`Work()` of that tick, so `HasNewNote` and the `newTick` flag are true together
on the same call.

---

## 3. Swing — current implementation (v1.5.3)

This is the area being reopened for the next phase, so it gets the most detail.

Swing is **whole-tick, integer countdown**. No fractional/subtick timing. The
arp advances only on `newTick` boundaries. Two parameters drive it:

- **Swing** (0–100, default 0) — 0 = straight, 100 ≈ 2:1 triplet shuffle.
- **Swing On** (`SwingPhaseVal`, 0/1) — which beat of the alternating pair gets
  the long wait.

State: a single `int ArpTicks` countdown and an `int ArpStepParity` that
strictly toggles 0↔1. In `Work()`, gated on `newTick`:

```csharp
if (_vs.Mode != 0 && _vs.ArpTicks > 0 && --_vs.ArpTicks == 0)
    StepArp(baseTrk);
```

`StepArp` fires the note, advances the arp index, then computes the *next*
countdown:

```csharp
float ratio  = 1f + _vs.Swing / 100f;                                   // 1.0 .. 2.0
int   longT  = (int)Math.Round(2.0 * _vs.Speed * ratio / (ratio + 1.0));
int   shortT = Math.Max(1, 2 * _vs.Speed - longT);
longT        = Math.Max(1, 2 * _vs.Speed - shortT);   // re-derive so long+short == 2*Speed exactly
bool  isLong = ((_vs.ArpStepParity + _vs.SwingPhaseVal) % 2 == 0);
int   baseT  = isLong ? longT : shortT;
_vs.ArpStepParity = 1 - _vs.ArpStepParity;            // strict toggle, never grows

int drift  = _vs.Humanize > 0 ? (int)Math.Round(_vs.Speed * _vs.Humanize / 200.0) : 0;
int jitter = drift > 0 ? _rng.Next(-drift, drift + 1) : 0;
_vs.ArpTicks = Math.Max(1, baseT + jitter);
```

The invariant `longT + shortT == 2 × Speed` is what keeps **average tempo
locked at every Swing value** — the trigger count over a loop is unchanged by
swing, only the spacing alternates. Humanize jitter is added on top of the
swung base, non-cumulative, so it never drifts the tempo either.

### 3.1 The granularity limitation — whole-tick only (resolved in v1.5.5, §10)

In whole-tick mode `longT`/`shortT` are whole ticks, so swing resolution is
bounded by Speed:

- **Speed=2**: the only integer split of `2×Speed=4` near a shuffle is 3/1, so
  *every* non-zero Swing collapses to the same 3:1 feel. No gradation.
- **Speed=3**: Swing=50 gives a clean 2:1.
- **Speed≥4**: the full 0–100 range produces meaningfully distinct ratios.

So swing is only nuanced at higher Speed. At the low Speed values you'd use for
fast arps over high BPM/TPB, swing is coarse or absent. **v1.5.5 (§10) fixes
this** by counting the step clock in sub-ticks when the host runs Sub-Tick
Timing, giving R× finer placement. This limitation is therefore only in effect
when Sub-Tick Timing is off.

---

## 4. How swing got here — the v1.1 development arc

Worth recording because three plausible-looking approaches each failed in an
instructive way *(from chat)*:

1. **Offset formula** — `_swOff = (int)(Speed × Swing / 300)`, long/short =
   `Speed ± _swOff`. **Granularity cliff**: at Speed=2, Swing=50 →
   `2×50/300 = 0.33` → truncates to 0 → perfectly straight. Needed Speed ≥ 6
   before Swing=50 did anything at all.
2. **Float ratio-accumulator** — `ArpAccum += 1` per tick, fire when
   `ArpAccum >= ArpTarget` where `ArpTarget` is a fractional long/short value.
   **Broke trigger counts**: firing at `>=` always rounds up to the next whole
   tick, so a 2.4/1.6 pair fired at 3+2=5 ticks instead of 4 — the arp drifted
   slow relative to the sequencer.
3. **Carry-over accumulator** — subtract the fractional overshoot so the
   average self-corrects. Fixed the count, but surfaced **"swing disappears
   after 16 ticks"**: `ArpStepParity` was incremented without bound and
   interacted badly with the float carry-over; and **Speed > 8 broke** because
   the float `ArpTarget` plus carry-over went unreliable at larger values.

**Resolution (shipping):** drop floats entirely. Integer countdown, `Round`ed
long/short summing exactly to `2×Speed`, parity *strictly toggled* `1 - parity`
so it never grows. Audible swing from Speed=2 (3:1) upward, exact trigger
counts at all Speed/Swing combinations. That is the §3 code.

Same v1.1 also fixed the `SetMode` clamp (`Math.Min(4, value)`) that made **Arp
Random (Mode 5) unreachable** from the pattern.

### 4.1 Master Groove sync — prototyped, then removed

The first idea for v1.1 was to follow the Buzz master groove
(`MasterInfo.GrooveData[]` / `GrooveSize` / `PosInGroove`) — multiply each step
duration by `GrooveData[step % GrooveSize]` so the arp shuffles in lockstep
with the song groove for free. A `ReadGrooveValue` helper and a `grooveVal`
multiplier were prototyped and then **removed** in favour of the explicit Swing
parameter, which was simpler and gave direct control. Master-groove sync is
therefore **not in any shipping build** — it remains an open roadmap option
(§7), and the two could coexist (groove as the base, Swing scaling on top).

### 4.2 Swing On display

`Swing On` shows as raw `0`/`1` in the pattern rather than named values.
`ValueDescriptions` was removed because it caused a **hang** *(from chat)*; the
root cause was never chased down. Worth revisiting if the parameter UI is
touched, but cosmetic.

---

## 5. Parameters (v1.5.x)

| Parameter   | Range    | Notes |
|-------------|----------|-------|
| Note        | C-0–B-9  | Root note (Core §7 encoding). |
| Velocity    | 1–127    | Sent to target; located by name ("Volume"/"Velocity"/"Vol"/"Vel") or by position (param after the note param). |
| Chord       | 0–50     | 51 chord types (hex 00–32). |
| Mode        | 0–5      | Chord / Up / Down / Up+Down / Down+Up / Random. |
| Speed       | 1–1024   | Pattern ticks between arp steps. Exact regardless of BPM/buffer. |
| Length      | 0–16384  | Note duration in ticks. 0 = no auto note-off (sustain to next). |
| Octaves     | 1–4      | Oct-Walk range, or pre-expanded span when Oct Walk off. |
| Step        | 1–8      | Chord tones advanced per arp step. |
| Oct Walk    | 0–2      | Off / Up / Ping-pong octave cycling. |
| Swing       | 0–100    | §3. |
| Swing On    | 0–1      | §3 / §4.2. |
| Humanize    | 0–100    | ±`Round(Speed×Humanize/200)` tick drift per step. §3. |
| Hum. Vel    | 0–100    | ±`Round(Velocity×HumVel/200)` velocity drift, clamped [1,127]. |
| Arp Reset   | 0–1      | Write 1 to restart the sequence at this row (stateless). |

`Length` was widened to 0–16384 *(from chat)* after the original 1–64 ceiling
was too short on high-BPM / high-TPB songs.

---

## 6. The v1.5 single-voice simplification

Through v1.4 the machine was multi-voice (up to 16 pattern tracks, each routing
to its own target) and leaned on reflection hacks — direct `pvalues` writes, a
`VoiceCache`, a `TryInitPValues` — and on **Core §14** multi-track note
recovery to deliver simultaneous chord rows without last-track-only collapse. A
roadmap item (the "v1.4" polyphonic arp, each step ringing on its own track)
was planned in this space.

v1.5 **scrapped all of it.** `MaxTracks = 1`; one machine drives one target on
one base track; multiple targets = multiple instances. Every reflection hack
removed; target machine + parameters cached, invalidated on settings change and
every 100 ticks. The result is far less code and no multi-track instability —
which is why **Core §14 no longer applies to this machine.** Chord mode still
writes multiple notes, but onto consecutive *target* tracks (Base, Base+1, …),
driven from this machine's single track, so there is no same-tick multi-track
write on Pedal Chord's own parameters.

---

## 7. Next-phase swing — open directions

The reason this file exists. The live constraint is §3.1: whole-tick swing is
coarse/absent at low Speed. Levers, roughly in order of payoff vs. effort:

1. **Subtick / fractional firing — DONE in v1.5.5 (§10).** ReBuzz 1827 ships
   **SubTickTiming**: the engine subdivides each tick into `SubTicksPerTick`
   sub-ticks and works machines per sub-tick. Pedal Chord now counts its step
   clock in sub-ticks when SubTickTiming is on, giving R× finer swing — see §10
   for the implementation and verification. Version eligibility is automatic
   (§9.2: managed machines report `HostVersion`=66 on 1827). The remaining
   levers below are still open.
2. **Master Groove sync (§4.1).** Bring back `GrooveData` as a base layer so
   the arp inherits the song's groove for free, with Swing scaling on top.
   Cheap and very Buzz-idiomatic.
3. **Groove pattern selector.** A small built-in library (Straight / Swing
   Light / Swing Heavy / Triplet / Reverse / Human) cycling across steps, with
   pattern length independent of the arp note count. More varied than a single
   Swing amount.
4. **Per-step offset column.** A pattern-writable `Offset` (−N..+N ticks, or
   sub-ticks once §7.1 lands) for total manual control per row. Best as a
   complement, not the sole control.

A natural v-next plan: rebuild on 1827, move the existing Swing math onto a
fractional/sub-tick base (1) so it works at all Speeds, then optionally layer
groove sync (2) as a base.

---

## 8. Roster maintenance

The Roster lists `pedal-chord` (control, last pushed 2026-05-23, 1817-preview)
with the **Addendum column blank**. When this file lands in the project, flip
that to **Y**, bump **ReBuzz vs** to **1827-preview** (v1.5.4, §9), update
**Last pushed**, and add under the per-machine notes index:

> ### pedal-chord — `ReBuzz_ManagedMachine_Notes_PedalChord.md`
> - Single-voice (v1.5+) chord/arp peer controller. Whole-tick integer swing;
>   next phase moves to sub-tick (1827) for low-Speed swing resolution.
> - Depends on: Build §1.2/§1.3/§2/§4; Core §2, §4, §7, §8, §15. Core §14
>   *was* relevant pre-v1.5 (multi-track) but no longer applies (§6).
>   1827: rebuilt clean as v1.5.4 — no source change (§9); §42 and §37 don't
>   apply.

---

## 9. The 1827 rebuild — v1.5.4

v1.5.4 is a recompile-and-retest against ReBuzz 1827-preview, verified against
the 1827 source. **No functional source changes were required**; only the
README version/changelog were touched. The build itself needs nothing special
— the csproj references `$(BuzzDir)\*.dll`, so building against a 1827 install
links the 1827 API DLLs automatically (§1).

### 9.1 Why nothing broke — the three 1827 changes vs. this machine

The three consequential 1827 changes (per Core §42, §37, and the SubTick work):

- **`pvalues` field-shape flip (Core §42)** — only bites the §14 multi-track
  reflection poll. Pedal Chord is single-voice since v1.5 (§6) and removed all
  reflection hacks; it never polls `pvalues`. **N/A.**
- **MasterTap → GUI-thread event (Core §37)** — only bites machines that hook
  `MasterTap`. Pedal Chord doesn't. **N/A.**
- **SubTickTiming / AudioBufferFillThread** — changes `Work()` cadence (finer,
  more frequent calls). Pedal Chord gates *all* per-tick logic on the §2
  `newTick` (PosInTick-reset) detection, which is robust to any call frequency.
  Verified in the 1827 source (`WorkManager.cs`): `masterInfo.PosInTick`
  advances by each processed chunk and resets to 0 *only* at the full tick
  boundary; SubTickTiming further subdivides the chunk and resets a separate
  `PosInSubTick`, but never touches `PosInTick`'s tick-relative meaning. So
  `newTick = pit < _prevPit` fires exactly once per tick, never spuriously
  mid-tick. **Correct on 1827, with or without SubTickTiming.**

The one remaining ReBuzz-internal reflection site, `EnsureTrackCount`, tries
the public `IMachine.TrackCount` setter first (still `{ get; set; }` in 1827)
and only falls back to reflection inside a try/catch — the Core §40/§16.4
defensive pattern. No issue.

### 9.2 The managed-machine interface-version mechanism (matters for §7)

Confirmed in `ManagedMachineDll.cs`: a managed machine reports
`nmi.Version = Global.Buzz.HostVersion`, and `ReBuzzCore.HostVersion => 66` on
1827. So **every managed machine running on 1827 reports interface version 66**
— inherited from the host at runtime, not baked into the DLL. The engine's
sub-tick setter-delivery gate (`MachineWorkInstance.Tick`, `WorkManager`
control/non-control loops) checks `DLL.Info.Version >= 42`, which a managed
machine therefore passes automatically. Consequence for the next phase: there
is **no version attribute to set in Pedal Chord's source** to "enable subtick";
eligibility is automatic on 1827. Sub-tick swing is purely a matter of reading
`SubTickInfo` in `Work()` and firing on sub-tick boundaries, plus the user
enabling SubTickTiming in engine settings.

---

## 10. Sub-tick swing — v1.5.5

> **Superseded by §11 (v1.6.0).** The sub-tick-edge step clock described here was
> replaced by a sample-fraction clock. It is kept for history and because the
> root-cause analysis in §11 builds on it. The §10.5 re-seed reasoning still
> applies in spirit (and the property is preserved by the §11 clock).

Resolves the §3.1 low-Speed granularity limit by counting the arp step clock in
**sub-ticks** instead of whole ticks when the host runs SubTickTiming. Pending
on-hardware confirmation at time of writing; verified by simulation (§10.3).

### 10.1 Mechanism (verified against the 1827 source)

- `IBuzzMachineHost.SubTickInfo` (`SubTicksPerTick`, `CurrentSubTick` [0..R-1],
  `SamplesPerSubTick`, `PosInSubTick`) is populated per processed chunk by
  `MachineManager.UpdateMasterAndSubTickInfoToHost()`, called in
  `WorkManager`'s render loop **before** each `ReadWork()` (which works control
  machines every chunk via `CollectControlMachinesThatCanWork`). With
  SubTickTiming on, chunks are clamped to sub-tick boundaries, so a control
  machine's `Work()` runs at least once per sub-tick.
- `MasterInfo.PosInTick` is unchanged by SubTickTiming — it stays tick-relative
  and resets only at the tick boundary (§2, §9.1). So the existing `newTick`
  detector is still the *tick* clock; `CurrentSubTick` edges give the *sub-tick*
  clock.
- When SubTickTiming is **off**, `SubTicksPerTick` is 0 → the machine uses
  `R = 1` → identical whole-tick behaviour. No new parameter; it follows the
  host setting.

### 10.2 The change (three edits, no new parameter)

Let `R = SubTicksPerTick` when `> 1`, else `1`. In `Work()`:

```csharp
int  R = (sti != null && sti.SubTicksPerTick > 1) ? sti.SubTicksPerTick : 1;
_curR  = R;
bool newStep = (R > 1) ? (newTick || sti.CurrentSubTick != _prevSub) : newTick;
_prevSub = (R > 1) ? sti.CurrentSubTick : 0;
```

- **Arp step** decrements/fires on `newStep` (sub-tick clock).
- **Note-offs** (Length, in ticks) stay gated on `newTick` — *this split is the
  critical correctness point*: counting Length in sub-ticks would make every
  note R× too short.

In `StepArp` the reload period is scaled by R, preserving the
`long + short == period` tempo lock at R× resolution:

```csharp
int   R      = _curR < 1 ? 1 : _curR;
int   period = 2 * _vs.Speed * R;                 // step units (sub-ticks)
float ratio  = 1f + _vs.Swing / 100f;
int   longT  = (int)Math.Round(period * ratio / (ratio + 1.0));
int   shortT = Math.Max(1, period - longT);
longT        = Math.Max(1, period - shortT);
// ... parity toggle as before ...
int drift  = _vs.Humanize > 0 ? (int)Math.Round(_vs.Speed * R * _vs.Humanize / 200.0) : 0;
```

`ArpTicks` (the countdown field, name retained) now counts step-units. It is
reloaded by `StepArp` on every fire, so the design stays in the proven
integer-countdown family (§4) — no floats, no accumulator fragility.

### 10.3 Verification (simulation)

64-tick runs, trigger count and gap spread by R/Speed/Swing:

- **Tempo locked at all R**: 33 fires/64t at Speed 2, 17 at Speed 4 — average
  gap = Speed exactly, every R and Swing. `long + short == period` always.
- **R=1 reproduces v1.5.4 exactly**: Speed 2 straight until Swing≈80, then the
  coarse 3:1 cliff.
- **R=8, Speed 2 (the win)**: Swing 20/40/60/80 → 1.13:1 / 1.46:1 / 1.67:1 /
  1.91:1 — smooth low-Speed gradation that whole-tick could not produce.
  Swing=100 approaches but doesn't reach a literal 2:1 (ratio = 1+Swing/100
  caps long at 2/3 of period; it tends to 2:1 as period grows with Speed/R).

### 10.4 Known transient

If the user toggles SubTickTiming (or changes SubTickResolution) *during*
continuous playback of a held arp, the in-flight `ArpTicks` countdown is in the
old unit for one step, producing a single off gap before the next `StepArp`
reload self-corrects. Every NoteOn calls `Start()`→`StepArp`, which reloads in
the current unit, so the window is only a sustained arp across a live setting
change — rare. Not worth rescaling state to avoid; documented instead.

### 10.5 Re-seed off-by-one fix (v1.5.6)

**Symptom.** Re-seeding the arp mid-stream with a NoteOn (a fresh root every
bar) made the first gap after each seed one (sub)tick short — `Speed − 1`
instead of the `gap == Speed` that §10.3 guarantees at Swing 0. A *held* note
that free-ran was always correct, so a single-chord spot check hid it; the
fault only surfaced when NoteOns recurred mid-stream. Reported repro: Mode Up,
Swing 0, Speed 8, root every 32 ticks → steps at 0,7,15,23 per bar instead of
0,8,16,24.

**Cause.** A re-seed routes NoteOn → `Start()` → `StepArp`, which fires the
first step and reloads `ArpTicks = baseT`. If that seed and `Work()`'s per-step
decrement (`--ArpTicks == 0`) both land on the same `newStep` edge, the
freshly-loaded counter eats one decrement and the next fire comes one unit
early. It is the §10.4 single-off-gap, but recurring on every re-seed instead
of once.

**Note on reproduction.** Static simulation of the exact v1.5.5 source did *not*
reproduce it: the `HasNewNote` branch `return`s before the decrement, so on that
model seed and decrement are already mutually exclusive and the gap is `Speed`.
The fault is reproduced only if the seed `StepArp` and the decrement both
execute for one edge — which can happen if the host delivers the seeding NoteOn
on a `Work()` call other than the one carrying the `newStep` edge. The fix is
therefore written to hold *regardless* of how the host schedules delivery,
rather than relying on the early `return`.

**Fix.** A per-edge guard `_firedThisStep`: reset to `false` at the top of every
`newStep` edge (before any `StepArp` this call), set to `true` by `StepArp`, and
checked by the decrement (`&& !_firedThisStep`). At most one step fires per
`newStep`, and a re-seed and the countdown can never both consume the same edge.
The early `return` in the `HasNewNote` branch is kept; the guard is the
load-bearing invariant. The explicit **Arp Reset** path (`PendingReset` → sets
`ArpTicks = 1`, fires via the same guarded decrement) is covered by the same
guard and tested independently.

**Invariant / acceptance.** A re-seeded arp (NoteOn or Arp Reset every N) and a
free-running arp produce identical gap sequences — no `Speed − 1` gap at any
seed boundary; the first step still lands on the seed tick; §10.3 (avg gap =
Speed, `long + short == period`, tempo lock) and the §10.4 transient are
unchanged; identical at R = 1 and R > 1. Verified across Speed ∈ {2,4,8},
Swing ∈ {0,50}, R ∈ {1,8}, re-seed every {16,32} for both entry points by
`tools/reseed_timing_test.py`, which also reproduces the pre-fix fault
(0,7,15,23) as a control so the test provably bites. Per-step cost is one bool
write/compare — negligible.

### 10.6 Open follow-ons (unchanged from §7)

Master-Groove sync (§4.1), a groove-pattern selector, and a per-step offset
column remain open. Sub-tick now gives the *resolution*; groove sync would give
the *source feel* — they compose (groove as base, Swing scaling on top).

---

## 11. Tempo-locked fraction step clock — v1.6.0

### 11.1 Symptoms that drove it

Two reports, one root cause: (a) arp triggers correct at default tempo but
drifting after a tempo change (e.g. 126→90 BPM, from a clean restart); (b)
toggling ReBuzz **Sub-Tick Timing** OFF threw the timing even at default tempo.
Pure tick counting (§2) is tempo-independent by construction, so the coupling
had to be in the sub-tick layer the v1.5.5 clock (§10) depended on.

### 11.2 Root cause (1827 engine source)

The machine's mode test was `R = SubTicksPerTick > 1 ? SubTicksPerTick : 1`, and
it counted `CurrentSubTick` edges, assuming exactly `R` edges per tick. In
`WorkManager.Generate`:

- The sub-tick **clamp** that makes chunks end on sub-tick boundaries is gated on
  `engineSettings.SubTickTiming` (`if (SubTickTiming && PosInSubTick+stp > SamplesPerSubTick) …`).
- The sub-tick **advance** (`PosInSubTick += stp; if (>=SamplesPerSubTick){ PosInSubTick=0; CurrentSubTick++ }`)
  is **not** gated — it runs regardless.
- `subTickInfo.SubTicksPerTick` is computed in `UpdateMasterInfo` and is **not**
  zeroed for managed machines when timing is off (the managed copy path,
  `MachineManager.UpdateMasterAndSubTickInfo`, passes it through raw; only the
  *native* `CopySubTickInfo` zeroes it).

So with Sub-Tick Timing OFF the machine still saw `SubTicksPerTick > 1` and a
`CurrentSubTick` that kept moving, but because chunks were no longer sub-tick
aligned the number of edges per tick no longer equalled `R` — and the mismatch
is tempo-dependent. Sample-accurate sim (`SubTickSize 260`, `SubTickResolution
Lower = ÷2`): at 126 BPM the machine assumed `R=10` but the engine emitted 7
edges/tick; at 90 BPM it assumed `R=14` but emitted 10–11 (irregular). The
`Speed×R` countdown then produced ~11.4-tick gaps at 126 and ~10.8 at 90 instead
of 8 — exactly the reported drift. With Sub-Tick Timing ON the clamp makes
edges = `R` and it happened to be correct, which is why default "worked".
`AudioBufferFillThread` and `SubTickResolution` are not causal (the former only
moves *who* calls the fill; `ThreadReadSpeedAdjust` time-stretches only when the
master **Speed** scrub ≠ 0).

### 11.3 The fix

Drop sub-tick-edge counting. Advance the step clock by the musical fraction
elapsed since the last `Work()` call: `dTicks = (PosInTick − prevPosInTick) /
SamplesPerTick` (with a single tick-wrap fixup; chunks never cross a tick
boundary). Accumulate `dTicks`; fire a step when the accumulator reaches the
current step length, carrying the remainder. Step length is computed directly in
**fractional ticks**: `span = 2·Speed`, `long = span·ratio/(ratio+1)`,
`short = span − long`, alternated by parity for swing; Humanize jitters it in
ticks. No `SubTickInfo` read at all.

Properties: tempo-locked by construction (a tick is a tick at any BPM);
independent of the Sub-Tick Timing setting; placement resolution = the audio
chunk (256 samples with timing off, the sub-tick size with it on), so swing is
smooth at low Speed — the v1.5.5 goal, kept, without the edge dependency. A
re-seed resets the accumulator and returns before accrual, so the §10.5 re-seed
property holds structurally (a seed re-anchors phase to the exact NoteOn). Arp
Reset sets `_stepAcc = _stepLen` to fire promptly. Note-offs stay on the tick
clock (Length in ticks). `VoiceState.ArpTicks` is now vestigial.

### 11.4 Verification

`tools/timing_test.py` co-simulates the 1827 loop (sub-tick on and off) and
asserts every fire stays within one placement chunk of the grid — i.e. no drift
— across BPM ∈ {60,90,100,126,128,140,174}, TPB ∈ {4,8,16}, sub-tick on/off,
with re-seed every bar, and swing spans summing to `2·Speed`. It also runs the
old edge-counter against the sub-tick-OFF engine to reproduce the bug (gaps ~11.4
at 126, ~10.8 at 90) as a control. The earlier `reseed_timing_test.py` is
replaced by this.

### 11.5 Caveat

Diagnosed and fixed from the 1827 source + sample-accurate simulation; not yet
confirmed on hardware at the time of writing. The fix removes the only
tempo-coupled element in the path, so it should resolve both reports — to be
verified in ReBuzz at 90 BPM with Sub-Tick Timing both on and off.
