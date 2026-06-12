# ReBuzz Managed Machine Roster

A directory of every public managed machine under
[github.com/thepedal](https://github.com/thepedal), one row per repo.

**Source of truth for the list:** the GitHub API
(`https://api.github.com/users/thepedal/repos?per_page=100`).
This file is a derived snapshot — refresh it when machines are added,
renamed, or archived.

**Last refreshed:** 2026-05-24 — full GitHub API pull (38 repos).
Reconciled the list and every `Last pushed` against `pushed_at`. Changes
this pull: added **pedal-juno106** (new); corrected **pedal-Faze-R** to its
canonical GitHub casing; refreshed 9 stale push dates (chord, fm, invFFT,
M1 and tracker had all been re-pushed since the prior snapshot). No
deletions, archived repos, or forks. `ReBuzz vs` is unaffected by the
pull — the API doesn't carry it.

**Version stamps (2026-05-24):** pedal-fm, pedal-invFFT, pedal-M1,
pedal-juno106 and pedal-tracker stamped **1827-preview** — their recent
v1.x commits explicitly target the ReBuzz 1827 "pvalues field" multi-track
fix, so they were demonstrably built/tested against 1827. Source: the
commit messages, not the Core log.

**Manual add (2026-06-12):** added **pedal-drumgrid** (v1.0, new) with a
dedicated addendum (`ReBuzz_ManagedMachine_Notes_PedalDrumGrid.md`, `Addendum=Y`).
Single-machine edit, not a full GitHub pull — `Last refreshed` above is unchanged.
`ReBuzz vs` left `?` pending the user's stamp (the machine doesn't pin to the 1827
pvalues fix — its trigger grid is global switches, which don't collide — so
there's no hard version dependency; stamp what it was built/tested on).

The point of this file is impact analysis. When something changes in
ReBuzz, this file plus the per-machine addenda lets you answer "which
machines need touching?" by grepping for the relevant Core/Build
`§N` reference.

A second consumer is the **pedal-rebuzz-song-writer** project: its spec→song
recipe needs per-machine **song-authoring facts** (note-column index, track
count / polyphony, direct vs control-driven). Those live in their own section
below, separate from the dev/impact-analysis roster.

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

## Roster (39 machines)

| Repo                    | Type       | Last pushed | ReBuzz vs     | Addendum | Description                                                |
|-------------------------|------------|-------------|---------------|----------|------------------------------------------------------------|
| BTDSys-PeerCtrl-ReBuzz  | control    | 2026-05-21  | 1817-preview  | Y        | Port of BTDSys PeerCtrl                                    |
| pedal-chord             | control    | 2026-05-23  | 1817-preview  |          | Chord peer controller with arpeggiation                    |
| pedal-chorus            | effect     | 2026-05-02  | ?             |          | Chorus effect                                              |
| pedal-comp              | effect     | 2026-05-02  | ?             | Y        | Compressor effect                                          |
| pedal-converb           | effect     | 2026-05-07  | ?             |          | Convolution reverb (SIMD, wavefile IRs)                    |
| pedal-dly-PCM41         | effect     | 2026-05-05  | 1819-preview  |          | PCM41-style tape delay                                     |
| pedal-do-nuttin         | template   | 2026-05-12  | ?             |          | "Do nothing" machine — minimal scaffold                    |
| pedal-drumgrid          | generator  | 2026-06-12  | ?             | Y        | Multi-out drum sampler — 16-lane trigger grid, embedded kits|
| pedal-eq                | effect     | 2026-05-17  | 1819-preview  |          | EQ effect (Core §33 source: v1.3 WM_NOIO handling)         |
| pedal-Faze-R            | generator  | 2026-05-23  | 1827-preview  | Y        | 8-voice phase-distortion synth (Casio CZ lineage)          |
| pedal-fft               | effect     | 2026-05-02  | ?             |          | FFT distortion with harmonics                              |
| pedal-filter            | effect     | 2026-05-19  | ?             |          | Filter effect                                              |
| pedal-fm                | generator  | 2026-05-23  | 1827-preview  |          | FM synth                                                   |
| pedal-folder            | effect     | 2026-05-16  | ?             |          | Wavefolding distortion                                     |
| pedal-follower          | control    | 2026-05-05  | ?             |          | Envelope follower → param assignment                       |
| pedal-gain              | effect     | 2026-04-29  | ?             |          | Gain with mute and inertia                                 |
| pedal-gain-multi        | effect     | 2026-04-25  | ?             |          | Multi-in gain with VU metering                             |
| pedal-gate              | effect     | 2026-05-18  | 1819-preview  |          | Noise gate (Core §16.1 source: v1.3 sidechain build)       |
| pedal-hallverb          | effect     | 2026-05-02  | ?             |          | Hall reverb                                                |
| pedal-hdist             | effect     | 2026-05-16  | ?             |          | Harmonic distortion                                        |
| pedal-invFFT            | generator  | 2026-05-23  | 1827-preview  | Y        | Inverse-FFT additive synth (K5000-inspired)                |
| pedal-juno106           | generator  | 2026-05-23  | 1827-preview  |          | Roland Juno-106 emulation                                  |
| pedal-LFmono            | effect     | 2026-04-29  | ?             |          | Low-frequency mono-maker                                   |
| pedal-lfo               | control    | 2026-05-05  | ?             |          | LFO → param assignment                                     |
| pedal-limit             | effect     | 2026-04-17  | ?             |          | Limiter effect                                             |
| pedal-M1                | generator  | 2026-05-23  | 1827-preview  | Y        | 8-voice poly dual-PCM-osc synth, Korg M1 voice architecture|
| pedal-mcomp             | effect     | 2026-05-19  | 1819-preview  |          | Multi-band compressor (Core mentions v1.1 GUI build)       |
| pedal-muter             | control    | 2026-05-14  | ?             | Y        | Mute other machines                                        |
| pedal-plaits            | generator  | 2026-05-12  | 1819-preview  |          | Port of Mutable Instruments Plaits                         |
| pedal-plate             | effect     | 2026-05-01  | ?             |          | Plate reverb                                               |
| pedal-presetter         | control    | 2026-05-13  | ?             |          | Change presets on target machines                          |
| pedal-profiler          | diagnostic | 2026-04-26  | ?             |          | Global CPU dashboard — v1                                  |
| pedal-profiler2         | diagnostic | 2026-05-22  | 1827-preview  | Y        | Per-machine inspector — v2 (Core §§34–41 source)           |
| pedal-ReBuzz-patcher    | utility    | 2026-04-08  | ?             |          | Patching machine                                           |
| pedal-retrig            | effect     | 2026-05-16  | ?             |          | Port of 'genre' by intoxicated (retrigger)                 |
| pedal-sh101             | generator  | 2026-05-06  | 1819-preview  | Y        | Monophonic Roland SH-101 emulation (synth-voice patterns)  |
| pedal-shaper            | effect     | 2026-05-16  | ?             |          | Waveshaper distortion                                      |
| pedal-tracker           | tracker    | 2026-05-22  | 1827-preview  | Y        | Tracker machine — Matilde-compatible                       |
| pedal-zplane            | effect     | 2026-05-07  | ?             |          | Z-Plane filter (4-corner morph)                            |

ReBuzz versions in this table come from the "Updated with findings
from …" log at the top of `ReBuzz_ManagedMachine_Notes_Core.md`, from
the machine's own addendum (e.g. BTDSys-PeerCtrl-ReBuzz's 1817-preview is
recorded in PeerCtrl §16), or from an explicit version reference in the
machine's commit history (e.g. the five 1827 "pvalues" multi-track fixes).
Remaining `?` rows just weren't stamped — that's a gap to fill in over
time, not evidence the machines are stale.

