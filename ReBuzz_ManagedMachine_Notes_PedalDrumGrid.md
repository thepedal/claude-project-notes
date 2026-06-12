# ReBuzz Managed Machine Development — Pedal DrumGrid Addendum

Source: ReBuzz current preview (June 2026; exact build unstamped — see Roster
`ReBuzz vs = ?`) + Pedal DrumGrid v1.0 build. A managed C# generator: a 16-lane
multi-out drum sampler. The trigger grid is the ReBuzz pattern editor itself
(16 global switch columns); each lane has its own audio output; samples come
from the wavetable (assigned in the GUI) or a self-contained `.pdrumgrid.xml`
kit with embedded audio.

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`; `Tracker §N`, `Chord §N`, `M1 §N`,
`PedalComp §N` point to those addenda; `Build §N` to the build notes. Internal
cross-references use plain `§N`.

Most findings here are *not* drum-specific — the trigger-grid layout technique
(§1) and the load-time pvalue clobber (§2) apply to any multi-track managed
machine.

---

## 1. Grouping a trigger grid: global switches, not track params

**Problem.** A drum grid wants all the lane triggers adjacent (one block), with
per-lane velocity/pitch off to the side. The obvious approach — one track per
lane with `Trigger`, `Velocity`, `Pitch` track params — can't produce that
layout. ReBuzz lays track parameters out **track-major**: `[T0:Trig T0:Vel
T0:Pitch][T1:Trig T1:Vel T1:Pitch]…`. The triggers interleave with velocity/pitch
and can never form a single adjacent block. (See the Roster song-authoring facts:
globals render *before* the per-track block, which repeats.)

**Technique.** Make the triggers **global** parameters — one `bool` switch per
lane (`Trig 1 … Trig 16`). A `bool` parameter is `ParameterType.Switch` (Core §9,
renders `0/1`). Globals render as a contiguous block left of the track section,
so 16 global trigger switches *are* the grid. Mark them `IsStateless` (Core §25):
they then stay out of the rack (no 16 useless sliders) but still appear as pattern
columns, and — because stateless params reset to `NoValue` each tick — a `1` on
consecutive rows **re-fires** every row (fast hats work). Velocity/Pitch stay as
**track** params, so they keep their natural per-track `[Vel, Pitch]` grouping on
the right.

**Bonus: no trigger collision.** `parametersChanged` is keyed by parameter, so 16
distinct global `Trig` params never collide on the same row — kick+hat+snare on
one step all deliver. This sidesteps the Core §14 multi-track problem *for the
triggers* entirely (it still applies to the shared Velocity/Pitch track params —
see §2). Net: the §14 sibling-poll recovery is **not** needed for triggering.

**Caveat.** The 16 trigger columns are always present (globals), but the per-lane
Velocity/Pitch columns only appear for tracks that exist. The machine
auto-initialises to 16 tracks (§8) so they're all there out of the box.

---

## 2. The pvalue-poll clobber at load — a §14/§16 recovery hazard

This is the most important reusable finding here. **Any machine that uses the
Core §14 / Tracker §16 "poll sibling tracks' `pvalues` from a setter" recovery is
exposed to it.**

The recovery reads a parameter's raw `pvalues` backing store for sibling tracks
(shape-tolerant: `ConcurrentDictionary` ≤1826, `int[256]` at 1827 — Tracker §16.3)
to recover values dropped by the §14 same-tick collision. The standard guard is
`if (raw != param.NoValue) accept(raw)`.

**The trap.** ReBuzz pushes each parameter's `DefValue` **through the setters at
load and on track creation — before playback**. At that moment the `int[256]`
`pvalues` array is freshly allocated and **all `0`**, *not* the `NoValue` (255)
sentinel it only becomes after the engine's first post-tick `Array.Fill`. So
`0 != NoValue` passes the guard, and the poll overwrites every sibling lane's held
value with `0`.

In DrumGrid this pinned the held **velocity** of every non-fired lane to `0` at
load → those lanes triggered at zero gain → **silent**. Only the directly-set
lane (skipped as `firedTrack`) survived. The debug trace was unambiguous:
`fire lane=2 vel=0 snap=26385smp` — trigger fine, sample fine, velocity zero.

**Two mitigations, and the right one:**
1. **Guard the poll on playback:** `if (!Buzz.Playing) return;` (and read only
   the relevant track). During playback the array is `NoValue`-filled for
   untouched tracks, so the guard works as intended.
2. **Never poll *sibling* lanes.** The original recovery clobbered because it
   walked every sibling track at load. Reading only the **firing lane's own**
   value is both safe and sufficient (see below).

**The follow-on, and why own-lane reads are actually required.** Removing the
poll entirely (an interim v1.0 choice) then exposed a second bug: **pattern-cell
velocity didn't affect the hit, but the column's header dial did.** Because the
trigger is a *global* param consumed in `Work` (§1) while Velocity is a *track*
param delivered through its own setter, on a row with both, the hit was read
before that row's `SetVelocity` had refreshed `_pendVel` — so cells appeared
inert while the dial (a live, held set) worked. The fix is to read the firing
lane's **own** Velocity/Pitch pvalue at trigger time, inside `SetTrig`:

```
on SetTrig(lane, true):
    if not Buzz.Playing:      return        # never read the array at load
    if lane >= TrackCount:    return        # no real track -> slot is uninit 0
    if velPvalue(lane) != NoValue:  _pendVel[lane]   = velPvalue(lane)
    if pitPvalue(lane) != NoValue:  _pendPitch[lane] = pitPvalue(lane) - 48
