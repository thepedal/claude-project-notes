# Pedal Presetter — State Notes

**Status:** v1.0 shipped. Single-file managed control machine. Targets
ReBuzz 1821-preview onwards (any version with `BuzzGUI.Common.dll`
exposing the `MachineExtensions` preset API — i.e., all current builds).

Cross-references throughout point at:

- `Core` → `ReBuzz_ManagedMachine_Notes_Core.md`
- `Build` → `ReBuzz_ManagedMachine_Notes_Build.md`
- `Muter` → `ReBuzz_ManagedMachine_Notes_PedalMuter.md`
- `Presets` → `ReBuzz_ManagedMachine_Notes_Presets.md`

---

## 1. What it does

Drives preset selection on foreign machines from the pattern editor.
Each track is assigned to a foreign target (via right-click `Commands`
menu); a per-track `Preset` byte parameter triggers preset apply on
that target when written from a pattern row. Empty cells do nothing.
Out-of-range indices fire the target's defaults preset.

The pattern editor's middle status-bar field shows the resolved preset
name(s) on cursor hover, via `DescribeValue`.

## 2. Architectural choices worth remembering

### 2.1 No file-walking; used the host's preset API

`BuzzGUI.Common.InterfaceExtensions.MachineExtensions` exposes
`GetPresetNames`, `GetPreset`, `GetPresetDictionary` as extension
methods on `IMachine`, plus the public `Preset.Apply(IMachine, bool)`
method. These do their own lazy disk read of `.prs.xml` and cache in a
process-static dictionary that's **shared with ReBuzz's own UI**
(`Presets §2`).

The dead-end was `IMachineDLL.Presets` — declared on the public
interface but never populated in ReBuzz core (`Presets §1.1`). The test
suite asserts it's null on a fresh machine.

This avoided ~150 lines of file-walking, path-discovery, and parameter
matching code; we get the host's index/name fallback behaviour
identical to its right-click preset menu for free.

### 2.2 Target assignment via right-click `Commands` menu

No GUI. Per-track submenus listing all foreign machines in the song,
each checkable. Persisted via `MachineState` (`PresetterState.Targets`,
string array of target names). Validate-on-use resolution survives
delete and rename — the resolver prefers the cached `IMachine` ref if
it's still in the graph (handles rename), falls back to name lookup
otherwise (handles graph reload).

### 2.3 UI-thread dispatch for `Preset.Apply`

`Preset.Apply` writes `IMachine.Data` (cross-process for native
targets) and can `MessageBox.Show` on exception paths. Both require UI
thread. `SetPreset` runs on the audio thread, captures `(track,
presetIndex)` as value types, dispatches via
`Application.Current.Dispatcher.BeginInvoke` (`Presets §5`). Same
pattern as `Muter §1.1`.

### 2.4 pvalues-polling fallback for `parametersChanged` collision

`Core §14`: when multiple tracks write the same parameter in one tick,
ReBuzz delivers our setter only for the last-written track. The fix is
the standard reflection probe of `ParameterCore.pvalues` (a
`ConcurrentDictionary<int, int>`), inspected from inside `SetPreset` to
recover the per-track values for this tick.

Verified during build: `MachineWorkInstance.cs:600-614` resets all
pvalues to NoValue after `parametersChanged.Clear()`, so values
observed inside `SetPreset` are guaranteed to be this-tick (not stale
from prior ticks). The reset path runs for both native and managed
machines — the bypass at `WorkManager.UpdateNonStateParametersToDefault`
is a separate mechanism for native machines only.

Confirmed playback path for managed-machine targets:
`ModernPatternEditor/PlayRecordManager.cs:232` calls
`Editor.cb.ControlChange(machine, group, track, param, value)` per
event, which lands in `ModernPatternEditorMachine.cs:114` and calls
`parameter.SetValue(track | (1<<16), value)`. That hits
`ParameterCore.SetValue` at line 332 (writes `pvalues[track]`) and
line 328 (writes `parametersChanged[parameter] = track`). pvalues
accumulates all per-track values; parametersChanged collapses to the
last-written track. Polling is therefore the correct recovery
mechanism — it sees every track's value at setter-fire time.

#### 2.4.1 The probe-retry bug (fixed post-v1.0)

