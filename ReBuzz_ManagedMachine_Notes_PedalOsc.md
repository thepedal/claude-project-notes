# ReBuzz Managed Machine Notes — Pedal OSC

Source: Pedal OSC v1.0 build (2026-07-18) — an **effect**-class "tap" machine
that passes audio through unchanged, measures per-block RMS, and streams it as an
OSC/UDP message to an external renderer. It is **Phase 1 of the ReBuzz→video
bridge** (see `ReBuzz_Video_Bridge_Brief`): the ReBuzz side of an audio-reactive
visuals pipeline where numbers leave the host over OSC and drive shader uniforms
in a separate OpenGL process.

Sections numbered locally. `Core §N` → `..._Notes_Core.md`; `Build §N` →
`..._Notes_Build.md`; `PedalComp §N` → the PedalComp addendum.

---

## 1. "At least one parameter is required" — the zero-param load rejection

The single biggest gotcha building this machine, because nothing about it is
visible until runtime. A managed machine that declares **no** `[ParameterDecl]`
property compiles and deploys fine, then ReBuzz's scanner rejects it at load:

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
got two that earn their place (§4).

**General rule** for any machine — control taps, do-nothing scaffolds, pure
routers included: **declare ≥1 `[ParameterDecl]`.** This belongs in Build notes
as a load-validation rule alongside the Core §9 `MinValue`/`MaxValue` rules;
recorded here as the machine that surfaced it.

---

## 2. Effect classification and the pass-through Work()

Pedal OSC is an **effect** (Core §2 / Build §7.2): the signature
`bool Work(Sample[] output, Sample[] input, int n, WorkModes mode)` = EffectBlock.
It is inserted inline, conventionally **just before Master** so it taps the full
mix. Work() copies `input`→`output` sample-for-sample (transparent), accumulating
sum-of-squares to derive per-block RMS across both channels.

- `input == null` when nothing upstream is connected/active — tested directly
  (not via a separate bool) so nullable flow-analysis proves `input` non-null in
  the loop below and no CS8602 is raised.
- RMS is measured per **chunk** (≤256 samples, ~188 Hz — Core §34), far more often
  than the ~60 fps a renderer consumes. The receiver undersamples, so
  last-writer-wins is the correct model — no queue, no backpressure.

---

## 3. All network I/O off the audio thread (the load-bearing rule)

Work() runs on the audio thread against a ~5.33 ms/chunk budget (Core §34), most
of which is already host overhead on a busy song. A blocking `socket.Send` there
manufactures dropouts. Therefore:

- Work() only writes a `volatile float _latestRms` — no allocation, no lock, no
  I/O.
- A separate **background sender thread** (`IsBackground`, ~125 Hz) reads that
  field, encodes one OSC message, and performs the `UdpClient.Send`.

For a single scalar a `volatile float` is sufficient (last-writer-wins). A
multi-value feature frame later swaps it for a lock-free SPSC ring — same "Work
writes, sender reads" shape. This is the general pattern for **any** machine that
exports data to an out-of-process consumer.

---

## 4. Parameters — Sensitivity, Smooth

Both are plain `0..127` ints → Byte params, chosen to sidestep every Core §9
pitfall (`MaxValue ≤ 254` avoids the NoValue sentinel; `MinValue ≥ 0` avoids the
silent range-offset). Parameter setters and Work() **both run on the audio
thread** (CallTick, Core §8), so reading the properties in Work() needs no
synchronisation.

- **Sensitivity** (DefValue 64 = ×1.0): scales RMS before send —
  `rms *= Sensitivity / 64f`. Dials the reactive range to the shader without
  touching audio.
- **Smooth** (DefValue 0 = raw): block-rate one-pole on the sent value —
  `coef = 1 − (Smooth / 127) · 0.98`, then `_smoothed += coef·(rms − _smoothed)`.
  Takes the jitter out of the visual when wanted.

For the first bring-up, leave Smooth 0 / Sensitivity 64 to observe the raw RMS.

---

## 5. Hand-rolled OSC encoder — one dll, no dependency

OSC is just a wire format: an address OSC-string (ASCII, null-terminated, padded
to a 4-byte boundary), a type-tag OSC-string (`,f…`), then big-endian float32
args. Encoding one message is ~30 lines, so the machine **hand-rolls it** rather
than pulling a NuGet OSC package — keeping deployment to a single `.dll` (Build
§1), with no extra dependency dll landing in `Gear\Effects`. `EncodeFloat` covers
the spike; `EncodeFloats(ReadOnlySpan<float>)` is already there for the
multi-value frame. The encoder is a helper file and needs no SDK usings (Build
§6.3) — pure BCL (`System.Buffers.Binary.BinaryPrimitives`).

