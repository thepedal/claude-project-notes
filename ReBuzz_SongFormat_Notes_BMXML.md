# ReBuzz Song-File (.bmxml) & Note-Data Format — State of Play

Source: full reverse-engineering session against ReBuzz **Build 1827**
(github.com/wasteddesign/ReBuzz source + byte-level analysis of real saved
songs). All claims below were verified either against the ReBuzz loader source
or by **byte-exact round-trip** (regenerate a real saved blob and compare ==
original). Where a fact is empirical-only it is marked *(empirical)*.

Updated with findings from the Limani assembly session: load-time machine
**positions** (§2.4), **held-note-on-load** suppression (§2.5), MPE event
values as **literal parameter values** not just notes (§4.3), and the corrected,
verified mechanism for **song tempo** — BPM/TPB live as events in a pattern on
the **Master's own track**, not in the Master parameters, which the BMXML
loader ignores (§8). The §8 tempo finding supersedes an earlier (wrong)
"can't be set from the file" conclusion. Also adds **multi-track/polyphony**
encoding (§4.6) — `<TrackCount>` plus track-major repeated columns, with the
per-record `track` field that earlier looked like a constant `reserved 0`.

**Control-machine layer (new, §12).** The earlier "Pedal Chord throws an NRE,
dropped" conclusion was **wrong** — the NRE came from a *fabricated* machine
block (§6's cardinal rule), not from the machine. With a **real** Pedal Chord
block, one control machine per voice drives a target generator with live
chords or arpeggios; §12 is the full song-authoring recipe (state-targeting,
the 14-column map, chord vs arp modes, polyphony rules), verified working in
Limani against `chordref.bmxml` (chords) and `arpref.bmxml` (arps). **§13** then
generalises the whole pipeline into a **spec → song** recipe — the standing goal
of being able to ask for "a piece in style X, instruments Y/Z, tempo T."

This file documents the **XML song format** and the **Modern Pattern Editor
(MPE) note-data blob** — i.e. how a `.bmxml` song is laid out on disk and how
playable note data is actually stored. It is a different concern from the
`ReBuzz_ManagedMachine_Notes_*` files (which cover writing the machines
themselves). Cross-references to those use `Core §N` etc.

Worked example throughout: the song **"Limani"** (**26 machines**: a 4-piece
Plaits drum kit played directly, + bass/lead/pad/comp synths each **driven by
its own Pedal Chord** control machine, ~64 s, D Hijaz). Its build script is the
canonical implementation of everything here. (The original Limani drove the
synths with hand-written melodies; it was later converted to the control-machine
layer of §12 — both arrangements are valid and the format facts are identical.)

---

## 1. File container & loader

- The loader dispatches on **extension**: `.bmx` / `.bmw` → legacy **binary**
  loader; **anything else** (incl. `.bmxml`) → `BMXMLFile` `XmlSerializer`.
- The serializable root type is `BMXMLSong`, but it carries
  `[XmlRoot("ReBuzzSong")]`, so the on-disk root element **must be
  `<ReBuzzSong>`**, not `<BMXMLSong>`. Using the class name fails to load.
- **Build 1827 requires a UTF-8 BOM** (`EF BB BF`) at the start of the file.
  Without it the parser throws *"Data at root level is invalid, line 1
  position 2"*. Always write the BOM.
- The XML declaration is `<?xml version="1.0" encoding="utf-8"?>`.

### 1.1 Root element

```xml
<ReBuzzSong xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:xsd="http://www.w3.org/2001/XMLSchema"
            Name="Limani" Build="1827"
            BuildString="ReBuzz Build 1827 2026-05-22T11:47:15&#xD;&#xA;">
```

- Only the **root** carries namespace declarations. In practice the document
  body uses **no namespaced elements** — there were **0** `xsi:nil` and **0**
  `xsi:type` occurrences in real saved songs. This is what makes raw-text
  splicing safe (see §6): you can move `<Machine>` subtrees between files as
  literal byte ranges without namespace fix-ups.

### 1.2 Top-level child order *(XmlSerializer is order-sensitive)*

```
SubSenctions   (sic — misspelled in the schema)
Machines
MachineGroup
Sequences
Waves
MachineConnections
InfoText
LoopStart
LoopEnd
SongEnd
Speed
```

Tail values for a 16-bar song: `LoopStart=0`, `LoopEnd=256`, `SongEnd=256`,
`Speed=0`. (`Speed` here is **0**; BPM/TPB are set elsewhere / by the user
after load — see §8.)

---

## 2. `<Machine>` element

### 2.1 Child order

```
Library
[Data]              # present for managed machines & editors; absent on Master
[EditorMachine]     # present on generators that have a pattern editor
Name
Patterns
ParameterGroups     # always exactly 3 groups: 0=Input, 1=Global, 2=Track
Attributes
Type
X
Y
IsMuted
MIDIInputChannel
OverrideLatency
OversampleFactor
IsWireless
BaseOctave
ParameterWindowVisible
GUIWindowVisible
IsBypassed
```

- **`Library`** = the machine's DLL filename minus the trailing `.NET.dll`.
  E.g. library `Pedal SH101` ⇒ `Pedal SH101.NET.dll`. (`Master` and
  `Modern Pattern Editor` are special built-ins.)
- Connections, sequences and editor links all resolve machines **by `Name`**,
  never by index. Names must be unique across the song.

### 2.2 `<Parameter>` child order

```
Values, Type, Name, Description, MinValue, MaxValue, NoValue, Flags, DefValue
```

### 2.3 Managed-machine `<Data>` (state blob)

A managed generator/effect stores its serialized state as base64 of:

```
[byte 0x02 = version][int32 LE = XML-bytes size][XML state bytes]
```

An **empty** state is base64 `AgAAAAA=` = `02 00 00 00 00` (version 2, size 0,
no XML). Plaits drums in the worked example all use this empty state; their
timbre is set by their own parameters, not by this blob.

> **Critical distinction:** an **editor** machine's `<Data>` is **NOT** this
> wrapped format — it is the **raw MPE blob** (starts `FF 01 …`), with no
> 5-byte version/size header. See §4.

### 2.4 Machine-view positions (`<X>` / `<Y>`)

Each machine carries normalized float coordinates (`<X>`, `<Y>`, roughly the
range −1.5 … +1.5; Master is at `0,0`). **Machines saved at `0,0` stack
directly on top of the Master in the machine view.** When assembling a song
from blocks that were never positioned (e.g. spliced from a file where the user
left everything at the origin), assign each visible generator a **distinct**
`X`/`Y` so nothing overlaps on first load. Hidden `Modern Pattern Editor`
machines can stay at `0,0` — they don't appear in the machine view.

The position element appears once per machine, right after `<Type>Generator</Type>`:
`<Type>Generator</Type><X>-1.2</X><Y>0.4</Y>` — target it there when editing.

> **Visible range is bounded — off-canvas machines don't render *(empirical,
> Limani §12 session).*** A machine loads, plays, and shows in the **sequencer**
> regardless of its `X`/`Y`, but the **machine view** only draws the populated
> region. In Limani the real machines span roughly **X ∈ [−1.2, 0.82],
> Y ∈ [−1.25, 1.0]**; four control machines parked at **X = 1.6** loaded and
> ran correctly (audible, sequenced) but were **invisible in the machine view**
> because they sat past the right edge. Keep every machine you want to *see*
> within the cluster the other machines occupy (a safe rule of thumb is to stay
> inside the min/max `X`/`Y` already used by the song); the user can drag them
> anywhere afterwards and ReBuzz re-saves the new spot. This is independent of
> connections — an unconnected generator (e.g. a Pedal Chord, §12) still draws,
> as long as it's on-canvas.

