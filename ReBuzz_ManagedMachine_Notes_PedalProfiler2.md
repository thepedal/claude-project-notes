# ReBuzz Managed Machine Notes — Pedal Profiler2

Source: Pedal Profiler2 v1.7 build on ReBuzz 1826-preview.

Pedal Profiler2 is a single-machine inspector — a follow-on to Pedal
Profiler v1 (the global CPU dashboard). It coexists with v1 and adds
per-machine introspection: live cost measurement, parameter activity
tracking, signal-chain navigation, spike attribution, and a suite of
diagnostic tools that grew organically as the May 2026 dropout
investigation deepened.

This document captures both the implementation details and the lessons
from using the tool, especially the audio-thread chunking finding that
closed that investigation. General ReBuzz facts surfaced during the
work live in `ReBuzz_ManagedMachine_Notes_Core.md` §§34–40; this file
references them as `Core §N`.

Local numbering starts at 1 and is independent of Core.

---

## 1. Architecture overview

Pedal Profiler2 is a control machine (`void Work()` — see Core §2)
with a separate resizable inspector window declared via
`PreferWindowedGUI = true, IsGUIResizable = true`. It coexists with the
v1 dashboard machine because they answer different questions: v1 shows
overall CPU and dropouts at a glance, v2 lets you drill into a specific
machine's contribution and into the host's own overhead.

The inspector layout, top to bottom:

1. Machine selector (dropdown + prev/next arrows + type tag + mute toggle)
2. **Cost** panel — three columns: ENGINE / SOLO / MARGINAL
3. **Engine Total** line + stacked-bar visualisation
4. Sparkline of measurement history
5. **Connections** — one-hop upstream/downstream chips, click to focus
6. **Spike Attribution** — summary + expandable per-spike list with intervals
7. **Parameters** table — global + per-track current values
8. **Activity** — type tag + track count
9. **Master Output** — L/R VU meters via MasterTap (Core §37)
10. **Profile All Machines** — Engine and Solo modes
11. **Buffer Sweep Log** — auto-recorded buffer-size sweep data
12. **Diagnostics** — Dump Internals to ReBuzz Debug Console
13. **Footer** — Global CPU / Engine settings / GC state

Per-machine selection persists across song save/load via `MachineState`
(Core §39) framed with the magic `"PP2S"`.

---

## 2. The three cost columns — what each measures

The Cost panel shows three different "cost" numbers because they
answer three different questions. Conflating them produces wrong
conclusions, and the names alone are not enough to disambiguate.

### 2.1 ENGINE — what the engine measured

Read from `MachinePerformanceData.PerformanceCount` and `SampleCount`
(Core §35). This is the engine's own per-machine accounting: actual
QPC ticks spent in the machine's `Work()` over the elapsed audio
samples. Smoothed by EMA across UI ticks for stability.

- **Pros:** live, continuous, non-invasive, accurate.
- **Cons:** managed machines only — native machines (Core §35.3)
  report zero permanently.
- **Use when:** you want to know "what's this machine actually
  costing me right now".

### 2.2 SOLO — buffer cost with everyone else muted

Triggered by **Measure Solo** button. Mutes every other machine for
~2 seconds while recording the buffer period from the timing bracket;
the average minus our own Work() time is the "solo cost". Restores
mute state after.

- **Pros:** works for native and managed machines alike. Captures the
  host's routing/dispatch contribution for this machine.
- **Cons:** invasive (interrupts playback for ~2 seconds). Includes
  per-chunk host overhead (Core §34) since the buffer cycle keeps
  running.
- **Use when:** comparing the smallest possible cost of running just
  this machine.

### 2.3 MARGINAL — savings from muting this machine

Triggered by **Measure Marginal** button. Mutes the target machine
while everything else keeps running; the buffer-period delta is the
marginal saving.

- **Pros:** directly answers "would muting this machine help".
- **Cons:** invasive. Can be negative or zero when the machine is
  cheap relative to noise floor.
- **Use when:** deciding whether to bypass a specific machine to fix
  dropouts.

### 2.4 Why the columns disagree

A machine showing ENGINE = 1.7% but SOLO = 95% is common and not a
bug. The gap is per-chunk host overhead (Core §34) plus per-buffer
mute-state churn during measurement. SOLO is dominated by host
overhead because ReBuzz runs every chunk regardless of whether any
machine is doing meaningful work — the chunk loop iterates the same
way whether there's one machine or twenty.

