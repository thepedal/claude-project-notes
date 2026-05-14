# ReBuzz Managed Machine Development — Pedal Muter Addendum

Source: Pedal Muter v1.0 build (a managed C# control machine that drives
`IMachine.IsMuted` on foreign targets).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. Internal cross-references use plain
`§N` and `§N.M` and stay within this file.

---

## 1. Driving `IMachine.IsMuted` on foreign machines

A control machine can mute any other machine by writing to its `IsMuted`
property. The write must happen on the WPF UI thread — like the other
machine-state operations covered in Core §21, calling the setter from
the audio thread can deadlock. There's also one concern unique to a
controller whose own mute state is automation-driven: when the controller
is itself bypassed, its `Work()` stops running but its parameter setters
do not.

### 1.1 The UI-thread-safe `IsMuted` setter

PedalMuter wraps every mute write in a small helper that dispatches to
the UI thread when needed and short-circuits if the target is already
in the requested state:

```csharp
static void SetMachineMutedUiSafe(IMachine m, bool muted)
{
    if (m == null) return;
    try { if (m.IsMuted == muted) return; } catch { return; }   // idempotent

    var d = System.Windows.Application.Current?.Dispatcher;
    if (d == null || d.CheckAccess())
    {
        try { m.IsMuted = muted; } catch { }
    }
    else
    {
        d.BeginInvoke(new Action(() =>
        {
            try { if (m != null) m.IsMuted = muted; } catch { }
        }));
    }
}
```

Idempotent short-circuit on `m.IsMuted == muted` makes it cheap to call
every `Work()` cycle as a re-assertion safety net.

### 1.2 The setter must dispatch immediately

If the controller's mute state is itself driven by pattern automation,
the controller can become bypassed. ReBuzz's zero-CPU bypass skips
`Work()` entirely while `IsMuted == true`, but `Tick()` (and therefore
your parameter setters) still runs. If the setter only marks state and
defers application to `Work()`, mute changes during the controller's
own bypass period **never reach the targets** — `Work()` never runs to
apply them.

The fix is to dispatch from the setter itself, not the work loop:

```csharp
public void SetMute(int value, int track) {
    var ts = _tracks[track];
    if (ts.MuteState == newState) return;
    ts.MuteState = newState;
    ApplyTrack(ts);   // dispatch IsMuted via UI thread NOW
}
```

`Work()` should still re-assert as a safety net (idempotent calls are
free), but the *primary* application path is the setter.

### 1.3 Refuse to mute yourself

A controller that can mute foreign machines can mute itself if its host
machine appears in its own assignment list. This freezes the controller
permanently — `Work()` stops, the setter that would unmute also stops
running (under bypass). Always check `ReferenceEquals(target, host.Machine)`
in the apply path and skip.

---

## 2. The stale-voice problem on mute/unmute

A new failure mode unique to driving `IsMuted` from automation: voices
frozen inside the target plugin's internal state across the bypass span.

### 2.1 What happens

With *Settings → Audio → Process muted machines* unchecked (the whole
point of using mute as a CPU-saving primitive), ReBuzz's bypass:

- excludes the target from `CollectMachinesThatCanWork` (no `WorkMachine`)
- which also means no `TickAndWork` (so no SCC-triggered Tick either)
- regular `AudioTick` from `CallTick()` may also skip muted machines
  in some build configs

The plugin process receives no parameter writes, no audio rendering
calls. Whatever voice was active at the moment of bypass — mid-attack,
mid-sustain, mid-release — **freezes in place**: envelope phase, sample
position, filter state, all preserved. The plugin's internal time
doesn't advance.

When `IsMuted = false` lands later (after some user-controlled span,
typically tens to hundreds of ticks), `WorkMachine` resumes calling
the plugin and the voice continues from the frozen state. The user
hears a phantom re-trigger of the note from earlier.

### 2.2 Two-sided Note-Off flush

A single Note-Off either at mute or at unmute is insufficient:

- **Mute-time only:** the plugin enters release before bypass engages,
  but for any voice with non-trivial release length, bypass freezes the
  release envelope mid-tail. Unmute then resumes the unfinished release
  and the user hears a long fade-out at full pre-bypass amplitude.

