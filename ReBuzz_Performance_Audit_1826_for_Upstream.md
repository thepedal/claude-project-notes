# ReBuzz 1826-preview — Per-chunk audio-thread overhead audit

**Purpose.** Source-level findings on the ReBuzz audio path, anchored
to the Pedal Profiler2 (PP2) dropout investigation of May 2026. Each
finding is paired with a concrete code location, an estimate of how
it contributes to per-chunk host overhead, and a fix recommendation
sized for incremental adoption.

**Audience.** ReBuzz maintainers / contributors.

**Sources.** Code references are against the ReBuzz 1826-preview
source tree. Empirical measurements are from PP2 v1.7.3 — see
`ReBuzz_PedalProfiler2_State.md` §7 (dropout investigation) and
`ReBuzz_ManagedMachine_Notes_Core.md` §§34–40 (host-internals
findings surfaced by PP2's reflection diagnostics).

---

## Executive summary

The PP2 Buffer Sweep diagnostic established that, at 48 kHz with
any ASIO buffer size from 256 to 2048 samples:

- Internal chunk size stays at ≤256 samples (driven by
  `SubTickSize = 260` on `IBuzz` and `WorkManager.cs:samplesToProcess
  = min(remaining/2, 256)`)
- Per-chunk budget therefore stays at ~5.33 ms
- **`Unaccounted` (budget − sum of per-machine engine counts) sits
  at ~5 ms, ~99% of the budget**, on songs ranging from empty +
  stopped to fully populated
- Dropout rate ~11.5% with ~41 ms peaks (9 × 4.69 ms — characteristic
  of ASIO skipping ~9 buffers to resync after a chunk overrun)

That ~5 ms of `Unaccounted` is per-chunk host overhead — bookkeeping,
scheduling, parameter dispatch, IPC marshalling, allocation — running
between or around the per-machine work that `MachinePerformanceData`
can attribute. Because chunks run back-to-back when the ASIO buffer
is large (2048-sample buffer = 8 back-to-back chunks), the overhead
sums into a real wall-clock cost; because each individual chunk has
no margin, **any small disturbance pushes one chunk over and ASIO
compensates with skips**.

This audit identifies code-level contributors to that ~5 ms. Two
upstream paths reduce the problem:

1. **Reduce per-chunk overhead** so each chunk holds more margin.
   This is what the eighteen findings below target. Most are
   isolated, low-risk, and individually small but cumulatively
   significant.
2. **Allow larger chunks when tick-aligned dispatch isn't needed.**
   Out of scope for a code-level audit; flagged here because it's the
   only structural escape from the per-chunk-overhead floor. Not
   addressed by the findings below.

Findings are ordered by per-chunk overhead reduction. A short
"top picks" list precedes the detailed sections.

---

## Top picks — the seven most fixable, highest-impact items

If only a handful land, these:

1. **§1** — Replace `ConcurrentDictionary<int,int>` in `ParameterCore`
   with `int[]`. Removes per-machine-per-chunk dictionary churn from
   the parameter-clear loop. Mechanical, isolated to one file.
2. **§2** — Preallocate the native IPC send buffer, drop the
   `List<byte> + .ToArray()` pattern. Removes the largest single
   source of audio-thread GC pressure for sessions with native
   machines. One file (`NativeMessage.cs`).
3. **§10** — Make `ReBuzzCore.MasterTapSamples` allocation-free and
   conditional on a live subscription. ~750 KB/sec of Gen0 garbage
   currently allocated per chunk even with no MasterTap consumer.
4. **§14** — Skip `UpdateMasterAndSubTickInfoToHost` when nothing
   changed. Dirty-flag the source structs; most chunks become a
   no-op.
5. **§11a** — Cache the master reference. `UpdateMasterParams`,
   `ReadWork`, and several other sites call
   `MachinesList.FirstOrDefault(m => m.DLL.Info.Type == Master)`
   every chunk. Cache once at construction.
6. **§15** — Pre-size and reuse `workTasks`; replace
   `Dictionary<MachineCore, bool>` with `HashSet<MachineCore>`. Drops
   the per-chunk array allocation in the work scheduler.
7. **§12** — Replace `DateTime.UtcNow.Ticks` with
   `Stopwatch.GetTimestamp()` for per-block timing. Internal
   consistency with `MachinePerformanceData.PerformanceCount` (which
   already uses QPC — see Core §35) and removes a kernel-mediated
   time read from every machine's Work bracket.

All seven are mechanical refactors confined to single files or single
methods. Total LOC change estimated at ~300 lines. No public API
surface changes. No threading-model changes.

The remaining findings are smaller paper cuts (still per-chunk
overhead) or per-sample optimisations (improve CPU under load but
don't directly add chunk-budget margin).

The single biggest architectural lever — splitting `ReBuzzCore.AudioLock`
into finer-grained locks (§4) — is included but **deferred to a
"longer-term" section** because the blast radius is large.

---

## Findings ordered by per-chunk overhead reduction

### §1 — `ConcurrentDictionary<int,int>` for parameter pvalues

**File.** `Core/ParameterCore.cs:181–182`.

```csharp
ConcurrentDictionary<int, int> values  = new ConcurrentDictionary<int, int>();
readonly ConcurrentDictionary<int, int> pvalues = new ConcurrentDictionary<int, int>();
// (Author's own comment immediately follows: "Is ConcurrentDictionary
//  needed. Adds locks and latency?")
```

**Per-chunk cost.** Every machine that ticks runs the clear loop in
`MachineWorkInstance.Tick()` (lines 600–622):

```csharp
foreach (var p in Machine.ParameterGroupsList[0].ParametersList)
    for (int i = 0; i < Machine.ParameterGroupsList[0].TrackCount; i++)
        p.SetPValue(i, p.NoValue);
foreach (var p in Machine.ParameterGroupsList[1].ParametersList)
    p.SetPValue(0, p.NoValue);
foreach (var p in Machine.ParameterGroupsList[2].ParametersList)
    for (int i = 0; i < Machine.TrackCount; i++)
        p.SetPValue(i, p.NoValue);
```

Each `SetPValue` enters the `ConcurrentDictionary` write path
(striped-lock acquire, hash, bucket walk, write, release). For a
typical machine with ~10 parameters per group across 3 groups and 8
tracks, that's ~250 dictionary writes per machine per chunk. Across
30 machines, ~7500 dictionary writes per chunk.

Native machines additionally hit the dictionary on the read side via
`AudioMessage.AudioTick` (`NativeMachine/AudioMessage.cs:80–134`),
which reads `GetPValue` for every parameter, every track, to assemble
the IPC payload.

**Why this access pattern doesn't need a dictionary.** Tracks are
dense small integer indices (0..N-1, N typically ≤16, max 256).
Access is per-track-sequential. The audio thread is the only writer
during Work. A plain `int[]` indexed by `track` is ~10–50× faster
and removes the lock entirely.

**Recommended fix.**

```csharp
// in ParameterCore
int[] pvalues;       // sized to TrackCount, resized on track-count change
int[] values;
public int GetPValue(int track) => pvalues[track];
public void SetPValue(int track, int val) => pvalues[track] = val;

// in MachineWorkInstance.Tick — clearing all tracks at once
Array.Fill(p.pvalues, p.NoValue);  // single call replaces TrackCount calls
```

Track-count changes (`SetMachineTrackCount`) are already serialized
through `workLock`, so the array can be resized safely there.

**Risk.** Low. ConcurrentDictionary's value semantics for missing
keys (returns `default(int) = 0` on `TryGetValue` failure path) are
not the same as `int[]` (which throws on out-of-range), but every
access in the codebase is already guarded by `track < TrackCount` or
goes through the parameter group's known shape. A `Debug.Assert`
during transition would catch any straggler.

**Magnitude.** Likely the single largest fix in the audit measured
in per-chunk µs saved. Hits both machine types.

---

### §2 — Native IPC send buffer allocates per call

**File.** `NativeMachine/NativeMessage.cs` (declaration line 68; send
path line 148).

```csharp
readonly List<byte> sendMessageData = new List<byte>();

// ... per call, after building up the payload via SetMessageData<T>:
fixed (byte* source = sendMessageData.ToArray())
{
    Unsafe.CopyBlock(dataRootPointer, source, (uint)size);
}
sendMessageData.RemoveRange(0, MessageBuffer.MaxSize);
```

Plus `receaveMessageTable = msg.ReceaveMessageData.ToArray();` on
the reply side (line 219).

**Per-chunk cost.** Every native-machine `AudioWork` / `AudioMultiWork`
/ `AudioTick` call allocates two byte arrays — one on send (size of
the entire payload, ~1–2 KB for a stereo sample block) and one on
receive. With N native machines and ~187 chunks/sec at 48k/256, that's
~750·N allocations/sec.

Underneath that, `SetMessageData<T>` (line 1307) and the per-sample
`SetMessageData(Sample[], int, bool)` (line 1348) build up
`sendMessageData` via `AddRange(byte[N])` calls. For a stereo 256-
sample block, that's 512 `AddRange` calls of 4-byte arrays into a
`List<byte>` — each one going through `ICollection<T>.CopyTo`.

**Recommended fix.** Replace `List<byte> sendMessageData` with a
preallocated `byte[] sendBuffer` (sized to `MessageBuffer.MaxSize`)
and an `int sendOffset`:

```csharp
byte[] sendBuffer = new byte[MessageBuffer.MaxSize];
int sendOffset;

public unsafe void SetMessageData<T>(T data) where T : unmanaged
{
    Unsafe.WriteUnaligned(ref sendBuffer[sendOffset], data);
    sendOffset += sizeof(T);
}

public unsafe void SetMessageData(Sample[] data, int nSamples, bool stereo)
{
    int bytes = nSamples * (stereo ? 8 : 4);
    fixed (Sample* src = data)
    fixed (byte* dst = &sendBuffer[sendOffset])
        Buffer.MemoryCopy(src, dst, bytes, bytes);
    sendOffset += bytes;
}

// On send:
fixed (byte* source = sendBuffer)
    Unsafe.CopyBlock(dataRootPointer, source, (uint)sendOffset);
sendOffset = 0;
```

Same pattern for the receive side — expose
`MessageContent.ReceaveMessageData` as a `Span<byte>` or hold its
backing array directly.

**Risk.** Low. The interface (`SetMessageData<T>`, etc.) is unchanged
from callers' perspective. The shared-memory `MessageBuffer.MaxSize`
constraint is already enforced.

**Magnitude.** Largest single GC-pressure reduction in the audit
when native machines are present.

---

### §3 — `AudioTick` builds throwaway `List<byte>` per native tick

**File.** `NativeMachine/AudioMessage.cs:80–134`.

```csharp
List<byte> globalVals = new List<byte>();
List<byte> trackVals  = new List<byte>();
foreach (var p in machine.ParameterGroupsList[1].ParametersList) {
    ...
    if (typeSize == 1)      globalVals.Add((byte)pValue);
    else if (typeSize == 2) globalVals.AddRange(GetBytes((ushort)pValue));
}
for (int i = 0; i < machine.TrackCount; i++)
    foreach (var p in machine.ParameterGroupsList[2].ParametersList) {
        ...
        trackVals.AddRange(GetBytes((ushort)pValue));
    }

SetMessageData(globalVals.ToArray());
SetMessageData(trackVals.ToArray());
```

Two `List<byte>` allocations per tick per native machine, plus two
`ToArray()` copies. Compounds with §1 (the `GetPValue` reads).

**Recommended fix.** Once §1 and §2 land, the payload layout for a
machine is fully determined by `(ParameterGroupsList[1], ParameterGroupsList[2],
TrackCount, typeSize)` — cache an "encode plan" once at machine init
time, then `AudioTick` becomes a direct write into the
preallocated send buffer with no intermediate lists.

**Risk.** Low; touches one method.

**Magnitude.** Compounds with §1 and §2. Native-only.

---

### §4 — Single `AudioLock` wrapping all of `ThreadRead`

**File.** `Audio/WorkManager.cs:113`.

```csharp
public int ThreadRead(float[] buffer, int offset, int count)
{
    lock (ReBuzzCore.AudioLock)
    {
        // entire DSP graph traversal: parameter dispatch, ticks,
        // per-machine work, IPC, master mix
    }
}
```

`AudioLock` is held for the duration of every chunk and is also taken
by **54 other sites** across the codebase (`grep -rn "lock (AudioLock)"
ReBuzz/`): file save/load, machine add/remove, connect/disconnect,
MIDI engine, wave operations, UI message handlers.

**Per-chunk cost.** Lock acquisition itself is cheap when uncontended
(a few ns), so on idle this isn't the source of the 5 ms. But the
moment any UI or file operation contends the lock, the audio thread
sleeps until release — and a single UI-side `lock (AudioLock)`
holding for >5 ms is a guaranteed chunk overrun under the current
budget. This is the architectural reason chunk sizes can't safely go
much below 256 samples regardless of what other fixes land.

**Recommended fix.** Out of scope for this audit's "eminently
fixable" framing — listed in the "longer-term" section at the end.
Sketch: split `AudioLock` into (a) a graph-topology read/write lock
where audio takes read and mutators take write, and (b) parameter
changes routed through an SPSC queue drained at the top of
`ThreadRead`.

**Magnitude.** Bounds achievable minimum buffer size more than
anything else. But the cost is in worst-case spike behaviour, not
average chunk overhead — the per-chunk fixes above address the
average; this addresses the variance.

---

### §5 — Per-task closure allocations in multithreaded work

**File.** `Audio/WorkManager.cs:644, 689, 724`.

```csharp
foreach (var machine in workList.Keys.OrderByDescending(m => m.performanceLastCount))
{
    var t = AudioEngine.TaskFactoryAudio.StartNew(() =>
    {
        if (machine.workDone) return;
        var workInstance = buzzCore.MachineManager.GetMachineWorkInstance(machine);
        workInstance.TickAndWork(workSamplesCount, true);
        machine.workDone = true;
    });
    workTasks.Add(t);
}
Task.WaitAll(workTasks.ToArray());
```

Three per-chunk overhead sources in one block:

- `OrderByDescending` allocates an enumerator + sort buffer per call
- Lambda captures `machine`, `workSamplesCount`, `buzzCore`,
  `engineSettings` — compiler generates a display class per call site
  allocated per `StartNew`
- `Task.WaitAll(workTasks.ToArray())` — `ToArray()` allocates a
  fresh array of `Task` refs every chunk

For 30 machines per chunk at 187 chunks/sec, ~5600 display-class
allocations/sec, ~5600 `Task` allocations/sec, plus the LINQ
overhead.

**Recommended fix.** Pre-sized worker pool with reused work-item
struct array. Each worker claims items via `Interlocked.Increment`
on a shared index. Algorithm 2 in `WorkThreadEngine.cs` is closer to
this shape and could become the only multi-threaded path (with the
Task-based variants deleted).

Less-invasive interim fix: replace `OrderByDescending` with an
in-place insertion sort on a reused `MachineCore[]` scratch (the
work list is small); replace the lambda body with a method call that
takes a struct parameter (eliminates display-class capture); replace
`workTasks.ToArray()` with a pre-sized, reused `Task[]` field.

**Risk.** Medium for the full pool rework; low for the interim
fixes. The interim path is recommended for the first pass.

**Magnitude.** Significant per-chunk GC pressure reduction in
multithreaded mode. In single-threaded mode (`Multithreading` engine
setting off), this code path isn't hit and the finding doesn't
apply.

---

### §10 — `MasterTapSamples` allocates per chunk even with no subscriber

**File.** `Core/ReBuzzCore.cs:1655`.

```csharp
internal void MasterTapSamples(float[] resSamples, int offset, int count)
{
    var s = GetSongTime();
    float scale = 1.0f / 32768.0f;
    for (int i = 0; i < count; i += 2) {
        maxSampleLeft  = Math.Max(maxSampleLeft,  Math.Abs(resSamples[offset+i]));
        maxSampleRight = Math.Max(maxSampleRight, Math.Abs(resSamples[offset+i+1]));
    }
    maxSampleLeft  *= scale;
    maxSampleRight *= scale;

    float[] samples = new float[count];        // unconditional allocation
    for (int i = 0; i < count; i++)
        samples[i] = resSamples[offset + i];

    dispatcher.BeginInvoke(() => {
        if (MasterTap != null)
            MasterTap.Invoke(samples, true, s);
    });
}
```

**Per-chunk cost.** ~4 KB allocation per chunk for the copy, plus the
peak-scan loop, plus a dispatcher queue item — **all of this runs
unconditionally**. The `if (MasterTap != null)` check is inside the
dispatcher callback, too late to elide the allocation. ~750 KB/sec
of Gen0 garbage at 48k/256, every session, whether or not anything
is listening.

Also: `maxSampleLeft` / `maxSampleRight` are never reset between
chunks — consumers reading them as block-level peak meters get
since-process-start peaks growing monotonically (correctness item §B).

**Recommended fix.**

```csharp
internal void MasterTapSamples(float[] resSamples, int offset, int count)
{
    if (MasterTap == null) return;             // early-out

    var s = GetSongTime();
    // ... peak scan ...

    // Ring of preallocated buffers, rotated per call so the dispatcher
    // callback gets a stable buffer that won't be overwritten before
    // it's consumed.
    float[] samples = tapRing[tapRingPos];
    tapRingPos = (tapRingPos + 1) & (TapRingLen - 1);
    Array.Copy(resSamples, offset, samples, 0, count);

    dispatcher.BeginInvoke(() => MasterTap?.Invoke(samples, true, s));
}
```

`TapRingLen = 4` is plenty; the dispatcher typically drains within
one or two chunks. PP2 v1.7.3's master VU consumer reads
opportunistically (`volatile float[] _latestSamples`) so the
ring-overwrite semantics are fine — see Core §37.2 for the audio-
thread contract PP2 follows.

Same treatment for `MachineConnectionCore.DoTap`
(`MachineConnectionCore.cs:99`) — also allocates per call when
subscribed.

**Risk.** Low. Public `MasterTap` event signature unchanged.
Consumers must already tolerate the audio thread's contract (no
mutation expected).

**Magnitude.** Significant — removes a guaranteed 4 KB/chunk
allocation that exists in every session.

---

### §11 — LINQ patterns on per-chunk hot paths

These are individually small but they're all on the per-chunk loop.
Listed together because the fixes are identical in shape.

**§11a — Master lookup via `FirstOrDefault` (per chunk).**

`Audio/WorkManager.cs:254` and `:576`:

```csharp
var master = buzzCore.SongCore.MachinesList.FirstOrDefault(
    m => m.DLL.Info.Type == MachineType.Master);
```

Linear scan + closure allocation per chunk. **Fix:** cache the
master reference once at construction (the reference is already
fetched at `WorkManager.cs:67` and discarded — just store it in a
field).

**§11b — `GetMachineWorkInstance` double-dictionary lookup.**

`MachineManagement/MachineManager.cs:1018`:

```csharp
if (!workInstances.ContainsKey(machine)) { ... }
else { return workInstances[machine]; }
```

Two hashes per lookup, called multiple times per chunk per machine.
**Fix:** `TryGetValue`.

**§11c — `HandleWorkRecursive` `OrderByDescending`.**

`Audio/WorkManager.cs:716`. Same fix as §5.

**§11d — `PlayPatternColumnEvents` quadratic enumeration.**

`Audio/WorkManager.cs:519`:

```csharp
for (int j = 0; j < ces.Count(); j++) {
    var pe = ces.ElementAt(j);
    ...
}
```

`Count()` and `ElementAt(j)` on a non-`IList` enumerable enumerate
from the start each iteration — O(N²) per column per chunk. **Fix:**
materialize once or use `foreach`.

**§11e — `UpdatePatternPositions` clears every pattern every chunk.**

Per machine × per pattern × per chunk, setting `PlayPosition = int.
MinValue` and then conditionally setting it again. **Fix:** filter to
patterns that are actually playing, or defer the reset to once-per-
tick.

**§11f — `CallTick` twin iteration.**

`Audio/WorkManager.cs:420` iterates all machines twice (once for
control, once for non-control). **Fix:** cache the partition (it
only changes on machine add/remove).

**Magnitude.** Each individually µs-scale. Combined, a measurable
chunk of the per-chunk fixed overhead — particularly §11a, which
runs even on empty songs (the master is the only machine present).

---

### §14 — `UpdateMasterAndSubTickInfoToHost` walks every managed host every chunk

**File.** `MachineManagement/MachineManager.cs:978`.

```csharp
internal void UpdateMasterAndSubTickInfoToHost()
{
    foreach (var machineHost in managedMachines.Values) {
        if (machineHost != null)
            UpdateMasterAndSubTickInfo(machineHost);
    }
}
```

Called from `WorkManager.ThreadRead` (~line 167) **every chunk**. Copies 10
fields × N managed machines, regardless of whether `masterInfo` /
`subTickInfo` actually changed since last chunk. Tempo doesn't change
between adjacent chunks in normal playback; this is propagating
identical values.

**Recommended fix.** Dirty-flag the source structs. Set the flag on
BPM / sample rate / sub-tick boundary transitions; clear on
propagation. Most chunks become a no-op.

Better long-term: change the managed-host `MasterInfo` /
`SubTickInfo` properties to be read-only views onto
`ReBuzzCore.masterInfo` / `subTickInfo` directly, removing the
per-chunk copy. This is a public-API change (the
`IBuzzMachineHost.MasterInfo` type), so it's a heavier lift than the
dirty-flag option.

**Risk.** Low for the dirty-flag fix. Medium for the view change.

**Magnitude.** Pure per-chunk fixed overhead — runs every chunk even
on empty songs (other than the Master itself). One of the
contributors to the empty-song-still-drops result from PP2's
step-4 elimination.

---

### §15 — `workTasks` and `workList` allocations

**File.** `Audio/WorkManager.cs:884` and surrounding.

```csharp
Task.WaitAll(workTasks);                          // OK
// but earlier:
var workTasks = new List<Task>();                 // per call
```

And `workList` is a `Dictionary<MachineCore, bool>` used as a set —
the value is always `true`.

**Recommended fix.** Hoist `workTasks` to a reused `List<Task>`
field; clear at start of chunk. Replace `workList` with
`HashSet<MachineCore>`. The dead `tickTasks` field at line 418 can
be removed.

**Risk.** Trivial. Same logic, different storage.

**Magnitude.** Small per chunk but pure overhead reduction.

---

### §17 — `EffectBlock` / `GeneratorBlock` Work always copies input and output (managed-only)

**File.** `ManagedMachine/ManagedMachineHost.cs:440–480`.

```csharp
// For effect blocks (current code):
for (int j = 0; j < n; j++) inputBuffer[j] = samples[j];   // unconditional
bool flag2 = EffectBlockWork(outputBuffer, inputBuffer, n, mode);
if (flag2 && (mode & WorkModes.WM_WRITE) != 0)
    for (int k = 0; k < n; k++) samples[k] = outputBuffer[k];
```

**Per-chunk cost.** Two 256-sample struct-copy loops per managed
effect-block machine per chunk. RyuJIT may vectorize the loops, but
`Array.Copy` is the explicit fast path. The bigger issue is that the
input copy is unconditional — for `WM_WRITE`-only modes, the input is
meaningless.

**Recommended fix.**

```csharp
if ((mode & WorkModes.WM_READ) != 0)
    Array.Copy(samples, inputBuffer, n);
bool flag2 = EffectBlockWork(outputBuffer, inputBuffer, n, mode);
if (flag2 && (mode & WorkModes.WM_WRITE) != 0)
    Array.Copy(outputBuffer, samples, n);
```

`GeneratorBlock` already skips the input copy. Could go further and
pass `samples` directly as the output buffer when the contract
permits (eliminates the output copy too) — that's an API contract
question.

**Risk.** Low for the mode check; check the documented contract of
`EffectBlockWork` before eliding the output copy.

**Magnitude.** Modest per-chunk savings per managed effect-block
machine. Adds up across 6+ such machines.

---

### §13 — `Tick` resets every pvalue every chunk, not just touched ones

**File.** `MachineManagement/MachineWorkInstance.cs:602–622`.

After the `parametersChanged` loop drains, the method unconditionally
walks every parameter of all three groups and sets every (param, track)
pvalue to `NoValue`. Coupled with §1's dictionary cost, this is the
bulk of `Tick`'s cost.

**Recommended fix.** Track exactly which (param, track) pairs were
touched (the `parametersChanged` dictionary already does this) and
reset only those:

```csharp
foreach (var kv in Machine.parametersChanged)
    kv.Key.SetPValue(kv.Value, kv.Key.NoValue);
```

**Risk.** Low. Same data flows, smaller scope.

**Magnitude.** Compounds with §1. The "reset everything" pass exists
as defensive coding against external pvalue writes that didn't go
through `parametersChanged`; if that's a real concern, gate the full
reset behind a debug flag.

---

### §12 — `DateTime.UtcNow.Ticks` for per-chunk timing

**File.** `Audio/WorkManager.cs:115, 248` and
`MachineManagement/MachineWorkInstance.cs:45, 78`.

```csharp
long time = DateTime.UtcNow.Ticks;
// ... work ...
buzzCore.PerformanceCurrent.EnginePerformanceCount += (DateTime.UtcNow.Ticks - time);
```

`DateTime.UtcNow` on modern Windows is implemented on top of
`GetSystemTimePreciseAsFileTime`, which involves a system page
transition. `Stopwatch.GetTimestamp()` is a much cheaper
`QueryPerformanceCounter` call.

**Internal consistency note.** Per Core §35, `MachinePerformanceData.
PerformanceCount` is already in `Stopwatch.Frequency` ticks (QPC).
So part of ReBuzz's per-machine perf tracking uses the fast path
while the engine's own per-chunk timing uses the slow one. Aligning
on `Stopwatch.GetTimestamp()` everywhere removes the inconsistency
and the cost.

**Recommended fix.**

```csharp
long time = Stopwatch.GetTimestamp();
// ...
buzzCore.PerformanceCurrent.EnginePerformanceCount += (Stopwatch.GetTimestamp() - time);
```

Convert ticks → time only at display.

**Risk.** None. Same semantics; lower cost.

**Magnitude.** Small per call but called 2N times per chunk (N
machines). Worth it for the consistency alone.

---

### §6 — `MachineCore.GetActivity` scans full 256 samples regardless of `nSamples`

**File.** `Core/MachineCore.cs:1346`.

```csharp
public bool GetActivity()
{
    bool isActive = false;
    float maxSample = 0.0f;
    var connections = Name == "Master" ? Inputs : Outputs;
    foreach (var output in connections) {
        Sample[] samples = (output as MachineConnectionCore).Buffer;
        for (int i = 0; i < samples.Length; i++) {
            maxSample = Math.Max(Math.Abs(samples[i].L), maxSample);
            maxSample = Math.Max(Math.Abs(samples[i].R), maxSample);
        }
    }
    return maxSample > AMP_EPS;
}
```

Called from `MachineWorkInstance.TickAndWork` and again from
`HandleWorkRecursive` as a "double check" pass. Three problems:

1. Scans `samples.Length` (always 256) rather than the actual
   `nSamples` for the chunk
2. `Math.Max(Math.Abs(...))` dependency chain; SIMD batch would be
   4–8× faster
3. `AMP_EPS = 1.0f` is essentially "any DC residual = active" at
   ±32768 internal scale (see Core §38). Combined with the "double
   check" pass, well-behaved machines that report their own activity
   correctly still get scanned.

**Recommended fix.** Scan `nSamples` not `.Length`. Use a
`Vector<float>` abs-max for the inner loop. Gate the "double check"
pass on a per-machine flag — defensive only against legacy native
plugins.

**Risk.** Low.

**Magnitude.** Per machine per chunk. Hits both machine types.

---

### §7 — `GetStereoSamples` and `GetMultiIOSamples` clear full 256 regardless of `nSamples`

**File.** `Core/MachineCore.cs:1244, 1278`.

```csharp
readonly Sample[] stereoSamples = new Sample[256];
internal Sample[] GetStereoSamples(int nSamples)
{
    for (int i = 0; i < stereoSamples.Length; i++)   // always 256
        stereoSamples[i].L = stereoSamples[i].R = 0;
    // ... sum inputs for 0..nSamples
}
```

**Recommended fix.** `Array.Clear(stereoSamples, 0, nSamples)`. Same
fix for `GetMultiIOSamples`. Also for any other site that loops over
`.Length` of a fixed-256 buffer instead of `nSamples`.

**Magnitude.** Small per call but the `Sample` struct is 8 bytes —
clearing 256 vs 32 (at short block sizes) is a real cache-traffic
delta.

---

### §8 — `FlushDenormalToZero` is mostly a no-op

**File.** `Common/Utils.cs:331–351`.

```csharp
public static void FlushDenormalToZero(Sample[] samples)
{
    /* commented out:
    for (int i = 0; i < samples.Length; i++) {
        samples[i].L += k_DENORMAL_DC;
        samples[i].R += k_DENORMAL_DC;
    }
    */
}

public static float FlushDenormalToZero(float f)
{
    f += k_DENORMAL_DC;
    //f -= k_DENORMAL_DC;
    return f;
}
```

The array form is completely commented out. The scalar form adds
`1e-15f` without subtracting it, so the master mix is permanently
DC-offset by ~1e-15 (inaudible but funny) and provides no actual
denormal protection. Called per-sample in the master mix loop.

This is per-sample work — doesn't directly attack per-chunk overhead
— but it's a free fix because the calls accomplish nothing.

**Recommended fix.** Two options:

a) **Delete all the calls.** They achieve nothing. Denormal
   protection then has to live inside machines (where Pedal builds
   already do it).