This was the conceptual key that unlocked the May 2026 investigation:
**if SOLO and MARGINAL are dominated by host overhead, machine-level
optimisation cannot fix host-overhead dropouts.**

---

## 3. The CpuPct metric is buffer-cycle utilisation, not CPU

`Profile2Snapshot.CpuPct` is computed as `AvgOtherMs / BudgetMs * 100`,
where:

- `BudgetMs` — wall-clock period between consecutive `Profiler.Work()`
  calls (i.e., the chunk period — Core §34.1)
- `AvgOtherMs` — `BudgetMs − profilerOwnWorkMs`

Because Profiler.Work() is invoked once per chunk and the driver clocks
chunks at fixed intervals (typically), `AvgOtherMs ≈ BudgetMs ≈ chunk
period`, regardless of how much real audio work the thread did.

**This means CpuPct is always near 100% when the system is healthy.**
It is not a measure of "how much CPU is in use" — it's a measure of
"is the chunk cycle keeping pace with the driver".

The real signal for dropouts is not CpuPct, it's:

- `PeakOtherMs` greatly exceeding `BudgetMs` — a single chunk that took
  much longer than its budget
- `TotalDropouts` / `TotalBuffers` ratio — how often deadlines are missed
- `WindowDropouts` — rolling count for the recent measurement window

A misleading consequence: a healthy session shows CpuPct ≈ 99.99%,
identical to a saturated session. Only the peaks and dropout counts
differentiate them. New users of the inspector instinctively read
CpuPct as headroom indicator and worry about the 99.99% number;
that's a false alarm.

---

## 4. Engine Total / Unaccounted — the diagnostic that ended the investigation

The Cost panel includes a critical line:

```
Engine Total: 0.02 ms (0.4%)  ·  unaccounted: 4.66 ms (99.6%)  ·  n=2 (+1 native)
```

- **Engine Total** — sum of ENGINE costs across all managed,
  non-control machines
- **unaccounted** — `budget − engine_total`
- **n** — number of machines contributing to the sum
- **+N native** — native machines whose engine cost can't be read
  (Core §35.3)

Unaccounted is the central diagnostic. It splits the buffer's wall
time into:

- Time the engine attributed to specific managed machines (Engine Total)
- Time the engine didn't attribute to any machine — host overhead,
  scheduling, native machine work, driver lag

A visual stacked bar below the text shows each managed machine in a
different colour, with the remainder filled in red (intensity scales
with size — bright red for >80%). Red dominating → host overhead
dominating.

### 4.1 What to do with the number

| Unaccounted | Reading                                              |
|-------------|------------------------------------------------------|
| < 30%       | Healthy — machines account for most of the work      |
| 30–60%      | Some host overhead — check connections, MT sync      |
| 60–90%      | Significant overhead — many small machines, deep chains |
| > 90%       | Host overhead dominates — likely a ReBuzz-level issue |

The May 2026 investigation hit >99% unaccounted across multiple test
conditions, leading to the chunking-architecture finding in Core §34.
Any reading consistently above 90% is worth investigating with the
Buffer Sweep (§5) and reporting upstream.

---

## 5. Buffer Sweep Log methodology

A self-running diagnostic that records `(buffer, budget, unaccounted,
dropout-rate)` tuples each time the user changes the ASIO buffer size
in ReBuzz's audio settings. Produces a small table over a session and
emits a verdict line after ≥2 rows.

### 5.1 Detection

In OnTick, watch `Profile2Snapshot.BudgetMs`. If it changes by more
than 10% from the prior tick's value, treat it as a buffer-size
change:

```csharp
bool changed = _sweepLastBudget > 0
            && Math.Abs(budget - _sweepLastBudget) / _sweepLastBudget > 0.10;

if (changed)
{
    _sweepStableSinceMs = now;
    // capture baseline dropout/buffer counters for later delta calc
    _sweepLastDropoutsSnapshot = snap.TotalDropouts;
    _sweepLastBuffersSnapshot  = snap.TotalBuffers;
}
```

After 3 seconds of stable readings (no further significant changes),
record a row. The 3-second wait lets the engine settle and the
dropout-rate sample become meaningful.