---

## Machines with detailed addenda

Ten machines have their own notes file, listed in table order. To support
impact analysis, each should grow a `## Depends on` section listing the
Core/Build §-references it relies on. The lists below are seeds — flesh
out from each addendum as time allows.

### BTDSys-PeerCtrl-ReBuzz — `ReBuzz_ManagedMachine_Notes_PeerCtrl.md`
- Port of BTDSys PeerCtrl: a peer controller that maps a 0–100% Value
  through a piecewise-linear curve onto a target machine's parameter,
  with inertia/glide and MIDI input.
- Depends on: Build §1 (csproj), Build §1.3 (deploy target); Core §2
  (control classification via `void Work()`), §4 (`SendControlChanges()`
  after writing to a target).

### pedal-comp — `ReBuzz_ManagedMachine_Notes_PedalComp.md`
- The original debugging target that produced Core §§1–N. Effectively
  the reference implementation for "managed effect machine" patterns.
- Depends on: Core (foundational — most sections trace back here),
  Build §1.2 (csproj properties), §1.3 (post-build deploy), §2
  (AssemblyName).

### pedal-drumgrid — `ReBuzz_ManagedMachine_Notes_PedalDrumGrid.md`
- 16-lane multi-out drum sampler. Trigger grid = 16 global switch columns in the
  pattern editor; per-lane Velocity/Pitch track columns; per-lane audio outputs;
  GUI wave assignment; self-contained `.pdrumgrid.xml` kits with embedded audio.
  The reusable findings are the trigger-grid layout technique (§1) and the
  load-time `pvalues` clobber that silences lanes (§2) — both apply to any
  multi-track managed machine.
