# ReBuzz Managed Machine Development — BTDSys PeerCtrl Addendum

Source: Full port session, May 2026. Original C++ BTDSys PeerCtrl v1.5/1.6
(Ed Powley, BTDSys, 2002–2008) ported to ReBuzz as a managed C# machine.
Repo: https://github.com/thepedal/BTDSys-PeerCtrl-ReBuzz

Released versions: v1.0, v1.1, v1.2, v1.3.

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. No content overlap with Core.

---

## 1. Machine type and deployment

PeerCtrl is a **control machine**: no audio inputs, no audio outputs. It
controls the parameters of other machines via `IParameter.SetValue()` and
responds to MIDI CC input. It does not generate or process audio.

```
InputCount  = 0
OutputCount = 0
```

Deployed to `C:\Program Files\ReBuzz\Gear\Generators\` (ReBuzz places
control machines in the Generators folder, not a separate Controls folder).

DLL name: `BTDSys PeerCtrl.NET.dll`. The `.NET` suffix is required for
ReBuzz to recognise the file as a managed machine.

---

## 2. `IBuzzMachine.Tick()` IS called on this machine

Unlike most managed machines where `Tick()` is dead code (Core §1), PeerCtrl
relies on `Tick()` for resolution retry and first-tick feedback. The reason
it fires here: ReBuzz calls `Tick()` on machines that are in the song graph
even if they produce no audio, as long as the song is playing.

**Critical:** `Tick()` is only called when the song is playing. It is
NOT called when the song is stopped or when the user is just moving
sliders interactively. All interactive value changes (slider drag, MIDI CC)
must therefore work without any dependency on `Tick()` having been called.

The `_initialising` flag is cleared in `SetValue()` (the first parameter
setter call after load), not in `Tick()`, for exactly this reason.

---

## 3. `Work()` is called despite no audio I/O

ReBuzz calls `Work()` on PeerCtrl even though it has zero inputs and
outputs. `Work()` returns `false` (no audio output). The sub-tick inertia
interpolation runs inside `Work()` via the `SendFreq` counter.

Do not attempt to remove `Work()` — the machine will fail to load.

---

## 4. `IParameter.SetValue()` on another machine's parameters

The primary mechanism for controlling other machines. Called in
`TrackAssignment.ApplyValue()`.

**Critical: never cache `IParameter` references across sessions.** ReBuzz
recreates `IParameter`/`IMachine` objects when a song is loaded. Cached
references from a previous load are stale and silently do nothing when
`SetValue()` is called on them. PeerCtrl resolves fresh on every `ApplyValue()`
call:

```csharp
var machine = buzz.Song.Machines.FirstOrDefault(m => m.Name == _machineName);
var param   = GetAllParams(machine)[_paramIndex];
param.SetValue(trackIndex, pval);
```

This is slightly slower than caching but is the only reliable approach across
save/reload cycles. Measured overhead is negligible for a control machine.

---

## 5. State persistence via `MachineState` / XML

The property must be named exactly `MachineState` and the type name must match
what ReBuzz uses for (de)serialisation. ReBuzz discovers the property by
reflection and serialises it as XML into the `.bmx` song file.

```csharp
public PeerCtrlState MachineState { get; set; }
```

### 5.1 `XmlSerializer` and `List<T>` accumulation bug

When a class has a `List<T>` property initialised to a non-null list in the
property declaration, `XmlSerializer` **adds** to the existing list rather
than replacing it. This produces duplicate/crossed points in the mapping
curve on every reload.

Fix: initialise `List<T>` properties to `null` so XmlSerializer replaces
rather than appending:

```csharp
// WRONG — XmlSerializer adds to this list, duplicating points on reload
public List<EnvPoint> Points { get; set; } = new List<EnvPoint>();

// CORRECT — XmlSerializer replaces null cleanly
public List<EnvPoint> Points { get; set; } = null;
```

Provide a fallback wherever the property is read:

```csharp
List<EnvPoint> ActivePoints => Points ?? DefaultPoints();
```

### 5.2 Saving per-track values for feedback restoration

ReBuzz restores parameter values by calling the machine's `SetValue()` setter
during song load. However, `SetValue()` may be called before `MachineState`
is applied (ordering is not guaranteed), and the song may not be playing
(so `Tick()` never fires). To reliably restore MIDI feedback positions,
save the current Value for each track explicitly in `PeerCtrlState`:

```csharp
// In MachineState getter:
st.SavedTrackValues.Add((int)(_tracks[t].ValueCurrent * 65534f + 0.5f));

