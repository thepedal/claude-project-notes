# Pedal Patcher — Machine Notes

Source: Pedal Patcher v1.1 (`pedal-ReBuzz-patcher-main`).
First managed machine in this project to use `EffectBlockMulti` (ReBuzz
≥ 1813-preview). See Build §7–8 for the general API rules; this file
covers machine-specific implementation details.

Sections use local 1–N numbering. References to `Build §N` and `Core §N`
point to the respective notes files.

---

## 1. Identity

| Property | Value |
|---|---|
| `MachineDecl.Name` | `"Pedal Patcher"` |
| `MachineDecl.ShortName` | `"Pedal Patch"` |
| `AssemblyName` | `Pedal Patch.NET` → `Pedal Patch.NET.dll` |
| Namespace | `PedalPatch` |
| Deploy target | `Gear\Effects\` |
| Machine type | Effect (EffectBlockMulti, MULTI_IO) |
| `NumInputs` / `NumOutputs` | 6 / 6 |
| `NumPatches` | 48 |
| State version | 2 |
| Repository | `https://github.com/thepedal/pedal-ReBuzz-patcher` (private) |

---

## 2. What the machine does

Pedal Patcher is a 6×6 audio patchbay. Each of the 6 input channels can
be independently routed to any combination of the 6 output channels, with
per-connection gain ramping to avoid clicks. 48 complete routing
configurations (patches) can be stored and switched between, either from
the GUI dropdown, via the `Patch` global parameter, or from the pattern
editor using track commands.

In the ReBuzz machine graph, each connection to an input or output pin has
a numbered circle selector (0–5). The number chosen there maps directly to
the row (for inputs) or column (for outputs) in the routing matrix in the
GUI.

---

## 3. Audio architecture

### 3.1 Work function type

Uses `EffectBlockMulti` (Build §7.1):

```csharp
public bool Work(IList<Sample[]> output, IList<Sample[]> input, int n, WorkModes mode)
```

`input[i]` is `null` when nothing is connected to input channel `i`.
`output[o]` is `null` when nothing is connected to output channel `o`.

### 3.2 Routing loop

Per-tick order:

1. Execute any pending track commands (before audio, so a pattern-triggered
   patch change takes effect within the same tick).
2. Zero all non-null output buffers.
3. For each `(i, o)` pair: if `input[i]` and `output[o]` are both non-null,
   mix `input[i]` into `output[o]` with the current gain for that pair.
4. Return `true` if any output buffer received audio; `false` otherwise.

### 3.3 Click-free gain ramping

Each `(i, o)` pair maintains an independent `gain[i,o]` (current) and
`targetGain[i,o]` (desired). When a cell is activated, `targetGain` is set
to `1f`; when deactivated, `0f`. The audio loop ramps `gain` toward
`targetGain` at a rate of `1 / (FadeTimeMs * SamplesPerSec / 1000)` per
sample. Default fade time is 10 ms. Maximum is 500 ms (settable in the GUI).

When both `gain` and `targetGain` are `0f`, the pair is skipped entirely
(no per-sample work, no VU update). The `VuLevel` for that pair decays
by ×0.85 per call instead.

### 3.4 VU metering

`VuLevel[i,o]` is an `internal readonly float[,]` (not `public` — a public
field on the machine class is mistaken for a parameter by ReBuzz's reflection
scanner, producing "at least one parameter is required" at load time).

Inside the audio loop, for each active `(i, o)` pair, `VuLevel[i,o]` is set
to `Max(peak_of_input_block, VuLevel[i,o] * 0.85f)` — fast attack, smooth
decay. The GUI polls this at 50 ms intervals via a `DispatcherTimer` and
colours the matrix cell orange when `VuLevel[i,o] > 0.001f` (~–60 dBFS).

---

## 4. Channel count and naming

### 4.1 Setting channel count

`host.InputChannelCount = NumInputs` and `host.OutputChannelCount = NumOutputs`
are set from a `Song.MachineAdded` handler, not the constructor (Build §8.1).
The constructor subscribes; `OnMachineAdded` fires when the machine's own
`IMachine` instance appears in the graph and sets the counts; `OnMachineRemoved`
unsubscribes both handlers.

### 4.2 Channel names

