# ReBuzz Managed Machine Development — Pedal SH101 Addendum

Source: ReBuzz 1819-preview source code + Pedal SH101 v1.0 build (a managed
C# generator: monophonic Roland SH-101 emulation — single VCO with mixable
pulse/saw/sub/noise, ZDF Moog ladder VCF, ADSR, LFO, portamento; no GUI,
no internal sequencer/arp).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. References to `PedalComp §N` point
to that addendum. Internal cross-references use plain `§N` and `§N.M`.

The findings here are largely *not* SH-101-specific — they're general
synth-voice patterns first crystallised on this build, applicable to any
managed generator with per-sample DSP.

---

## 1. `FastPow2` — base-2 fast 2^x for the audio hot path

Any pitch-to-frequency conversion or filter cutoff modulation involves
`MathF.Pow(2f, x)`, called once per sample per source. `MathF.Pow` is a
software function in .NET (~30 ns on x64); two calls per sample at 48 kHz
costs ~3 ms per second of audio per voice. For a synth voice that's a
real chunk of the budget.

PedalComp §5 covers the same idea with a base-10 form (`FastDbToLin`).
Specialising for base 2 is shorter and slightly faster — the polynomial
coefficients for `2^xf` are the natural log series.

```csharp
// Fast 2^x — replaces MathF.Pow(2f, x) in the audio hot path.
// IEEE 754 trick: directly construct the float by setting its exponent
// bits, then multiply by a small polynomial for the fractional part.
//
// Accuracy: ~0.04% (well under 1 cent of pitch / well under 1 dB of
// cutoff). PedalComp §5 covers the base-10 form; this is base-2.
static float FastPow2(float x)
{
    float xi = MathF.Floor(x);
    float xf = x - xi;
    float p  = 1f + xf * (0.69315f + xf * (0.24023f + xf * 0.05550f));
    int   e  = Math.Clamp((int)xi + 127, 1, 254);
    return BitConverter.Int32BitsToSingle(e << 23) * p;
}
```

The exponent clamp `[1, 254]` is essential — out-of-range inputs produce
NaN/Inf otherwise, and the audio thread is the worst place to discover
that. The clamp gives saturating behaviour at the edges of the float
range, which is what an audio engine actually wants.

**Input range bounds verified for SH-101 voice:**

| Use site | Expression | Worst-case input range |
|---|---|---|
| VCO frequency | `(pitchSemis - 69) / 12` | ~[-5.75, +5.08] |
| VCF cutoff modulation | `envOct·env + lfoOct·lfo + kbdOct` | ~[-11, +11] |

Both fit comfortably within the safe range of the polynomial. The
polynomial's worst-case error is at `xf = 0.5` where it differs from
`2^0.5 = 1.41421` by ~0.0006 — about 0.5 cents of pitch.

Use `FastPow2` for **per-sample** Pow2 calls. Keep `MathF.Pow` /
`MathF.Exp` for buffer-rate coefficient setup — accuracy matters more
there than throughput.

---

## 2. Control-rate coefficient updates

The dominant cost in any virtual-analog filter is usually the bilinear
pre-warp tangent: `g = tan(π·fc/sr)`. `MathF.Tan` in .NET is software
and runs at maybe 30–50 ns per call. Most ZDF / TPT filter
implementations cache coefficients against `(fc, res, sr)` — but with
envelope and LFO modulation, `fc` changes every sample, so the cache
hit rate is effectively zero and `tan` runs per sample anyway.

The fix: **update filter coefficients at control rate, not audio rate.**
Recompute every N samples and hold the coefficients between updates.

```csharp
// Hot loop:
const int FILTER_UPDATE_MASK = 16 - 1;   // 16 must be a power of 2
int filterUpdateCounter = 0;

for (int i = 0; i < n; i++)
{
    // ... compute fc as before ...

    if ((filterUpdateCounter & FILTER_UPDATE_MASK) == 0)
        _filter.UpdateCoefs(fc, resN, sr);
    filterUpdateCounter++;

    float filtered = _filter.Process(oscOut);
    // ...
}
```

