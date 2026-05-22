# ReBuzz Managed Machine Development — Pedal Tracker Notes

Source: ReBuzz 1818-preview source code + multi-session debugging of Pedal Tracker
(a managed C# generator modelled on BTDSys Matilde Tracker). These details
extend the original `ReBuzz_ManagedMachine_Notes.md` with everything learned
about pattern-data delivery, sub-tick timing, and Matilde-faithful playback.

---

## 1. Pattern-data delivery has bottlenecks and asymmetries

### 1.1 `parametersChanged` is keyed by parameter, not (parameter, track)

`MachineCore.parametersChanged` is `ConcurrentDictionary<ParameterCore, int>`
where the value is the track index of the *most recent* writer. When two pattern
tracks set the same parameter on the same row, the second write **overwrites**
the first dictionary entry. ReBuzz's `Tick()` then iterates `parametersChanged`
and calls your setter once per parameter — losing all but the last track's
delivery.

This affects **every track parameter**, not just `Note`. Wave, Volume, Cmd,
Arg, Cmd2, Arg2 all share this bottleneck.

### 1.2 `pvalues` reset to `NoValue` post-tick

`MachineWorkInstance.cs:600-615` does this after every successful `Tick()`:

```csharp
foreach (var p in Machine.ParameterGroupsList[2].ParametersList)
    for (int i = 0; i < Machine.TrackCount; i++)
        p.SetPValue(i, p.NoValue);
```

This means `pvalues[track] == value` in `ParameterCore.SetValue` is **false**
on the next tick (NoValue ≠ value), so consecutive identical pattern values
DO fire setters. You don't need to worry about "value didn't change so setter
got suppressed".

### 1.3 The §14 workaround must extend beyond Note

The original notes describe polling `pvalues` from inside `SetNote` to recover
sibling-track notes that `parametersChanged` dropped. **This extends naturally
to every track parameter.** Recover Wave, Volume, and effect columns the same
way. In Pedal Tracker:

```csharp
// Held as IParameter references + ConcurrentDictionary<int,int> pvalue refs
// for every track parameter, looked up once via reflection.
void PollAllPValues(int firedTrack)
{
    if (!EnsurePollFields()) return;
    for (int t = 0; t < MAX_VOICES; t++)
    {
        var v = _voices[t];
        int pv;
        // Note (sibling tracks only — firedTrack got its real call)
        if (t != firedTrack &&
            _ownNotePValues.TryGetValue(t, out pv) &&
            pv != _ownNoteParam.NoValue && pv != 0)
        {
            v.PendingNote = (byte)pv;  v.HasNewNote = true;
        }
        // Wave, Volume, Cmd0/Arg0, Cmd1/Arg1 — recover ANY parameter whose
        // setter didn't fire, guarded by !v.HasNewXxx so a real setter call
        // earlier in this tick takes precedence.
        // ...
    }
}
```

Reflection bootstrap (do once, lazily):

```csharp
static ConcurrentDictionary<int,int> GetPValuesField(IParameter p)
{
    var fi = p.GetType().GetField("pvalues",
        BindingFlags.NonPublic | BindingFlags.Instance);
    return fi?.GetValue(p) as ConcurrentDictionary<int,int>;
}
```

Call `PollAllPValues(track)` from inside your `SetNote` — by the time any
setter fires, the pattern editor has already written to all relevant
`pvalues` for that row, so a single poll captures the entire row's data
regardless of which setter Tick() iteration fires first.

### 1.4 Modern Pattern Editor drops events on the loop-wrap moment

`PlayRecordManager.Play()` at `ModernPatternEditor.NET/PlayRecordManager.cs:137-141`
handles the song-loop wrap with this branch:

```csharp
else if (prevPlayPos > playPosition)        // wrap detected
{
    if (playInfo.PreviousPattern != null)
        PlayPatternEvents(playInfo.PreviousPattern,
                          prevPlayPos, pat.Length * PatternEvent.TimeBase, ...);
    PlayPatternEvents(pat, 0, playPosition, ...);
}
```

At the *exact moment* of wrap, `playPosition` is `LoopStart` (typically `0`),
so `PlayPatternEvents(pat, 0, 0, ...)` is called with an empty range.
`GetEvents(start, end)` filters with `time >= start && time < end`, so events
with `time = 0` (row 0 of the pattern) are NOT dispatched on the wrap tick.
They get delivered on the *next* sub-tick when `playPosition` has advanced past 0.

This causes asymmetric setter delivery on loop wraps: `Note` reliably comes
through (different code path for `ParameterType.Note`), but Wave/Volume/Cmd/Arg
may or may not fire setters on the wrap tick depending on sub-tick boundaries.
**The `pvalues` polling workaround in §1.3 catches this.**

---

## 2. Sub-tick timing and `MasterInfo.PosInTick`

### 2.1 Setters fire on sub-tick boundaries, not only on tick==0

`MachineWorkInstance.Tick()` line 571:

```csharp
if (Machine.Ready && (forceTick ||
    ReBuzzCore.masterInfo.PosInTick == 0 ||
    (engineSettings.SubTickTiming &&
     ReBuzzCore.subTickInfo.PosInSubTick == 0 &&
     Machine.DLL.Info.Version >= MachineManager.BUZZ_MACHINE_INTERFACE_VERSION_42)))
```

If `SubTickTiming` is on (default in modern ReBuzz), setters fire multiple
times per row at sub-tick boundaries. **A sub-tick is not a row.** Same-row
setter calls are common.

### 2.2 `host.MasterInfo` is a snapshot, not live

`MachineManager.UpdateMasterAndSubTickInfoToHost()` snapshots `PosInTick`
once per WorkManager iteration (audio chunk, ≤256 samples). The snapshot
happens at `WorkManager.cs:176`, *before* `CallTick()` which fires setters.

For `GeneratorBlockWork` / `EffectBlockWork` the host's `MasterInfo.PosInTick`
stays at the snapshot value for the entire Work() call. Only the legacy
per-sample `GeneratorWork` and `EffectWork` paths increment it sample-by-sample
inside `ManagedMachineHost.Work()`.

### 2.3 Don't use `PosInTick` for row detection

Comparing `pit < prevPit` looks like it should fire at every tick boundary,
but because of sub-tick timing and chunk-aligned snapshots, the host pit value
seen by your machine can vary unpredictably across the boundary.

**Use `Buzz.Song.PlayPosition` instead.** It increments by exactly 1 per
tick, wraps to `LoopStart` on song-loop wrap, and is consistent across all
sub-ticks within a single row.

```csharp
int songPos = -1;
try { songPos = Buzz?.Song?.PlayPosition ?? -1; } catch { }
bool newRow = songPos != _prevSongPos;
// ... do row-edge work ...
_prevSongPos = songPos;
```

---

## 3. Transport state — `Stop` requires a falling-edge handler

### 3.1 `IBuzz.Playing` is a `bool` set/get

When the user presses Stop, ReBuzz pauses the pattern engine (no more setter
calls) but keeps invoking your machine's `Work()` — the audio graph stays
running so downstream machines can render silence/tails.

If you don't detect this, voices ring on indefinitely after Stop.

### 3.2 Polling pattern

```csharp
bool _wasPlaying;

void CheckTransport()
{
    bool nowPlaying;
    try { nowPlaying = Buzz?.Playing ?? false; }
    catch { nowPlaying = _wasPlaying; }     // never break audio on a poll fail

    if (_wasPlaying && !nowPlaying) StopAllVoices();   // fade everything out
    _wasPlaying = nowPlaying;
}
```

`StopAllVoices()` should set `TargetGain = 0` on every voice and clear pending
triggers, retrigger schedules, note-delay schedules. The existing anti-click
envelope handles the actual fade.

---

## 4. Looping samples and the deferred-trigger machinery

### 4.1 Loop-flagged waves never finish naturally

When `IWave.Flags & WaveFlags.Loop != 0` and the loop region spans the whole
sample, the voice will play forever — `pos >= trackEnd` never trips because
`pos` wraps at `loopEnd`. Successive triggers must replace the still-sounding
voice without a click.

### 4.2 Deferred-trigger pattern

When a new note arrives on a sounding voice:

1. **Stage** the trigger: store `PendingSnap`, `PendingMidi`, `PendingPos`,
   set `HasPendingTrigger = true`, set `TargetGain = 0` (fade out).
2. **Render** ramps `CurrentGain` toward `TargetGain` at `FADE_STEP = 1/64`
   per sample (~1.3 ms at 48 kHz).
3. **Inline-fire**: when `CurrentGain <= NEAR_ZERO && TargetGain == 0`, check
   `HasPendingTrigger`. If true, swap to the staged snapshot, reset position
   to `PendingPos`, set `TargetGain = 1`, refresh local hot-vars from new
   snapshot, continue the same Render() loop.

Total perceived gap: ~2.5 ms (fade-out + fade-in). Inaudible clicks, audible
attack restart.

### 4.3 `PendingPos` MUST capture the sample-offset effect at trigger time

This was the subtle bug behind "kick doesn't sound on loop wrap":

```csharp
// In TriggerNote — deferred path:
v.HasPendingTrigger = true;
v.PendingSnap       = snap;
v.PendingMidi       = midi;
v.PendingPos        = (int)startPos;   // ← carries 09xx sample-offset!
```

If you set `PendingPos = 0` and the user's pattern uses `09 50` (start at
31% through), the deferred re-trigger lands at the silent pre-attack of the
sample. To the listener it sounds like the note didn't fire. The fix is to
honour the captured `startPos` from the trigger-time `09xx` evaluation.

