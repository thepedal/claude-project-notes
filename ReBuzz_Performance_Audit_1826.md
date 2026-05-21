# ReBuzz Performance Audit — 1826-preview

Source: ReBuzz 1826-preview source tree. Focus: the audio path
from `WorkManager.ThreadRead` down through `MachineWorkInstance`,
`ManagedMachineHost` / `NativeMachine` IPC, and the connection /
buffer plumbing in `MachineCore` and `MachineConnectionCore`.

Findings are ordered by expected impact, **not** by file. Each item
notes the file/line and what makes it costly. Items marked **HOT**
run on every audio block (call rate = `sampleRate / blockSize` —
~187 Hz at 48 kHz / 256-sample blocks); items marked **VERY HOT**
run per sample inside the audio block.

A separate "audio-correctness" section at the end flags a few things
that look like bugs rather than performance issues but were spotted
along the way.

---

## 1. ConcurrentDictionary for per-track pvalues — almost certainly the #1 win

`ReBuzz/Core/ParameterCore.cs:181–182`:

```csharp
ConcurrentDictionary<int, int> values  = new ConcurrentDictionary<int, int>();
readonly ConcurrentDictionary<int, int> pvalues = new ConcurrentDictionary<int, int>();
```

The file even contains the comment "Is ConcurrentDictionary needed.
Adds locks and latency?" — the answer is no, and yes.

**Where it bites.** `SetPValue` / `GetPValue` are called from the
audio thread on every tick:

- `MachineWorkInstance.Tick()` line ~600–620: for every machine that
  ticks, iterates **every parameter of every group**, and for every
  track sets `p.SetPValue(i, p.NoValue)` — i.e. clears the entire
  pvalue table. Each call enters a `ConcurrentDictionary` write path
  (TryAdd / TryUpdate, internal `Monitor.Enter` on a striped lock,
  hash computation, bucket walk).
- `AudioMessage.AudioTick` line ~80–150: reads `GetPValue` for every
  parameter for every track to assemble the IPC tick payload.
- `ParameterCore.SetValue` (line 308) writes both `values` and
  `pvalues` on every external parameter change.

**Why it's wrong for the workload.** Tracks are dense integer
indices 0..N-1, N is small (≤16 typical, ≤256 max). The access
pattern is sequential and the audio thread is single-writer per
parameter. A plain `int[]` (with `Volatile.Read/Write` if there's
genuinely a cross-thread reader) would be one cache line vs a
hashtable with bucket array, entry array, striped lock array, and
boxing-free but still ref-cell-tracking concurrency machinery.

**Expected gain.** For a song with ~30 machines averaging 8 tracks
and ~10 parameters each, that's ~2400 ConcurrentDictionary touches
per tick from `Tick()` alone, plus the AudioTick reads. Per-tick
fixed cost in the high microseconds, likely tens of microseconds at
small block sizes. Replacing both `values` and `pvalues` with
`int[]` per (parameter, track-count) should be 10–50× faster on this
hot path and removes a significant fraction of audio-thread lock
contention.

**Recommendation.** Allocate `int[]` sized to `TrackCount`. Resize
on `SetMachineTrackCount` (already serialized through `workLock` in
`SetValue`). The Tick "clear to NoValue" loop becomes
`Array.Fill(pvalues, NoValue)` — one call per parameter instead of
N calls. The IPC payload assembly in `AudioTick` becomes an array
read, not a dictionary lookup.

---

## 2. Native-machine IPC payload assembly allocates and copies excessively

`ReBuzz/NativeMachine/NativeMessage.cs:68` declares
`readonly List<byte> sendMessageData = new List<byte>();` and every
`SetMessageData<T>` call does:

```csharp
byte[] array = GetArray(sizeof(T));  // reused 4-byte scratch
fixed (byte* ptr = array) { *(T*)ptr = data; }
sendMessageData.AddRange(array);     // copies into List<byte>
```

`AddRange` on `List<byte>` for a 4-byte source array goes through
`ICollection<T>.CopyTo`, with a length check, capacity check, and
internal `Array.Copy`. Doing this 4 bytes at a time is genuinely
expensive when the payload is a 1024-byte stereo buffer of 256
samples (512 AddRange calls per Work) — and that's the **prologue**
to the real cost:

`NativeMessage.cs:148`:

```csharp
fixed (byte* source = sendMessageData.ToArray())
{
    Unsafe.CopyBlock(dataRootPointer, source, (uint)size);
}
sendMessageData.RemoveRange(0, MessageBuffer.MaxSize);
```

`sendMessageData.ToArray()` **allocates a fresh `byte[]` the size of
the whole payload on every send** — every native-machine Work, every
audio block. Then `RemoveRange(0, ...)` shuffles whatever's left.
`receaveMessageTable = msg.ReceaveMessageData.ToArray();` (line 219)
does the same on the reply side. **HOT, allocates per call.**