### 2.5 Load-time held note (silence on load)

Every machine's **Track**-group `Note` parameter (`<Type>Note</Type>`) stores a
*current* value — a snapshot of the last note sounding when the song was saved
— and this is **separate from the sequenced notes** in the editor blob. On load
ReBuzz applies this stored value, so a **non-zero** Track `Note` makes the
machine sound the instant the file loads (a sustaining synth holds it as a
drone; a percussive Plaits just makes one decaying hit). This is independent of
the Play button.

To guarantee **no sound until Play**, set every generator's Track `Note` stored
value to its `NoValue` (`0`). Playback is unaffected — performance notes come
from the sequencer/editor blob, not this parameter:

```python
# clear all held notes (the <Value> precedes the <Type>Note</Type>)
out = re.sub(r'<Value>\d+</Value>(\s*</Value>\s*</Values>\s*<Type>Note</Type>)',
             r'<Value>0</Value>\1', out)
```

> Note for multi-track machines (§4.6): only **track 0**'s `<Value>` is
> immediately before `</Values>`, so this regex clears just that one. Tracks
> 1…N take the parameter default (Note default `0`), so they stay silent anyway.

### 2.6 Track count (polyphony)

The number of tracks (voices) is the **`<TrackCount>`** of the **Track**
parameter group (`ParameterGroups[2]`). The loader builds the machine with that
many tracks. Raising it (e.g. to 6) gives that many note columns per pattern;
the matching change in the editor blob is in §4.6. Some machines are
monophonic (e.g. Pedal SH101) and ignore a higher count.

---

## 3. Where playable notes live (the big one)

**ReBuzz ignores the `PatternCore` `<Columns>` for playback.** A pattern's
note data on the PatternCore side can be fully populated and the song will
still play silently. The authoritative note data lives in **hidden "Modern
Pattern Editor" machines**, one per generator.

Consequences for hand-building songs:

- Give each generator **one** PatternCore pattern with **empty** columns:

  ```xml
  <Patterns>
    <Pattern>
      <Name>00</Name>
      <Length>256</Length>   <!-- rows; this is where pattern length lives -->
      <Columns />
    </Pattern>
  </Patterns>
  ```

- Put the real notes in that generator's editor machine's blob (§4–§5).
- The **pattern name** in PatternCore, in the editor blob, and in the sequence
  event must all **match** (we use `00` everywhere).

### 3.1 Editor machines

- Library `Modern Pattern Editor`, `Type` = Generator, connected to `Master`.
- Names are `XmlConvert.EncodeName("\x01peN")` ⇒ literal text **`_x0001_peN`**
  (N = 1, 2, 3 …). `_x0001_pe1` is **always Master's** editor.
- A generator points at its editor via `<EditorMachine>_x0001_peN</EditorMachine>`.
- Each editor is itself wired to Master in `<MachineConnections>` (same as a
  normal generator-to-Master connection).

### 3.2 PatternCore note encoding (for reference / parity)

Even though PatternCore columns are ignored at playback, the note **byte value**
encoding is shared with the MPE blob:

```
value = octave*16 + noteIdx + 1        (noteIdx: C=0,C#=1,D=2,D#=3,E=4,F=5,
                                         F#=6,G=7,G#=8,A=9,A#=10,B=11)
decode:  v-=1 ;  noteIdx = v % 16 ;  octave = v // 16
```

Verified against real input: `C#4=66`, `D#4=68`, `D2=35`, and the drum
trigger `C-4=65`.

---

## 4. Modern Pattern Editor (MPE) blob — byte layout (fully decoded)

The editor's `<Data>` is base64 of this little-endian binary blob. **All of it
was confirmed by byte-exact round-trip** against 8 real saved blobs (drums +
synths, including multi-pattern and empty cases).

### 4.1 Header (10 bytes for an empty blob)

```
FF 01                       # 2-byte magic
uint32 LE  totalBlobLength   # *** includes the magic and these 4 bytes ***
int32  LE  patternCount
```

> The `totalBlobLength` field was the long-standing "mystery bytes." It is the
> **entire blob length in bytes**. Verified: empty blob = 10; Bass = 925;
> Pad = 998; Lead = 2387; Comp = 925 — each equal to the actual byte length.
> **You must compute and write this when generating a blob.**

An **empty** editor blob is exactly:
`FF 01  0A 00 00 00  00 00 00 00` (length 10, patternCount 0). Master's
`_x0001_pe1` is normally this.

