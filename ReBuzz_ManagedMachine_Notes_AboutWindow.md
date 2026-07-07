# ReBuzz Managed Machine Notes — Right-click "About" pattern

Source: Pedal Profiler2 v1.9.4 build on ReBuzz 1834-preview; first
end-to-end confirmation of the pattern.

Every managed machine can add items to its right-click context menu in
ReBuzz's Machine View by exposing an `IEnumerable<IMenuItem> Commands`
property. This doc captures the minimal paste-in idiom for an "About..."
window on any machine, together with a source-code trace showing why it
works and a diagnostic checklist for when it doesn't.

The doc is deliberately portable — copy the code blocks straight into a
new machine, change the four placeholders, done.

---

## 1. The four-piece paste

### 1.1 csproj — add `BuzzGUI.Common` reference

`MenuItemVM` and `SimpleCommand` live in `BuzzGUI.Common.dll`, which is
not referenced by default in a bare-bones managed-machine project. Add
alongside the existing ReBuzz / BuzzGUI.Interfaces references:

```xml
<ItemGroup>
  <Reference Include="ReBuzz">
    <HintPath>C:\Program Files\ReBuzz\ReBuzz.dll</HintPath>
    <Private>false</Private>
  </Reference>
  <Reference Include="BuzzGUI.Interfaces">
    <HintPath>C:\Program Files\ReBuzz\BuzzGUI.Interfaces.dll</HintPath>
    <Private>false</Private>
  </Reference>
  <Reference Include="BuzzGUI.Common">                                     <!-- add this block -->
    <HintPath>C:\Program Files\ReBuzz\BuzzGUI.Common.dll</HintPath>
    <Private>false</Private>
  </Reference>
</ItemGroup>
```

