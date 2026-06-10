# ReBuzz Song-Writer (pedal-rebuzz-song-writer) — Project Overview

Source: compiled 2026-06-10 from the **pedal-rebuzz-song-writer** build sessions
against ReBuzz **Build 1827**. This is the orientation layer — what the project
is, how its parts stack, the authoring loop, and the rules that must not be
broken. It carries no new format facts of its own; every mechanism it names is
documented in detail elsewhere and cross-referenced.

This file documents the **song-writer tool** — the Python library and workflow
that assemble loadable `.bmxml` songs. It is a different concern from the two
sibling bodies of notes: the **format facts** (how a `.bmxml` is laid out on disk
and how note data is stored) live in `ReBuzz_SongFormat_Notes_BMXML.md`,
cross-referenced here as **BMXML §N**; the **machine-development** notes (writing
the machines themselves) are `ReBuzz_ManagedMachine_Notes_*`, cross-referenced as
**Core §N / Build §N / Roster**. Sections below are numbered locally.

Worked examples throughout: **Limani** (D-Hijaz, single-pattern, 28 machines —
the same song that anchors the BMXML notes) and **Last Call** (A-blues, sectioned
arrangement, a drum bus + per-synth gains + master limiter, and a real direct-note
FM lead on top of a Pedal-Chord arp, each synth through its own effect) — Last Call is authored as a single
declarative `compose(spec)` dict (§2). **DslDemo** is a short piece in the fluent
DSL, showing the imperative form of the same model.

---

## 1. What this is, and what to trust

Generate **loadable ReBuzz `.bmxml` songs programmatically**, from a byte-exact
model of the format, with a toolkit reusable enough to commission arbitrary songs
(style / key / tempo / instruments) rather than hand-built one-offs. This is the
implementation of the **spec → song** goal stated in BMXML §13.

Confidence is not uniform across what the tool produces — be explicit about it:

| Layer | Confidence | Why |
|-------|-----------|-----|
| **File structure** (machines, routing, patterns, blobs) | very high | verified byte-exact against real saves; the validator (§2) enforces it before every write |
| **Mix** (levels, balance, peaks, timing) | now measurable | the render → stem → measure loop (§3) replaced ear-guessing with numbers |
| **Musical quality** (does it actually sound good) | low / unverifiable in-tool | composed without hearing it; only a human render confirms it |

### The cardinal rule

