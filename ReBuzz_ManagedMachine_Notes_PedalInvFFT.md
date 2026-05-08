# ReBuzz Managed Machine Development — Pedal invFFT Addendum

Source: ReBuzz 1819-preview source code + Pedal invFFT v2.0 build (an
8-voice polyphonic extension of the v1.0 monophonic K5000-inspired
additive synth using inverse FFT with overlap-add resynthesis,
16 partials per voice, with amp and brightness ADSRs, static spectrum
shaping, formant filter, and glide).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. References to `PedalComp §N`,
`PedalSH101 §N`, etc., point to those addenda. Internal cross-references
use plain `§N`.

The findings here are mostly specific to inverse-FFT / overlap-add
synthesis, but several patterns generalize to any frame-based DSP
machine in ReBuzz where parameters update at a different rate from
samples.

---

## 1. Architecture choice — FFT_SIZE, HOP_SIZE, and the hop rate

For frame-based resynthesis, three constants determine almost everything:
**N (FFT_SIZE)**, **H (HOP_SIZE)**, and the implied window choice. Pedal
invFFT uses N=2048, H=512, Hann window, giving 75% overlap (H = N/4).

The four numbers that fall out of this:

| Quantity              | Formula        | Value at sr=48000 |
|-----------------------|----------------|-------------------|
| Latency               | ~N/2 samples   | ~21 ms            |
| Bin width             | sr/N           | 23.4 Hz           |
| Hop rate              | sr/H           | ~94 Hz            |
| Intrinsic OLA ramp    | N samples      | ~43 ms            |

**Latency** is the time from a parameter change taking effect to the
listener hearing it. Half the FFT window because the windowed frame
peaks at the centre and tapers to zero at both ends — what's audible
*now* is what was deposited in the spectrum half a window ago.

**Bin width** sets the worst-case pitch error if you don't spread
partials across bins (§2). 23 Hz is over a semitone in the bass —
unacceptable without bin spreading.

**Hop rate** caps how fast spectrum-domain parameters can modulate.
At 94 Hz, anything modulating slower than ~10 ms is rendered
faithfully; faster than that gets quantized to the hop boundary.

**Intrinsic OLA ramp** is the time the overlap-add buffer takes to
build up from one frame's worth of signal to four frames' worth at
75% overlap. Note onsets and offsets can't be sharper than this
regardless of envelope settings (§13).

Tradeoff space: doubling N halves the bin width and the hop rate,
doubles the latency. Halving H doubles the hop rate (better envelope
tracking) but doubles the iFFT cost. For pads/leads (long attacks,
smooth timbral movement), N=2048 H=512 is comfortable. For percussion
you'd want N=512 H=128 — same overlap ratio, ~5 ms latency, ~11 ms
ramp, but 16× the iFFT count per second.

---

## 2. The fractional-bin problem and the Hann lobe weight

A 100 Hz fundamental at 48 kHz with N=2048 lands at bin 4.27 — not on
the integer grid. Just rounding to bin 4 means a 6.4 Hz pitch error,
**over a semitone in the bass.** Unusable without bin spreading.

The standard fix: each partial deposits energy across ~5 adjacent bins,
weighted by the spectral leakage profile of the analysis/synthesis
window. For a Hann window, the main lobe is ~4 bins wide; a 5-bin
deposit kernel centred on the fractional bin position with Hann-shaped
weights gives essentially perfect pitch placement at trivial cost (5×
the per-partial bin writes — still cheap).

The weight function is the Hann window's frequency-domain lobe
evaluated at the offset between the bin and the fractional target:

```csharp
// Ŵ(d) = sinc(d) − 0.5·sinc(d−1) − 0.5·sinc(d+1)
static float HannLobeWeight(float d)
{
    return Sinc(d) - 0.5f * Sinc(d - 1f) - 0.5f * Sinc(d + 1f);
}

static float Sinc(float x)
{
    if (MathF.Abs(x) < 1e-6f) return 1f;
    float px = MathF.PI * x;
    return MathF.Sin(px) / px;
}
```

Derivation: the Hann window in the time domain is `(1 − cos(2πn/N))/2`,
which is a sum of three complex exponentials. Its DTFT is therefore
three Dirichlet kernels — one at offset 0 with weight 1, two at
offsets ±1 with weight −0.5. Sample at integer bins, get the weight
function above.

The 5-bin truncation (bins `kCent−2` through `kCent+2`) captures
essentially all the lobe energy: at d=±2 the weight is exactly zero
(`sinc(2)=0`, `sinc(1)=0`, `sinc(3)=0` → weight=0), so going wider
buys nothing.

---

## 3. Scheme A vs Scheme B — where the window lives

Two ways to do iFFT/OLA synthesis, equivalent in output but very
different in cost and spectral cleanliness:

