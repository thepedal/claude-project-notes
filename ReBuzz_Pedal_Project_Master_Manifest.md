# ReBuzz Pedal Project — Master Manifest

**Repo:** `thepedal/claude-project-notes` — the canonical, versioned home of all project
knowledge. The Claude Project is a disposable cache of this repo; when they disagree,
this repo wins. If you are a fresh Claude instance (or a future human) bootstrapping
from scratch: **read this file first**, then follow the reading order in §3.

Last manifest update: 2026-07-10.

---

## 1. What this project is

Development of **managed (.NET) machines for ReBuzz** — an open-source rebuild of the
Jeskola Buzz modular music studio — plus two sibling efforts that grew out of it:

1. **Machine development** — 41+ machines (effects, generators, controllers, trackers,
   diagnostics) in the "Pedal" series, each in its own GitHub repo under
   `github.com/thepedal/`. The notes distil hard-won facts about the ReBuzz managed
   machine API, build/deploy conventions, and per-machine design decisions.
2. **Song-writer tooling** (`pedal-rebuzz-song-writer`) — a Python library that
   assembles loadable `.bmxml` ReBuzz songs programmatically from a byte-exact model
   of the format (spec → song).
3. **Host performance work** — profiling ReBuzz itself (via the Pedal Profiler
   machines), contributing measured, bit-exact optimisation PRs upstream to
   `wasteddesign/ReBuzz`, and recording findings as dated, immutable documents.

Everything here was built collaboratively with Claude across many sessions. The notes
files are the distilled ground truth of those sessions — conversations are ephemeral,
notes are not.

---

## 2. The three bodies of notes and their cross-reference conventions

The notes are one interlinked system, not independent documents. Cross-references use
a fixed shorthand — learn it before reading anything else:

| Shorthand | Resolves to | Covers |
|---|---|---|
| **Core §N** | `ReBuzz_ManagedMachine_Notes_Core.md` | ReBuzz managed-machine API facts: Work signatures, threading, host interop, engine behaviour |
| **Build §N** | `ReBuzz_ManagedMachine_Notes_Build.md` | csproj conventions, deploy targets, framework pinning, multi-IO/sidechain patterns |
| **BMXML §N** | `ReBuzz_SongFormat_Notes_BMXML.md` | The `.bmxml` song-file format, byte-level |
| **Roster** | `ReBuzz_ManagedMachine_Notes_Roster.md` | The machine inventory: repo, type, last push, ReBuzz version tested, addendum flag |
| **(per-machine) §N** | `ReBuzz_ManagedMachine_Notes_<Name>.md` | Section numbers in each addendum are local to that file |

The **Roster** is the impact-analysis index: when ReBuzz changes, grep the roster and
addenda for the relevant `Core §N` / `Build §N` references to answer "which machines
need touching?" It also carries a separate song-authoring section consumed by the
song-writer (note-column indices, track counts, direct vs control-driven).

---

## 3. File map and reading order

### Tier 1 — Orientation (read first)

| File | What it is |
|---|---|
| `ReBuzz_Pedal_Project_Master_Manifest.md` | This file |
| `ReBuzz_ManagedMachine_Notes_Roster.md` | Machine inventory + conventions + impact-analysis index |
| `ReBuzz_SongWriter_Notes_Overview.md` | Orientation layer for the song-writer sub-project |

### Tier 2 — Reference (consult constantly; authoritative)

| File | What it is |
|---|---|
| `ReBuzz_ManagedMachine_Notes_Core.md` | The managed-machine API bible |
| `ReBuzz_ManagedMachine_Notes_Build.md` | Build/deploy bible — csproj rules live here (§1.2–§1.4) |
| `ReBuzz_SongFormat_Notes_BMXML.md` | The `.bmxml` format bible |
| `ReBuzz_ManagedMachine_Notes_Presets.md` | Preset (`.prs.xml`) format and bundling |

### Tier 3 — Per-machine addenda (consult when touching that machine)