### 4.2 Per pattern

```
asciiz patternName          # null-terminated; must match PatternCore + sequence
int32  f1   = 4             # constant (empirical; likely TPB/col-default)
int32  f2   = 4             # constant
int32  recordCount          # = number of column-records (see §4.4)
recordCount × column-record
```

`recordCount == 0` ⇒ an **empty pattern** (just a named placeholder, no notes).
`recordCount == <machine's full column count>` ⇒ a populated pattern; **all**
columns are emitted, but only the note column carries events.

### 4.3 Per column-record

```
asciiz columnName           # = the GENERATOR's name (e.g. "Bass"), not editor's
int32  colIdx               # 0-based column index
int32  track                # track index (0 for single-track machines & non-track columns)
byte   flag     = 0
int32  eventCount
eventCount × { int32 pos ; int32 value }     # pos = row * 240 ; value = column's param value
int32 ×4 trailer  (all = 4)                  # per-column, not per-row
```

- `pos = row * 240` *(empirical; 240 = internal ticks per row)*.
- `track` is the voice index. It is `0` for non-track columns and for any
  single-track machine — which is why it looks like a constant zero until you
  examine a multi-track machine (§4.6). For a track-group parameter column it
  carries the track number.
- `value` is the **literal value of that column's parameter** at that row. For a
  **Note** column it's the note byte (§3.2; e.g. `66` = C#4, `65` = C-4 drum
  trigger). For a non-note parameter column it's that parameter's raw integer
  value — e.g. the Master's BPM/TPB columns carry `60` and `4` verbatim (§8),
  not note-encoded. So events automate any pattern-editable parameter, not just
  notes.
- The trailer is four int32s each `= 4`, **once per column** regardless of how
  many events — so blob size scales with event count only.

### 4.4 Column counts & note-column index per machine

The number of column-records equals the machine's exposed pattern-column count.
Notes land in that machine's **Note parameter** column — which is **not always
the last column**, so determine it empirically (enter one note, save, decode).
The table below is the **single-track** layout; `base` = the colIdx of the first
track-group parameter (= number of non-track columns), and machines with >1
track-group param have more than one track column (see §4.6).

| Machine (Library) | role | total cols (1 trk) | note col | track-group params | `base` |
|---|---|---|---|---|---|
| Pedal Plaits   | drums | 14 | **12** | 2 (Note, Velocity) | 12 |
| Pedal SH101    | Bass | 26 | **25** (last) | 1 (Note) — **monophonic** | 25 |
| Pedal invFFT   | Pad  | 29 | **28** (last) | 1 (Note) | 28 |
| Pedal Juno106  | Comp | 26 | **25** (last) | 1 (Note) | 25 |
| Pedal Faze-R   | Lead | 69 | **67** (NOT last) | 2 (Note@67, Velocity@68) | 67 |

(`base` is also `total cols − (track-params × 1)` for the single-track case.)

### 4.5 Length-agnostic

The blob has **no row-count field** (f1/f2 are the constant `4`, not the row
count). The same format therefore serves **any** pattern length — a 16-row clip
and a 256-row full-song pattern use identical structure; only the event `pos`
values and the PatternCore `<Length>` differ. This is why a single long
pattern per machine works.

### 4.6 Multiple tracks (polyphony)

A machine's track count is set by **`<TrackCount>`** in its **Track** parameter
group (group index 2). The loader reads `machineData.ParameterGroups[2].TrackCount`
and builds the machine with that many tracks; per-track parameter values that
aren't present default (so you need not expand the Track group's `<Values>` to
N entries — though ReBuzz writes N on save). Setting `<TrackCount>6</TrackCount>`
makes the machine load with 6 tracks; the pattern editor then shows that many
note columns.

In the **blob**, extra tracks do **not** add new colIdx values — each
track-group parameter column **repeats once per track at the same colIdx**, in
**track-major** order (all of track 0's track columns, then track 1's, …), and
each record carries its **`track`** index (§4.3). So:

```
recordCount = base + trackCount × (number of track-group params)
column layout = [ col 0 … col base-1 (non-track, track 0) ]
                [ for t in 0..trackCount-1:  for each track-param colIdx pc:  (pc, track=t) ]
```

Example — 6-track Juno106 (`base`=25, one track param Note@25):
`recordCount = 25 + 6×1 = 31`; columns 0–24 once, then colIdx 25 six times with
`track` = 0…5. Verified **byte-exact** against a real 6-track save. A 6-track
Faze-R (two track params 67/68) gives `67 + 6×2 = 79`, emitting `(67,t)` then
`(68,t)` for each track.

A given voice's notes live in that track's note-column record; **track 0** is the
first track-param record group, so an existing single-track part stays in place
when you raise the track count and leave tracks 1…N empty.

**Monophonic machines.** Some generators cap at one track (e.g. **Pedal SH101**),
and a higher `<TrackCount>` won't give real polyphony — verify per machine. When
in doubt, save one instance at the desired track count and decode it.

---

## 5. Verified generator & decoder (Python)

This `build_blob` reproduced **all 8** single-track reference blobs
byte-for-byte; `build_blob_mt` reproduces the multi-track (6-track) reference
byte-for-byte. Use `build_blob_mt` for everything — it subsumes `build_blob`.