```

This works because the sequencer fills **every** pvalue for a row before **any**
setter runs — so when `SetTrig` fires, the same-row Velocity/Pitch pvalue is
already there, independent of setter ordering or whether `SetVelocity` is invoked
at all. The two guards are load-bearing: `Buzz.Playing` keeps it off the
all-`0` array at load, and **`lane < TrackCount`** keeps it off the slots beyond
the real track count (also still `0`, not `NoValue`) — without that bound, a
machine with `TrackCount = 1` but triggers on lanes 2+ pins those lanes' held
velocity to 0 and only lane 0 sounds (the exact regression that caught this).
Own-lane + hits-only means it can't clobber siblings, and same-row multi-lane
hits each read their own value, so the Core §14 collision is a non-issue without
any sibling walk.

**Rule of thumb:** when a trigger and its modifiers (velocity, pitch) live in
*different* parameters — especially global trigger + track modifier — don't rely
on the modifier's setter having run first. Read the modifier's pvalue at the
instant the trigger fires. And the §14 sibling-poll is only ever valid
mid-playback; an unguarded poll that also runs at load (they all do, via
`DefValue`) is a clobber waiting to happen.

---

## 3. Multi-out per-lane voice architecture

Standard multi-out generator (Tracker §12), one output per drum lane:

- `OutputCount = LANES + 1` declared **statically** in `MachineDecl` — out 0 =
  master sum, outs 1..N = per-lane dry. Never auto-sync `OutputCount` to track
  count: a saved connection to a channel ≥ current count crashes on reload
  (Tracker §12.2). Unconnected per-lane slots arrive `null` and cost nothing.
- Defining the `Work(IList<Sample[]> output, int n, WorkModes)` overload is what
  sets `MULTI_IO` (Tracker §12.1).
- One `Voice` per lane: one-shot playback, linear interpolation, a short
  anti-click envelope, a deferred re-trigger fade-cross (Tracker §4.2), and choke
  groups (lanes sharing a non-zero group cut each other — hi-hats). v1.0 keeps the
  DSP deliberately simple; Tracker's interpolation table + full fade-cross are the
  upgrade path.
- `OUT_SCALE = 32768` maps the snapshot's ±1.0 floats to the Buzz sample domain
  (Core §38).

**Testing gotcha (cost us a session):** connecting a *per-lane* output ("Track 1")
into the mixer instead of **Master** (out 0) means you hear only that one lane.
For a multi-out machine, check the connection's source channel before suspecting
the code.

---

## 4. Swing as a per-step trigger delay (reusing Chord's ratio math)

Pedal Chord (§3, §11) computes a long/short step-pair split with the invariant
`long + short == 2·Speed`, keeping average tempo locked at every swing value.
DrumGrid doesn't own a step clock — triggers arrive from pattern rows — so the
same ratio is recast as a **delay on the off-beat step**:

```
ratio       = 1 + Swing/100                 // 1.0 .. 2.0
longTicks   = 2·ratio / (ratio + 1)         // 1.0 .. 1.333 (per step pair)
delayTicks  = longTicks − 1                 // 0 .. 1/3 tick on the off-beat
delaySamples = delayTicks · SamplesPerTick
```

Off-beat parity comes from `Song.PlayPosition` (Tracker §2.3); a `Swing Phase`
switch flips which step of the pair is delayed (Chord's `SwingPhaseVal`). The
delay is realised with the sub-tick note-delay countdown in the voice (Tracker
§7.7). Because it's a per-trigger delay rather than a step clock, it's
tempo-locked and **Sub-Tick-Timing-independent by construction** — no
`SubTickInfo` read at all (cf. Chord §11.3).

---

## 5. Self-contained kit: embedded audio + names + defaults

`Save Kit` writes a fully self-contained `.pdrumgrid.xml` — no external files or
wavetable slots needed to share/reload it.

- Per lane: `<Sample kind="embedded" rootNote="…">BASE64</Sample>` where the
  payload is a **16-bit PCM WAV** (`WavWriter`, snapshot float ±1 → 16-bit). Save
  bakes audio from each lane's in-memory snapshot **regardless of original
  source** (wavetable / file / already-embedded). `WavReader` gained `Stream` and
  `byte[]` overloads to decode it back.
- `<Defaults velocity pitch chokeGroup>` round-trip per-lane velocity/pitch
  (captured from the held `_pendVel`/`_pendPitch` on save, re-applied on load) and
  choke group. A `name` attribute per `<Lane>` carries the sample/lane label
  (captured from `IWave.Name` on assignment, editable in the GUI).
- Loader kinds understood: `embedded` (base64 WAV), `file` (wav on disk), and
  `wavetable` (1-based slot). `embedded` is what Save writes.
- `PrebuildAll()` runs on the UI thread after any load so the first hit never
  decodes/allocates on the audio thread (M1 §2).

Other kinds (`file`, `wavetable`) remain for hand-authored kits; for very large
kits a zip package (manifest + `samples/`) would beat base64-in-XML's ~33%
overhead, but drum one-shots are small enough that single-file wins.

---

## 6. GUI-driven non-parameter state, and persisting it

Wave assignment was deliberately moved **off the pattern grid** (it started as 16
`IsWaveNumber` globals) into the GUI, so the trigger block butts directly against
the per-track Velocity/Pitch section (§1). Consequence worth internalising:

> Once state stops being a declared parameter, ReBuzz no longer auto-persists it.
> It **must** be serialised into `MachineState` (Core §39) by hand.

So `MachineState` (v3) serialises the kit's full per-lane state — kind, name,
choke, and the wavetable slot / file path / **embedded audio bytes** — making an
embedded-kit song self-contained on reload. The kit is the **single source of
truth**; the GUI reads/writes it and `MachineState` (de)serialises it. Version is
gated (`v3` writes names; `v2`/`v1` still read) per Core §39.2.

Velocity/Pitch are deliberately **not** duplicated into `MachineState` — they ride
the track params (ReBuzz-persisted with the song), avoiding the duplicate-state
drift PedalTracker §13.1 warns about. The kit *file* is where velocity round-trips
as a load-time default.

GUI is built in code (no XAML/BAML — Chord §1), `UserControl` since it's control-
composed (Core §26.7 — `FrameworkElement` only if an `OnRender` surface is added).

---

## 7. Wavetable read API — members verified, and the gotchas

Confirmed against this build (the enumeration path the notes hadn't pinned down):

- `Buzz.Song.Wavetable.Waves` → `IList<IWave>` (0-based slots).
- `IWave`: `.Layers` (`IList<IWaveLayer>`), `.Name` (string), `.Flags`
  (`WaveFlags`).
- `IWaveLayer`: `.SampleCount`, `.SampleRate`, `.RootNote` (Buzz-byte form —
  convert via M1 §2.1 `BuzzByteToMidi`), `.GetDataAsFloat(buf, …)` (Tracker §7.1).

**Gotchas that cost a build each:**
- `WaveFlags.Stereo` lives on **`IWave`**, not `IWaveLayer` — reading
  `layer.Flags` is a compile error (CS1061). Read `wave.Flags`.
- `waveIndex` in the pattern/picker is **1-based**; `Waves` is 0-based, so
  `slot = waveIndex − 1`.
- Do **not** reference `ReBuzz.exe` in the `.csproj` — it's a native apphost with
  no managed metadata → MSB3246 "PE image does not have metadata". Reference only
  `BuzzGUI.Interfaces` + `BuzzGUI.Common` (Build §1.1); every type used here comes
  from those.

---

## 8. Auto-initialising the machine's own `TrackCount`

A machine with a fixed lane count wants those tracks present without the user
manually adding them — and present **immediately on insertion**, not only once
playback starts. There's **no declarative hook** — `MachineDecl` has `MaxTracks`
(an upper bound) but no `MinTracks`/initial-count. So set it imperatively via
`IMachine.TrackCount`, with these constraints:

- **UI thread only.** The setter triggers `CMI_SetNumTracks` (Core §19, §21);
  calling it on the audio thread risks the IPC deadlock §21 warns about. Marshal
  with `System.Windows.Application.Current.Dispatcher.BeginInvoke(...)`.
- **Drive it from the constructor, but deferred + retrying.** `host.Machine` is
  null that early (Core §16.1), so you can't set it inline. But a `BeginInvoke`
  scheduled from the constructor runs *after* construction unwinds, by which
  point `host.Machine` is populated — so the tracks appear as soon as the machine
  is added, before any `Work`. The catch is you must **retry**, not mark-done on
  first attempt: if the dispatched call still finds `host.Machine == null`, leave
  the "done" flag clear so a later attempt can fix it. A first-`Work` call is the
  safety net (it reschedules if needed, and `BeginInvoke` is safe to *queue* from
  the audio thread even though the setter must *run* on the UI thread).
- **Bump-up-only.** `if (m.TrackCount < n) m.TrackCount = n;` — never truncate.
  Old songs saved at a smaller count get padded to 16 (lossless: existing track
  data is preserved, empty tracks added); songs already at 16 are a no-op. Only
  raised **once** (the `_tracksInit` latch), so a user who later reduces the count
  by hand isn't fought.

```csharp
bool _tracksInit, _tracksScheduled;

