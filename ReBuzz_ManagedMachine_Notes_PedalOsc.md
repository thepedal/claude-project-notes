# ReBuzz Managed Machine Notes — Pedal OSC

Source: Pedal OSC **v1.1** (2026-07-24) — an **effect**-class "tap" machine that
passes audio through unchanged, measures level, derives **beat and bar phase**
from the host transport, and streams the lot as an **OSC bundle** over UDP to an
external renderer. It is the ReBuzz side of a **host → real-time video bridge**
(see `ReBuzz_Video_Bridge_Brief`): numbers leave the host over OSC and drive
shader *uniforms* in a separate OpenGL process.

Repo: `github.com/thepedal/pedal-osc` (GPL-3.0). The `receiver/` folder there
holds the Python consumer side (diagnostic printer + moderngl renderer).

Sections numbered locally. `Core §N` → `..._Notes_Core.md`; `Build §N` →
`..._Notes_Build.md`; `PedalComp §N` → the PedalComp addendum. Engine-source
citations are against **ReBuzz 1835-preview**.

---

## 1. "At least one parameter is required" — the zero-param load rejection

The biggest gotcha building this machine, because nothing about it is visible
until runtime. A managed machine that declares **no** `[ParameterDecl]` property
compiles and deploys fine, then ReBuzz's scanner rejects it at load:

```
System.Exception: at least one parameter is required
   at ReBuzz.ManagedMachine.ManagedMachineDLL.LoadManagedMachine(String path)
   at ReBuzz.FileOps.MachineDLLScanner.ValidateDll(...)
```

The machine never enters the browser and — exactly like the `Invalid MaxValue` /
negative-`MinValue` traps (Core §9) — the only diagnostic is the Debug Console
(Ctrl+D); the machine list simply omits it with no popup.

Pedal OSC is conceptually parameter-free (a pass-through tap), so this bit
immediately. The fix is to declare at least one parameter; rather than a dummy it
got three that earn their place (§4).

**General rule** for any machine — control taps, do-nothing scaffolds, pure
routers included: **declare ≥1 `[ParameterDecl]`.** Promoted to **Build §11** as
a load-validation rule alongside the Core §9 declaration rules; recorded here as
the machine that surfaced it.

---

## 2. Effect classification and the pass-through Work()

Pedal OSC is an **effect** (Core §2 / Build §7.2): the signature
`bool Work(Sample[] output, Sample[] input, int n, WorkModes mode)` = EffectBlock.
It is inserted inline, conventionally **just before Master** so it taps the full
mix. Work() copies `input`→`output` sample-for-sample (transparent) while
accumulating sum-of-squares for RMS and tracking block peak.

- `input == null` when nothing upstream is connected/active — tested directly
  (not via a separate bool) so nullable flow-analysis proves `input` non-null in
  the loop below and no CS8602 is raised.
- Level is measured per **chunk** (≤256 samples, ~188 Hz — Core §34), far more
  often than the ~60 fps a renderer consumes. The receiver undersamples, so
  last-writer-wins is the correct model — no queue, no backpressure.

---

## 3. Beat and bar phase — the reason to tap inside the host

The differentiator over any external audio-listening visualizer: a generic
visualizer must *infer* tempo from a waveform; this machine **knows** it, because
ReBuzz is the sequencer. Two host sources combine.

### 3.1 `MasterInfo` — tempo and position within a tick

`host.MasterInfo` (`Buzz.MachineInterface.MasterInfo`) exposes:

| Field | Meaning |
|---|---|
| `BeatsPerMin` | tempo |
| `TicksPerBeat` | TPB |
| `SamplesPerSec` | sample rate |
| `SamplesPerTick` | `(60 * SPS) / (BPM * TPB)` |
| `PosInTick` | `[0 .. SamplesPerTick-1]` |
| `TicksPerSec`, `GrooveSize`, `PosInGroove`, `GrooveData` | groove/swing |