This makes parameter-recovery (§1.3) doubly important: if the `09 xx` setter
got dropped on a loop wrap, the deferred re-trigger plays from position 0
instead of where Matilde-authored patterns expect.

---

## 5. Matilde-faithful semantics

These differ from FastTracker / Protracker conventions and from anything
documented in Buzz machine documentation. Verified against
`github.com/clvn/matilde/Track.cpp` (canonical source) and
`github.com/Buzztrax/buzzmachines/Matilde` (BSD mirror).

### 5.1 Volume column is logarithmic

```cpp
float davolume = float(m_Vals.volume) / 128.0f;
m_fBaseVolume = m_fVolume = davolume * davolume;   // logaritmic volume!
```

In C#:

```csharp
float davol = pendingVolume / 128f;
v.Volume    = davol * davol;
```

| Pattern value | Linear (wrong) | Matilde (correct)   |
|--------------:|---------------:|--------------------:|
| 0x80          | 1.0  (0 dB)    | 1.0    (0 dB)       |
| 0x40          | 0.5  (-6 dB)   | 0.25   (-12 dB)     |
| 0x20          | 0.25 (-12 dB)  | 0.0625 (-24 dB)     |
| 0xFE          | 1.98 (+6 dB)   | 3.94   (+12 dB)     |

dB readout: `40 * log10(value / 128.0)`, not 20*log10.

### 5.2 `09xx` sample offset is proportional

```cpp
m_fSampleOffset = (argument == 0 ? 0x100 : argument);
m_pChannel->m_Resampler.m_iPosition =
    (int)((m_fSampleOffset * m_pSample->GetSampleLength()) / 256.0f);
```

So `09 50` means "start at 0x50/256 = 31.25% through the sample", regardless
of sample length. Not "frame `0x50 * 256`" (that's the Protracker convention,
explicitly NOT what Matilde does). `09 80` = midpoint, `09 00` = end.

```csharp
int arg = v.ActiveArg[sampleOffsetSlot];
double startPos = (arg == 0)
    ? snap.Length - 1
    : (snap.Length * arg) / 256.0;
```

### 5.3 Wave-column change resets track volume to unity

From Matilde's `Tick()`:

```cpp
if (m_Vals.instrument > 0 || !m_oGotInstrument) {
    // ...
    m_fBaseVolume = m_fVolume = 1.0f;     // ← reset on Wave change
    m_fSampleOffset = 0.0f;
    // ...
}
```

A subsequent Volume column entry on the same row applies *on top* of this
reset. So set volume to 1.0 in your Wave-change branch, before the Volume
branch runs.

### 5.4 Empty effect cells mean "no effect", not "carry previous"

Matilde reads `m_Vals.effects[i]` fresh every row from the pattern engine —
an empty cell is just `command=0, argument=0`. There is no concept of
"continuous effect across empty rows".

To replicate in a managed machine where setters only fire for non-empty cells,
do **row-edge clearing**: at every newRow boundary, for any effect slot whose
`HasNewCmd[s]` is false, zero `ActiveCmd[s]` and `ActiveArg[s]`.

```csharp
for (int s = 0; s < CMD_SLOTS; s++)
{
    if (v.HasNewCmd[s])
    {
        v.HasNewCmd[s] = false;
        v.ActiveCmd[s] = v.PendingCmd[s];
        v.ActiveArg[s] = v.PendingArg[s];
    }
    else if (newRow)
    {
        v.ActiveCmd[s] = 0;     // ← clears stale carried effect
        v.ActiveArg[s] = 0;
    }
}
```

**Caveat:** the FastTracker "preserve previous nibble" idiom (e.g. `04 00`
keeps prior vibrato speed/depth) is tested by checking the *argument's
nibbles*, not the cell being empty. `04 00` is a non-empty cell — its setter
fires, `HasNewCmd[s]` becomes true, the row-edge clear doesn't trip — so
the idiom still works.

### 5.5 `0Cxx` set-volume is a Pedal Tracker addition (not in Matilde)

Matilde has no `0xC` effect command. If you implement one for ergonomics,
keep the same logarithmic curve as the Volume column for consistency:
`v.Volume = (arg/128f)²`.

---

## 6. Two effect columns per track

Matilde has two effect columns. The clean implementation is per-voice arrays:

```csharp
public const int CMD_SLOTS = 2;
public readonly byte[] PendingCmd = new byte[CMD_SLOTS];
public readonly byte[] PendingArg = new byte[CMD_SLOTS];
public readonly bool[] HasNewCmd  = new bool[CMD_SLOTS];
public readonly byte[] ActiveCmd  = new byte[CMD_SLOTS];
public readonly byte[] ActiveArg  = new byte[CMD_SLOTS];
```

Each column gets its own setter (`SetCmd`/`SetArg` + `SetCmd2`/`SetArg2`),
delegating to a shared `StoreCmd(track, slot, value)` / `StoreArg(track, slot, value)`.

Effect dispatch loops both slots through a single `ApplyEffectSlot(v, slot, anyPortaActive)`
helper. Composition rules:

- **Different effects in slots** (e.g. `04` vibrato + `0A` volume slide) →
  compose naturally; each updates its own state.
- **Same effect in both slots** (e.g. two `04`s) → second write wins because
  both update the same backing fields.
- **Two `01`/`02` pitch slides** → accumulate on `CurrentMidi` (each slot
  contributes its own delta).
- **`03` portamento in both slots** → must only advance once per row. Guard
  with `if (slot == 0 || !anyPortaActive)` so a 03 in slot 1 only fires when
  slot 0 doesn't have one.

For trigger-time effect detection (`03`/`09`/`0E9`/`0ED`), scan both slots:

```csharp
static int FindCmdSlot(Voice v, byte cmd)
{
    for (int s = 0; s < Voice.CMD_SLOTS; s++)
        if (v.ActiveCmd[s] == cmd) return s;
    return -1;
}
static int FindExtendedSubCmd(Voice v, byte subHighNibble)
{
    for (int s = 0; s < Voice.CMD_SLOTS; s++)
        if (v.ActiveCmd[s] == 0x0E &&
            (v.ActiveArg[s] & 0xF0) == subHighNibble) return s;
    return -1;
}
```

For auto-cancel of continuous effects (e.g. vibrato stops when `04` is gone),
check **both** slots — only cancel when no slot has the effect:

```csharp
bool any04 = FindCmdSlot(v, 0x04) >= 0;
if (!any04) v.VibratoActive = false;
```

---

## 7. ReBuzz API edge knowledge

These were either undocumented or only learned via reflection diagnostics.

### 7.1 `IWaveformBase.GetDataAsFloat`

```csharp
void GetDataAsFloat(float[] output, int outoffset, int outstride,
                    int channel, int offset, int count);
```

Use `outstride = 1`, `outoffset = MAX_PAD (16)` to leave room for negative
indexing in interpolators (sinc / cubic look behind the start).

### 7.2 `IWaveLayer.RootNote` is in Buzz-byte form

```csharp
// Buzz-byte: high nibble = octave, low nibble = (semi+1)
// So C-4 = 0x40 (oct 4, semi 0+1 → 1)
// To linear MIDI:
int oct  = (b >> 4);
int semi = (b & 0xF) - 1;
int midi = oct * 12 + semi;
```

### 7.3 `WaveFormat` and `WaveFlags`