`GetChannelName(bool input, int index)` returns `inputLabels[index]` or
`outputLabels[index]` as appropriate. Default labels are `"In 1"`–`"In 6"`
and `"Out 1"`–`"Out 6"`. The user can rename them via double-click in the GUI;
custom names are saved in MachineState.

---

## 5. Parameters

### 5.1 Global parameter

| Name | Type | Range | Default | Notes |
|---|---|---|---|---|
| `Patch` | int (Word) | 0–47 | 0 | Active patch index. Stateful. Changing it calls `RefreshTargetGains()` which applies all the routing state of the new patch to the gain targets, triggering fades. |

`MaxValue = NumPatches - 1 = 47`. Keep it at 47 or below — 65535 is the
Word NoValue sentinel and is invalid as MaxValue (Core §9 / general rule).

### 5.2 Track parameters

Two per track. Both stateless (pattern-driven only, not stored in machine
state).

| Name | Type | Range | NoValue | Notes |
|---|---|---|---|---|
| `Command` | Byte | 0–8 | 255 | See §6 for command table |
| `Argument` | Word | 0–0xFFFF | 0xFFFF | High byte = input index, low byte = output index (both 0-based internally; the pattern description says 1-based for the user) |

`MaxTracks = 16`. `Tracks` is a `TrackState[]` array populated by ReBuzz;
its length reflects the current track count.

### 5.3 Settings (not parameters)

These are stored in MachineState but are not `[ParameterDecl]` — they have
no rack row or pattern column. They are plain C# properties wired to GUI
controls.

| Property | Type | Default | Range |
|---|---|---|---|
| `FadeTimeMs` | int | 10 | 0–500 |
| `ConfirmOnClear` | bool | true | — |
| `PreserveClipboard` | bool | false | — |

---

## 6. Track commands

Executed at the top of `Work()`, before audio processing. The `Argument`
word encodes `(input_index << 8) | output_index`, both 0-based.

| Command value | Name | Effect |
|---|---|---|
| 0 | Unplug | Disconnect `input[ii]` → `output[oo]` |
| 1 | Plug | Connect `input[ii]` → `output[oo]` |
| 2 | Plug Exclusive | Connect `input[ii]` → `output[oo]`, disconnect all other inputs from `output[oo]` first |
| 3 | Connect Input | Connect `input[ii]` to all outputs |
| 4 | Connect Output | Connect all inputs to `output[oo]` |
| 5 | Disconnect Input | Disconnect `input[ii]` from all outputs |
| 6 | Disconnect Output | Disconnect all inputs from `output[oo]` |
| 7 | Connect All | Connect every input to every output |
| 8 | Clear Patch | Disconnect everything in the current patch |

NoValue for Command is `255` (Byte type, MaxValue 8 — well clear of the
254 ceiling). NoValue for Argument is `0xFFFF` (Word type, MaxValue
`0xFFFF` — **this is the Word NoValue sentinel and is technically
invalid as MaxValue**; see §9 below).

---

## 7. Routing state

`bool[,,] routing` — dimensions `[NumPatches, NumInputs, NumOutputs]` =
`[48, 6, 6]` = 1728 cells. Each cell is independently toggleable.

`SetConnection(input, output, value)` writes to `routing[_currentPatch, ...]`
and simultaneously updates `targetGain[input, output]`. This means gain
ramps begin immediately on the next `Work()` call, with no separate
"apply" step needed.

`RefreshTargetGains()` rescans the full current patch and sets all
`targetGain[i,o]` values at once — used after patch switches and after
song load (`ImportFinished`).

---

## 8. Patch operations