**Scheme A: deposit unwindowed sinusoid, multiply by window in time
domain.** Deposit at the +k bin a value approximating the DTFT of an
unwindowed sinusoid (a Dirichlet kernel ≈ `N·sinc(k − k*)`). After
iFFT, you get a clean cosine spanning the full frame. Multiply by the
Hann window. Overlap-add.

**Scheme B: deposit Hann-windowed sinusoid spectrum directly, no
time-domain window.** Deposit at the +k bin the DTFT of a *Hann-
windowed* sinusoid (the Hann lobe weight from §2). After iFFT, you
get a Hann-shaped envelope modulating the cosine — *the windowed
signal directly*. Overlap-add without further multiplication.

For sustained sinusoids, both schemes produce identical output. The
differences emerge when you truncate the deposit kernel to 5 bins:

- Scheme A's truncated sinc has sidelobes at −13 dB, leaking into
  neighboring partials.
- Scheme B's truncated Hann lobe has sidelobes around −32 dB —
  negligible bleed.

Scheme B also saves a per-sample multiply across the entire OLA frame
(N=2048 multiplies per hop, gone). Pedal invFFT uses Scheme B; the
per-hop cost saving alone is significant, and the cleaner spectral
behavior is the whole reason to spread bins in the first place.

---

## 4. Hermitian symmetry for real output

`FFT.Inverse()` produces a complex output. To get a real time-domain
signal, the input spectrum must be Hermitian symmetric: `X[N−k] =
conj(X[k])` for all k.

For each partial deposit, write to *both* bins:

```csharp
for (int dk = -2; dk <= 2; dk++)
{
    int bin = kCent + dk;
    if (bin <= 0 || bin >= posBinLimit) continue;

    float w = HannLobeWeight(bin - kStar);

    _specRe[bin]     += reC * w;
    _specIm[bin]     += imC * w;

    int negBin = FFT_SIZE - bin;
    _specRe[negBin] += reC * w;     // same real part
    _specIm[negBin] -= imC * w;     // negated imag
}
```

The DC bin (k=0) and the Nyquist bin (k=N/2) are their own conjugates;
they should be written only once, not paired. The `bin <= 0 || bin >=
posBinLimit` guard takes care of this — `posBinLimit = N/2` skips the
Nyquist bin, and `bin <= 0` skips DC.

Reading only the real part of the iFFT output works because the
imaginary part is all zero up to floating-point error if the spectrum
is truly Hermitian. It's defensive to discard it explicitly rather
than trust it stayed clean — but ignoring it is fine in practice.

---

## 5. Gain derivation — DEPOSIT_GAIN and COLA_INV

For a single partial at amplitude `A`:

1. **Deposit at +k and −k bins**: each gets `A·N/4` magnitude (this is
   the Hann lobe centre weight `Ŵ(0)=1`, scaled to make iFFT produce a
   unit-amplitude signal). The factor `N/4 = 512` is `DEPOSIT_GAIN`.

2. **iFFT** applies `1/N` normalization. Per-bin contribution becomes
   `A/4`. Sum of +k and −k contributions gives a Hann-windowed cosine
   with peak `A/2` per frame.

3. **OLA accumulation** at H=N/4 (75% overlap): four windowed frames
   contribute to each output sample. The Hann window's COLA constant
   at this overlap is exactly 2, so the stationary signal sums to
   `A · cos(...)`.

4. **Output scaling**: `output = olaBuf · COLA_INV · 32768`, where
   `COLA_INV = 1/2` is the per-frame scale that compensates for the
   OLA buildup if we want a single isolated frame to peak at A.

The math could be folded into one constant, but keeping `DEPOSIT_GAIN`
and `COLA_INV` separate makes the per-stage scaling self-documenting.
The path from "amp=1 partial" to "±32768 output" is:

```
A=1 → deposit N/4 at ±k bins → iFFT gives 0.5·cos(...) per frame
    → OLA sums 4 frames into 1·cos(...) (stationary)
    → drain × COLA_INV × 32768 → ±32768·cos(...)
```

The COLA_INV factor is what keeps a single hop's worth of audio
(during the ramp-up at note onset) from being twice as loud as the
sustained signal.

---

## 6. Phase tracking across hops

For partials at fractional bins, the spectrum deposit centred on
`kCent ≈ kStar` differs each hop because `kStar` is fractional. If
phases aren't tracked across hops, successive frames at the same
partial frequency land with random phase relationships and beat
against themselves at the hop rate.

Per partial, advance phase by the partial frequency's contribution
over one hop:

```csharp
_phases[p] += (2π · HOP_SIZE / sr) · partialFreq;
while (_phases[p] >  π) _phases[p] -= 2π;
while (_phases[p] < -π) _phases[p] += 2π;
```

The phase advance constant `(2π · HOP_SIZE / sr)` is independent of
partial — compute it once per hop, multiply by each partial's
frequency. The wraparound keeps `_phases[p]` bounded; without it the
float accumulates over thousands of hops and eventually loses
precision.