Wire default: address `/rebuzz/rms`, one float, UDP `127.0.0.1:9000` (loopback
for the spike; a two-machine LAN hop is a later phase — see the brief).

---

## 6. Sample scale — normalise by 32768

ReBuzz samples are ±32768 float (PedalComp §1), so RMS is divided by 32768 to
land the sent value in ~0..1 for a shader uniform. **Confirm empirically** on
first use (log a peak with a hot signal): if peaks sit near 1.0 rather than
~32768, that build is the ±1.0 domain and the `SampleScale` constant becomes 1f.

---

## 7. Build setup and the two-namespace using footgun

csproj follows the house template (Build §1.2): `net10.0-windows`; references
`ReBuzz.dll` + `BuzzGUI.Interfaces.dll` (both `Private=false`, **not**
`BuzzGUI.Common`); deploys via `OutputPath = $(BuzzDir)\Gear\Effects` with
`AppendTargetFrameworkToOutputPath` / `AppendRuntimeIdentifierToOutputPath`
false; `DebugType none` / `DebugSymbols false` / `GenerateDependencyFile false` /
`NoWarn MSB3277`. (`UseWPF` is not required for this machine's code — it uses no
WPF types — though Build §1.2 lists it as the house default; add it to align.)

Bring-up cost three round-trips, each a documented footgun re-learned:

1. **Placeholder HintPaths** (`..\lib\…`) → MSB3245, reference not found. Fix:
   point at the ReBuzz install (`$(BuzzDir)\…`), or copy the `<Reference>` block
   from a machine that already builds.
2. **`using BuzzGUI.Interfaces;` alone** → CS0246 on every SDK type. The authoring
   types (`IBuzzMachine`, `IBuzzMachineHost`, `MachineDecl`, `ParameterDecl`,
   `Sample`, `WorkModes`) live in **`Buzz.MachineInterface`**; `BuzzGUI.Interfaces`
   carries only the host-graph types (`IMachine`, `IBuzz`, …). Both usings, every
   time — Build §6.2, and §6.4 names this exact mistake.
3. **`Global` did not resolve** under either SDK using → teardown reworked off it
   (§8). Pedal Chorus, the house reference effect, doesn't use `Global`, so there
   was no known-good import to copy.

The three MSB3277 version-conflict warnings (WindowsBase / Microsoft.VisualBasic
/ System.Drawing) come from `ReBuzz.dll`'s transitive WPF closure and are benign
(Build §4) — silenced by `NoWarn`.

---

## 8. Teardown — best-effort IDisposable

The machine starts its background sender thread in the ctor.
`Global.Buzz.Song.MachineRemoved` (the Build §8 MultiIO-example teardown hook)
would not resolve here (§7.3), so teardown is via `IDisposable`: if ReBuzz calls
`Dispose()` on removal, the thread stops and the socket closes; if not,
`IsBackground` lets the thread die with the process. Cost: deleting the machine
**mid-session** leaves the sender looping on a frozen value until ReBuzz closes —
acceptable for the spike.

**PRODUCTION TODO:** confirm the correct machine-removed hook (or whether ReBuzz
calls `Dispose()` on managed machines at all) and wire it so a mid-session delete
stops the sender immediately. Resolve where `Global` actually lives while doing
so, and back-fill §7.3.

---

## Depends on

- **Build** §1.2 (mandatory csproj properties), §1.3 (deploy → `Gear\Effects`;
  here via `OutputPath` rather than a Copy target), §2 (`.NET` AssemblyName →
  browser name **and** managed-loader routing), §4 (`NoWarn MSB3277`), §6.2 &
  §6.4 (both SDK usings; the `BuzzGUI.Interfaces`-alone footgun), §7.2
  (EffectBlock `Work` signature).
- **Core** §2 (effect classification), §8 (CallTick — parameter setters on the
  audio thread), §9 (`ParameterDecl` type inference + `MinValue`/`MaxValue`
  rules — **and the new "≥1 parameter required" load rule, §1 above**), §11
  (`MachineDecl`), §34 (audio-thread chunking / per-chunk `Work` budget — the
  reason all I/O is off-thread).
- **PedalComp** §1 (±32768 sample scale).
- Companion (non-notes): `ReBuzz_Video_Bridge_Brief` (pipeline design). The
  Phase-1 `python-osc` receiver + `moderngl` one-uniform shader are the external
  renderer side of this machine.

Built on ReBuzz **&lt;stamp the build tested&gt;**. v1.0.