**Why N = 16 works.** At 48 kHz, 16 samples = 333 µs = 3 kHz update rate.
That's well above the bandwidth of any envelope (a 1 ms attack reaches
its target in ~50 samples) or any LFO (max ~30 Hz on the SH-101). The
audible difference between per-sample and per-16-sample coefficient
updates is below noise floor on every test patch.

**When to drop N.** Fast envelope + heavy filter modulation is the test
case where zipper noise could appear. If a 0–100% Env Amount sweep with
a 1 ms attack and high resonance shows audible stepping, drop the mask
to `8 - 1` (~6 kHz updates) or `4 - 1` (~12 kHz). Each halving doubles
the per-sample cost contribution from `tan`.

**Counter resets per buffer.** `filterUpdateCounter = 0` is set inside
`Work()`, so every buffer starts with an update. This is desirable —
buffer boundaries are natural refresh points, and ensures coefficients
match the start-of-buffer `fc` rather than wherever the previous buffer's
last update happened to leave them.

The pattern generalises to any per-sample DSP element where coefficient
setup is expensive and the modulation source is slower than audio rate.
Pre-warp tangents, log/exp curve mappings, biquad coefficient solves —
all candidates.

---

## 3. Split-API pattern for DSP blocks: `UpdateCoefs` + `Process`

When a DSP block has expensive coefficient setup but cheap per-sample
work, split the API into two methods:

```csharp
public sealed class MoogLadder
{
    public void UpdateCoefs(float fc, float res, int sr) { /* expensive */ }
    public float Process(float input)                    { /* cheap    */ }
}
```

Two benefits over the single `Process(input, fc, res, sr)` form:

1. The caller controls the coefficient update cadence (§2). Combined
   API forces per-sample evaluation of the cache check at minimum, even
   when the cache always hits.
2. Per-sample-constant parameters don't need to be passed every sample.
   `fc`, `res`, `sr` aren't in the inner method's signature, which is
   cleaner and slightly cheaper (fewer register pressures, smaller call
   frame).

The cache short-circuit inside `UpdateCoefs` still matters even with
control-rate calls — when the user isn't modulating the filter, every
update returns immediately:

```csharp
public void UpdateCoefs(float fc, float res, int sr)
{
    if (fc == _cFc && res == _cRes && sr == _cSr) return;
    _cSr = sr; _cFc = fc; _cRes = res;
    // ... expensive coefficient computation ...
}
```

Compose the two patterns: control-rate calls cap the maximum per-sample
cost; the cache check makes the no-modulation case essentially free.

---

## 4. ZDF Moog ladder — closed-form feedback resolution

A 4-pole resonant ladder with global feedback `k·y4` around the whole
chain has a delay-free loop: `y4` depends on `y3` depends on `y2` depends
on `y1` depends on the feedback-modified input which depends on `y4`.
Solving this analytically (rather than naively introducing a unit delay)
is what makes ZDF (zero-delay-feedback) filters track Moog hardware
behaviour up to and including self-oscillation.

The full derivation, working through Vadim Zavalishin's TPT (topology-
preserving transform) one-pole stages:

```
Each stage: y_k = G·x_k + (1-G)·s_k        where G = g/(1+g), g = tan(π·fc/sr)
                                                 s_k = state of stage k

Cascade with input_with_fb = input - k·y4:
    y1 = G·(input - k·y4) + (1-G)·s1
    y2 = G·y1             + (1-G)·s2
    y3 = G·y2             + (1-G)·s3
    y4 = G·y3             + (1-G)·s4

Substitute through to express y4 in terms of input and states only:
    y4 = G⁴·(input - k·y4) + S
       where S = G³(1-G)·s1 + G²(1-G)·s2 + G(1-G)·s3 + (1-G)·s4

Solve for y4:
    y4·(1 + k·G⁴) = G⁴·input + S
    y4 = (G⁴·input + S) / (1 + k·G⁴)
```