```python
import struct, base64

def build_blob(machine_name, patterns):
    # single-track form. patterns: list of dict(name, ncols, notecol, events=[(row,value)])
    body = struct.pack('<i', len(patterns))
    for p in patterns:
        body += p['name'].encode('latin1') + b'\x00'
        body += struct.pack('<iii', 4, 4, p['ncols'])       # f1, f2, recordCount
        for c in range(p['ncols']):
            body += machine_name.encode('latin1') + b'\x00'  # column name = GEN name
            body += struct.pack('<ii', c, 0) + b'\x00'       # colIdx, track(=0), flag
            evs = p['events'] if c == p['notecol'] else []
            body += struct.pack('<i', len(evs))
            for row, val in sorted(evs):
                body += struct.pack('<ii', row * 240, val)
            body += struct.pack('<iiii', 4, 4, 4, 4)         # trailer
    total = 2 + 4 + len(body)                                 # magic + lenfield + body
    return b'\xff\x01' + struct.pack('<I', total) + body

def build_blob_mt(machine_name, patname, track_param_cols, trackcount, events):
    # multi-track. track_param_cols = colIdx of each Track-group param (e.g. [25]
    # Note, or [67,68] Note+Velocity). Non-track columns are 0..min(cols)-1.
    # events keyed by (track, colIdx) -> [(row,value),...]. Track-major order.
    base = min(track_param_cols)
    recs = base + trackcount * len(track_param_cols)
    body = struct.pack('<i', 1) + patname.encode('latin1') + b'\x00' \
         + struct.pack('<iii', 4, 4, recs)
    def rec(colidx, track, evs):
        r = machine_name.encode('latin1') + b'\x00' + struct.pack('<ii', colidx, track) \
          + b'\x00' + struct.pack('<i', len(evs))
        for row, val in sorted(evs): r += struct.pack('<ii', row * 240, val)
        return r + struct.pack('<iiii', 4, 4, 4, 4)
    out = b''.join(rec(c, 0, []) for c in range(base))        # non-track columns
    for t in range(trackcount):
        for pc in track_param_cols:
            out += rec(pc, t, events.get((t, pc), []))
    return b'\xff\x01' + struct.pack('<I', 2 + 4 + len(body + out)) + body + out

def to_data_element(blob):                                    # editor <Data> = raw blob, b64
    return base64.b64encode(blob).decode('ascii')
```

```python
def decode_blob(blob):
    rd_i = lambda o: (struct.unpack_from('<i', blob, o)[0], o + 4)
    def rd_s(o):
        s = o
        while blob[o]: o += 1
        return blob[s:o].decode('latin1'), o + 1
    o = 6                                   # skip magic + length field
    npat, o = rd_i(o); pats = []
    for _ in range(npat):
        name, o = rd_s(o); f1, o = rd_i(o); f2, o = rd_i(o); rec, o = rd_i(o)
        cols = []
        for _ in range(rec):
            cn, o = rd_s(o); col, o = rd_i(o); track, o = rd_i(o); o += 1; evc, o = rd_i(o)
            ev = []
            for _ in range(evc):
                pos, o = rd_i(o); val, o = rd_i(o); ev.append((pos // 240, val))
            o += 16                          # trailer
            if ev: cols.append((col, track, ev))
        pats.append((name, rec, cols))
    assert o == len(blob)                    # clean parse ends exactly at EOF
    return pats
```

---

## 6. Assembling a song from real machine blocks (method that works)

**Lesson learned the hard way:** hand-fabricated machine blocks (empty/guessed
ParameterGroups, hand-cloned editors) load but are **non-functional** — the
generators show up yet you cannot add tracks/patterns/notes to them. The only
reliable approach is to **reuse `<Machine>` blocks that ReBuzz itself wrote**
(create the machine in ReBuzz, save, and splice the exact block).

Workflow used for Limani:

1. Keep a real saved song as the **skeleton** (correct order, BOM, namespaces).
   Here: `synthref.bmxml` (Master + Bass/Lead/Pad/Comp + their editors).
2. Splice in real machine blocks from another saved song (the drum kit from
   `DrumTest.bmxml`) as **literal text**.
3. **Renumber** colliding editor names. Both source songs used
   `_x0001_pe1…pe5`. Final scheme: `pe1`=Master, `pe2…pe5`=synth editors,
   `pe6…pe9`=drum editors. For each moved drum block update its editor's
   `<Name>`, the generator's `<EditorMachine>`, and its `<MachineConnection>`
   `<Source>`.
4. For every generator, set PatternCore to one empty `00` pattern at the full
   length (§3), and replace its editor's `<Data>` with a freshly generated blob
   (§5). Give the **Master** a `00` pattern too and write BPM/TPB into its
   `pe1` editor blob (§8).
5. Rebuild `<MachineConnections>` (every generator **and** every editor → Master)
   and `<Sequences>` (one per generator **plus one for `Master`** carrying the
   tempo pattern, §8).
6. Assign distinct `<X>`/`<Y>` to every visible generator (§2.4), clear all
   Track `Note` stored values to `0` (§2.5), set `LoopEnd`/`SongEnd` to the song
   length, and write with the BOM.

### 6.1 Nesting-aware block extraction (gotcha)

