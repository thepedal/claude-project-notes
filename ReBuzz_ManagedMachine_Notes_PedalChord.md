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

Last built/tested against **1817-preview** per the Roster; not re-verified
against 1827. See §7 for the 1827 consideration relevant to the next phase.

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

### 3.1 The granularity limitation (the live issue)

Because `longT`/`shortT` are whole ticks, swing resolution is bounded by Speed:

- **Speed=2**: the only integer split of `2×Speed=4` near a shuffle is 3/1, so
  *every* non-zero Swing collapses to the same 3:1 feel. No gradation.
- **Speed=3**: Swing=50 gives a clean 2:1.
- **Speed≥4**: the full 0–100 range produces meaningfully distinct ratios.

So today swing is only nuanced at higher Speed. At the low Speed values you'd
use for fast arps over high BPM/TPB, swing is coarse or absent. This is the
central thing to fix in the next phase (§7).

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

1. **Subtick / fractional firing.** ReBuzz 1827 ships **SubTickTiming** (Buzz
   machine interface v42+): the engine subdivides each tick and delivers
   setter calls at sub-tick boundaries, and the Modern Pattern Editor stores
   event times in a sub-tick base so off-grid events play at their true
   position instead of snapping up. For Pedal Chord this is the real fix —
   firing the arp at a fractional position within a tick (via `PosInTick` +
   sample counting, the machinery is already half there) would give smooth
   swing at *any* Speed, killing the §3.1 cliff. Caveat: only machines built
   against interface v42+ see sub-tick setter delivery, and the machine is
   currently stamped 1817-preview — **rebuild/retest against 1827 first.**
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
that to **Y**, and add under the per-machine notes index:

> ### pedal-chord — `ReBuzz_ManagedMachine_Notes_PedalChord.md`
> - Single-voice (v1.5+) chord/arp peer controller. Whole-tick integer swing;
>   next phase moves to sub-tick (1827) for low-Speed swing resolution.
> - Depends on: Build §1.2/§1.3/§2/§4; Core §2, §4, §7, §8, §15. Core §14
>   *was* relevant pre-v1.5 (multi-track) but no longer applies (§6).
