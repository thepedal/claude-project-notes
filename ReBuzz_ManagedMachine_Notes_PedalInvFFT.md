# ReBuzz Managed Machine Development — Pedal invFFT Addendum

Source: ReBuzz 1819-preview source code + Pedal invFFT v2.2 build (the
v2.1 polyphonic K5000-inspired iFFT/OLA additive synth with stretch
and animation, plus v2.2's per-voice key-synced LFO and six routing
destinations — Pitch, Brightness, Stretch, Volume, Formant Centre,
and Animation Depth — closing the modulation infrastructure).

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
readonly float[]   _olaBuf     = new float[FFT_SIZE];   // OLA accumulator
readonly float[]   _phases     = new float[N_PARTIALS]; // per-partial phase
readonly float[]   _animPhases = new float[N_PARTIALS]; // animation phase (§20)
readonly Envelope  _ampEnv     = new Envelope();        // sample-rate
readonly Envelope  _brightEnv  = new Envelope();        // hop-rate (§11)

// LFO state (§21). Sync offset is per-voice random for organic chord
// LFO behavior; private RNG seeded uniquely per voice for S&H values.
readonly float  _lfoSyncOffset;
readonly Random _rng;
float _lfoPhase;
float _lfoSnH;

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

---

## 19. Stretch — inharmonic partials via power-curve warping

v2.1 step 1. A single parameter (`Stretch`, 0..127, default
64=neutral) warps the partial-frequency formula from the harmonic
series to arbitrary inharmonic ratios. Implementation is one line in
`Voice.RunHop`:

```csharp
// Was: partialFreq = _freqHz * (p + 1)
float stretchExp  = 1f + (_machine.Stretch - 64) / 64f * 0.3f;
float partialFreq = _freqHz * MathF.Pow(p + 1, stretchExp);
```

### 19.1 Power-curve vs piano-physics

The two natural formulas for inharmonic partials are:

- **Piano-string** (Fletcher): `f_n = n · f₁ · √(1 + B · n²)` —
  derives from string-stiffness physics, gives the asymptotically
  progressive stretch real piano strings exhibit. B ≈ 0.0001 to
  0.001 for typical strings.
- **Power-curve**: `f_n = (n)^k · f₁` — pure mathematical scaling,
  no physical motivation but always well-defined for any k > 0.

Pedal invFFT uses power-curve. Reasons in priority order:

1. **Symmetric**. k > 1 stretches, k < 1 compresses, both
   well-defined. The piano formula needs clamping or branch logic
   for negative B (compression) since `1 + B·n²` can go negative
   for high n.
2. **Smooth parameter sweep**. Linear in k gives perceptually smooth
   parameter response. Piano B has a nonlinear relationship to
   audible inharmonicity — small B values are inaudible, large B
   values explode quickly.
3. **Predictable at extremes**. At k = 1.3 the 16th partial sits
   at ~28× the fundamental, dramatic but musical. The piano formula
   at comparable inharmonicity goes nonlinear in ways that are
   harder to dial in.