// In MachineState setter:
float v = Clamp01(value.SavedTrackValues[t] / 65534.0f);
_tracks[t].ValueCurrent = _tracks[t].ValueTarget = v;
```

On the first `Tick()` after load, send MIDI feedback from `ValueCurrent`
(the saved value) before `SyncFromParamValues()` can overwrite it with
whatever the physical controller is currently sending.

---

## 6. `_initialising` flag pattern

The flag guards `SendNow()` from firing during song load before machines are
resolved. Unlike other managed machines, clearing it in `Work()` or `Tick()`
is NOT sufficient — both may be called too late or not at all when the song
is stopped.

Clear `_initialising` in **`SetValue()`** (the first real parameter setter
call, which always happens during or after load):

```csharp
public void SetValue(int value, int track)
{
    if (_initialising)
    {
        _initialising = false;
        ResolveAllMachines();
    }
    ValueChange(_tracks[track], track, value / 65534.0f);
}
```

Also clear it in `MachineState.set` since that is called before `SetValue()`
in some load orderings.

---

## 7. MIDI feedback — sending CC to hardware controllers

Uses NAudio.Midi (shipped with ReBuzz) to send CC messages back to hardware.

```xml
<Reference Include="NAudio.Midi">
  <HintPath>$(ReBuzzDir)\NAudio.Midi.dll</HintPath>
  <Private>false</Private>
</Reference>
```

Key rules:

- **Never cache `MidiOut` instances per-send** — open/close on every message
  causes noticeable latency. Keep a static `Dictionary<int, MidiOut>` keyed
  by device index and reuse open handles.
- **Thread-lock the dictionary** — `MidiControlChange()` runs on the audio
  thread; feedback sends can happen from multiple paths. Use `lock (_lock)`.
- **Send to all output devices** — avoids the need for a device selector UI
  and works regardless of which port the controller is connected to.
- **Deduplicate** — track `LastFeedbackSent` (0–127) per assignment; only
  send when the CC value actually changes. Without this, every `Tick()` floods
  the controller.
- **Suppress feedback echo** — when a value change originates from MIDI CC
  (`fromMidi=true`), pass `sendFeedback: false` to `ApplyValue()`. The
  controller already knows its own position; echoing it back causes a feedback
  loop that makes LED rings oscillate continuously.

CC message encoding for NAudio:

```csharp
static int MakeCC(int ctrl, int val, int ch) =>
    (0xB0 | ((ch - 1) & 0x0F)) | (ctrl << 8) | (val << 16);
midiOut.Send(MakeCC(controller, ccVal, channel));
```

### 7.1 BCR2000 note

The Behringer BCR2000 uses **absolute CC mode** (sends 0–127). The PeerCtrl
`Inc/Dec (96/97)` option is specifically for the Doepfer Pocket Dial
protocol (relative CC 96/97 with the controller number in the value byte).
Do not enable Inc/Dec for BCR2000.

---

## 8. Machine graph resolution

`IParameter` and `IMachine` references become stale after song reload.
`ResolveAllMachines()` re-resolves all assignments by name lookup:

```csharp
var machine = buzz.Song.Machines.FirstOrDefault(m => m.Name == a.MachineName);
var allp    = GetAllParams(machine);
a.ResolvedParam = allp[a.ParamIndex];
```

`GetAllParams()` returns a flat list skipping group 0 (internal params):

```csharp
public static List<IParameter> GetAllParams(IMachine m)
{
    var list = new List<IParameter>();
    for (int g = 1; g < m.ParameterGroups.Count; g++)
        list.AddRange(m.ParameterGroups[g].Parameters);
    return list;
}
```

The `ParamIndex` stored in `TrackAssignment` is an index into this flat list.
It is stable as long as the target machine's parameter group structure does
not change between sessions. `MachineName` (string) is the persistent key;
`ResolvedParam` is a runtime-only reference marked `[XmlIgnore]`.

On song load, resolution is retried every `Tick()` until all assignments
succeed — the target machine may not yet be in the graph when
`MachineState.set` is first called.

---

## 9. `SyncFromParamValues()` — polling own parameters each tick

Because `Tick()` is not called when the song is stopped, and `SetValue()`
may be called by ReBuzz to restore parameter state without the machine
having been running, PeerCtrl reads back its own `Value` parameter each
`Tick()` via `IParameter.GetValue(track)` and re-applies to targets if the
value has changed:

```csharp
IParameter valueParam = trackGroup.Parameters.First(p => p.Name == "Value");
int raw = valueParam.GetValue(t);
if (raw == valueParam.NoValue) continue;
float v = Clamp01(raw / 65534.0f);
if (Math.Abs(v - ts.ValueCurrent) > 0.00001f || ts.LastSent < 0f)
{
    ts.ValueCurrent = ts.ValueTarget = v;
    SendNow(ts);
}
```

This handles song reload correctly regardless of the ordering of
`MachineState.set` vs `SetValue()` calls.

---

## 10. MIDI CC input and the `fromMidi` flag

`MidiControlChange(ctrl, channel, value)` is called by ReBuzz on the audio
thread for every incoming MIDI CC. The `fromMidi` flag threads through the
entire value-change path to suppress feedback echo:

```
MidiControlChange()
  └─ ValueChange(ts, t, v, fromMidi: true)
       └─ SendNow(ts, fromMidi: true)
            └─ ApplyValue(v, buzz, sendFeedback: false)   ← no echo
