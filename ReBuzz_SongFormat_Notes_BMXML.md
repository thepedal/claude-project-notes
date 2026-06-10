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

**Control-machine layer (§12).** The earlier "Pedal Chord throws an NRE,
dropped" conclusion was **wrong** — the NRE came from a *fabricated* machine
block (§6's cardinal rule), not the machine. §12 is now the **control-machine
family**: the shared wiring pattern, then **Pedal Chord** (notes — chords/arps,
incl. the row-0 **note-off entry rule** §12.9 for voices that enter after bar 1)
and **Pedal Presetter** (in-pattern **preset automation** §12.10, incl. the
Build-1827 **one-preset-fire-per-row** stagger rule). All verified byte-exact
against real reference songs (`chordref`, `arpref`, `presetter_ref`) and working
in Limani. **§13** generalises the whole pipeline into a **spec → song** recipe —
the standing goal of being able to ask for "a piece in style X, instruments
Y/Z, tempo T."

This file documents the **XML song format** and the **Modern Pattern Editor
(MPE) note-data blob** — i.e. how a `.bmxml` song is laid out on disk and how
playable note data is actually stored. It is a different concern from the
`ReBuzz_ManagedMachine_Notes_*` files (which cover writing the machines
themselves). Cross-references to those use `Core §N` etc.

Worked example throughout: the song **"Limani"** (**28 machines**: a 4-piece
Plaits drum kit played directly, + bass/lead/pad/comp synths each **driven by
its own Pedal Chord** control machine, + a **Pedal Presetter** setting timbres,
~64 s, D Hijaz). Its build script is the canonical implementation of everything
here. (The original Limani drove the synths with hand-written melodies; it was
later converted to the control-machine layer of §12 — both arrangements are
valid and the format facts are identical.)

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

The position element appears once per machine, right after the machine `<Type>`:
`<Type>Generator</Type><X>-1.2</X><Y>0.4</Y>` — target it there when editing.
**`Effect` machines (e.g. Pedal Gain Multi, §15) carry the same `<X>`/`<Y>`** in
the same place; `set_position` rewrites both `Generator` and `Effect` (an early
version matched only `Generator`, so spliced effects silently kept the template's
coordinate and stacked).

> **Guard it in the build.** `assert_no_overlap(xml, eps=0.05)` scans every
> visible machine (all but `Modern Pattern Editor`, which legitimately sit at
> `0,0`) and raises if any two are within `eps`, listing the colliding pair and
> coordinate. Call it right before `write_bmxml` so a stacked layout can never
> ship. `machine_positions(xml)` returns the `(name, x, y)` list it checks.

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
    Octaves 2**, Arp-Reset@row0), the progression arpeggiated, bars 5–16. Enters
    via a row-0 note-off (§12.9).
  - *Comp* (Juno106, oct 3) ← **CompChord**: block *chord* stabs on rows 4 & 12,
    bars 9–16. Enters via a row-0 note-off (§12.9).
- All drum hits trigger note value **65** (C-4); the kit's timbres come from each
  Plaits machine's own parameters (only the kick was tuned in the source song —
  snare/hats were default Plaits and may need parameter tweaks).
- **Preset automation:** a **Pedal Presetter** (`Presets`, 3 tracks, editor
  `pe14`) sets the non-bass synth timbres at song start (§12.10), fired on
  **staggered rows 0/1/2**: Pad ← invFFT `46` (Pad - Soft Strings), Lead ←
  Faze-R `58` (Pluck - Reso Tine), Comp ← Juno106 `17` (Classic — Polysynth).
- **Machine inventory (28):** Master + its editor `pe1`; 4 synths + their (now
  empty) editors `pe2…pe5`; 4 Plaits drums + editors `pe6…pe9`; 4 Pedal Chords
  (`PadChord`,`CompChord`,`LeadArp`,`BassArp`) + editors `pe10…pe13`; 1 Pedal
  Presetter (`Presets`) + editor `pe14`. The Pedal Chord and Presetter
  *generators* are **not** audio-connected; only their editors → Master (§12).
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
12. Late-entering chord/arp voices (Lead, Comp) **sounded from tick 0** instead
    of at their bar ⇒ a Pedal Chord with no row-0 root plays a stale/looped root;
    write a row-0 **note-off (255)** to hold it silent until its first root
    (§12.9).
13. A Pedal Presetter firing **several targets on the same row** applied only the
    **last** one ⇒ Build-1827 `parametersChanged` collision + broken pvalues
    fallback; **stagger one preset per row** (§12.10).

---

## 12. Control machines that drive other machines (Pedal Chord, Pedal Presetter)

A **control machine** is a `Type` Generator that produces **no audio of its own**
and instead writes to a *target* machine at playback time, reading what to write
from its own pattern. Two are documented here: **Pedal Chord** (writes notes —
chords/arps, §12.1–§12.9) and **Pedal Presetter** (fires preset changes,
§12.10). They share one wiring pattern, so learn it once:

- The control machine is `Type` **Generator** but has **no `<MachineConnection>`**
  (it makes no audio). **Only its editor** (`_x0001_peN`) connects to Master,
  like any editor (§3.1).
- It **is sequenced** (one `<Sequence>` on its own name) so its pattern plays.
- Its **target generator is otherwise normal** — connected to Master, sequenced —
  but its **own editor pattern is emptied** (the control machine injects live).
  Build the empty target with `build_blob_mt(target,'00',tcols,tracks,{})`.
