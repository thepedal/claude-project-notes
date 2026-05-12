# ReBuzz Managed Machine Development — Preset API Addendum

Source: ReBuzz 1821-preview source code, with the preset surface verified
across `BuzzGUI.Interfaces`, `BuzzGUI.Common`, and `BuzzGUI.ParameterWindow`.

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`; references to `Build §N` point to
`ReBuzz_ManagedMachine_Notes_Build.md`. Internal cross-references use
plain `§N` and `§N.M` and stay within this file.

This file covers the **consumer** side of the preset API — how a managed
machine reads another machine's preset bank and applies a preset to it.
The producer side (writing a `.prs.xml` bundle to ship with your own
machine) is covered in `Build §3`.

---

## 1. Where the API actually lives

The public preset surface is in two places, and only one of them works.

### 1.1 The dead one — `IMachineDLL.Presets`

`BuzzGUI.Interfaces/IMachineDLL.cs` declares:

```csharp
ReadOnlyCollection<string> Presets { get; }
```

This property is **vestigial in ReBuzz**. The implementation in
`ReBuzz/Core/MachineDLL.cs:33` is an auto-property with a `set;` accessor,
and a search across the entire ReBuzz codebase finds no callsite that
ever assigns it. The test suite asserts the property is null on a fresh
machine (`ReBuzzTests/Automation/Assertions/InitialStateAssertions.cs:350`):

```csharp
machine.DLL.Presets.Should().BeNull();
```

Do not use `target.DLL.Presets` — it's null for every machine ReBuzz
loads, regardless of whether the machine ships a preset bank.

### 1.2 The real one — `MachineExtensions` in `BuzzGUI.Common`

The functional API is a set of extension methods on `IMachine` in
`BuzzGUI.Common.InterfaceExtensions.MachineExtensions`:

```csharp
public static IEnumerable<string>   GetPresetNames(this IMachine machine);
public static Preset                GetPreset(this IMachine machine, string name);
public static PresetDictionary      GetPresetDictionary(this IMachine machine);
public static void                  SetPreset(this IMachine machine, string name, Preset preset);
public static void                  ImportPresets(this IMachine machine, string filename);
```

Plus the public `Preset` and `PresetDictionary` classes themselves (in
`BuzzGUI.Common.Presets`) — `Preset` has a public `Apply(IMachine, bool
setdata)` method that performs the entire load.

Three calls cover the read-and-apply workflow:

```csharp
using BuzzGUI.Common.InterfaceExtensions;          // GetPreset, GetPresetNames
using BuzzGUI.Common.Presets;                      // Preset

var names  = target.GetPresetNames().ToList();     // 1. enumerate
var preset = target.GetPreset(names[idx]);         // 2. fetch
preset.Apply(target, setdata: true);               // 3. apply
```

This is exactly the path ReBuzz's own preset-selection UI takes — see
`SelectPresetAction.Do()` in `BuzzGUI.ParameterWindow/Actions/SelectPresetAction.cs:21`:

```csharp
public void Do()
{
    oldpreset = new Preset(vm.Machine, false, true);
    preset.Apply(vm.Machine, true);
}
```

Identical behaviour to "user right-clicked the machine and chose preset
N", because it's the same code.

---

## 2. Caching and disk I/O

`GetPresetDictionary` is lazy, file-backed, and process-cached. The
implementation (MachineExtensions.cs:83) is roughly:

```csharp
static readonly Dictionary<string, PresetDictionary> presetDictionaries = new();

public static PresetDictionary GetPresetDictionary(this IMachine machine)
{
    if (presetDictionaries.ContainsKey(machine.DLL.Name))
        return presetDictionaries[machine.DLL.Name];

    var xmlpath = Path.Combine(GetDirectory(machine), machine.DLL.Name + ".prs.xml");
    if (File.Exists(xmlpath))
        pd = PresetDictionary.Load(machine, xmlpath);

    var prspath = Path.Combine(GetDirectory(machine), machine.DLL.Name + ".prs");
    if (File.Exists(prspath))
        pd.Merge(PresetDictionary.Load(machine, prspath));  // legacy binary format

    presetDictionaries[machine.DLL.Name] = pd;
    return pd;
}
```

Two facts to internalise:

- **Disk I/O happens exactly once per machine type per ReBuzz session.**
  First call to `GetPresetNames` (or `GetPreset`) on any instance of a
  given machine reads `.prs.xml` from `<install>/Gear/Generators/` or
  `<install>/Gear/Effects/`. Every subsequent call — across every
  instance of the same machine type, across every consumer in the
  process — is a hash-map lookup.
- **The cache is shared with ReBuzz's own UI.** The `presetDictionaries`
  field is a `static readonly Dictionary` in `BuzzGUI.Common.dll`, which
  is loaded into the same AppDomain that hosts managed machines. If the
  user has ever opened the parameter window on the target machine (or
  right-clicked it to see its preset menu), ReBuzz has already populated
  the cache and your call costs nothing.

The `GetDirectory` helper at MachineExtensions.cs:71 computes the path
from `Path.GetDirectoryName(Assembly.GetExecutingAssembly().Location) +
"Gear" + (Generators|Effects)`, dispatched on `machine.DLL.Info.Type`.
You don't need to know the install path — the extension method handles
all of it.

Practical implication for any preset-consuming control machine: just
call the extension methods. Don't write a parallel preset bank loader,
don't hard-code install paths, don't walk the file system. The work is
already done in the host.

---

## 3. `Preset.Apply()` behaviour

`Preset.Apply(IMachine machine, bool setdata)` in
`BuzzGUI.Common/Presets/Preset.cs:308` does three things in sequence:

### 3.1 Parameter restore — two paths

**Fast path** (when both the preset and the target have parameter names
unique within their groups):

```csharp
foreach (var p in machine.AllNonInputStateParametersAndTracks())
    p.Item1.SetValue(p.Item2, GetValueOrDefault(p.Item1, p.Item2));