**Cost shape.** Per native-machine Work call: one ~1–2 KB allocation
for the send buffer, one for the receive buffer, plus the
hundreds-of-AddRange overhead. With N native machines and 187
audio blocks/sec, an N=8 song burns 16 × 187 ≈ 3000
~1KB allocations/sec ≈ 3 MB/sec of Gen0 garbage. Gen0 collections
on the audio thread are an obvious source of dropouts.

**Recommendation.**

1. Replace `List<byte> sendMessageData` with a preallocated
   `byte[] sendBuffer` and an `int sendOffset`. `SetMessageData<T>`
   becomes `Unsafe.WriteUnaligned(ref sendBuffer[sendOffset], data);
   sendOffset += sizeof(T);` — no allocation, no intermediate copy.
2. `CopyMessageToSendBuffer` becomes a single `Unsafe.CopyBlock`
   from `sendBuffer` directly, no `ToArray()`.
3. `SetMessageData(Sample[] data, int n, bool stereo)` is currently
   per-sample `AddRange`s of 4 bytes — replace with one
   `Buffer.MemoryCopy` (or a tight unsafe loop) writing all 8n
   bytes contiguously into `sendBuffer`. This alone is probably a
   20–50× speedup on the audio-buffer marshalling path for stereo
   blocks.
4. On the receive side, `MessageContent.ReceaveMessageData` should
   expose the backing buffer directly (or a `ReadOnlySpan<byte>`)
   so callers don't need `.ToArray()`.

This is the largest single source of GC pressure on the audio
thread in any session with native machines.

---

## 3. `AudioTick` builds throwaway `List<byte>` per call

Same file as above, `AudioMessage.cs:91–134`:

```csharp
List<byte> globalVals = new List<byte>();
List<byte> trackVals  = new List<byte>();

foreach (var p in machine.ParameterGroupsList[1].ParametersList) {
    int typeSize = p.GetTypeSize();
    int pValue   = p.GetPValue(0);
    if (typeSize == 1)      globalVals.Add((byte)pValue);
    else if (typeSize == 2) globalVals.AddRange(GetBytes((ushort)pValue));
}
for (int i = 0; i < machine.TrackCount; i++) {
    foreach (var p in machine.ParameterGroupsList[2].ParametersList) {
        ...
        trackVals.AddRange(GetBytes((ushort)pValue));
    }
}

...
SetMessageData(globalVals.ToArray());
SetMessageData(trackVals.ToArray());
```

Two `List<byte>` allocations + a pair of `ToArray()` copies +
`GetBytes` (which itself returns scratch but `AddRange` re-copies
it) **per tick per native machine**. The `GetPValue` calls hit the
ConcurrentDictionary issue from §1.

**Recommendation.** Once the `pvalues` array refactor in §1 is done,
this whole function can write parameter bytes straight into the
sendBuffer from §2 with no intermediate `List<byte>`. The shape of
the payload is fully determined by the (global params, track params,
track count, type sizes) which can be cached at machine init time —
construct an "encode plan" once, then per-tick it's just a sequence
of typed reads + writes into the existing scratch.

---

## 4. The single `AudioLock` wraps the entire `ThreadRead`

`ReBuzz/Audio/WorkManager.cs:113`:

```csharp
public int ThreadRead(float[] buffer, int offset, int count)
{
    lock (ReBuzzCore.AudioLock)
    {
        // ... entire DSP graph processing, hundreds of microseconds to milliseconds ...
    }
}
```

`AudioLock` is held for the duration of the whole graph traversal —
parameter updates, ticking, work for every machine, IPC round-trips,
master mix, the lot. The same `AudioLock` is grabbed by 54 other
sites across the codebase (every file save/load, machine add/remove,
connect/disconnect, wave operations, MIDI engine, UI message
handlers). Any of those that fires from a non-audio thread will
serialize behind the audio block, and the audio block will serialize
behind any of those that's already running.

This is the architectural choice that limits how short ReBuzz's
buffer size can safely go: with everything funneled through one
mutex, a UI-side `lock (AudioLock)` that holds for 2 ms during a
file dialog is a guaranteed dropout at 1.3 ms buffer latency.

**Recommendation (largest, hardest).** Split `AudioLock` into a
small set of finer-grained locks scoped to what they actually
protect: machine-graph topology (add/remove/connect/disconnect),
machine state (parameters, tracks), and the audio engine's
read-side. Most of the 54 sites are graph-topology operations that
only need a "no graph mutation while audio is mid-block" guarantee —
that can be a single read/write lock where the audio thread takes
read (cheap, can be parallel with itself if the algorithm allows),
and topology mutators take write (blocking, but rare and acceptable
to wait one audio block). Parameter changes from UI shouldn't need
any audio-thread serialization — they should go through a SPSC
queue drained at the top of `ThreadRead`, removing the entire
`SetValue` → `lock(machine.workLock)` path from the audio thread's
contention surface.