b) **Set MXCSR DAZ+FTZ once on the audio thread at startup** —
   `Sse.SetFlushToZeroMode` and `Sse3.SetDenormalsAreZerosMode`. This
   genuinely flushes denormals. It is per-thread and affects any
   plugins running on that thread; some hosts deliberately don't do
   it for this reason. Most pro audio hosts do.

**Risk.** Low for (a). Medium for (b) — affects any hosted DSP code.

**Magnitude.** Cleanup, plus genuine denormal protection that
currently doesn't exist.

---

### §9 — `MachineConnectionCore.UpdateBuffer` per-sample modulo, broken amp interpolation

**File.** `Core/MachineConnectionCore.cs:122`.

```csharp
float ampStart   = Utils.FlushDenormalToZero(interpolatorAmp.Value / 0x4000);
float ampCurrent = (int)interpolatorAmp.Tick();
float ampStep    = Utils.FlushDenormalToZero(((ampStart - ampCurrent / 0x4000) / 0x4000) / nSamples);

for (int i = 0; i < nSamples; i++) {
    latencyBuffer[latencyBufferWritePos].L = samples[i].L * ampStart * panL;
    latencyBuffer[latencyBufferWritePos].R = samples[i].R * ampStart * panR;
    latencyBufferWritePos = (latencyBufferWritePos + 1) % latencyBuffer.Length;
    ampStart += ampStep;
}

for (int i = 0; i < nSamples; i++) {
    buffer[i].L = latencyBuffer[latencyBufferReadPos].L;
    buffer[i].R = latencyBuffer[latencyBufferReadPos].R;
    latencyBufferReadPos = (latencyBufferReadPos + 1) % latencyBuffer.Length;
}
```

