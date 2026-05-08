# ReBuzz Managed Machine Development — Pedal invFFT Addendum

Source: ReBuzz 1819-preview source code + Pedal invFFT v1.0 build (a
managed C# generator: monophonic K5000-inspired additive synth using
inverse FFT with overlap-add resynthesis, 16 partials, with amp and
brightness ADSRs, static spectrum shaping, formant filter, and glide).

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
├── PedalInvFFT.cs           ~480 lines  Machine, params, audio loop,
│                                         RunHop, glide, transport stop
├── FFT.cs                    ~150 lines  Radix-2 in-place complex FFT
├── Envelope.cs               ~200 lines  Linear-attack, exp-decay/release
│                                         ADSR with forced-release fade
├── PedalInvFFT.NET.csproj
└── README.md
```

State worth listing explicitly — the fields that took the most
thought to get right:

```csharp
// Synthesis state (per-hop and per-sample)
readonly float[]  _specRe = new float[FFT_SIZE];        // Re spectrum
readonly float[]  _specIm = new float[FFT_SIZE];        // Im spectrum
readonly float[]  _olaBuf = new float[FFT_SIZE];        // OLA accumulator
readonly float[]  _phases = new float[N_PARTIALS];      // per-partial phase

// Hop scheduling
int   _samplesUntilNextHop = 0;          // 0 ⇒ run hop on next Work()
int   _olaReadPos          = 0;          // drain index into _olaBuf

// Pitch and glide
float _freqHz       = 440f;              // recomputed each RunHop
float _currentMidi  = 60f;               // continuous semitones
float _targetMidi   = 60f;               // glide target

// Envelopes
readonly Envelope _ampEnv    = new Envelope();  // sample-rate, drain loop
readonly Envelope _brightEnv = new Envelope();  // hop-rate, batched (§11)

// Pending events (drained at top of Work — PedalSH101 §6.3)
bool _hasNoteOn       = false;
bool _hasNoteOff      = false;
byte _pendingBuzzNote = 0;

// Transport edge detection (Core §27)
bool _wasPlaying = false;
```

The two envelope instances differ only in how they're advanced, not in
class or initialisation. The split between sample-rate and hop-rate
update sites is the core architectural choice that makes the whole
machine work cleanly — keeping the per-sample drain loop minimal is
why CPU and timing both behave.