**Never fabricate a `<Machine>` block. Splice a real one.** Every machine in
every song is lifted verbatim from a real ReBuzz save. Hand-building a machine
block from scratch is the root cause behind the original load-time NRE and the
"can't edit / can't add tracks" failures (BMXML §6, error chronology §11 #3/#5),
and it stays banned — including a *fifth* synth: an extra lead is spliced from the
machine reference, not invented (§2, §5). A new machine type means saving a real
block of it once and recording its **note-column index / track params** in the
**Roster** song-authoring section before the tool can emit it.

---

## 2. How the library is layered

The Python package (`rebuzz`) is built in layers; the lower ones are byte-level
primitives, the upper two are what a human actually drives.

| Part | Role |
|------|------|
| **blob** | byte-exact Modern Pattern Editor blob builders (BMXML §4, §5); note-value encoding, note-off `255` |
| **blocks** | read a save; find / extract / edit `<Machine>` blocks; set params, positions; grow a bus to N inputs; overlap check |
| **control** | Pedal Chord / Pedal Presetter state + patterns — the control-machine layer (BMXML §12) |
| **song** | connections, sequences, splice, tempo (BMXML §8), loop / song-end, held-note clear, file write |
| **theory** | note parsing, scales / modes, chord & arp-mode code tables |
| **dsl** | `Song / Section / Chords / Arp / Drums / Melody` → `compile()`; `Song.synth()` adds an extra synth for a `Melody`; `Song.fx()` inserts an effect before a synth gain; `compose(spec)` builds a whole song from one declarative dict — the musical front end |
| **validate** | structural + loop-safety checks; runs as `assert_valid` before every write |
| **mix** | measure pre-fader stems (peak / RMS / LUFS / clip), solve bus gains — an **offline** tool (needs numpy), not part of the byte-exact core |

Alongside the library sit the per-song **build scripts** (Limani, Last Call,
DslDemo), a **machine-reference** builder + a **catalog** generator (which emits a
machine table: type, params/ranges, note column per machine), a **stem-measurement**
CLI, and the **PowerShell packer** (§3). A reference set of real saves is the only
legal source of machine blocks (the cardinal rule, §1).

**Voices** the DSL understands: `Chords` (block chords on a rhythm, optionally
per-section), `Arp` (one root per bar + arp config, optional swing), `Drums`
(step-string grooves per kit slot, optional shuffle swing), and `Melody` — a real
monophonic line of pitched note-ons/offs written **straight into a synth** with no
control machine (BMXML §20). The four built-in synth slots (Bass/Lead/Pad/Comp)
are fixed by the reference rig; `Song.synth(name, library, note_col)` registers a
fifth, spliced from the machine reference, that a `Melody` can play.

---

## 3. The authoring workflow loop

```
 compose ─► build ─► validate ─► deliver ─► render ─► measure ─► trim / limit ─► (repeat)
  (DSL or   (splice   (assert_   (PowerShell (human,   (stem      (set bus gains,
  build_*)  blocks)   valid)      writer)    to wav)   measure)   limiter ceiling)
```

- **compose** — a `Song` in the DSL, or a hand-written build script for full control.
- **build** — splice real blocks, attach editor blobs, wire connections + sequences.
- **validate** — `assert_valid` runs before every write, so a wiring or loop bug
  fails the build instead of surfacing as silence or a drone inside ReBuzz.
- **deliver** — the song is shipped as a PowerShell **gzip + base64** writer with a
  round-trip check, because plain browser downloads corrupt the `.bmxml` /
  mangle the BOM (BMXML §9). The user runs it in `Downloads`.
- **render** — the human renders in ReBuzz, ideally as **pre-fader stems**.
- **measure** — the stem-measurement tool reports true levels and solves bus gains.
- **trim / limit** — gains applied per synth (and to the drum bus); a Pedal Limit caps peaks.

Everything the tool can verify offline (structure, levels, peaks, timing,
clipping) it closes itself; only subjective musical quality needs the human ear.

---

## 4. Invariants (do not break these)

1. **Splice, never fabricate** machine blocks (§1; BMXML §6).
2. **`assert_valid` before write** — every build script does this.
3. **Loop-safety** — wherever a *control-driven* voice falls silent, release it
   two ways: a note-off on the **target synth** (cuts a long release/sustain the
   control machine can no longer shorten) **and** a one-row stop pattern on its
   **Pedal Chord** (halts further triggers), both at the silent section's start,
   which also covers the loop wrap (BMXML §12.9.1). A direct-note `Melody`
   releases itself (it writes its own note-offs), and drums (one-shot) and tick-0
   voices are exempt.
4. **Gain-stage from measurement**, not by ear — and watch for *source* clipping:
   a clipped stem means no downstream trim can truly fix it (this is exactly what
   the pad analysis caught — it was pinned at full scale at the source).
5. **Deterministic builds** — re-running a build yields identical bytes; tests
   assert this and Limani has a byte-baseline for regression.
6. **Delivery via PowerShell**, UTF-8 BOM preserved (BMXML §9). Commit messages
   contain **no double quotes** (Build §10 convention).
7. **Playable notes live in the MPE blob**, not the PatternCore `<Columns>`
   (which ReBuzz ignores at playback — BMXML §3, §11 #4).

---

## 5. Conventions worth knowing

- A generator's **note column = (#Global-group params) + (Note's index among
  Track-group params)** — derivable statically from a saved block, so a new
  machine's note column can be read off the catalog rather than re-derived by
  enter-and-decode. (The column is **not** always the last one — Faze-R's is col
  67 of 69, Pedal FM's is 42; see the Roster song-authoring notes.)
- **Submix topology**: drums sum on one **Pedal Gain Multi**; each **synth gets its
  own single Pedal Gain**; the drum bus and all synth gains then feed the
  **(limiter) → Master**. Per-input gain lives in both the connection `Amp` and the
  machine's per-track `Amp`, kept equal. Per-synth gains exist so the human can
  render a **stem per synth** and the measurement loop can read relative levels.
- A **gain machine sizes itself to its inputs**: Pedal Gain / Gain Multi keep their
  per-connection faders in an `Input` group whose track count must equal the
  connection count. The builder grows that group (new channels default to unity,
  the unconnected `<Values />` form handled) — a Gain Multi to its source count, a
  single Pedal Gain to one.
- **dB → amp**: `16384 = unity`; `amp = round(16384 · 10^(dB/20))`.
- **Presetter staggering**: never fire two presets on the same row — Build 1827
  applies only the last (BMXML §12.10). One preset per row.

---

## 6. Status & open frontiers

**Done:** a byte-exact format model; the `rebuzz` library; a composition DSL;
structural + loop-safety validation; measurement-driven gain-staging; a master
chain (drum bus + per-synth gains → limiter → Master); a catalog of ~35 spliceable
machines; a **real melodic lead** — a direct-note `Melody` voice on a synth spliced
in via `Song.synth()` (e.g. Pedal FM), each synth on its own recordable gain (Last
Call's "Lead2"; BMXML §20); and a one-call **`compose(spec)`** entry point that
builds a whole song from a single JSON-serialisable dict (BMXML §21); and
**per-synth serial FX** — an effect (or chain) inserted between a synth and its
gain, e.g. Last Call's Chorus/Plate/Hallverb/slapback per instrument (BMXML §22).

**Open (roughly in leverage order):**
- **Per-section voicing** overrides for the chord / arp voices.
- Add the machines still missing from the machine reference (**M1, PeerCtrl, LFO, Muter**).
- **Polyphonic / multi-track direct melodies** (chords or harmony lines) and
  per-note velocity, on top of the current monophonic `Melody`.

---

## 7. Repo map & where to look

```
src/rebuzz/        the library (the modules in §2)
src/build_*.py     song builds: limani, lastcall (a compose(spec) dict), dsl_demo, machine_ref
src/catalog_machines.py   MACHINES.md generator
src/mix_report.py  stem-measurement CLI
src/pack_ps1.py    .bmxml → PowerShell writer
refs/              real ReBuzz saves spliced into songs (the only legal machine source)
  MachineRef.bmxml   one instance of every available machine — the splice source
  MACHINES.md        generated catalog: type, params/ranges, note column per machine
songs/             built .bmxml + their Write-*.ps1
tests/             pytest: validate / dsl / mix (determinism + byte-identity)
docs/              ReBuzz_SongFormat_Notes_BMXML.md (the deep format/technique reference)
```

| Need | Where |
|------|-------|
| Install, quickstart, package surface | `README.md` |
| Byte layout, recipes, control machines, effects, DSL, mixing, limiter, direct-note lead, compose(spec), per-synth FX | `ReBuzz_SongFormat_Notes_BMXML.md` (**BMXML §§1–22**) |
| Per-machine params / note columns / ranges | `refs/MACHINES.md` |
| Writing the machines themselves | `ReBuzz_ManagedMachine_Notes_*` (**Core / Build / Roster**) |

Format-notes quick index: BMXML §§1–9 format & assembly · §10 Limani design ·
§11 error chronology · §12 control machines · §13 spec→song recipe · §14
multi-pattern · §15 effects/buses · §16 validation · §17 DSL · §18
measurement-driven mixing · §19 master limiter · §20 direct-note melodic lead · §21 compose(spec) · §22 per-synth FX.