`ReBuzz_ManagedMachine_Notes_<MachineName>.md` — currently: AboutWindow, PedalChord,
PedalComp, PedalDrumGrid, PedalFazeR, PedalInvFFT, PedalM1, PedalMuter, PedalPresetter,
PedalProfiler2, PedalSH101, PedalTracker, PeerCtrl. The Roster's `Addendum` column is
the authoritative list — some machines have handoff docs living in their own repos
instead (e.g. `drum_matrix_HANDOFF.md` in `pedal-drummatrix`).

### Tier 4 — Dated findings (immutable records; never overwrite)

| File | What it records |
|---|---|
| `ReBuzz_Performance_Audit_1826.md` (+ `_by_MachineType`, `_for_Upstream`) | The May 2026 host-overhead audit on build 1826 |
| `ReBuzz_Perf_Sweep_Findings_2026-07-05.md` | The allocation/UpdateBuffer optimisation sweep — findings + PR sequence |
| `ReBuzz_Perf_AB_Result_2026-07-06.md` | Measured PRE/POST outcome of the sweep (~8% G0 + best-case render time) |
| `ReBuzz_RenderNondeterminism_Finding_2026-07-08.md` | Open engine-robustness item: `det_grid_08x04` does not render bit-identically; undermines byte-exact gating |

Dated findings are fixed records. Living status belongs in handoff docs (e.g.
`ReBuzz_Perf_Handoff_2026-07-04e.md`), which reference the findings rather than
absorbing them.

### Assets

| File | What it shows |
|---|---|
| `pr108_scaling.png` | Scaling chart attached to upstream PR #108 discussion (see the perf notes that reference it) |

---

## 4. Cardinal rules (non-negotiable)

1. **Never fabricate a `<Machine>` block in a `.bmxml`. Splice a real one** from an
   actual ReBuzz save. Hand-built blocks were the root cause of load-time NREs and
   "can't edit / can't add tracks" failures (BMXML §6, §11). This includes "just one
   more synth" — splice it.
2. **Mandatory csproj properties for every managed machine** (Build §1.2) — all six,
   in every machine csproj, added whenever a csproj is scaffolded, modified for any
   reason, or merely reviewed and found lacking:
   ```xml
   <PropertyGroup>
     <TargetFramework>net10.0-windows</TargetFramework>
     <UseWPF>true</UseWPF>
     <DebugType>none</DebugType>
     <DebugSymbols>false</DebugSymbols>
     <GenerateDependencyFile>false</GenerateDependencyFile>
     <NoWarn>MSB3277</NoWarn>
   </PropertyGroup>
   ```
   Rationale: ReBuzz loads only the `.dll`; `.pdb` and `.deps.json` are dead weight in
   `C:\Program Files\ReBuzz`. `net10.0-windows` matches ReBuzz's own DLLs (older
   targets hard-fail with CS1705). All machines target .NET v10 or higher.
