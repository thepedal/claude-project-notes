# ReBuzz Performance Audit — 1826-preview, by machine-type impact

Same eighteen performance findings and six correctness notes as
`ReBuzz_Performance_Audit_1826.md`, reorganized by **which kind of
machine they affect**:

- **Native-only** — fixes that matter when there's at least one native
  (C++ DLL, IPC-hosted) machine in the song. Land none of these and
  managed-only sessions see no change.
- **Managed-only** — fixes inside the managed-machine host path. Pure
  native-machine sessions see no change.
- **Both** — fixes in the shared engine plumbing (work scheduler,
  parameter store, connection buffers, master mix, denormal handling).
  These move the needle regardless of what's plugged in.

The numbering matches the original audit (§1–§18, A–F) for
cross-reference. Code samples, full fix recipes, and the
suggested-order-of-attack are in that file; this document is the
partitioning view.

---

## Summary table

| §  | Finding                                                        | Affects                  |
|----|----------------------------------------------------------------|--------------------------|
| 1  | `ConcurrentDictionary` for `pvalues`/`values` in `ParameterCore` | **Both** (native-skewed) |
| 2  | Native IPC payload — `List<byte>` + `ToArray()` per send       | **Native only**          |
| 3  | `AudioTick` builds throwaway `List<byte>`                      | **Native only**          |
| 4  | Single `AudioLock` wrapping all of `ThreadRead`                | **Both**                 |
| 5  | Per-task closure allocations in multithreaded work             | **Both**                 |
| 6  | `MachineCore.GetActivity` full 256-sample scan                 | **Both**                 |
| 7  | `GetStereoSamples` / `GetMultiIOSamples` clear full 256        | **Both**                 |
| 8  | `FlushDenormalToZero` is mostly a no-op                        | **Both**                 |
| 9  | `MachineConnectionCore.UpdateBuffer` — modulo, broken interp   | **Both**                 |
| 10 | `DoTap` / `MasterTapSamples` per-block allocations             | **Both**                 |
| 11 | LINQ + double dictionary lookups on hot paths                  | **Both**                 |
| 12 | `DateTime.UtcNow.Ticks` instead of `Stopwatch`                 | **Both**                 |
| 13 | `Tick` resets every pvalue every block, not just touched ones  | **Both** (native-skewed) |
| 14 | `UpdateMasterAndSubTickInfoToHost` propagates to every managed host | **Managed only**     |
| 15 | `workTasks.ToArray()` per call; `workList` as `Dictionary<,bool>` | **Both**              |
| 16 | Reflection in `ManagedMachineHost.Commands`/`MachineState`     | **Managed only**         |
| 17 | `EffectBlock`/`GeneratorBlock` work always copy in *and* out   | **Managed only**         |
| 18 | Oversample uses Hermite resampler                              | **Native only**          |
| A  | Connection amp step math evaluates to ~zero                    | **Both**                 |
| B  | Master tap peak meters never reset                             | **Both**                 |
| C  | `RefreshMachineParams` track-index in wrong scope              | **Native only**          |
| D  | `UpdateMasterParams` triggers `RefreshMachineParams` over-eagerly | **Native only**       |
| E  | `MaxEngineLockTime = 0` dead assignment every block            | **Both**                 |
| F  | ASIO `audioInBuffer` size vs `BlockCopy` bound                 | **Both** (input path)    |

Counts: **5 native-only**, **3 managed-only**, **16 both**.

Net read: the heavy hitters split fairly cleanly. The largest single
*native-only* costs (IPC payload marshalling, AudioTick list-building)
are the most localized fixes — both inside `NativeMessage.cs` /
`AudioMessage.cs`, no API surface change. The largest *managed-only*
finding (§17) is a few lines in `ManagedMachineHost.Work`. Everything
else is shared engine code, where the gains apply universally but the
risk surface is correspondingly larger.

---

## Native-only findings