The interface comment is explicit that **MasterInfo and SubTickInfo are only
valid in Work and parameter setters**. ReBuzz refreshes them immediately before
each Work batch (`WorkManager` → `MachineManager.UpdateMasterAndSubTickInfoToHost()`),
and advances `PosInTick += samplesToProcess` per chunk, wrapping to 0 and
advancing the song by one tick at the boundary.

### 3.2 `PlayPosition` — which tick

`MasterInfo` gives position *within* a tick but not *which* tick. The absolute
song position in ticks is `ISong.PlayPosition`, reached from the machine as:

```
host.Machine.Graph.Buzz.Song.PlayPosition
```

**This path is audio-thread safe and cheap.** Every getter on it is a plain
field-backed read — `MachineCore.Graph`, `SongCore.Buzz`, `ReBuzzCore.Song`,
`SongCore.PlayPosition` (`get { return playPosition; }`). Four field reads and
three null checks. (The *setter* is heavy — dispatcher marshalling, next-tick
deferral — but is never touched here. Contrast `SongCore.Machines`, which runs
LINQ per call: **not** everything on this graph is audio-thread safe, so check
before adding another host read to `Work()`.)

`IBuzz.Playing` is likewise a plain field read (`get => playing;`) and gives
transport state.

### 3.3 The derivation

```csharp
float posInTick = (float)mi.PosInTick / mi.SamplesPerTick;   // 0..1 within tick

int tpb        = mi.TicksPerBeat;
int tickInBeat = ((tick % tpb) + tpb) % tpb;
beatPhase      = (tickInBeat + posInTick) / tpb;             // 0..1 within beat

int ticksPerBar = tpb * beatsPerBar;
int tickInBar   = ((tick % ticksPerBar) + ticksPerBar) % ticksPerBar;
barPhase        = (tickInBar + posInTick) / ticksPerBar;     // 0..1 within bar
```