This is invasive but it's the single change that would unlock
sub-5ms buffer sizes safely.

---

## 5. Per-task closure allocations in the multithreaded work algorithms

`ReBuzz/Audio/WorkManager.cs:644, 689, 724` all look like this:

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

Three problems on one hot path:

1. **`OrderByDescending(...)` LINQ on every iteration** of the
   outer `while` loop in `HandleWorkAlgorithmGroups`. Allocates an
   enumerator, sorts on every call, throws it all away.
2. **Lambda captures `machine`, `workSamplesCount`, `buzzCore`,
   `engineSettings`** — the C# compiler synthesizes a display class
   and allocates one **per task created**. With 30 machines per
   block at 187 blocks/sec, that's 5600+ display-class allocations/
   sec, plus 5600+ Task objects, plus the workTasks.ToArray()
   allocation for `WaitAll`.
3. **`Task.WaitAll(workTasks.ToArray())`** — the `ToArray` is
   gratuitous; there's a `Task.WaitAll(IEnumerable<Task>)` overload,
   and even better a `Task.WaitAll(params Task[])` already-have-array
   path. The existing code allocates a fresh array of refs every
   block.

`HandleWorkRecursive` is worse: it allocates a `Task` per **input
edge** of every machine, recursively, every block.

**Recommendation.** Switch from `Task` to a pre-sized worker pool
with a lock-free work-stealing queue (or even just `Parallel.For`
with `Partitioner.Create` reused across blocks). Algorithm 2 in
`WorkThreadEngine` is closer to the right shape — make it the only
multi-threaded option and remove the `Task`-based variants.

If keeping the current structure short-term, at minimum:

- Replace `OrderByDescending` with an in-place insertion-sort on a
  reused `MachineCore[]` scratch (the work list is small).
- Make the lambda body call a method instead of capturing — e.g.
  store `(machine, samples)` pairs in a reused struct array and have
  workers `Interlocked.Increment` an index counter to claim items,
  same pattern as `WorkThreadEngine.GetWorkItem` but with no lock.
- Replace `Task.WaitAll(workTasks.ToArray())` with
  `Task.WaitAll(workTasks)` (List<Task> has a direct overload via
  IEnumerable but it'll still allocate; better is a pre-sized array
  of Tasks kept around and reused).

---

## 6. `MachineCore.GetActivity` runs a full 256-sample max-scan per machine per block

`ReBuzz/Core/MachineCore.cs:1346`:

```csharp
public bool GetActivity()
{
    bool isActive = false;
    float maxSample = 0.0f;
    var connections = Name == "Master" ? Inputs : Outputs;

    foreach (var output in connections)
    {
        Sample[] samples = (output as MachineConnectionCore).Buffer;
        for (int i = 0; i < samples.Length; i++)
        {
            maxSample = Math.Max(Math.Abs(samples[i].L), maxSample);
            maxSample = Math.Max(Math.Abs(samples[i].R), maxSample);
        }
    }
    return maxSample > AMP_EPS;
}
```

Called from `MachineWorkInstance.TickAndWork` and again from
`HandleWorkRecursive` as a "Some machines report the active state
wrong, double check" pass. **HOT, VERY HOT inner loop.**

Three things:

1. `samples.Length` is **always 256** — the buffer is always
   allocated full-size, regardless of `nSamples` actually processed.
   So even on tiny blocks (e.g. 32 samples), this scans 256.
2. `Math.Max(Math.Abs(...))` on each channel produces a long
   dependency chain. With `MathF` and a small SIMD batch, this
   loop is 4–8× faster on modern CPUs.
3. `AMP_EPS = 1.0f` — but samples here are in Buzz's internal
   ±32768 scale (the master conversion divides by 32768 at the
   final output stage). So `AMP_EPS = 1.0f` is effectively
   `1 / 32768 = -90 dBFS`. That's an extremely generous activity
   threshold; almost any DC-offset or denormal residual will keep
   `IsActive == true`. Combined with the per-machine env-tail issue
   in `Pedal invFFT §17` notes, this means machines that report
   their own activity correctly **still get scanned** by this
   double-check, with the bias toward "active = true" hiding the
   silent-skip path.

**Recommendation.**

- Scan only `nSamples`, not `samples.Length`. Pass `nSamples` in or
  cache it on the connection. The buffer-length-vs-actual-samples
  confusion shows up in several other places (§7, §8) so this is
  worth fixing structurally.