For integer-bin partials phase tracking is mathematically unnecessary
(the windowed cosine recovers cleanly regardless), but the cost is
trivial and it removes a class of bug entirely — fractional bins
*always* need it.

---

## 7. OLA drain pattern

The audio thread fragment that ties iFFT frames to per-sample output:

```csharp
int produced = 0;
while (produced < n)
{
    if (_samplesUntilNextHop <= 0)
    {
        RunHop();                             // shift, build, iFFT, accumulate
        _samplesUntilNextHop = HOP_SIZE;
        _olaReadPos          = 0;
    }

    int batch = Math.Min(n - produced, _samplesUntilNextHop);
    for (int i = 0; i < batch; i++)
    {
        float env = _ampEnv.Process();
        float s   = _olaBuf[_olaReadPos + i] * gain * env;
        output[produced + i] = new Sample(s, s);
    }
    produced            += batch;
    _olaReadPos         += batch;
    _samplesUntilNextHop -= batch;
}
```

Two state variables track position:

- `_samplesUntilNextHop` counts down; when it hits 0, run the next hop.
- `_olaReadPos` is the index into `_olaBuf` for the *next* sample to
  drain. Reset to 0 when a new hop runs (the shift moves fresh samples
  to the front).

The shift in `RunHop()`:

```csharp
Array.Copy (_olaBuf, HOP_SIZE, _olaBuf, 0, FFT_SIZE - HOP_SIZE);
Array.Clear(_olaBuf, FFT_SIZE - HOP_SIZE, HOP_SIZE);
```

Moves the still-ringing tail of past frames forward, zeros the new
tail position. The fresh frame's iFFT output adds into all FFT_SIZE
samples; the first HOP_SIZE of those become the next batch to drain,
the remaining `FFT_SIZE − HOP_SIZE` are residual tails that overlap
with future frames.

Initialising `_samplesUntilNextHop = 0` makes the first `Work()` call
trigger an immediate hop — clean start without special-case code.

---

## 8. Silent fast-path and OLA buffer reset on retrigger

When the amp envelope is in `Idle`, no signal exists anywhere and the
entire hop machinery can be skipped:

```csharp
if (!_ampEnv.IsActive)
{
    for (int i = 0; i < n; i++) output[i] = new Sample(0f, 0f);
    return true;
}
```

This is the "free silence" fast path. `IsActive` returns true for
Attack/Decay/Sustain/Release, false only for Idle. So the path runs
only between notes (or before the first note).

The OLA buffer must be cleared when transitioning *into* an active
state, not on every retrigger:

```csharp
void TriggerNote(byte buzzNote)
{
    bool wasIdle = !_ampEnv.IsActive;       // capture BEFORE NoteOn
    // ... pitch / glide setup ...
    if (wasIdle)
    {
        Array.Clear(_olaBuf, 0, FFT_SIZE);
        _samplesUntilNextHop = 0;
        _olaReadPos          = 0;
    }
    _ampEnv.NoteOn();
}
```

When transitioning from Idle, the OLA buffer holds stale residue —
either silence (if we never played) or the last note's release tail
that decayed below `IsActive`'s threshold. Either way, clear it so
the new note starts cleanly. When the env is *already* active
(retrigger during release or sustain), DON'T clear — the OLA tail
naturally crossfades into the new pitch's frames (§9).

`wasIdle` must be captured *before* `NoteOn()` — the NoteOn transitions
the env out of Idle, and the check would always be false otherwise.
Same gotcha as PedalSH101 §6.1.

---

## 9. Click-free retrigger via OLA crossfade

In a conventional sample-based synth, retriggering during a release
tail requires fade-out / fade-in machinery to avoid clicks. With OLA
synthesis you get this for free: the tail of the old pitch keeps
ringing through the OLA buffer for ~32 ms (the intrinsic ramp, §13)
while new frames at the new pitch start accumulating. The two pitches
crossfade naturally over the overlap period.