A **populated** PatternCore column contains nested `<Machine>NAME</Machine>`
tags (the column's owning machine). A naive `<Machine>.*?</Machine>` regex
therefore splits in the wrong place. Extract top-level `<Machine>` blocks by
**tracking `<Machine>` / `</Machine>` depth**:

```python
import re
def machine_blocks(raw):
    seg = raw[raw.index('<Machines>')+10 : raw.index('</Machines>')]
    out, depth, start = [], 0, None
    for m in re.finditer(r'</?Machine>', seg):     # matches <Machine> & </Machine>,
        if m.group(0) == '<Machine>':              #   NOT <EditorMachine>/<MachineConnections>
            if depth == 0: start = m.start()
            depth += 1
        else:
            depth -= 1
            if depth == 0: out.append(seg[start:m.end()])
    return out
```

(Songs whose columns are empty don't hit this — but always use the
depth-aware version to be safe.)

### 6.2 Connection & sequence templates

```xml
<MachineConnection>
  <Source>NAME</Source><Destination>Master</Destination>
  <Amp>16384</Amp><Pan>16384</Pan>
  <SourceChannel>0</SourceChannel><DestinationChannel>0</DestinationChannel>
</MachineConnection>

<Sequence>
  <Events>
    <Event><Time>0</Time><Type>PlayPattern</Type>
           <Span>256</Span><Pattern>00</Pattern></Event>
  </Events>
  <Machine>NAME</Machine><IsDisabled>false</IsDisabled>
</Sequence>
```

`Time` = start row, `Span` = pattern length in rows (= PatternCore `<Length>`),
`Pattern` = pattern name. Tile short loops by emitting one event per
repetition (e.g. `Time` 32, 48, 64 …, `Span` 16) **or** use one long pattern
with a single event (`Time` 0, `Span` 256) — both work.

---

## 7. Machine roster / verified Library names

From the user's `index.txt` (the machine browser), confirmed correct as `Library`
strings:

- **Generators:** Pedal Tracker, Pedal FM, Pedal Juno106, Pedal SH101,
  Pedal invFFT, Pedal M1, Pedal Plaits, Pedal Faze-R, Pedal Add-R.
- **Control machines:** Pedal Muter, Pedal Profiler, Pedal Profiler2,
  Pedal Chord, BTDSys PeerCtrl, Pedal Follower, Pedal LFO, Pedal Presetter,
  Pedal ReTrig.
- **Effects:** Pedal Comp, Pedal Chorus, Pedal Gain, Pedal Gain Multi,
  Pedal Hallverb, Pedal Plate, Pedal Dly PCM41, Pedal Filter, Pedal LFmono,
  Pedal Do Nuttin', Pedal EQ, Pedal FFT, Pedal Gate, Pedal Limit, Pedal Patch,
  Pedal ConVerb, Pedal Resonator, Pedal Z-Plane, Pedal Folder, Pedal Shaper,
  Pedal HDist, Pedal MComp, Pedal S950.

> **Pedal Chord (control machine) — works in-song; see §12.** An early attempt
> threw a NullReferenceException at load and Pedal Chord was wrongly dropped from
> the design. **Root cause was a *fabricated* Pedal Chord block** (the §6 cardinal
> rule), not the machine. Using a **real** saved Pedal Chord block, it drives a
> target generator with live chords/arps and loads cleanly. Limani now uses four
> of them (one per synth). Full recipe in **§12**.

---

## 8. Song tempo (BPM / TPB) — set via a Master-track pattern

**Tempo does not come from the Master's parameters.** The BMXML loader reads
BPM/TPB from the *in-session* Master prototype and never copies the saved
Master parameter values into it (`BMXMLFile.cs` ~242–245 reads
`masterGlobals.Parameters[1/2].GetValue(0)`, with **no** `CopyParameters` for
the Master — unlike every other machine). So editing the file's Master `<Value>`
for BPM/TPB has **no effect** on load. (The binary `.bmx` loader *does* restore
them, because it `SetValue`s all params from the file before reading — a
load-path asymmetry, not a format difference.)

**Tempo travels as events in a pattern on the Master's own track.** This is the
working mechanism, verified against a real saved file:

- The Master has its own `Modern Pattern Editor` (`_x0001_pe1`) and a PatternCore
  pattern (e.g. `00`). Its editor blob holds that pattern with **3 column-records
  = the Master's Global params**, in order: **col 0 = Volume, col 1 = BPM,
  col 2 = TPB**.
- Put an event at **row 0** in the columns you want to set, with the **literal**
  value (not note-encoded): BPM `60` in col 1, TPB `4` in col 2. Leave col 0
  (Volume) empty to keep master volume.
- Add a `<Sequence>` with `<Machine>Master</Machine>` playing that pattern
  (`Time 0`, `Span` = pattern length). The Master must be sequenced for the
  events to fire.

Concrete generator (column-indexed events; reuses the §5 layout):

```python
def build_master_tempo_blob(bpm, tpb):
    # Master Global columns: 0=Volume, 1=BPM, 2=TPB ; literal values at row 0
    return build_blob_cols('Master', '00', 3, {1: [(0, bpm)], 2: [(0, tpb)]})
# build_blob_cols is build_blob (§5) generalized to take {col: [(row,value)]}
```

Notes & caveats:

- **Values are literal.** col 1 value `60` = 60 BPM; col 2 value `8` = TPB 8.
  Confirmed against a reference saved at BPM 60 / TPB 8.
- It is good practice to also set the Master Global `<Value>` for BPM/TPB to the
  same numbers (harmless metadata; honored by the `.bmx` loader or a fixed XML
  loader), but the **pattern events are what actually take effect**.
- The cached 1817–1827 source had the Master BPM/TPB `SubscribeEvents`
  **commented out** (`ReBuzzCore.CreateMaster` ~1815–1816) and the play engine
  derives tempo from internal `bpm`/`tpb` fields — which led to an earlier
  (wrong) conclusion that a Master pattern couldn't drive tempo. The **shipping
  Build 1827 applies it**; trust the live artifact over the cached source.
- Timing maths: at 60 BPM / TPB 4 the row rate is 4 rows/s, so 16 bars =
  256 rows ≈ 64 s. The top-level `<Speed>` element is unrelated to tempo
  (a legacy int, `0` in working files).

---

## 9. Delivery (avoiding file corruption)

Plain browser downloads were observed to **corrupt** the `.bmxml` (BOM /
encoding mangling). Reliable method: ship a small PowerShell script that holds
the file as an embedded **gzip + base64** payload and writes exact bytes:

```powershell
$bytes = [Convert]::FromBase64String(($b64 -replace '\s',''))
$ms = New-Object System.IO.MemoryStream(,$bytes)
$gz = New-Object System.IO.Compression.GzipStream($ms,[IO.Compression.CompressionMode]::Decompress)
$out = New-Object System.IO.MemoryStream; $gz.CopyTo($out); $gz.Close()
[IO.File]::WriteAllBytes((Join-Path $env:USERPROFILE 'Downloads\Song.bmxml'), $out.ToArray())
```

Run with: `Get-Content .\Write-Song.ps1 -Raw | Invoke-Expression`. Put the
base64 in a single-quoted here-string (`@' … '@`) and strip whitespace before
decoding. Always emit with the UTF-8 BOM (§1).

---

## 10. Appendix — "Limani" musical design

- **Scale:** D Hijaz / Phrygian-dominant — D E♭ F♯ G A B♭ C
  (noteIdx D=2, E♭=3, F♯=6, G=7, A=9, B♭=10, C=0).
- **Tempo / form:** 60 BPM, TPB 4 (16 rows/bar); 16 bars × 16 rows = 256 rows
  ≈ 64 s. Per-bar root degrees
  `D D E♭ D G G A D  D E♭ B♭ A G E♭ A D`.
- **Harmony (diatonic D-Hijaz triad per root degree)** — used by every control
  machine so the chords and arps share one progression:
  D→Major, E♭→Major, G→Minor, A→Diminished, B♭→Augmented. As Pedal Chord
  chord-indices (§12.3): D/E♭ = `0`, G = `1`, A = `5`, B♭ = `6`.
- **Arrangement (build-up).** Drums play **directly** (their own editor blobs);
  the four synths are each **driven by a Pedal Chord** (§12) and their own
  patterns are empty:
  - *Bass* (SH101, oct 2) ← **BassArp**: octave-bass *arp* (Mode Up, chord type
    `50` = root+octave, Octaves 1, **Speed 8** so it bounces root@row0 /
    octave@row8). Plays throughout.
  - *Pad* (invFFT, oct 4) ← **PadChord**: block *chord* (Mode 0) on row 0 of
    every bar. Plays throughout. (Intro bars 1–2 are bass + pad alone.)
  - *Kick* (Plaits): rows 0,8 — from bar 3.
  - *HatClosed* (Plaits): straight 8ths (rows 0,2,…,14) — from bar 3.
  - *HatOpen* (Plaits): row 14 accent — from bar 3.
  - *Snare* (Plaits): rows 4,12 — from bar 5.
  - *Lead* (Faze-R, oct 4) ← **LeadArp**: *arp* Up (Mode 1, **Speed 2,
    Octaves 2**, Arp-Reset@row0), the progression arpeggiated, bars 5–16.
  - *Comp* (Juno106, oct 3) ← **CompChord**: block *chord* stabs on rows 4 & 12,
    bars 9–16.
- All drum hits trigger note value **65** (C-4); the kit's timbres come from each
  Plaits machine's own parameters (only the kick was tuned in the source song —
  snare/hats were default Plaits and may need parameter tweaks).