- **Unmute-time only:** the first flush attempt likely runs while
  `IsMuted` is still true (`BeginInvoke` dispatch is async, takes 1–N
  buffers to land). During that window the target is excluded from the
  work list, SCC writes don't fire, and `UpdateNonStateParametersToDefault`
  wipes pvalues at end-of-buffer. The flush silently no-ops.

The combination works:

```csharp
public void SetMute(int value, int track) {
    var ts = _tracks[track];
    if (ts.MuteState == newState) return;
    bool isMuting   = !ts.MuteState && newState;
    bool isUnmuting =  ts.MuteState && !newState;
    ts.MuteState = newState;

    if (isMuting)         FlushTargetsNoteOff(ts);          // immediate
    else if (isUnmuting)  ts.PostUnmuteFlushBuffers = 32;   // armed counter

    ApplyTrack(ts);   // dispatch IsMuted change
}

public void Work() {
    foreach (var ts in _tracks) {
        ApplyTrack(ts);   // re-assert IsMuted
        if (ts.PostUnmuteFlushBuffers > 0) {
            ts.PostUnmuteFlushBuffers--;
            FlushTargetsNoteOff(ts);   // re-send Note-Off + SCC
        }
    }
}
```

### 2.3 Multi-buffer flush spans the dispatch race

The post-unmute counter must span the worst-case `BeginInvoke` dispatch
latency. Empirically, 4 buffers (~20–40 ms) is too short — the UI thread
can stall longer than that under load, and long-release pads still leak
on the rare slow dispatch. 32 buffers (~200–400 ms) covers comfortably
without being audible if no fresh notes are arriving.

The constant should be left in user-tunable form at the top of the file —
some plugins need 64+, especially polyphonic pads with multi-second
attacks where the voice is still in attack at unmute time.

### 2.4 The flush itself

```csharp
static void FlushStaleVoicesOnMachine(IMachine target) {
    var noteParam = FindNoteParam(target);
    if (noteParam == null) return;

    int tc = Math.Max(16, target.TrackCount);
    for (int t = 0; t < tc; t++) {
        try { noteParam.SetValue(t, 255); } catch { }   // 255 = Note-Off
    }
    try { target.SendControlChanges(); } catch { }
}
```

`FindNoteParam` does a multi-pass search: explicit `ParameterType.Note`
first (managed targets), then `ParameterGroups[2].Parameters[0]` (the
standard Buzz layout for native generators), then any
`ParameterGroupType.Track`. Falls through to no-op for effects without
a note parameter.

Note: `pt_note = 255` is the *standard* Buzz Note-Off but not universal.
A few synths treat 255 as "no event" rather than "release voice"; if
even 64+ flush buffers don't help, the plugin needs a Velocity/Gate
parameter zeroed instead. Diagnose by writing a quick test pattern
that triggers a long sustain note and confirming a manual mid-pattern
255 row releases it; if not, the synth doesn't honour pt_note Note-Off.

---

## 3. `IParameter.GetValue()` does NOT return the current pvalue

A subtle footgun, discovered during the v1.0 stale-voice flush work.

### 3.1 The mistaken assumption

It's tempting to gate parameter writes on what's "currently" in the
parameter:

```csharp
// WRONG — silently always-true (or always-false), depending on prior state
int curr = noteParam.GetValue(track);
if (curr == noteParam.NoValue)   // assumes "no event this row"
    noteParam.SetValue(track, 255);
```

The intent: only kill the voice if the pattern isn't delivering a
real note this row.

### 3.2 What `GetValue` actually returns

`IParameter.GetValue(track)` returns the parameter's last actually-played
value, not the current `pvalues` dictionary entry. For a Note parameter,
that means it returns 60 (or whatever) for "the last actually-played
note on this track", not 0 (NoValue) for "no event this row".

The pvalue dictionary entry — what `AudioTick` reads from — is
write-only through the public `IParameter` interface; there's no
official getter for "what's currently in the pvalue slot for this tick".

### 3.3 Consequence

Any logic that gates SetValue calls on GetValue == NoValue is a no-op
in the common case. Either:

- **Drop the gate entirely** and accept that you might overwrite a
  concurrent note (often fine — the use cases for these gates tend to
  be edge cases like "don't kill a fresh trigger on the mute row").

