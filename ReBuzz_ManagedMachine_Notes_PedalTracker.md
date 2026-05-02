# ReBuzz Managed Machine Development — Pedal Tracker Addendum

Source: ReBuzz 1818-preview source code + multi-session debugging of Pedal Tracker
(a managed C# generator modelled on BTDSys Matilde Tracker). These details
extend the Core notes (`ReBuzz_ManagedMachine_Notes_Core.md`) with everything
learned about pattern-data delivery, sub-tick timing, and Matilde-faithful playback.

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. Internal cross-references use plain
`§N` and `§N.M` and stay within this file.

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

### 1.3 The Core §14 workaround must extend beyond Note

The Core notes describe polling `pvalues` from inside `SetNote` to recover
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