```

For non-MIDI value changes (slider, pattern):

```
SetValue()
  └─ ValueChange(ts, t, v, fromMidi: false)
       └─ SendNow(ts, fromMidi: false)
            └─ ApplyValue(v, buzz, sendFeedback: true)    ← sends CC
```

The MIDI path also writes back to the machine's own `Value` parameter so the
pattern editor slider follows the hardware control visually.

---

## 11. `IMenuItem` / context menu — `DependencyObject` workaround

WPF's binding system resolves `IsChecked` through `TypeDescriptor`, which
sees the interface's getter-only property declaration even when the concrete
class adds a setter. This causes a `TwoWay binding on read-only property`
exception at runtime.

Fix: derive the menu entry class from `DependencyObject` and use a
`DependencyProperty` for `IsChecked`. `DependencyProperty` is always
TwoWay-bindable regardless of interface constraints:

```csharp
sealed class MenuEntry : DependencyObject, IMenuItem, INotifyPropertyChanged
{
    public static readonly DependencyProperty IsCheckedProperty =
        DependencyProperty.Register("IsChecked", typeof(bool), typeof(MenuEntry),
            new PropertyMetadata(false));

    public bool IsChecked
    {
        get => (bool)GetValue(IsCheckedProperty);
        set => SetValue(IsCheckedProperty, value);
    }
    // ... other IMenuItem members
}
```

---

## 12. Settings dialog — STA thread

The settings dialog is a WPF `Window` opened on a dedicated STA thread so it
doesn't block the audio thread and runs independently of ReBuzz's UI thread:

```csharp
var thread = new Thread(() => { new SettingsWindow(this).ShowDialog(); });
thread.SetApartmentState(ApartmentState.STA);
thread.IsBackground = true;
thread.Start();
```

Single-instance enforcement: store `_settingsWindow` on the machine. If it
is non-null, call `_settingsWindow.Dispatcher.Invoke(() => _settingsWindow.Activate())`
to bring the existing window to the front. Never call `Activate()` directly
from a different thread — it will throw `InvalidOperationException`.

---

## 13. `DarkCombo` — custom dropdown control

WPF's `ComboBox` in .NET 10 uses a `ControlTemplate` with hardcoded brush
resource keys (`ComboBox.Static.Background`, etc.) that cannot be overridden
via `Style`, `ItemContainerStyle`, or `SystemColors` resource dictionary
entries. Every approach short of a full `ControlTemplate` replacement leaves
the closed-state text area white.

Solution: replace `ComboBox` entirely with a custom `ContentControl` subclass
(`DarkCombo`) backed by a `Border` (header) + `Popup` (dropdown list) +
`ListBox` (items).

### 13.1 Keyboard navigation — the cross-HwndSource problem

WPF `Popup` creates a separate Win32 `HwndSource` (its own OS-level window).
Windows routes keyboard messages to whichever Win32 window has OS focus.

- If the Popup's ListBox is given focus → keyboard messages go to the Popup's
  HwndSource → the parent window's `PreviewKeyDown` handlers never fire.
- If the Popup is not given focus → parent window has keyboard focus but the
  Popup's ListBox does not receive key events.

**Working solution:** never focus the Popup. Make `DarkCombo` itself
`Focusable = true` and override `OnPreviewKeyDown`. When the dropdown is
open, intercept all navigation keys and manually drive `_list.SelectedIndex`.
The `_navigating` flag suppresses `SelectionChanged` auto-commit during
keyboard navigation:

```csharp
protected override void OnPreviewKeyDown(KeyEventArgs e)
{
    if (_popup.IsOpen)
    {
        int idx = _list.SelectedIndex < 0 ? 0 : _list.SelectedIndex;
        switch (e.Key)
        {
            case Key.Down:     Navigate(Math.Min(idx+1, max)); e.Handled=true; return;
            case Key.Up:       Navigate(Math.Max(idx-1, 0));   e.Handled=true; return;
            case Key.PageDown: Navigate(Math.Min(idx+8, max)); e.Handled=true; return;
            case Key.PageUp:   Navigate(Math.Max(idx-8, 0));   e.Handled=true; return;
            case Key.Return:   Commit(_list.SelectedIndex);    e.Handled=true; return;
            case Key.Escape:   _popup.IsOpen=false;            e.Handled=true; return;
        }
    }
}
```

### 13.2 `_popup.Opened` / `_popup.Closed` — ordering trap

Subscribe to `_popup.Opened` and `_popup.Closed` **after** `_popup` is
assigned. These lines dereference `_popup` immediately:

```csharp
// WRONG — _popup is null here, throws NullReferenceException
_popup.Opened += ...;
_popup = new Popup { ... };