### §2 — Native IPC payload assembly allocates per send

`NativeMachine/NativeMessage.cs:148, 219` and the whole
`SetMessageData<T>` family at lines 1307+. Every call to
`AudioMessage.AudioWork` / `AudioMultiWork` / `AudioTick` /
`AudioBeginFrame` etc. builds its payload into a `List<byte>` (via
hundreds of small `AddRange(byte[4])` calls), then does
`sendMessageData.ToArray()` — a fresh ~1–2 KB allocation per send,
per call. On the receive side `msg.ReceaveMessageData.ToArray()` does
it again.

The per-sample stereo loop in `SetMessageData(Sample[], int, bool)`
(line 1348) is the worst-shaped contributor: 512 `AddRange` calls per
256-sample stereo block. Replace with a preallocated `byte[]
sendBuffer` + `int sendOffset` and write directly via
`Unsafe.WriteUnaligned`; the sample-array path becomes one
`Buffer.MemoryCopy` call.

**Why native-only.** Managed machines never enter `NativeMessage`. The
managed-machine path in `MachineWorkInstance.WorkMachineManaged` calls
`machineHost.Work(samples, ...)` directly — a virtual call to a delegate
that was wired up via `Delegate.CreateDelegate` at construction. No
IPC, no marshalling, no `List<byte>`.

**Magnitude.** Largest single source of audio-thread GC pressure when
native machines are in the song. Easily 3+ MB/sec of Gen0 garbage in a
modest 8-native-machine session.

### §3 — `AudioTick` builds throwaway `List<byte>` per native tick

`NativeMachine/AudioMessage.cs:80–134`. Two `List<byte>` allocations
per tick per native machine, plus two `ToArray()` copies to hand off to
`SetMessageData`. Driven by `GetPValue` reads against the
`ConcurrentDictionary` (§1).

After §1 and §2 land, this whole function becomes "write parameter
bytes directly into the sendBuffer", with the encode layout cached at
machine init time from the parameter group / type-size table.

**Why native-only.** Managed machines are ticked through
`ManagedMachineHost.Tick(machine)` (`MachineManager.cs:600`), which
invokes the machine's own `Tick()` method through a cached delegate.
Parameters arrive at the managed machine via per-parameter setter
calls (`SetNote`, `SetChord`, etc.) wired through
`MachineParameter.Delegates`. There is no payload-building step.

### §18 — Oversample path uses a Hermite resampler not tuned for integer ratios

`MachineWorkInstance.cs:488` (`Oversample`), driven from
`ProcessWork`/`ProcessMultiWork`. Up- and down-sample both go through
`WorkManager.SpeedDown`, a generic Hermite interpolator. For ×2/×4
integer ratios, half-band FIRs are dramatically more efficient and
have far better stopband.

**Why native-only.** The managed path in
`MachineWorkInstance.WorkMachineManaged` (line 226) doesn't check
`Machine.oversampleFactorOnTick` at all. Managed machines are not
oversampled by the host — if a managed machine wants oversampling, it
does it internally (as Pedal Plaits does). Only the native path
`ProcessWork` / `ProcessMultiWork` honours `oversampleFactorOnTick`.

This is also worth flagging because it's an **asymmetry in the host
behaviour** worth knowing about: per-machine oversample setting is a
no-op for managed machines and silently does nothing. Whether that's
intentional or a missing feature is a design call.

### Correctness C — `RefreshMachineParams` track-index in wrong scope

