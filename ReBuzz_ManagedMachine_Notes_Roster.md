# ReBuzz Managed Machine Roster

A directory of every public managed machine under
[github.com/thepedal](https://github.com/thepedal), one row per repo.

**Source of truth for the list:** the GitHub API
(`https://api.github.com/users/thepedal/repos?per_page=100`).
This file is a derived snapshot — refresh it when machines are added,
renamed, or archived.

**Last refreshed:** 2026-05-22 (35 repos).

The point of this file is impact analysis. When something changes in
ReBuzz, this file plus the per-machine addenda lets you answer "which
machines need touching?" by grepping for the relevant Core/Build
`§N` reference.

---

## Conventions

- **Repo** — GitHub repo name. Buzz machine browser display name comes
  from the `<AssemblyName>` in the `.csproj` (see Build §2), not the
  repo name. They're usually aligned: `pedal-comp` → "Pedal Comp.NET" →
  shown as "Pedal Comp".
- **Type** — broad classification only:
  - **effect** — audio in, audio out (`Work(Sample[] out, Sample[] in, ...)`)
  - **generator** — audio out, no audio in (synth)
  - **control** — no audio I/O, `void Work()` (Core §2)
  - **tracker** — pattern editor / sequencer surface
  - **diagnostic** — profiler / inspector
  - **utility / template** — scaffolding or build-time tooling
- **Last pushed** — most recent `git push` (UTC date, from GitHub API).
- **ReBuzz vs** — the ReBuzz preview build this machine was last
  built/tested against, if explicitly known from the Core/Build notes
  log. `?` means not tracked; assume current preview but verify before
  acting on it.
- **Addendum** — `Y` means a dedicated
  `ReBuzz_ManagedMachine_Notes_<Name>.md` exists in the project notes.

---

## Roster (35 machines)

| Repo                    | Type       | Last pushed | ReBuzz vs     | Addendum | Description                                                |
|-------------------------|------------|-------------|---------------|----------|------------------------------------------------------------|
| BTDSys-PeerCtrl-ReBuzz  | control    | 2026-04-04  | ?             |          | Port of BTDSys PeerCtrl                                    |
| pedal-chord             | control    | 2026-04-25  | 1817-preview  |          | Chord peer controller with arpeggiation                    |
| pedal-chorus            | effect     | 2026-05-02  | ?             |          | Chorus effect                                              |
| pedal-comp              | effect     | 2026-05-02  | ?             | Y        | Compressor effect                                          |
| pedal-converb           | effect     | 2026-05-07  | ?             |          | Convolution reverb (SIMD, wavefile IRs)                    |
| pedal-dly-PCM41         | effect     | 2026-05-05  | 1819-preview  |          | PCM41-style tape delay                                     |
| pedal-do-nuttin         | template   | 2026-05-12  | ?             |          | "Do nothing" machine — minimal scaffold                    |
| pedal-eq                | effect     | 2026-05-17  | 1819-preview  |          | EQ effect (Core §33 source: v1.3 WM_NOIO handling)         |
| pedal-fft               | effect     | 2026-05-02  | ?             |          | FFT distortion with harmonics                              |
| pedal-filter            | effect     | 2026-05-19  | ?             |          | Filter effect                                              |
| pedal-fm                | generator  | 2026-05-03  | ?             |          | FM synth                                                   |
| pedal-folder            | effect     | 2026-05-16  | ?             |          | Wavefolding distortion                                     |
| pedal-follower          | control    | 2026-05-05  | ?             |          | Envelope follower → param assignment                       |
| pedal-gain              | effect     | 2026-04-29  | ?             |          | Gain with mute and inertia                                 |
| pedal-gain-multi        | effect     | 2026-04-25  | ?             |          | Multi-in gain with VU metering                             |
| pedal-gate              | effect     | 2026-05-18  | 1819-preview  |          | Noise gate (Core §16.1 source: v1.3 sidechain build)       |
| pedal-hallverb          | effect     | 2026-05-02  | ?             |          | Hall reverb                                                |
| pedal-hdist             | effect     | 2026-05-16  | ?             |          | Harmonic distortion                                        |
| pedal-invFFT            | generator  | 2026-05-09  | ?             | Y        | Inverse-FFT additive synth (K5000-inspired)                |
| pedal-LFmono            | effect     | 2026-04-29  | ?             |          | Low-frequency mono-maker                                   |
| pedal-lfo               | control    | 2026-05-05  | ?             |          | LFO → param assignment                                     |
| pedal-limit             | effect     | 2026-04-17  | ?             |          | Limiter effect                                             |
| pedal-M1                | generator  | 2026-05-10  | ?             | Y        | 8-voice poly dual-PCM-osc synth, Korg M1 voice architecture|
| pedal-mcomp             | effect     | 2026-05-19  | 1819-preview  |          | Multi-band compressor (Core mentions v1.1 GUI build)       |
| pedal-muter             | control    | 2026-05-14  | ?             | Y        | Mute other machines                                        |
| pedal-plaits            | generator  | 2026-05-12  | 1819-preview  |          | Port of Mutable Instruments Plaits                         |
| pedal-plate             | effect     | 2026-05-01  | ?             |          | Plate reverb                                               |
| pedal-presetter         | control    | 2026-05-13  | ?             |          | Change presets on target machines                          |
| pedal-profiler          | diagnostic | 2026-04-26  | ?             |          | Global CPU dashboard — v1                                  |
| pedal-profiler2         | diagnostic | 2026-05-21  | 1826-preview  | Y        | Per-machine inspector — v2 (Core §§34–40 source)           |
| pedal-ReBuzz-patcher    | utility    | 2026-04-08  | ?             |          | Patching machine                                           |
| pedal-retrig            | effect     | 2026-05-16  | ?             |          | Port of 'genre' by intoxicated (retrigger)                 |
| pedal-shaper            | effect     | 2026-05-16  | ?             |          | Waveshaper distortion                                      |
| pedal-tracker           | tracker    | 2026-05-09  | ?             | Y        | Tracker machine — Matilde-compatible                       |
| pedal-zplane            | effect     | 2026-05-07  | ?             |          | Z-Plane filter (4-corner morph)                            |

ReBuzz versions in this table come from the "Updated with findings
from …" log at the top of `ReBuzz_ManagedMachine_Notes_Core.md`. Most
rows are `?` because the version wasn't stamped — that's a gap to
fill in over time, not evidence the machines are stale.

---

## Machines with detailed addenda

Six machines have their own notes file. To support impact analysis,
each should grow a `## Depends on` section listing the Core/Build
§-references it relies on. The lists below are seeds — flesh out from
each addendum as time allows.

### pedal-comp — `ReBuzz_ManagedMachine_Notes_PedalComp.md`
- The original debugging target that produced Core §§1–N. Effectively
  the reference implementation for "managed effect machine" patterns.
- Depends on: Core (foundational — most sections trace back here),
  Build §1.2 (csproj properties), §1.3 (post-build deploy), §2
  (AssemblyName).

### pedal-tracker — `ReBuzz_ManagedMachine_Notes_PedalTracker.md`
- Matilde-compatible tracker; cross-machine pattern manipulation.
- Depends on: Core §§1–2 (Work / control classification), §14
  (multi-track note workaround) likely relevant, Tracker addendum
  §§11–12 (programmatic pattern manipulation, multi-out per-track
  generators), MPE integration (Tracker addendum §11.1, §11.2).

### pedal-invFFT — `ReBuzz_ManagedMachine_Notes_PedalInvFFT.md`
- Additive synth via inverse FFT.
- Depends on: Core §§ generator/Work patterns, Note encoding (§7),
  presets (`Presets.md` for `.prs.xml` format). Fill in specifics
  from the addendum.

### pedal-M1 — `ReBuzz_ManagedMachine_Notes_PedalM1.md`
- Dual-PCM-osc poly synth.
- Depends on: Core §3 (Note parameter type), §7 (note encoding), §14
  (multi-track note workaround — critical for poly), §15 (lazy init
  of host.Machine), Presets format. Fill in specifics from the
  addendum.

### pedal-muter — `ReBuzz_ManagedMachine_Notes_PedalMuter.md`
- Control machine that mutes peers.
- Depends on: Core §2 (control classification via `void Work()`), §4
  (`SendControlChanges()` after writing to a target), §8 (control
  machines tick first).

### pedal-profiler2 — `ReBuzz_ManagedMachine_Notes_PedalProfiler2.md`
- Per-machine inspector; the May 2026 dropout investigation.
- Depends on: Core §§34–40 (audio-thread chunking,
  `MachinePerformanceData`, `EngineSettings`, `MasterTap`, Buzz sample
  scale, `MachineState` format, reflection-cache pattern). Core §2
  (control machine).

---

## Gaps and orphans

- **`ReBuzz_ManagedMachine_Notes_PedalSH101.md` has no matching public
  repo.** Either the machine is private, archived, or named differently
  on GitHub. Worth resolving — either publish, rename the notes file,
  or annotate the addendum to say "intentionally private".
- **29 machines have no dedicated addendum.** That's fine — most don't
  need one. Add an addendum only when a machine surfaces non-obvious
  findings worth recording, or when its setup deviates from Core/Build
  conventions in a way future-you will need to remember.
- **27 machines have an unknown ReBuzz version.** Not necessarily a
  problem, but when ReBuzz changes in a way that might affect them,
  the answer to "is this still good?" is "build it and find out."
  Stamping the version in each machine's `README.md` would close this
  gap incrementally.

---

## Maintenance

When you push a new machine, edit an existing one, or notice this file
is stale:

1. **For a new machine:** add a row to the roster, in alphabetical
   order. Set `Last pushed` to today, `ReBuzz vs` to the current
   preview, `Addendum` blank unless one's being written. One-line
   description.
2. **For a modified machine:** update `Last pushed`. Bump `ReBuzz vs`
   if the build was retested against a newer preview.
3. **For a new Core/Build §:** if the new finding applies to specific
   existing machines, add a `Depends on Core §N` line to each affected
   machine's addendum (creating an addendum if one didn't exist).
4. **Full refresh from GitHub:** ask Claude to fetch
   `https://api.github.com/users/thepedal/repos?per_page=100` and
   reconcile against this file's roster (additions, deletions,
   description changes). Bump the "Last refreshed" date at the top.

When asking Claude for impact analysis ("ReBuzz changed X in the new
preview, what breaks?"), this file plus the per-machine addenda
together are the answer surface. Quality of the answer scales with
how filled-in the `Depends on` sections and `ReBuzz vs` column are.