### 5.2 The verdict line

After ≥2 rows accumulate, the panel emits one of two verdicts:

- *"unaccounted is roughly constant (X.XX ms): fixed-cost host
  overhead"* — when (max − min) / mean of unaccounted across rows is
  < 30%. Diagnostic of fixed-cost per-chunk overhead — the Core §34
  chunking signature.
- *"unaccounted scales with buffer: includes per-sample work"* — when
  variation exceeds the threshold. Diagnostic that some component
  actually scales with chunk size.

### 5.3 What it proved

In the May 2026 investigation, the verdict was consistently "constant"
with unaccounted ≈ 5 ms across 256 / 512 / 1024 / 2048 sample ASIO
buffers — definitively showing that:

- ASIO buffer size does not change ReBuzz's chunk size
- Per-chunk host overhead is the binding constraint
- No machine-side fix is possible; the issue is structural in ReBuzz

This table is the artifact to capture and send upstream when
reporting host-overhead issues. The raw numbers (buffer column
showing 256 across all rows; unaccounted column showing 99% across
all rows) are unambiguous.

---

## 6. Spike attribution and timing

A "spike" is a single chunk where wall-clock elapsed exceeded
`BudgetMs × threshold` — typically a chunk that took multiple
buffer-periods to complete. The machine records the most recent
8 spikes with:

```csharp
class Spike2Record
{
    public double   ElapsedSec;
    public int      Bpm;
    public double   BudgetMs;
    public double   SpikeMs;
    public string[] ActiveMachines;
}
```

### 6.1 Per-machine attribution

For each spike, the inspector counts how many recorded spikes had the
currently selected machine in their `ActiveMachines` list. The summary
line shows `Active during N of last M spikes (P%)`. High percentages
with high engine-cost are suspect; high percentages with low
engine-cost suggest the machine is just along for the ride (always
running, never the cause).

### 6.2 Interval statistics

When ≥2 spikes are captured, the summary line includes
`intervals μ=Xms σ=Yms (label)` where the label is:

- **periodic** (CV < 0.15) — strong indication of a regular host
  event (timer, DPC, GC) rather than random load fluctuation
- **semi-periodic** (CV < 0.40)
- **irregular** — otherwise

CV = coefficient of variation = σ / μ. Periodic spikes at constant
intervals are a smoking gun for a specific regular cause. Irregular
spikes correlate with content (note triggers, parameter automation).
For the May 2026 investigation the label was "irregular" — confirming
the spikes were random small-disturbance amplifications, not a
periodic host event.

### 6.3 Display

The expandable spike list shows each row with timestamp, BPM,
duration, Δ-since-previous, and the active-machine list (the
currently-selected machine highlighted in red, the row tinted when
present). Useful for eyeballing whether spikes cluster at certain
song positions or arrive evenly distributed.

---

## 7. GC pressure tracking

A footer line shows live garbage collection state:

```
GC: G0=225 (1.3/s) · G1=33 (0.00/s) · G2=7 (0.00/s) · heap 67.1 MB · last G2 41 s ago
```

Each generation's per-second rate is EMA-smoothed across the last
~10 seconds. The line flashes red for 1.5 seconds after each Gen 2
collection and turns orange (warning) if Gen 2 rate stays above
0.5/s sustained.

### 7.1 Diagnostic value

If audible crackles correlate visually with the Gen 2 flash, GC
pauses are the cause. Fix path:

- Reduce per-tick allocations (string interpolation, LINQ, transient
  lambdas, dictionary lookups that box)