That last line is the closed form. With `y4` in hand, the per-stage
outputs `y1..y3` are recomputed forward through the cascade — needed
to update the integrator states `s1..s3` (state update for each TPT
1-pole stage is `s_new = 2·y - s`, the trapezoidal-integrator increment).

**Pre-warping cap.** `tan(π·fc/sr)` blows up as `fc → sr/2`. Cap the
argument before `MathF.Tan`:

```csharp
float wd = MathF.PI * fc / sr;
if (wd > 1.55f) wd = 1.55f;     // π/2 ≈ 1.5708 — stay below the asymptote
float g = MathF.Tan(wd);
```

Without the cap, very high cutoffs produce NaN coefficients that
propagate forever through the integrator states.

**Resonance scaling.** `k = res * 4` puts self-oscillation exactly at
`res = 1.0`. Going higher diverges; the parameter declaration should
clamp at 127 → 1.0.

**Cache the per-stage products.** The `S` calculation in the per-sample
hot path uses `G³(1-G)`, `G²(1-G)`, `G(1-G)`, `(1-G)` — four values
that depend only on `G`. Compute once in `UpdateCoefs` and cache:

```csharp
// In UpdateCoefs:
float G2 = _G * _G;
float G3 = G2 * _G;
_G4          = G2 * G2;
_G3oneMinusG = G3 * _oneMinusG;
_G2oneMinusG = G2 * _oneMinusG;
_GoneMinusG  = _G * _oneMinusG;

// In Process:
float S = _G3oneMinusG * _s1
        + _G2oneMinusG * _s2
        + _GoneMinusG  * _s3
        + _oneMinusG   * _s4;
```

Saves 6 multiplies per sample. Trivial individually, real in aggregate.

**`y4` and `y4cf` are equivalent.** The closed-form `y4cf` and the
re-cascaded `y4` compute the same value in exact arithmetic. They differ
by a few ulps in float. Returning either is fine; the SH-101 build
returns the cascade `y4` for symmetry with the state-update path.

---

## 5. Branch hoisting — per-sample switches to buffer-constant coefficients

When a per-sample loop contains a branch on a value that's constant for
the whole buffer (parameter-driven mode selectors typically are), the
branch is wasted. Resolve it once before the loop, into pre-baked
coefficients that combine in a branchless arithmetic form.

**Worked example — three-way PWM source.** The original form:

```csharp
// In hot loop — switch fires every sample
switch (pwmSrc)
{
    case 1: pwm = 0.5f + lfo * pwmAmt * 0.4f; break;   // LFO mod
    case 2: pwm = 0.5f - env * pwmAmt * 0.4f; break;   // Env mod (downward)
    default: pwm = 0.5f - pwmAmt * 0.4f; break;        // Manual offset
}
```

Hoisted form: build three coefficients once outside the loop, then
combine them branchlessly inside.

```csharp
// Outside loop:
float pwmBase   = 0.5f + ((pwmSrc == 0) ? -pwmAmt * 0.4f : 0f);
float pwmLfoCo  = (pwmSrc == 1) ?  pwmAmt * 0.4f : 0f;
float pwmEnvCo  = (pwmSrc == 2) ? -pwmAmt * 0.4f : 0f;

// Inside loop — one mul-add chain, no branches:
float pwm = pwmBase + lfo * pwmLfoCo + env * pwmEnvCo;
```

The trick: pick the coefficient form so that exactly one of the three
cases activates — the others have their coefficient set to zero. Mode
selection becomes a multiply-by-zero rather than a branch.

Same pattern for VCA mode (env vs gate selector) and Kbd Follow (off /
half / full):