3. **Mandatory post-build deploy target** (Build §1.3): every machine csproj copies
   `$(TargetPath)` to `C:\Program Files\ReBuzz\Gear\Generators\` or `...\Gear\Effects\`
   via a `DeployToReBuzz` target with `AfterTargets="Build"` and
   `ContinueOnError="true"` (ReBuzz holds DLL handles while running).
4. **Host perf work never touches machine csprojs** — host-engine scope only, all
   branches probe-free (instrumentation stays on `perf/instrument`), all changes
   bit-exact and render-gated. Host-bundled `Machines/*` and `FakeNative*` are
   permanently out of remit.
5. **Dated finding docs are immutable.** New results get new dated files.
6. **Trust the roster's confidence markers.** `ReBuzz vs = ?` means unverified against
   the current build; verify before acting on it.

---

## 5. External ground truth (what this repo does NOT contain)

- **Machine source code** — 41+ repos under `github.com/thepedal/` (see Roster for the
  full list). This repo is notes only.
- **The song-writer library** — `pedal-rebuzz-song-writer` repo.
- **ReBuzz itself** — upstream `wasteddesign/ReBuzz`. The notes span preview builds
  **1817 → 1834**; the July 2026 perf work is anchored to upstream commits
  (`c841786` pre-sweep, `e9837a7` post-#116, `355deba` main at the nondeterminism
  finding) and PRs #108–#123.
- **The Windows environment** — ReBuzz installed at `C:\Program Files\ReBuzz`
  (machines deploy into `Gear\Generators` / `Gear\Effects`), Visual Studio,
  Release/x64 builds, RME Fireface ASIO for measurement work.
- **Real ReBuzz save files** — the splice sources for `<Machine>` blocks. These live
  wherever the song-writer's machine reference is kept; a fresh environment needs at
  least one real save per machine type used in songs.

---

## 6. Re-initiating the Claude Project from scratch

1. Clone this repo.
2. Create a new Claude Project.
3. Paste the block below into the project's **custom instructions** (this is the
   canonical copy — keep it in sync with the project settings):

   > **csproj deployment rules (mandatory for every machine).** Every managed machine
   > `.csproj` in this project must include the following three properties in a
   > `<PropertyGroup>`:
   > ```xml
   > <DebugType>none</DebugType>
   > <DebugSymbols>false</DebugSymbols>
   > <GenerateDependencyFile>false</GenerateDependencyFile>
   > ```
   > This applies to: (a) any new machine you scaffold, (b) any existing machine you
   > modify where the `.csproj` is touched for any reason, and (c) any time you review
   > a `.csproj` and notice these are missing — flag it and add them. Rationale: ReBuzz
   > only needs the `.dll`; `.pdb` and `.deps.json` are unused at runtime and shouldn't
   > be deployed to `C:\Program Files\ReBuzz`.
   >
   > Every new machine should deploy its dll to the relevant dir in
   > `C:\Program Files\ReBuzz`.
   >
   > All machines written for .NET v10 or higher.

4. Upload **all** files from this repo into project knowledge (including this manifest
   and `pr108_scaling.png`).
5. Open the first conversation with: *"Read ReBuzz_Pedal_Project_Master_Manifest.md first, then the Roster."*

Note the custom instructions are the abridged three-property form; the full six-property
rule (with `TargetFramework`, `UseWPF`, `NoWarn`) is in Build §1.2 and is what actually
applies.

---

## 7. Maintenance discipline

- **New machine shipped** → update the Roster (new row, `ReBuzz vs` stamped with the
  build it was tested on); write an addendum if the machine warranted one; push both
  here and re-upload to the Claude Project.
- **ReBuzz build bump** → note it in the relevant Core/Build sections; grep Roster +
  addenda for impacted `§N` references.
- **New investigation concluded** → dated finding file, immutable; handoff doc updated
  to reference it.
- **Any session that surfaced a durable fact not yet written down** → it goes into the
  appropriate notes file before the session is considered closed. Conversations are
  not a storage medium.
- **Repo ↔ Claude Project sync** — after any notes change, re-upload the changed files
  to the project. Periodically diff the project's file list against `git ls-files`.

### Known sync gaps as of 2026-07-10 (verify and clear)

- In repo but **not** in the Claude Project: `ReBuzz_Performance_Audit_1826.md`,
  `ReBuzz_Performance_Audit_1826_by_MachineType.md`,
  `ReBuzz_Performance_Audit_1826_for_Upstream.md`.
- In the Claude Project but **not confirmed in the repo** (check `git ls-files`):
  `ReBuzz_Perf_Sweep_Findings_2026-07-05.md`, `pr108_scaling.png`.
- Referenced by the notes but in **neither**: `ReBuzz_Perf_Handoff_2026-07-04e.md`
  (the living perf handoff), `perf_instrument_render_benchmark.patch`, and the
  per-machine handoffs living in machine repos (e.g. `drum_matrix_HANDOFF.md`).
  Decide deliberately whether these belong here; if they stay external, this manifest's
  §5 is their pointer of record.
