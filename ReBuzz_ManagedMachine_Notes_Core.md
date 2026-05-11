# ReBuzz Managed Machine Development — Core Notes

Source: ReBuzz 1817-preview source code + debugging session for Pedal Chord.
Updated with findings from Pedal EQ v1.2 build (ReBuzz 1819-preview).
Updated with findings from Pedal Dly PCM41 v1.0 build (ReBuzz 1819-preview).
Updated with findings from Pedal Plaits v0.4 build (ReBuzz 1819-preview).
Updated with findings from Pedal Plaits v0.6 build (ReBuzz 1819-preview).
Updated with findings from Pedal Plaits v0.7 build (ReBuzz 1819-preview).
Updated with findings from Pedal Plaits v1.2 build (ReBuzz 1819-preview).
These details are absent from official documentation.

Sections 1–31 are general findings that apply across managed machines.
Machine-specific addenda live in separate files (Pedal Comp, Pedal Tracker,
Pedal Muter). Each addendum uses its own local 1–N numbering and refers back
to this file as `Core §N`.

---

## 1. `IBuzzMachine.Tick()` is never called

For managed machines, `IBuzzMachine.Tick()` is **dead code**. ReBuzz does not
call it. All per-tick logic must live in `Work()`.

What ReBuzz *does* call automatically before `Work()`:
- Parameter setter methods (`SetNote`, `SetChord`, etc.) — delivered via
  `ManagedMachineHost.Tick()` which iterates `machine.parametersChanged`.
- These are called because the pattern engine (or another machine) called
  `IParameter.SetValue()` on your parameters.

**Pattern:** Use your setters to store incoming values, then process them in
`Work()`.

```csharp
// Setter — called by ReBuzz from managedMachineHost.Tick()
public void SetNote(Note value, int track)
{
    _pendingNote[track] = value.Value;
    _hasNote[track]     = true;
}

// Work — where you actually act on them
public void Work()
{
    for (int v = 0; v < MaxVoices; v++)
    {
        if (!_hasNote[v]) continue;
        _hasNote[v] = false;
        // ... fire notes on target machine
    }
}
```

---

## 2. Control machine classification — `void Work()`

A managed machine becomes a **control machine** when it has exactly this
method signature:

```csharp
public void Work() { }   // no parameters, void return
```

`ManagedMachineDll.GetWorkFunctionType()` detects this via reflection:

```csharp
if (parameters.Length == 0 && method.ReturnType == typeof(void))
    return WorkFunctionTypes.Control;
```

Effects of being a control machine:
- `machine.IsControlMachine = true`
- Ticked **first** in `WorkManager.CallTick()`, before all generators/effects
- No audio output (OutputChannelCount = 0)
- Appears in the Generators tab in the machine browser

`MachineDecl` has **no Flags property** — you cannot set `CONTROL_MACHINE`
there. The void `Work()` signature is the only mechanism.

---

## 3. Note parameter type — use `Note`, not `int`

ReBuzz determines `ParameterType.Note` from the **C# type of the method's
first parameter**, not from MinValue/MaxValue ranges.

```csharp
// WRONG — shown as hex in pattern editor, no piano keyboard input
public void SetNote(int value, int track) { ... }

// CORRECT — piano keyboard grid, z=C-4, s=C#-4, etc.
public void SetNote(Note value, int track) { ... }
```

`Note` is `Buzz.MachineInterface.Note` — a struct with:
- `byte Value` — the Buzz note encoding
- `int ToMIDINote()` — converts to 0-based MIDI
- `static Note FromMIDINote(int x)` — converts from 0-based MIDI
- `const int Min = 1`, `Max = 156` (`(16*9)+12`), `Off = 255`

When the parameter is `Note` type, ReBuzz hard-codes
`MinValue=1, MaxValue=156, NoValue=0` regardless of what's in `ParameterDecl`.

`ParameterDecl` has **no `Type` property and no `NoValue` property** — adding
either causes a compile error.

Minimal correct declaration:

```csharp
[ParameterDecl(Name = "Note", IsStateless = true,
    Description = "Root note — z=C-4, s=C#-4 …")]
public void SetNote(Note value, int track) { ... }
```

---

## 4. `SendControlChanges()` — required after writing to a target

When your machine calls `IParameter.SetValue()` on another machine's parameter,
that value is written into `pvalues`. For native (C++) machines, `pvalues` are
read by `AudioTick()`. But `AudioTick()` already ran for the target (during
`CallTick()`) **before** your `Work()` was called.

To get the target to read your freshly-written value within the **same tick**,
call:

```csharp
targetMachine.SendControlChanges();
```

This sets `sendControlChangesFlag = true` on the target. When the target's
`TickAndWork()` runs, it detects the flag and calls `Tick(true, true)` —
running an extra `AudioTick()` that reads your pvalue — **before** calling
`WorkMachineNative()`.