```csharp
// VCA — branchless mix of env vs gate sources:
float vcaUseEnv  = (vcaMode == 1) ? 1f : 0f;
float vcaUseGate = 1f - vcaUseEnv;
float gateLvl    = _gateActive ? 1f : 0f;
// In loop:
float gain = env * vcaUseEnv + gateLvl * vcaUseGate;

// Kbd Follow — single coefficient:
float kbdFollowCoef = (kbdFollowMode == 1) ? (1f / 24f)
                   : (kbdFollowMode == 2) ? (1f / 12f) : 0f;
// In loop:
float kbdOctaves = (currentPitch - 60f) * kbdFollowCoef;
```

The branch isn't *expensive* — modern branch predictors handle constant
predicates trivially. The win is more about cleanliness and amenability
to future SIMD vectorization than raw cycles. But it's a discipline
worth keeping; the per-sample loop should contain only operations that
genuinely need to run every sample.

---

## 6. Monophonic voice triggering — glide, click-free retrigger, pending events

A monophonic synth voice has subtler triggering semantics than a
polyphonic one. Three things matter:

### 6.1 First note doesn't glide

Initial state has `_currentPitchSemis = 60f` (or whatever default).
If glide is enabled and the first note is C-5 (semi 72), the user hears
the voice glide *up* from a fictitious C-4 — wrong on hardware SH-101,
which only glides between successively-played notes.

The `wasIdle` check fixes this: if the envelope is currently inactive,
snap pitch immediately rather than glide.

```csharp
void TriggerNote(byte buzzNote)
{
    int oct  = (buzzNote >> 4);
    int semi = (buzzNote & 0xF) - 1;
    int midi = oct * 12 + semi;
    float pitch = midi;

    bool wasIdle = !_env.IsActive;

    _targetPitchSemis = pitch;
    if (Glide == 0 || wasIdle)
        _currentPitchSemis = pitch;     // snap
    // else: leave _currentPitchSemis to glide via per-sample one-pole

    _env.NoteOn();
    _gateActive = true;
    _lfo.NoteOn(_lfoDelaySamps);
}
```

`wasIdle` must be captured *before* `_env.NoteOn()` — otherwise the
note-on will have just transitioned the env out of Idle and the check
will always be false on retriggers from rest.

### 6.2 Click-free retrigger — `NoteOn` doesn't reset `_level`

The standard textbook ADSR resets the envelope level to 0 on note-on.
For a fresh note this is fine; for a retrigger during the release tail
of a previous note, it produces a click — a sudden zero crossing that
ramps back up.

The fix: don't touch `_level` in `NoteOn`. The envelope re-enters the
Attack stage from wherever it currently sits. If the previous note's
release has already fully decayed (`_level ≈ 0`), behaviour is identical
to the textbook form. If it hasn't, the new attack ramps up smoothly
from the partial release level.

```csharp
public void NoteOn()
{
    // Don't touch _level — smooth retrigger from current value.
    _stage = Stage.Attack;
}
```

The attack uses a slight overshoot target (1.05 instead of 1.0) to make
the curve hit unity in finite time and feel snappier:

```csharp
case Stage.Attack:
{
    const float aTarget = 1.05f;
    _level = aTarget + (_level - aTarget) * _aCoef;
    if (_level >= 1f) { _level = 1f; _stage = Stage.Decay; }
    break;
}
```

Without overshoot, exponential approach to exactly 1.0 takes literally
infinite time. With overshoot, the explicit `>= 1f` check terminates
the attack cleanly within a few time constants.

### 6.3 Pending-event flags drained at top of `Work()`

Setters write a pending byte + flag rather than acting immediately:

```csharp
public void SetNote(Note value, int track)
{
    byte v = value.Value;
    if (v == 0) return;          // NoValue
    if (v == Note.Off)
    {
        _hasNoteOff = true;
        return;
    }
    _pendingBuzzNote = v;
    _hasNoteOn = true;
}
```

`Work()` drains them at the top:

```csharp
if (_hasNoteOn)  { TriggerNote(_pendingBuzzNote); _hasNoteOn = false; }
if (_hasNoteOff) { _env.NoteOff(); _gateActive = false; _hasNoteOff = false; }
```