- Replace `Math.Max(Math.Abs(...))` with a SIMD `Vector<float>`
  abs+max — `System.Numerics.Vectors` is available; one 256-element
  pass becomes ~32 vector ops.
- Trust the machine's own returned `isActive` when it's a managed
  machine (those are the ones you control). The "double check" is
  defensive coding against badly-behaved native plugins; gate it on
  `Machine.DLL.Info.Version` or a per-machine flag, don't run it for
  every well-behaved machine.

---

## 7. `MachineCore.GetStereoSamples` clears the full 256-sample scratch every call

`ReBuzz/Core/MachineCore.cs:1244`:

```csharp
readonly Sample[] stereoSamples = new Sample[256];
internal Sample[] GetStereoSamples(int nSamples)
{
    for (int i = 0; i < stereoSamples.Length; i++)      // ← always 256
        stereoSamples[i].L = stereoSamples[i].R = 0;

    foreach (var input in Inputs) {
        var inputCore = input as MachineConnectionCore;
        for (int i = 0; i < nSamples; i++) {
            stereoSamples[i].L += inputCore.Buffer[i].L;
            stereoSamples[i].R += inputCore.Buffer[i].R;
        }
    }
    return stereoSamples;
}
```

Same buffer-size confusion as §6. Clears all 256 samples even if
nSamples is 32 or 64. **HOT, per machine per block.** Small but
adds up across the graph.

Also: returns the shared internal buffer, which means callers can't
rely on it surviving across calls. That's fine, but means there's
zero benefit from making the buffer larger than nSamples max — so
either clear only `nSamples` or use `Array.Clear(stereoSamples, 0,
nSamples)`.

`GetMultiIOSamples` at line 1278 has the same issue plus a worse
one: when reusing an existing channel buffer, it clears
`ci.Length` (the whole buffer) rather than `nSamples`.

**Recommendation.** Clear only what's about to be summed into. Use
`Array.Clear` (it dispatches to a vectorized memset internally) not
a hand-rolled loop. Same fix everywhere in `MachineCore.cs` that
loops over `.Length` instead of `nSamples`.

---

## 8. `FlushDenormalToZero` is mostly a no-op and is called in tight loops as if it mattered

`ReBuzz/Common/Utils.cs:331–351`:

```csharp
public static void FlushDenormalToZero(Sample[] samples)
{
    /*
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

The `Sample[]` overload is **completely commented out**. The scalar
overload adds a tiny DC offset (`1e-15f`) but never subtracts it —
which the comment in your own Core notes §30.5 explicitly warns
against: "Doesn't work reliably under aggressive optimization — the
JIT may fold the two operations into a no-op." Here there's no
second operation at all, just permanent DC injection.

Then the no-op array form is called in tight loops:

- `MachineCore.UpdateOutputs:1271` — every output per Work call
- `MachineCore.UpdateOutputs(List, ...):1339` — every multi-output
- `MachineConnectionCore.UpdateBuffer:146` —
  `Utils.FlushDenormalToZero(Buffer)`, no-op

And the scalar form is called per-sample at the master mix:

- `WorkManager.ThreadRead:231` — `buffer[offset+i] =
  FlushDenormalToZero(buffer[offset+i] * audioOutMul)`, ie a
  function-call wrapper around `(x * c) + 1e-15f`. Per output
  sample.
- `WorkManager.ReadWork:598–599` — same per L/R sample of master.

So the audio output is permanently DC-offset by ~1e-15 (inaudible
but not zero), and the array-form denormal "protection" doesn't
exist at all. Real denormals reaching feedback delay lines or
filter integrators **are not protected**.

**Recommendation.**

1. Remove all `FlushDenormalToZero(Sample[])` call sites — they
   accomplish nothing.
2. Remove `FlushDenormalToZero(float)` from the per-sample master
   loop; replace with the (much faster) MXCSR DAZ+FTZ approach: set
   denormals-are-zero and flush-to-zero on the audio thread's MXCSR
   once, at thread start, via
   `System.Runtime.Intrinsics.X86.Sse.SetFlushToZeroMode(...)` and
   `Sse3.SetDenormalsAreZerosMode(...)`. Caveats from Core §30.5
   apply (per-thread, can affect hosted plugins' behaviour) — but
   this is a host, the choice of denormal handling belongs to the
   host, and most pro audio hosts make exactly this choice.
3. If per-thread MXCSR is rejected on the grounds in Core §30.5,
   then the protection has to live inside machines (which is where
   Pedal builds already do it). In that case the host-side calls
   should be deleted entirely — they currently provide no
   protection while still costing a function call per sample.

The current state is the worst of both worlds: it has the cost of
denormal protection but provides none of it.

---

## 9. `MachineConnectionCore.UpdateBuffer` — modulo per sample, broken interpolation, no-op flush

`ReBuzz/Core/MachineConnectionCore.cs:122`:

```csharp
internal void UpdateBuffer(Sample[] samples, int nSamples)
{
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

    Utils.FlushDenormalToZero(Buffer);   // no-op (see §8)
}
```

This is **VERY HOT** — runs per sample for every connection in the
graph. Issues:

1. `%` operator on every sample for ring-buffer wrap. Integer
   division on x86-64 is 20–80 cycles; branchless wrap
   (`if (pos == len) pos = 0;`) is 1 cycle and predictable. With
   the typical `addedLatency == 0` case, `latencyBuffer.Length`
   is fixed at 256 — power of two — so `pos & (len-1)` is also
   one cycle. Today's code: two `%` per sample per connection.
2. `ampStep` math is wrong. Read it carefully: `ampStart` is a
   normalized 0..1 float; `ampCurrent` is `(int)Tick()` cast to
   float, so `ampCurrent / 0x4000` is also 0..1. The subtraction
   produces a small float (the per-tick delta), and then it's
   divided by **`0x4000` again** and then by `nSamples`. That
   makes `ampStep` ~`delta / 16384 / 256 ≈ delta / 4M`. Effectively
   zero. So the per-sample amp interpolation **is doing nothing** —
   the entire smoothing path is dead code. Connection amp changes
   land as instantaneous jumps, which is presumably audible as
   clicks on slider moves.
3. The final `FlushDenormalToZero(Buffer)` is the no-op array form
   from §8.
4. The two-pass write-then-read structure means the data is touched
   twice through L2/L3 cache. When `addedLatency == 0` (the common
   case — no latency compensation in use), there's no reason for a
   ring buffer at all; could write directly to `buffer[i]`. Add a
   fast path: `if (addedLatency == 0) { /* direct path */ }`.

**Recommendation.**

- Fast path for the common `addedLatency == 0` case: skip the
  latency ring entirely, write straight to `buffer`.
- For the ring-buffer case, use bitmask wrap when length is a
  power of two (which the default 256 is).
- Fix the `ampStep` math. The interpolator class is presumably
  already returning the right thing — just use its output directly
  rather than the double-division. Look at how
  `interpolatorAmp.SetTarget(value, 4)` is being driven: it looks
  like it's set to interpolate over 4 ticks, but `Tick()` is being
  called once and the result divided. Either drive the interpolator
  per-sample over `nSamples` ticks, or compute a proper linear
  ramp from the previous block's end-value to the current
  `interpolatorAmp.Value`.
- Drop the no-op flush.
- Consider Vector<float> for the multiply: amp and pan are scalars
  for the duration of the block in any sane fix to (2), so this
  becomes a textbook SIMD-vectorizable pair of multiplies.

---

## 10. `MachineConnectionCore.DoTap` allocates per call, and `ReBuzzCore.MasterTapSamples` allocates harder

`MachineConnectionCore.cs:99` (DoTap), inside the audio thread:

```csharp
public void DoTap(Sample[] sampleBuffer, int nSamples, bool stereo, SongTime songTime)
{
    if (Tap != null)
    {
        float[] samples = new float[nSamples * 2];      // allocation per call
        for (int i = 0; i < nSamples; i++) {
            samples[j++] = sampleBuffer[i].L;
            samples[j++] = sampleBuffer[i].R;
        }
        dispatcher.BeginInvoke(() => { Tap?.Invoke(samples, stereo, songTime); });
    }
}
```

Allocates a fresh `float[]` per output per Work call **and** queues
a dispatcher delegate, **only when the Tap event is subscribed**.
With a meter or scope subscribed to every output, this generates
hundreds of allocations + dispatcher queue items per second.

`ReBuzzCore.MasterTapSamples` (`Core/ReBuzzCore.cs:1655`) is worse:

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

    float[] samples = new float[count];      // unconditional allocation
    for (int i = 0; i < count; i++)
        samples[i] = resSamples[offset + i];

    dispatcher.BeginInvoke(() => {
        if (MasterTap != null)
            MasterTap.Invoke(samples, true, s);
    });
}
```

Three bugs/wastes:

- **Allocates unconditionally**, even when `MasterTap == null`.
  Then checks `if (MasterTap != null)` inside the dispatcher
  callback — too late to avoid the cost.
- **Allocates per block** at the master, called for every audio
  block. That's the 4KB-per-block allocation rate noted earlier:
  ~750 KB/sec of Gen0 garbage just from the master tap.
- `maxSampleLeft` / `maxSampleRight` are **never reset between
  blocks** — they hold the maximum since process start. So any
  consumer reading those as block-level peak meters gets the
  all-time peak, growing monotonically.

**Recommendation.**