The original v1.0 `TryProbePValues` set
`_pvalueProbeAttempted = true` at the top of the function, *before*
attempting the actual reflection lookup. Intent was "try once even on
failure" — but the consequence was that any first-call failure (e.g.
a transient state where `ParameterGroups` hadn't been fully populated
yet, or any of the other guard returns in the probe path) permanently
disabled polling for the lifetime of the machine instance.
`_ownPresetPValues` stayed null, every subsequent `SetPreset` hit the
single-track fallback path, and only the last-written track in a
multi-track row applied — i.e. the highest-numbered track.

The fix: retry until success. Replace the "attempted" flag with a
success check on `_pvaluesField != null` (see §2.4.2 for why this is
FieldInfo rather than the value). Use local variables inside the try
block; only commit `_ownPresetParam`, `_pvaluesField`, and
`_pvaluesIsArray` to the instance fields after the entire probe path
succeeds. A `_probeAttempts` counter accumulates failures so the
success log line can report "succeeded after N attempts" — useful
diagnostic to confirm whether the bug ever manifested in a given
session.

Added diagnostic via `Buzz.DCWriteLine` — one-shot per state so the
debug console shows exactly which path was hit without flooding:

- **Information** `SetPreset first call: ...` — confirms the setter is
  firing at all
- **Information** `ProbeOk: ...` — first successful probe
- **Information** `Polling fired N apply(s) on this tick.` — first time
  polling fires applies
- **Warning** `ProbeFailParamGroups` / `ProbeFailNoPreset` /
  `ProbeFailNoField` / `ProbeFailWrongType` / `ProbeFailException` —
  per-path failure diagnostics with type names and field lists so the
  next debugging session has actionable data
- **Warning** `Probe not yet succeeded — using single-track fallback`
  — first time fallback dispatches

**Critical workflow note:** `ReBuzz/Common/DebugWindow.xaml.cs:50`
reconfigures `Log.Logger` *on debug-window construction*. Any
`DCWriteLine` calls before the user opens the Debug Console go to
Serilog's default no-op sink and are lost. Open the console *first*,
then play the pattern.

Retry cost: ~100 ns of `FieldInfo.GetValue` per probe call, much
less for the success short-circuit path. Worst case in the
field-genuinely-missing scenario is one reflection lookup per pattern
preset event (typically <30/sec) = negligible CPU. No bound on the
retry count — the success check makes successful sessions free, and
unbounded retry tolerates startup races of arbitrary length.

#### 2.4.2 `pvalues` storage shape varies between ReBuzz builds

The `pvalues` field on `ParameterCore` has been observed in two forms
in the field:

- **`int[]`** — current ReBuzz builds (observed in the user's runtime
  during v1.0.1 debugging). Indexed by track; one slot per track up to
  the parameter group's `TrackCount`. Reset to `NoValue` post-tick by
  the same mechanism (`MachineWorkInstance.cs`).
- **`ConcurrentDictionary<int, int>`** — older / 1821-preview source.
  Same semantics, looked up by track key.

The polling implementation captures the `FieldInfo` (not the value)
during the probe and re-reads through it on every setter call. This
serves two purposes: it transparently supports either shape via a
shape flag set at probe time, and it survives reallocation of the
backing storage when `TrackCount` changes (the array gets replaced;
holding a stale reference would silently miss writes).

If a future ReBuzz build changes the shape again, the probe's
`ProbeFailWrongType` diagnostic will report the new declared type and
we extend the polling branch accordingly. The two-shape branching has
been tested against the `int[]` form; the `ConcurrentDictionary` form
is kept for compatibility with older installs.

#### 2.4.3 The default-zero / pre-reset gate

When the polling code first went live against the `int[]` shape it
fired 16 applies on the very first tick of pattern playback for a
single-track write. Investigation:

- The `int[]` is allocated default-initialised to all zeros.
- The post-tick reset (`MachineWorkInstance.cs:611`) writes `NoValue`
  to slots `0..TrackCount-1` — but only *at end of tick*, AFTER our
  setter fires.
- Slots `TrackCount..arr.Length-1` are *never* reset.

So during the very first setter call of a session, every slot is at
default zero. Zero is also a perfectly valid pattern-write (it means
"fire preset 0, the first preset in the bank"). The filter
`pv != noVal && pv >= 0 && pv <= PRESET_MAX` cannot distinguish
"slot at default zero" from "pattern wrote preset zero" — and so
fires for every slot in `0..arr.Length`.