```csharp
enum WaveFormat { Int16, Float32, Int32, Int24 }
[Flags] enum WaveFlags { Loop = 1, Not16Bit = 4, Stereo = 8, BidirectionalLoop = 16 }
```

`GetDataAsFloat` handles the bit-depth conversion internally — your machine
sees floats regardless of the wave's storage format.

### 7.4 `DescribeValue` populates the second status-bar panel

```csharp
public string DescribeValue(IParameter p, int value)
{
    switch (p.Name) { /* return formatted string per parameter */ }
    return null;     // null falls back to ParameterDecl.ValueDescriptions
}
```

Discovered by reflection in `ManagedMachineHost.cs:255`. Useful for showing
"0x80 (unity, 0.0 dB)" instead of just "128".

### 7.5 `IBuzz.DCWriteLine` for debug output

One line per call — multi-line strings get truncated. To dump a multi-line
diagnostic:

```csharp
foreach (var line in text.Split('\n'))
    Buzz.DCWriteLine(line.TrimEnd('\r'));
```

### 7.6 `IBuzz.Song.PlayPosition` — see §2.3

### 7.7 `MasterInfo.SamplesPerTick` for sub-tick scheduling

For effects like `0EDx` (note delay) and `0E9x` (retrigger), schedule by
sample count using `host.MasterInfo.SamplesPerTick * delayTicks`. Decrement
the counter every Work() by `n` samples.

### 7.8 `ParameterCore.pvalues` field type is NOT stable across builds

The private `pvalues` field we reflect into for multi-track note
recovery changed type between ReBuzz builds — `ConcurrentDictionary<int,int>`
up to ~1818, `int[256]` in 1827. A bare `as` cast silently nulled on
the new type and killed all multi-track playback. The reader must
detect the field shape at runtime. **This is a specific instance of a
general hazard — see §16 for the full lesson and the audit list of
every reflection site that needs checking on each ReBuzz upgrade.**

---

## 8. Diagnostic methodology that worked

When pattern data looks wrong and you can't tell whether ReBuzz delivered
it or your code dropped it, the only way to know is **a per-track ring buffer
of recent events with timestamps**. Pure counters don't show ordering or
context; pure breakpoints don't survive the realtime audio thread.

Format that worked well:

```
w<workCallNumber> song=<PlayPosition> pit=<PosInTick> : <event-name>(<args>)
```

Examples from the diagnostic dump that solved the loop-wrap bug:

```
w 5127 song=  0 pit=  256 : SetArg0(50)
w 5127 song=  0 pit=  256 : SetCmd0(09)
w 5127 song=  0 pit=  256 : PollNote(41)
w 5127 song=  0 pit=  256 : PollWave(01)
w 5127 song=  0 pit=  256 : PollVol(80)
w 5128 song=  0 pit=  256 : trigger nb=41 slot=1 act=True cg=1.000 cmd0=09/50 cmd1=00/00
w 5128 song=  0 pit=  256 : trig-deferred nb=41 cg=1.000
```

Each entry tells you (a) when it happened in audio-thread time, (b) which
song row it's on, and (c) what the event was. Side-by-side comparison of
"first play" vs "loop wrap" rows immediately revealed the missing setter
calls.

Implementation skeleton:

```csharp
const int EVENT_LOG_SIZE = 16;
readonly string[][] _eventLog;     // [track][slot]
readonly int[]      _eventLogPos;

void LogEvent(int track, string what)
{
    int p = _eventLogPos[track];
    int songPos = -1, pit = 0;
    try { songPos = Buzz?.Song?.PlayPosition ?? -1; } catch { }
    try { pit = host?.MasterInfo?.PosInTick ?? 0; } catch { }
    _eventLog[track][p] = $"w{_workCalls,5} song={songPos,3} pit={pit,5} : {what}";
    _eventLogPos[track] = (p + 1) % EVENT_LOG_SIZE;
}
```

**Strip this for production** — string formatting on the audio thread costs
real time. Keep counters (`_workCalls`, `_setNoteCalls`, etc.) and one-shot
status strings (`_lastTriggerResult`); those are cheap.

---

## 9. Quick reference: the order of operations per audio chunk

This is the canonical sequence per WorkManager iteration (≤256 stereo
samples). Knowing it cold answers most "why doesn't X happen at Y?" questions.

1. `samplesToProcess = min(remaining/2, 256)` — clamped to not cross a tick
   boundary
2. `UpdatePatternPositions(samplesToProcess)` — advances every pattern's
   PlayPosition; fires `OnPatternPlayPositionChange`; pattern editors walk
   their grids and call `parameter.SetValue` for every non-empty cell crossed
   in this chunk → writes `pvalues`, populates `parametersChanged`
3. `UpdatePatternColumnEvents` — MIDI columns dispatch
4. `UpdateMasterAndSubTickInfoToHost` — snapshots `PosInTick` etc. into each
   managed machine's host
5. `CallTick` → `Tick()` per machine → if `PosInTick == 0` (or sub-tick
   boundary): invoke `manageMachineHost.Tick()` → iterate `parametersChanged`
   → call your setters → then `parametersChanged.Clear()` → `pvalues` reset
   to NoValue
6. `ReadWork` → your `Work(output, n, mode)` runs
7. `UpdateNonStateParametersToDefault` — for native machines (managed bypass)
8. `subTickInfo.PosInSubTick += samplesToProcess; masterInfo.PosInTick += samplesToProcess`
9. If `PosInTick >= SamplesPerTick`: `PosInTick = 0`, song-position advance
   (with loop-wrap if applicable)

`pvalues` is therefore valid to read **inside any setter** (between steps 5
start and 5 end), but is reset by the time `Work()` runs (step 6). This is
why §1.3 polls from `SetNote`, not from `Work()`.

---

## 10. File structure that worked for Pedal Tracker

For reference if building a similar tracker-style machine:

```
PedalTracker/
├── PedalTracker.cs        (~1500 lines)  setters, Work, TriggerNote,
│                                          Apply/RunPerTickEffects,
│                                          transport, diagnostics, menu
├── Voice.cs               (~360 lines)   per-voice state, Render with
│                                          deferred-trigger fade-cross
├── Wavetable.cs           (~330 lines)   WavetableAccess, snapshot cache,
│                                          PickLayerForNote, IWave.PropertyChanged
│                                          subscription + invalidation
├── Interpolation.cs       (~250 lines)   None / Linear / Cubic / B-Spline /
│                                          Sinc16 / Sinc32 (4096-phase
│                                          Blackman-Harris)
├── PedalTracker.csproj
└── README.md
```

Voice state worth listing explicitly — these are the fields that took the
most thought to get right:

```csharp
public WaveSnapshot Snap;        // currently-playing snapshot
public double Pos, Increment;    // sub-sample position + frames/sample
public double BaseIncrement;     // un-modulated rate (vibrato modulates Increment)
public double CurrentMidi;       // continuous; pitch slides update this,
                                  //   Increment recomputed from this each tick
public float CurrentGain;        // anti-click envelope (ramps to TargetGain)
public float TargetGain;
public bool  HasPendingTrigger;  // deferred-trigger machinery
public WaveSnapshot PendingSnap;
public int    PendingMidi, PendingPos;
public int    NoteDelaySamples;  // sub-tick scheduling for 0EDx
public int    NoteCutSamples;    // 0ECx
public int    RetrigPeriodSamples, RetrigCountdown;
public bool   RetrigActive;      // 0E9x
public bool   PortaActive;       // 03 in flight
public double PortaTargetMidi, PortaSpeedSemis;
public bool   VibratoActive;     // 04 active this row
public double VibratoSpeed, VibratoDepthCents, VibratoPhase;
```

Anti-click envelope constants:

```csharp
const float FADE_STEP = 1f / 64f;     // ~1.3 ms at 48 kHz, ~1.5 ms at 44.1
const float NEAR_ZERO = 1e-4f;
```

---

## 11. Programmatic pattern manipulation across machines

Added in v1.3 alongside the Matilde import feature. There are two real
gotchas in this area; neither shows up in any obvious place.

### 11.1 Each managed machine has its own dedicated pattern-editor instance

The mental model "there's *the* Modern Pattern Editor in the song" is
wrong. ReBuzz creates a fresh editor instance per managed machine
(`ReBuzzCore.CreateEditor` at `ReBuzzCore.cs:1755`). Each editor instance
has its own `MPEPatternsDB` containing only that machine's patterns.