- Both tap paths: maintain a ring of preallocated `float[]`
  buffers (say 4 deep) and rotate through them, so the dispatcher
  callback gets a stable buffer that won't be overwritten before
  it's consumed. No allocation per block.
- Early-out: check `Tap == null` / `MasterTap == null` and skip
  everything — don't allocate, don't compute the peak, don't
  dispatch.
- Reset the master peak per block (or accumulate properly into a
  separate "blockPeak" that the meter consumer reads + clears).

---

## 11. LINQ and double-dictionary-lookup patterns on hot paths

Multiple sites. Each individually small; collectively a paper cut.

**`WorkManager.UpdateMasterParams` (line 254)** — called every
audio chunk:

```csharp
var master = buzzCore.SongCore.MachinesList.FirstOrDefault(
    m => m.DLL.Info.Type == MachineType.Master);
```

Linear scan of all machines + a closure, every block. Cache the
master reference once at construction (it's already done at line
67, just discarded — store it in a field). `WorkManager.ReadWork`
line 576 also does `FirstOrDefault` to find Master.

**`MachineManager.GetMachineWorkInstance` (line 1018)** — called
multiple times per block per machine:

```csharp
if (!workInstances.ContainsKey(machine)) { ... }
else { return workInstances[machine]; }
```

Double dictionary lookup. Replace with `TryGetValue`.

**`WorkManager.CallTick` (line 420)** — iterates all machines
twice, once for control, once for non-control. Predicate inside
each loop is non-trivial (`machine.IsControlMachine && (...
masterInfo.PosInTick == 0 || ...)`). Either cache the
control/non-control partition (it only changes on machine
add/remove) or merge into one loop with the partition built into
the iteration order.

**`HandleWorkRecursive` (line 716)** — `OrderByDescending` on
every recursive call. Same fix as §5.

**`PlayPatternColumnEvents` (line 519)**:

```csharp
for (int j = 0; j < ces.Count(); j++) {
    var pe = ces.ElementAt(j);
    ...
}
```

`ces` is whatever `column.GetEvents(...)` returns — if it's not an
`IList`, `Count()` enumerates the whole thing and `ElementAt(j)`
**re-enumerates from the start each iteration**. That's O(N²) per
column per block. Even if `GetEvents` returns a List, `ElementAt`
on an unknown `IEnumerable<T>` doesn't always hit the optimized
path. Materialize to an array once or use `foreach`.

**`UpdatePatternPositions`** does `foreach (var machine in
buzzCore.SongCore.Machines) foreach (var pattern in
machine.Patterns) pattern.PlayPosition = int.MinValue;`. That's
every machine × every pattern, every audio block, for the sole
purpose of resetting a property that's then conditionally set
again below. Filter to patterns that are actually playing, or
defer the reset to once-per-tick rather than once-per-block.

---

## 12. `DateTime.UtcNow.Ticks` for per-block timing

`WorkManager.cs:115, 248` and `MachineWorkInstance.cs:45, 78`:

```csharp
long time = DateTime.UtcNow.Ticks;
// ... work ...
buzzCore.PerformanceCurrent.EnginePerformanceCount += (DateTime.UtcNow.Ticks - time);
```

`DateTime.UtcNow` is implemented on top of `GetSystemTimePreciseAsFileTime`
on modern Windows, which has ~1 µs resolution and goes through a
kernel transition (or at least a VDSO-style call on the system page).
For per-machine performance measurement at 187 Hz across N machines,
that's 2N kernel-ish calls per block.

`Stopwatch.GetTimestamp()` is `QueryPerformanceCounter` — much
faster, doesn't synchronize with wall clock, exactly what
"performance count" wants. Convert ticks to time only when
displaying.

**Recommendation.** Replace all four sites with `Stopwatch.GetTimestamp()`.
Compute frequency once. The savings are probably small (a few
hundred ns × 2 × N machines × 187/sec = low microseconds/sec) but
it's a one-line fix and removes a kernel-ish syscall from the
audio path.

---

## 13. `MachineWorkInstance.Tick` clears the full pvalues array even when nothing changed

`MachineWorkInstance.cs:602–622`. After processing
`parametersChanged` events, the method **unconditionally** iterates
every parameter of all three groups, setting every (param, track)
pvalue to `NoValue`. Coupled with the ConcurrentDictionary cost in
§1, this is the bulk of `Tick`'s cost.

But even after fixing §1 with arrays, the logical cost is
unnecessary: only the parameters in `parametersChanged` actually
need their pvalue reset. The "reset everything" pass is defensive —
guarding against the case where a parameter was set externally
without going through `parametersChanged`.

**Recommendation.** Track exactly which (param, track) pairs were
touched (the `parametersChanged` dictionary already does this) and
reset only those. The keys of `parametersChanged` already tell you
the parameters; the value tells you the track. After §1, this
loop becomes O(actual changes) rather than O(all parameters ×
all tracks).