The double-modulo is sign-safe (C# `%` preserves sign). Guard
`TicksPerBeat > 0 && SamplesPerTick > 0` before dividing.

Verified by simulation over 10.7 s at 126 BPM / TPB 4 / 256-sample chunks: phase
stays in `[0,1)`, wraps exactly once per beat, implied tempo 126.00.

**Behaviour worth knowing:** `PlayPosition` only advances while playing, so the
phase *freezes* on stop rather than free-running — correct, but it means a
consumer should treat `playing == 0` as "hold", not "reset". Bar counting is from
song start, so starting mid-song lands on whatever beat that position genuinely
is.

**Buzz has no time signature**, so beats-per-bar cannot be read from the host —
hence the `Beats/Bar` parameter (§4).

---

## 4. Parameters — Sensitivity, Smooth, Beats/Bar

All three are plain `0..127`-style ints → Byte params, chosen to sidestep every
Core §9 pitfall (`MaxValue ≤ 254` avoids the NoValue sentinel; `MinValue ≥ 0`
avoids the silent range-offset). Parameter setters and Work() **both run on the
audio thread** (CallTick, Core §8), so reading the properties in Work() needs no
synchronisation.

- **Sensitivity** (0–127, def 64 = ×1.0): scales level before send —
  `level *= Sensitivity / 64f`.
  *Known limitation:* max is ×1.98, which cannot reach full scale from a typical
  ~0.26 RMS peak. Renderer-side gain covers it today; widen the mapping (e.g.
  `/32` for ×4, or an exponential curve) when next revising, for consumers with
  no gain of their own.
- **Smooth** (0–127, def 0 = raw): block-rate one-pole on the sent level —
  `coef = 1 − (Smooth / 127) · 0.98`, then `_smoothed += coef·(level − _smoothed)`.
- **Beats/Bar** (1–16, def 4): bar length for `barPhase`; see §3.3.

---

## 5. The wire format — an OSC bundle of addressed floats

One **bundle** per send (~125/sec), schema **v1**:

| Address | Value |
|---|---|
| `/rebuzz/v` | schema version (`1`) |
| `/rebuzz/rms` | smoothed level, 0..1 |
| `/rebuzz/peak` | block peak, 0..1 |
| `/rebuzz/beat` | phase within beat, 0..1 |
| `/rebuzz/bar` | phase within bar, 0..1 |
| `/rebuzz/bpm` | tempo |
| `/rebuzz/playing` | 1 = running, 0 = stopped |
| `/rebuzz/beatsperbar` | from the Beats/Bar parameter |

**Why a bundle of per-value addresses** rather than one packed multi-float
message: addressed values are what off-the-shelf OSC tools (TouchDesigner,
Resolume, lighting desks) map to channels natively, which keeps
*adopt-a-renderer* on the table instead of locking the project into a bespoke
one; and the bundle keeps every value from the same audio block atomic, so
nothing tears across packets. Measured size 228 bytes — far inside the ~1400-byte
single-frame limit the brief calls for.

`/rebuzz/v` rides in every bundle deliberately: once the renderer lives in its
own repo, the wire format *is* the interface between them, and a receiver needs
to detect a schema change rather than silently misread floats.

### 5.1 Hand-rolled encoder — one dll, no dependency

OSC is just a wire format:

- **OSC-string**: ASCII, null-terminated, null-padded to a 4-byte boundary.
- **Message**: address OSC-string, type-tag OSC-string (`,f…`), then big-endian
  float32 args.
- **Bundle**: `"#bundle"` OSC-string, an 8-byte time tag (`1` = immediate), then
  per element a big-endian int32 length followed by the element bytes.

That's ~60 lines, so the machine **hand-rolls it** rather than pulling a NuGet OSC
package — keeping deployment to a single `.dll` (Build §1), no extra dependency
dll in `Gear\Effects`. The encoder is a helper file and needs no SDK usings
(Build §6.3) — pure BCL (`System.Buffers.Binary.BinaryPrimitives`).

Pad alignment is computed **relative to each element's own start**, not the
absolute buffer offset, so message encoding is correct both standalone and nested
inside a bundle.

Both layouts were validated byte-for-byte against a reference OSC implementation
before shipping.

---

## 6. Threading — the audio thread never does I/O

`Work()` runs on the audio thread against a ~5.33 ms/chunk budget (Core §34), most
of which is already host overhead on a busy song. A blocking `socket.Send` there
manufactures dropouts. Therefore:

- `Work()` computes the frame and publishes it. No allocation, no lock, no I/O.
- A separate **background sender thread** (`IsBackground`, ~125 Hz) copies the
  newest frame, encodes the bundle, and performs the `UdpClient.Send`.

### 6.1 The handoff — a 4-slot lock-free ring

v1.0 used a single `volatile float`, which is sufficient for one scalar. A
multi-value frame is not: independent volatile fields can tear, delivering `rms`
from one audio block and `beatPhase` from the next.

v1.1 uses a **pre-allocated ring of 4 `FeatureFrame` structs plus a volatile
`_newest` index**:

```csharp
int next = (_newest + 1) & (SlotCount - 1);
_slots[next].Rms = …;  /* … fill … */
_newest = next;          // volatile write publishes the completed slot
```

The reader does one volatile read then one struct copy. Single producer, single
consumer, no locks, no allocation. Four slots with the writer only ~1.5× faster
than the reader means a slot cannot be lapped mid-copy in practice. This is the
general pattern for **any** machine exporting data to an out-of-process consumer,
and the shape MIDI events will need when they're added.

---

## 7. Sample scale — normalise by 32768

ReBuzz samples are ±32768 float, so level is divided by 32768 to land the sent
value in ~0..1 for a shader uniform. Confirmed two ways: PedalComp §1, and
independently in the engine source — `WorkManager`'s master output stage scales
by `audioOutMul = 1 / 32768.0f`, and the wavetable mix multiplies by `32768.0f`
on the way in.

Empirically, a normally-loud mix reads **~0.25 RMS peak** — well below 1.0, which
is expected (a full-scale sine RMSes at 0.707). Consumers need gain; see §4.

---

## 8. Build setup and the two-namespace using footgun

csproj follows the house template (Build §1.2): `net10.0-windows`; references
`ReBuzz.dll` + `BuzzGUI.Interfaces.dll` (both `Private=false`, **not**
`BuzzGUI.Common`); deploys via `OutputPath = $(BuzzDir)\Gear\Effects` with
`AppendTargetFrameworkToOutputPath` / `AppendRuntimeIdentifierToOutputPath`
false; `DebugType none` / `DebugSymbols false` / `GenerateDependencyFile false` /
`NoWarn MSB3277`. (`UseWPF` is not required — this machine touches no WPF types.)

Bring-up cost three build round-trips, each a documented footgun re-learned:

1. **Placeholder HintPaths** (`..\lib\…`) → MSB3245, reference not found. Fix:
   point at the ReBuzz install (`$(BuzzDir)\…`), or copy the `<Reference>` block
   from a machine that already builds.
2. **`using BuzzGUI.Interfaces;` alone** → CS0246 on every SDK type. The authoring
   types (`IBuzzMachine`, `IBuzzMachineHost`, `MachineDecl`, `ParameterDecl`,
   `Sample`, `WorkModes`, `MasterInfo`) live in **`Buzz.MachineInterface`**;
   `BuzzGUI.Interfaces` carries the host-graph types (`IMachine`, `IBuzz`,
   `ISong`). Both usings, every time — Build §6.2, and §6.4 names this exact
   mistake. v1.1 needs both for real, since it walks `.Graph.Buzz.Song`.
3. **`Global` did not resolve** under either SDK using. Now answered: `Global` is
   in **`BuzzGUI.Common`** (`BuzzGUI.Common/Global.cs`) — a *third* assembly the
   house csproj template does not reference. Pedal OSC does not need it (§9), but
   any machine that does must add the `BuzzGUI.Common` reference and using.
   Recorded in Build §6.1.

The three MSB3277 version-conflict warnings (WindowsBase / Microsoft.VisualBasic
/ System.Drawing) come from `ReBuzz.dll`'s transitive WPF closure and are benign
(Build §4) — silenced by `NoWarn`.

---

## 9. Teardown — `IDisposable` is the correct hook (confirmed)

The machine starts its sender thread in the constructor and must stop it on
removal. **`IDisposable` is sufficient and correct**, confirmed in the engine
source:

```
MachineManager.DeleteMachine(machine)
  → managedMachines[machine].Release()
      → if (machine is IDisposable) ((IDisposable)machine).Dispose();
```

`ManagedMachineHost.Release()` disposes the machine explicitly, with a source
comment stating that CLR/disposable machines must have `Dispose()` called to free
resources. So `Dispose()` sets `_running = false`, joins the sender (200 ms) and
closes the socket; `IsBackground` remains a backstop for process exit.

No `Global.Buzz.Song.MachineRemoved` subscription is needed — which is fortunate,
since `Global` is not in a referenced assembly (§8.3). **The v1.0 "production
TODO" on this point is closed.**

---

## Depends on

- **Build** §1.2 (mandatory csproj properties), §1.3 (deploy → `Gear\Effects`;
  here via `OutputPath` rather than a Copy target), §2 (`.NET` AssemblyName →
  browser name **and** managed-loader routing), §4 (`NoWarn MSB3277`), §6.1–§6.2
  & §6.4 (both SDK usings; `Global` in `BuzzGUI.Common`), §7.2 (EffectBlock
  `Work` signature), §10 (commit conventions), **§11 (load validation — the
  ≥1-parameter rule this machine surfaced)**.
- **Core** §2 (effect classification), §8 (CallTick — parameter setters on the
  audio thread), §9 (`ParameterDecl` type inference + `MinValue`/`MaxValue`
  rules), §11 (`MachineDecl`), §34 (audio-thread chunking / per-chunk `Work`
  budget — the reason all I/O is off-thread).
- **PedalComp** §1 (±32768 sample scale).
- Companion (non-notes): `ReBuzz_Video_Bridge_Brief` (pipeline design),
  `pedal_osc_HANDOFF.md` (project state and next actions).

Built and run against ReBuzz **1835-preview**; engine-source citations are
against the same. v1.1.