`MachineManager.cs:1307`. Iterates `nativeMachines.Values` only;
managed machines never get refreshed on tempo change. (Whether they
need to is a separate question — the docstring says "Force param
refresh on machines defined in gear.xml", and gear.xml's
`ForceParamRefreshOnTempoChangeEnabled` flag was specifically for old
native machines that latched their tempo-derived state at tick time
and didn't recompute when BPM changed.)

The actual `track++` scope bug (track only ever takes value 0 inside
the inner foreach) is therefore a native-only correctness issue.

### Correctness D — `UpdateMasterParams` calls `RefreshMachineParams` over-eagerly

`WorkManager.cs:254`. Called from inside the BPM/TPB change branch.
Native-only because the target is `RefreshMachineParams` (which
iterates `nativeMachines` only — see C).

---

## Managed-only findings

### §14 — `UpdateMasterAndSubTickInfoToHost` walks every managed host every block

`MachineManager.cs:978`:

```csharp
internal void UpdateMasterAndSubTickInfoToHost()
{
    foreach (var machineHost in managedMachines.Values)
    {
        if (machineHost != null)
            UpdateMasterAndSubTickInfo(machineHost);
    }
}
```

Called from `WorkManager.ThreadRead` (~line 167) **every block**.
Copies 10 fields × N managed machines, regardless of whether the
source `masterInfo`/`subTickInfo` actually changed since last block.

**Why managed-only.** This is the managed-machine equivalent of the
`WriteMasterInfo`/`WriteSubTickInfo` calls embedded in the native IPC
payload (the natives get their copy as part of `AudioTick` /
`AudioBeginFrame`). Managed machines read from their per-host
`MasterInfo`/`SubTickInfo` properties — those instances live on the
`ManagedMachineHost` object, and the propagation function above is
what keeps them current.

**Fix.** Either dirty-flag the source structs and propagate on change
only (most blocks become a no-op), or — better — make the managed-host
`MasterInfo` property a read-only view onto `ReBuzzCore.masterInfo`
directly, removing the per-block copy entirely. The latter is a public-
surface change (the `IBuzzMachineHost.MasterInfo` type), so it's a
heavier lift than the dirty-flag option.

For session-time relevance: with ~30 managed machines this is small
(a few µs per block) but it's pure waste; the master tempo doesn't
actually change between two consecutive audio blocks except on the
literal block where the user moves the BPM slider.

### §16 — Reflection-based access in `Commands` / `MachineState` properties

`ManagedMachineHost.cs:263, 317, 353`. Each property fetch does:

```csharp
PropertyInfo property = dll.machineType.GetProperty("Commands");
if (property == null) return null;
MethodInfo getMethod = property.GetGetMethod();
...
object obj = getMethod.Invoke(machine, null);
```

`GetProperty` and `GetGetMethod` are reflection lookups (string-keyed
hash on the type's member table); `Invoke` is the slow reflection
invocation path. Both are avoidable: the rest of `ManagedMachineHost`
(line ~180 onward) caches every method binding once at construction
via `Delegate.CreateDelegate`. `Commands`, `MachineState` (get **and**
set), and a handful of other property paths missed the same treatment.

**Why managed-only.** This is entirely inside `ManagedMachineHost`.
Native machines route property-ish information through other IPC
messages and the reflection cost doesn't apply.

**Hot-path relevance.** Not audio-thread. `Commands` is per right-click
menu open; `MachineState` is per save/load. So this isn't a real-time
audio fix — it's UI responsiveness and song-load time. Still worth
folding in if `ManagedMachineHost` is being touched anyway.

### §17 — `EffectBlock` / `GeneratorBlock` work copies input and output unconditionally

`ManagedMachineHost.cs:440–480`. For an effect block:

```csharp
if ((mode & WorkModes.WM_READ) != 0)
    for (int j = 0; j < n; j++) inputBuffer[j] = samples[j];

bool flag2 = EffectBlockWork(outputBuffer, inputBuffer, n, mode);

if (flag2 && (mode & WorkModes.WM_WRITE) != 0)
    for (int k = 0; k < n; k++) samples[k] = outputBuffer[k];
```

Two hand-rolled per-sample copy loops (struct copies — `Sample` is 8
bytes). RyuJIT will probably vectorize them, but `Array.Copy` /
`Span<T>.CopyTo` is explicit and uses a vectorized intrinsic
unconditionally. The bigger issue is **whether the in-copy and
out-copy are necessary at all**:

- **Input copy** — protects against the user's machine mutating the
  caller's input array. For pure-generator block mode (`WM_WRITE`
  only, no `WM_READ`), the input is meaningless and the copy is pure
  waste. The current code does avoid the input copy for the generator
  block path, **but the effect block path always reads and copies in
  regardless of mode**.
- **Output copy** — only needed if the user's machine writes to
  `outputBuffer` rather than to `samples` directly. Could pass
  `samples` as the output buffer when the mode permits, dropping the
  copy entirely. This is an API contract question (does
  `EffectBlockWork(outputBuffer, inputBuffer, ...)` documented contract
  promise that outputBuffer is a private buffer?), so worth confirming
  before changing.

**Why managed-only.** This is the per-block work invocation for
managed machines specifically. Native machines have their own copying
problem — see §2 — but it's structurally different (IPC marshalling,
not in-process struct copies).