Order matters: process `note-on` before `note-off`. If a pattern row
contains both (some trackers represent fast retriggers this way), the
voice ends up with the new note triggered and the gate off — which is
"start envelope, immediately release" — usually the intended behaviour.

**Atomicity.** On x64, `byte`, `bool`, and `int` reads/writes are atomic.
The .NET memory model preserves write order within a thread on x64
(TSO-like), so `_pendingBuzzNote = v` followed by `_hasNoteOn = true`
becomes visible to the audio thread in that order. The audio thread
reading `_hasNoteOn == true` is guaranteed to see the prior
`_pendingBuzzNote` write. Marking `_hasNoteOn` `volatile` would make
this guarantee architecture-independent; on x64 it isn't strictly
necessary.

---

## 7. PolyBLEP and shared-phase oscillator structure

The SH-101 has a single VCO core that simultaneously produces saw and
pulse outputs (and the sub from a flip-flop divider). The waveforms are
*correlated* — both tap the same phase. Mixing saw + pulse from the same
phase gives a particular harmonic content that's neither saw nor pulse;
two independent oscillators tuned in unison would produce something
different and less authentic.

```csharp
// Single phase accumulator drives both:
float saw = 2f * _phase - 1f;
saw -= PolyBlep(_phase, dt);                    // smooth wrap discontinuity

float pulse = (_phase < pwm) ? 1f : -1f;
pulse += PolyBlep(_phase, dt);                  // rising edge at phase = 0
float pulseFallEdge = _phase + 1f - pwm;
if (pulseFallEdge >= 1f) pulseFallEdge -= 1f;
pulse -= PolyBlep(pulseFallEdge, dt);           // falling edge at phase = pwm

_phase += dt;
if (_phase >= 1f) _phase -= 1f;
```