The fix is a two-part gate:

1. **Clamp polling range to `TrackCount`**, not `arr.Length`. This
   bounds out the never-reset slots beyond `TrackCount` permanently.
2. **Latch a `_resetHasRun` flag** the first time any slot in
   `0..upper` is observed as `NoValue`. That observation is the
   proof-positive that the post-tick reset has fired for this
   parameter at least once. Until the flag latches true, polling is
   suppressed and only the setter-arg track applies — the standard
   single-track fallback path.

Trade-off: on the very first tick of pattern playback after machine
creation (specifically, the first tick that fires our setter at all),
multi-track preset writes get reduced to the last-written track. From
the *next* tick onward (after this tick's post-tick reset clears
slots `0..TC-1` to `NoValue`), polling becomes reliable and
multi-track writes fire fully. In a looping song, even row 0
multi-track writes work correctly on the second loop iteration; the
issue is genuinely limited to the very first row of the very first
playback.

This is fundamental, not a workaround: there is no way to distinguish
default-zero from pattern-wrote-zero without observing the reset
mechanism, because the `int[]` itself doesn't carry any "this slot
has been written" sidecar state. The only stronger alternative would
be hooking the parameter's `SetValue` via reflection, which trades a
contained limitation for a much larger maintenance liability against
ReBuzz internals.

Diagnostic logs (one-shot, requires Debug Console open before
playback):

- `Post-tick reset observed (slot N = NoValue)` — flag has latched
- `Polling fired N apply(s)` — first reliable polling call
- `Pre-reset state on first preset row: ...` — pre-reset path fired

#### 2.4.4 Pre-reset state — the slot>0 heuristic refinement

The §2.4.3 gate originally fell back to "setter args only" while the
reset hadn't been observed. For users whose first preset row of
playback is a multi-track write (the originally-reported scenario),
this meant the first row still exhibited the "highest track wins"
behaviour they were trying to fix.

The heuristic on the pre-reset path: alongside firing the setter-args
track unconditionally, also sweep all other slots in `0..upper` and
fire applies for any with `pv > 0`. Non-zero values can only come
from a real pattern write (the `int[]` default is 0), so this catches
multi-track writes whose presets are anything other than 0.

The remaining miss is narrower: a preset-0 write to a non-setter-arg
track on the very first preset row of playback. From any subsequent
preset row onward, the reset has fired between rows and polling
becomes reliable for all presets including 0.

This is fundamental and cannot be tightened further without observing
the SetValue calls directly (e.g. by hooking ParameterCore via
reflection) — a trade-off that adds significant maintenance liability
against ReBuzz internals for a narrow edge case.

There is also a degenerate "always-all-tracks-written" case where the
reset signal (`NoValue` in some slot) is never observable — every
slot in `0..TC-1` is always overwritten between resets. The pre-reset
heuristic continues indefinitely in that case, which means preset-0
writes to non-setter-arg tracks are persistently missed. Realistic
usage rarely hits this because patterns are typically sparse, but
it's worth knowing.

### 2.5 Self-target guard

`ReferenceEquals(machine, host.Machine)` check at both assignment time
(in `AssignTarget`) and apply time (in `ApplyPresetCore`). Same defence
as `Muter §1.3`. The double-check at apply catches the edge case of
"user pointed me at machine X, renamed X to my own name, deleted me
and recreated me with the old name."

### 2.6 `DescribeValue` for the status bar

Reflected per `IBuzzMachine.cs` Update 1 / `ManagedMachineHost.cs:255`.
Called by `ModernPatternEditor.UpdateStatusBar` on cursor moves;
populates the middle field as `"<vs> (<v>) <DescribeValue>"`.

The signature is `(IParameter, int value)` — no track. Per-target
preset names mean we can't disambiguate by signature alone. Two
approaches considered:

- Reflect into `ModernPatternEditor.CursorPosition` to recover the
  track. Rejected: ties us to ModernPatternEditor internals; fails for
  any non-status-bar caller (tooltips, value pickers, undo
  descriptions).
- Summarise across all assigned tracks. Picked. Deterministic across
  call contexts, degrades to a single name when only one track is
  assigned.