Three issues:

1. **`%` per sample** — integer-divide on x86-64 is ~20–80 cycles; a
   bitmask wrap (`pos & (len-1)`) is one cycle when `len` is a power
   of two, which the default 256 is.
2. **`ampStep` math evaluates to near-zero** — both `ampStart` and
   `ampCurrent / 0x4000` are normalized 0..1 floats, and the
   difference is then divided by `0x4000` (16384) again and by
   `nSamples`. Net step ~`delta / 4M`. The interpolation is dead
   code; amp changes land as instantaneous jumps. Audible as clicks
   on slider moves. (Tracked as correctness item §A.)
3. **Two-pass write-then-read** when `addedLatency == 0` — the
   latency ring serves no purpose and the data is touched twice
   through cache.

**Recommended fix.**

```csharp
// Fast path: addedLatency == 0 (the common case)
if (addedLatency == 0) {
    for (int i = 0; i < nSamples; i++) {
        buffer[i].L = samples[i].L * ampStart * panL;
        buffer[i].R = samples[i].R * ampStart * panR;
        ampStart += ampStep;
    }
} else {
    int mask = latencyBuffer.Length - 1;   // require power-of-two
    // ... same loop but with bitmask wrap
}
```

And fix the `ampStep` math: drive the interpolator per-sample over
`nSamples` ticks, or compute a proper linear ramp from the previous
chunk's end-value to the current target.