**Magnitude.** Per managed effect-block machine per audio block, 2×
512-byte struct copies for a 256-sample stereo block. Across, say, 6
managed effect-block machines, that's ~6 KB of memcpy work per block.
Not enormous, but Array.Copy is "free" by comparison and the
mode-aware elision is just correctness.

---

## Both — engine-shared findings

These are listed in the same order as the original audit for direct
cross-reference. Where one machine type is hit harder than the other,
that's noted in the body.

### §1 — `ConcurrentDictionary` for `pvalues`/`values` (Both, native-skewed)

`ParameterCore.cs:181–182`. The author's own comment "Is
ConcurrentDictionary needed. Adds locks and latency?" answers itself.
`SetPValue` / `GetPValue` on the audio thread go through a striped-
lock hash table when an `int[track]` indexed by `track` would do.

**Why both.** Two hot loops hit pvalues:

- `MachineWorkInstance.Tick()` lines ~600–622 iterate every parameter
  of every group and call `SetPValue(track, NoValue)` for every track,
  for **every machine** (managed or native) that ticks this block.
  This is the bulk of the cost and applies equally to both.
- `AudioMessage.AudioTick` (§3) reads `GetPValue` for every parameter
  for every track on the native IPC payload assembly path. Managed
  machines don't have an analogous read loop — managed parameters
  flow through cached setter delegates wired up at construction.

So both pay the **write** cost (the Tick clear-to-NoValue loop) but
**only native pays the additional read cost**. A native-heavy session
will see a larger win from this fix than a managed-only session, but
both gain.

### §4 — Single `AudioLock` wrapping all of `ThreadRead` (Both)

`WorkManager.cs:113`. The entire DSP graph traversal runs inside
`lock (ReBuzzCore.AudioLock)`. The same lock is grabbed by 54 other
sites — file save/load, machine add/remove, connect/disconnect, MIDI
engine, wave operations, UI message handlers. Any of those that fires
serializes against the audio block.

**Why both.** This is the engine's outermost concurrency primitive.
Native vs managed doesn't enter into it — both machine types' Work
calls happen inside this lock. The split-the-lock recommendation in
the original audit applies to both worlds equally.

This is also the architectural item that limits how short the audio
buffer can safely be set, irrespective of machine choice.

### §5 — Per-task closure allocations in multithreaded work (Both)

`WorkManager.cs:644, 689, 724`. Every per-block multithreaded work
dispatch creates a fresh `Task` plus a compiler-generated display
class for the captured `machine`/`workSamplesCount`/`buzzCore`
locals. With 30 machines per block at 187 blocks/sec, that's 5600+
allocations/sec each of Task and display class, before counting
`OrderByDescending` LINQ allocations and `workTasks.ToArray()`.

**Why both.** The scheduler doesn't discriminate. Every machine that
gets scheduled (managed or native) goes through `TaskFactoryAudio.
StartNew(() => ...)`. Managed and native sessions equally suffer.