- Depends on: Build §1.2 (csproj — fully compliant), §1.3 (deploy →
  `Gear\Generators`), §2 (`.NET` AssemblyName), §1.1/§6.1 (reference only
  `BuzzGUI.Interfaces` + `BuzzGUI.Common`, **not** `ReBuzz.exe`); Core §9
  (`bool`→Switch), §25 (`IsStateless`), §14 + §42 (multi-track collision /
  `int[256]` pvalues), §26.7 (GUI base class), §38 (sample scale), §39
  (`MachineState`); Tracker §§12.1–12.3 (multi-out), §16.3 (pvalues reader), §4.2,
  §7.1, §7.7, §2.3; Chord §3, §11 (swing); M1 §2, §2.1; PedalTracker §13.1.
  Not pinned to 1827 (global triggers don't collide; the pvalues recovery was
  removed) — stamp the actual build.

### pedal-Faze-R — `ReBuzz_ManagedMachine_Notes_PedalFazeR.md`
- 8-voice polyphonic phase-distortion synth (Casio CZ lineage): two PD
  oscillators per voice (mix/ring/sync), a DCW "wave" envelope standing in
  for a filter, amp + pitch envelopes, one LFO, a gentle non-resonant tone
  LP, selectable Off/2×/4× oversampling; no GUI. Faze-R-specific material is
  the phase-distortion engine + contextual DCW (§§1–3) and the oversampling
  decimator (§4); poly/voicing is inherited from M1/Core.
- Depends on: Build §1.2 (csproj — fully compliant), §1.3 (deploy →
  `Gear\Generators`; also ships a preset bundle), §2 (`.NET` AssemblyName);
  Core §14 (multi-track note recovery — poly), §27, §29; M1 §5, §7 (tone-LP
  topology), §10 (poly voice architecture); SH101 §1 (`FastPow2`), §8 (mixer
  headroom). Built on 1827-preview.

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
- Depends on: Core §§34–41 (audio-thread chunking,
  `MachinePerformanceData`, `EngineSettings`, `MasterTap`, Buzz sample
  scale, `MachineState` format, reflection-cache pattern, and §41
  `AudioBufferFillThread`). Core §2 (control machine).
  1827: built/tested through v1.7.7 on 1827-preview — MasterTap moved
  to a GUI-thread event (§37) and the fill-thread cadence (§41) both
  required handling; see PP2 §13.

### pedal-sh101 — `ReBuzz_ManagedMachine_Notes_PedalSH101.md`
- Monophonic Roland SH-101 emulation: single VCO (saw/pulse/sub/noise off
  one shared phase), ZDF Moog-ladder VCF, ADSR, LFO, portamento; no GUI,
  no internal sequencer. The addendum's findings are general per-sample
  synth-voice patterns, not SH-101-specific.
- Depends on: Build §1.2 (csproj — fully compliant), §1.3 (deploy →
  `Gear\Generators`), §2 (`.NET` AssemblyName suffix), §4 (`NoWarn
  MSB3277`); Core (managed-generator `Work()` / note handling), §7 (note
  encoding); PedalComp §1 (±32768 sample range), §5 (fast curve
  approximation — generalised to base-2 `FastPow2` here).

### pedal-tracker — `ReBuzz_ManagedMachine_Notes_PedalTracker.md`
- Matilde-compatible tracker; cross-machine pattern manipulation.
- Depends on: Core §§1–2 (Work / control classification), §14
  (multi-track note workaround) likely relevant, Tracker addendum
  §§11–12 (programmatic pattern manipulation, multi-out per-track
  generators), MPE integration (Tracker addendum §11.1, §11.2).
  1827: §16.6 confirms SubTickTiming + AudioBufferFillThread are
  red herrings for the multi-out contract (already checked clean).