**Risk.** Low for the fast path. Medium for the amp math fix —
needs a quick audition to confirm no regression.

**Magnitude.** Per-sample work per connection per chunk. Doesn't
directly attack the empty-song overhead but matters as soon as
there are connections carrying signal.

---

## Smaller items / paper cuts

### §16 — Reflection in `ManagedMachineHost.Commands` and `MachineState`

`ManagedMachine/ManagedMachineHost.cs:263, 317, 353`. `GetProperty` +
`GetGetMethod` + `Invoke` on every access. Not audio-thread (UI menu
and save-load paths), but the same `Delegate.CreateDelegate` pattern
already used for the Work / Tick bindings (line 181+) would
eliminate the reflection cost. Recommended if `ManagedMachineHost`
is being touched anyway.

### §18 — Oversample path uses Hermite resampler

`MachineWorkInstance.cs:488`. For ×2/×4 integer ratios, half-band
FIRs are dramatically more efficient and have better stopband than
the generic Hermite interpolator used for both up- and down-sample.
Per-sample work, only relevant when oversample is on. Also worth
noting that the managed-machine path doesn't honour
`oversampleFactorOnTick` at all (`MachineWorkInstance.
WorkMachineManaged` line 226) — managed machines self-oversample if
they want to. This is an asymmetry, intentional or not, worth
documenting.

