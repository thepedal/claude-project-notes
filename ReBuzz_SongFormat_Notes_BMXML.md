# ReBuzz Song-File (.bmxml) & Note-Data Format — State of Play

Source: full reverse-engineering session against ReBuzz **Build 1827**
(github.com/wasteddesign/ReBuzz source + byte-level analysis of real saved
songs). All claims below were verified either against the ReBuzz loader source
or by **byte-exact round-trip** (regenerate a real saved blob and compare ==
original). Where a fact is empirical-only it is marked *(empirical)*.

This file documents the **XML song format** and the **Modern Pattern Editor
(MPE) note-data blob** — i.e. how a `.bmxml` song is laid out on disk and how
playable note data is actually stored. It is a different concern from the
`ReBuzz_ManagedMachine_Notes_*` files (which cover writing the machines
themselves). Cross-references to those use `Core §N` etc.

Worked example throughout: the song **"Limani"** (18 machines: a 4-piece Plaits
drum kit + bass/lead/pad/comp synths, ~64 s, D Hijaz). Its build script is the
canonical implementation of everything here.

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
int32  reserved = 0
byte   flag     = 0
int32  eventCount
eventCount × { int32 pos ; int32 value }     # pos = row * 240 ; value = note byte
int32 ×4 trailer  (all = 4)                  # per-column, not per-row
```

- `pos = row * 240` *(empirical; 240 = internal ticks per row)*.
- `value` is the §3.2 note byte (e.g. `66` = C#4, `65` = C-4 drum trigger).
- The trailer is four int32s each `= 4`, **once per column** regardless of how
  many events — so blob size scales with event count only.

### 4.4 Column counts & note-column index per machine

The number of column-records equals the machine's exposed pattern-column count.
Notes land in that machine's **Note parameter** column — which is **not always
the last column**, so determine it empirically (enter one note, save, decode):

| Machine (Library) | role in Limani | columns | **note column** |
|---|---|---|---|
| Pedal Plaits   | drums (Kick/Snare/HatClosed/HatOpen) | 14 | **12** |
| Pedal SH101    | Bass | 26 | **25** (last) |
| Pedal invFFT   | Pad  | 29 | **28** (last) |
| Pedal Juno106  | Comp | 26 | **25** (last) |
| Pedal Faze-R   | Lead | 69 | **67** (NOT last — col 68 exists) |

### 4.5 Length-agnostic

The blob has **no row-count field** (f1/f2 are the constant `4`, not the row
count). The same format therefore serves **any** pattern length — a 16-row clip
and a 256-row full-song pattern use identical structure; only the event `pos`
values and the PatternCore `<Length>` differ. This is why a single long
pattern per machine works.

---

## 5. Verified generator & decoder (Python)

This `build_blob` reproduced **all 8** reference blobs byte-for-byte.

```python
import struct, base64

def build_blob(machine_name, patterns):
    # patterns: list of dict(name=str, ncols=int, notecol=int, events=[(row,value),...])
    body = struct.pack('<i', len(patterns))
    for p in patterns:
        body += p['name'].encode('latin1') + b'\x00'
        body += struct.pack('<iii', 4, 4, p['ncols'])       # f1, f2, recordCount
        for c in range(p['ncols']):
            body += machine_name.encode('latin1') + b'\x00'  # column name = GEN name
            body += struct.pack('<ii', c, 0) + b'\x00'       # colIdx, reserved, flag
            evs = p['events'] if c == p['notecol'] else []
            body += struct.pack('<i', len(evs))
            for row, val in sorted(evs):
                body += struct.pack('<ii', row * 240, val)
            body += struct.pack('<iiii', 4, 4, 4, 4)         # trailer
    total = 2 + 4 + len(body)                                 # magic + lenfield + body
    return b'\xff\x01' + struct.pack('<I', total) + body

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
            cn, o = rd_s(o); col, o = rd_i(o); _, o = rd_i(o); o += 1; evc, o = rd_i(o)
            ev = []
            for _ in range(evc):
                pos, o = rd_i(o); val, o = rd_i(o); ev.append((pos // 240, val))
            o += 16                          # trailer
            if ev: cols.append((col, ev))
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
   (§5).
5. Rebuild `<MachineConnections>` (every generator **and** every editor → Master)
   and `<Sequences>` (one per generator).
6. Set `LoopEnd`/`SongEnd` to the song length; write with the BOM.

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

> **Pedal Chord caveat:** earlier attempts to use Pedal Chord (a *control*
> machine) as an arpeggiator inside the song threw a NullReferenceException at
> load. It was dropped from the Limani design; bass/lead/pad/comp are plain
> generators.

---

## 8. Playback parameters

- Pattern timing is **TPB 4** (4 rows per beat ⇒ 16 rows per bar).
- BPM is **set after load** (Limani is designed for **60 BPM**). At 60 BPM /
  TPB 4 the row rate is 4 rows/s, so 16 bars = 256 rows ≈ **64 s**.
- The root `Speed` element was `0` in working files; tempo came from the
  ReBuzz transport, not this field.

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
- **Form:** 16 bars × 16 rows = 256 rows; per-bar root degrees
  `D D E♭ D G G A D  D E♭ B♭ A G E♭ A D`.
- **Arrangement (build-up):**
  - *Bass* (SH101, oct 2): root on rows 0 & 8 of every bar — plays throughout.
  - *Pad* (invFFT, oct 4): root on row 0 of every bar — plays throughout.
    (Intro bars 1–2 are bass + pad alone.)
  - *Kick* (Plaits): rows 0,8 — from bar 3.
  - *HatClosed* (Plaits): straight 8ths (rows 0,2,4,6,8,10,12,14) — from bar 3.
  - *HatOpen* (Plaits): row 14 accent — from bar 3.
  - *Snare* (Plaits): rows 4,12 — from bar 5.
  - *Lead* (Faze-R, oct 4–5): composed D-Hijaz phrase, bars 5–16, leaning on
    F♯→G and B♭→A, with an E♭→F♯ augmented 2nd, resolving to D4 in bar 16.
  - *Comp* (Juno106, oct 3): root on rows 4,12 — from bar 9.
- All drum hits trigger note value **65** (C-4); the kit's timbres come from each
  Plaits machine's own parameters (only the kick was tuned in the source song —
  snare/hats were default Plaits and may need parameter tweaks).

---

## 11. Error chronology (all resolved) — quick index

1. Root must be `<ReBuzzSong>` (the `[XmlRoot]` alias), not `<BMXMLSong>`.
2. Build 1827 needs the UTF-8 BOM + correct top-level/Machine/Parameter order.
3. Pedal Chord as a control machine ⇒ NRE on load ⇒ dropped.
4. Patterns silent ⇒ PatternCore columns are ignored; editor machines + MPE
   blobs are authoritative.
5. Synths "not editable / can't add tracks or patterns" ⇒ root cause was
   **fabricated** machine blocks. Fix: reuse ReBuzz-generated blocks verbatim
   (§6). Confirmed editable once real SH101/Faze-R/invFFT/Juno106 blocks were
   spliced in.
6. MPE blob length header ("mystery bytes") ⇒ it's the **uint32 total blob
   length** (§4.1); cracking it enabled byte-exact blob generation.