Without this, notes are silently lost (pvalues reset to NoValue by
`UpdateNonStateParametersToDefault()` before the next tick's AudioTick).

```csharp
void FireNote(IParameter np, IMachine target, int track, int midiNote)
{
    if (np == null || target == null || track < 0) return;
    try { np.SetValue(track, BN.FromMidi(midiNote)); } catch { return; }
    try { target.SendControlChanges(); } catch { }   // ← essential
}
```

`SendControlChanges()` is defined on `IMachine` (line 81 of IBuzz.cs):
```csharp
void SendControlChanges();   // CMICallbacks::SendControlChanges
```

---

## 5. `TrackCount` is 0 for control machines — do not gate on it

`host.Machine.TrackCount` returns 0 for a control machine even while the
pattern engine is actively delivering notes to its tracks. Do not use it as a
loop bound.

```csharp
// WRONG — loop never runs, all notes silently dropped
int voices = host.Machine.TrackCount;   // always 0!
for (int v = 0; v < voices; v++) { ... }

// CORRECT — iterate all possible voice slots
for (int v = 0; v < MaxVoices; v++) { ... }
```

---

## 6. Finding the Note parameter on a target machine

For native Buzz generators the standard parameter group layout is:
- `ParameterGroups[0]` — Input (volume/pan per connection)
- `ParameterGroups[1]` — Global parameters
- `ParameterGroups[2]` — Track parameters (Note is always index 0 here)

Multi-pass search that handles native and managed targets:

```csharp
IParameter FindNoteParam(IMachine m)
{
    if (m?.ParameterGroups == null) return null;

    // Pass 1: explicit ParameterType.Note (managed machines)
    foreach (var pg in m.ParameterGroups)
    {
        if (pg?.Parameters == null) continue;
        foreach (var p in pg.Parameters)
            if (p?.Type == ParameterType.Note) return p;
    }

    // Pass 2: group index 2 = track group in standard Buzz layout
    //         (index 1 for 2-group machines)
    int tgi = m.ParameterGroups.Count > 2 ? 2 : m.ParameterGroups.Count - 1;
    if (tgi >= 0)
    {
        var pg = m.ParameterGroups[tgi];
        var p  = pg?.Parameters?.FirstOrDefault(x => x != null);
        if (p != null) return p;
    }

    // Pass 3: any Track-typed group
    foreach (var pg in m.ParameterGroups)
    {
        if (pg?.Type != ParameterGroupType.Track || pg.Parameters == null) continue;
        var p = pg.Parameters.FirstOrDefault(x => x != null);
        if (p != null) return p;
    }

    // Pass 4: last non-empty group (last resort)
    for (int gi = m.ParameterGroups.Count - 1; gi >= 0; gi--)
    {
        var pg = m.ParameterGroups[gi];
        if (pg?.Parameters == null || pg.Parameters.Count == 0) continue;
        return pg.Parameters.FirstOrDefault(x => x != null);
    }

    return null;
}
```

---

## 7. Buzz note encoding

```
Byte layout:  [octave : 4 bits][semitone : 4 bits]
Semitone:     1=C, 2=C#, 3=D, 4=D#, 5=E, 6=F, 7=F#, 8=G, 9=G#, 10=A, 11=A#, 12=B
Range:        1 (C-0) to 156=0x9C (B-9)
No-note:      0
Note-off:     255 = 0xFF

Buzz → MIDI:  octave = (b >> 4); semitone = (b & 0xF) - 1; midi = octave*12 + semitone
MIDI → Buzz:  ((midi/12) << 4) | ((midi%12) + 1)

Note.ToMIDINote() and Note.FromMIDINote() implement these correctly.
```

---

## 8. Tick ordering in the audio engine

`WorkManager.CallTick()` ticks machines in two passes:
1. All `IsControlMachine == true` machines  
2. All other machines

Within each pass, machines are iterated in `SongCore.MachinesList` order.

For native machines, `workInstance.Tick()` calls `audiom.AudioTick()` which
reads `pvalues` and sends them to the native machine process via IPC.

For managed machines, `workInstance.Tick()` calls `managedMachineHost.Tick()`
which iterates `machine.parametersChanged` and calls the setter delegates
(`SetNote` etc.) — it does **not** call `IBuzzMachine.Tick()`.

After `CallTick()`, `ReadWork()` runs the work algorithm which calls
`TickAndWork()` for each machine. Control machines are processed first here
too (`CollectControlMachinesThatCanWork()`).

---

## 9. `ParameterDecl` — available properties

```csharp
public class ParameterDecl : Attribute
{
    public string Name         { get; set; }
    public string Description  { get; set; }
    public int    MinValue     { get; set; }
    public int    MaxValue     { get; set; }
    public object DefValue     { get; set; }   // NOTE: object, not int
    public float  ResponseTime { get; set; }
    public Transformations Transformation     { get; set; }
    public int    TransformUnityValue         { get; set; }
    public float  TransformMin { get; set; }
    public float  TransformMax { get; set; }
    public Descriptors ValueDescriptor        { get; set; }
    public string[] ValueDescriptions         { get; set; }
    public int    DecimalDigitCount           { get; set; }   // default 1
    public bool   IsStateless  { get; set; }
    public bool   IsWaveNumber { get; set; }
    public bool   IsTiedToNext { get; set; }
    public bool   IsAscii      { get; set; }
    public string ValidAscii   { get; set; }

    // NO Type property — type inferred from C# parameter type
    // NO NoValue property
}
```

Type inference rules (from `MachineParameter.Create()`):
- `bool` property/param → `ParameterType.Switch` (NoValue=255)
- `Note` param → `ParameterType.Note` (Min=1, Max=156, NoValue=0, all fixed)
- `int`/`float` with `IsWaveNumber=true` → `ParameterType.Byte` with Wave flag
- `int` with `MaxValue > 254` → `ParameterType.Word` (NoValue=65535)
- `int` with `MaxValue <= 254` → `ParameterType.Byte` (NoValue=255)
- `int` with `IsStateless=true, Min=0, Max=255` → `ParameterType.Byte` (NoValue=-1)

**`MaxValue` ceilings — NoValue sentinels are off-limits**

Both Byte and Word parameter types reserve their NoValue as an unsigned
sentinel that ReBuzz writes to `pvalues` to mean "no event this tick".
That sentinel value cannot be used as `MaxValue` — ReBuzz validates the
declaration at load time and throws `Invalid MaxValue` if it is:

| Type | NoValue sentinel | Effective `MaxValue` ceiling |
|------|-----------------|------------------------------|
| `ParameterType.Byte` (`MaxValue ≤ 254`) | 255 | **254** |
| `ParameterType.Word` (`MaxValue > 254`) | 65535 | **65534** |

```csharp
// WRONG — 255 is NoValue for Byte; throws "Invalid MaxValue" at load
[ParameterDecl(MinValue = 0, MaxValue = 255)]
public int MyParam { get; set; }

// CORRECT
[ParameterDecl(MinValue = 0, MaxValue = 254)]
public int MyParam { get; set; }

// WRONG — 65535 is NoValue for Word; throws "Invalid MaxValue" at load
[ParameterDecl(MinValue = 1, MaxValue = 65535)]
public int MyWordParam { get; set; }

// CORRECT
[ParameterDecl(MinValue = 1, MaxValue = 65534)]
public int MyWordParam { get; set; }
```

The error surfaces as:

```
System.Exception: <ParameterName>: Invalid MaxValue
  at ReBuzz.ManagedMachine.MachineParameter.Error(...)
  at ReBuzz.ManagedMachine.MachineParameter.Create(...)
  at ReBuzz.ManagedMachine.ManagedMachineDLL.LoadManagedMachine(...)
```

The machine silently fails to appear in the browser with no further
diagnostic. Check any parameter whose `MaxValue` is exactly 255 or 65535.
65534 is the absolute ceiling for any non-Note integer parameter.

For `ValueDescriptions`, `MinValue` and `MaxValue` are auto-set to
`0` and `descriptions.Length - 1`.

---

## 10. IBuzz.DCWriteLine — debug console output

Available on `IBuzz` (the object behind `host.Machine.Graph.Buzz`):

```csharp
Buzz?.DCWriteLine("[MyMachine] debug message");
Buzz?.DCWriteLine("[MyMachine] error", DCLogLevel.Error);
```

Opens in ReBuzz via menu or `BuzzCommand.DebugConsole`. Use this instead of
`System.IO.File` for in-process debug output — file I/O can fail silently
if the working directory or permissions differ from expectations.

---

## 11. MachineDecl — available properties

```csharp
public class MachineDecl : Attribute
{
    public string Name       { get; set; }
    public string ShortName  { get; set; }
    public string Author     { get; set; }
    public int    InputCount  { get; set; }
    public int    OutputCount { get; set; }
    public int    MaxTracks   { get; set; }
    // NO Flags property
}
```

`MachineInfoFlags.CONTROL_MACHINE` and similar flags are set automatically
by ReBuzz based on the `Work()` signature — not via `MachineDecl`.

---

## 12. Source reference

All of the above was verified against **ReBuzz 1817-preview** source, primarily:

| File | What it covers |
|------|---------------|
| `ReBuzz/ManagedMachine/ManagedMachineDll.cs` | WorkFunctionType detection, MachineInfo setup |
| `ReBuzz/ManagedMachine/ManagedMachineHost.cs` | How setters are called, how Work() is invoked |
| `ReBuzz/ManagedMachine/MachineParameter.cs` | ParameterDecl → ParameterType mapping |
| `ReBuzz/Core/ParameterCore.cs` | SetValue, pvalues, parametersChanged |
| `ReBuzz/Audio/WorkManager.cs` | CallTick(), tick ordering, work algorithm |
| `ReBuzz/MachineManagement/MachineWorkInstance.cs` | TickAndWork, sendControlChangesFlag |
| `ReBuzzGUI/BuzzGUI.Interfaces/MachineInterface/IBuzzMachine.cs` | Interface documentation |
| `ReBuzzGUI/BuzzGUI.Interfaces/MachineInterface/Note.cs` | Note struct |
| `ReBuzzGUI/BuzzGUI.Interfaces/MachineInterface/ParameterDecl.cs` | ParameterDecl attribute |
| `ReBuzzGUI/BuzzGUI.Interfaces/MachineInterface/MachineDecl.cs` | MachineDecl attribute |

---

## 13. Work() call frequency for control machines

**`void Work()` is called once per audio buffer, NOT once per pattern tick.**

At 126 BPM, 4 TPB, 44100 Hz sample rate:
- SamplesPerTick = 44100 × 60 / (126 × 4) ≈ 5250 samples
- Audio buffer ≈ 184 samples → Work() called ~28–29 times per tick

Any countdown in Work() must be gated to fire only once per pattern tick.
Use `IBuzzMachineHost.MasterInfo.PosInTick` to detect tick boundaries:

```csharp
int _prevPosInTick = int.MaxValue;

public void Work()
{
    int pit = host?.MasterInfo?.PosInTick ?? 0;
    bool newTick = pit < _prevPosInTick;   // PosInTick resets to 0 each tick
    _prevPosInTick = pit;

    // ... per-tick logic gated on newTick ...
}
```

`MasterInfo` is on `IBuzzMachineHost` (NOT on `IBuzz`). The comment in the
interface confirms: *"MasterInfo and SubTickInfo are only valid in Work and
parameter setters."*

`MasterInfo` fields: `BeatsPerMin`, `TicksPerBeat`, `SamplesPerSec`,
`SamplesPerTick`, `PosInTick` [0..SamplesPerTick-1], `TicksPerSec`.

Parameter delivery (`HasNewNote` etc.) from `IBuzzMachineHost.Tick()` always
arrives before the first `Work()` call of that tick, so the `newTick` flag
and the `HasNewNote` flag will be true together on the same `Work()` call.

---

## 14. Multi-track simultaneous note delivery — ReBuzz engine bug and workaround

**The bug:** `machine.parametersChanged` is a `Dictionary<IParameter, int>` mapping
parameter → track index. When multiple pattern tracks fire at the same tick (e.g.
voices 0 and 1 both have a note at row 0), the pattern engine calls
`IParameter.SetValue()` for each track in sequence:

```
SetValue(track=0, C4)  → parametersChanged[noteParam] = 0
SetValue(track=1, C#4) → parametersChanged[noteParam] = 1  ← overwrites!
```

`managedMachineHost.Tick()` then iterates `parametersChanged` and only calls
`SetNote(C#4, 1)`. Track 0's note is silently dropped. This affects any managed
machine with more than one active pattern track simultaneously — notes,
velocities, and any other track parameter can all be lost.

**The workaround:** All tracks' pvalues are still valid at the moment `SetNote`
is called (the post-Tick pvalue reset hasn't run yet). Poll them:

```csharp
// Fields:
IParameter _ownNoteParam = null;
System.Collections.Concurrent.ConcurrentDictionary<int,int> _ownNotePValues = null;

// Called lazily on first SetNote — ParameterGroups exist by then:
void TryInitPValues()
{
    try
    {
        if (_ownNoteParam == null)
        {
            var pg = host?.Machine?.ParameterGroups;
            if (pg == null || pg.Count < 3) return;
            _ownNoteParam = pg[2].Parameters.FirstOrDefault(
                p => p?.Type == ParameterType.Note);
        }
        if (_ownNoteParam == null || _ownNotePValues != null) return;

        // pvalues is ConcurrentDictionary<int,int> on ParameterCore — not on IParameter
        var fi = _ownNoteParam.GetType().GetField("pvalues",
            System.Reflection.BindingFlags.NonPublic |
            System.Reflection.BindingFlags.Instance);
        if (fi != null)
            _ownNotePValues = fi.GetValue(_ownNoteParam)
                as System.Collections.Concurrent.ConcurrentDictionary<int,int>;
    }
    catch { }
}

public void SetNote(Note value, int track)
{
    if ((uint)track >= MaxVoices) return;
    _vs[track].PendingNote = value.Value;
    _vs[track].HasNewNote  = true;

    if (_ownNotePValues == null) TryInitPValues();
    if (_ownNotePValues != null)
    {
        int noVal = _ownNoteParam.NoValue;   // 0 for Note type
        for (int t = 0; t < MaxVoices; t++)
        {
            if (t == track) continue;
            int pv;
            if (_ownNotePValues.TryGetValue(t, out pv) && pv != noVal)
            {
                _vs[t].PendingNote = (byte)pv;
                _vs[t].HasNewNote  = true;
            }
        }
    }
}
```

**Critical implementation details:**

- `pvalues` is `ConcurrentDictionary<int,int>` on `ParameterCore` — casting to
  `Dictionary<int,int>` returns null silently. Verify the type against the
  ReBuzz source before caching.
- `TryGetValue` on a `ConcurrentDictionary` is allocation-free and safe on the
  audio thread.
- The field lookup (`GetField`/`GetValue`) happens only once and is cached. Do
  NOT call `Delegate.CreateDelegate` or `MethodInfo.Invoke` on the audio thread —
  these can deadlock if the JIT/runtime type-loading lock is held by the UI thread.
- `_ownNoteParam` must be obtained LAZILY (on first `SetNote` call), not in the
  constructor. `ParameterGroups` are populated by `CreateParameterDelegates()`
  which runs AFTER the constructor. See §15.

---

## 15. ParameterGroups initialization timing — after the constructor

`ManagedMachineHost` creates the machine object in this order:

```csharp
// ManagedMachineHost constructor:
machine = dll.CreateMachine(this);          // 1. Your constructor runs
CreateDelegates();                           // 2. Work() delegate created
parameterDelegates = dll.CreateParameterDelegates(machine); // 3. ParameterGroups built
```

`ParameterGroups` do not exist until step 3. Any code in your constructor that
accesses `host.Machine.ParameterGroups` will find an empty or null collection.
The `Host` property setter (if you define one) may or may not be called after
construction depending on ReBuzz version.

**Consequence:** Parameter-group dependent initialisation must be lazy — deferred
to the first setter or `Work()` call, when the machine is fully set up.

```csharp
// WRONG — ParameterGroups empty at this point
public PedalChordMachine(IBuzzMachineHost host)
{
    this.host = host;
    _ownNoteParam = host.Machine.ParameterGroups[2].Parameters.First(...); // null!
}

// CORRECT — defer until first use
void TryInit()
{
    if (_ownNoteParam != null) return;
    var pg = host?.Machine?.ParameterGroups;
    if (pg?.Count > 2)
        _ownNoteParam = pg[2].Parameters.FirstOrDefault(...);
}
```

---

## 16. Audio thread safety — what is and isn't safe

Operations safe on the audio thread (inside `Work()` or parameter setters):
- Reading/writing plain fields and pre-allocated arrays
- `ConcurrentDictionary.TryGetValue` (allocation-free read)
- Calling a pre-compiled `Func<T,U>` delegate
- `IParameter.SetValue()` and `IMachine.SendControlChanges()` on targets
- `IBuzz.DCWriteLine()` (adds latency/glitches but won't crash)

**Operations that can freeze or deadlock the audio thread:**
- `Delegate.CreateDelegate()` — involves JIT type loading, can contend with
  locks held by the UI thread
- `MethodInfo.Invoke(obj, new object[]{...})` — boxes value types, allocates
  on every call, and may contend with JIT locks
- LINQ on collections that might be modified concurrently by the UI thread
- Any blocking call (`Thread.Sleep`, `Monitor.Enter` with contention, etc.)

**GC pressure:** Even small per-call allocations accumulate. At 126 BPM,
4 TPB: `Work()` is called ~28× per tick, ~500 ticks/second ≈ 14,000 calls/second.
Allocating even 64 bytes per call = ~900 KB/s. Keep the audio hot-path
allocation-free.

---

## 17. Sending notes to multiple target machines simultaneously

When a control machine must fire notes to N different target machines in the
same `Work()` call, each `SetValue()` + `SendControlChanges()` pair is independent
per target. However, calling `SendControlChanges()` immediately after each
`SetValue()` is problematic: the first SCC may cause the audio engine to
schedule an extra `AudioTick` for machine A, but by the time machine B's note
is written and SCC is called, machine A's tick has already consumed its
window and machine B doesn't sound.

**What works:** Call `SetValue()` for ALL targets first, then call
`SendControlChanges()` for each target after. This ensures all values are
written before any machine processes them.

However, for simultaneous triggers that all originate in the same `Work()` call,
ReBuzz appears to process only one SCC-triggered extra tick per audio buffer.
The reliable workaround: fire at most one new note per `Work()` call, letting
the audio thread naturally spread simultaneous triggers across successive
buffers (~4ms apart at typical buffer sizes). This is inaudible and reliable:

```csharp
// In Work():
bool _firedThisWork = false;

// At start of Work():
_firedThisWork = false;

// In HasNewNote processing:
if (vs.HasNewNote)
{
    if (_firedThisWork) continue;  // process in next Work() call
    vs.HasNewNote = false;
    _firedThisWork = true;
    // ... fire note + SCC
}
```

**However:** for managed machines using the §14 multi-track workaround,
simultaneous notes are already spread across voices correctly. In that scenario,
each voice fires in the same `Work()` call and multiple SCC calls do work in
practice for native target machines — tested with 6 simultaneous voices.

---

## 18. The full audio loop sequence per buffer

Understanding the order helps predict when your values are visible to targets:

```
1. UpdatePatternColumnEvents()      ← writes pvalues on all machines for this tick
2. CallTick()
   a. Control machines first:
      managedMachineHost.Tick() → iterates parametersChanged → calls your SetNote etc.
   b. All other machines:
      AudioTick() → reads pvalues → sends to native machine process
3. ReadWork()
   a. CollectControlMachinesThatCanWork → HandleWorkList  ← your Work() runs here
   b. CollectMachinesThatCanWork        → process generators/effects
      For each: TickAndWork():
        if (sendControlChangesFlag) { Tick(forceTick); flag=false; }
        WorkMachine();
4. UpdateNonStateParametersToDefault()  ← resets pvalues to NoValue for non-SCC machines
```

Your `Work()` fires at step 3a. At that point, the target machines have already
run their regular `AudioTick` (step 2b) for this buffer. To deliver a note
within the same buffer, call `SendControlChanges()` on the target — this causes
the extra `Tick` in step 3b's `TickAndWork()`.

---

## 19. EnsureTrackCount — native machine track initialization

Native Buzz generators often start with TrackCount = 0 or 1. `AudioTick()` only
sends track parameters for `i < machine.TrackCount`, so notes to track 0 fail
silently if TrackCount is 0.

`IMachine.TrackCount` may or may not have a setter on the interface in a given
ReBuzz version. Use a try/catch with reflection fallback:

```csharp
void EnsureTrackCount(IMachine m, int needed)
{
    if (m == null || m.TrackCount >= needed) return;
    try { m.TrackCount = needed; return; } catch { }
    try
    {
        var prop = m.GetType().GetProperty("TrackCount");
        if (prop?.CanWrite == true) prop.SetValue(m, needed);
    }
    catch { }
}

// Call before SetValue:
EnsureTrackCount(target, baseTrack + 1);
np.SetValue(baseTrack, BN.FromMidi(midiNote));
target.SendControlChanges();
```

---

## 20. Diagnostics — routing debug via DCWriteLine

Add a right-click command that writes routing diagnostics to the ReBuzz debug
console (accessible via the DebugConsole command):

```csharp
void ShowDiagnostics()
{
    var sb = new System.Text.StringBuilder();
    for (int v = 0; v < MaxVoices; v++)
    {
        string name = _state.Get(v).MachineName;
        if (string.IsNullOrEmpty(name)) continue;
        IMachine   tgt = ResolveTarget(v);
        IParameter np  = tgt != null ? FindNoteParam(tgt) : null;
        sb.AppendFormat(
            "Voice {0}: target=[{1}]  resolved={2}  noteParam={3} (hash={4})  tracks={5}\n",
            v + 1, name,
            tgt != null ? "YES" : "NO - not in Song.Machines",
            np  != null ? "YES" : "NO - not found",
            np?.GetHashCode().ToString() ?? "n/a",
            tgt?.TrackCount ?? -1);
    }
    string msg = sb.ToString();
    try
    {
        IBuzz buzz = Buzz;
        foreach (string line in msg.Split('\n'))
            if (line.Trim().Length > 0)
                buzz?.DCWriteLine("[MyMachine] " + line.Trim());
        Application.Current?.Dispatcher?.BeginInvoke((Action)(() =>
            buzz?.ExecuteCommand(BuzzCommand.DebugConsole)));
    }
    catch { }
}
```

**Key diagnostics to check:**
- `resolved=NO` → machine name in state doesn't match `Song.Machines`
  (case-sensitive `==` comparison in `FirstOrDefault`)
- `noteParam=NO` → `FindNoteParam` search failed for this machine type
- `tracks=0` → target has no tracks; notes silently dropped by `AudioTick`
- Same hash on two voices → both voices share one `IParameter` — any
  `SetValue` from voice 2 overwrites voice 1's value

---

## 21. 64-bit native machines and cross-process IPC — what is safe from the audio thread

ReBuzz runs 64-bit native machines (e.g. Infector, most modern VSTs) in a
separate process (`ReBuzzEngine.exe`). Operations on these machines that appear
to be simple property reads or writes may actually involve inter-process
communication (IPC). Calling them from the audio thread causes a deadlock because
the audio callback is already holding locks that the IPC mechanism needs.

**Known operations that require IPC and must NOT be called from the audio thread:**

- `IMachine.TrackCount = N` (setter) — triggers `CMI_SetNumTracks` in the
  native process. Deadlocks when called during an audio callback.
- `IMachine.TrackCount` (getter) — may also involve IPC; read it on the UI
  thread and cache the result.
- Any method that modifies machine structure rather than just setting parameter
  values.

**Safe from the audio thread:**

- `IParameter.SetValue(track, value)` — writes to a local pvalue store, no IPC.
- `IMachine.SendControlChanges()` — sets a local boolean flag only.
- `IMachine.Name` — read-only string property, locally cached.

**The pattern that caused the deadlock:**

```csharp
// WRONG — called from Work() / FireNote() on the audio thread
void EnsureTrackCount(IMachine m, int needed)
{
    if (m.TrackCount >= needed) return;
    m.TrackCount = needed;  // IPC → deadlock on 64-bit native machines
}
```

**The correct pattern — call on the UI thread only:**

```csharp
// In ResolveCache() which is always called via Dispatcher.BeginInvoke:
void ResolveCache()
{
    _tgt = Buzz?.Song?.Machines?.FirstOrDefault(m => m.Name == _state.TargetMachine);
    _np  = FindNoteParam(_tgt);
    // Pre-expand tracks here, on the UI thread, before audio starts firing notes.
    if (_tgt != null) EnsureTrackCount(_tgt, MaxSlotsNeeded);
}
```

**Symptoms of this deadlock:**
- Song freezes after assigning a machine (playing stops, save hangs).
- No exception in the audio thread — the freeze is silent.
- Only manifests above a certain number of instances (e.g. works fine with 5,
  breaks at 6), because the race condition probability increases with instance count.
- `DCWriteLine` logging shows nothing because the UI thread is also blocked by
  the audio thread deadlock.

**Debugging technique that worked:**
Since both `DCWriteLine` and the ReBuzz debug console require the UI thread,
they are useless when the freeze is a deadlock. Use a thread-safe file logger
instead (`File.AppendAllText` inside a `lock`) — it writes to disk even when
the UI is completely frozen:

```csharp
static readonly object _logLock = new object();
static readonly string _logPath = Path.Combine(Path.GetTempPath(), "MyMachine_debug.txt");
static void Log(string msg)
{
    try
    {
        lock (_logLock)
            File.AppendAllText(_logPath,
                DateTime.Now.ToString("HH:mm:ss.fff") + " " + msg + "\n");
    }
    catch { }
}
```

Add `Log(...)` calls before and after each suspicious operation. After the
freeze, read the file — the last line before the freeze is where execution stopped.

---

## 22. Static fields shared across machine instances

A `static` field on a managed machine class is shared across **all instances**
of that machine type loaded in the same ReBuzz session. This is almost never
what you want.

**Example bug:**

```csharp
// WRONG — all instances share this buffer
static readonly int[] _buildBuf = new int[128];

int[] BuildNotes(...)
{
    // Writes to _buildBuf — race condition if two instances call this
    // simultaneously on the audio thread
    for (int i = 0; i < count; i++) _buildBuf[i] = ...;
    return _buildBuf;
}
```

**Fix:** make scratch buffers instance fields:

```csharp
// Correct — each instance has its own buffer
readonly int[] _buildBuf = new int[128];
```

The bug may not manifest with a single instance or even a small number, but
becomes a race condition if ReBuzz processes multiple control machines
concurrently (multi-threaded audio graph).

---

## 23. WPF dialogs and the ReBuzz dispatcher

`Application.Current.Dispatcher` is the ReBuzz main UI thread. Calling
`ShowDialog()` on the dispatcher thread blocks it with a nested message pump.
With multiple machine instances all trying to show dialogs via `BeginInvoke`,
this can produce deeply nested pump states that deadlock at higher instance counts.

**Wrong pattern:**

```csharp
void OpenSettings()
{
    Application.Current?.Dispatcher?.BeginInvoke((Action)(() =>
    {
        var win = new SettingsWindow(...);
        win.ShowDialog();  // blocks the dispatcher — dangerous with multiple instances
    }));
}
```

**Correct pattern — dedicated STA thread:**

```csharp
void OpenSettings()
{
    // Snapshot any ReBuzz object data on the calling thread (safe)
    var names = Buzz?.Song?.Machines?.Select(m => m.Name).ToList();
    var stateSnap = _state;

    var t = new System.Threading.Thread(() =>
    {
        var win = new SettingsWindow(stateSnap, names);
        if (win.ShowDialog() == true)
        {
            _state = win.Result;
            // Marshal back to dispatcher only for operations that need it
            Application.Current?.Dispatcher?.BeginInvoke((Action)ResolveCache);
        }
    });
    t.SetApartmentState(System.Threading.ApartmentState.STA);
    t.IsBackground = true;
    t.Start();
}
```

The STA thread has its own WPF dispatcher and message pump, completely
independent of ReBuzz's. `ShowDialog()` blocks only that thread. `ResolveCache`
is then marshalled back to the ReBuzz dispatcher for the one operation that
requires it (`Song.Machines` enumeration).

**Note:** `ParameterGroups` are built after the constructor, so any
initialisation that accesses `host?.Machine?.ParameterGroups` must be deferred
until first use (see §15). Do not call `ResolveCache` from the constructor or
the `Host` setter — call it lazily from `OpenSettings` and the `MachineState`
setter instead.

---

## 24. Song.Machines is not safe to enumerate from the audio thread

`Buzz.Song.Machines` is an `ObservableCollection<IMachine>` with WPF change
notifications. Enumerating it with LINQ (`FirstOrDefault`, `Where`, etc.) from
the audio thread while the UI thread adds or removes machines can cause:

- Thread-safety violations (`InvalidOperationException: collection was modified`)
- Deadlocks if the collection holds a lock that the UI thread also needs
- Silent corruption of the LINQ iterator state

This is the cause of the "song freezes when adding the Nth instance" class of
bugs — with N instances all running `Work()`, the probability of hitting the
race during machine import rises until it becomes near-certain.

**Rule:** never access `Song.Machines` from `Work()` or any method called
from the audio thread. Cache the resolved `IMachine` reference on the UI thread
and use the cached value in `Work()`.

```csharp
// Fields — written on UI thread, read on audio thread (reference reads are atomic)
IMachine   _tgt = null;
IParameter _np  = null;

// Called only on UI thread (Dispatcher.BeginInvoke or MachineState setter)
void ResolveCache()
{
    _tgt = Buzz?.Song?.Machines?.FirstOrDefault(m => m.Name == _state.TargetMachine);
    _np  = _tgt != null ? FindNoteParam(_tgt) : null;
    if (_tgt != null) EnsureTrackCount(_tgt, MaxNeeded);  // IPC-safe here
}

// Called on audio thread — only reads cached references
public void Work()
{
    if (_tgt == null || _np == null) return;
    // ... use _tgt and _np safely
}
```

Reference field reads and writes are atomic in .NET (guaranteed for reference
types on 32-bit and 64-bit platforms), so the audio thread reading a reference
that the UI thread is replacing is safe — it will see either the old or the new
value, never a torn pointer.

---

## 25. `IsStateless = true` hides parameters from the rack entirely

`IsStateless` on a `ParameterDecl` does **not** mean "show in the rack but
don't record to the pattern sequencer". It removes the parameter from the
rack UI altogether. The parameter is invisible to the user and cannot be
adjusted interactively.

**Discovered:** Pedal EQ v1.2. Both `LMQ`/`HMQ` (Q per bell band) and the
four solo controls were marked `IsStateless = true` on the mistaken belief
that it would prevent unwanted sequencer writes while keeping the controls
visible. All six parameters disappeared from the rack. The fix was to remove
`IsStateless` from every parameter that needs to be user-adjustable.

```csharp
// WRONG — parameter disappears from the rack entirely
[ParameterDecl(
    Name              = "LM Q",
    MinValue          = 0,
    MaxValue          = 16,
    DefValue          = 6,
    IsStateless       = true,   // ← hides it
    ValueDescriptions = new[] { "0.3", "0.4", ... })]
public int LMQ { get; set; } = 6;

// CORRECT — visible and adjustable in the rack
[ParameterDecl(
    Name              = "LM Q",
    MinValue          = 0,
    MaxValue          = 16,
    DefValue          = 6,
    ValueDescriptions = new[] { "0.3", "0.4", ... })]
public int LMQ { get; set; } = 6;
```

**Current known use:** there is no documented mechanism for "show in rack but
do not record automation to the pattern". If you need a parameter that is
user-adjustable but not automatable, simply leave `IsStateless` off and
accept that it appears in the pattern editor. `IsStateless` should only be
used when the parameter is genuinely not intended for user interaction at all.

---

## 26. Machine GUI — `IMachineGUIFactory` discovery pattern

Verified against ReBuzz 1819-preview source (`MachineDLL.cs`,
`ParameterWindowVM.cs`, `IMachineGUIFactory.cs`, `IMachineGUI.cs`).

### 26.1 How ReBuzz finds and displays a machine GUI

When the parameter window opens for a machine, `MachineDLL.GUIFactory` scans
the machine's assembly for an exported type implementing
`BuzzGUI.Interfaces.IMachineGUIFactory`. If found, it instantiates the
factory and calls `CreateGUI(host)`. The returned `IMachineGUI` is cast as
`FrameworkElement` and embedded at the **top of the parameter window**
(`ParameterWindowVM.EmbeddedGUI`).

```
Machine DLL assembly
  └── PedalEQGuiFactory : IMachineGUIFactory   ← discovered by assembly scan
        CreateGUI(host) → PedalEQGui            ← instantiated here
                                                   Machine = machine set by ParameterWindowVM
```

The scanning code (from `MachineDLL.cs`):
```csharp
foreach (var type in assembly.GetExportedTypes())
{
    if (type.GetInterface("BuzzGUI.Interfaces.IMachineGUIFactory") != null)
        return guiFactory = Activator.CreateInstance(type) as IMachineGUIFactory;
}
```

### 26.2 Required interfaces and attributes

**`IMachineGUIFactory`** (in `BuzzGUI.Interfaces`):
```csharp
public interface IMachineGUIFactory
{
    IMachineGUI CreateGUI(IMachineGUIHost host);
}
```

**`[MachineGUIFactoryDecl]`** attribute (in `BuzzGUI.Interfaces`):
```csharp
[AttributeUsage(AttributeTargets.Class)]
public class MachineGUIFactoryDecl : Attribute
{
    public bool PreferWindowedGUI;   // false = embedded in param window (default)
    public bool IsGUIResizable;
    public bool UseThemeStyles;
}
```

**`IMachineGUI`** (in `BuzzGUI.Interfaces`) — has **one member only**:
```csharp
public interface IMachineGUI
{
    IMachine Machine { get; set; }   // set by ParameterWindowVM after CreateGUI()
}
```

There is **no `Close` event** on `IMachineGUI`. Any such event in older
examples is incorrect.

**`IMachineGUIHost`** (in `BuzzGUI.Interfaces`):
```csharp
public interface IMachineGUIHost
{
    void DoAction(IAction a);
}
```

**`IMachine.ManagedMachine`** — returns the `IBuzzMachine` instance. Cast
this to your concrete machine class inside the `Machine` setter to get a
direct reference for reading parameter properties:

```csharp
public IMachine Machine
{
    get => _machine;
    set
    {
        _machine = value;
        _eq = value?.ManagedMachine as PedalEQMachine;
    }
}
```

### 26.3 Minimal correct implementation

```csharp
// ── Factory ───────────────────────────────────────────────────────────────
// PreferWindowedGUI = false (default) → GUI appears embedded in parameter
// window above the sliders. Set true for a separate floating window.

[MachineGUIFactoryDecl(PreferWindowedGUI = false)]
public class MyMachineGuiFactory : IMachineGUIFactory
{
    public IMachineGUI CreateGUI(IMachineGUIHost host) => new MyMachineGui();
}

// ── GUI ───────────────────────────────────────────────────────────────────
// Must be a FrameworkElement (e.g. UserControl) as well as IMachineGUI,
// because ParameterWindowVM casts it: EmbeddedGUI = MachineGUI as FrameworkElement

public class MyMachineGui : UserControl, IMachineGUI
{
    MyMachine   _machine;
    IMachine    _iMachine;
    DispatcherTimer _timer;

    public IMachine Machine
    {
        get => _iMachine;
        set { _iMachine = value; _machine = value?.ManagedMachine as MyMachine; }
    }

    public MyMachineGui()
    {
        // Build UI in code (no XAML needed) ...

        _timer = new DispatcherTimer { Interval = TimeSpan.FromMilliseconds(100) };
        _timer.Tick += (_, __) => Refresh();
        _timer.Start();
        Unloaded += (_, __) => _timer.Stop();
    }

    void Refresh()
    {
        if (_machine == null) return;
        // Read _machine.MyProperty directly — int reads are atomic on x64,
        // safe to call on the UI thread while audio thread writes them.
    }
}
```

### 26.4 Reading machine state in the GUI

**Preferred:** read properties directly from the cast `IBuzzMachine` reference
(`_machine.LSGain` etc.). Integer property reads are atomic on x64 — safe
across the audio/UI thread boundary for display purposes with no locking.

**Avoid:** `IParameter.GetValue(track)` returns the parameter's
*last-played* value, not the current pvalue slot. For global parameters set
via the rack this is usually correct, but it is an indirect and fragile path
compared to reading the property getter directly.

### 26.5 csproj requirement

Any machine DLL that includes a GUI class must add `<UseWPF>true</UseWPF>`
to its `<PropertyGroup>`. Without it the .NET SDK does not reference the WPF
assemblies and all WPF types (`UserControl`, `TextBlock`, `Brush`,
`DispatcherTimer`, etc.) are invisible to the compiler. See Build notes §3.

### 26.6 Things that do NOT work

- **`[MachineGUI(typeof(MyGui))]` attribute on the machine class** — this
  attribute does not exist in `BuzzGUI.Interfaces` or `Buzz.MachineInterface`.
- **`public UserControl GUI` property on the machine class** — ReBuzz does
  not check for this via reflection.
- **Scanning for `IMachineGUI` directly** — ReBuzz scans for
  `IMachineGUIFactory`, not `IMachineGUI`. A class that only implements
  `IMachineGUI` without a factory will not be discovered.

---

## 27. Transport stop — `Work()` keeps running but setters don't

When the user presses ReBuzz's Stop button, the pattern engine pauses (no more
setter calls fire) but `Work()` keeps being invoked — the audio graph stays
running so downstream tails and effects can finish rendering. Any voice with
non-zero sustain rings on indefinitely until the user manually triggers a
note-off, which they can't do from a stopped pattern.

**The fix:** poll `IBuzz.Playing` at the top of `Work()` and on the falling
edge force every active voice into a fast fade. Don't reuse the user's Release
time — for a long-release patch (multi-second pad) the "voice fades over 8
seconds after Stop" feels broken even if it's technically following the
envelope. A fixed ~5 ms fade overrides the user setting cleanly.

```csharp
bool _wasPlaying;

public bool Work(/* ... */)
{
    // ... drain pending pattern events first ...

    int sr = host?.MasterInfo?.SamplesPerSec ?? 48000;

    bool nowPlaying = _wasPlaying;
    try { nowPlaying = host?.Machine?.Graph?.Buzz?.Playing ?? false; }
    catch { /* keep previous value — never break audio on a poll glitch */ }
    if (_wasPlaying && !nowPlaying) ForceFadeAllVoices(sr);
    _wasPlaying = nowPlaying;

    // ... rest of Work() ...
}
```

This was first documented as a tracker-specific concern (Pedal Tracker §3) but
applies to every synth voice with sustain — promoted here to make sure new
synths get it from the start.

### 27.1 Implementation choices for the fade itself

- **Per-envelope forced-release flag.** Add a `ForcedRelease(sr)` method to
  the envelope that sets stage to Release with a fixed ~5 ms coef, bypassing
  the user's Release setting. NoteOn clears the flag so a fresh Play after
  Stop attacks normally. Cleanest when the voice already has one or more
  ADSRs. This is what Pedal invFFT does.
- **Anti-click target-gain ramp.** If the voice already has a separate
  click-prevention envelope ramping `CurrentGain → TargetGain` per sample
  (the Pedal Tracker pattern), set `TargetGain=0` on Stop and let that ramp
  handle the fade. Cheaper if the machinery already exists.
- **HardReset.** Snap the envelope to Idle, output silence immediately.
  Simplest, but produces an audible click on whatever amplitude the
  OLA/voice buffers happen to hold at the moment of Stop. Not recommended
  unless the voice's amplitude is guaranteed to be near zero by other means.

5 ms is a good default fade duration. 1–2 ms can click on high-amplitude
signals; 10 ms is still imperceptible as a fade. If clicks persist at 5 ms,
the fade target is fighting an OLA-tail or filter ringing elsewhere —
extending the fade to 10 ms is safe.

### 27.2 Thread-safety of the polling read

`IBuzz.Playing` is a plain bool getter, locally cached in ReBuzz with no IPC —
safe to call on the audio thread. The `try/catch` around the chain is
defensive: a null `host`, `Graph`, or `Buzz` during init or teardown shouldn't
cascade into an audio-thread exception. On any failure, keep the previous
value (`nowPlaying = _wasPlaying`) so the falling-edge detector doesn't fire
spuriously.

### 27.3 Initialisation and edge-case behaviour

`_wasPlaying` defaults to `false`. The cases worth verifying:

- **Machine loaded into an already-playing song.** First `Work()` sees
  `nowPlaying=true, _wasPlaying=false` — no falling edge fires — and proceeds
  normally. The next Stop press triggers the fade as expected.
- **Machine loaded into a stopped song.** Both values stay `false`; no fade
  fires until a Play→Stop transition happens.
- **Rapid Play→Stop with no notes between.** Falling edge still fires, but
  with no active envelopes there's nothing to fade. `ForcedRelease` should be
  idempotent against Idle envelopes (early-return).

No special init logic is needed; the falling-edge detector handles all states
correctly from `_wasPlaying = false`.

### 27.4 Where to place the polling

At the top of `Work()`, *after* draining pending pattern events but *before*
updating envelope coefficients. The order matters:

1. **Pending events first.** A note-on right before Stop should still trigger;
   the falling edge then fades it within the same buffer. Reversing the order
   would force-release the still-pending note before it ever sounded.
2. **Falling-edge check second.** Captures the Stop and triggers forced
   release.
3. **Coef updates third.** `UpdateCoefs` is dirty-checked against user
   parameters; the forced-release coef should be held in a separate field so
   coef updates don't overwrite it.

Reversing any of the steps introduces edge-case bugs.

---

## 28. Parameter `Name` and `Description` go through XML/XAML — avoid special characters

ReBuzz serializes machine state to XML (preset bundles, song saves) and
passes parameter descriptions to XAML for the rack tooltip layer. Special
characters in `Name` or `Description` that have meaning in those formats
can wedge the application at the moment they're parsed — typically the
first interaction that touches the parameter (transport play, parameter
window open, save). The machine still loads cleanly because the
declarations themselves don't pass through the broken paths.

**Discovered:** Pedal invFFT v0.3 used `Name = "Even/Odd"` and a Tilt
description containing `<` and `>` (e.g. `<64 boosts lows, >64 boosts
highs`). Build succeeded, the machine appeared in the browser and on
the rack, but ReBuzz hung on the first attempt to play. Renaming to
`"Balance"` and rewording the description without angle brackets
resolved it.

**Characters to avoid:**

| Where         | Char(s)              | Reason                              |
|---------------|----------------------|-------------------------------------|
| `Name`        | `/`                  | breaks preset XML key lookup        |
| `Name`        | `<` `>` `&` `"` `'`  | XML special characters              |
| `Description` | `<` `>`              | XAML reads as element-tag delimiters |
| `Description` | `&`                  | XAML entity-reference start         |
| `Description` | `"`                  | XAML attribute delimiter            |

Plain Unicode punctuation that does NOT cause problems in practice:
em-dashes (`—`), en-dashes (`–`), curly quotes, ellipses, degree signs.
These are used in many existing Pedal-series machines without issue.

**Symptoms when one slips through:**
- Machine builds successfully — no compile error.
- Machine loads cleanly and appears in the rack.
- First transport play hangs ReBuzz: audio thread blocks and the UI
  becomes unresponsive together.
- No exception in the debug console; no log entry; the parser blocks
  silently or returns malformed state that breaks downstream code.

**Safe pattern:**
- **Names:** alphanumeric, spaces, hyphens. `"Amp Attack"`, `"Low-Mid Q"`,
  `"Balance"`. If a name needs separation, prefer space or hyphen over
  punctuation.
- **Descriptions:** words instead of symbols. Replace `<64 boosts lows`
  with `64 neutral, lower boosts lows`; replace `0..1 amplitude` with
  `0 to 1 amplitude`; replace `≤ 50%` with `at or below 50 percent`.
  Clarity is unaffected; debugging hours saved.

The cost of being conservative here is zero — descriptions and names
are human-readable strings whose presentation isn't materially worse
for saying "or" instead of `/` or "less than" instead of `<`. The cost
of ignoring the rule is a hang that's hard to attribute because the
build succeeds and the machine loads.

---

## 29. Sample-rate change detection — don't cache `_sr` from a one-shot init

`host.MasterInfo.SamplesPerSec` is reported per-`Work()` call and can change
at runtime — most commonly when the user switches audio devices or changes
ReBuzz's output sample rate during a session. Machines that grab `_sr` once
on first-Work init and then cache it forever will produce **wrong pitch on
every oscillator** and **wrong cutoff on every filter** after the rate
changes, because all `dt` and coefficient computations downstream are still
using the old rate.

The symptom is subtle: nothing breaks visibly, audio still plays, but
everything is pitched off by the ratio `oldSr / newSr`. From 48k to 44.1k
that's a flat 1.5 semitones; from 96k to 48k it's an octave high. Easy to
miss in casual testing because the off-pitch is consistent across all
notes — it sounds "in tune with itself".

### 29.1 The wrong pattern

```csharp
bool _initialized;

public bool Work(/* ... */)
{
    if (!_initialized)
    {
        float sr = _host?.MasterInfo?.SamplesPerSec ?? 44100f;
        _voice.Init(sr);   // propagates _sr down to all engines/filters
        _initialized = true;
    }
    // ... uses _sr cached during Init ...
}
```

This handles first-call setup but never re-initialises when the host
changes rate. The cached `_sr` becomes stale silently.

### 29.2 The right pattern

```csharp
float _lastSr;   // 0 = never initialised

public bool Work(/* ... */)
{
    float currentSr = _host?.MasterInfo?.SamplesPerSec ?? 44100f;
    if (MathF.Abs(currentSr - _lastSr) > 0.5f)
    {
        _voice.Init(currentSr);   // re-init with new rate
        _lastSr = currentSr;
    }
    // ... rest of Work ...
}
```

The `> 0.5f` tolerance is a guard against floating-point noise — a host
reporting `44100.001` one tick and `44100.0` the next shouldn't trigger
spurious re-inits. Real device changes always cross integer rate
boundaries (44100 → 48000, etc.), so 0.5 Hz is plenty of margin.

### 29.3 Trade-off — re-init resets state

Calling `Voice.Init(newSr)` typically resets oscillator phases, filter
integrator state, RNG seeds, envelope state. For a rate change mid-note
this means an audible click as everything snaps to its default state.

In practice this is acceptable: rate changes happen at device-switch
time, which is almost always with playback stopped. The state-resetting
branch never fires during a song.

If a particular machine has expensive state that's painful to reset
(e.g., a long convolution tail, an accumulated FFT history), refactor
`Init(sr)` into two methods:

- `SetSampleRate(float sr)` — recomputes `_sr` and any rate-dependent
  coefficients; does NOT touch oscillator phases, envelopes, or
  accumulator state.
- `Reset()` — zeros all per-voice state, called from `NoteOn` or
  explicit voice-reset paths.

Then `Init(sr)` becomes `SetSampleRate(sr) + Reset()` for the
first-time call, and a runtime rate-change calls only
`SetSampleRate(currentSr)`. The detection pattern at the top of `Work`
stays the same; only the engine-side method it calls changes.

### 29.4 What this affects across an engine bank

Every component with a sample-rate-derived value needs to pick up rate
changes. In the Pedal Plaits architecture (the build that surfaced
this), that's:

| Component         | Sample-rate dependencies                              |
|-------------------|-------------------------------------------------------|
| Oscillator engine | `dt = freq / sr` per partial / per oscillator         |
| Filter (SVF/TPT)  | `f = tan(π·fc/sr)` or `2·sin(π·fc/sr)` coefficient    |
| LPG               | One-pole `a = dt / (rc + dt)`, `dt = 1/sr`            |
| Decay envelope    | `coef = exp(-1 / (tau · sr))`                         |
| Noise clock       | `clockDt = clockHz / sr`                              |

Every engine that touches any of these computes them from the
host-provided `_sr` at control rate (once per Render call). With the
`_lastSr` re-init pattern in place, those computations always see the
current rate without any per-engine plumbing.

### 29.5 The general principle — `MasterInfo` is runtime-variable

The same principle applies to tempo. If a machine derives anything from
`MasterInfo.SamplesPerTick` (an LFO whose rate is tempo-locked, a delay
time in ticks, etc.), don't cache that value at init either — read it
from `MasterInfo` per Render. The cost is one struct field access per
block, negligible compared to the cost of getting it wrong.

The signature pattern: **anything from `MasterInfo` is a runtime
variable, not a setup constant.** Treat it like a parameter — read on
demand, never cache across calls without an explicit invalidation
check.

### 29.6 Discovered

Pedal Plaits v0.3 → v0.4 — the WaveshapingEngine produced wrong pitch
after the user switched audio device from 48 kHz to 44.1 kHz mid-session.
Investigation showed every engine in the build cached `_sr` from a single
`Voice.Init(sr)` call on first Work, so none of them noticed the host's
rate change. The §29.2 pattern (with the 0.5 Hz tolerance) fixed it
across all engines without per-engine code changes.

---

## 30. Denormal floats stall the CPU on decaying-state DSP

Floating-point values below ~1.18e-38 (the IEEE-754 denormal/subnormal
range) are not handled by the FPU hardware on x86 — they trap to CPU
microcode. Each FP op on a denormal value runs **50–100× slower** than
on a normal float. In an audio inner loop processing 44 100 samples per
second, this can spike the engine's CPU from <1% to >50% — and the
slowdown is sticky: it persists for as long as the state values remain
in the denormal range, which can be hundreds of milliseconds or longer.

This bites any DSP component whose state decays exponentially toward
zero without being continuously re-excited. The pattern is:

1. A state variable is set to a non-zero value (an impulse, a NoteOn,
   a parameter change).
2. The state decays in subsequent samples via repeated multiplication
   by a coefficient < 1.
3. After enough samples, the state crosses ~1e-38 into denormal range.
4. From that point until the state is flushed to exact zero, every
   operation on it traps to microcode.

Common offenders in this project's DSP territory:
- **Bass-drum-style resonators** — SVF / biquad integrators excited by
  an impulse then ringing/decaying with no further input.
- **Pitch envelopes** in percussive engines — they decay toward zero
  unbounded unless the engine explicitly stops advancing them.
- **Modal / Rings-style banks** — a row of high-Q resonators excited
  briefly then left to decay independently.
- **Reverb tank state** — internal delay/feedback variables decay
  toward zero between input bursts, especially during quiet passages.

Pitched/sustained engines (oscillators with bounded phase, filters
with continuous input) are **not** vulnerable — their state never
decays to zero, it oscillates or is constantly re-excited.

### 30.1 The signature symptom

CPU stays elevated long after the audible part of the sound is gone.
Often **pitch-dependent**, because resonators decay at a rate
proportional to their center frequency: high-frequency resonators
reach denormal range faster than low-frequency ones.

The classic user report is *"CPU is fine for low notes, jumps and
stays high for high notes"* — exactly the signature of denormals
in a resonator whose decay time is short enough to cross 1e-38
before the engine's amplitude envelope can cut off processing.

For Pedal Plaits' bass drum specifically (which surfaced this finding),
at C-5 with brightness=1 the SVF state crossed denormal range in
~170 ms, whereas the amp envelope didn't reach its IsSilent threshold
for ~7 s — leaving ~6.8 seconds of microcode-trap-bound CPU per kick.

### 30.2 The fix — flush small values to zero

Add explicit zero-flushes after each state update, with a threshold
well above denormal range and well below audibility:

```csharp
// After SVF state update
_resLp += f * _resBp;
float hp = impulse - _resLp - damp * _resBp;
_resBp += f * hp;

// Denormal flush — must run every sample, not per-block
if (_resBp > -1e-25f && _resBp < 1e-25f) _resBp = 0f;
if (_resLp > -1e-25f && _resLp < 1e-25f) _resLp = 0f;
```

Threshold choice — `1e-25` is roughly:
- 13 orders of magnitude above the denormal threshold (1e-38)
- 25 orders of magnitude below peak audio (1.0, equivalent to ~-500 dB)

So the flush catches values long before they enter the slow-path range
and is inaudible even at extreme analytical/observability levels.
Don't make the threshold too tight: `1e-30` is safely above denormal
arithmetically but uncomfortably close to FP precision limits, and a
margin matters for resonators whose state oscillates around zero (a
near-zero value might be denormal on one sample, non-denormal the
next, oscillating slowly across the boundary).

For positive-only state (envelopes), only one comparison is needed:

```csharp
_pitchEnv *= _pitchCoef;
if (_pitchEnv < 1e-25f) _pitchEnv = 0f;
```

### 30.3 The cost

Two comparisons + one rarely-taken store per sample, per protected
state variable. Modern CPUs predict the not-taken branch well because
state is in normal range for the entire audible portion of the
sound. Real-world cost: ~1 ns per sample per protected variable. For
an SVF with 2 state vars, that's ~0.01% CPU at 44.1 kHz. Cheap
insurance against catastrophic slowdowns.

### 30.4 Where to apply it (and where you don't need to)

The rule: any state variable that **decays toward zero unboundedly
and isn't continuously re-excited** needs denormal protection on
every sample of its update.

Apply:
- ✓ SVF / biquad integrators after impulse excitation (bass drum,
  snare modes, modal banks, plate reverbs)
- ✓ Pitch envelopes on percussive engines
- ✓ Reverb tank delay-line writes and feedback paths
- ✓ Amp envelopes — *if* their decay isn't gated by a IsSilent /
  IsActive check that stops the engine well above denormal range.
  (Pedal Plaits' amp env doesn't need protection because IsSilent
  fires at 1e-5, halting engine execution before denormal range can
  be reached. If an engine doesn't stop processing on env-silent,
  protect it.)

Skip:
- ✗ Oscillator phase accumulators — bounded in [0, 1], never approach
  denormal range.
- ✗ Filter state in continuously-excited filters (Pedal Plaits'
  filtered-noise engine, sustaining synth filters, compressor
  envelope detectors with constant program material). The constant
  input keeps the state out of denormal range.
- ✗ State variables you explicitly clamp to zero on every sample
  (e.g., granular engines that do `if (env <= 0f) env = 0f;` already).

### 30.5 Alternatives considered

**SSE flush-to-zero (FTZ) and denormals-are-zero (DAZ) flags.** These
MXCSR bits make the CPU treat denormals as zero globally, eliminating
the slowdown. Available via `System.Runtime.Intrinsics.X86.Sse.SetFlushToZeroMode`
in .NET. *Not used* because:

1. Settings are per-thread, and the audio thread isn't ours to
   configure reliably — ReBuzz creates and destroys audio threads
   on its own schedule.
2. The setting is global to all DSP code on the thread, which can
   affect plugins or hosted code that expects denormals to be handled
   correctly (some compressors deliberately let denormals occur to
   measure RMS over very small windows; some Class A audio designs
   rely on near-zero behaviour).
3. Explicit flushes are localized, predictable, and obviously visible
   in the code where the problem could occur.

**Tiny DC offset trick.** The classic
`state += 1e-25f; state -= 1e-25f;` pattern, which keeps state
slightly above zero. Doesn't work reliably under aggressive
optimization — the JIT may fold the two operations into a no-op
(JIT optimizations can be more aggressive than typical C/C++
compilers because the runtime knows everything). Explicit
comparison-flush is more robust.

### 30.6 Discovered

Pedal Plaits v0.5 → v0.6 — the BassDrumEngine's CPU usage stayed
elevated for several seconds after triggering a kick, but only at
higher notes (C-4 and above). Investigation showed that at higher
resonance frequencies the Chamberlin SVF state decayed into denormal
range (~170 ms at C-5 with brightness=1) before the amplitude envelope
crossed the IsSilent threshold (~7 s at MORPH=80). During the gap,
every SVF update operation trapped to microcode at ~50× normal cost.
Adding explicit `1e-25f` flushes to the SVF integrators (`_resLp`,
`_resBp`) and to the pitch envelope (`_pitchEnv`) brought CPU back
to baseline immediately when the audible tail finished.

The same protection should be added preemptively to SnareDrum, HiHat,
and any future modal-style engine (Rings clone, plate reverb, etc.)
before their DSP is ported, since they share the same architectural
pattern of impulse-excited resonator decay.

---

## 31. Chamberlin SVF impulse response peaks at f, not f·Q

The Chamberlin state-variable filter — cheap LP/BP/HP-from-one-pair-of-
integrators SVF used throughout this project's percussive engines — has
a well-known gain characteristic for **sinusoidal** input at resonance:
the BP output amplitude is approximately Q (= 1/damp). The mental model
that follows from this — *"high Q means loud ringing"* — is **wrong for
impulse excitation**.

After a unit impulse into the standard Chamberlin SVF:

```csharp
_lp += f * _bp;
hp = input - _lp - damp * _bp;
_bp += f * hp;
```

with `f = 2·sin(π·fc/sr)` and `damp = 1/Q`:

- **Peak BP amplitude** is approximately `f` (which is tiny at audio
  rates — ~0.02 at C-3, ~0.005 at C-2)
- **Decay time constant** is approximately `Q / (π·fc)` seconds

Q controls how *long* the resonator rings, not how *loud* its peak gets.
Both quantities together determine the total integrated energy in the
ring (Q·f² ∝ Q·fc²), but the **instantaneous** output during the ring
stays small at all times.

### 31.1 The numbers

Concrete values at 44.1 kHz with Q=30 (typical snare-shell setting):

| Note  | fc (Hz) | f       | Peak _bp | Ring τ   |
|-------|---------|---------|----------|----------|
| C-2   | 65      | 0.00926 | ~0.01    | 147 ms   |
| C-3   | 131     | 0.01853 | ~0.02    | 73 ms    |
| C-4   | 262     | 0.0370  | ~0.04    | 37 ms    |
| C-5   | 523     | 0.0741  | ~0.07    | 18 ms    |

At a typical snare body pitch around 180 Hz, peak BP amplitude is
~0.025 — roughly **30× quieter** than a comparable BP filter
continuously excited by white noise (which sits at RMS ~0.4 for a
~5 kHz BPF on ±1 white noise).

### 31.2 When this bites you

The trap is impulse-excited SVFs **mixed with another continuously-
excited DSP path** in the same output. The continuously-excited path
has normal-size amplitude (~0.2–0.5). The impulse-excited path is
invisibly tiny (~0.02). The mix is dominated by the loud component, and
any control that affects only the quiet path appears to have no
audible effect.

The signature symptom: *"control X doesn't do anything"* when control
X drives the impulse-excited path and another path dominates the mix.
Most painfully, this includes the **note column itself** when the
SVFs are tuned to track pitch but the dominating noise path is at a
fixed cutoff.

### 31.3 When this does NOT bite you

Filters with continuous input — BP filters on noise sources, sustained
oscillators feeding LP/BP/HP, voice LPGs processing engine output —
are immune. The input is always at normal amplitude, the filter's
steady-state gain is Q at resonance, and the output sits at sensible
audio levels.

Single-output percussive engines that don't mix the resonator with
anything else also don't see it. The Pedal Plaits bass drum is a
case: its SVF output is the *only* component in the OUT path, so the
quiet absolute amplitude is just compensated by the overdrive gain
stage downstream (`MathF.Tanh(_resBp * overdrive)`) and nobody
notices.

### 31.4 The fix — normalize by 1/f

When mixing an impulse-excited SVF with a continuously-excited path,
divide the SVF output by `f` (the coefficient that determines its
peak amplitude):

```csharp
// In per-block setup, alongside the filter coefficient:
float f = 2f * MathF.Sin(MathF.PI * fc / sr);
float tonalGain = 1f / MathF.Max(f, 1e-6f);   // peak normalised to ~1.0

// In the per-sample loop:
float tonal = _bp * tonalGain * ampEnv;
float outSample = tonalMix * tonal + noiseMix * noiseBp;
```

The `MathF.Max(f, 1e-6f)` guards against divide-by-zero if fc happens
to be very low (which won't happen with a real note frequency, but is
cheap insurance).

Output now sits at peak ~1.0 at any pitch — a "normal" audio-level
signal that mixes naturally with other normal-level signals. The gain
is recomputed once per block at control rate, alongside the filter
coefficient — no per-sample cost.

### 31.5 Alternative — boost the impulse instead

A symmetric way to read the same problem: the impulse is too small.
Instead of normalizing the output, scale the input:

```csharp
float impulse = _impulseCounter > 0 ? (1f / f) : 0f;
```

Equivalent end result, marginally cheaper (one division per trigger
instead of one per block). The choice is stylistic. Output-side
normalisation is easier to reason about — *"output sits at ~1.0, like
everything else"* — and makes the dependence on the filter
coefficient explicit at the mix point. Input-side normalisation is
closer to physical intuition — *"hit the resonator hard enough that
it rings audibly"*.

### 31.6 Where this applies in this project

Audit the codebase for this pattern: any impulse-excited Chamberlin
SVF whose output is mixed with another path before reaching the user.

- **Pedal Plaits BassDrumEngine**: single output path, no mix → not
  affected.
- **Pedal Plaits SnareDrumEngine**: SVF tonal mixed with BP noise →
  affected. Fixed in v0.7.
- **Pedal Plaits HiHatEngine**: 6-square sum + noise → HP filter →
  output. HP filter is *continuously* excited by the input mix (not
  impulse-excited), so not affected.
- **Pedal Plaits FilteredNoiseEngine**: SVF continuously excited by
  variable-clock noise, not impulse-excited → not affected.

The pattern to watch for in any future engine: an SVF whose `input`
sample is mostly zero with occasional impulses, while its output is
summed into a final mix alongside other audio paths. That combination
is the trigger condition.

### 31.7 Discovered

Pedal Plaits v0.7 — the SnareDrumEngine's note column was inaudible,
even though `_baseHz` was tracking correctly through `NoteOn` and into
the resonator coefficients. Investigation showed the two SVF
resonators were producing tonal output at peaks of 0.02–0.07 across
the playable note range, while the continuously-excited 5 kHz noise
BP output was sitting at RMS ~0.4. The noise component dominated the
perceived sound entirely, and since the noise BPF was at a fixed
frequency, no note-dependent character could emerge.

Adding `tonalGain = 1/f` per-block, applied to the resonator sum,
brought the tonal level to ~1.0 peak and made the snare's body pitch
clearly audible across the keyboard. The output mix scale was lowered
from 0.6 to 0.3 to keep the combined level within headroom (the
boosted tonal can reach ~2.2 peak at TIMBRE=1 with both resonators
contributing).

The same `1/f` normalisation should be added preemptively to any
future modal-style engine (Rings clone, plate reverb tank readouts
mixed with dry signal, drum machines with multi-component shells)
where impulse-excited high-Q resonators sit in a mix with
continuously-excited paths.

---

## 32. Per-Work parameter smoothing — eliminating zipper noise on modulated parameters

ReBuzz delivers parameter changes per buffer. The naïve render pattern
is to snapshot parameter values at the top of `Work()` and use them as
constants for the buffer's entire render. That pattern works fine when
parameters change only at human pace — user knob movement or pattern
automation at row boundaries. Both are slow enough that buffer-rate
parameter updates aren't audibly stepped.

Once external modulator machines come into play — LFO machines,
envelope-follower machines, MIDI-CC bridges, modulation-matrix machines
that write to your parameters at tick rate — the snapshot-then-hold
pattern breaks audibly. Each buffer holds the previous source value
flat for n samples, then jumps to the new value at the next buffer
boundary. The discontinuity creates zipper noise — a sharp
clicking/rasping artefact whose audibility scales with how aggressively
the parameter is being modulated, and with how short the host's buffer
size is.

Pedal Plaits v1.2 introduces a two-stage smoothing pattern that fixes
this generically for any continuous parameter. The pattern is broadly
applicable — any managed machine whose parameters are likely targets
for external modulation should consider it.

### 32.1 The two-stage design

```
                user-set / pattern automation / external modulator
                                │
                                ▼ (instantaneous, per-Work)
                       target value (= current property value)
                                │
                                ▼ (exponential approach toward target)
                  smoothed value, end-of-Work
                                │
                                ▼ (linear interpolation between
                                    prev-Work-end and this-Work-end)
                      per-sub-block param value
                                │
                                ▼
                         engine.Render(...)
```

**Stage 1 — Per-Work exponential smoothing toward target.** Each
`Work()` reads the current parameter property values as targets. The
persistent smoothed-state field moves toward those targets by a
coefficient derived from a time constant:

```csharp
const float SMOOTH_SECS = 0.010f;   // 10 ms tau
float smoothCoef = 1f - MathF.Exp(-n / (SMOOTH_SECS * sr));
float hmStart  = _smHarmonics;
float hmEnd    = hmStart + (harmonicsTarget - hmStart) * smoothCoef;
_smHarmonics   = hmEnd;             // persist for next Work
```

The `1 - exp(-n/(T·sr))` form gives mathematically exact exponential
smoothing regardless of buffer size — total smoothing time stays at
10 ms whether the host runs 64-sample or 1024-sample buffers. A common
shortcut is `smoothCoef = n / (T*sr)` (linear approximation, fine for
small n/sr ratios), but the exact form costs one `MathF.Exp` per Work
and removes the buffer-size dependency.

**Stage 2 — Per-sub-block linear interpolation.** With `hmEnd`
representing the *end-of-buffer* smoothed value, `hmStart` is whatever
the smoothed state was at the end of the previous Work. Per sub-block
(the engine's control-rate block of typically 8-32 samples),
interpolate linearly between buffer-start and buffer-end values:

```csharp
int i = 0;
while (i < n) {
    int blk = Math.Min(BLOCK_SIZE, n - i);
    float t = (i + blk * 0.5f) / n;          // block-midpoint progress 0..1
    pBlock.Harmonics = hmStart + (hmEnd - hmStart) * t;
    // ... fill other smoothed fields the same way ...
    engine.Render(scratchOut, scratchAux, blk, in pBlock);
    i += blk;
}
```

Stage 2 is what actually eliminates zipper noise. Stage 1 alone would
still have discontinuities at buffer boundaries — just smaller ones.
Stage 2 ensures the value transitions linearly through the buffer
rather than stepping at sub-block boundaries.

### 32.2 Why a single mechanism isn't sufficient

Three obvious one-stage alternatives, and why each falls short:

**Alternative A — Just lerp old-snapshot to new-snapshot per sub-block,
no exponential.** Transitions complete in exactly one buffer regardless
of buffer size. At small buffers (64 samples = 1.5 ms at 44.1 kHz) this
is still audibly fast — modulation source changes still produce
discontinuities, just spread over a shorter window. At large buffers
(1024+ samples = 23 ms) this over-smooths and adds modulation lag. No
way to tune smoothing time independent of buffer size.

**Alternative B — Just per-Work exponential, no per-sub-block lerp.**
The smoothed value updates once per Work, then holds constant for the
whole buffer. Reduces the size of buffer-boundary jumps (each is now
`smoothCoef × (target - previous)` instead of `target - previous`) but
doesn't eliminate them. Zipper noise persists at reduced amplitude.

**Alternative C — Per-sample exponential filter on the parameter.**
Mathematically clean. Expensive: an `exp(-1/(T*sr))` coefficient is
typically pre-computed, but per-sample multiply-add is still ~3 ops per
parameter per sample. For 6 smoothed parameters × 44.1 kHz × 44 voices,
this isn't free.

The two-stage pattern combines the best of all three: tunable smoothing
time independent of buffer size (Stage 1), zipper-free continuous
output (Stage 2), and amortized cost (Stage 1 runs once per Work,
Stage 2 is a single lerp per sub-block).

### 32.3 Lazy initialization

Smoothed-state fields start at 0 by default. If the user has already
set parameters via preset or song load before the first `Work()`, the
smoothed values would start at 0 and ramp toward the user's values
over ~30 ms — producing a fade-in on machine load that's almost-but-
not-quite right.

Lazy init solves this: on the first `Work()` after construction, snap
the smoothed state to current target values rather than smoothing
toward them:

```csharp
bool _smoothInit;

// At top of Work(), after target computation:
if (!_smoothInit) {
    _smHarmonics = harmonicsTarget;
    _smTimbre    = timbreTarget;
    // ... all smoothed params
    _smoothInit  = true;
}
```

This must be done per-machine-instance (not as a class default)
because ReBuzz can construct multiple instances and each needs its own
init from its own loaded parameter values. The constructor doesn't
have safe access to parameter values yet (they're loaded after the
constructor by `CreateParameterDelegates`, see §15), so the lazy-init
flag is the cleanest pattern.

### 32.4 What to smooth, what to leave raw

Not every parameter benefits from smoothing. The categorisation is by
what the parameter *means* musically:

**Smooth these:** continuous "character" parameters that engines read
every block (e.g. Harmonics, Timbre, Morph, filter cutoff/resonance,
volume, envelope shape coefficients, decay times). These are the
parameters external modulators are most likely to target. Smoothing
produces clean continuous response.

**Leave raw:** discrete enums (engine selector, waveform selector,
target selector — any parameter where intermediate values are
meaningless). NoteOn-only parameters (oscillator phase reset, S&H
trigger). Pitch parameters where semitone steps should be instantaneous
on note triggers, not glide (otherwise pattern transposition unintendedly
becomes portamento).

**The NoteOn-snapshot exception.** Even smoothed parameters need their
*raw* (unsmoothed) values at NoteOn time when initialising voice state.
The DecayEnv's initial value and the LPG's strike level should reflect
what the user *intends* at note trigger, not whatever the smoothing
state happens to be passing through mid-ramp. Pedal Plaits handles this
by keeping `ParamsSnapshot()` returning raw values for the NoteOn code
path, while the ongoing render uses smoothed values:

```csharp
if (_hasNoteOn) {
    _voice.NoteOn(midi, _velocity, ParamsSnapshot());   // raw values
}
// ... ongoing render below uses smoothed values
```

This is subtle but important. Without this split, rapid note triggering
during a smoothing ramp gets envelope shapes that don't match the
user's settings — and the user can't easily diagnose why, because the
settings *look* right.

### 32.5 Interaction with other modulation sources

Smoothing applies to the *base parameter value*. Other modulation
sources (velocity routing, modulation matrix offsets) add on top of the
smoothed value, with the final result clamped to the parameter's valid
range:

```csharp
pBlock.Harmonics = Math.Clamp(
    hmStart + (hmEnd - hmStart) * t        // smoothed base
    + velocityOffset,                       // trigger-time modulation
    0f, 1f);                                // clamp
```

Ordering matters. Smoothing is for *base parameter changes* (user knob,
pattern automation, external LFO targeting the param directly).
Trigger-time modulations (velocity routing, key tracking, etc.) are
*instant on note trigger* — smoothing them would defeat their purpose.
Pedal Plaits v1.3 separates these concerns by smoothing the base and
adding trigger-instant offsets after.

The clamp must happen *after* both additions. A common mistake is to
clamp the smoothed value before adding the offset, then add — that
loses information when the offset would push back into range:

```csharp
// WRONG — clamps smoothed first, then adds offset
float hm = Math.Clamp(hmStart + (hmEnd - hmStart) * t, 0f, 1f);
pBlock.Harmonics = hm + velocityOffset;     // can go negative or > 1
```

### 32.6 Cost analysis

Per `Work()` for N parameters smoothed:
- 1 `MathF.Exp` to compute smoothCoef (shared across all params)
- N multiply-adds to compute end-of-buffer values
- N field stores

Per sub-block:
- N lerps for the per-block parameter struct

For Pedal Plaits with 6 smoothed parameters and typical n=184 samples
(15 sub-blocks per Work at BLOCK_SIZE=12), total smoothing cost is
~1 Exp + ~100 lerps per Work — negligible against engine render cost.

The `MathF.Exp` is the most expensive operation in the smoothing path.
At ~30 ns/call in software .NET, that's 30 ns added per Work — invisible
in any practical CPU budget. For aggressive optimisation, the
coefficient could be cached against sample rate (`_lastSr` already
exists per §29) so Exp only runs on sample-rate changes — but this is
premature.

### 32.7 Generalisation

The pattern as documented here uses Plaits' BLOCK_SIZE=12, but the
two-stage approach applies to any block granularity:

- **Per-sample render (no sub-block).** Lerp per-sample between
  buffer-start and buffer-end values. The linear math is unchanged;
  the inner loop runs n times instead of n/BLOCK_SIZE.
- **Variable sub-block** (e.g. SH-101 §2 filter-coefficient cadence).
  The smoothing-side math is unchanged; the consumer reads the lerped
  value at its own cadence.
- **Multi-rate engines** where some parameters are sub-block-rate and
  others are sample-rate. Apply the same lerp at each consumer's
  natural granularity.

In all cases the smoothing time constant is the user-visible parameter
to tune. 10 ms is a good default for "modulation should be responsive
but not zippery." Pads with deliberately slow modulation tolerate
30–50 ms; percussive synths with fast envelope-style modulation want
3–5 ms. The formula is unchanged — only `SMOOTH_SECS` differs.

### 32.8 When NOT to add this

For machines that aren't intended as modulation targets — fixed-
processing utilities (Pedal Muter, Pedal Comp), pattern-event
producers (Pedal Chord), or rare-update machines (Pedal Mixer-style
gain stages) — the smoothing infrastructure is overhead with no
audible benefit. Skip it. Adding smoothing to *every* managed machine
"just in case" bloats the codebase and obscures which machines are
genuinely modulation-friendly.

A reasonable heuristic: if you'd expect a user to wire an LFO machine
into one of your parameters at any point, add smoothing for that
parameter. If the parameter is "set once at song load, forgotten
about" (e.g. limiter ceiling, output routing), skip it.