Combined with the click-free envelope retrigger pattern from
PedalSH101 §6.2 (`NoteOn()` doesn't reset `_level`), you get a fully
click-free voice with no explicit anti-click code. Old pitch's
amplitude tail decays naturally as its OLA frames age out; new pitch
builds up via the same mechanism.

The only audible artifact is a brief spectral smear during the ~32 ms
crossfade where both pitches are present. For musical playing this
sounds like a smooth glide; for percussive retriggers it's barely
perceptible. Setting `Glide > 0` (§12) makes the crossfade pitch
slide explicit and continuous.

---

## 10. Hop-rate vs sample-rate parameter updates

Different parameters need different update rates. The choice depends
on where the parameter is *consumed*, not how often it changes.

**Sample-rate consumers** (called for every output sample):
- Amp envelope: applied per-sample in the drain loop, scales the
  drained OLA value before output. Must update per-sample to give a
  smooth gain curve.

**Hop-rate consumers** (used once per RunHop):
- Brightness envelope: applied to the spectrum each time a frame is
  built. The envelope only needs to be sampled once per RunHop —
  the spectrum builder reads `_brightEnv.Level` once at the top of
  RunHop. (See also §11.)
- Glide: `_currentMidi` advances per-hop; `_freqHz` is recomputed in
  RunHop. The intrinsic OLA crossfade smooths the inter-hop
  pitch jumps.
- Formant coefficients: `centerHz`, `widthOct`, `peakLin` are
  computed once in RunHop and cached for the partial loop.
- Brightness/Tilt/Balance: read once per RunHop into the partial loop.

**The pattern**: any parameter whose *consumer* runs once per hop
should be updated once per hop. Doing it per-sample wastes work —
and in one specific case (§11), causes a hang.

For envelopes specifically, "advance once per hop" can be implemented
two ways: a batched loop calling `Process()` HOP_SIZE times, or a new
`Process(int samples)` method that uses `Pow` on the exponential coefs
to advance N samples in one scalar update. The batched loop is simpler
and at HOP_SIZE=512 costs ~5 µs at modern CPU speed — negligible
compared to the iFFT cost in the same RunHop call.

This is the same pattern as PedalComp §6 (dirty-check coefficient
caching) and PedalSH101 §2 (control-rate filter coefficients), seen
from a different angle: the right rate at which to compute something
is the rate at which it's consumed, not the rate at which it changes.

---

## 11. Per-sample Process on a non-amp envelope instance hangs ReBuzz

**Symptom**: Adding a second `Envelope` instance and calling its
`Process()` method per sample in the same drain loop where
`_ampEnv.Process()` is already called per sample causes ReBuzz to
hang immediately on transport play. The machine builds, loads,
places into the song, accepts pattern data — and the moment Play is
pressed, ReBuzz becomes unresponsive.

**What we know:**

- `_ampEnv.Process()` per-sample (one envelope instance) works fine —
  v0.2 through v0.3.1 shipped this way.
- A second envelope instance, identical class, used only for state
  tracking (return value not consumed), called per-sample in the same
  loop, hangs.
- The second envelope's other methods (`UpdateCoefs`, `NoteOn`,
  `NoteOff`, `ForcedRelease`, `Level` getter) all work fine in
  combination — only the per-sample `Process()` call is the trigger.
- Calling `Process()` on the second instance HOP_SIZE times in a
  batched loop (same total calls per second, just consolidated to one
  call site) works fine.

**What we don't know**: the actual mechanism. The Envelope class is
allocation-free, lock-free, and contains no transcendental functions
in `Process()` (just a switch + multiply + comparison). The second
instance is constructed identically. The default param values
(`AmpSustain=100` vs `BrightSustain=0`) lead to different runtime
states but neither produces denormals or NaNs in the relevant code
paths. JIT compilation has already happened by the time the
per-sample loop runs.

**Workaround**: advance hop-rate envelopes in a batched loop at the
end of `RunHop()`:

```csharp
void RunHop()
{
    // ... shift OLA, build spectrum, iFFT, accumulate ...

    // Advance the brightness env by HOP_SIZE samples.
    for (int s = 0; s < HOP_SIZE; s++)
        _brightEnv.Process();
}
```

The Level read at the top of the *next* RunHop reflects state at
end-of-this-hop = start-of-next-hop, identical to per-sample timing.
No audible difference.

**Open question**: this remains unexplained. Things worth trying next
time someone has the patience to dig further: call `Process()` on the
*same* `_ampEnv` instance twice per sample (does the issue depend on
the instance, or just the call count?); replace the second envelope
with a stub that does the same arithmetic but as inline code (does
the issue depend on the call site being a method call?); reproduce
in a much simpler machine with no spectrum/iFFT machinery at all
(does it depend on something about this machine's audio thread shape,
or is it general?). None of these were tried under the time pressure
of getting v1.0 shipped.

---

## 12. Glide in semitone space at hop rate

Glide on a frame-based synth must update at the rate that the
spectrum builder reads pitch — once per hop. The implementation is a
per-hop one-pole on a continuous-MIDI float:

```csharp
float _currentMidi = 60f;
float _targetMidi  = 60f;

// In RunHop, before computing _freqHz for this hop's spectrum:
if (Glide > 0 && _currentMidi != _targetMidi)
{
    float glideMs = 5f * MathF.Pow(400f, Glide / 127f);   // 5 ms .. 2 s
    float coef    = MathF.Exp(-1000f * HOP_SIZE / (glideMs * sr));
    _currentMidi  = _targetMidi + (_currentMidi - _targetMidi) * coef;
    if (MathF.Abs(_currentMidi - _targetMidi) < 0.001f)
        _currentMidi = _targetMidi;
}
_freqHz = 440f * MathF.Pow(2f, (_currentMidi - 69f) / 12f);
```