- **Read the pvalues dictionary directly** via reflection on
  `ParameterCore.pvalues` (`ConcurrentDictionary<int,int>`) as
  documented in Core §14 for inspecting *your own* pvalues. This works
  on *foreign* parameters too, with the same caveats: lazy init,
  allocation-free `TryGetValue`, no `Invoke` on the audio thread.
  §4 below is the canonical worked example of this path.

For the Pedal Muter v1.0 stale-voice flush, the chosen approach was
"drop the gate" — the flush only runs at mute/unmute transitions, and
killing a concurrent fresh trigger on those exact rows is a worthwhile
trade for a flush that actually fires.

### 3.4 Diagnostic value

When debugging "this SetValue isn't doing anything", verify with a
DCWriteLine before *and* after the SetValue, reading from the pvalues
dictionary via reflection — not via `GetValue`. If GetValue says the
write took effect but the plugin doesn't respond, the issue isn't the
write; it's the read side (AudioTick skipped due to mute, or plugin
ignoring the value).

---

## 4. The Core §14 trap hits Mute-driving control machines hard

The `parametersChanged` overwrite documented in Core §14 / Tracker §1.1
isn't just about Note delivery on chord rows — it bites *any* managed
machine whose single parameter is being written across multiple tracks
in one tick. Pedal Muter discovered this the hard way during v1.0
field testing.

### 4.1 Symptom

User has Pedal Muter with 6 tracks active, each assigned to a different
target machine, each driven by its own PeerCtrl track. They flip all
six PeerCtrl sliders simultaneously (or hit the "Unmute All Targets"
menu, or have pattern rows that write Mute on multiple tracks at the
same row).

Result: **only one target changes state.** Specifically, the *last*
track in iteration order. The other five stay where they were.

The user reproduces it with the panic-button menu: each click unmutes
one more target, working backwards through the track list. That's the
smoking-gun signature — multiple `SetValue` calls collapsing to one
setter call, every time, with the iteration-last writer winning.

### 4.2 Root cause

`SetValue(track=N, value=v)` writes `pvalues[N]=v` AND
`parametersChanged[muteParam]=N`. Six writes in succession leave
`pvalues[0..5]` all updated but `parametersChanged[muteParam] = 5`
only. The managed-host `Tick()` then calls `SetMute(value, track=5)`
once, and the other five writes vanish silently. The C#-side state for
tracks 0–4 doesn't update, `ApplyTrack` never runs for them, target
machines keep their old `IsMuted`.

This is the same mechanic as Core §14's chord-row trap, except the
"chord" here is "six tracks of the same control parameter changing in
one tick" rather than "six tracks of Note changing in one tick".

### 4.3 Fix — §14 recovery, but for switch parameters

Same shape as the Tracker §14 workaround for `Note` polling, with one
notable simplification: Mute is a `Byte`-typed switch parameter with
discrete values 0/1, not a 0..156 note. Range-check pvalues before
applying:

```csharp
void PollSiblingMutePValues(int firedTrack)
{
    if (!EnsurePollSetup()) return;     // lazy reflection bootstrap
    int noValue = _ownMuteParam.NoValue;

    foreach (var kv in _ownMutePValues)
    {
        int track  = kv.Key;
        int pvalue = kv.Value;
        if (track == firedTrack) continue;
        if ((uint)track >= MAX_TRACKS) continue;
        if (pvalue == noValue) continue;
        if (pvalue != 0 && pvalue != 1) continue;     // sanity guard for switch

        bool desired = (pvalue == 1);
        var ts = _tracks[track];
        if (ts.MuteState == desired) continue;        // already in sync

        ApplyMuteTransition(track, desired);          // factored helper
    }
}
```

Call from `SetMute` after the main transition, **including** on the
"already in sync" early-return path — because the *fired* track being
a no-op doesn't mean sibling tracks were no-ops:

```csharp
public void SetMute(int value, int track)
{
    // ... bounds & lazy resolve ...
    bool newState = (value == 1);
    var ts = _tracks[track];
    if (ts.MuteState == newState) {
        PollSiblingMutePValues(track);   // still poll!
        return;
    }
    ApplyMuteTransition(track, newState);
    PollSiblingMutePValues(track);
}
```