```

Each parameter on the target is looked up by name in the preset; if
found, the stored value is written; if not, the parameter's `DefValue`
is written. This is the "modern" path and the one almost every managed
machine takes.

**Fallback path** (used when either side has duplicate parameter names):

```csharp
// Iterate preset entries in order, tracking group/track transitions,
// advancing a per-group parameter pointer, and verifying each step by
// name match before writing.
```

This mirrors the parameter-index-stability rule from `Build §3.3`. Both
paths route through `IParameter.SetValue(track, value)`, so the
"two parallel storage locations" trap from `PedalTracker §13.1` — where
a direct property write updates the field but not `values[]` — is
avoided automatically. We don't have to think about it.

### 3.2 Attribute restore

```csharp
foreach (var ma in machine.Attributes)
{
    var pa = Attributes?.Where(x => x.Name == ma.Name).SingleOrDefault();
    var val = pa != null ? pa.Value : ma.DefValue;
    if (ma.HasUserDefValue && ma.UserDefValueOverridesPreset) val = ma.UserDefValue;
    if (val != ma.Value) ma.Value = val;
}
```

Each target attribute is set from the preset's attribute value, or its
default if the preset doesn't mention it. The
`UserDefValueOverridesPreset` flag (per-attribute, set by the target
machine) correctly suppresses preset writes for user-locked attributes.

### 3.3 Data restore — only for `LOAD_DATA_RUNTIME` machines

```csharp
if (setdata && (machine.DLL.Info.Flags & MachineInfoFlags.LOAD_DATA_RUNTIME) == MachineInfoFlags.LOAD_DATA_RUNTIME)
    machine.Data = Data;
```

Machines that declare `LOAD_DATA_RUNTIME` (typically samplers and
wavetable engines that hold large state outside parameters) get their
blob restored too. The `setdata` argument lets the caller skip this
selectively — useful if you want to apply preset *parameters* without
clobbering loaded samples, but for the standard "load preset N" path
you pass `true` and let it happen.

---

## 4. Out-of-range names produce a defaults preset

`GetPreset` for an unknown name returns a default-values preset, not
null (MachineExtensions.cs:128):

```csharp
public static Preset GetPreset(this IMachine machine, string name)
{
    var pd = GetPresetDictionary(machine);
    if (name == "<default>" || !pd.ContainsKey(name))
        return new Preset(machine, defvalue: true, includedata: true);
    return pd[name];
}
```

The constructor `Preset(IMachine, true, true)` walks the target's
parameters and attributes and snapshots their `DefValue`s. Apply-ing
that preset resets the machine to its declared defaults.

Useful side effect for pattern-driven preset selection: an
out-of-range integer in the pattern row doesn't need explicit bounds
checking. A row value beyond the bank size produces a coherent
reset-to-defaults event, which is a perfectly reasonable interpretation
of "this preset index doesn't exist." If you want strict bounds (no-op
instead of reset), check `idx < names.Count` before calling
`GetPreset(names[idx])`.

---

## 5. Thread safety — must apply on the UI thread

`Preset.Apply()` is not safe to call from the audio thread. Three
things in it require UI-thread dispatch:

- **`machine.Data = Data` (line 358)** writes to `IMachine.Data`, which
  for native targets goes cross-process via `CMachineInterface::Save`
  and `CMachineInterfaceEx::Load` (per the comment on `IMachine.cs:53`).
  This has the same UI-thread requirement as `IMachine.IsMuted`
  (`PedalMuter §1.1`).
- **`MessageBox.Show` on the exception paths (lines 138, 362)** —
  catches around the `Data` getter and setter call `MessageBox.Show`
  from whatever thread `Apply` is on. Audio-thread message boxes are
  visually wrong at best and a deadlock vector at worst.
- **Attribute writes** (`ma.Value = val`) — `IAttribute.Value` is
  treated like other `IMachine`-property writes; safer on UI thread.

The dispatch pattern is identical to `PedalMuter §1.1` — wrap the apply
in a UI-thread closure:

```csharp
static void ApplyPresetUiSafe(IMachine target, int idx)
{
    if (target == null || target.DLL == null) return;

    var d = System.Windows.Application.Current?.Dispatcher;
    if (d == null || d.CheckAccess())
    {
        ApplyPresetCore(target, idx);
    }
    else
    {
        d.BeginInvoke(new Action(() => ApplyPresetCore(target, idx)));
    }
}