Four operations act on the current patch only; one acts on all patches.
All live on `CMachine` and are called directly from the GUI (no parameter
plumbing needed since they're not automatable).

| Method | Description |
|---|---|
| `CopyCurrentPatch()` | Snapshot current patch routing into `clipboard[,]`. Sets `HasClipboard`. |
| `PasteCurrentPatch(merge: false)` | Replace current patch with clipboard. Clears clipboard unless `PreserveClipboard`. |
| `PasteCurrentPatch(merge: true)` | OR current patch with clipboard (union). |
| `ClearCurrentPatch()` | Zero all cells in current patch. Prompts if `ConfirmOnClear`. |
| `ClearAllPatches()` | Zero all 48 patches. Always prompts (no ConfirmOnClear check — considered destructive enough to always warn). |

---

## 9. Serialisation

MachineState version 2. Binary format, written via `BinaryWriter` in fixed
field order:

```
byte   version          (= 2)
int32  FadeTimeMs
bool   ConfirmOnClear
string inputLabels[0..5]    (6 × BinaryWriter.Write(string))
string outputLabels[0..5]   (6 × BinaryWriter.Write(string))
bool   routing[p,i,o]       (48×6×6 = 1728 booleans, in p/i/o order)
```

Version 1 compatibility: if `version == 1` is read, one extra `bool`
(`_autoName`, now removed) is discarded before reading the labels. This
means v1 songs load cleanly.

`MachineState.set` stores the data as `pendingLoad`; actual restoration
happens in `ImportFinished()` after the song graph is fully assembled. This
is the standard pattern for any machine that needs to query the graph at
restore time (though Pedal Patcher doesn't currently need to — it just
refreshes gains and fires property-change notifications).

### 9.1 Known issue — Argument MaxValue

`Argument` is declared `MaxValue = 0xFFFF` (65535). The Word NoValue
sentinel is also 65535, making `0xFFFF` technically an invalid MaxValue
(it's the ceiling, not a valid user value — Core general rule, see Core §9).
In practice ReBuzz does not throw on this declaration for Pedal Patcher,
but it may do so in a future version. The safe fix is `MaxValue = 0xFFFE`
(65534). Changing it is a non-breaking parameter-compatibility change
since it only tightens the valid range by one value that was never
meaningfully used.

---

## 10. GUI

### 10.1 Structure

Single `UserControl` (`GUI.xaml` / `GUI.xaml.cs`). All WPF elements are
created in code-behind at `BuildMatrix()` time rather than XAML (the
matrix size is a compile-time constant but building it in code is
simpler). The XAML provides the outer chrome (toolbar row, matrix
scroll area, footer row).

### 10.2 Cell states

Each matrix button has three visual states driven by `RefreshMatrix()`
(on routing change) and `RefreshVu()` (on timer tick):

| State | Background | Border |
|---|---|---|
| Not routed | `#333333` (dark grey) | `#505050` |
| Routed, no signal | `#1572E8` (blue) | `#44AAFF` |
| Routed + signal | `#E88200` (orange) | `#FFAA22` |

Signal threshold: `VuLevel[i,o] > 0.001f` (~–60 dBFS).

### 10.3 Mouse interaction

Left-click toggles the clicked cell and enters drag mode, painting all
subsequently entered cells to the same value as the initial toggle. Right-
click clears the clicked cell and enters drag mode that clears on enter.
Drag terminates on any `PreviewMouseUp` event on the `UserControl`.

### 10.4 Property change relay

`OnMachinePropertyChanged` catches `INotifyPropertyChanged` events from
`CMachine` and dispatches to the UI thread via `Dispatcher.BeginInvoke`.
Handled properties: `Routing`, `CurrentPatch`, `HasClipboard`,
`PreserveClipboard`, `InputLabels`, `OutputLabels`, `FadeTimeMs`.

### 10.5 Label renaming

Double-clicking any row or column label opens `PromptDialog.Ask()` — a
lightweight `Window` with a `TextBox` and OK/Cancel buttons, built
entirely in code (no XAML). The result is passed to
`machine.SetInputLabel()` / `machine.SetOutputLabel()`, which fires
`Notify(nameof(InputLabels))` / `Notify(nameof(OutputLabels))` and
also updates `GetChannelName()` for ReBuzz connection labels.

---

## 11. csproj notes

- `DebugType=none` / `DebugSymbols=false` / `GenerateDependencyFile=false`
  are in the Release config only. Debug config keeps full symbols for local
  development. This is a deliberate divergence from the Build §1.2 "both
  configs" default — acceptable for a machine that's actively developed,
  since debug symbols stay in `Debug\` and never reach the gear folder.
- `UseWindowsForms=true` is present but not required by this machine (no
  WinForms usage). Harmless; left in from early scaffolding.
- `AllowUnsafeBlocks=true` is present but not used. Same reason.
- Deploy target: `$(BuzzDir)\Gear\Effects\`, defaulting to
  `C:\Program Files\ReBuzz`. Override with
  `dotnet build /p:BuzzDir="D:\MyReBuzz"`.