`Private=false` prevents the DLL being copied into the machine's build
output — it ships with ReBuzz itself and is loaded from
`C:\Program Files\ReBuzz\` at runtime.

The standing csproj rules still apply — every managed machine csproj
must have:

```xml
<DebugType>none</DebugType>
<DebugSymbols>false</DebugSymbols>
<GenerateDependencyFile>false</GenerateDependencyFile>
<TargetFramework>net10.0-windows</TargetFramework>   <!-- or higher -->
```

and a `DeployToReBuzz` target copying the .dll to the relevant folder
under `C:\Program Files\ReBuzz\`.

### 1.2 Usings — add three

At the top of the machine's `.cs` file:

```csharp
using System.Collections.Generic;   // IEnumerable<T>
using System.Windows;               // MessageBox
using BuzzGUI.Common;               // MenuItemVM, SimpleCommand
```

`BuzzGUI.Interfaces` (for `IMenuItem`) is almost certainly already in
the usings — the machine class implements `IBuzzMachine` from there.

### 1.3 Version const — single source of truth

At the top of the machine class, add a version constant:

```csharp
public class <MachineName>Machine : IBuzzMachine
{
    internal const string Version = "1.0.0";
    ...
}
```

Reference this const from the About dialog AND anywhere else the version
appears (dump headers, log lines, telemetry). Bumping the version becomes
a one-line change instead of a search-and-replace across the codebase.

### 1.4 Commands property — the menu entry

Add inside the machine class, near the top:

```csharp
public IEnumerable<IMenuItem> Commands
{
    get
    {
        yield return new MenuItemVM()
        {
            Text = "About...",
            Command = new SimpleCommand()
            {
                CanExecuteDelegate = p => true,
                ExecuteDelegate    = p => MessageBox.Show(
                    $"<Machine Display Name>   v{Version}\n\n" +
                    "<one-line description of what the machine does>\n\n" +
                    "<url>\n" +
                    "<License>",
                    "About <Machine Display Name>")
            }
        };
    }
}
```

Placeholders to change per machine:

| Placeholder | Example |
|---|---|
| `<MachineName>Machine` (class name) | `Profiler2Machine`, `FazeRMachine` |
| `<Version>` (const value) | `"1.0.0"` |
| `<Machine Display Name>` | `Pedal Profiler2`, `Pedal FazeR` |
| description | `Per-machine CPU inspector for ReBuzz.` |
| URL | `github.com/thepedal/<repo>` |
| License | `MIT License`, etc. |

That's everything. Rebuild, restart ReBuzz, right-click the machine in
Machine View — an "About..." entry appears at the bottom of the menu,
below the standard entries.

---

## 2. Why this works — the source-code trace

For debugging when the entry doesn't appear, it helps to know the
plumbing. ReBuzz builds the Machine View right-click menu at
`MachineControl.cs:465` (approx., trunk as of 1834):

```csharp
var cmds = machine.Commands;
if (cmds != null && cmds.Count() > 0)
{
    l.Add(new MenuItemVM() { IsSeparator = true });
    l.AddRange(cmds);
}
```

`machine` here is `IMachine` (i.e. `MachineCore`). Following the chain:

1. **`MachineCore.Commands`** getter (`MachineCore.cs:200`) calls
   `SetCommands()`.
2. **`SetCommands`** (`MachineCore.cs:1154`) calls
   `buzz.MachineManager.GetCommands(this)`.
3. **`MachineManager.GetCommands`** (`MachineManager.cs:1118`) branches
   on managed vs native:
   ```csharp
   if (machine.DLL.IsManaged && managedMachines.ContainsKey(machine))
       return managedMachineHost.Commands;
   ```
4. **`ManagedMachineHost.Commands`** (`ManagedMachineHost.cs:263`) does
   the reflection:
   ```csharp
   PropertyInfo property = dll.machineType.GetProperty("Commands");
   MethodInfo   getMethod = property.GetGetMethod();
   object       obj       = getMethod.Invoke(machine, null);
   return obj as IEnumerable<IMenuItem>;
   ```
5. **`ManagedMachineDll.machineType`** (`ManagedMachineDll.cs:47`) is
   found by:
   ```csharp
   machineType = Assembly.GetTypes()
       .FirstOrDefault(t => t.GetInterface("IBuzzMachine") != null);
   ```

So ReBuzz looks up **the first class in the assembly implementing
`IBuzzMachine`**, calls `GetProperty("Commands")` on it, invokes the
public getter, casts to `IEnumerable<IMenuItem>`, and appends the
entries to the menu after a separator.

Implications:

- The **property name must be exactly `Commands`** (case-sensitive).
- The **getter must be public**.
- The **return type must be assignable to `IEnumerable<IMenuItem>`** —
  `yield return` builds a compiler-generated enumerator that satisfies
  this.
- The pattern is duck-typed, not enforced by `IBuzzMachine`. The
  `IBuzzMachine` interface comment says `// {U} public IEnumerable<IMenuItem>
  Commands { get; }` — the `{U}` means "user-optional".
- If the class has no `Commands` property, `ManagedMachineHost.Commands`
  returns `null` and the menu shows only the standard entries.

---

## 3. If it doesn't appear — diagnostic checklist

If everything above is in place and the "About..." entry still doesn't
show in the right-click menu, work through these in order:

### 3.1 The deployed DLL is stale

Managed DLLs are loaded from `C:\Program Files\ReBuzz\Gear\<subdir>\`
into the ReBuzz process. If ReBuzz is running while you build, Windows
holds the DLL file locked and the `DeployToReBuzz` `Copy` step fires
silently with `ContinueOnError="true"` — the build succeeds but the file
on disk stays at the old version.

Check the timestamp:

```
C:\Program Files\ReBuzz\Gear\<subdir>\<MachineName>.NET.dll
```

If it's not fresh, close ReBuzz, rebuild, verify the timestamp updated,
reopen ReBuzz.

### 3.2 ReBuzz wasn't restarted after the new DLL was deployed

Managed machine assemblies are JIT-loaded once per ReBuzz session and
stay in memory. Even if the file on disk is refreshed mid-session, the
running `MachineDLL.Assembly` is the old one. Song reloads don't help.
**Close ReBuzz completely and reopen.**

### 3.3 The instance was created before the new DLL loaded

Delete any existing instances of the machine in the current song, then
add a fresh one and right-click that. Rules out any caching in the
Machine View around the old instance.

### 3.4 Prove the getter is being reached

If steps 3.1–3.3 don't fix it, instrument the getter and check the
Debug Console (Ctrl+D in ReBuzz):

```csharp
public IEnumerable<IMenuItem> Commands
{
    get
    {
        try { _host?.Machine?.Graph?.Buzz?.DCWriteLine("[<MachineName>] Commands getter invoked"); } catch { }
        yield return new MenuItemVM() { ... };
    }
}
```

Right-click the machine. If the line appears in the Debug Console, the
plumbing is running and something downstream (menu rendering, item
filter) is off. If it doesn't appear, either the deployed DLL is still
stale, or the reflection is failing to find the property (check the
property name spelling, capitalisation, and that the class is public).

---

## 4. Notes

### 4.1 Multiple menu items

`Commands` returns `IEnumerable<IMenuItem>`, so `yield return` more items
to add more entries. Each shows up in the machine's right-click menu
below the separator, in yield order.

```csharp
public IEnumerable<IMenuItem> Commands
{
    get
    {
        yield return new MenuItemVM() { Text = "Reset counters", Command = ... };
        yield return new MenuItemVM() { Text = "Export state...", Command = ... };
        yield return new MenuItemVM() { Text = "About...",       Command = ... };
    }
}
```

### 4.2 Submenus

`MenuItemVM.Children` is an `IEnumerable<IMenuItem>` — set it to a list
of child `MenuItemVM`s to build a submenu. See `MachineControl.cs` for
worked examples of the `Groups` submenu.

### 4.3 Why `MessageBox` and not a themed WPF window

For an About dialog, `MessageBox.Show(...)` is one line, has no
dependencies beyond `System.Windows`, and gives users the OS-native "OK"
button they expect. For machines with a rich inspector aesthetic
(dark theme, custom fonts) a themed WPF `Window` may fit better — but
that adds ~80 LOC and stops being copy-paste-portable. Use MessageBox
for consistency across the machine library; reserve custom windows for
the machine's actual inspector UI.

### 4.4 Separator handling

The context menu's own separator is inserted automatically before your
entries (see the source trace, step 0). You don't need to prepend
`new MenuItemVM() { IsSeparator = true }` yourself — but you can add
separators *between* your own entries with the same idiom if there's
more than one and they're logically distinct groups.

### 4.5 Interaction with other reflection surfaces

The same `dll.machineType.GetProperty(name)` pattern in
`ManagedMachineHost` is used for `MachineState` (save/load) and other
optional properties. Any user-optional convention documented as `{U}` in
`IBuzzMachine.cs` follows the same rules: exact name, public getter, and
returning the expected type. See `ManagedMachineHost.cs:321/353` for the
`MachineState` case.