The editors are **hidden** from the public machine list. Their names start
with `\u0001 + "pe"` (`\x01pe1`, `\x01pe2`, ...) and `SongCore.Machines`
filters them out via `!m.Hidden`:

```csharp
public ReadOnlyCollection<IMachine> Machines =>
    machinesList.Where(m => !m.Hidden && m.Ready).Cast<IMachine>().ToReadOnlyCollection();
```

So **searching `Buzz.Song.Machines` for the editor will never find it**.
The first cut of MatildeImport tried this and failed at the lookup step.

The right way: each `MachineCore` has a public `EditorMachine` property
typed as the internal `MachineCore`. It's reachable via reflection on any
`IMachine` instance:

```csharp
// MachineCore.EditorMachine — public getter, internal type.
// Reflection finds it by name on the IMachine's runtime type.
object editorIm = ReadAny(machine, "EditorMachine");
// MachineCore : IMachine, so we can cast back and use the public API.
IBuzzMachine editorMM = (editorIm as IMachine)?.ManagedMachine;
// editorMM is now the ModernPatternEditorMachine instance for THIS machine.
```

Source and destination machines have different editor instances and
different `MPEPatternsDB`s. Cross-machine pattern copy needs to read
from one and write to the other:

```csharp
// Resolve once per side; keep two database refs and route by pattern.Machine.
var (srcDb, ...) = ResolveOneSide(source, "source");
var (dstDb, ...) = ResolveOneSide(destination, "destination");

// When fetching MPEPattern wrappers later:
object db = ReferenceEquals(p.Machine, srcMachine) ? srcDb : dstDb;
mpePattern = getMpePatternMethod.Invoke(db, new object[] { p });
```

### 11.2 Public `IPatternColumn.SetEvents` doesn't update MPE's display

There are **two parallel event lists** for any column on a managed
machine:

- `PatternColumnCore.patternEvents` (in ReBuzz core, what
  `IPatternColumn.SetEvents` writes to).
- `MPEPatternColumn.eventList` (in MPE, what the editor renders from
  and what `MPEPatternColumn.PlayColumnEvents` dispatches at play time).

MPE caches its own list and doesn't sync from the core's list when it
changes. So writing through `IPatternColumn.SetEvents` updates ReBuzz
core but **the user sees no change in the pattern editor and playback
also doesn't pick it up** (since MPE drives playback by walking its own
`eventList`, not the core's).

To programmatically write pattern data, you have to drive MPE directly:

```csharp
// Per pattern:
destination.CreatePattern(name, length);
//   ^ fires IMachine.PatternAdded → MPE auto-creates an empty MPEPattern.

var dstPattern = destination.Patterns.Last(p => p.Name == name);
object dstMpePattern = getMpePattern.Invoke(dstDb, new object[] { dstPattern });

// Per (parameter, track) you want to populate:
object dstCol = createNewColumn.Invoke(dstMpePattern, new object[] { dstParam, track });
//   ^ MPEPattern.CreateNewColumn(IParameter, int) → fresh MPEPatternColumn.

// Write events:
setEvents.Invoke(dstCol, new object[] { events, /*set:*/ true, /*play:*/ false });
//   ^ MPEPatternColumn.SetEvents(PatternEvent[], bool, bool) — internal.
//     play=false avoids triggering audio-thread side effects.

// UI refresh:
dstPattern.NotifyPatternChanged();
```

The internal `SetEvents(PatternEvent[], bool, bool)` overload (3 args,
not 2) is the one MPE itself uses for bulk operations. The `play`
parameter is the difference: with `play=true` it dispatches notes
through the audio thread as if recorded; with `play=false` it's a
silent paste.

### 11.3 Cross-tracker parameter name mapping

Matilde's track parameter names (verified against
`Buzztrax/buzzmachines/Matilde/Tracker/Tracker.cpp`):

```
Note · Wave · Volume · Effect1 · Argument1 · Effect2 · Argument2
```

Pedal Tracker's:

```
Note · Wave · Volume · Cmd · Arg · Cmd2 · Arg2
```

For the import to round-trip cleanly the column-byte semantics are
identical (Buzz pt_note for Note; raw 0x00–0xFE bytes for the rest)
so the per-event translation is a straight pass-through. Only the
*column identity* needs mapping. A static dictionary suffices:

```csharp
static readonly Dictionary<string, string> ParamAlias = new()
{
    { "Note", "Note" }, { "Wave", "Wave" }, { "Volume", "Volume" },
    { "Effect1", "Cmd" },  { "Argument1", "Arg"  },
    { "Effect2", "Cmd2" }, { "Argument2", "Arg2" },
    // Identity passthroughs for Pedal-Tracker → Pedal-Tracker imports.
    { "Cmd", "Cmd" }, { "Arg", "Arg" },
    { "Cmd2", "Cmd2" }, { "Arg2", "Arg2" },
};
```

Source columns whose `Parameter.Name` doesn't appear in the destination
machine's parameter list (e.g. Matilde's global parameters showing up
as columns at the global-group level) get skipped with a warning.
Expect ~4 such skips per pattern from Matilde imports — that's
`AmpDecay`, `Offset`, `Quantize`, `Tuning` (Matilde's globals declared
in `Tracker.cpp`).

### 11.4 Other observations from the v1.3 work

- `IMachine.CreatePattern(name, length)` and `IMachine.DeletePattern(p)`
  fire `PatternAdded` / `PatternRemoved` events. MPE subscribes to these
  per-machine in `MachineVM.cs`, so it auto-syncs its `MPEPatternsDB`
  whenever you create or delete patterns through the public API. You
  don't need to manually register with MPE for new patterns — but you
  DO need to populate their column data, since MPE's auto-add creates
  an empty `MPEPattern` shell only.
- `IMachine.TrackCount = N` works as expected on managed machines.
  Setting it grows or shrinks the per-track parameter columns
  automatically.
- `IMachine.ManagedMachine` is part of the public `IMachine` interface
  and returns the `IBuzzMachine` instance directly — no reflection
  needed for managed-machine self-introspection (unlike `EditorMachine`
  which is internal-typed).
- `PatternEvent.TimeBase = 240` ticks per Buzz tick, hardcoded as
  `public const int` on the struct. Pattern lengths are stored in Buzz
  ticks; positions in events are in TimeBase units. Copying patterns
  verbatim doesn't need any time-base conversion as long as both
  machines use the same editor (which they do if both use MPE).

---

## 12. Multi-output per-track generators

Added in v1.4 alongside the multi-out feature for routing each pattern
track to a separate output. Two non-obvious things here.

### 12.1 Activate multi-out by defining the IList Work overload

ReBuzz's managed-machine host detects which Work overload your class
defines via reflection at registration time
(`ManagedMachineHost.cs:194-198`):

```csharp
// Single stereo out — sets MachineInfoFlags.MULTI_IO = false
public bool Work(Sample[] output, int n, WorkModes mode);

// Multi-out — sets MachineInfoFlags.MULTI_IO = true automatically
public bool Work(IList<Sample[]> output, int n, WorkModes mode);
```

Define exactly one. The flag is set at `ManagedMachineDll.cs:84` based
on which signature the machine class exposes. No explicit `MULTI_IO`
declaration needed; no attribute parameter for it either.

`IList<Sample[]>` arrives with one entry per output channel. Each entry
is a stereo `Sample[]` (with per-element `.L` and `.R` floats). ReBuzz
only allocates a `Sample[]` buffer for outputs that have an active
connection; unconnected outputs are `null` in the list. So the host
side does the connection-tracking for you — your render code just
checks `if (output[i] == null) continue;` and skips writing to that
slot.

### 12.2 Static `OutputCount` with headroom — NOT dynamic auto-sync

The first instinct is "make `OutputCount` track `TrackCount` so I don't
expose more outputs than needed." This crashes on song reload. The
sequence:

1. ReBuzz instantiates the machine. `MachineDecl.OutputCount` sets
   the initial `MachineCore.OutputChannelCount` at
   `MachineManager.cs:165`.
2. ReBuzz restores saved connections from the .bmx file. A connection
   from "Track 3" has `SourceChannel = 3`.
3. The audio thread calls into `MachineWorkInstance.WorkMachineManaged`
   for the first time. Line 145 sizes `multiSamplesOut` to
   `Machine.OutputChannelCount`. Line 154 indexes
   `multiSamplesOut[outConnection.SourceChannel]`.