The trade-off: power-curve doesn't reproduce specific real-world
inharmonic timbres (piano stretch, bell ratios, gamelan tunings)
directly. A future "Stretch Mode" parameter (README #6) could add
piano, bell, gamelan, etc. as preset modes with their own formulas,
keeping power-curve as the smooth-sweep default.

### 19.2 Default preserves v2.0 sound

At Stretch=64, `(_machine.Stretch - 64) / 64f * 0.3f = 0`, so
stretchExp = 1.0 exactly. `MathF.Pow(p+1, 1.0)` returns p+1 within
float precision. v2.0 patches loaded into a v2.1 build sound
bit-near-identical at default Stretch.

### 19.3 CPU cost

One `Pow` call per partial per hop per voice. At 8 voices × 16
partials × 94 hops/sec ≈ 12 000 Pow calls/sec. `MathF.Pow` is
~50 ns on modern x64, so ~600 µs/sec total — well under 0.1% CPU.
The exponent is hoisted out of the partial loop (computed once per
RunHop), so the inner-loop cost is just the Pow itself.

### 19.4 Interactions

- **Brightness, Tilt, Balance** all act on the partial amplitudes;
  they don't care about the partial frequencies. Compose cleanly.
- **Formant filter** operates in absolute Hz, so a stretched partial
  passes through the same formant peak it would have at a different
  harmonic position — gives unusual vowel-with-attitude textures
  at moderate stretch.
- **Pitch (fundamental)**: unaffected. The first partial (`p=0`,
  amplitude `1/(0+1) = 1`) sits at `_freqHz · 1^k = _freqHz`
  regardless of Stretch. The played note is always the played note;
  only the spectrum around it changes.
- **Glide**: orthogonal. Glide moves `_freqHz`; Stretch warps the
  partial structure built on top of it.
- **Animation (§20)**: orthogonal. Animation modulates per-partial
  amplitudes; Stretch sets per-partial frequencies. Combined, you
  get bell timbres with their own slow shimmer.

---

## 20. Harmonic micro-animation — per-partial amplitude shimmer

v2.1 step 2. Two parameters (`Anim Rate`, `Anim Depth`) plus a
per-voice `_animPhases` array drive slow per-partial amplitude
modulation. Each partial wobbles at its own rate, with the rate
spread chosen to keep partials from synchronizing. Default Anim
Depth = 0 turns the feature off entirely, preserving v2.0 sound.

### 20.1 Why amplitude (not phase, not frequency)

Three candidate modulation targets for per-partial animation:

| Target    | Effect                               | Notes                       |
|-----------|--------------------------------------|------------------------------|
| Amplitude | Each partial pulses softly           | Most audible, cleanest fit  |
| Phase     | Partials drift in alignment          | Subtle; mostly inaudible    |
| Frequency | Partials detune slightly             | Fights iFFT/OLA crossfade   |

Amplitude wins because it's the most directly audible at hop-rate
update speeds and integrates with the existing per-partial amp
chain without any architectural friction. Phase animation is
reserved as a future feature (README #3) — different effect, not a
substitute. Frequency animation is theoretically interesting but
interacts with the per-frame OLA phase tracking in ways that need
careful handling.

### 20.2 Per-partial state

Per voice: a `float[N_PARTIALS] _animPhases` array, randomized at
construction with phases drawn independently from the spectral
phase array `_phases`. 64 bytes per voice on top of v2.0's per-voice
footprint (~8.5 KB). Eight voices × 64 bytes = 512 bytes additional
state per machine instance — invisible.

The phases are explicitly *not* reset on note retrigger. Animation
is meant to be a slow background process, decoupled from note
timing; resetting on retrigger would create a perceptible "starting
state" each note that defeats the point of the feature (which is
making sustained tones feel unmoored from event-driven structure).
Initialized once at voice construction is enough.

### 20.3 Per-partial rate spread

Each partial advances at base rate × (0.7 + p × 0.04), giving rates
in [0.7×, 1.3×] of the base, linearly spread across the 16 partials.
The choice was between:

- **Fixed irrational ratios** — pre-computed table of 16 numbers
  with no rational relationships. Maximally non-synchronizing but
  involves hand-designing the table.
- **Linear spread** — simple, predictable. Partial 15's rate is
  1.86× partial 0's. Not strictly irrational, but the ratios that
  appear (1.86, 1.71, 1.57, …) are non-integer enough that practical
  sync periods are minutes-to-hours rather than seconds.

Linear spread won on simplicity. The non-synchronizing property
needs to hold over musical timescales (~10–30 seconds of sustain),
and the spread is plenty wide for that. Subjectively, the spectrum
keeps evolving over long pads with no audible "I've heard this
exact pattern before" moments.

### 20.4 Why default-off

Anim Depth = 0 means the feature is entirely off — the `if (animOn)`
fast-path skips both the per-partial Sin and the phase advance.
Default 0 was chosen for two reasons:

1. **Preserve v2.0 sound at default values.** v2.0 patches loaded
   into v2.1 sound identical until the user reaches for animation.
   Same principle as Stretch=64.
2. **Make discovery deliberate.** A subtle-on-by-default would let
   the feature work but at a level too gentle for new users to
   notice. Off-by-default forces engagement with the parameter,
   which produces deliberate sound design rather than wondering
   why patches sound slightly different.

### 20.5 CPU cost when active

Per voice per hop, when Anim Depth > 0:

- 16 × `MathF.Sin` (~20 ns each) for the modulation factor
- 16 × float arithmetic for phase advance and 2π wrap

At 8 voices × 94 hops/sec × 16 partials, that's ~12 000 Sin/sec —
well under 0.1% CPU. Hop-rate updates are inherently smooth in
audio because the OLA averages across 4 hops, so the effective
modulation is a hop-rate carrier mechanically smoothed by the
overlap.

When Anim Depth = 0 the entire branch is skipped — zero overhead
at default settings.

### 20.6 Interactions

- **Stretch (§19)**: orthogonal and complementary. Stretch sets
  where partials sit in frequency; Animation sets how their
  amplitudes evolve in time. Bell timbres with animation get their
  own slow shimmer, mimicking real bell physics where decoupled
  modes interact.
- **Brightness, Tilt, Balance, Formant**: all multiply into amp
  before animation. Order is 1/n base → spectrum shaping →
  animation → deposit. So animation's ±50% swing is relative to
  whatever amp the spectrum-shape modifiers produced, which is the
  natural composition.
- **Brightness envelope**: per-voice and at hop rate. A note with
  brightness env *and* animation engaged has two slow spectrum
  modulations layered — env shaping the cutoff, animation rippling
  individual partial amplitudes around it. They combine naturally.
- **Per-voice independence**: each voice has its own `_animPhases`
  array with independent random initial offsets. Polyphonic chords
  get genuinely different shimmer per voice — chord textures feel
  richer than just "the same patch played at three pitches".

### 20.7 Connection to the K5000 lineage

Kawai's K5000 had a "harmonic LFO" that modulated per-harmonic
amplitudes for similar reasons. v2.1's animation is a small subset
of that — single sine per partial, fixed rate spread — but it
captures the essential character. A future expansion could add
multiple animation modes (random walk, multi-rate sine sum, sample
& hold) via an Anim Mode parameter, getting closer to what the
K5000 offered without fundamentally changing the architecture.

---

## 21. LFO architecture — per-voice, key-synced, hop-rate

v2.2 step 1. Per-voice LFO state, computed once per RunHop, with the
resulting value reused across however many routing destinations are
active that hop. Three new parameters in step 1: LFO Rate, LFO Shape,
LFO Pitch (the first routing destination).

### 21.1 Per-voice rather than shared

Each Voice owns its own LFO state (phase, S&H value, sync offset,
private RNG). The alternative — a single machine-level LFO whose value
is used by all voices — was rejected because it makes polyphonic
chords feel mechanical: every voice modulates in lockstep, which is
exactly what a multi-voice synth is meant to avoid. Per-voice LFOs
let chords feel like an ensemble rather than one voice multiplied.

The cost is small: three floats and a Random per voice (~50 bytes),
plus one extra Sin call per voice per hop. At 8 voices × 94 hops/sec,
~750 LFO computes per second — well under any concern.

### 21.2 Key-sync to a per-voice random offset, not zero

The LFO key-syncs on every NoteOn (resets phase to a target value).
The conventional choice for the target is 0 — every note starts at
LFO phase 0. For polyphony this creates a problem: when a chord
strikes, all voices reset to phase 0 simultaneously, so the chord
voices' LFOs are perfectly synchronized for at least one cycle. With
vibrato active, the entire chord wobbles in unison — defeating the
per-voice independence the architecture is supposed to give.

The fix is a small but important variation: each voice picks a random
phase at construction (`_lfoSyncOffset`), and NoteOn resets `_lfoPhase`
to *that* value rather than to 0. The eight voices in a chord still
key-sync, but they sync to eight different offsets, so the LFO phases
are spread across the cycle. Chords with vibrato sound like an
ensemble; chords with LFO Bright sweeps shimmer asymmetrically; chords
with S&H stretch each pick their own random tuning per cycle.

This is meaningfully better than a free-run LFO would be, too: free-run
loses the predictability of "the LFO is at phase X right after each
NoteOn", which matters for things like ramp shapes and S&H. The
random-offset key-sync keeps that predictability per voice while
breaking it across voices.

### 21.3 Four shapes via parameter quarter-banding

LFO Shape is a single 0..127 int parameter quarter-banded to four
discrete shapes:

```csharp
int shapeBand = _machine.LFOShape / 32;     // 0..3
switch (shapeBand)
{
    case 0: return MathF.Sin(_lfoPhase);
    case 1: return 2f * MathF.Asin(MathF.Sin(_lfoPhase)) / MathF.PI;
    case 2: return MathF.Sin(_lfoPhase) >= 0f ? 1f : -1f;
    default: /* S&H */  ...
}
```

| Range  | Shape           | Cost                       | Notes                      |
|--------|-----------------|----------------------------|----------------------------|
| 0..31  | Sine            | 1 Sin                      | Default; smooth            |
| 32..63 | Triangle        | 1 Sin + 1 Asin             | Linear ramps; same zeros   |
| 64..95 | Square          | 1 Sin + 1 compare          | Hard switching             |
| 96..127| Sample & Hold   | 1 RNG call on phase wrap   | Random stepping            |

Triangle uses `2·asin(sin(φ))/π` rather than a piecewise abs formula
because it produces zero crossings at exactly the same phase points
as the sine — which makes mixing or A/B'ing different shapes feel
consistent. The extra Asin call is cheap.

S&H samples a fresh value from the per-voice Random whenever the
phase wraps a cycle. The held value is also refreshed at NoteOn so
retriggered notes get a fresh random rather than whatever was held
from the previous note's last cycle.

### 21.4 Rate range capped at 10 Hz

Rate is log-mapped from 0.1 Hz to 10 Hz, with default 64 giving
~1 Hz. The cap at 10 Hz is below the conventional 20 Hz LFO ceiling
because the LFO is computed at hop rate (~94 Hz at 48 kHz). At 10 Hz
that's ~9 samples per LFO cycle — smooth. At 20 Hz it would be
~5 samples — audible stair-stepping, especially with hard shapes
(square, S&H). Better to cap below the perceptible-quantization
threshold than to expose rates that produce artefacts.

If audio-rate modulation is ever wanted (FM-style effects), it would
need a per-sample rather than per-hop compute path — different
infrastructure.

### 21.5 Pitch modulation applied to MIDI, not Hz

The first routing destination, LFO → Pitch:

```csharp
float pitchModSemi  = lfoVal * (_machine.LFOPitch - 64) / 64f * 3f;
float effectiveMidi = _currentMidi + pitchModSemi;
_freqHz = 440f * MathF.Pow(2f, (effectiveMidi - 69f) / 12f);
```

The modulation is applied to `_currentMidi` (continuous semitones)
without mutating it, then `_freqHz` is computed from the modulated
value. This composes cleanly with glide: glide tracks `_currentMidi`
via the per-hop one-pole, the LFO wobbles around it, and `_freqHz`
falls out of the combined value. Vibrato + glide does the right
thing — the centre of the vibrato follows the slide.

Doing this in MIDI space (one Pow call) rather than Hz space (would
need `_freqHz *= MathF.Pow(2f, mod/12f)` after the original Pow) keeps
the per-hop math simple.

---

## 22. LFO routing destinations — pattern and per-destination notes

v2.2 step 2. Five additional routing destinations beyond Pitch, all
following the same bipolar-depth pattern.

### 22.1 The shared routing pattern

Each destination has a depth parameter `LFO X`, defaulting to 64 (no
modulation). The modulation amount is computed identically for all
destinations:

```csharp
float xMod = lfoVal * (_machine.LFOX - 64) / 64f * MAX_SWING;
```

`(LFOX - 64) / 64f` gives a coefficient in `[-1, +1]` from the bipolar
depth knob (negative below 64 inverts the modulation direction, positive
above 64 sends it the natural way). Multiplied by `lfoVal` (also
`[-1, +1]`) gives a normalized modulation in `[-1, +1]`. The
destination-specific `MAX_SWING` scales it to the destination's natural
units.

The MAX_SWING values:

| Destination     | MAX_SWING | Rationale                                           |
|-----------------|-----------|-----------------------------------------------------|
| LFO Pitch       | 3         | ±3 semitones — wide enough for siren, narrow
                                 enough to dial subtle vibrato precisely             |
| LFO Bright      | 63        | Full sweep in 0..127 param space                    |
| LFO Stretch     | 32        | Half-sweep — full would push to extremes that
                                 aren't always musical at the LFO rate               |
| LFO Volume      | 0.5       | ±50% multiplicative — true tremolo without
                                 silencing the voice at full deflection              |
| LFO Formant     | 63        | Full sweep in 0..127 param space                    |
| LFO Anim        | 63        | Full sweep — lets LFO turn animation on/off when
                                 base depth is near 0, or wobble it in mid-range     |

All depth knobs default to 64. With LFO Pitch at 64 and the other five
also at 64, the LFO advances internally but routes nowhere — default
sound is identical to v2.1.

### 22.2 Where each destination applies in RunHop

Most destinations modulate parameter values in their natural parameter
spaces, then derived quantities are computed from the modulated value.
This keeps the modulation perceptually even — for example, modulating
Formant Centre in param space (before the log-to-Hz mapping) gives
constant-octave-rate sweeps regardless of base centre frequency.

```csharp
// Pitch — modulates _currentMidi (semitones)
float pitchModSemi  = lfoVal * (LFOPitch - 64) / 64f * 3f;
float effectiveMidi = _currentMidi + pitchModSemi;
_freqHz = 440f * MathF.Pow(2f, (effectiveMidi - 69f) / 12f);

// Brightness — modulates the param value, then clamp + use
float brightLfoMod    = lfoVal * (LFOBright - 64) / 64f * 63f;
float effectiveBright = clamp(Brightness + brightEnvMod + brightLfoMod, 0, 127);

// Stretch — modulates the param value, then derive the exponent
float stretchLfoMod    = lfoVal * (LFOStretch - 64) / 64f * 32f;
float effectiveStretch = Stretch + stretchLfoMod;
float stretchExp       = 1f + (effectiveStretch - 64) / 64f * 0.3f;

// Formant Centre — modulates the param value, then log-map to Hz
float formantLfoMod   = lfoVal * (LFOFormant - 64) / 64f * 63f;
float effectiveCentre = clamp(FormantCentre + formantLfoMod, 0, 127);
formantCenterHz       = 100f * MathF.Pow(60f, effectiveCentre / 127f);

// Anim Depth — modulates the param value, then the existing anim path
//              picks up the modulated value
float animLfoMod   = lfoVal * (LFOAnim - 64) / 64f * 63f;
int   animDepthInt = clamp(AnimDepth + animLfoMod, 0, 127);

// Volume — multiplicative; applied per-partial in the deposit loop
float volMod = 1f + lfoVal * (LFOVolume - 64) / 64f * 0.5f;
// ...then in partial loop: amp *= volMod;
```

LFO Volume is the one that modulates a derived quantity (per-partial
amp in the deposit loop) rather than a parameter value. It applies
uniformly across all partials, so the result is a pure tremolo
amplitude swing rather than a spectrum-shape change. Multiplicative
rather than additive because volume is naturally multiplicative — at
full negative deflection (LFOVolume=0) when the LFO is at +1, the
multiplier becomes 0.5 (half volume), and at LFO=-1 it becomes 1.5
(50% boost).

### 22.3 Why parameter-space modulation rather than derived-space

For Brightness, Stretch, Formant Centre, and Anim Depth, the modulation
is applied in the *parameter* space (0..127), not in the derived
quantity (e.g. Hz, exponent). Reasons:

- **Perceptual evenness.** A given LFO depth produces the same
  perceptual range regardless of base setting. ±63 in Formant Centre
  param space is always ~6 octaves of sweep, regardless of where the
  centre is parked. ±63 directly in Hz would sweep barely an octave at
  high settings and 6 octaves at low — same numerical depth, very
  different musical effect.
- **Clamping is straightforward.** Param values clamp to [0, 127]
  trivially. Derived quantities have weirder valid ranges.
- **It matches the existing parameter-change semantics.** Twisting the
  Brightness knob and modulating it via LFO produce the same kind of
  sweep — just one is automated.

The exception is LFO Pitch, which modulates `_currentMidi` directly
(semitone space) rather than the (nonexistent) "MIDI parameter". This
is fine because semitone space *is* the perceptual space for pitch.

### 22.4 LFO Anim — modulating the modulation

The most distinctive of the destinations is LFO → Anim Depth. With
base Anim Depth near 0 and LFO Anim active, the LFO pushes Anim Depth
between 0 (animation off) and ±63 (moderate animation). The clamp to
[0, 127] means negative excursions are absorbed at 0, so the LFO
effectively switches animation on for half the cycle and off for the
other half. This produces "shimmer pulsing in and out" textures that
nothing else in the synth's surface can do.

With base Anim Depth around 64 and LFO Anim active, the LFO wobbles
animation between roughly 0 and 127 — so animation is always on but
its intensity breathes. Different effect, also useful.

### 22.5 CPU cost of full modulation

When all six destinations are at full deflection, per voice per hop:

- 1 LFO compute (Sin + maybe Asin or compare; one branch)
- 6 multiply-add modulation computations
- 1 extra Pow (for pitch — though there'd be one anyway)
- 1 clamp per modulated parameter

At 8 voices × 94 hops/sec, ~750 LFO + ~4500 modulation ops per second.
Lost in the noise of the partial deposit loop, which dominates the
per-voice cost.

---

## 23. The 64-preset bank — design and conventions

v2.2 ships with `PedalInvFFT_Presets.prs.xml`, 64 presets generated
from a Python script (`gen_presets.py`, kept in source but not
deployed per Build §3.4). The bank covers:

| Category    | Count | Purpose                                              |
|-------------|-------|------------------------------------------------------|
| Pad         | 10    | Slow attacks, sustained, brightness env, mild anim  |
| Lead        | 8     | Fast attack, presence, vibrato/wah variants          |
| Pluck       | 8     | Short attack/decay, percussive                       |
| Bell        | 10    | Heavy stretch, brightness env, anim/LFO variants     |
| Bass        | 4     | Low/mid spectrum, high partials rolled off           |
| Vox         | 6     | Heavy formant, vowel character + animations          |
| Anim        | 10    | Animation prominent, often LFO-modulated             |
| FX          | 8     | Extreme settings, demonstrate feature interactions   |

### 23.1 Preset naming convention

Each preset is named `<Category> - <Description>`, e.g.
`Pad - Soft Strings`, `Lead - Vibrato Sax`, `Bell - Crystal SH`. The
hyphen-with-spaces form (rather than colons) is XML-safe per Core §28
and gives a clean alphabetical sort within ReBuzz's preset menu —
all Pads cluster together, all Bells cluster together, etc.

### 23.2 Sparse override pattern

The generator script defines each preset as a dict of name→value
overrides, with unspecified parameters defaulting to the machine's
`DefValue` (tracked separately in a `DEFAULTS` dict that mirrors the
machine's source). This keeps preset definitions short and focused
on what matters — a "Pad - Default" preset is six lines:

```python
"Pad - Default": {
    "Volume": 56, "Amp Attack": 60, "Amp Release": 80,
    "Brightness": 80, "Bright Decay": 95, "Bright Amount": 92,
    "Anim Depth": 18,
},
```

The XML emitter expands each preset to the full 28-parameter form
that ReBuzz expects, filling in defaults for omitted keys.

### 23.3 Forward compatibility with new parameters

Build §3.3's append-only rule means new parameters (in future v2.3+
work) get appended to the end of the declaration list and PARAM_INDEX.
Existing preset overrides — keyed by name — still resolve to the same
indices. Presets that don't mention the new parameter get its
machine-level default, so old presets keep working without any
edits.

The implication for v2.2: when v2.3 lands with (say) spectral morph,
this entire 64-preset bank stays valid. Some presets might benefit
from being updated to use the new feature, but none will break.

---

## 24. Defensive track-index bounds checking — external writes can exceed MaxTracks

v2.2.1 hot-fix. PedalInvFFT crashed with `IndexOutOfRangeException` in
`SetNote` when Pedal Muter was used to mute the machine. The crash
revealed an assumption that doesn't hold in practice: that the `track`
parameter delivered to a setter is always in `[0, MaxTracks-1]`.

### 24.1 What actually arrives at the setter

`MaxTracks` in `MachineDecl` is an upper bound advertised to the host,
not a hard contract. ReBuzz routes parameter writes through
`ManagedMachineHost.SetParameterValue(index, track, value)`, and the
`track` argument is whatever the *writer* supplied — there's no
clamp at the host layer. Any control machine driving your
parameters can write to track values outside your declared range:

- **Pedal Muter** (PedalMuter notes — `Track-scan fallback in
  FlushStaleVoicesOnMachine = 16`) scans tracks 0..15 when injecting
  Note.Off=255 to flush stale voices on a target machine, regardless
  of the target's actual MaxTracks. The 16-track ceiling covers
  polyphonic synths that haven't bumped their `TrackCount` to match
  their MaxTracks (Core §19). For PedalInvFFT (MaxTracks=8), tracks
  8..15 are out-of-range writes.
- **Pattern editor**, **MIDI converters**, and other tools generally
  write within your declared range, but there's no protocol-level
  guarantee.

The setter must therefore treat `track` as untrusted input and
range-check before using it as an array index.

### 24.2 The minimal fix

Guard the firing-track write only:

```csharp
public void SetNote(Note value, int track)
{
    if (track >= 0 && track < _voices.Length)
    {
        byte v = value.Value;
        if (v == Note.Off)        _voices[track].QueueNoteOff();
        else if (v != 0)          _voices[track].QueueNoteOn(v);
    }

    // Sibling-polling loop is already bounded to _voices.Length —
    // safe regardless of the firing track value.
    ...
}
```

Skip silently on out-of-range rather than throwing or logging. The
write was meant for "track 8" which doesn't exist on this voice —
the only meaningful response is to ignore it. Logging would spam
under Pedal Muter's per-mute 16-track sweep.

### 24.3 The polling loop already does the right thing

For the Pedal Muter scenario specifically, the existing
sibling-polling block in `SetNote` (§18) handles the *correct*
behaviour automatically:

1. Pedal Muter writes Note.Off=255 to pvalues[0..15] for the target's
   note parameter.
2. ReBuzz's `parametersChanged` collision (§18) means our `SetNote`
   is called once with `track=15` (or whichever was last written).
3. Our bounds check skips the firing-track write (track 15 is
   out-of-range for our 8 voices).
4. The polling loop iterates `_voices.Length` slots (0..7) and reads
   pvalues[0..7], all of which contain 255 (Note.Off).
5. Each voice gets `QueueNoteOff()` — exactly the mute behaviour
   Pedal Muter intended.

So PedalInvFFT now handles Pedal Muter's stale-voice flush
*correctly* (all voices released on mute), not just *without
crashing*. The bounds-check fix unlocked a feature interaction the
original code couldn't reach.

### 24.4 Generalization to other track-parameter setters

Any `[ParameterDecl]` method that takes an `int track` parameter and
uses it to index per-voice state must apply the same defensive
check. PedalInvFFT currently only has `SetNote`, but future
parameters identified in the README's future-work list — per-track
Detune, per-track Volume offsets, per-track Brightness offsets —
would each need the guard.

The pattern generalizes beyond polyphonic synths. Any managed
machine with track parameters should bounds-check the `track`
argument before indexing per-track state, on the assumption that
`MaxTracks` is a hint to the host but not a contract from external
writers.

### 24.5 Why this didn't surface earlier

The crash needed three conditions simultaneously:

- A polyphonic managed machine with `MaxTracks > 1` (so the setter
  uses `track` as an array index in the first place).
- An external control machine that writes track values beyond
  `MaxTracks`. Pedal Muter is the most prominent in this project's
  ecosystem; the pattern editor and MIDI converters stay within
  range, so manual play-and-test wouldn't trigger it.
- The user actually trying to mute the polyphonic synth via the
  external controller — the specific interaction that brings the
  out-of-range write to `SetNote`.

PedalM1 (Core §14 reference for the polyphonic-managed-synth
pattern) is presumably also vulnerable; worth checking and
hot-fixing there too if it isn't already.