void ScheduleEnsureTracks()        // from ctor AND first Work (both idempotent)
{
    if (_tracksInit || _tracksScheduled) return;
    _tracksScheduled = true;
    var disp = System.Windows.Application.Current?.Dispatcher;
    if (disp != null) disp.BeginInvoke((Action)EnsureInitialTracks);
    else { _tracksScheduled = false; EnsureInitialTracks(); }
}
void EnsureInitialTracks()         // runs on the UI thread
{
    _tracksScheduled = false;
    if (_tracksInit) return;
    var m = host?.Machine;
    if (m == null) return;         // not ready — a later ScheduleEnsureTracks retries
    if (m.TrackCount < LANES) m.TrackCount = LANES;   // public setter (still {get;set;} @1827)
    _tracksInit = true;
}
```

**Why not Work-only (the earlier cut).** Triggering solely from the first `Work`
means the tracks don't appear until the user presses play — add the machine, open
the pattern stopped, and you still see one track. Worse, latching "done" on the
first attempt regardless of success means a null `host.Machine` permanently skips
it. The constructor-scheduled + retry form fixes both.

Bonus: when ReBuzz creates the new tracks it pushes each track param's `DefValue`
through the setters, so the new lanes' `Velocity`/`Pitch` initialise to 127/0
(`_pendVel`/`_pendPitch`) — no extra wiring. (This is also why the §2 capture's
`lane < TrackCount` guard now permits all 16 lanes.)

## Depends on

- **Build** §1.2 (csproj — fully compliant: `net10.0-windows`, `UseWPF`, no
  pdb/deps, `NoWarn MSB3277`), §1.3 (deploy → `Gear\Generators`), §2 (`.NET`
  AssemblyName suffix), §1.1 / §6.1 (reference only `BuzzGUI.Interfaces` +
  `BuzzGUI.Common`; **not** `ReBuzz.exe` — §7).
- **Core** §9 (`bool` → Switch — §1), §16.1 (lazy `host.Machine` — §8), §19/§21
  (`TrackCount` setter, UI thread / `CMI_SetNumTracks` IPC — §8), §25 (`IsStateless`
  hides from rack / pattern re-fire — §1), §14 + §42 (multi-track
  `parametersChanged` collision and the `int[256]` `pvalues` shape — §2), §26.7
  (GUI base class — §6), §38 (Buzz sample scale — §3), §39 (`MachineState`
  framing + versioning — §6).
- **Tracker** §12.1–§12.3 (multi-out contract, static `OutputCount` — §3), §16.3
  (shape-tolerant `pvalues` reader — §2), §4.2 (deferred re-trigger fade — §3),
  §7.1 (`GetDataAsFloat` — §7), §7.7 (sub-tick note delay — §4), §2.3
  (`Song.PlayPosition` — §4).
- **Chord** §3, §11 (swing ratio / tempo-locked split, recast as delay — §4).
- **M1** §2 (wavetable snapshot off the audio thread — §5), §2.1 (`BuzzByteToMidi`
  root-note form — §7).
- **PedalTracker** §13.1 (don't duplicate param values into `MachineState` — §6).

1827 note: DrumGrid does **not** pin to the 1827 "pvalues field" multi-track fix.
Its trigger grid is global switches (which don't collide — §1), and the one place
that touched the `pvalues` array (the velocity recovery) was removed (§2). So it
builds against any preview that supports .NET 10 managed machines, the multi-out
contract, and the wavetable API of §7. Stamp the actual build in the Roster.