### §6 — `MachineCore.GetActivity` scans full 256 samples (Both)

`MachineCore.cs:1346`. Three problems: scans `samples.Length` (always
256) instead of `nSamples`, uses `Math.Max(Math.Abs(...))` instead of
SIMD, and the `AMP_EPS = 1.0f` threshold is essentially "any DC
residual = active" given Buzz's internal ±32768 scale.

**Why both.** `GetActivity` runs on every machine's outputs (or
master's inputs) regardless of machine type. The "double-check" pass
in `HandleWorkRecursive` runs it again on the same buffers.

### §7 — `GetStereoSamples`/`GetMultiIOSamples` clear full 256 (Both)

`MachineCore.cs:1244, 1278`. Clears `stereoSamples.Length` (always
256) instead of `nSamples`. Same buffer-size confusion as §6. Hits
every machine that has inputs.

**Why both.** These are machine-type-agnostic — they prepare input
buffers before the per-type Work dispatch.

### §8 — `FlushDenormalToZero` is mostly a no-op (Both)

`Common/Utils.cs:331–351`. The `Sample[]` overload is commented out
entirely; the scalar overload adds `1e-15f` without subtracting it.
Called per-output and per-sample at the master mix. Per-sample wrap
cost without any actual denormal protection.

**Why both.** Master-mix wrap (`WorkManager.cs:231, 598–599`) runs
universally. The per-output `UpdateOutputs` calls run after every
machine's work. Neither path discriminates by machine type.

Recommendation in the original audit was to set MXCSR DAZ+FTZ once on
the audio thread and delete the wrapper calls. That removes the cost
universally and provides actual denormal protection — equally helpful
to all machines hosted by ReBuzz.

### §9 — `MachineConnectionCore.UpdateBuffer` (Both)

`MachineConnectionCore.cs:122`. Modulo per sample for ring wrap;
amp-interpolation step math evaluates to ~zero (correctness item A);
no-op flush at the end. Two-pass write-then-read despite latency
buffer not actually being used when `addedLatency == 0`.

**Why both.** Every connection in the graph runs through
`UpdateBuffer` regardless of what's at either end. Managed source
into managed sink, native source into native sink, mixed — all the
same code path.

This is one of the highest-frequency per-sample loops in the codebase
(runs for every connection every block), so a fast path for the
common `addedLatency == 0` case is worth its weight.

### §10 — `DoTap` and `MasterTapSamples` allocate per call (Both)

`MachineConnectionCore.cs:99` and `Core/ReBuzzCore.cs:1655`. Tap
allocates `new float[nSamples * 2]` per call when subscribed; master
tap allocates `new float[count]` per block **even when not
subscribed**. Peak meters never reset (correctness item B).

**Why both.** Connection-level taps run on every connection's output,
master-tap runs at the master. Neither cares about source-machine
type.

### §11 — LINQ and double-dictionary lookups on hot paths (Both)

`UpdateMasterParams` `FirstOrDefault` (line 254), `GetMachineWorkInstance`
double lookup (line 1018), `CallTick` twin iteration, `HandleWorkRecursive`
`OrderByDescending`, `PlayPatternColumnEvents` `ElementAt` (O(N²)),
`UpdatePatternPositions` clearing positions on every pattern in the
song every block.

**Why both.** All of these are engine-side iteration patterns over
the machine list or pattern data. Machine-type-agnostic.

### §12 — `DateTime.UtcNow.Ticks` for per-block timing (Both)

`WorkManager.cs:115, 248` and `MachineWorkInstance.cs:45, 78`.
`DateTime.UtcNow` involves a kernel-mediated time read on Windows;
`Stopwatch.GetTimestamp()` is a much cheaper `QueryPerformanceCounter`
call and is the standard idiom for performance measurement.

**Why both.** `MachineWorkInstance` wraps every machine Work call
(managed or native) in `DateTime.UtcNow.Ticks` brackets.