---

## Correctness items (not performance but spotted along the way)

### §A — Connection amp interpolation step evaluates to ~zero

See §9 point 2. Connection amp changes click instead of smoothing.

### §B — Master tap peak meters never reset

`Core/ReBuzzCore.cs:1655`. `maxSampleLeft` / `maxSampleRight`
accumulate since process start.

### §C — `RefreshMachineParams` track-index in wrong scope

`MachineManager.cs:1307`. `track++` is outside the inner `foreach` but
inside the outer — `track` only ever takes value 0 inside the
`SetPValue` call. Native-only (iterates `nativeMachines` only).
Whether per-track refresh on tempo change is even needed is a
separate question (see gear.xml `ForceParamRefreshOnTempoChangeEnabled`),
but the variable scope looks unambiguously buggy.

### §D — `UpdateMasterParams` calls `RefreshMachineParams` over-eagerly

`Audio/WorkManager.cs:254`. Triggered inside the BPM/TPB change
branch even when only one of the two changed. Cosmetic; flagged for
completeness.

### §E — `MaxEngineLockTime = 0` dead assignment every chunk

`MachineWorkInstance.cs:80`. Set to 0 every Work call, never set
elsewhere. (Note: PP2's reflection dump confirms
`MachinePerformanceData.MaxEngineLockTime` is in the struct but
remains zero in all observed sessions — see Core §35.)