static void ApplyPresetCore(IMachine target, int idx)
{
    try
    {
        var names = target.GetPresetNames().ToList();
        if (idx < 0 || idx >= names.Count) return;   // strict bounds; omit for defaults-on-OOR
        var preset = target.GetPreset(names[idx]);
        if (preset == null) return;
        preset.Apply(target, setdata: true);
    }
    catch { /* Apply or GetPreset can throw on corrupt banks */ }
}
```

Capture only the integer index on the audio thread; do the
`GetPresetNames` / `GetPreset` lookup inside the UI closure. The
extension methods are themselves cache-friendly enough to call from UI
thread without concern (the first-touch case does file I/O but is rare
and unavoidable wherever it runs).

### 5.1 Don't preset yourself

A controller that drives presets on foreign machines can preset itself
if the assignment list includes the host machine. This corrupts the
controller's own parameter state mid-pattern and likely leaves it
unable to continue. Same defensive check as `PedalMuter §1.3`:

```csharp
if (ReferenceEquals(target, host.Machine)) return;
```

---

## 6. csproj — referencing `BuzzGUI.Common` alongside `BuzzGUI.Interfaces`

The preset extension methods live in `BuzzGUI.Common.dll`, not
`BuzzGUI.Interfaces.dll`. A preset-consuming control machine therefore
needs both references. Standard pattern for ReBuzz's typical install
path:

```xml
<ItemGroup>
  <Reference Include="BuzzGUI.Interfaces">
    <HintPath>C:\Program Files\ReBuzz\BuzzGUI.Interfaces.dll</HintPath>
    <Private>false</Private>
  </Reference>
  <Reference Include="BuzzGUI.Common">
    <HintPath>C:\Program Files\ReBuzz\BuzzGUI.Common.dll</HintPath>
    <Private>false</Private>
  </Reference>
</ItemGroup>
```

`<Private>false</Private>` is essential on both — ReBuzz already loads
these DLLs into its AppDomain. If your machine ships its own copy in
the gear folder, the .NET loader resolves your copy first and you end
up with two distinct `BuzzGUI.Common` assemblies in memory, neither
sharing the static `presetDictionaries` cache with the other. The
right-click menu and your machine then read from different copies of
the dictionary, which is a silent correctness bug.

`BuzzGUI.Common` itself targets `net10.0-windows10.0.26100.0` with
`<UseWPF>true</UseWPF>` and `<UseWindowsForms>true</UseWindowsForms>`
(per its own csproj). Referencing it pulls those into the build graph
but does not load WinForms unless you actually touch a WinForms type;
WPF is already in scope per `Build §1.2`.

---

## 7. What this changes for a preset-consuming machine

Things you do **not** need to write:

- A `.prs.xml` parser. `PresetDictionary.Load` handles it, including
  the legacy binary `.prs` format if both exist.
- Install-path discovery. `GetDirectory` resolves `Gear/Generators` vs
  `Gear/Effects` from `machine.DLL.Info.Type`.
- An in-memory preset cache. `presetDictionaries` is process-static and
  shared with the UI.
- The unique-name-vs-index parameter matching logic. `Preset.Apply`
  handles both paths transparently.
- Attribute restoration with the `UserDefValueOverridesPreset` rule.
  `Preset.Apply` already respects it.
- `IMachine.Data` restoration for `LOAD_DATA_RUNTIME` targets.
  `Preset.Apply(machine, setdata: true)` does it.

Things you **do** need to write:

- The pattern-editor surface — how the user indicates "fire preset N on
  track T" (typically a per-track `Preset` byte parameter, `IsStateless
  = true` so each row is an event not a held value).
- The target assignment list — which foreign machine each track drives.
  Persist via `MachineState` (per `IBuzzMachine.cs:28`) and re-resolve
  by name on load.
- The UI-thread dispatch wrapper (§5).
- Defensive checks: target deleted from song, target is self, preset
  index out of range (or accept the defaults-preset behaviour from §4),
  external writes exceeding `MaxTracks` (per `PedalInvFFT §24` if your
  control machine has tracks).
- The `parametersChanged`-collision polling fallback from `Core §14`
  if you need to fire presets on multiple tracks within a single tick.

Everything else is provided by the host.