Format: empty → null (default formatting); all-same → just the name;
mixed → `"T0:name1 T2:name2 ..."`.

## 3. File layout

```
PedalPresetter/
├── PedalPresetter.cs        (~460 lines, single file)
├── PedalPresetter.csproj
└── README.md
```

All build hygiene per `Build §1.2`: `DebugType=none`,
`DebugSymbols=false`, `GenerateDependencyFile=false`. AssemblyName
`Pedal Presetter.NET` (the `.NET` suffix is the managed-machine loader
signal per `Build §2`). Post-build deploys to `C:\Program
Files\ReBuzz\Gear\Generators`.

Two references, both `Private=false` to share the host's loaded
assemblies (critical for the static preset cache sharing — see
`Presets §6`):

- `BuzzGUI.Interfaces.dll`
- `BuzzGUI.Common.dll`

## 4. Known limitations

- **No GUI.** Target assignment is right-click only. v2 candidate: an
  embedded `IMachineGUIFactory` panel showing all 16 tracks with live
  target + last-fired-preset display, dropdowns for reassignment.
- **No MIDI driving.** Pattern editor is the only event source. A
  MIDI-program-change → preset-index mode would be a reasonable add.
- **No fire-and-hold semantics.** Preset changes are events; the
  pattern shows the firing row, not the held state. Practically this
  works because the target's parameter values persist until something
  else changes them, but a "current preset N" highlight in the pattern
  editor isn't surfaceable from our side.
- **Status-bar field is ~150 px.** Multi-track mixed-target summaries
  can overflow. Acceptable — the v2 GUI would carry the full display.
- **Renaming the target during a session, then saving and reloading,
  works.** Deleting the target during a session silently no-ops on
  apply (correct). Deleting and recreating the target with a different
  name requires manual reassignment via the menu.

## 5. v2 ideas, ordered by likely value

1. **Embedded GUI panel** — live status for all tracks. Biggest UX
   improvement, modest implementation cost (~300 lines).
2. **MIDI program change driving** — assign a target a MIDI channel,
   program-change messages select presets.
3. **"Apply on assign"** — option to fire the current pattern row's
   preset value when a target is freshly assigned, so the user doesn't
   have to wait for the next row to hear the change.
4. **Preset name caching** — currently `DescribeValue` calls
   `GetPresetNames` on every cursor move. Already cheap (warm dict
   lookup), but a per-target preset-names cache on our side keyed by
   `IMachine` would shave the LINQ work. Premature optimisation
   unless profiling flags it.

## 6. Most-relevant source pointers

In the ReBuzz 1821-preview source (paths from repo root):

- `ReBuzzGUI/BuzzGUI.Common/InterfaceExtensions/MachineExtensions.cs`
  — `GetPresetNames`, `GetPreset`, `GetPresetDictionary` extension
  methods; static `presetDictionaries` cache at line 69.
- `ReBuzzGUI/BuzzGUI.Common/Presets/Preset.cs:308` — `Apply` method
  with fast and fallback parameter-matching paths.
- `ReBuzzGUI/BuzzGUI.Common/Presets/PresetDictionary.cs` — `Load`
  parses `.prs.xml` into a serializable dict.
- `ReBuzzGUI/BuzzGUI.ParameterWindow/Actions/SelectPresetAction.cs:21`
  — confirms `Preset.Apply` is exactly what ReBuzz's own preset-select
  UI calls.
- `ReBuzz/ManagedMachine/ManagedMachineHost.cs:255` — `DescribeValue`
  reflection probe.
- `ReBuzz/ManagedMachine/ManagedMachineHost.cs:263-278` — `Commands`
  property reflection probe.
- `ReBuzz/ManagedMachine/ManagedMachineHost.cs:317-360` —
  `MachineState` XML serialization.
- `ModernPatternEditor/ModernPatternEditor.NET/PatternControl.xaml.cs:1380-1401`
  — `UpdateStatusBar`, the consumer of `DescribeValue`.
- `ReBuzz/Core/ParameterCore.cs:164` — `pvalues` field for the polling
  probe.
- `ReBuzz/MachineManagement/MachineWorkInstance.cs:595-614` —
  parametersChanged dispatch + pvalues reset confirmation.