---

## Song-authoring facts (`.bmxml` generation)

A separate consumer of this roster is the **pedal-rebuzz-song-writer** project,
whose spec→song recipe (`ReBuzz_SongFormat_Notes_BMXML.md` §13) needs, for every
machine type it emits: which pattern column carries the **Note**, how many
**tracks** (voices) the machine supports, and whether it's played **directly** or
driven by a **control machine**. These are easy to get wrong — the note column is
**not always the last** (Faze-R's is col 67 of 69) and some "synths" are
monophonic regardless of `<TrackCount>` — so the rule is **determine empirically**
(enter one note, save, decode) and record the result here once a machine has been
characterized.

Conventions: note value = `octave*16 + noteIdx + 1`; note-off (where supported) =
`255`. Everything below is verified byte-exact against real saves; the deeper
mechanics are in `ReBuzz_SongFormat_Notes_BMXML.md` §4.4 (note-column table),
§4.6 (multi-track layout) and §12 (control machines).

### Generators (carry or receive notes)

| Machine (Library) | role in Limani | play mode | total cols (1 trk) | note col | track-group params | polyphony | song-gen notes |
|---|---|---|---|---|---|---|---|
| Pedal Plaits  | drums (Kick/Snare/Hats) | direct | 14 | **12** | 2 (Note, Velocity) | one mono machine per drum | trigger note **65** (C-4); timbre from the machine's own params |
| Pedal SH101   | Bass | direct *or* control-driven | 26 | **25** (last) | 1 (Note) | **monophonic** — `<TrackCount>` ignored | fine as a mono arp target; can't hold a chord |
| Pedal invFFT  | Pad  | control-driven (chord) | 29 | **28** (last) | 1 (Note) | multi-track (raise `<TrackCount>`) | chord target ⇒ **≥3 tracks** (Limani uses 6) |
| Pedal Juno106 | Comp | control-driven (chord) | 26 | **25** (last) | 1 (Note) | multi-track — **6-track verified** | chord target ⇒ **≥3 tracks** |
| Pedal Faze-R  | Lead | control-driven (arp) | 69 | **67** (NOT last) | 2 (Note@67, Velocity@68) | 8-voice poly | note col is mid-pattern — always decode to confirm |

### Control machines (no audio; only their editor → Master; sequenced; target named in state)

| Machine (Library) | drives | own pattern | key columns | target tracks needed | song-gen notes |
|---|---|---|---|---|---|
| Pedal Chord    | one generator | 14 cols, single-track | col 0 = **Note** (root), col 2 = chord type, col 3 = mode (0 block / 1–5 arp) | chord mode spreads across the target's consecutive tracks ⇒ **≥3**; arp mode is one note ⇒ mono OK | note-off = **255** in col 0; a voice entering after bar 1 needs a **row-0 note-off** (§12.9) |
| Pedal Presetter | up to **16** generators | `Preset` only, multi-track | `Preset` (Byte 0–253, NoValue 255) at colIdx **0** per track, `base`=0 | n/a (sets timbre, not notes) | preset index = 0-based **bank position**; **one fire per row** — stagger multi-target changes (§12.10) |

As more machines get used in songs, add a row here (and grab a real `<Machine>`
block of each into the song-writer's `refs/`).

---

## Gaps and orphans

- **29 machines have no dedicated addendum.** That's fine — most don't
  need one. Add an addendum only when a machine surfaces non-obvious
  findings worth recording, or when its setup deviates from Core/Build
  conventions in a way future-you will need to remember. (Newest without
  one: **pedal-juno106**.)
- **24 machines have an unknown ReBuzz version.** Not necessarily a
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
5. **Characterized a machine for song generation:** add a row to
   **Song-authoring facts** with its note-column index, track-group
   params, polyphony, and play mode. Get the note column empirically
   (enter one note, save, decode) — it isn't always the last column.

When asking Claude for impact analysis ("ReBuzz changed X in the new
preview, what breaks?"), this file plus the per-machine addenda
together are the answer surface. Quality of the answer scales with
how filled-in the `Depends on` sections and `ReBuzz vs` column are.