// CORRECT
_popup = new Popup { ... };
_popup.Opened += ...;
_popup.Closed  += ...;
```

The lambdas in `MouseEnter` etc. that reference `_popup` through `this` are
fine — they execute later when `_popup` is set.

### 13.3 Per-item colour highlighting

`DarkCombo.AddItem(string s, bool highlighted = false)` colours items in
bright green (`#33FF77`) when `highlighted=true`. Used to mark tracks with
existing assignments so free slots are immediately visible. The `DarkCombo`
creates `ListBoxItem` objects directly (not strings) so per-item `Foreground`
can be set independently of the `ItemContainerStyle`.

---

## 14. Mapping curve (`EnvData` / `CurveCanvas`)

The piecewise-linear mapping curve transforms the 0–100% Value parameter into
the 0–100% parameter range sent to the target. Stored as sorted `List<EnvPoint>`
(X=input 0–65535, Y=output 0–65535).

Evaluation:

```csharp
float mappedY = Mapping.Evaluate(value01 * 65535f);
float f       = mappedY / 65535f;                    // 0=min, 1=max
int   pval    = (int)(f * (max - min) + min);
```

Note: the original C++ used `f = 1 - mappedY/65535` (inverted). The port
uses the non-inverted form with `DefaultPoints` `(0,0)→(65535,65535)` for an
intuitive "low in → low out" default.

`CurveCanvas` is a `FrameworkElement` subclass with custom `OnRender`,
mouse-drag point editing, and a right-click context menu (Mirror, Invert,
Reset to linear).

---

## 15. Inertia / glide

Mirrors the original `CTrack::UpdateInertia()`. When `Inertia > 0`, each
`ValueChange` sets a step size and marks the track as `Sliding`. Each
`Work()` call advances `ValueCurrent` toward `ValueTarget`. The `SendFreq`
attribute controls how many `Work()` calls elapse between pushes.

`SlidingFromMidi` tracks whether the slide was initiated by MIDI input so
feedback is suppressed during glide from MIDI (mirrors the `fromMidi` flag).

---

## 16. Roster entry

| Repo | Type | Last pushed | ReBuzz vs | Description |
|------|------|-------------|-----------|-------------|
| BTDSys-PeerCtrl-ReBuzz | control | 2026-05-22 | 1817-preview | Port of BTDSys PeerCtrl |

Depends on: Build §1 (csproj), Build §1.3 (deploy target).