### §F — ASIO `audioInBuffer` size vs `BlockCopy` bound

`Audio/AudioEngine.cs:140`. Partial-buffer guards not visible in the
audit's snippet. Tagged for review rather than confirmed bug.

---

## Longer-term: `AudioLock` split (§4 detail)

`ReBuzzCore.AudioLock` is the single biggest determinant of how short
a buffer ReBuzz can safely use. Split sketch:

1. **Graph topology** — read/write lock. Audio thread takes read;
   topology mutators (add/remove machine, connect/disconnect) take
   write. Most of the 54 sites currently grabbing `AudioLock` are
   topology operations and only need this lock.
2. **Parameter changes from UI** — SPSC queue drained at the top of
   `ThreadRead`. Removes `lock(machine.workLock)` from the
   `ParameterCore.SetValue` audio-thread contention surface.
3. **Engine state changes** (BPM, sample rate, settings) — these
   already need full audio-thread quiescence; keep a dedicated mutex
   for them.

This is a larger change than the seven top picks, with corresponding
blast radius. Recommended only after the smaller items have shipped
and the average-case chunk overhead is reduced — at which point the
remaining work is primarily about worst-case variance, which is what
the lock split addresses.

---

## Mapping back to the PP2 empirical findings

| PP2 observation                                     | Audit findings most directly relevant |
|-----------------------------------------------------|---------------------------------------|
| `Unaccounted` ≈ 99% on empty song                   | §10, §14, §11a, §15, §12 — pure per-chunk overhead unrelated to machine work |
| `Unaccounted` scales with managed machine count      | §1, §13, §14, §17 — overhead per managed machine per chunk |
| `Unaccounted` scales with native machine count       | §1, §2, §3, §13 — IPC marshalling + dictionary cost per native machine |
| 41 ms peak spikes from single-chunk overruns         | §4 (AudioLock contention is the most likely spike source); §5, §11c (sched-side spikes in MT mode) |
| Sub-256-sample chunk size driven by `SubTickSize=260`| Out of scope (architectural); audit fixes provide headroom *within* the chunk size |