---

## 14. `UpdateMasterAndSubTickInfoToHost` syncs to every managed machine every block

`MachineManager.cs:978`:

```csharp
internal void UpdateMasterAndSubTickInfoToHost()
{
    foreach (var machineHost in managedMachines.Values) {
        if (machineHost != null)
            UpdateMasterAndSubTickInfo(machineHost);
    }
}
```

Called from `WorkManager.ThreadRead` line ~167 every block. Copies
10 fields × N managed machines, even if the values haven't changed
since last block.

This is small per-call but it's also unnecessary: `MasterInfo` and
`SubTickInfo` rarely change between blocks (only on BPM/SR change
or sub-tick transition). Track a "dirty" flag in `WorkManager`
that's set when any of the source fields change, and only propagate
on dirty. Most blocks become a no-op.

Even better: have managed machines read from a shared `MasterInfo`
struct backed by `ReBuzzCore.masterInfo` directly rather than copy
into per-host structs. This requires the public surface
(`IBuzzMachineHost.MasterInfo`) to expose a read-only view, which
may be a breaking change for some managed-machine API consumers —
but it eliminates the propagation entirely.

---

## 15. `WorkManager.workTasks.ToArray()` per call in HandleWorkList

`WorkManager.cs:884` — `Task.WaitAll(workTasks);` actually passes
the List form, but `WaitAll` has no `IEnumerable` overload that
avoids array conversion — internally it'll copy to an array
regardless. The `tickTasks` field (line 418) is declared but
never used (dead code).

Similar small issue: `workList` is a `Dictionary<MachineCore,
bool>` used as a set. The `bool` value is always `true`. Replace
with `HashSet<MachineCore>` to drop the value storage; or, since
order matters in some passes, use a `List<MachineCore>` with the
"already in list" check moved up the call chain.

---

## 16. Reflection-based property access in `ManagedMachineHost.Commands` and `MachineState`

`ManagedMachineHost.cs:263, 317, 353`:

```csharp
public IEnumerable<IMenuItem> Commands
{
    get {
        PropertyInfo property = dll.machineType.GetProperty("Commands");
        if (property == null) return null;
        MethodInfo getMethod = property.GetGetMethod();
        ...
        object obj = getMethod.Invoke(machine, null);   // slow path
        ...
    }
}
```

Not audio-thread, but called per right-click and per save. Cache
the `MethodInfo` once at construction (same pattern as the work
delegate caching done at line 181). Better: convert it into a
`Func<IEnumerable<IMenuItem>>` delegate at construction time via
`Delegate.CreateDelegate`, matching how `Work` is wired. Then the
hot UI path doesn't pay the reflection cost.

`MachineState` get/set is similar — XML serialization is the bulk
of its cost, but the per-call `GetProperty` + `Invoke` is
avoidable.

---

## 17. `MachineWorkInstance` work-mode buffer copies that could be array assignments

`ManagedMachineHost.cs:440–480` — every `Work` call for a
`GeneratorBlock` / `EffectBlock` machine copies the input buffer
to a private `inputBuffer`, runs the user's block, then copies
`outputBuffer` back into `samples`:

```csharp
for (int j = 0; j < n; j++) { inputBuffer[j] = samples[j]; }
bool flag2 = EffectBlockWork(outputBuffer, inputBuffer, n, mode);
if (flag2 && (mode & WorkModes.WM_WRITE) != 0) {
    for (int k = 0; k < n; k++) { samples[k] = outputBuffer[k]; }
}
```

`Sample` is a small struct (8 bytes). The hand-rolled copy loops
are likely to be vectorized by RyuJIT, but `Array.Copy(samples, 0,
inputBuffer, 0, n)` / `Array.Copy(outputBuffer, 0, samples, 0, n)`
is explicit, doesn't depend on JIT inlining, and uses a vectorized
copy intrinsic.

Beyond that: the **whole copy is avoidable** when the user's
machine accepts `samples` directly. The pattern of "copy in, work
in-place on private buffer, copy out" only makes sense if you need
to protect against the user mutating the input array — which is
exactly what `WM_READWRITE` is for. For `WM_WRITE` (generator),
the input copy is pure waste. Audit the work-mode handling and
elide the input copy when `(mode & WM_READ) == 0`. (The current
code does skip the *input* copy for the generator-only block, but
the effect-block path always copies in regardless of mode.)

---

## 18. `oversample` paths allocate fresh sample arrays through `GetParts`

`MachineWorkInstance.cs:328`:

```csharp
private List<Sample[]> GetParts(List<Sample[]> sampleArray, int count, int offset)
{
    oversampleMultiInParts.Clear();
    for (int i = 0; i < sampleArray.Count; i++) {
        ...
        var to = oversampleMultiInPartsBuffers[i];
        Array.Copy(from, offset, to, 0, count);
        oversampleMultiInParts.Add(to);
    }
    return oversampleMultiInParts;
}
```

`List.Add` won't allocate if capacity already holds, so this is OK.
But `Oversample` (line 488) uses `WorkManager.SpeedDown` (the
Hermite resampler) for both upsample and downsample — and
`SpeedDown` is a generic linear-interp/Hermite resampler not
specifically tuned for integer-ratio oversampling. For ×2 / ×4
integer oversample, half-band FIRs are dramatically more efficient
and have better stopband. The current path is doing per-sample
Hermite for every machine being oversampled, every block — fine if
you have one ×2 machine in the song, painful if you have several.

Not a quick fix — replacement requires a real half-band FIR
implementation. But flagging because oversample-enabled songs can
have surprisingly high CPU vs non-oversample, and that gap is
larger than it needs to be.

---

## Audio-correctness issues spotted along the way

These are not performance — they're things that look wrong and
should probably be tracked separately. Including here because they
turned up during the read.

### A. `MachineConnectionCore.UpdateBuffer` amp step is effectively zero

See §9 point 2. Connection amp changes are not smoothed despite
the code reading like it intends to smooth them. Expect clicks
on amp slider drags.

### B. `ReBuzzCore.MasterTapSamples` peak meters never reset

See §10 point 3. `maxSampleLeft` / `maxSampleRight` accumulate
since process start.

### C. `RefreshMachineParams` track-index update is in the wrong scope

`MachineManager.cs:1318`:

```csharp
foreach (var g in machine.ParameterGroupsList) {
    int track = 0;
    foreach (var p in g.ParametersList) {
        if (p.Flags.HasFlag(ParameterFlags.State) && p.Type != ParameterType.Note)
            p.SetPValue(track, p.GetValue(track));
    }
    track++;
}
```

`track++` is outside the inner `foreach` but inside the outer.
Result: `track` only ever takes the value 0 inside the
SetPValue call. So per-track refresh on tempo change only ever
hits track 0. Possibly intentional, possibly a bug — the variable
name and structure suggest "for every parameter in every group,
for every track refresh state params" was the intent.

### D. `WorkManager.UpdateMasterParams` calls `RefreshMachineParams` even when only one of BPM/TPB changed

Minor. Not wrong in any audible sense, just over-eager.

### E. `Machine.PerformanceDataCurrent.MaxEngineLockTime = 0;` set every block, never set anywhere else

`MachineWorkInstance.cs:80`. Dead assignment.

### F. ASIO `audioInBuffer` is 512 floats but `Buffer.BlockCopy` is bounded by `audioInBuffer.Length * 4` = 2048 bytes

`AudioEngine.cs:140`. Looks right but worth a closer look — there's
no guard that `bytesRemaining` aligns to the float boundary, and
the partial-buffer logic at the bottom isn't shown in my snippet.
Tag for review, not a confirmed bug.

---

## Suggested order of attack

If picking a small number of changes to ship first, in roughly
descending value-per-effort order:

1. **§1 — Replace ConcurrentDictionary with int[] for pvalues/values.**
   Mechanical, isolated to `ParameterCore`, large per-tick wins,
   no API change.

2. **§2 + §3 — Native IPC payload buffer.** Replace `List<byte>
   sendMessageData` with a preallocated `byte[]`. Removes the
   biggest source of audio-thread GC pressure when native machines
   are in use. Localized to `NativeMessage.cs` / `AudioMessage.cs`.

3. **§10 — Fix the master tap and connection tap allocations.**
   Two files, ring-of-buffers pattern, no API change. Stops the
   ~750 KB/sec Gen0 churn at the master.

4. **§8 — Delete the no-op denormal flushes, set MXCSR once.**
   Trivial code change, removes a per-sample function call from
   the master mix, fixes the "no denormal protection actually
   exists" bug.

5. **§6, §7 — Clear/scan `nSamples` not `.Length`.** Easy bulk fix
   across `MachineCore` and `MachineConnectionCore`. Free CPU at
   short block sizes.

6. **§5 — Switch the multithreaded work path away from per-block
   Task allocation.** Larger change but removes 5000+ display-class
   + Task allocations/sec in multithreaded mode.

7. **§9 — Fast path for `addedLatency == 0` in `UpdateBuffer`, fix
   `ampStep` math.** Mid-effort, fixes one of the click sources
   *and* speeds up every connection.

8. **§4 — Split `AudioLock`.** Largest invasive change, largest
   unlock on minimum buffer size. Save for last.

The rest (§11–§18, correctness items A–F) are paper cuts and small
fixes worth doing opportunistically when touching the surrounding
code.