`ApplyMuteTransition` is the factored helper containing the
mute/unmute flush dispatch and `ApplyTrack` call, so the recovery path
gets the exact same Note-Off injection, post-unmute counter, and
`IsMuted` dispatch as the originally-fired track.

### 4.4 Reflection setup is identical to Tracker §1.3

Lazy bootstrap, walk the type hierarchy on `ParameterCore` (the
runtime type behind the `IParameter` reference), find the field named
`pvalues` of type `ConcurrentDictionary<int,int>`, cache the
reference. Same lifecycle: `ParameterGroups` isn't populated until
after the constructor finishes (Core §15), so do this from inside the
first setter call, not from the constructor.

```csharp
IParameter _ownMuteParam;
ConcurrentDictionary<int, int> _ownMutePValues;
bool _pollSetupAttempted;

bool EnsurePollSetup()
{
    if (_ownMutePValues != null) return true;
    if (_pollSetupAttempted) return false;
    _pollSetupAttempted = true;

    var pg = host?.Machine?.ParameterGroups;
    if (pg == null || pg.Count <= 2) return false;
    _ownMuteParam = pg[2]?.Parameters?.FirstOrDefault(p => p?.Name == "Mute");
    if (_ownMuteParam == null) return false;

    var t = _ownMuteParam.GetType();
    while (t != null && _ownMutePValues == null)
    {
        var f = t.GetField("pvalues",
            BindingFlags.Instance | BindingFlags.NonPublic | BindingFlags.Public);
        if (f != null)
            _ownMutePValues = f.GetValue(_ownMuteParam)
                              as ConcurrentDictionary<int, int>;
        t = t.BaseType;
    }
    return _ownMutePValues != null;
}
```

The cached `IParameter` reference and `ConcurrentDictionary<int,int>`
reference both live for the machine's lifetime — no need to refresh
them across saves/loads.

### 4.5 Cascade benefit — three failure modes, one fix

Any code path that issues multiple `SetValue` calls in one tick
benefits from the same recovery:

1. **Peer control fan-out.** PeerCtrl writing to N Pedal Muter tracks
   from inside its own `Work()` lands as N `SetValue` calls in a
   single audio buffer.
2. **Programmatic "do thing to many tracks".** Menu commands like
   "Unmute All Targets" iterating the track array.
3. **Multi-track pattern rows.** Pattern automation with Mute events
   on multiple tracks at the same row.

All three previously exhibited the "only the last track responds"
symptom. The §4.3 polling fixes all three with no code change at the
call sites.

### 4.6 Diagnostic logging that proves the recovery worked

When debugging multi-track writes, log every recovery so DebugView (or
the Trace listener of your choice) shows you the workaround engaging:

```
T+12345ms SetMute track=5 value=0 true→false (UNMUTE) ...
T+12345ms §14 RECOVERY track=0 pvalue=0 true→false ...
T+12345ms §14 RECOVERY track=1 pvalue=0 true→false ...
T+12345ms §14 RECOVERY track=2 pvalue=0 true→false ...
T+12345ms §14 RECOVERY track=3 pvalue=0 true→false ...
T+12345ms §14 RECOVERY track=4 pvalue=0 true→false ...
T+12345ms §14 RECOVERY: applied 5 missed transitions
```

If the recovery line count after a multi-track write doesn't match the
number of expected transitions, either pvalues isn't populated as
expected (reflection bootstrap failed — check the `EnsurePollSetup`
log line) or the writes weren't actually in the same tick.

---

## Summary of constants worth exposing

A few constants that earned their place at the top of source files
during the v1.0 build, in case future control machines hit similar
problems:

| Constant | Default | Range tried | Rationale |
|---|---:|---:|---|
| `STALE_FLUSH_BUFFERS` | 32 | 4 (too short) – 128 (overkill) | Multi-buffer Note-Off flush window after unmute; covers BeginInvoke dispatch race |
| `MAX_TRACKS` (Pedal Muter) | 64 | — | Per-track Mute parameter slots; used as track-count for `MachineDecl(MaxTracks=…)` and the assignment array |
| Track-scan fallback in `FlushStaleVoicesOnMachine` | 16 | — | Lower bound on tracks scanned for Note-Off injection when `target.TrackCount` reports fewer; covers polyphonic synths that haven't bumped their TrackCount yet |