- **Machine inventory (26):** Master + its editor `pe1`; 4 synths + their (now
  empty) editors `pe2…pe5`; 4 Plaits drums + editors `pe6…pe9`; 4 Pedal Chords
  (`PadChord`,`CompChord`,`LeadArp`,`BassArp`) + their editors `pe10…pe13`. The
  Pedal Chord *generators* are **not** audio-connected; only their editors → Master
  (§12.2).
- **Earlier (direct-melody) version** — still valid, kept for reference: each
  synth carried its own notes (Bass root on rows 0 & 8; Pad root on row 0; Lead a
  composed D-Hijaz phrase leaning on F♯→G and B♭→A with an E♭→F♯ augmented 2nd,
  resolving to D4 in bar 16; Comp root on rows 4,12). The conversion to §12
  replaced these with control machines but changed none of the format facts.

---

## 11. Error chronology (all resolved) — quick index

1. Root must be `<ReBuzzSong>` (the `[XmlRoot]` alias), not `<BMXMLSong>`.
2. Build 1827 needs the UTF-8 BOM + correct top-level/Machine/Parameter order.
3. Pedal Chord as a control machine ⇒ NRE on load ⇒ *was* dropped. **Resolved:**
   the NRE came from a **fabricated** Pedal Chord block, not the machine (same
   root cause as #5). With a real saved block it loads and drives a target
   cleanly — now the basis of the §12 control-machine layer.
4. Patterns silent ⇒ PatternCore columns are ignored; editor machines + MPE
   blobs are authoritative.
5. Synths "not editable / can't add tracks or patterns" ⇒ root cause was
   **fabricated** machine blocks. Fix: reuse ReBuzz-generated blocks verbatim
   (§6). Confirmed editable once real SH101/Faze-R/invFFT/Juno106 blocks were
   spliced in.
6. MPE blob length header ("mystery bytes") ⇒ it's the **uint32 total blob
   length** (§4.1); cracking it enabled byte-exact blob generation.
7. Synths droned the instant the song loaded ⇒ a non-zero **Track `Note`**
   stored value is a load-time held note; clear it to `0` (§2.5).
8. Spliced-in machines all stacked on Master ⇒ they were saved at `X=Y=0`;
   assign distinct positions (§2.4).
9. BPM/TPB wouldn't load from the Master parameters ⇒ the BMXML loader ignores
   them; tempo must be a **pattern on the Master track** (col 1 = BPM,
   col 2 = TPB, literal values, row 0) with the Master sequenced (§8).
10. Multi-track (6-track) synths ⇒ set `<TrackCount>` and repeat each track
    column per track; the int32 after colIdx that looked like `reserved 0` is the
    **track index** (§4.6). SH101 is monophonic and won't honor it.
11. Four Pedal Chord control machines loaded, played, and showed in the sequencer
    but were **invisible in the machine view** ⇒ they were positioned at `X=1.6`,
    off the right edge of the drawn canvas; moving them inside the populated
    `X`/`Y` range fixed it (§2.4).

---

## 12. Driving a generator with a control machine (Pedal Chord) — chords & arps

A **control machine** writes notes onto a *target* generator at playback time,
instead of the target carrying its own notes. **Pedal Chord** (Library
`Pedal Chord`) is the worked example: it reads a **root note + chord type** from
its own pattern and plays either a **block chord** or an **arpeggio** on the
target. Verified working in Limani against two real reference songs —
`chordref.bmxml` (chord mode → 6-track Juno106) and `arpref.bmxml` (arp mode).

> The Pedal Chord *machine internals* (timing, swing, build/deploy) are a
> separate concern documented in `ReBuzz_ManagedMachine_Notes_PedalChord.md`;
> that file's **§5 parameter table is the authoritative column order** reused
> below. This section is purely about **using it inside a `.bmxml`**.

### 12.1 Topology (what connects to what)

```
[Pedal Chord gen]  --(NO audio connection)              its editor (peN) --> Master
        |  reads its own pattern (root + chord type + mode/speed/…)
        |  writes notes live onto -->  [target generator] --> Master (normal audio)
        |                                     ^ target's OWN pattern is EMPTY
        + is itself SEQUENCED (Time 0, Span = song length, Pattern 00)
```

Key, non-obvious points (all verified):

- The Pedal Chord is `Type` **Generator** in the file, but it has **no
  `<MachineConnection>`** of its own — it makes no audio. **Only its editor**
  (`_x0001_peN`) connects to Master, exactly like any editor (§3.1). Adding an
  audio connection for the generator is wrong.
- It **is sequenced** like any generator (one `<Sequence>` with
  `<Machine>PedalChordName</Machine>`), so its pattern actually plays.
- The **target generator is unchanged** structurally — still connected to Master,
  still sequenced — but its **own editor pattern is empty** (no events); the
  control machine injects the notes. Build the empty target pattern with
  `build_blob_mt(target,'00',tcols,tracks,{})` (full columns, zero events).
- One Pedal Chord drives **one** target (single-voice since v1.5). For N driven
  voices, use **N** Pedal Chords (Limani uses 4).

### 12.2 The machine block — reuse a real one, then retarget

Per §6's cardinal rule, **do not fabricate** the Pedal Chord block (that is what
caused the historical NRE, §11 #3). Take a real saved Pedal Chord `<Machine>`
block and change four things per instance:

1. `<Name>` → unique name (e.g. `LeadArp`).
2. `<EditorMachine>` → its editor `_x0001_peN`.
3. `<Data>` → a fresh **state blob** naming the target (§12.3).
4. PatternCore → one empty `00` pattern at song length (§3); position `<X>/<Y>`
   inside the visible canvas (§2.4).

Its PatternCore `<Columns>` are empty (`<Columns />`), so the simple
`<Patterns>…</Patterns>` and `<Data>…</Data>` regex replacements are safe (no
nested-`<Machine>` issue, §6.1).

### 12.3 State blob — how the target is named

The Pedal Chord's managed `<Data>` is the standard wrapped state (§2.3:
`[0x02][int32 size][XML]`), where the XML is a `PedalChordState` carrying the
**target machine name** and **base track**:

```xml
<?xml version="1.0" encoding="utf-8"?>
<PedalChordState xmlns:xsi="..." xmlns:xsd="...">
  <TargetMachine>Pedal Juno106</TargetMachine>   <!-- the target's Name -->
  <BaseTrack>0</BaseTrack>
</PedalChordState>
```

The XML payload begins with a **UTF-8 BOM** (`EF BB BF`) inside the size-counted
bytes. `TargetMachine` is matched by **Name** (§2.1). Generator:

```python
import struct
def build_pedalchord_state(target, basetrack=0):
    xml = ('<?xml version="1.0" encoding="utf-8"?>\r\n<PedalChordState '
           'xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" '
           'xmlns:xsd="http://www.w3.org/2001/XMLSchema">\r\n'
           '  <TargetMachine>%s</TargetMachine>\r\n  <BaseTrack>%d</BaseTrack>\r\n'
           '</PedalChordState>') % (target, basetrack)
    body = b'\xef\xbb\xbf' + xml.encode('utf-8')
    return bytes([2]) + struct.pack('<i', len(body)) + body   # 0x02 ver, size, XML
```

### 12.4 The pattern — 14 columns, root + chord type + mode

The Pedal Chord is **single-track itself**; its editor pattern has **14
column-records = its 14 Track-group parameters**, in this order (from the dev
addendum §5):

| col | parameter | use when authoring |
|----|-----------|--------------------|
| 0  | **Note**     | **root note** (Core/§3.2 encoding), at the rows the harmony changes |
| 1  | Velocity     | optional |
| 2  | **Chord**    | **chord-type index 0–50** (table below) |
| 3  | **Mode**     | **0 = block chord; 1 Up, 2 Down, 3 Up+Down, 4 Down+Up, 5 Random** (arps) |
| 4  | Speed        | arp ticks per step (whole pattern ticks) |
| 5  | Length       | note length in ticks; 0 = sustain to next |
| 6  | Octaves      | arp octave span 1–4 |
| 7  | Step         | chord tones advanced per arp step |
| 8  | Oct Walk     | 0 off / 1 up / 2 ping-pong |
| 9  | Swing        | 0–100 |
| 10 | Swing On     | 0/1 |
| 11 | Humanize     | 0–100 |
| 12 | Hum. Vel     | 0–100 |
| 13 | **Arp Reset**| write **1 at row 0** to restart the arp deterministically at song start |

Authoring rules that matter:

- Put **root** (col 0) + **chord type** (col 2) at the row of each harmonic
  change (e.g. one per bar at `row = bar*16`). The machine builds the chord
  tones from root+type internally — **you do not compute chord notes**; you only
  give the root (in the §3.2 note encoding, at the octave you want) and the type
  index.
- **Block chord:** Mode `0` (the default). Only cols 0 & 2 are needed.
- **Arpeggio:** set Mode/Speed/Octaves (and Arp Reset) as **events at row 0** of
  the pattern — this is how `arpref.bmxml` stores them and it is what drives
  playback. (The machine's stored *parameter* values can also hold them, but the
  row-0 pattern events are sufficient and what we use.)
- Build the pattern with the column-indexed generator (`build_blob_cols`, §8 /
  §5): `build_blob_cols(pedalchord_name, '00', 14, {0:[...roots...], 2:[...types...], 3:[(0,mode)], ...})`.
  Note the blob's **column name = the Pedal Chord generator's own Name**, not the
  editor's (§4.3).

**Chord-type index — common values** (full 0–50 list lives in `PedalChord.cs`
`Intervals[]` and the dev addendum):
`0`=Major `{0,4,7}` · `1`=Minor `{0,3,7}` · `2`=Dom7 · `3`=Min7 · `4`=Maj7 ·
`5`=Diminished `{0,3,6}` · `6`=Augmented `{0,4,8}` · `7`=Sus4 · `8`=Sus2 ·
`48`=Power `{0,7}` (root+5th) · `49`=Dim7 · `50`=Oct `{0,12}` (root+octave —
handy for octave-bass arps).