PolyBLEP is a polynomial approximation of a band-limited step. It adds
a correction near each waveform discontinuity that smooths the step over
~2 samples, removing the audible aliasing that naive saw / pulse
generators produce. Cheaper than oversampling, with good results up to
~10 kHz fundamental (for higher pitches the band-limit isn't ideal but
it's still better than naive).

```csharp
static float PolyBlep(float t, float dt)
{
    if (t < dt)
    {
        t /= dt;
        return t + t - t * t - 1f;
    }
    if (t > 1f - dt)
    {
        t = (t - 1f) / dt;
        return t * t + t + t + 1f;
    }
    return 0f;
}
```

`dt = freq / sr` is the phase increment per sample (the normalised
frequency). The two branches handle the two-sample window around each
discontinuity.

**Sub-oscillator with PolyBLEP.** On real hardware the sub is a CMOS
flip-flop that toggles on each rising edge of the main pulse — it
produces a square wave at half (or quarter) the main frequency. A
separate phase counter at the divided rate gives the same output and
lets us reuse PolyBLEP for the sub edges:

```csharp
int   subDiv   = (subType == 0) ? 2 : 4;
float subDt    = dt / subDiv;
float subWidth = (subType == 2) ? 0.25f : 0.5f;     // narrow pulse variant

float sub = (_subPhase < subWidth) ? 1f : -1f;
sub += PolyBlep(_subPhase, subDt);
float subFallEdge = _subPhase + 1f - subWidth;
if (subFallEdge >= 1f) subFallEdge -= 1f;
sub -= PolyBlep(subFallEdge, subDt);

_subPhase += subDt;
if (_subPhase >= 1f) _subPhase -= 1f;
```

The two phases (`_phase` and `_subPhase`) drift independently across
notes. That's authentic — the SH-101 phases are free-running, not
reset on note-on. NoteOn() should *not* zero either phase.

---

## 8. Output range and mixer headroom

ReBuzz's nominal sample range for generators is ±32768 (PedalComp §1).
For a synth voice with multiple summable sources (pulse + saw + sub +
noise) all peaking at ±1 internally, the mixer can hit ±4 before VCA
and master volume scale it back down. Multiplied by 32768 that's
±131072 at the output — way past nominal range.

In practice this is OK if the user is sensible (one or two sources at
unity, or master volume backed off), and most downstream stages handle
modest over-range gracefully. But it's a real footgun for patches with
all sources at 100%.

**Three reasonable approaches**, listed in increasing intervention:

1. **Document and trust the user.** Real SH-101 hardware has the same
   property — turn everything to 10 and back off the volume. v1.0 ships
   this way.

2. **Apply a fixed mixer scaling.** A `1/4` factor on the mixer output
   keeps all-100% in nominal range:
   ```csharp
   float mixed = (pulse * pulseLevel + saw * sawLevel
                + sub * subLevel + noise * noiseLevel) * 0.25f;
   ```
   Costs a few dB of headroom at typical (single-source) settings, but
   guarantees the output stage can't clip downstream.

3. **Soft-saturate the output.** A `tanh` or polynomial saturator at
   the VCA stage absorbs the over-range and adds a touch of authentic
   analog warmth. More expensive per sample.

Option 1 is the SH-101 approach; option 2 is the safe-default approach;
option 3 is the "make it sound like analog" approach. Pick consciously
per machine — the SH-101 build chose option 1, partly for authenticity
and partly because the hardware behaviour is part of the instrument's
character.

---

## 9. Summary table — per-sample cost optimization patterns

For future synth voice builds, this is the priority order observed on
the SH-101:

| Optimization | Saves per sample | Pattern |
|---|---|---|
| Control-rate filter coefs (§2) | ~30 ns × (N-1)/N | Update every 16 samples |
| FastPow2 (§1) | ~25 ns × call count | Replace per-sample `MathF.Pow(2,…)` |
| Cache stage products in coef block (§4) | ~3 ns × 6 muls | Move to `UpdateCoefs` |
| Branch hoisting (§5) | ~1 ns × branch count | Combine to mul-add |
| Skip zero-level mixer paths | variable | Per-source `if (level > 0)` guards |

Items 1 and 2 dominate. Without them, a synth voice on a typical CPU
runs maybe 100 voices in real time; with them, comfortably 300+.

The runaway tax in untreated code is `MathF.Tan` per sample inside the
filter, multiplied across however many filter stages and voices the
patch uses. Anyone building a virtual-analog filter who hasn't measured
this surprises themselves the first time they profile.

---

## 10. Known issues / future work

These are deliberately deferred from v1.0; documenting so future
iterations don't have to rediscover them.

- **LFO phase frozen during idle.** When the voice is idle (env Idle,
  gate off), `Work()` takes the silent fast-path and `_lfo.Process()`
  never runs. The LFO phase doesn't advance. A note triggered after an
  idle gap picks up from wherever the LFO was at the *previous* note's
  last sample, not from where a free-running LFO would actually be. On
  real SH-101 the LFO is genuinely free-running. Fix: advance LFO phase
  in the idle branch, or skip the idle fast-path when the LFO has any
  active modulation route. Not audible enough at v1.0 to justify the
  work.

- **No glide-end clamp.** The portamento one-pole asymptotically
  approaches the target but never quite reaches it: `currentPitch =
  target + (currentPitch - target) * coef`. Multiplication by `coef < 1`
  shrinks the delta forever but leaves a long tail of sub-cent error.
  Inaudible, but a `MathF.Abs(currentPitch - target) < 1e-4f` clamp
  would be tidier and save fp work in steady-state.

- **White noise, not pink.** SH-101 noise is closer to pink-ish (filtered
  through analog stages). Uniform white sounds slightly bright in
  isolation. A 1-pole lowpass at ~3 kHz on the noise output would be
  more authentic. Easy to add.

- **Mixer headroom (§8).** v1.0 ships authentic-but-loud. If user
  feedback says it clips downstream too easily, switch to option 2 in §8
  (fixed `0.25` scaling) — preset-compatible since presets store knob
  values, not internal levels.