---

## Suggested order of attack

1. **§1** ConcurrentDictionary → int[]   *(parameter store)*
2. **§10** MasterTapSamples allocation   *(reuse buffers, early-out)*
3. **§14** UpdateMasterAndSubTickInfoToHost   *(dirty flag)*
4. **§11a** Cache master reference   *(field replaces FirstOrDefault)*
5. **§15** Hoist `workTasks`, drop `Dictionary<,bool>`   *(reuse collections)*
6. **§12** DateTime.UtcNow.Ticks → Stopwatch.GetTimestamp   *(consistency)*
7. **§2** Native IPC send buffer preallocation   *(NativeMessage.cs)*
8. **§3** AudioTick encode plan   *(after §1 and §2)*
9. **§13** Tick clears only touched pvalues   *(after §1)*
10. **§17** EffectBlock mode-aware copy elision   *(managed-only)*
11. **§7** Clear nSamples, not .Length   *(MachineCore)*
12. **§6** GetActivity scan nSamples + SIMD   *(MachineCore)*
13. **§11b–f** Remaining LINQ paper cuts
14. **§5** Multithreaded scheduler interim fixes (full pool rework deferred)
15. **§8** Denormal handling: delete no-op calls, optionally set MXCSR
16. **§9** Connection buffer fast path + amp ramp fix   *(also addresses §A)*