### 12.5 Polyphony — chord vs arp need different target tracks

- **Chord mode** writes the chord's notes across **consecutive target tracks**
  (BaseTrack, BaseTrack+1, …), one note per track, simultaneously. So a triad
  needs the target to have **≥3 tracks** — raise the target's `<TrackCount>`
  (§2.6/§4.6). Limani's chord targets (Pad, Comp) are 6-track. A target with
  too few tracks simply drops the extra chord tones.
- **Arp mode** plays **one note at a time on the single BaseTrack**, so the
  target needs **only 1 track** — a monophonic generator (e.g. SH101) is a fine
  arp target. (Limani's BassArp drives the monophonic SH101; LeadArp drives the
  6-track Faze-R but uses only track 0.)

### 12.6 Silence on load

The Pedal Chord's own col-0 **Note** is a Track-group `Note` parameter, so the
§2.5 held-note clear (`<Type>Note</Type>` current value → 0) applies to it too
and stops it injecting a root the instant the file loads. Run that clear over
the whole document as usual; it covers synths and Pedal Chords alike.

### 12.7 Assembly delta (added to the §6 workflow)

For each driven voice: (a) splice a real Pedal Chord block, retargeted (§12.2–3);
(b) add its editor `_x0001_peN` with the §12.4 pattern blob; (c) **empty** the
target's own editor blob; (d) add **one connection** `peN → Master` (the chord
*generator* gets none); (e) add **one sequence** for the Pedal Chord generator;
(f) position the generator on-canvas (§2.4). Keep the target's own connection and
sequence as-is.

### 12.8 Reference files

`chordref.bmxml` — chord mode (`PdlChrd` → 6-track Juno106; roots col 0, types
col 2, state targeting). `arpref.bmxml` — arp mode (Mode 1 Up, Speed 2,
Octaves 2 at row 0; TPB 8). Both round-tripped; the generators in §12.3/§12.4
reproduce their blobs byte-for-byte.

---

## 13. Generalized recipe — authoring a new song from a spec

The standing goal: turn a request like *"a ~1-minute piece in style X, with
instruments A/B/C, at tempo T"* into a loadable `.bmxml`. Everything above is the
toolkit; this is the order of operations. (Limani is one instantiation of it.)

**Inputs to pin down first:** genre/mood, key + scale, tempo (BPM, TPB),
time-signature & length (→ rows), instrument roles, arrangement/build-up.

1. **Scale & harmony → numbers.** Write the scale as a `noteIdx` set
   (C=0…B=11). Choose per-section/per-bar **root degrees**. For chordal parts,
   pick the **diatonic triad quality** on each degree (built from scale tones)
   and map it to a Pedal Chord chord-index (§12.3): major→`0`, minor→`1`,
   diminished→`5`, augmented→`6`, etc. (Limani's D-Hijaz gave D/E♭=0, G=1,
   A=5, B♭=6.) Note value at a chosen octave = `octave*16 + noteIdx + 1` (§3.2).

2. **Pick machines (the roster).** From §7 / `ReBuzz_ManagedMachine_Notes_Roster.md`
   choose a generator per role (drums, bass, pad, lead, comp, …). For each,
   record **polyphony** (mono vs multi-track) and whether it will be played
   **directly** or **driven by a control machine** (§12). Crucially, obtain a
   **real saved `<Machine>` block** for every machine type you use (§6 cardinal
   rule) — save it once in ReBuzz if you don't already have one.

3. **Tempo & length.** Rows/bar = `TPB × beats-per-bar` (4/4 + TPB 4 → 16).
   Total rows = `bars × rows-per-bar`. Set the **Master-track tempo pattern**
   (§8: col 1 = BPM, col 2 = TPB, literal, row 0), the PatternCore `<Length>`,
   `LoopEnd`/`SongEnd`, and every sequence `Span` to the total rows.

4. **Write the parts.**
   - *Direct part:* notes go in the generator's editor blob (§4–§5); use
     `build_blob_mt` with the per-track events for polyphony (§4.6).
   - *Control-driven part (chords/arps):* one Pedal Chord per voice (§12) —
     block chord (Mode 0) or arp (Mode 1–5); empty the target's own pattern;
     give chord targets enough tracks (§12.5).
   - Drums: trigger note (Plaits uses `65` = C-4); timbre from each drum
     machine's own parameters.

5. **Assemble** (§6 + §12.7). Real skeleton song → splice real blocks → renumber
   editors so names don't collide (`pe1` = Master's editor, then one `_x0001_peN`
   per generator editor, then one per control-machine editor) → PatternCore = one
   empty `00` per generator → fill editor blobs → connections
   (**every generator and every editor → Master; control-machine *generators*
   get none**) → sequences (every generator + Master + each control machine) →
   distinct on-canvas positions (§2.4) → clear held notes (§2.5) → set length
   tails → write **with BOM** (§1).

6. **Deliver** (§9): emit the PowerShell **gzip+base64** writer, with a
   decode→compare **round-trip check** before shipping.

7. **Self-verify before delivery (do this every build).** Re-parse the XML;
   decode **every** editor blob and assert `offset == len(blob)`; confirm:
   machine count; each control machine's state targets the intended generator;
   driven targets have **no** note events; chord targets have enough tracks;
   connections include each editor and **exclude** control-machine generators;
   one sequence per generator + Master + each control machine; all positions in
   range; no non-zero held `Note`. A clean parse + clean decode + these checks
   is the bar for "ready to load."

**What's reusable vs. per-song.** Reusable: the format facts (§§1–6, 8),
the blob generators (§5, §8, §12.3), the assembly + verify pipeline (§6, §13.5–7),
the machine roster (§7). Per-song: scale/roots/qualities, machine selection,
tempo/length, and the part data. Adding a **new machine type** mainly means
saving a real block of it once and recording its **note-column index / track
params** (§4.4) in the roster.