4. If `SourceChannel >= OutputChannelCount`,
   `List<>.set_Item(int, T)` throws `ArgumentOutOfRangeException`
   **before our Work() ever runs**. Any auto-sync logic at the top
   of Work() is unreachable.

The fix is to declare `OutputCount` with full headroom up front:

```csharp
[MachineDecl(
    Name        = "Pedal Tracker",
    MaxTracks   = MAX_VOICES,           // 16
    OutputCount = MAX_VOICES + 1)]      // 17 = 1 master + 16 max tracks
```

This is free at runtime: `MachineWorkInstance.cs:150-160` only
allocates `Sample[]` buffers for outputs that have a connection. The
17 - N unconnected slots stay `null` in the list and skip the render
loop. Zero memory waste, zero per-Work overhead.

`MachineDecl` attribute arguments must be compile-time constants, so
expressions like `MAX_VOICES + 1` work as long as `MAX_VOICES` is
declared `public const int` on the same class.

### 12.3 Master output convention

For any tracker-style multi-out generator, conventional layout is:

- **Output 0** — summed master mix (with master gain applied)
- **Outputs 1..N** — per-track dry signals (no master gain)

The "no master gain on per-track" part matters: if you apply master to
both, the master fader gain-stages everything you've routed externally,
which defeats the whole point of per-track routing for external
processing. Keep per-track outputs dry; the user's external mixer
handles per-track levels.

### 12.4 Channel naming for the connection menu

ReBuzz looks up `string GetChannelName(bool input, int index)` by name
via reflection (`ManagedMachineHost.cs:247`). Define it to make the
right-click → "Connect…" submenu show readable labels:

```csharp
public string GetChannelName(bool input, int index)
{
    if (input) return null;            // generator has no inputs
    if (index == 0) return "Master";
    return "Track " + index;           // 1-based for the user
}
```

Returning `null` falls back to a generic "out N" label.

### 12.5 Render strategy: per-track scratch, then publish

Allocate a dedicated stereo scratch pair per voice (16 of them, each
sized to a max audio block — 256 samples is the ReBuzz cap, but
allocate generously e.g. 2048 to weather any future change). The
render loop is identical to single-out except every voice writes to
its own scratch instead of a shared mix.

```csharp
// Render
for (int t = 0; t < MAX_VOICES; t++) {
    if (!_voices[t].Active) continue;
    Voice.Render(_voices[t], _trackL[t], _trackR[t], n,
                 interp, _engineRate, /*master:*/ 1f);
}
// Publish out 0 — sum + master gain
if (output[0] != null) {
    for (int i = 0; i < n; i++) {
        float l = 0, r = 0;
        for (int t = 0; t < MAX_VOICES; t++) { l += _trackL[t][i]; r += _trackR[t][i]; }
        output[0][i].L = l * masterGain * OUT_SCALE;
        output[0][i].R = r * masterGain * OUT_SCALE;
    }
}
// Publish outs 1..N — per-track dry
for (int t = 0; t < Math.Min(output.Count - 1, MAX_VOICES); t++) {
    if (output[t + 1] == null) continue;
    for (int i = 0; i < n; i++) {
        output[t + 1][i].L = _trackL[t][i] * OUT_SCALE;
        output[t + 1][i].R = _trackR[t][i] * OUT_SCALE;
    }
}
```

This makes render cost independent of routing topology (always render
all voices, even ones whose track output is unconnected — they still
need to contribute to the master sum on out 0). Per-voice connection
bookkeeping isn't needed.

### 12.6 Mono samples on stereo outputs

Each multi-out slot is always a stereo `Sample[]` regardless of source
material. For a mono wave the natural choice is centre-panned mono:
`L = R = mono * gain`. If your wave-snapshot code already duplicates
the mono channel into both `L` and `R` arrays at load time (Pedal
Tracker's `WavetableAccess.SnapAll` does this with
`Array.Copy(L, MAX_PAD, R, MAX_PAD, sampleCnt)`), no special-case is
needed in the render loop — the per-voice render path handles mono
and stereo identically and the per-track output naturally gets
centre-panned mono content.

---

## 13. Parameter persistence, mute design, and audio/UI threading

Three closely related issues that all surfaced during v1.5 / v1.6 work.
Together they govern any feature that exposes a UI surface for the
machine's own globals (right-click menus, custom WPF windows,
attribute panels) or that toggles `IMachine` properties from inside
`Work()`.

### 13.1 The direct-property-write trap

There are **two parallel storage locations** for any managed-machine
parameter:

- `ParameterCore.values[track]` — ReBuzz core's dictionary, written by
  `parameter.SetValue(track, value)`, read at save time by
  `BMXMLFile.cs:858`.
- The C# property's backing field — read by your code, written by
  either the property's setter (when called via `SetValue`) or by
  direct assignment.

Writing directly to the C# property updates only the field. The C#
state changes, audio behaviour reflects the change, the UI can even
read the new value — **but `values[]` stays at the old value, and
that's what gets written to the saved song.**

The trap manifests as: "I set this from a right-click menu / custom
window / hot-key handler, the change worked at runtime, but it's
gone after save and reload."

```csharp
// WRONG — updates the field but not values[].  Won't persist.
items.Add(new MenuEntry(id, "Cubic", () => InterpParam = 2));

// RIGHT — routes through ParameterCore.SetValue, which writes to
// values[] AND fires our property setter.
items.Add(new MenuEntry(id, "Cubic",
    () => WritebackParameter("Interpolation", 2)));

void WritebackParameter(string name, int value)
{
    var groups = host?.Machine?.ParameterGroups;
    if (groups == null || groups.Count < 2) return;
    foreach (var p in groups[1].Parameters)
        if (p?.Name == name) { p.SetValue(0, value); return; }
}
```

This applies to **any** custom UI that mutates your own globals.
Pattern automation, peer control, the right-click parameter properties
dialog all already use `SetValue` — those work correctly without any
special code on your end. The bug only appears when *you* write a
custom path that touches the property directly.

For bool parameters use `value ? 1 : 0` when calling `SetValue`.

### 13.2 One-way authority for parameter ↔ machine-state mirrors

If a parameter on your machine logically corresponds to a property on
`IMachine` (e.g. `MuteParam` ↔ `IMachine.IsMuted`), the temptation is
bidirectional sync — propagate changes in either direction so they
stay consistent. **Don't.** It races.

The race shape: setting `IMachine.IsMuted` from the audio thread must
be marshalled to the UI thread (see §13.3) which is asynchronous via
`Dispatcher.BeginInvoke`. Between the dispatch and the UI thread
running, the next audio-thread Work cycle reads the *stale* `IsMuted`
state, sees a discrepancy with the parameter, and "corrects" it —
reversing the parameter change that started the cascade.

This is especially nasty with peer-control / multi-instance setups
where many dispatches queue up at once and the audio thread keeps
ticking through stale state for several Work cycles before any of
the UI callbacks land.

The correct design is **one-way authority with idempotent
re-assertion**:

- The parameter is the source of truth. Direct mutations to the
  mirrored `IMachine` property are unilateral; we don't try to
  propagate them back.
- Every Work cycle, re-assert the desired `IMachine` state from the
  parameter. Make the assertion idempotent (short-circuit when
  already matching) so the steady-state cost is one comparison.

```csharp
// In every Work():
bool wantMuted = _muteParam && _muteGain <= 1e-4f;
SetMachineMutedUiSafe(wantMuted);    // no-op when already matching
```

User-perceived consequence: clicking the UI mute button while the
parameter says unmute briefly toggles the visual mute then snaps back
on the next Work cycle. The parameter wins. This is a feature, not
a regression — it gives users a clear mental model of "this parameter
controls mute; the UI button is a hard override that the parameter
ignores."

### 13.3 Setting `IMachine` properties from the audio thread

Direct: `host.Machine.IsMuted = true` from inside `Work()`.

What ReBuzz does: `MachineCore`'s setter runs on the calling thread,
fires `PropertyChanged.Raise(this, "IsMuted")` synchronously. The
machine view's `MachineControl.cs:143` listener catches the
notification, re-raises its own `PropertyChanged` for
`MachineBackgroundColor` and `NameText` — **also synchronously, on
the audio thread**.

WPF's data-binding system silently fails to update visuals when the
notification arrives on a non-UI thread. The internal state changes
correctly (the machine actually mutes), the right-click parameters
view reads the right value (because that view is queried on the UI
thread), but the machine-view box doesn't darken and its name doesn't
gain parentheses.