- **Reuse a real saved `<Machine>` block** for the control machine and retarget
  it (§6 cardinal rule). Fabricating one is what caused the historical Pedal
  Chord NRE (§11 #3).
- The target is named in the control machine's managed **`<Data>` state blob**
  (§2.3 wrapper); each machine has its own little state schema (§12.3, §12.10).
- **Init order is automatic:** control machines tick before audio machines, so a
  row-0 write lands before the target renders that row.

Everything below is about **using these inside a `.bmxml`**; each machine's
internals live in its own `ReBuzz_ManagedMachine_Notes_*` file (whose parameter
tables are the authoritative column order reused here).

### 12.0 Pedal Chord — what it does

**Pedal Chord** (Library `Pedal Chord`) reads a **root note + chord type** from
its own pattern and plays either a **block chord** or an **arpeggio** on the
target. Verified working in Limani against two real reference songs —
`chordref.bmxml` (chord mode → 6-track Juno106) and `arpref.bmxml` (arp mode).
Its internals (timing, swing, build/deploy) are in
`ReBuzz_ManagedMachine_Notes_PedalChord.md`; that file's **§5 parameter table is
the authoritative column order** reused below.

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

### 12.9 Entering on the timeline — the row-0 note-off rule

A Pedal Chord with **no root event at row 0** does **not** stay silent at the
start: at song start it sounds a stale/zero root, and on loop the previous
root carries over across the loop point. So a chord/arp that is meant to enter
*later* (Lead at bar 5, Comp at bar 9 in Limani) will instead sound from tick 0.

**Fix — write a note-off at row 0** of the root column (col 0). The Pedal Chord's
Note parameter documents **value `255` as "off" — it releases the voice**. So:

> **Rule (general):** for every Pedal Chord whose root column has no event at
> row 0, prepend a `(row 0, 255)` event to col 0. It holds the voice silent
> until the first real root, so the machine enters on the timeline.

Both Limani arps that enter late carry `(0,255)` in col 0; the two that play from
bar 1 (Pad, Bass) already have a real root at row 0 and need no note-off. This is
distinct from the §2.5 *held-note* clear (which zeroes the **stored parameter**
so nothing sounds on load) — the note-off is a **sequenced pattern event** that
governs the **looping playback** entry point. Use both.

#### 12.9.1 Releasing the target across silent sections and the loop

The §12.9 note-off keeps a late-entering voice silent *within* the pattern it
opens. It does nothing once the Pedal Chord stops being **sequenced**: in a
sectioned song the chord is only placed during the sections where its target
plays, so when the target falls silent (a following section, or the loop wrap)
nothing tells it to stop, and the held note **drones** over the next section or
the next pass's intro.

Two things are needed at the start of every section where the target should be
silent, because they fix different halves of the problem:

1. **A note-off on the *target's own* pattern**, on every track. This releases
   the voice that is **actually sounding**. It is the only thing that cuts a
   patch with a **long release/sustain** — the control machine cannot shorten an
   envelope it has already triggered.
2. **A stop pattern on the *Pedal Chord*** — a single col-0 note-off, one row
   long, sequenced at the same point. This halts any **further arpeggiation
   triggers** so the arp doesn't keep firing into a section it shouldn't.

> **Rule:** at the start of every silent section, give the target a row-0
> note-off on every track **and** sequence a one-row stop pattern on its Pedal
> Chord. A silent **first** section therefore gets both at song tick 0 — that is
> what releases the target on the **loop**. A target that re-articulates at
> tick 0 (Pad, Bass — their chord fires a root at bar 0) needs neither; **drums
> are exempt** (one-shot Plaits can't drone).

> **Refinement — active but late at tick 0.** "Re-articulates at tick 0" means an
> actual note **on row 0**, not merely being active in the first section. A voice
> whose first section opens with a *mid-bar* strike (e.g. a comp on `..x...x.`,
> first hit on beat 2) is **not** triggered at tick 0, so it still needs the row-0
> **target note-off** to release whatever it carried across the loop wrap — even
> though it has no silent section. (Its Pedal Chord's own first pattern already
> opens with the §12.9 entry note-off, so no extra stop pattern is required there.)
> The compiler adds the row-0 target release whenever a voice is *not* articulated
> on row 0, whether because it's silent or because its first hit is later.

Mechanism 2 alone leaves a long tail ringing (the arp stops, but the last note
decays for its full release); mechanism 1 alone re-triggers cleanly but lets the
arp keep playing if the chord is still sequenced. Together they cover section
entry, mid-song exit, and the loop wrap, and they never cut a voice that stays
active across a boundary (a bridge running straight into a chorus), because that
boundary is not a silent section.

```python
silent = [start*bar for name, start in arrangement if not plays_here(name)]
# (1) target: note-off on every track at each silent-section start row
synth_ev = {(t, notecol): [(r, NOTE_OFF) for r in silent] for t in range(ntracks)}
# (2) chord: a one-row stop pattern, placed at each silent-section start
STOP = '_stop'; pats.append((STOP, 1)); blobpats.append((STOP, ncols, {0: [(0, NOTE_OFF)]}))
chord_placements = [(start*bar, sec_len, name) if plays_here(name)
                    else (start*bar, 1, STOP) for name, start in arrangement]
```

This complements §12.9 (keep all three): §12.9 cleans section entries *within* a
pass; the target note-off + chord stop guarantee a clean exit and loop. Validator
check 8 (§16) enforces both at tick 0.


### 12.10 Pedal Presetter — preset automation

**Pedal Presetter** (Library `Pedal Presetter`) is a **multi-track** control
machine (up to **16 tracks**): each track is bound to one target machine, and a
stateless per-track **`Preset`** index fires a preset change on that target when
a value lands on a row. It's how you switch a synth's timbre from the timeline —
ReBuzz otherwise has no in-pattern preset automation. Verified byte-exact against
`presetter_ref.bmxml` and working in Limani (sets Lead/Pad/Comp timbres).
Internals: `ReBuzz_ManagedMachine_Notes_PedalPresetter.md` (and the preset-API
consumer side in `ReBuzz_ManagedMachine_Notes_Presets.md`).

It follows the shared control-machine pattern (§12 intro): generator unconnected,
editor → Master, sequenced; targets normal but their own patterns emptied.

**State blob — the per-track targets.** The managed `<Data>` (§2.3 wrapper) holds
a `PresetterState` whose `<Targets>` is **always exactly 16 `<string>` entries**,
track-indexed: a machine **Name** where assigned, `<string xsi:nil="true" />`
where not. (That inner `xsi:nil` is fine — the state blob is its own XML document,
separate from the song body, §1.1.)

```python
def build_presetter_state(targets, max_tracks=16):           # verified byte-exact
    t=(list(targets)+[None]*max_tracks)[:max_tracks]
    lines=['<?xml version="1.0" encoding="utf-8"?>',
           '<PresetterState xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" '
           'xmlns:xsd="http://www.w3.org/2001/XMLSchema">','  <Targets>']
    for nm in t:
        lines.append('    <string>%s</string>'%nm if nm is not None
                     else '    <string xsi:nil="true" />')
    lines+=['  </Targets>','</PresetterState>']
    body=b'\xef\xbb\xbf'+'\r\n'.join(lines).encode('utf-8')
    return bytes([2])+struct.pack('<i',len(body))+body        # 0x02 ver, size, XML
```

**Pattern — one `Preset` column per track.** Set `<TrackCount>` (Track group) to
the number of targets. The only pattern parameter is `Preset` (Byte, range
**0–253**, NoValue 255), so there are **no non-track columns** — `base = 0` and
the Preset column repeats at **colIdx 0** per track, track-major (§4.6). Build it
with the multi-track generator: `build_blob_mt('Presets','00',[0],N,events)` with
`events[(track,0)] = [(row, preset_index)]`.

- **Preset index = 0-based position in the target's bank** as ReBuzz loads it
  (a `.prs.xml` `PresetDictionary`; the `Item Key`s in file order). Calibrated:
  Juno106 index `1` = "Bass — Funky". So indices are **bank-order dependent** —
  the target's bank must be present in ReBuzz or the index falls **out of range**
  and fires the target's **default** preset (a reset), not nothing.
- **Empty cell does nothing** (the param is `IsStateless`; each row is a one-shot
  event, not held). Practical behaviour is "fire-and-hold" — the applied preset
  persists until something else changes the target.
- Neutralise load-time firing by setting the **stored** `Preset` value to NoValue
  `255` for each track (analogous to §2.5), so only the sequenced events fire.

> **Rule (Build 1827) — one preset fire per row.** Firing **multiple tracks on
> the same row** hits a `parametersChanged` collision; the Presetter's
> pvalues-polling fallback for that case is unreliable on 1827 (the Core §42
> `pvalues` field-shape change), so only the **last** track lands. **Stagger
> multi-target changes onto separate rows** (Limani fires Pad/Lead/Comp on rows
> 0/1/2). This is the same class of 1827 reflection breakage noted for Pedal
> Chord's old multi-track path (§7 / PedalChord §9.1).

**Other caveats.** Targets with smoothed parameters (Plaits etc., per Core §32)
**glide** ~30 ms on a change rather than switching instantly. Assignments survive
save/load and target rename; a deleted target leaves a dangling assignment that
silently fires nothing.

**Assembly delta.** Like §12.7: splice a real Presetter block, set `<Name>`,
`<EditorMachine>`, the state `<Data>`, `<TrackCount>`, position; add its editor
`peN` with the staggered Preset pattern; add **one** connection `peN → Master`
(the generator gets none) and **one** sequence on the Presetter. The targets are
ordinary generators (here, already driven by their own Pedal Chords) — the
Presetter only sets their timbre, independent of the notes.

**Reference.** `presetter_ref.bmxml` — 1 track → `Pedal Juno106`, Preset `1` at
row 0. `build_presetter_state` and `build_blob_mt('Presets','00',[0],…)`
reproduce its state and pattern blobs byte-for-byte.

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
     give chord targets enough tracks (§12.5). For any voice that **enters
     after bar 1**, write a row-0 **note-off (255)** so it stays silent until
     its first root (§12.9).
   - *Preset automation (optional):* a Pedal Presetter (§12.10) to set/switch
     target timbres from the timeline — one track per target, preset index =
     bank position, **staggered onto separate rows** (one fire per row, §12.10).
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
   late-entering chord/arp voices carry a row-0 note-off (§12.9); any Presetter
   fires **one preset per row** (no two targets on the same row, §12.10);
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

---

## 14. Multi-pattern arrangement (per-section patterns + sequence tiling)

§3–§8 build each machine with **one** pattern spanning the whole song (one
`<Pattern>`, one editor blob, one `PlayPattern` event at Time 0). That loads and
plays, but a multi-minute song becomes one enormous pattern that's miserable to
navigate. ReBuzz's native model is **named patterns placed on a timeline**, so a
machine can carry several short patterns (per bar, or per section) that the
sequencer drops at different times — and the same pattern can be **reused** at
many times. Verified byte-exact against a real save (`refs/DrumTest.bmxml`),
e.g. `Kick` carrying `KickLoop`+`EndKick` tiled by a 13-event sequence.

### 14.1 The three things that change

1. **PatternCore — multiple `<Pattern>` entries.** One per pattern, each with its
   own `<Name>` and `<Length>`; `<Columns />` stays empty as always (real events
   live in the editor blob; the sequence references patterns by name). See
   `patterns_xml(specs)` where `specs = [(name, length_rows), ...]`.
2. **Editor blob — `patternCount > 1`, positions PATTERN-RELATIVE.** The blob's
   leading int32 pattern count becomes N; each pattern block (name + records) is
   emitted in turn. **Event positions are rows *within that pattern* × 240**, not
   absolute song rows. (In the single-pattern case the two coincide only because
   the pattern length equals the whole song.) An empty pattern may be written as
   0 records (real saves do this for `EndKick`); our builders instead emit full
   columns with zero events, which also loads — both are valid.
3. **Sequence — one `<Event>` per placement.** Each event is `Time` (absolute
   start row), `Type` `PlayPattern`, `Span` (= the pattern's length), `Pattern`
   (name). Repeat a name at several `Time`s to tile/reuse it.

### 14.2 Library functions

The single-pattern functions (§5, §6.2) are now thin wrappers over multi-pattern
cores, so existing single-pattern songs build **byte-identical**:

- `build_blob_cols_multi(mname, [(patname, ncols, colevents), ...])` — Pedal
  Chord / Master style. `build_blob_cols(...)` calls it with one pattern.
- `build_blob_mt_multi(mname, track_param_cols, trackcount, [(patname, events), ...])`
  — multi-track synth style. `build_blob_mt(...)` calls it with one pattern.
- `build_blob(mname, patterns)` was already a list — pass several pattern dicts
  with pattern-relative `events`.
- `patterns_xml(specs)` / `sequence_multi(machine, placements)` /
  `sequences_xml_multi(seqs)` — `pattern_xml`, `sequence`, `sequences_xml` are the
  single-entry wrappers.

`placements = [(time, span, pattern), ...]`; `seqs = [(machine, placements), ...]`.

### 14.3 What sectioning simplifies

- **Reuse.** Identical sections (e.g. two verses, two choruses) are authored once
  and placed at both `Time`s — fewer distinct patterns, edit-once-fix-both.
- **The row-0 note-off (§12.9) mostly disappears.** A voice that enters late is
  simply not sequenced until it enters, so there's no stale looped root to
  silence. Keep a per-pattern note-off only where a pattern *itself* doesn't open
  on a root at relative row 0 (e.g. a comp pattern that first hits on beat 2).
- **Self-contained arp config.** Put the Pedal Chord arp params (Mode, Speed,
  Octaves, Swing, Swing On) at **relative row 0 of every arp pattern**, not once
  at song row 0, so each pattern (and each reuse) is independent of play order.

### 14.4 Arp re-seed timing — Speed is literal (Pedal Chord ≥ v1.6.0)

Boogie/comp patterns restate the chord root at the **top of every bar**, which
re-seeds the arp (NoteOn → `Start()` → `StepArp`) each bar. A pre-v1.6.0 Pedal
Chord lost one step-unit on every re-seed, so the arp ran a tick fast (you had to
set Speed = N+1 to get an N-tick grid). A single held chord hid it — only a
*re-seeding* pattern exposed it. **Pedal Chord v1.6.0 fixes the re-seed seam**, so
arp `Speed` is once again literal (gap == Speed at Swing 0; §10.3 invariants
hold). Generate arp `Speed` values directly; do not add a `+1`. If a future song
plays arps a tick fast, suspect the loaded Pedal Chord DLL version before the
generator.

### 14.5 Proving "same piece" after sectioning

To confirm a sectioned rebuild is musically identical to a flat one: decode every
editor blob → for each sequence placement, offset the pattern's relative rows by
its `Time` → the **absolute** event set. Compare to the flat version: drum hits
must match exactly; Pedal Chord **roots (col 0, value ≠ 255) and chord types
(col 2)** must match exactly. Note-offs (255) and repeated arp-config events are
allowed to differ — they're functionally idempotent. `build_lastcall.py` (the
per-section "Last Call") is the worked example; it reconstructs the flat version's
612 drum hits and 228 chord roots/types exactly.

---

## 15. Effects & multi-input routing (submix bus — Pedal Gain Multi)

§§1–14 only ever wired generators/editors straight to Master. An **effect** has
audio **in and out**, so it sits *between* sources and Master: sources connect
**into** it, and it connects **to** Master. A *multi-input* effect (e.g. **Pedal
Gain Multi**) accepts several sources on separate input channels — a submix bus.
Verified byte-exact against `refs/gainref.bmxml` (4 generators → one Gain → Master).

### 15.1 The machine

- `<Type>` is **`Effect`** (not `Generator`/`Master`) — the first non-generator
  in this project.
- It's a managed machine but its `<Data>` **state blob is empty** (`02 00000000`)
  — routing is *not* stored in the machine; it lives entirely in the connections.
  (Contrast Pedal Chord, whose state names its target.)
- It carries its own pattern editor (an 8-column, event-free `00` pattern) and
  gets a sequence entry like any managed machine. (An effect processes audio
  continuously whenever it has input, so the pattern/sequence is only an
  automation surface — an empty one is fine; we still sequence it Time 0,
  Span = song length for consistency.)
- Params: global `Pan`, `Gain` (Byte, default 100), `Mute`, and **`Solo 1`–`Solo 6`**
  → it supports **up to 6 inputs**. `Amp` is a **per-track array** (one value per
  input).

### 15.2 Multi-in connection encoding

Each source is an ordinary `<MachineConnection>` to the **same `<Destination>`**
with an **incrementing `<DestinationChannel>`**:

```
Kick      -> DrumBus  SourceChannel 0  DestinationChannel 0
Snare     -> DrumBus  SourceChannel 0  DestinationChannel 1
HatClosed -> DrumBus  SourceChannel 0  DestinationChannel 2
HatOpen   -> DrumBus  SourceChannel 0  DestinationChannel 3
DrumBus   -> Master   SourceChannel 0  DestinationChannel 0
```

**Per-input gain is the connection `<Amp>`** (16384 = unity), and ReBuzz mirrors
it into the machine's per-track `Amp` param. Set *both* to be safe:
`set_param(block, 'Amp', 16384)` neutralises the param array;
`connection(src, dst, amp, ...)` sets the wire. `<Pan>` per track defaults to
16384 (centre).

### 15.3 Building one

1. Splice the real Gain block (`refs/gainref.bmxml`): `set_name` it (e.g.
   `DrumBus`), `set_editor` to a fresh `peN`, `set_patterns(PAT)`, `set_position`,
   and `set_param('Amp', 16384)` to start at unity.
2. Build its editor as an empty 8-column blob **with the bus's own name** so the
   embedded machine-name matches: `build_blob_cols('DrumBus', '00', 8, {})`.
3. Reroute: the source **generators** connect to the bus on channels 0..n-1
   (their pattern **editors** stay on Master — only audio reroutes); add one
   `bus -> Master`; add `busEditor -> Master`.
4. Sequence the bus (Time 0, Span = length, `00`).

Library support: `connection(src, dst='Master', amp=16384, pan=16384, src_ch=0,
dst_ch=0)`, `connections_xml_multi(conns)` (each entry a `'src'` string for
→Master or a tuple of `connection` args), and `set_param(block, name, value)`.

### 15.4 Roster note

**Pedal Gain Multi** — `<Type>Effect`, Library `Pedal Gain Multi`, ≤ 6 inputs
(Solo 1–6), per-track `Amp`/`Pan`, empty state blob, 8-column editor. Worked
example: `build_lastcall.py` busses the four drums through one instance at unity
before Master. To set a starting drum balance, give each drum's input connection
(and the matching `Amp` track) a value below 16384.

### 15.5 One gain per source (single Pedal Gain) — for stems

The Gain Multi is right for a *group* submix (the drums), but for the synths it's
often better to give **each source its own single `Pedal Gain`** and let them sum
downstream at the limiter (or Master). Same arithmetic — N gains into channel 0 of
the limiter sum exactly like N channels of one Gain Multi — but each source now has
a **dedicated, soloable output node**, so the human can render true **per-source
stems** and the measurement loop (§18) can read their relative levels. This is how
the DSL routes synths: `Bass → BassGain → Limit`, `Pad → PadGain → Limit`, ….

A single `Pedal Gain` (Library `Pedal Gain`, spliced from `MachineRef`) is a
one-input Gain Multi: same `<Type>Input</Type>` group (one track of `Amp`/`Pan`),
plus globals `Gain`/`Mute`/`Inertia` left at default. Build it like a bus but with
a **3-column** editor (`build_blob_cols(name, '00', 3, {})`) and an Input group
sized to **1** (§20). The source's mix trim goes on that single input `Amp`
(connection + per-track, kept equal); the gain then connects on to `dest` at unity.
An unconnected `Pedal Gain` stores `<Values />` for its Amp, so `set_input_tracks`
must populate the self-closing form (§20).

## 16. Validation (`rebuzz.validate`)

`validate(xml)` runs a set of static checks over an **assembled** song and
returns a `Report` (`.errors`, `.warnings`, `.ok`); `assert_valid(xml)` raises
`ValidationError` on any error and returns the xml for chaining. Both builds call
`assert_valid` just before `write_bmxml`, so a wiring or loop-safety mistake stops
the build instead of surfacing as a glitch (or a hang) inside ReBuzz.

The checks, and the section each enforces:

| # | Check | Severity | Enforces |
|---|-------|----------|----------|
| 1 | Connection `Source`/`Destination` names a real machine | error | §11 |
| 2 | Every sequenced `Pattern` exists in that machine's PatternCore | error | §14 |
| 3 | Each placement's `Span` equals the pattern `Length` | error | §14.2 |
| 4 | On a multi-channel destination, `DestinationChannel`s are unique | error | §15.2 |
| 5 | Stored Track `Note` values are `0` (nothing droning on load) | warn | §2.5 |
| 6 | A visible machine sits inside the machine-view canvas | warn | §10 |
| 7 | No two visible machines share a position | error | §10 |
| 8 | A Pedal-Chord-driven target not re-triggered at tick 0 carries a row-0 note-off on **every** track **and** has its chord sequenced at tick 0 with a stop | error | §12.9.1 |

Check 8 is the loop-drone guard. It decodes each Pedal Chord's `TargetMachine`
(from the state blob) and its tick-0 placement; if no driver fires a root at song
row 0, the target must release every track at row 0 of the pattern it plays at
tick 0. Master is exempt from check 4 (it sums all inputs on channel 0). Drums
are never flagged by check 8 because they are not Pedal-Chord targets. Directly
note-programmed held-note generators (a future hand-written lead) are *not* yet
covered by check 8 — extend `driven` in `validate.py` when one is added.

`decode_blob(raw)` (the FF01 reader used by the validator) is also exported for
inspecting any editor blob: `-> {'mname', 'patterns': {name: {(track,col): [(row,val)]}}}`.

Tests live in `tests/` (`python3 -m pytest tests/ -q`): both songs validate
clean, every check fires on a deliberately broken song, and each build is
byte-deterministic across runs.

## 17. Composition DSL (`rebuzz.theory` + `rebuzz.dsl`)

A higher-level authoring layer that compiles musical objects to a validated
`.bmxml`. It writes nothing new to the format — the compiler assembles the same
rig as `build_lastcall.py` (synthref synths driven by Pedal Chords, Plaits drums,
a Pedal Gain Multi for the drum submix and one Pedal Gain per synth, optional
Presetter + limiter) and emits bytes through
the same builders. So everything in §§1–16 still holds; this section is just the
front door.

**`rebuzz.theory`** — `parse_note('Bb') -> 10`; `Scale(root, mode)` with
`.degree_index(n)` / `.index_of(token)` / `.value(token, octave)`; the verified
`CHORD_CODES` (maj 0, min 1, dom7 2, dim 5, aug 6, oct 50) and `ARP_MODES`
(block 0, up 1, down 2, updown 3, random 5) tables via `chord_code` / `arp_mode`.
Scales include major/minor and the modes plus hijaz, blues, and the pentatonics.

**`rebuzz.dsl`** — the model and compiler:

- **Synth slots** (fixed by synthref): `Bass`=SH101 (mono), `Lead`=Faze-R,
  `Pad`=invFFT, `Comp`=Juno106. **Drum slots** (Plaits): `Kick`, `Snare`,
  `HatClosed`, `HatOpen`. A voice picks a slot by name; unused slots are dropped.
- **Step-strings** — one bar split into `len(s)` equal steps; any char but `.`/` `
  is a hit. Length must divide the bar (`tpb*4` rows). `'x...x...x...x...'` on a
  32-row bar → rows 0, 8, 16, 24.
- **Voices** — `Chords(slot, octave, chord, rhythm='x', sections=None)` (block
  chords struck on the rhythm; `rhythm` may be a per-section dict), `Arp(slot,
  octave, chord, mode, speed, octaves, swing, sections)` (one root per bar + arp
  config), `Drums({slot: stepstr | {section: stepstr}}, swing, subdivision)`, and
  `Melody(slot, octave, phrases, grid, sections)` — a real monophonic line of
  pitched note-ons/offs written straight into a synth (no Pedal Chord; §20).
  `sections=` limits a voice to named sections.
- **`Song`** — `section(name, roots)`, `add(voice)`, `synth(name, library,
  note_col, tracks)` (register an extra synth from `MachineRef` to host a
  `Melody`; §20), `arrange([names...])` (start bars + `LoopEnd`/`SongEnd` computed
  for you), `mix(**dB)` (per-slot trim on the bus, dB→amp), `presets(**{slot:
  index})`, `limiter(ceiling, isp)` (§19), then `compile()` → validated xml (or
  `write(path)`).

The compiler releases each control-driven target wherever it falls silent — a
note-off on the target's own pattern plus a §12.9.1 stop pattern on its Pedal
Chord (covering entry, exit, and the loop) — prepends the §12.9 entry note-off to
each Pedal Chord pattern, and runs `assert_valid` before returning. Worked
example: `src/build_dsl_demo.py` (a short A-blues — arp bass, block pad, comped
stabs, a swung chorus-only lead, a per-section-keyed kit, a pad trim, staggered
presets).

Out of scope for now (planned): per-section *voicing* overrides for a chord
voice, serial FX chains, and polyphonic/velocity direct melodies. The one-call
`compose(spec)` entry point is now implemented (§21). Tests: `tests/test_dsl.py`.

## 18. Measurement-driven mixing (`rebuzz.mix`)

Closes the render→measure→trim loop in code. Render each instrument to its own
**pre-fader** stem (its natural level, before the gain bus), then:

```python
from rebuzz.mix import measure_stems, recommend_gains, format_report
m = measure_stems({'kick':'01_kick.wav', ..., 'pad':'07_pad.wav'})
print(format_report(m, recommend_gains(m)))           # levels + solved bus gains
```

`measure(path)` reports peak, RMS, **active RMS** (gated to when the instrument
plays — stable across sparse vs sustained material), **LUFS** (K-weighted,
BS.1770; uses scipy if present), spectral centroid, and **clip %**.
`recommend_gains(meas, targets, basis, avoid_boost)` solves the per-input bus
gains to hit a target balance: `gain = target - measured`, then (with
`avoid_boost`) the whole mix is normalised so the loudest-needed stem sits at
unity and the rest attenuate — no gain added, no Master clipping, raise Master
to taste. CLI: `python3 src/mix_report.py <stem_dir>`.

Two things it caught on Last Call that ear-mixing missed:
1. **The pad clips at the source** (51% of samples at full scale) — so its
   measured level is only a *lower bound*, and no downstream trim truly fixes the
   overload. The proper fix is to reduce the synth (lower the invFFT preset
   Volume); the bus trim (now −28 dB) is the stop-gap. `clip_pct > 1%` is
   surfaced as a `SOURCE CLIPPING` note in the report.
2. **Basis matters.** A sparse transient (the snare) reads quiet on any
   integrated measure and can become the no-boost ceiling, skewing the absolute
   recommendation. `basis=` (active_rms / lufs / peak / rms) and per-role
   `targets=` are exposed so the balance can be tuned to the material; the
   *relative* diagnosis (pad ~17 dB hot) is robust regardless.

`rebuzz.mix` needs numpy (scipy optional for LUFS) and is deliberately not
imported by `rebuzz/__init__`, so the core song-building package stays
dependency-free. Tests: `tests/test_mix.py` (synthetic signals).

## 19. Master safety limiter (Pedal Limit)

`Pedal Limit` is a managed Effect (look-ahead brickwall maximizer) spliced from
`refs/limitref.bmxml`. Same archetype as Pedal Gain Multi: empty state blob
(`02 00000000`), its own Modern Pattern Editor backend (a **3-column** empty
pattern) plus a sequence entry, and it sums its inputs at channel 0 like Master.

Parameters (all global, stored as a single Track-0 value, so `set_param` works):

| Param | Type | Range | Meaning |
|-------|------|-------|---------|
| Amp | Word | 0–65534 (16384 = unity) | input gain |
| Pan | Word | 0–32768 | pan |
| Threshold | Byte | 0–200 @ **−0.1 dB/step** (0 = 0 dBFS) | brickwall ceiling; lower = drive harder |
| Output Level | Byte | 0–200 @ −0.1 dB/step | final ceiling; makeup = Output/Threshold |
| ISP | Byte | 0/1 | 4× cubic inter-sample (true-peak) detection |

(The `Output Level` parameter name escapes to `Output_x0020_Level` in the XML.)
Makeup gain is `Output/Threshold`, so **Threshold == Output gives no makeup** —
a transparent safety ceiling that only caps peaks, leaving level and dynamics
untouched. Use it as the final machine before Master.

**Last Call** uses it as a true-peak safety net for modern listening: the DrumBus
and every per-synth `Pedal Gain` sum into `Limit` (ch 0), `Limit → Master`, set to
**Threshold = Output = −1.0 dBFS (value 10), ISP on** — guaranteeing ≤ −1 dBTP into
Master to avoid lossy-codec overshoot, without loudness maximisation. The
`limiter()` helper in `build_lastcall.py` and the `dest=` argument on the gain
builders make the (buses + per-synth gains) → limiter → Master chain reusable. For
more loudness/glue, lower Threshold below Output for `Output−Threshold` dB of
makeup (e.g. Threshold −3, Output −1 → +2 dB).

## 20. A real melodic lead — direct notes + an extra synth (`Melody`, `Song.synth`)

Every voice up to here drives a synth indirectly: a **Pedal Chord** reads root
notes and plays the chord/arp, while the synth's own editor stays empty (just
the §12.9 release note-offs). A *real* melodic line is the opposite — pitched
note-ons and note-offs written **straight into the synth's pattern**, no control
machine. Two pieces make this work in the DSL.

**1. An extra synth, spliced from `MachineRef`.** The four DSL slots (Bass/Lead/
Pad/Comp) are fixed by `synthref`, so a *fifth* synth has to come from somewhere.
`Song.synth(name, library, note_col, tracks=1)` registers one; the compiler
splices it from `refs/MachineRef.bmxml` by Library (the cardinal rule still holds
— never fabricate a `<Machine>`), renames it, attaches a fresh Modern Pattern
Editor, positions it, sequences it whole-song, routes it through **its own Pedal
Gain** into the master chain (§15), and makes it eligible for `mix()` and
`presets()` like any built-in slot. `note_col` is the machine's note column from
the catalog (Pedal FM = 42, Faze-R = 67, …).

**2. A `Melody` voice.** `Melody(slot, octave, phrases, grid=16, sections=None)`
holds, per section, a list of `(step, note, length)` in grid units (16th notes by
default → 16 steps/bar). `note` is a token: `'A5'`, `'C#6'`, `'Eb5'`, or bare
`'A'` (uses the voice's `octave`). `column_events()` resolves the phrases over
the **arrangement** into `(row, value)` events — so a phrase written once plays at
every placement of that section (the Last Call verse line lands at both verses
automatically).

**Monophonic note-offs.** On a mono synth a fresh note-on cuts the previous note,
so note-offs are emitted **only** where they're actually needed: a rest after a
note, a phrase end, or the start of a section where the lead is silent. Any
note-off that would collide with the next note-on is dropped (the retrigger
handles it) and offs past song-end are clamped — giving clean legato runs and a
loop-safe line (it never drones, even though `Melody` isn't a Pedal-Chord target
and so isn't covered by validation check 8, §12.9.1). Because the lead writes its
own releases, the only loop-safety obligation is that the last note releases
before song-end and silent sections open with a note-off — both automatic.

**Gain machines size to their inputs.** A `Pedal Gain Multi`'s per-connection
faders live in its `<Type>Input</Type>` group — one `<Value><Track>k</Track>…` per
channel plus a group `<TrackCount>` — and a single `Pedal Gain` has the same group
with one track. A spliced template carries however many the save had (the Gain
Multi 4; an unconnected Pedal Gain stores `<Values />`, i.e. none), so the group
has to be **sized to the actual connection count**: `blocks.set_input_tracks(block,
n)` rebuilds the Input group to exactly `n` tracks (existing fader values kept, new
ones filled from `<DefValue>` = unity, the empty self-closing form handled) and
sets `<TrackCount> = n`. The DrumBus calls it with its drum count; each per-synth
`Pedal Gain` calls it with 1.

**Per-synth gains for stems.** Rather than summing all synths on one shared Gain
Multi, the DSL gives **each synth its own `Pedal Gain`** → limiter (drums keep a
shared Gain Multi). The math is identical — the gains sum at the limiter's channel
0 exactly as a shared bus would — but every synth now has a dedicated, soloable
output node, so the human can render **per-synth stems** and the `rebuzz.mix` loop
(§18) can measure their true relative levels. Each synth's `mix()` trim sits on
its gain's single input Amp (machine + connection kept equal, §15).

**Last Call's Lead2** is a Pedal FM (note col 42, mono) on preset *Lead Bright*
(FM bank index 15), playing through its own `Lead2Gain` into the limiter. It plays
an A-blues line — palette A C D E♭ E G with chord-tone colour (F♯ over D7, G♯ over
E7, B in the bridge) and the ♭3/♭5 blue notes up high — that **carries the verses**
(where the arp lead rests), **climbs into the bridge** alongside the arp for a
two-lead climax, and **resolves in the outro**, sitting out the choruses so the
two leads never crowd each other. See `Melody`/`Song.synth` in
`src/build_lastcall.py`.

## 21. One-call commission entry point (`compose(spec)`)

The whole project exists to turn a *spec* into a song (§13). §17's DSL is the
ergonomic imperative form of that; `compose(spec)` is the **declarative** form —
a single dict, JSON-serialisable end to end, mapped to a `Song` in one call. It
adds no new format capability; it's a faithful front end over the DSL (the demo
spec compiles **byte-identically** to its fluent twin, asserted in the tests).

Spec shape (all but `name` optional):

```python
{
  'name': 'My Song', 'bpm': 96, 'tpb': 8, 'key': 'A', 'scale': 'blues',
  'sections': {'Verse': ['A','A','D','E'], 'Chorus': ['D','D','A','E']},
                                      # or {'Verse': {'roots': [...], 'bars': N}}
  'arrange': ['Verse','Chorus','Verse'],
  'synths':  [{'name':'Lead2','library':'Pedal FM','note_col':42,'tracks':1}],
  'voices':  [
     {'type':'arp',    'slot':'Bass','octave':2,'chord':'dom7','mode':'up','speed':8,'octaves':1},
     {'type':'chords', 'slot':'Pad', 'octave':4,'chord':'dom7'},
     {'type':'chords', 'slot':'Comp','octave':3,'chord':'dom7',
                       'rhythm':{'Verse':'..x...x.','Chorus':'x.x.x.x.'}},
     {'type':'drums',  'patterns':{'Kick':'x...x...','Snare':'..x...x.'},'swing':67},
     {'type':'melody', 'slot':'Lead2','octave':5,'grid':16,
                       'phrases':{'Verse':[[0,'A5',2],[4,'C6',2],[8,'E5',4]]}},
  ],
  'mix':     {'Pad': -20},
  'presets': {'Bass': 0, 'Lead2': 15},
  'limiter': {'ceiling': -1.0, 'isp': True},
}
```

Each `voices` entry's fields mirror the matching voice class (§17): `chords`,
`arp`, `drums`, `melody`. `compose(spec)` returns an **un-compiled** `Song`, so the
caller still gets `.compile()` / `.write(path)` and the full validation pass. An
unknown voice `type` raises. Worked example: **`src/build_lastcall.py`** *is* a
spec — the entire Last Call (sections, all voices, the FM melodic lead, drums,
mix, presets, limiter) is one `LASTCALL_SPEC` dict, round-tripped through
`json.dumps`/`loads` to prove it's plain data, then built with one `compose()`
call. It compiles **byte-identically** to the earlier hand-written DSL build, so
the declarative form is a faithful front end on a real, complete arrangement —
"swap a few fields to commission a different song" against a whole song, not a toy.