Working in MIDI space (continuous semitones) rather than Hz keeps the
slide perceptually linear — a 1-octave slide at the bottom of the
keyboard sounds the same speed as one at the top.

The per-hop coefficient maps the dialed time onto our hop rate:
`coef = exp(-1000 · H / (glideMs · sr))`. Each hop reduces the
remaining distance by `(1 − coef)`. At glideMs=100 ms with H=512,
sr=48000: coef ≈ 0.9, so each hop closes 10% of the gap.

The sub-cent residual snap (`< 0.001 semitones`) avoids a long
asymptotic tail that the one-pole would never quite resolve. 0.001
semitone is well below pitch-discrimination threshold.

**First-note-from-rest snap**: PedalSH101 §6.1 pattern. When the env
was idle at trigger time, snap `_currentMidi` directly to the new
note in `TriggerNote`. Otherwise the first note slides up from
whatever stale `_currentMidi` value happened to be sitting there.

The pitch jumps between hops do produce a tiny phase discontinuity
at hop boundaries — but the OLA crossfade (§9) smooths these into a
continuous slide. This works for any musical glide time; very fast
slides (under one hop period, ~10 ms) would show stair-stepping, but
nobody dials in glide times that short.

---

## 13. Architecture limits worth surfacing in the README

Three intrinsic limits of N=2048, H=512 OLA synthesis, all worth
documenting so users don't try to fight them:

**~32 ms intrinsic OLA ramp.** The OLA buffer takes about 32 ms to
build up from one frame's worth of signal to the steady-state four
overlapping frames. Note onsets are floor-bound to this — even with
`AmpAttack=0`, the audible attack is ~32 ms. (The envelope hits unity
fast, but the OLA itself is still ramping.) Same for note-offs: a
hard `AmpRelease=0` cuts the env, but the OLA tail still rings for
~32 ms.

For percussive content this is too soft. The fix is a smaller FFT and
faster hop — N=512 H=128 gives ~3 ms ramp at the cost of 16× the
iFFT work.

**~21 ms inherent latency.** Half the FFT window. Acceptable for
slow-attack patches; soft for tight rhythmic playing. Same fix.

**~94 Hz max modulation rate.** Spectrum-domain parameters
(brightness, formant centre, etc.) update once per hop. Modulating
them faster than the hop rate gets quantized to hop boundaries.
Plenty for any musical envelope curve, but no use trying to do AM or
audio-rate FM with these parameters — they're slow controls, not
signals.

These three are coupled — they all derive from the same N, H choice.
You can't make one better without making the others worse (or paying
in CPU). Any v2 that wants to handle percussion needs to at minimum
support a different `(N, H)` mode for that use case.

---

## 14. Diagnostic methodology — bisect for an unknown hang

Captured separately from §11 because it generalizes. Any time a
machine builds and loads cleanly but hangs ReBuzz at runtime in a
way that doesn't surface a stack trace, the bisect pattern works:

1. **Save the hanging version.** Don't try to fix it directly.
2. **Save the most recent working version.** This is your "before"
   baseline.
3. **Bisect by feature, not by line.** Identify the chunks of code
   that differ between working and hanging. Re-add them one cluster
   at a time on top of the working baseline. Each test is a build,
   place into a song, press play.
4. **The first cluster that hangs is the suspect cluster.**
5. **Bisect again within that cluster** if it's more than a few lines.
6. **Once isolated, design a workaround.** Don't expect to understand
   the cause — sometimes the workaround is the deliverable and the
   root cause stays open.

The cost is N test cycles at ~30 seconds each, plus the editing time
between. Far cheaper than staring at a working-vs-hanging diff hoping
to spot the offender.

`DCWriteLine` debug output is unreliable for hangs — the UI thread
that consumes log messages is also the one that's deadlocked. For
this class of bug, file-based logging via `File.AppendAllText` (with
a lock for thread-safety) is the only thing that survives. See Core
§21 for the pattern.

---

## 15. File structure that worked for Pedal invFFT

```
PedalInvFFT/
├── PedalInvFFT.cs           ~390 lines  Machine, params, audio loop,
│                                         voice orchestration, mix
│                                         buffer, transport stop,
│                                         chord-delivery polling (§18)
├── Voice.cs                 ~280 lines  Per-voice state and rendering:
│                                         OLA buf, phases, envelopes,
│                                         glide, hop scheduling, RunHop
├── FFT.cs                    ~150 lines  Radix-2 in-place complex FFT
├── Envelope.cs               ~200 lines  Linear-attack, exp-decay/release
│                                         ADSR with forced-release fade
├── PedalInvFFT.NET.csproj
└── README.md
```

State worth listing explicitly — the fields that took the most
thought to get right. The split between machine-level (shared) and
voice-level (per-voice) is itself a design choice; see §16 for the
reasoning.