- Confirm `GCSettings.LatencyMode = SustainedLowLatency` is active
  (ReBuzz's LowLatencyGC setting — Core §36)
- For sustained pressure, schedule `GCSettings.LargeObjectHeapCompactionMode = CompactOnce` 
  off the audio thread (e.g., in a background timer)

### 7.2 What it ruled out (May 2026)

The investigation showed Gen 2 firing once every ~20–40 seconds while
dropouts happened continuously at 11.5% rate. Gen 1 rate was
effectively zero. **GC was not the cause** despite being the strongest
a priori hypothesis. Eliminating it left chunking overhead (Core §34)
as the remaining explanation.

The G2 timestamp display ("last G2 41 s ago") is the critical detail
— a single Gen 2 cannot cause continuous dropouts spread over many
seconds. Correlating the timestamp with the dropout counter delta is
the empirical test.

---

## 8. Master Output VU via MasterTap

The "Master Output" section shows live L/R peak and RMS meters from
the actual rendered master-bus audio, captured via the MasterTap hook
(Core §37).

### 8.1 Bar visualisation

Each channel has a Canvas with rectangles:

- Background dark grey
- Filled portion proportional to peak-hold value (decaying at
  6 dB/sec UI-side)
- Coloured zones — green up to -18 dB, yellow -18 to -6 dB, orange
  -6 to 0 dB
- White line marker showing instantaneous current peak (distinct
  from peak-hold)
- Subtle RMS marker as a lighter vertical line
- Tick marks at -36, -24, -18, -12, -6 dB for scale reference

Chose WPF Rectangles over Unicode block characters after testing —
text-based bars had unpredictable widths due to font fallback of `█`
and `░` characters in some setups. Rectangles render predictably at
any DPI.

### 8.2 dBFS scaling

MasterTap samples are in Buzz's ±32768 convention (Core §38). Divide
by 32768 before computing dB:

```csharp
double db = lin < 1e-6f ? -120 : 20.0 * Math.Log10(lin / 32768.0);
```

A correctly-scaled meter shows negative dBFS values during normal
music (typically -20 to -6 dB peaks), with 0 dB indicating digital
full scale.

### 8.3 Thread safety

The audio-thread handler updates `volatile float` fields with peak and
RMS. The UI tick reads them — single-float reads/writes are atomic
on x64. Peak-hold decay logic runs UI-side, not in the audio handler,
which lets the audio handler stay minimal:

```csharp
volatile float _masterPeakL, _masterPeakR;
volatile float _masterRmsL,  _masterRmsR;

// Audio thread — sets the volatiles
void OnMasterTap(float[] s, bool stereo, SongTime t) { /* ... */ }

// UI thread — reads them and applies peak-hold decay
void RefreshMasterOutput()
{
    float peak = _masterPeakL;        // atomic on x64
    /* apply decay, draw bar */
}
```

No locks, no allocations, no exceptions out of the audio handler.

### 8.4 Use case

Confirms "is audio actually flowing" during high-CPU sessions. A
100%-CPU buffer-cycle utilisation showing the meter at -inf dB
(silence) is qualitatively different from one showing -12 dB peaks
(normal high-load operation). Same CpuPct number; very different
audio reality.

---

## 9. Reflection-dump diagnostic

The "Dump Internals (DC)" button walks `IBuzz`, `Song`, the selected
`IMachine`, a parameter, and the snapshot via reflection, and writes
the entire structure to ReBuzz's Debug Console. Two phases:

- **Phase 1** — top-level walk at depth 1: every property and field
  of the root objects, formatted as `name : type : value`, with
  collection counts and brief previews
- **Phase 2** — drill into specific objects of interest
  (`BuzzPerformanceData`, `AudioEngine`, `EngineSettings`,
  `MachinePerformanceData`)

The dump output is the primary tool for discovering new ReBuzz APIs.
The chunking finding, the engine-perf finding, and the MasterTap
subscription path all started with reflection dumps that surfaced
fields not previously known to be reachable. Run it against a
specific machine in a specific state, paste the output, iterate.

### 9.1 Output format

```
── IMachine 'MComp' ──
  runtime type: ReBuzz.Core.MachineCore
  assembly:     ReBuzz
  properties (53):
    AllInputs    List<IMachineConnection>  = [MachineConnectionCore] count≈1
    ...
  fields (69):
    performanceData             MachinePerformanceData  = MachinePerformanceData
    performanceDataCurrent      MachinePerformanceData  = MachinePerformanceData
    performanceLastCount        Int64                   = 185
    ...
```

Collection counts and short previews keep the output compact. Complex
objects show as `TypeName` with a brief ToString preview. Field/property
counts in the header help spot when a build adds new surface.

### 9.2 Adding to the dump

When investigating a new question, add another
`DumpObject(GetProp(buzz, "X"), label, Line, 1)` line to the Phase 2
section, ship a build, run, paste the dump back. The pattern works
well: each round narrows the search space by a level of nesting. Most
new ReBuzz findings in §§34–40 (Core) emerged from 2–4 dump-iteration
cycles.

---

## 10. Profile All Machines — two modes

The Profile All section has two buttons answering different questions.

### 10.1 Profile All (Engine)

Reads engine perf for every non-control non-native machine at T0,
waits 1 second, reads again, computes deltas, displays a sorted
bar chart. ~1 second total, non-invasive — playback continues
normally.

The preferred mode for surveying a song's CPU distribution. The bars
scale against the buffer budget (not the heaviest machine), so
absolute proportions stay meaningful across runs and across songs.

### 10.2 Profile All (Solo)

Iterates solo measurement across every non-control machine. ~2
seconds per machine — measurably invasive on longer songs (a song
with 15 machines takes ~30 seconds, with audible muting throughout).

Useful when comparing per-machine costs *including* the host's
per-chunk overhead contribution. The Solo numbers are always larger
than Engine numbers; the gap represents the per-chunk overhead the
host attributes to no specific machine.

### 10.3 Click to focus

Each row in the result panel is a button that switches the
inspector's machine selector to that machine. Quick way to drill from
"this song's heaviest machine" to "what's that machine doing in
context" without re-navigating the dropdown.

---

## 11. Audio-thread overhead disclosure

PP2's own `Work()` does a single `Stopwatch.GetTimestamp()` per call
and updates a circular buffer of timestamps. Sub-microsecond per
invocation.

The UI thread does considerably more — sparkline drawing, parameter
polling, engine-perf reads across all machines per tick at 10 Hz. The
UI tick runs on the WPF dispatcher off the audio thread's hot path,
but lives in the same process and competes for CPU cores.

In the May 2026 investigation, closing the PP2 GUI window made no
measurable difference to dropout rate — so PP2's UI thread is not
contributing materially under load. This should be re-tested after
any future UI changes (especially additions that touch every
machine per tick).

The MasterTap hook (§8) adds work directly on the audio thread, but
the work is bounded — one linear pass over the master buffer with
cheap arithmetic. At 48 kHz with 256-sample chunks, measured
contribution is well under 50 µs per invocation.

---

## 12. The case study — May 2026 dropout investigation

A multi-week investigation traced an 11.5% dropout rate to ReBuzz's
chunking architecture. The sequence is preserved here as a worked
example of the elimination approach the inspector enables.

**Sequence of tests, with results:**

1. **Reported symptom.** 99.998% buffer-cycle utilisation, 41 ms
   peaks, 11.5% dropout rate. Audible glitches throughout playback,
   on every song.
2. **OS priority change.** `NormalAppPriority` → `AllFocusOnAudio`.
   **No effect** on dropout rate. → ruled out OS thread scheduling.
3. **Driver swap.** Fireface USB → ASIO4ALL. **No effect**. →
   ruled out driver-specific behaviour.
4. **Empty song + stopped transport.** Still 11.5%, still 41 ms
   peaks. → ruled out song content and active playback work.
5. **GC tracker (§7).** Gen 2 firing once per ~30 s; Gen 1
   effectively zero. → ruled out garbage collection.
6. **PP2 GUI closed.** No change in dropout rate. → ruled out PP2's
   own UI overhead.
7. **Engine Total / Unaccounted (§4).** Showed 99.7% unaccounted. →
   pointed at host overhead, not machine work.
8. **Buffer Sweep (§5).** Across 256 / 512 / 1024 / 2048 sample
   ASIO buffers: `BudgetMs` stayed at 5.33 ms; `Unaccounted` stayed
   at ~5 ms. Verdict: "fixed-cost host overhead". → confirmed
   chunking architecture as the binding constraint.

**Final diagnosis:** ReBuzz processes audio in ≤256-sample chunks
regardless of ASIO buffer size; per-chunk host overhead is ~5 ms;
chunk wall-clock budget is 5.33 ms at 48 kHz; zero margin for any
disturbance. Not fixable from the machine side. Upstream report sent
to ReBuzz devs with the Buffer Sweep table and the
Engine Total / Unaccounted numbers as the supporting evidence.

The tool that solved it (PP2 v1.7) didn't exist when the
investigation started — it grew section by section as each hypothesis
needed visibility and the next layer needed instrumentation. The
final layout (§1) is the residual structure of that investigation
preserved as features. Each section corresponds to a question that
had to be answered to move forward.