### §13 — `Tick` resets every pvalue every block (Both, native-skewed)

`MachineWorkInstance.cs:602–622`. Even after `parametersChanged` is
drained, the method unconditionally walks every parameter of all
three groups and sets every (param, track) pvalue to `NoValue`. Same
"hits both machine types but native pays additional reads" story as
§1.

**Why both.** The reset loop runs for every machine that ticks. The
relevance for native machines compounds via §3 (AudioTick reads
pvalues immediately afterward to assemble the IPC payload), so native
sessions feel the §1+§13 combination more sharply.

### §15 — `workTasks.ToArray()` per call; `workList` as `Dictionary<,bool>` (Both)

`WorkManager.cs:884`. Pre-sized array reused across blocks would
eliminate the per-block allocation; `HashSet<MachineCore>` or
`List<MachineCore>` would drop the unused bool value. Dead
`tickTasks` field at line 418.

**Why both.** Scheduler-side; agnostic.

### Correctness A — Connection amp step evaluates to ~zero (Both)

`MachineConnectionCore.cs:129`. The `ampStep = ((ampStart -
ampCurrent / 0x4000) / 0x4000) / nSamples` math divides by 16384 once
too many times. Connection amp changes land as instantaneous jumps
rather than smoothed ramps — audible clicks on slider drag.

**Why both.** Every connection; same code path regardless of source
type.

### Correctness B — Master tap peak meters never reset (Both)

`ReBuzzCore.cs:1655`. `maxSampleLeft` / `maxSampleRight` accumulate
since process start. Anyone reading them as block-level peaks gets
all-time peak, monotonically.

### Correctness E — `MaxEngineLockTime = 0` dead assignment (Both)

`MachineWorkInstance.cs:80`. Set to 0 every Work call, never set
anywhere else.

### Correctness F — ASIO `audioInBuffer` `BlockCopy` bound (Both, input path)

`AudioEngine.cs:140`. Worth a review — partial-buffer guards not
visible in the original audit's snippet. Tag for inspection.

---

## Practical guidance

### "I only run managed machines"

Skip §2, §3, §18, correctness C and D. Everything else still helps.
Your biggest wins are still §4 (`AudioLock`), §1 (pvalues array), §10
(tap allocations), §8 (MXCSR), §9 (connection buffer), in roughly
that order by effort-adjusted impact — plus §17 (managed-only effect
block double-copy elision) as a low-risk bonus pickup.

The managed-machine-specific items (§14, §16, §17) are individually
small and the audit's headline numbers don't really lean on them.

### "I only run native machines"

Skip §14, §16, §17. Everything else applies. Your biggest wins are
§2 + §3 (native IPC payload — single biggest GC source in your
configuration), §1 (pvalues, hit twice as hard for natives), §4
(`AudioLock`), §10 (tap), §8 (MXCSR), §13 (Tick clear).

### "I run both"

Everything in the audit applies. The first audit's value-per-effort
ranking holds without modification.

### Risk profile by partition

- **Native-only fixes** are localized to two files (`NativeMessage.cs`,
  `AudioMessage.cs`) and one method block in `MachineManager.cs`.
  Lowest blast radius if something regresses — only native-machine
  sessions are affected. Good candidates for "ship first, get
  confidence".
- **Managed-only fixes** are in `ManagedMachineHost.cs` and
  `MachineManager.UpdateMasterAndSubTickInfo*`. Also localized;
  managed-machine API surface needs care on §14 and §17. §16 is pure
  refactor (cache the reflection, identical observable behaviour).
- **Both fixes** are in the shared engine. Higher reach, higher
  reward, higher risk. §4 (`AudioLock` split) is the largest blast
  radius in the entire audit — affects every machine, every test
  song, every API consumer.

---

## Audit cross-reference

Original audit (full code samples and fix recipes):
`ReBuzz_Performance_Audit_1826.md`. Section numbers (§1–§18, A–F)
are identical between the two documents.