```csharp
// ── PedalInvFFTMachine: shared across voices ──────────────────────
readonly float[] _specRe = new float[FFT_SIZE];        // Re scratch
readonly float[] _specIm = new float[FFT_SIZE];        // Im scratch
readonly FFT     _fft    = new FFT(FFT_SIZE);          // shared FFT
readonly float[] _mixBuf = new float[16384];           // mix accumulator
readonly Voice[] _voices;                              // N_VOICES = 8

bool _wasPlaying = false;                              // transport edge

// Reflection-cached pvalues handle (§18)
IParameter _ownNoteParam = null;
ConcurrentDictionary<int,int> _ownNotePValues = null;

// ── Voice: per-voice state ────────────────────────────────────────
readonly float[]  _olaBuf    = new float[FFT_SIZE];    // OLA accumulator
readonly float[]  _phases    = new float[N_PARTIALS];  // per-partial phase
readonly Envelope _ampEnv    = new Envelope();         // sample-rate
readonly Envelope _brightEnv = new Envelope();         // hop-rate (§11)

int   _samplesUntilNextHop = 0;                        // hop schedule
int   _olaReadPos          = 0;                        // drain index
float _freqHz              = 440f;                     // recomputed/RunHop
float _currentMidi         = 60f;                      // continuous semis
float _targetMidi          = 60f;                      // glide target

bool _hasNoteOn       = false;                         // pending events
bool _hasNoteOff      = false;
byte _pendingBuzzNote = 0;

readonly PedalInvFFTMachine _machine;                  // for param reads
```

The two envelope instances per voice differ only in how they're
advanced, not in class or initialisation. The split between
sample-rate (amp) and hop-rate (brightness) update sites is the core
architectural choice that makes the per-voice DSP work cleanly —
keeping the per-sample drain loop minimal is why CPU and timing both
behave, and it's what made eight voices fit comfortably in the
budget.

---

## 16. Polyphony architecture — Voice class and shared scratch

v1.0 was monophonic. v2.0 lifted the per-voice DSP into a `Voice` class
and gave the machine an array of them. The migration was deliberately
staged: a structural refactor first (Voice class created, machine
delegates through `_voices[0]`, N_VOICES still 1) so behavior stayed
identical to v1.0, then a single two-line bump (N_VOICES = MaxTracks =
8) to actually enable polyphony. Each step was independently testable.
This is worth doing for any nontrivial refactor — bisecting "the
refactor broke the sound" against "the polyphony broke the sound" is
much harder when both happen in the same commit.

### 16.1 Track-to-voice mapping is fixed

Track index *is* voice index. `SetNote(value, track)` routes directly
to `_voices[track]`. There is no dynamic voice allocation, no
last-note-priority stealing, no "find an idle voice and use it" logic.
Polyphony comes from the user placing notes on multiple tracker
columns — the standard ReBuzz tracker convention, as in PedalM1 §1.

The cost of this simplicity is that hammering the same track faster
than the env can release retriggers the same voice every time, rather
than spreading new notes across spare voices. For the K5000 / pad /
lead use case Pedal invFFT targets, that's the right behavior — a
track *is* a melodic line and shouldn't borrow other tracks' voices.
A polyphonic synth aimed at fast keyboard playing would want a
dynamic allocator instead.

### 16.2 Shared scratch, per-voice OLA

The choice that took thought was which buffers to share and which to
make per-voice:

| Buffer            | Per-voice or shared? | Reason                              |
|-------------------|----------------------|-------------------------------------|
| `_specRe/_specIm` | shared               | Cleared at top of every RunHop      |
| `_fft`            | shared               | No state beyond read-only twiddles  |
| `_olaBuf`         | per-voice            | Holds residual ringing across hops  |
| `_phases`         | per-voice            | Continuous across hops (§6)         |
| `_ampEnv`         | per-voice            | Tracks per-voice NoteOn/Off         |
| `_brightEnv`      | per-voice            | Same                                |
| Glide state       | per-voice            | Each voice's pitch evolves alone    |

The shared scratch is safe because voices process strictly
sequentially within `Work()`. Each voice's `RunHop` clears the
scratch, fills it from its own state, inverse-transforms, and adds
the result to its own per-voice OLA. The scratch is "owned" by
whichever voice is currently calling `RunHop`; no cross-talk possible
without parallelism, which the architecture doesn't have.

If voices were ever parallelized (one thread per voice, mixing in a
join step), the scratch would need to become per-voice too — or each
voice would need its own short-lived stack scratch. At 8 voices the
sequential cost is comfortable so this hasn't been worth doing.

### 16.3 Mix buffer accumulation

Voices accumulate into a `float[] _mixBuf` rather than writing
directly to the `Sample[] output`. After the voice loop, a final
pass converts each float to a `Sample(s, s)`. This pattern adds one
buffer copy compared to writing direct, but it sidesteps the awkward
`Sample` struct semantics for accumulation (`output[i] += s` doesn't
compile cleanly when `output[i]` is a `Sample` and `s` is a float).
The cost is negligible — `Array.Clear(_mixBuf, 0, n)` + n adds + n
struct constructions, all linear in n.