The fix is to marshal the assignment via `Application.Current.Dispatcher.BeginInvoke`,
which puts the property write on the UI thread so the
`PropertyChanged` cascade reaches WPF's bindings on the right thread:

```csharp
void SetMachineMutedUiSafe(bool muted)
{
    var m = host?.Machine;
    if (m == null || m.IsMuted == muted) return;       // idempotent

    var dispatcher = System.Windows.Application.Current?.Dispatcher;
    if (dispatcher == null || dispatcher.CheckAccess())
    {
        m.IsMuted = muted;                              // already on UI thread
    }
    else
    {
        dispatcher.BeginInvoke(new System.Action(() =>
        {
            try { if (host?.Machine != null) host.Machine.IsMuted = muted; }
            catch { }
        }));
    }
}
```

`BeginInvoke` is non-blocking — the audio thread doesn't stall waiting
for the UI thread. The idempotence check at the top means most calls
are a single comparison and a return.

This pattern generalises to **any** `IMachine` property whose change
drives a WPF binding. Setting `IsBypassed`, `Name`, or anything else
from `Work()` will hit the same trap. Audit every assignment to a
host-property from the audio thread and route it through a
UI-thread-safe wrapper.

### 13.4 Mute-with-fade design pattern

Pattern parameter (`MuteParam : bool`, `MuteInertiaParam : int`) +
runtime fade state (`_muteGain : float`, `_muteTarget : float`). The
parameter setter sets `_muteTarget`; Work() advances `_muteGain`
toward target every block. Output gain is the per-sample lerp from
the entry-block gain to the exit-block gain.

Per-block trajectory advance:

```csharp
static float AdvanceFade(float current, float target, int n, int inertiaSamples)
{
    if (current == target) return current;
    if (inertiaSamples <= 0) return target;
    float step = n / (float)inertiaSamples;
    return target > current
        ? System.Math.Min(target, current + step)
        : System.Math.Max(target, current - step);
}
```

Per-sample apply inside publish loops:

```csharp
float invN  = (n > 1) ? 1f / (n - 1) : 0f;
float delta = end - start;
for (int i = 0; i < n; i++) {
    float gain = start + delta * (i * invN);
    out[i] = source[i] * gain;
}
```

For per-track variants (16 independent fades), use stack-allocated
spans for the per-track start/end gains so the audio thread doesn't
allocate:

```csharp
System.Span<float> tmStart = stackalloc float[MAX_VOICES];
System.Span<float> tmEnd   = stackalloc float[MAX_VOICES];
```

Useful template for any "smooth-toggleable" feature: per-track Pan,
Solo, ducking, future automation lanes.

### 13.5 The `_workCalls == 0` snap-on-load trick

When a song loads, ReBuzz pushes saved parameter values through your
setters before the audio engine starts. If your setter triggers a
fade (e.g. `_muteTarget = value ? 0 : 1`), the fade will start from
whatever the field initializer was (typically 1.0) and ramp to the
loaded value — audible fade in/out at song load.

The fix: snap the gain to target when no Work() has run yet:

```csharp
public bool MuteParam
{
    set
    {
        if (_muteParam == value) return;
        _muteParam = value;
        _muteTarget = value ? 0f : 1f;
        // On song load (before any Work() has run), snap so we don't
        // fade in/out from the previous state.
        if (_workCalls == 0) _muteGain = _muteTarget;
    }
}
```

Diagnostic counters double as a "have we started rendering yet?"
flag. Cheap, no extra state, robust against the load sequence.

### 13.6 Bug-class summary

If you find yourself writing a custom UI for your own parameters
or mirroring a parameter to an `IMachine` property, run through this
checklist:

| Question | Right answer |
|----------|--------------|
| Does the change come from a custom UI handler? | Use `parameter.SetValue`, never direct property assignment. |
| Are you setting an `IMachine` property from `Work()`? | Marshal via `Dispatcher.BeginInvoke`. |
| Does state need to mirror in two directions? | Don't. Pick one as authoritative; re-assert idempotently every Work. |
| Does a fading parameter affect audio at song-load time? | Snap to target when `_workCalls == 0`. |
| Does the fade need to coexist with the host's zero-CPU bypass? | Re-assert `IsMuted` from the parameter+fade state every Work cycle so bypass engages exactly when the fade hits zero. |

---

## 14. Zero-CPU bypass and the resume problem (considered, rejected)

> **Verdict from v1.7**: this approach was tried in v1.5–v1.6.4 and
> ultimately rolled back.  The machinery described in this section is
> NOT present in the current production code.  See §15 for the design
> we settled on (always-render).  This section is preserved as
> reference material for anyone tempted to add host-side bypass back —
> the trade-offs and the corner cases are non-obvious and easy to
> re-derive incorrectly.  If you find yourself wanting zero-CPU mute,
> read §15.4 first to see why we walked away from it.

ReBuzz's zero-CPU bypass for muted machines (engaged when
**Settings → Audio → Process muted machines** is unchecked AND
`IMachine.IsMuted == true`) gates the entire render path: line 229 of
`MachineWorkInstance.WorkMachineManaged` returns immediately with a
silentBuffer write, our managed `Work()` is never called for that
audio block. Saves real CPU when many trackers are muted in dense
projects.

The trap: **Tick still fires** during the bypass period.
`MachineWorkInstance.Tick` at line 568 runs whenever
`PosInTick == 0` (or sub-tick boundary), independent of mute state.
That's how managed machines get parameter setters delivered. So while
Work is bypassed, our property setters keep running on every tick
boundary, dispatched by the pattern engine as song position advances
through whatever rows happen during the muted period.

Per-track per-row setters (`SetNote`, `SetWave`, `SetVolume`,
`SetCmd`, `SetArg`) write to `voice.PendingX` fields and set
`HasNewX = true`. **`HasNewX` flags are one-shot**, normally cleared
by `ApplyPendingParameters` inside `Work()`. With Work bypassed,
the flags never clear; subsequent tick boundaries overwrite the
`PendingX` values with whatever's on the latest row.

After many bypassed ticks, voice state is:

- `PendingNote` / `PendingWave` / `PendingVolume` / `PendingCmd[]` /
  `PendingArg[]` all hold the values from the **most recent row**
  delivered during the bypass — usually a row from many beats ago,
  not the row the song will be on when bypass releases.
- `HasNewX` flags are all true.
- Active voices that were sounding when bypass engaged are frozen:
  their `Pos` doesn't advance because Render didn't run, but they're
  still flagged `Active = true` with whatever `CurrentGain` they had.
  The note that "should" have ended naturally at sample end is
  still hanging in pause.
- Per-voice envelope/effect state (vibrato phase, porta progress,
  sub-tick countdowns) is similarly frozen.

When `IsMuted` clears and Work resumes, two things go wrong without
intervention:

1. The first call to `ApplyPendingParameters` consumes the stale
   `HasNewX` flags and triggers a note from a row in the past
   (whatever the last row before resume was). Audibly this is a
   missed beat or a snare hitting where there should be silence.
2. Active-but-frozen voices continue rendering from their stale
   `Pos`, playing the wrong moment of the sample.

### 14.1 Detection: was Work just resumed from bypass?

The audio thread can't directly observe "Work was bypassed for the
last N audio blocks." It can only see whether Work is running NOW
and what `IMachine.IsMuted` says. A reliable heuristic:

- Track `_prevIsMuted` — what `IsMuted` was last time Work ran.
- Track `_muteCyclesObservedInWork` — how many Work cycles have
  observed `IsMuted == true`. Increment when Work runs while muted,
  reset to zero whenever Work runs while unmuted.

The resume-from-bypass condition is then:

```csharp
bool isMutedNow = host?.Machine?.IsMuted ?? false;
if (_prevIsMuted && !isMutedNow && _muteCyclesObservedInWork == 0)
{
    // We're back from the zero-CPU bypass — discard stale state.
    DiscardStaleState();
    _prevSongPos = int.MinValue;   // next genuine row registers as a new row
}
_prevIsMuted = isMutedNow;
_muteCyclesObservedInWork = isMutedNow ? _muteCyclesObservedInWork + 1 : 0;
```

The counter distinguishes the bypass case from the
`ProcessMutedMachines = true` case (where Work runs throughout the
muted period and the inner gate at line 263 substitutes silentBuffer
for the output AFTER Work has run). In that mode pending state never
accumulates — `ApplyPendingParameters` consumed it as it arrived
during muted-but-running Work cycles — so the discard isn't needed
and would in fact drop legitimate just-delivered setter values.