Then correctness items §A–§F, then the §4 lock split as the final
larger lift.

---

## Files referenced

- `Audio/WorkManager.cs` — top-of-chunk overhead concentrates here
- `Core/ParameterCore.cs` — pvalues store (§1)
- `Core/MachineCore.cs` — buffer handling, activity scan (§6, §7)
- `Core/MachineConnectionCore.cs` — per-connection per-sample work
  (§9)
- `Core/ReBuzzCore.cs` — master tap (§10), AudioLock declaration
- `MachineManagement/MachineManager.cs` — per-host propagation (§14)
- `MachineManagement/MachineWorkInstance.cs` — per-machine work
  bracket (§12, §13)
- `ManagedMachine/ManagedMachineHost.cs` — managed Work entry, reflection
  (§16, §17)
- `NativeMachine/NativeMessage.cs` — IPC payload buffer (§2)
- `NativeMachine/AudioMessage.cs` — audio IPC messages (§3)
- `Common/Utils.cs` — denormal helpers (§8)

## Related documents (in this project)

- `ReBuzz_PedalProfiler2_State.md` — the PP2 dropout investigation,
  including the Buffer Sweep table that is the headline empirical
  evidence.
- `ReBuzz_ManagedMachine_Notes_Core.md` §§34–40 — host-internals
  documentation surfaced by PP2's reflection diagnostics.
- `ReBuzz_Performance_Audit_1826.md` — the original exploratory
  audit. This document supersedes it for upstream-submission
  purposes; the original retains value as the detailed working
  record.
- `ReBuzz_Performance_Audit_1826_by_MachineType.md` — the
  managed-vs-native partitioning view. Useful for assessing which
  fixes matter to which session configurations; this document folds
  in the salient bits.