`_mixBuf` is allocated at machine construction at 16384 floats —
4× the realistic ReBuzz max buffer of ~4096 samples. A guard at the
top of `Work()` returns silence if `n` exceeds the buffer size,
which would only happen if ReBuzz ever started passing larger
chunks than current 1819-preview does. No audio-thread allocations.

### 16.4 Memory and CPU budget

Per-voice state at FFT_SIZE=2048, N_PARTIALS=16:
- `_olaBuf`: 8 KB
- `_phases`: 64 B
- Two envelope instances: ~200 B
- Misc scalars: ~50 B
- Total: ~8.5 KB per voice

Machine-level shared:
- `_specRe`/`_specIm`: 16 KB total
- `_mixBuf`: 64 KB
- FFT twiddle tables: ~32 KB
- Total: ~112 KB

Eight voices × 8.5 KB + 112 KB ≈ 180 KB per machine instance. Well
under any concern.

CPU at 48 kHz with 8 active voices: roughly 5% of one modern desktop
core, dominated by the per-voice FFTs (one per hop per voice = ~750
FFTs per second). Half-active voices (Release tail) cost the same as
fully active ones; only Idle and Sustain-at-zero voices are skipped
entirely (§17).

---

## 17. IsActive vs IsAudible — silent-path gating in a polyphonic synth

The v0.x silent-fast-path skipped the entire RunHop / drain machinery
when `_ampEnv.IsActive` was false (envelope at Idle). That worked for
v1.0 because the only way to silence the synth was to fully release
the note. For v2.0 with multiple voices, the same predicate is wrong
in a percussive-patch case that v1.0 had silently been wasting CPU on:

> Patch with `Amp Sustain = 0` and short `Amp Decay`. Note triggers,
> Decay drops level toward 0, env transitions to Sustain stage with
> level pinned at 0. `IsActive` returns true (stage isn't Idle), so
> RunHop keeps running and producing OLA content that gets multiplied
> by 0 at the drain stage. Pure wasted work until NoteOff arrives.

The fix splits the predicate. Two distinct properties, used at
different sites:

```csharp
public bool IsActive => _ampEnv.IsActive;        // any non-Idle stage

public bool IsAudible
{
    get
    {
        if (!_ampEnv.IsActive) return false;
        if (_ampEnv.CurrentStage == Envelope.Stage.Sustain
            && _ampEnv.Level <= 0f)
            return false;
        return true;
    }
}
```

| Stage   | Level     | IsActive | IsAudible | Renders? |
|---------|-----------|----------|-----------|----------|
| Idle    | 0         | false    | false     | no       |
| Attack  | 0 → 1     | true     | true      | yes      |
| Decay   | 1 → sus   | true     | true      | yes      |
| Sustain | > 0       | true     | true      | yes      |
| Sustain | **0**     | true     | **false** | **no**   |
| Release | falling   | true     | true      | yes      |

`IsAudible` gates the silent fast-path and the per-voice render
decision. `IsActive` is kept for transport-stop iteration: a voice in
Sustain-at-zero is parked but not Idle, so `ForcedRelease` should
still fire on it — the env's internal Idle check makes the call a
no-op anyway, but iterating on `IsActive` makes intent explicit.

The Attack/Decay/Release stages always render, even at level zero,
because their level is *changing* — Release entered at level 0 needs
exactly one `Process` call to transition to Idle, and the gate has
to let that one call through. Sustain is the only "parked" state
where the env can sit at zero without anything happening, which is
why it's the only stage with a level check.

### 17.1 Why this is the right resolution

The naïve fix would be to make Sustain-at-zero transition to Idle
directly. That breaks NoteOff semantics — a note held with sustain=0
should still respect the user's hold, in case `Bright Amount` is
positive and the brightness env is still doing something audible
(though the user can't currently hear it, the state is still
"holding"). The split predicate keeps the state machine honest: the
voice is still active, just not currently producing output.

It also generalizes — if v3 ever adds a per-voice volume parameter,
`IsAudible` would extend to "amp env active AND voice volume > 0".
Centralizing the audibility logic in one property keeps that growth
local.

### 17.2 What about brightness env when voice render is skipped?

When a voice is gated out by `IsAudible`, neither RunHop nor the
drain loop runs, so `_brightEnv.Process` doesn't tick either. The
brightness env state freezes at whatever it was last left at.

This is fine in the Sustain-at-zero case because the user can't hear
the voice anyway. When NoteOff drains in via `DrainEvents`, amp env
transitions Sustain→Release at level 0; one Process call in Release
detects level < threshold and snaps to Idle; brightness env
transitions Sustain→Release in parallel and ticks normally during
the (one-sample-long) Release rendering. By the time anything could
become audible again, the brightness env has caught up. Worst case
is a barely-perceptible discontinuity at the start of the next
NoteOn, well below noise floor for any reasonable patch.

If a future patch design genuinely needs the brightness env ticking
during a silent Sustain (e.g. brightness env in Decay while amp env
is Sustain-at-zero, where the user expects the brightness env to
finish its decay even though no audio is heard), this gating will
need rethinking. Hasn't come up.

---

## 18. Chord delivery — the Core §14 workaround in this synth

Polyphonic playback from pattern data requires the Core §14 polling
workaround. Without it, two notes on the same row across two tracks
collapse to last-track-only because of the `parametersChanged`
dictionary collision. Pedal invFFT's specific implementation:

### 18.1 Lazy reflection bootstrap

Reflection lookup happens lazily on first `SetNote` call rather than
in the constructor. `host.Machine.ParameterGroups` is populated by
ReBuzz's `CreateParameterDelegates()` *after* the machine
constructor runs (Core §15), so a constructor-time lookup returns
`null`. By the time any note is being delivered, the groups exist:

```csharp
IParameter _ownNoteParam = null;
ConcurrentDictionary<int,int> _ownNotePValues = null;

void TryInitPValues()
{
    try
    {
        if (_ownNoteParam == null)
        {
            var pg = host?.Machine?.ParameterGroups;
            if (pg == null || pg.Count < 3) return;
            _ownNoteParam = pg[2].Parameters.FirstOrDefault(
                p => p?.Type == ParameterType.Note);
        }
        if (_ownNoteParam == null || _ownNotePValues != null) return;

        var fi = _ownNoteParam.GetType().GetField("pvalues",
            BindingFlags.NonPublic | BindingFlags.Instance);
        if (fi != null)
            _ownNotePValues = fi.GetValue(_ownNoteParam)
                as ConcurrentDictionary<int,int>;
    }
    catch { /* leave fields null; subsequent SetNote calls skip polling */ }
}
```

The `try/catch` swallows any reflection failure silently. If the
pvalues field gets renamed in a future ReBuzz, polling stops working
and chord rows degrade to last-track-only — but the synth keeps
running, monophonic per track. Better than throwing and breaking
audio.

### 18.2 Polling pattern in SetNote

```csharp
public void SetNote(Note value, int track)
{
    // Handle the firing track normally.
    byte v = value.Value;
    if (v == Note.Off)        _voices[track].QueueNoteOff();
    else if (v != 0)          _voices[track].QueueNoteOn(v);

    // Recover sibling tracks.
    if (_ownNotePValues == null) TryInitPValues();
    if (_ownNotePValues != null)
    {
        int noVal = _ownNoteParam.NoValue;
        for (int t = 0; t < _voices.Length; t++)
        {
            if (t == track) continue;
            if (_ownNotePValues.TryGetValue(t, out int pv)
                && pv != noVal && pv != 0)
            {
                if (pv == Note.Off) _voices[t].QueueNoteOff();
                else                _voices[t].QueueNoteOn((byte)pv);
            }
        }
    }
}
```

Three values to filter on:
- `pv == 0` — `NoValue` for `Note` type. No event this row on this
  track. Skip.
- `pv == 255` — `Note.Off`. Queue NoteOff on this voice.
- Anything else (1..156) — real note. Queue NoteOn with the byte.

The `pv != noVal && pv != 0` double-check is defensive. NoValue is
hardcoded to 0 for Note type (Core §3), so the two conditions are
equivalent today, but keeping both means the code stays correct if
ReBuzz ever changes the convention.

### 18.3 Why no `!HasNoteOn && !HasNoteOff` guard

PedalM1's polling has a guard checking the per-voice pending flags
before overwriting (PedalM1 §6). PedalInvFFT's polling doesn't —
because `QueueNoteOn` and `QueueNoteOff` are idempotent (they just
set a flag and a byte), re-queuing the same value is a no-op.

The guard would matter if SetNote could fire multiple times within
a single Tick for the Note parameter, e.g. across sub-tick
boundaries. It can't — `parametersChanged` collapses all per-tick
writes for a given parameter into a single setter call. The polling
loop runs exactly once per tick where any note-related setter
fires, and re-running the same poll within that tick produces
identical results, so idempotency is enough.

If PedalInvFFT ever grows additional track parameters (per-track
velocity, per-track detune, etc.), each one would need its own
`_ownXxxParam` + `_ownXxxPValues` pair, and SetNote (or all
relevant setters) would extend the polling loop accordingly. At
that point the `!HasXxx` guards become useful for the same reason
they do in PedalM1: the redundancy pattern (any setter firing
recovers all the others) means the same sibling can be
re-recovered from multiple polling sites within the same tick,
and idempotency arguments only hold for individually idempotent
operations. Cross-parameter consistency is a different question.