### 14.2 Discard, but preserve persistent column state

The voice-level state to discard:

- `HasNewNote`, `HasNewWave`, `HasNewVolume`, `HasNewCmd[]` — the
  one-shot flags that would consume stale pending values.
- `Active`, `InLoop`, `PingPongDir`, `EffectSamplesAccum` — frozen
  voice state.
- `VibratoActive` + phase/speed/depth, `PortaActive` + target/speed
  — effects attached to the no-longer-playing note.
- `NoteDelaySamples`, `NoteCutSamples`, `RetrigPeriodSamples`,
  `RetrigCountdown`, `RetrigActive` — sub-tick countdowns scheduled
  relative to a timeline that's now in the past.
- `CurrentGain`, `TargetGain` — voice envelope (since voice is
  deactivating).
- `HasPendingTrigger`, `PendingSnap`, `PendingMidi`, `PendingPos` —
  deferred-trigger handoff state.

The voice-level state to **preserve**:

- `ActiveWaveSlot` — tracker convention is that note-only rows
  (rows where the Wave column is empty) play the last-explicitly-set
  wave. If you reset this on resume, a `kick . . .` pattern where
  only row 0 specifies the wave would play silence on rows 4, 8,
  12 of the next pattern after resume.
- `Volume` — same logic. The user might have a track muted on a
  row that set a non-default Volume column value; reset would snap
  to 1.0, audible level jump.
- `Pos` doesn't matter (the voice is being deactivated; Pos is
  re-seated by the next genuine note trigger anyway), but no harm
  leaving it.

Pedal Tracker's `Voice.DiscardForResume()` implements exactly this
partial-reset, alongside the full `Voice.Reset()` that's used at
machine construction.

### 14.3 The unmute-from-inside-bypass flow

When the user toggles the parameter back to unmute via peer control
or pattern automation during the bypass period, the setter fires
(Tick is still running). In Pedal Tracker, the `MuteParam` setter's
unmute path immediately calls `SetMachineMutedUiSafe(false)` to
dispatch `IsMuted = false` on the UI thread:

```csharp
public bool MuteParam {
    set {
        if (_muteParam == value) return;
        _muteParam = value;
        _muteTarget = value ? 0f : 1f;
        ...
        // Unmute path: clear ReBuzz's bypass immediately so Work
        // gets called again on the next audio cycle.  Mute path:
        // leave IsMuted alone — top-of-Work assertion will set it
        // to true once the fade-down completes (gain hits zero).
        if (!value) SetMachineMutedUiSafe(false);
    }
}
```

Without the immediate dispatch, the unmute would be stuck: Work
needs to run to clear `IsMuted`, but `IsMuted = true` keeps Work
bypassed. Setter on the audio thread is the only thread where we
can break the cycle from inside the bypass.

Once `IsMuted` clears, the next Work cycle runs, the resume-from-
bypass detection fires, `DiscardForResume` clears stale state, and
the next genuine pattern row delivers a clean trigger.

### 14.4 The cost-trade-off

Zero-CPU bypass is genuinely zero CPU: not just our Work() but the
entire output-buffer write path is skipped, and `silentBuffer` is
substituted directly. For a project with 16 trackers where 12 are
muted at any moment, the saving is the render cost of those 12
machines.

The cost is one row of pattern data lost across each mute boundary.
Whatever pending setter state had accumulated when bypass releases
is discarded, so the next genuine pattern row is the first audio
the user hears. At typical TPB×BPM (16 ticks per beat at 156 BPM ≈
24 ms per row) the loss is sub-perceptual for most musical
contexts. For slow tempos or live-performance triggering it can be
audible as a missed first beat after un-muting.

If a future requirement makes that row-loss unacceptable, the
alternative discussed but not implemented in v1.6.3 is "stop new
triggers immediately, fade existing voices over MuteInertia,
engage `IsMuted = true` only when all voices are quiet." This
preserves natural ring-out at the cost of delayed bypass engagement
and slightly more complex state machinery; see the v1.6.3 design
discussion for the trade-off shape.

### 14.5 Generalisation: detecting any host-side bypass resume

The `_prevState + _cyclesObserved` pattern works for any host-side
bypass condition the audio thread can observe via an `IMachine`
property. Examples beyond `IsMuted`:

- `IMachine.IsBypassed` — passes input through unchanged for
  effects (but generators don't use this)
- `IMachine.IsSoloed` interaction with `Song.SoloMode` — a
  generator that's not soloed during solo mode is bypassed at line
  166 of `WorkMachineManaged`
- `IMachine.IsSeqMute` — sequence-mute, separate from per-machine
  mute

Any of these cause Work to skip while Tick may still fire (depending
on the bypass site). If you build a feature that depends on Work
running continuously to manage state, plan for the skipped-Work
case the same way: detect resume, discard whatever stale state has
accumulated, let the next genuine setter call deliver fresh data.


---

## 15. Always-render mute (production design from v1.7)

The production master-mute and per-track-mute design after v1.7 is
deliberately simpler than what v1.5–v1.6.4 tried. The full machinery
described in §14 was removed because the trade-off didn't work for
this machine's workloads. This section documents what's actually in
the code and why we chose it.

### 15.1 The design

Two layers of fade gain applied in the publish loops, no interaction
with `IMachine.IsMuted`:

```csharp
// Master mute (one parameter, one fade gain pair)
public bool MuteParam { ... }                  // pattern-automatable switch
float _muteGain   = 1f;                        // current
float _muteTarget = 1f;                        // ramp toward

// Per-track mutes (16 parameters, 16 fade gain pairs)
public bool MuteT1..MuteT16 { ... }            // 16 individual switches
float[] _trackMuteGain   = new float[16];      // all initialized to 1.0
float[] _trackMuteTarget = new float[16];      // ditto
```

Setters update the corresponding `_muteTarget` (or `_trackMuteTarget[t]`)
and snap `_muteGain` to target on first call (`_workCalls == 0`)
to avoid audible flips on song load. Work() runs continuously every
audio block. The publish loops compute per-block fade trajectories
once (via `AdvanceFade(current, target, n, inertiaSamples)`) and
linearly interpolate per sample inside both the master sum and the
per-track multi-out copies. `IMachine.IsMuted` is never written from
our side.

The cost: the machine renders all 16 voice slots every audio block,
regardless of whether anything is audible. Voices that aren't
triggered cost nothing (the render loop skips `!v.Active` voices).
Active voices triggered at root pitch use v1.6.2's interpolation
bypass and are nearly free. The actual CPU footprint of a fully
muted Pedal Tracker is small enough that "always render" is
practically the same as "zero-CPU bypass" for most projects.

### 15.2 Why this works for trackers

Trackers play short, transient samples (drums, plucks, hits) at
their native pitch most of the time. Each voice's render loop is
a tight `src[i0]` read plus a couple of multiplies (gain, fade
trajectory). A full 16-voice block at 256 samples is in the order
of 4096 multiply-adds and 4096 adds for the master sum — single-
digit microseconds on modern CPUs. The render cost of an entire
muted Pedal Tracker is negligible.

The host-side bypass was an attempt to optimise something that
wasn't a bottleneck in practice, at the cost of breaking smooth
peer-controlled mute swaps. Bad trade for this machine type.

For a managed machine that actually does expensive per-block work
(e.g. an FFT-heavy effect, granular synthesis, convolution), the
trade-off would tilt the other way and §14's machinery would be
worth revisiting.

### 15.3 Peer-controlled swap as the deciding workflow

The use case that made always-render essential: 8 (or more)
Pedal Tracker instances arranged into two groups (e.g.
{a,c,e,g} vs {b,d,f,h}), each tracker playing a near-identical
pattern variation, with a peer-control machine toggling
`MuteParam` on one group at a time. The user wants to A/B
between groups smoothly to compose live.

With host-side bypass: the muted group's voices freeze
mid-render, pattern engine ticks accumulate one row of stale
setter state, and on resume the audible result is one of:
- A missed beat at the swap row (DiscardForResume drops state)
- A late trigger from a row already past (no DiscardForResume)
- A stale-Pos artefact from the frozen voice continuing to render

All three are perceptible. None match what the user hears with
a downstream gain machine doing the same A/B switch.

With always-render: both groups render continuously. The
parameter just crossfades between two identical signals via the
output gain. Bit-identical to the downstream-gain solution,
no perceptible artefacts.

The peer workflow is the most demanding case but it's also the
most musical case — composers reach for it because it sounds
right. Optimising the implementation for "musically right at
all costs" was the right call.

### 15.4 If you're tempted to add zero-CPU bypass back

Read §14 in full first, then think about whether the workloads
you care about actually save enough CPU to be worth the
behavioural complexity. The signposts:

- **What does an idle voice cost?** If your render loop has
  expensive per-block work that doesn't depend on having active
  voices (FFT setup, convolution kernel application, oversampled
  filter state), you might genuinely save CPU by skipping Work.
  Pedal Tracker doesn't — idle voices are skipped via `!v.Active`
  early in the loop, no fixed per-block overhead.
- **Will users ever swap muted instances?** If yes, host-side
  bypass introduces audible artefacts on swap and you'll be
  fighting them with DiscardForResume-style mitigations that
  trade one kind of artefact for another. If no (e.g. mute is
  only used to silence permanently-unused tracks), bypass is
  free.
- **Can you keep both?** A per-instance setting ("Smooth Mute"
  vs "Zero-CPU Mute") is technically possible — the bypass-
  resume detection plays nicely alongside always-render code as
  long as the bypass is opt-in. We considered this in v1.6.4
  development but rejected it for surface-area reasons. Worth
  revisiting only if a real workload appears that benefits.
- **The UI mute button stays a hard kill regardless.** Clicking
  the standard Buzz mute button on the machine sets
  `IMachine.IsMuted = true` directly via `MachineControl.cs:207`
  — no path goes through your code. With ProcessMutedMachines=
  false, ReBuzz bypasses you. Whatever your parameter design,
  the UI button is always a hard kill. Don't try to mirror it
  to your parameter (see §13.2 for why bidirectional sync
  doesn't work).

### 15.5 The lesson generalised

When a host offers a CPU-saving optimisation that requires you
to give up state continuity, evaluate the trade-off against the
*actual* workloads users will exercise, not the worst case the
optimisation was designed to address.

For a tracker: workloads are mostly cheap (sample reads, fade
multiplies). State continuity matters because the user can
swap-mute machines musically. Conclusion: always render.

For a different machine type the answer might flip. The point
isn't that host-side bypass is bad — it's that the framework
exposes it as an option, and you have to decide whether your
workload benefits more from the CPU saving or suffers more from
the state discontinuity. Pedal Tracker's answer happens to be
"render always". Document yours either way.

---

## 16. Cross-build reflection fragility (the 1827 pvalues break)

This is the most important lesson for long-term maintenance: **every
place we reach into ReBuzz internals via reflection is a latent break
waiting for the next ReBuzz build.** One of them (the `pvalues` field)
broke between 1818 and 1827 and took out all multi-track playback.

### 16.1 What broke

The multi-track simultaneous-note recovery (§ "Multi-track
simultaneous-note workaround", and notes §1) depends on reading each
parameter's private `pvalues` field by reflection. The recovery
exists because ReBuzz delivers only ONE `SetNote` per parameter per
tick — when several tracks have a note on the same row they all write
the shared Note parameter and only the last writer survives in the
engine's `parametersChanged` dictionary. We recover the others by
reading their raw `pvalues` before the post-tick NoValue reset.

In ReBuzz ≤1818, `ParameterCore.pvalues` was:

```csharp
ConcurrentDictionary<int, int> pvalues;
```

In ReBuzz 1827 it became:

```csharp
int[] pvalues = new int[256];
```

Our reflection did:

```csharp
fi.GetValue(p) as ConcurrentDictionary<int, int>;   // → null on int[]
```

The `as` cast silently returned **null** for the new `int[]` field.
No exception, no error — `GetPValuesField` just returned null for
every parameter, `EnsurePollFields` bailed, `PollAllPValues` recovered
nothing, and only the one directly-delivered track ever triggered.

Symptom as seen by the user: "multi-out only plays the last track."
The "last" track is whichever one wrote the Note parameter last that
tick (the surviving `parametersChanged` entry).

### 16.2 Why it was hard to see

The failure is silent and several layers removed from the symptom:

- The routing/dispatch path is completely intact — buffers, channel
  mapping, `UpdateOutputs`, summing all unchanged. Reading that code
  reveals nothing because nothing there is wrong.
- The bug is in note *delivery*, not audio *routing*, but the symptom
  ("only one track of audio") looks like a routing problem.
- The `as` cast fails silently. There's no crash to trace back from.

What cracked it: instrumenting our OWN Work to log, per call, which
voices were active and which output slots held data. The trace showed
`act=2` (one voice) with all per-track output slots empty — proving
our render/publish were correct and the problem was upstream, in how
many voices ever got triggered. From there the pvalues recovery was
the obvious suspect.

**Lesson: when a symptom points "downstream" but the downstream code
is provably correct, instrument the boundary — log what your own code
actually received and produced. Don't keep re-reading the code that
isn't wrong.**

### 16.3 The fix: tolerate multiple field shapes

`GetPValuesField` now returns a reader (`Func<int,int>`) that detects
the backing representation at runtime and closes over whichever is
present:

```csharp
static System.Func<int, int> GetPValuesField(IParameter p)
{
    var fi = p.GetType().GetField("pvalues",
        BindingFlags.NonPublic | BindingFlags.Instance);
    if (fi == null) return null;
    object raw = fi.GetValue(p);
    if (raw == null) return null;
    int noValue = p.NoValue;

    if (raw is int[] arr)
        return track => ((uint)track < (uint)arr.Length) ? arr[track] : noValue;

    if (raw is ConcurrentDictionary<int, int> dict)
        return track => dict.TryGetValue(track, out int v) ? v : noValue;

    if (raw is System.Collections.IDictionary idict)        // last resort
        return track => idict.Contains(track) ? (int)idict[track] : noValue;

    return null;
}
```

The reader closes over the array reference. Safe because 1827 never
reallocates `pvalues` (fixed `int[256]`, cleared via `Array.Fill`).
Same DLL now works on 1818 (dictionary) and 1827 (array).

### 16.4 The general rule for reflection into ReBuzz

Every reflection access into ReBuzz internals must:

1. **Find by name, not assume type.** Get the `FieldInfo`/`MethodInfo`
   first; check it's non-null before using it.
2. **Detect the shape at runtime.** Use `is`-pattern checks
   (`raw is int[]`, `raw is ConcurrentDictionary<...>`), never a bare
   `as` cast that silently nulls on a type change.
3. **Degrade gracefully.** If the field/shape is unrecognised, return
   a safe default and keep the machine working (even if a feature is
   reduced) rather than crashing or silently producing wrong output.
4. **Fail loudly in debug.** Consider a one-time `DCWriteLine` when a
   reflection target isn't found, so the next break is visible in the
   console instead of manifesting as a mystery audio symptom.

### 16.5 Other reflection sites to audit on each ReBuzz upgrade

These are the places Pedal Tracker reaches into ReBuzz internals.
**Check every one when a new ReBuzz build appears:**

| Site | What it reflects | Risk if it breaks |
|------|------------------|-------------------|
| `GetPValuesField` | `ParameterCore.pvalues` | Multi-track notes lost (this bug) |
| Matilde import | `MachineCore.EditorMachine`, MPE internals (`MPEPatternsDB`, `MPEPattern`, `MPEPatternColumn`) | Pattern import fails |
| `EnsurePollFields` | `ParameterGroups[2].Parameters` by name | pvalue recovery disabled |

The Matilde-import chain (notes §11) is the most elaborate and the
most likely to break next — it reflects through three layers of
ModernPatternEditor internals that have no public-interface contract.
When it breaks, the same defensive approach applies: detect shape,
tolerate variation, degrade gracefully (import becomes a no-op with a
console warning rather than a crash).

### 16.6 Things that were NOT the cause (ruled out, don't re-chase)

The user had enabled two new 1827 settings; both were investigated and
are **red herrings** for this bug:

- **SubTickTiming** — chops the work block into sub-tick-sized pieces,
  so `Work()` is now called with variable, smaller `nSamples`
  (observed: 256, 202, 54, 18...) multiple times per tick. This
  changes block sizing but not note delivery. Our per-block math and
  newRow detection (via `Song.PlayPosition`) handle variable `n`
  correctly.
- **AudioBufferFillThread** — a background output-buffer fill in
  `CommonAudioProvider`, a layer above per-machine Work dispatch.
  Doesn't touch the multi-out contract.

The actual cause was purely the `pvalues` field-type change, which is
independent of both settings.
