# ReBuzz Managed Machine Development — Pedal M1 Addendum

Source: ReBuzz 1819-preview source code + Pedal M1 v1.0 build (a managed
C# generator: 8-voice polyphonic dual-PCM-oscillator synth, voice
architecture loosely modelled on the Korg M1 — VDF lowpass + VDA + ADSR
envelopes for filter and amp, two per-voice LFOs with independent depths
to pitch / VDF / VDA, oscillators read from the ReBuzz wavetable; no
GUI, no internal sequencer/arp).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. References to `Build §N` point
to `ReBuzz_ManagedMachine_Notes_Build.md`. References to `Tracker §N`
point to `ReBuzz_ManagedMachine_Notes_PedalTracker.md`. References to
`SH101 §N` point to `ReBuzz_ManagedMachine_Notes_PedalSH101.md`.
Internal cross-references use plain `§N` and `§N.M`.

The findings here are largely *not* M1-specific — they're general
patterns for any managed sample-playback or PCM-based polyphonic voice
with per-sample DSP. The Korg M1's particular architecture (dual PCM
osc + gentle VDF + ADSR + 2 LFOs) is just the concrete vehicle.

---

## 1. Voice architecture and signal flow

The M1 voice is straightforward by modern standards: two PCM oscillators
mixed, then through a digital lowpass filter (VDF), then an amplifier
(VDA), with three modulation sources (pitch / filter / amp envelopes
plus two LFOs). For Pedal M1 we keep the architecture but use ADSR
envelopes (the M1's six-stage break-point envelope is a v1.x feature)
and skip the pitch envelope for v1 (most M1 patches don't lean on it).

```
  OSC1 ─┐
        ├── mix ── VDF (lowpass) ── VDA ── voice output
  OSC2 ─┘
              │              │
   FilterEnv ─┘    AmpEnv ───┘
   LFO1 ──────┴──────────────┴──── (also → pitch via osc rate)
   LFO2 ──────┴──────────────┴──── (also → pitch via osc rate)
```

The polyphony model maps tracks one-to-one with voices: track index
*is* voice index, with `MAX_VOICES = MaxTracks = 8`. There is no
independent voice allocation or stealing layer. Chord polyphony comes
from the user placing notes on multiple tracks at the same row, which
is the standard ReBuzz tracker convention. This makes Core §14's
multi-track simultaneous-delivery polling absolutely critical — without
it, chord rows degenerate to last-track-only.

Each voice carries:
- Two oscillator slots (`Voice.Osc1`, `Voice.Osc2`), each with their
  own snapshot reference, position, increment, and active flag
- ADSR amp envelope and ADSR filter envelope (both per-voice — every
  voice has its own envelope state)
- One TPT 2-pole lowpass filter (per-voice integrator state)
- Two LFOs (per-voice phase, restarted on note-on)
- Anti-click target/current gain ramp
- Pending-event flags drained at top of `Work()` (SH101 §6.3)
- Pending-trigger machinery for deferred retriggers (§4)

---

## 2. Wavetable snapshot cache — and the property-change trap

Pedal Tracker established the pattern: build a `WaveSnapshot` on first
access that copies the wave's float data plus a small head/tail padding
zone, cache it, serve subsequent triggers from cache instantly. The
padding (`Interpolation.MAX_PAD = 4`) lets cubic interpolators read
`pos±2` near the start/end of the sample without bounds checks.

```csharp
public sealed class WaveSnapshot
{
    public int      WaveIndex;
    public float[]  DataL;            // pad-offset; real data starts at DataL[MAX_PAD]
    public float[]  DataR;            // null if mono
    public bool     Stereo;
    public int      Length;
    public int      LoopStart, LoopEnd;
    public bool     Loop, BidiLoop;
    public int      RootMidi;         // linear MIDI 0..127
    public int      SampleRate;       // wave's native rate
}
```

The first version of the cache subscribed to `IWave.PropertyChanged`
and invalidated *all* cached snapshots on any property notification:

```csharp
// DON'T DO THIS. It looks responsible. It is not.
void OnWaveChanged(object sender, PropertyChangedEventArgs e)
{
    InvalidateAll();
}
```

The reasoning at the time was "snapshots are cheap and we want to
catch wave edits." Both halves are wrong:

- **Snapshots are not cheap.** A 96 kHz mono 8-second piano sample is
  ~3 MB. The build path allocates a `float[]` of that size, calls
  `IWaveLayer.GetDataAsFloat` to copy the data, then loops twice to
  fill the head/tail padding. None of that should run on the audio
  thread.
- **`PropertyChanged` fires for things you don't think it fires for.**
  Without source access I can't enumerate them, but observable
  wavetable objects in ReBuzz fire `PropertyChanged` during normal
  playback for reasons unrelated to actual sample data. Every fire =
  whole cache nuked = next trigger has to rebuild = audible click.

**The fix that worked: don't subscribe at all.** Cache holds for the
lifetime of the `WavetableAccess` instance. If the user replaces a
wave in the wavetable while the machine is loaded, the snapshot is
stale until the machine is reloaded. That's a one-off workflow note in
the README rather than ongoing audio glitches on every fresh trigger.

If a future build wants real wave-edit detection, the right approach
is per-wave invalidation (only invalidate the specific wave whose data
actually changed) and filtering on `PropertyName` to skip UI-only
properties — not the brute-force version.

### 2.1 Buzz-byte note encoding

`IWaveLayer.RootNote` is declared `int` in BuzzGUI.Interfaces but
holds a Buzz-byte note value (high nibble = octave, low nibble =
semitone+1, with C-4 = `0x40`). The conversion to linear MIDI:

```csharp
static int BuzzByteToMidi(int b)
{
    int oct  = (b >> 4);
    int semi = (b & 0xF) - 1;
    if (semi < 0) semi = 0;
    int m = oct * 12 + semi;
    if (m < 0)   m = 0;
    if (m > 127) m = 127;
    return m;
}
```

The `int` declaration is misleading — early builds used `byte` to
match the conceptual content and got a CS1503 conversion error from
the compiler. Always check Buzz API declarations against actual
returned types; the documentation may use the conceptual term ("Buzz
byte") while the C# type is wider.

---

## 3. Anti-click ramp — interaction with envelope and gain stage

Voice gain is the product of three quantities multiplied each sample:

```csharp
float velAmp = 1f - ampVelN + ampVelN * v.Velocity;
float trem   = 1f + (lfo1 * lfo1ToA + lfo2 * lfo2ToA) * 0.5f;
if (trem < 0f) trem = 0f;
float amp    = aenv * velAmp * v.CurrentGain * trem;
```

Of these, `CurrentGain` is the anti-click ramp — a per-sample
one-pole-style approach to `TargetGain`:

```csharp
public const float FADE_STEP = 1f / 64f;   // ~1.3 ms at 48 kHz
public const float NEAR_ZERO = 1e-4f;

// In per-sample loop:
if (v.CurrentGain < v.TargetGain)
{
    v.CurrentGain += Voice.FADE_STEP;
    if (v.CurrentGain > v.TargetGain) v.CurrentGain = v.TargetGain;
}
else if (v.CurrentGain > v.TargetGain)
{
    v.CurrentGain -= Voice.FADE_STEP;
    if (v.CurrentGain < v.TargetGain) v.CurrentGain = v.TargetGain;
}
```

The ramp serves two roles. First, it absorbs any sample-data
discontinuity at the start of a fresh note: even if the wave starts at
a non-zero value, multiplying by `CurrentGain` ramping from 0 to 1
shapes the onset over ~64 samples regardless of what the env Attack
parameter is set to. Second, it provides the fade-out half of the
deferred-trigger pattern (§4).

**`FADE_STEP` tuning.** 1/64 (1.3 ms at 48 kHz) is the SH-101
inheritance and works for monophonic synth voices. For sample-playback
voices specifically, pushing to 1/256 (5.3 ms) is reasonable if there's
residual clicking — slower onset, but still well under the threshold of
"perceptibly soft attack." For arpeggios at any musically reasonable
tempo (<200 BPM, 16th notes) even 5.3 ms is below 5 % of the
inter-note interval.

**`amp` is the last quantity multiplied.** The order matters: the ramp
multiplies a signal that has already been through the filter, so a
filter transient at note-on can still leak through if the ramp is
slower than the transient. This is why §4 also resets the filter at
fire time, *and* §5 lays out exactly when that reset is safe.

---

## 4. The deferred-trigger pattern — the heart of click-free polyphony

The naive trigger path on a sample-playback voice clicks badly on
arpeggio retriggers. Walk through what happens:

1. Voice is sustaining a note. `CurrentGain = 1`, env in Sustain at
   level 1.0, oscillator reading some position in the loop region with
   instantaneous sample value V. Output is V × ~1 × ~1 × scale =
   loud.
2. Retrigger arrives. Naive handler resets `Pos` to `MAX_PAD`, calls
   `Filter.Reset()`, calls `Env.NoteOn()`.
3. Next sample: oscillator reads position `MAX_PAD`, returning the
   first sample of the wave (call it V'). Filter integrators are zero.
   Output is V' × small × scale.

The discontinuity from V (in the sustain region, large) to V' (start
of attack, small) is a step of arbitrary size. Multiplied by an
envelope that hasn't ramped down yet, this is an audible click —
loudness depending on where in the loop the previous voice was.
Resetting the filter mid-cycle adds a second click on top, because
zeroing the integrators while their state was driving non-zero output
is a step from `Filter.Process(V)` to 0 in one sample.

The fix is the deferred-trigger pattern from Tracker §4.2: when a
retrigger arrives on a still-sounding voice, *stage* it instead of
firing it, and fade the existing voice to silence first. Fire only
when the existing voice has reached `CurrentGain ≈ 0`.

### 4.1 The two-path structure

`TriggerVoice` is now a thin wrapper that decides defer vs. fire-now:

```csharp
void TriggerVoice(int track, byte buzzNote)
{
    var v = _voices[track];

    if (v.CurrentGain > Voice.NEAR_ZERO)
    {
        // Voice still sounding — stage and fade.
        v.HasPendingTrigger   = true;
        v.PendingTriggerNote  = buzzNote;
        v.TargetGain          = 0f;
        return;
    }

    FireTriggerNow(track, buzzNote);
}
```

The actual trigger logic moves into `FireTriggerNow`, which is *only*
ever called when `CurrentGain` is at zero — either from a fresh voice
(constructor or post-release default) or from the inline-fire path
when the deferred fade-out completes:

```csharp
void FireTriggerNow(int track, byte buzzNote)
{
    var v = _voices[track];
    // … pitch, snapshots, oscillator state …
    v.Filter.Reset();          // safe here: amp is 0, no audible step
    v.AmpEnv.NoteOn();
    v.FilterEnv.NoteOn();
    v.Lfo1.NoteOn();
    v.Lfo2.NoteOn();
    v.TargetGain = 1f;
}
```

### 4.2 The inline-fire check

The deferred trigger fires from inside the per-sample voice loop, the
moment `CurrentGain` reaches near-zero:

```csharp
// Anti-click ramp (above) brings CurrentGain toward TargetGain.

if (v.HasPendingTrigger
    && v.CurrentGain <= Voice.NEAR_ZERO
    && v.TargetGain == 0f)
{
    FireTriggerNow(vi, v.PendingTriggerNote);
    v.HasPendingTrigger = false;
}

// Pitch / osc / filter / amp computation follows.
```

Ordering matters within the per-sample loop:

1. Tick LFOs and envelopes (uses pre-fire state — that's fine, the
   one sample at the swap boundary is silent anyway).
2. Anti-click ramp (may bring `CurrentGain` to 0).
3. Inline-fire check (may call `FireTriggerNow`, which `NoteOn`'s the
   envelopes and resets the filter).
4. Compute output sample (uses `aenv` from step 1 and `CurrentGain`
   from step 2 — both 0 at the swap sample, so the swap sample
   contributes 0 to output).

The next sample reads post-fire env/LFO values and `CurrentGain`
ramps up from 0. Total perceived gap: ~64 samples down + some samples
up to audibility (depending on env attack) — call it ~2-3 ms. Inaudible
at any musical arpeggio rate.

### 4.3 Voice activity must include `HasPendingTrigger`

The naive `IsActive` predicate is `env.IsActive || CurrentGain > NEAR_ZERO ||
TargetGain > 0`. This breaks for the case where the env reaches Idle
*and* CurrentGain reaches 0 in the same sample as a deferred trigger
is waiting:

- Sample N-1: env is in Release at small level, `CurrentGain = 0.02`,
  voice processes normally, deferred trigger sits in `HasPendingTrigger`.
- Sample N: env Tick snaps level to 0 (Release stage's `_level < 1e-4`
  branch) and transitions to Idle. Anti-click ramp brings
  `CurrentGain` to 0.

If the IsActive check at the *top* of sample N's voice loop is the
naive one, it sees env Idle, `CurrentGain = 0`, `TargetGain = 0`, and
returns false → voice skipped → inline-fire check never runs → deferred
trigger lost.

The fix:

```csharp
public bool IsActive
    => AmpEnv.IsActive || CurrentGain > NEAR_ZERO || TargetGain > 0f
    || HasPendingTrigger;
```

`HasPendingTrigger` keeps the voice processing through the swap sample
so the inline-fire path runs.

### 4.4 Successive deferrals collapse to most recent

If a second retrigger arrives during a deferred fade-out (very fast
arpeggios), the second call to `TriggerVoice` sees `CurrentGain` still
above `NEAR_ZERO` and goes through the defer path again — overwriting
`PendingTriggerNote` with the newer note. When fade-out completes, the
inline-fire fires the *latest* staged note. Earlier staged notes are
dropped. This is the right behaviour for fast retriggers — the user's
intent is "play the most recent note as soon as you can."

---

## 5. Filter state staleness — the silent click on fresh triggers

After the deferred-trigger pattern was in place, retrigger clicks went
away but a subtler click remained on *fresh* triggers — voices that
had been inactive for a while before the new note. The cause is
specific to per-sample DSP gated by an `IsActive` skip:

When a voice goes inactive (env Idle, `CurrentGain ≈ 0`), the per-sample
loop's `if (!v.IsActive) continue;` skips the entire voice processing
block. That includes `Filter.Process(...)`. The filter integrator state
(`ic1eq`, `ic2eq` for the SVF) is *frozen* at whatever values it held
the last time the voice was active.

Hours later, the voice fires again. The first sample of the new note
goes into `Filter.Process(input)`. The SVF equation is:

```
v3 = input - ic2eq
v1 = a1*ic1eq + a2*v3
v2 = ic2eq + a2*ic1eq + a3*v3
```

If `ic1eq` and `ic2eq` are non-zero (left over from the previous note),
they contribute to `v1` and `v2` independent of the new `input`. The
filter's first few output samples reflect a transient that has nothing
to do with the new note's audio — and even though `CurrentGain` is
ramping up from 0, the transient can be loud enough to leak through
the ramp as a click.

**The fix: reset the filter state at fire time.** This was tricky to
get right because mid-sustain resets *cause* clicks, not fix them —
zeroing the integrators while they're driving live audio is itself a
discontinuity. The deferred-trigger pattern resolves the conflict
neatly:

`FireTriggerNow` is — by construction — only ever called when
`CurrentGain = 0`. At that exact moment, `amp = aenv × CurrentGain ×
… = 0`, so `output = filtered × 0 = 0`. Zeroing the filter integrators
in the same sample produces no audible step, because nothing is being
multiplied by the filter output anyway. The reset is *invisible*.

```csharp
void FireTriggerNow(int track, byte buzzNote)
{
    // … oscillator setup, pitch, etc. …
    v.Filter.Reset();           // safe: amp is 0 at fire time
    v.AmpEnv.NoteOn();
    v.FilterEnv.NoteOn();
    v.Lfo1.NoteOn();
    v.Lfo2.NoteOn();
    v.TargetGain = 1f;
}
```

The general principle: **state-resetting operations are safe precisely
when they happen at audio-zero crossings.** The deferred-trigger
pattern's main purpose is the fade-out itself, but a hidden second
purpose is creating a guaranteed-silent moment in which other resets
become free.

This generalises to any per-voice state that's frozen during inactivity
and read on reactivation: filter integrators, LFO sample-and-hold
state, anything with persistent memory. If it's read on fire, it
should be reset on fire (during the silent moment) — not on note-off
(where the audio is still sounding).

---

## 6. The §14 polling workaround applies to Volume too, not just Note

Core §14 documents the multi-track simultaneous-delivery bug: when
multiple track parameters are written in the same audio tick, only the
last setter actually fires — `parametersChanged` is keyed by parameter
ID, not (parameter, track), so earlier writes are overwritten before
ReBuzz iterates the dictionary.

In a polyphonic synth, this bug applies independently to *every* track
parameter. For Pedal M1 that's both `Note` and `Volume` (track-level
velocity). A chord row with `Note=C4, Vol=80` on track 0 and `Note=E4,
Vol=120` on track 1 sees both setters lose their first-track call —
SetNote(track 0) and SetTrackVolume(track 0) both vanish.

The polling workaround needs to recover both:

```csharp
void PollSiblingTracks(int firedTrack)
{
    EnsurePollFields();
    if (_ownNotePValues == null && _ownVolumePValues == null) return;

    int noteNoVal = _ownNoteParam?.NoValue ?? 0;
    int volNoVal  = _ownVolumeParam?.NoValue ?? 255;

    for (int t = 0; t < MAX_VOICES_CONST; t++)
    {
        var v = _voices[t];

        if (t != firedTrack && _ownNotePValues != null
            && !v.HasNoteOn && !v.HasNoteOff)
        {
            if (_ownNotePValues.TryGetValue(t, out int pv)
                && pv != noteNoVal && pv != 0)
            {
                if (pv == 255) v.HasNoteOff = true;
                else { v.HasNoteOn = true; v.PendingBuzzNote = (byte)pv; }
            }
        }

        if (_ownVolumePValues != null && !v.HasNewVelocity)
        {
            if (_ownVolumePValues.TryGetValue(t, out int pv)
                && pv != volNoVal)
            {
                v.HasNewVelocity   = true;
                v.PendingVolumeRaw = (byte)pv;
            }
        }
    }
}
```

Two design points worth flagging:

- **Guard with `!HasXxx`.** A real setter call earlier in the same
  tick takes precedence over the poll. The flag tells us "this track
  has already been handled by a real setter" so we don't overwrite
  with a possibly-stale pvalue.
- **Call from both setters.** `PollSiblingTracks` runs from inside both
  `SetNote` and `SetTrackVolume`. Either setter firing for any track
  is enough to recover all the others. The redundancy is harmless
  because the `!HasXxx` guards prevent double-handling.

`EnsurePollFields` lazily resolves `_ownNoteParam` and
`_ownVolumeParam` via the `Machine.ParameterGroups` walk (Core §15)
because parameter groups aren't populated when the constructor runs.

---

## 7. Filter character — why "gentle non-resonant 2-pole" matters

The Korg M1's VDF is famously *not* a synth filter in the analog
tradition. It's a digital lowpass with very mild — almost imperceptible
— resonance. The character of an M1 patch comes from the PCM samples
and envelope-shaping, not from filter sweeps. Picking the wrong filter
topology is the easiest way to make an "M1 emulation" not sound like
an M1 at all.

For Pedal M1 the choice was a TPT (topology-preserving) state-variable
filter taking only the lowpass output, with `k = 1/Q` clamped to a
gentle range:

```csharp
float qN = 0.5f + (resN < 0f ? 0f : (resN > 1f ? 1f : resN)) * 3.5f;
float k  = 1f / qN;
```

`Q` ranges from 0.5 (essentially flat, Butterworth-ish) to 4.0 (mild
emphasis). The filter never self-oscillates. A user dialling
"Resonance = 127" gets perceptible emphasis around the cutoff but no
analog-style screaming.

The TPT formulation comes from Vadim Zavalishin's textbook:

```csharp
float wd = MathF.PI * fcHz / sr;
if (wd > 1.55f) wd = 1.55f;
float g  = MathF.Tan(wd);

float den = 1f + g * (g + k);
_a1 = 1f / den;
_a2 = g  * _a1;
_a3 = g  * _a2;

// Per-sample:
float v3 = input - _ic2eq;
float v1 = _a1 * _ic1eq + _a2 * v3;
float v2 = _ic2eq + _a2 * _ic1eq + _a3 * v3;
_ic1eq   = 2f * v1 - _ic1eq;
_ic2eq   = 2f * v2 - _ic2eq;
return v2;     // lowpass tap
```

The pre-warp cap (`wd > 1.55f`) is the same one SH101 §4 documents
for the Moog ladder — `tan(π/2)` blows up to infinity, and unbounded
`g` produces NaN coefficients that propagate forever through the
integrators.

Coefs are control-rate updated (every 16 samples per voice, gated on
the *sample index* not a per-voice counter — see §9). A cache check
inside `UpdateCoefs` short-circuits when `(sr, fc, res)` matches the
last computation, so the buffer-rate envelope/LFO update at the top of
`Work()` doesn't actually trigger a recompute when nothing relevant
has changed.

If you're building a different vintage-digital sampler emulation
(Mirage, FZ-1, S-50, etc.), the filter character is similar — gentle,
non-resonant or barely-resonant, more tone-control than synth-filter.
The Moog ladder from SH-101 would be wrong for any of them.

---

## 8. Per-osc playback rate — FastPow2 in the per-sample path

Each oscillator's playback rate has to update every sample to honour
LFO pitch modulation. The naive form uses `Math.Pow(2, semis/12)`,
which is software-implemented in .NET (~30 ns per call). Per-voice ×
per-osc × per-sample × buffer-size, that's tens of thousands of `Math.Pow`
calls per second — measurable on the budget.

`FastPow2` from SH101 §1 handles this in ~1 ns:

```csharp
void UpdateOscIncrement(Voice.Osc o, double playMidi,
                        int octParam, int transposeParam, int detuneParam, int sr)
{
    if (!o.Active || o.Snap == null) return;
    double octShift   = octParam - 2;
    double semiShift  = transposeParam - 12;
    double centsShift = (detuneParam - 50) * 0.01;
    double effectiveMidi = playMidi + octShift * 12.0 + semiShift + centsShift;

    float semiDelta = (float)(effectiveMidi - o.Snap.RootMidi);
    float rate      = Pow2(semiDelta / 12f);

    double srRatio  = (double)o.Snap.SampleRate / sr;
    o.Increment     = rate * srRatio;
}
```

Worth noting: the parameter offset trick (PedalComp §2) puts all
parameters at non-negative `MinValue`, so `octParam`, `transposeParam`,
and `detuneParam` are unsigned ranges centred at 2, 12, 50 respectively.
The DSP subtracts the offset to get the signed shift. This pattern is
becoming standard across the Pedal series — see PedalComp §2 for the
reasoning.

`srRatio` handles the wave's native rate vs the playback rate. A 96 kHz
sample played back at 48 kHz needs `Increment = 2.0` per output sample
to maintain pitch. The user's M1 piano samples are 24-bit/96 kHz mono,
so this matters in practice.

---

## 9. Filter coefficient gating — gate on sample index, not a shared counter

A subtle bug from the v1 build worth flagging. The first version of
the filter-update gating used a counter shared across the per-voice
loop:

```csharp
// BUGGY — don't do this
int filterUpdateCounter = 0;
for (int i = 0; i < n; i++)
    for (int vi = 0; vi < MAX_VOICES; vi++)
    {
        if ((filterUpdateCounter & 15) == 0)
            v.Filter.UpdateCoefs(...);
        v.Filter.Process(...);
        filterUpdateCounter++;
    }
```

With `MAX_VOICES = 8`, `filterUpdateCounter` advances by 8 per output
sample (one increment per voice). Voice 0's filter updates when the
counter is at multiples of 16 — which happens on alternating samples
(0, 16=after 2 samples, 32=after 4 samples, etc.). Voice 1's filter
updates when `filterUpdateCounter & 15 == 0` for voice 1's iteration,
which never happens because voice 1 always sees odd offsets.

**Most voices' filters never get their coefficients updated.**

The fix is to gate on the sample index `i` directly:

```csharp
for (int i = 0; i < n; i++)
    for (int vi = 0; vi < MAX_VOICES; vi++)
    {
        if ((i & 15) == 0)
            v.Filter.UpdateCoefs(...);
        v.Filter.Process(...);
    }
```

Now every voice's filter updates on the same sample boundaries (samples
0, 16, 32, …), which is what control-rate gating is supposed to mean.

The general principle: any quantity that should update at control rate
"every N samples" should be gated on the sample index, not a counter
that gets advanced per inner-loop iteration. The cache check inside
`UpdateCoefs` itself (which short-circuits when nothing changed) is
fine and cheap, but it doesn't substitute for correct gating.

---

## 10. Output range and mixer headroom

Per PedalComp §1, generator machines write directly to ±32768. With two
oscillators at unity level, a worst-case input to the filter is ±2;
the filter is bounded close to unity at most cutoffs; envelope and
gain are at most 1; so the per-voice output is bounded by ~2. With 8
voices summing, peak is ~16 — well over nominal range.

Pedal M1 applies a fixed 0.5× mixer scaling on the output stage:

```csharp
float scale = volN * 0.5f * SAMPLE_SCALE;
output[i] = new Sample(sumL * scale, sumR * scale);
```

This is the SH101 §8 "option 2" approach (fixed mixer scaling), chosen
deliberately over option 1 (trust the user). Sampler-style synths
often have many voices ringing simultaneously, and unlike the SH-101
where it's always one voice, the click/clip risk for a polyphonic synth
is real if the user happens to play a few sustained notes. -6 dB of
headroom in the mixer trades a small loudness cost for guaranteed
non-clipping at typical patch settings. If a user finds patches too
quiet, master `Volume` is the lever.

If a future polyphonic build wants to take more risk, option 3 (soft
saturate the output stage) is a nice middle ground — keeps full
loudness at moderate levels and gracefully absorbs over-range at high
levels.

---

## 11. LFO — per-voice phase, key-sync default

Each voice gets its own LFO state for both LFO1 and LFO2, restarted at
note-on (M1 default behaviour — "key sync"). This is *not* the SH-101
approach, where LFO and oscillator phases drift independently across
notes (SH101 §7). For sample-based instruments the M1 convention is
right: vibrato and tremolo should be predictable per-note, not "wherever
the LFO happened to be when you played it."

```csharp
public void NoteOn()
{
    _phase        = 0f;
    _delayCounter = _delaySamples;
    _fadeCounter  = _fadeSamples;
    _shCurrent    = NextRandom();
    _smoothFrom   = _shCurrent;
    _smoothTo     = NextRandom();
}
```

The five waveforms are Triangle, Sawtooth (descending), Square, Sample
& Hold, and Smooth Random (linear interpolation between random
targets per cycle). Triangle defaults to phase=0 outputting -1 — this
*is* a quirk worth understanding. Combined with a key-sync restart, a
fresh note's first LFO sample is -1 for triangle, +1 for square, and
random for S&H. If the user has a non-zero LFO depth to filter cutoff,
this means the cutoff *jumps* to its modulated value at the moment of
trigger. With small depths it's inaudible; with large depths it's a
detectable filter shift on each note.

If a future build wants the LFO start phase to be musically configurable
(e.g. "always start at LFO=0" for triangle), that's a parameter to add,
not a default to change — the phase=0 start is what hardware does and
some patches depend on it.

**Random source is xorshift32**, which is fast and audio-thread safe
(no allocations, no locks). Per-voice RNG state, seeded at construction,
keeps voices uncorrelated.

---

## 12. Oscillator looping — the cubic-at-loop-wrap caveat

Oscillator playback handles forward looping, bidirectional looping, and
non-looped one-shots. The forward-loop wrap is the standard subtract-
span:

```csharp
int loopEnd = Interpolation.MAX_PAD + s.LoopEnd;
if (s.Loop && o.Pos >= loopEnd)
{
    if (s.BidiLoop)
    {
        o.Pos = loopEnd - (o.Pos - loopEnd);
        o.BackwardsBidi = true;
    }
    else
    {
        double span = s.LoopEnd - s.LoopStart;
        if (span < 1) span = 1;
        while (o.Pos >= loopEnd) o.Pos -= span;
    }
}
else if (!s.Loop && o.Pos >= Interpolation.MAX_PAD + s.Length)
{
    o.Active = false;
    return v;
}
```

**Caveat: cubic interpolation across the loop wrap clicks slightly.**
The cubic interpolator reads `pos-1, pos, pos+1, pos+2`. After the
wrap, `pos` is back at `MAX_PAD + LoopStart`. Reading `pos-1` returns
`DataL[MAX_PAD + LoopStart - 1]` — which is the actual sample data
just before the loop start, not the data just before the loop end.
There's a discontinuity in the "virtual stream" the cubic interpolator
sees, even when the loop endpoints are themselves smooth.

For waves with carefully designed loop points (most commercial sample
libraries) the discontinuity is small and inaudible. For waves with
crude loop points it clicks on every iteration. The proper fix is
"loop padding" — copying samples from beyond `LoopEnd` into the
pre-`LoopStart` pad zone so the interpolator's tap window crosses the
loop seamlessly. Not implemented in v1; documented as a v1.x feature.

The v1 test samples (24-bit/96 kHz piano recordings, full natural
decay) are 5.7 s and 7.7 s long with no loop metadata in the WAV,
so this caveat doesn't apply in normal use — they're never looped at
musical tempi.

---

## 13. Voice-active pruning is currently absent

Pedal M1 v1 doesn't aggressively prune voices when their env reaches
Idle. A note-off transitions the env to Release, the env decays
naturally, and when it reaches Idle the voice's `IsActive` returns
true *as long as `CurrentGain > NEAR_ZERO`*. Since note-off doesn't
touch `CurrentGain` (which would shorten the user's release time —
wrong), `CurrentGain` stays at 1 indefinitely after the env ends.

Net effect: a voice that has finished playing keeps ticking through
the per-sample loop, computing `aenv (= 0) × CurrentGain (= 1) × … = 0`
forever, until the next trigger arrives (which goes through the
deferred path because `CurrentGain > NEAR_ZERO`).

This is wasted CPU but doesn't cause clicks. With 8 voices it's not
significant; for 32+ voice machines it would matter.

The fix, when it becomes worth doing: detect env transition to Idle
and snap `CurrentGain` to 0 in the same instant. Audio is already 0
(env is 0), so zeroing `CurrentGain` is inaudible:

```csharp
// After AmpEnv.Tick(), inside the voice loop:
if (!v.AmpEnv.IsActive && v.TargetGain > 0f)
{
    v.TargetGain  = 0f;
    v.CurrentGain = 0f;     // safe: env is 0, audio is silent
}
```

The next sample's `IsActive` returns false, voice is skipped, until a
new trigger arrives. The voice then goes through the fresh-trigger
path (instant fire, not deferred) because `CurrentGain = 0`.

Documented here for a future v1.x — not in v1.

---

## 14. Parameter design — appendable layout, M1 conventions where compatible

Per Build §3.3, parameter declaration order is part of the preset
contract. The Pedal M1 v1 layout, in order:

```
OSC1 Wave / Octave / Transpose / Detune / Level     (5)
OSC2 Wave / Octave / Transpose / Detune / Level     (5)
OSC Mode                                             (1)
Portamento                                           (1)
Cutoff / Resonance / Key Track / Filter EG Int /
  Filter Vel                                         (5)
Filter Attack / Decay / Sustain / Release            (4)
Amp Attack / Decay / Sustain / Release / Vel         (5)
LFO1 Wave / Speed / Delay / Fade / →Pitch /
  →VDF / →VDA                                        (7)
LFO2 Wave / Speed / Delay / Fade / →Pitch /
  →VDF / →VDA                                        (7)
Volume                                               (1)
                                                   ----
                                                    41 globals
```

Track parameters: `Note` (stateless, IsStateless=true) and `Volume`
(stateful, 0..127 velocity).

**Future parameters append-only.** When v1.1 adds the M1 6-stage amp
envelope (BreakLevel, Slope) and the pitch envelope (4 ADSR + intensity),
they go *after* `Volume` (master) — preserving existing presets'
parameter indexes. Same rule for any v1.x additions.

**Bipolar parameters use the offset trick.** Octave (-2..+2) is encoded
0..4 with default 2. Transpose (±12 semitones) is encoded 0..24 with
default 12. Detune (±50 cents) is encoded 0..100 with default 50.
Key Track (full inverse to full positive) is encoded 0..127 with default
64. The DSP subtracts the offset to recover the signed value.
This complies with PedalComp §2 (1819-preview's silent-offset rule for
negative `MinValue`).

**Two parameters share the name "Volume".** A global `Volume` (master
output) and a track `Volume` (velocity). They live in different
parameter groups (1 and 2) so no actual collision; Buzz convention
allows it. The shared name is fine from a UX standpoint — context makes
the distinction clear in the rack vs. the pattern editor.

---

## 15. Per-sample DSP cost summary

For an 8-voice polyphonic build, the per-voice per-sample budget at
48 kHz is roughly 250 ns to stay under 10% CPU on a single core. Pedal
M1's voice DSP fits comfortably:

| Operation | Cost |
|---|---|
| 2 × LFO Tick | ~10 ns each (xorshift + branch + phase advance) |
| 2 × Env Tick | ~5 ns each (one-pole-style coef approach) |
| Anti-click ramp | ~3 ns (two-branch comparison + add) |
| Inline-fire check | ~1 ns (three-condition && branch, predictable) |
| Pitch update (continuous) | ~2 ns (one-pole approach to TargetMidi) |
| 2 × `UpdateOscIncrement` | ~3 ns each (FastPow2-based) |
| 2 × `TickOsc` | ~30 ns each (cubic interpolator + position math) |
| Filter `Process` | ~10 ns (5 muls + 2 adds, no transcendentals) |
| Filter `UpdateCoefs` | amortised (every 16 samples, ~50 ns each) |
| Amp computation | ~5 ns |
| **Total per voice per sample** | **~110-130 ns** |
| **8 voices × 110 ns × 48000 samples/s** | **~50 ms/s ≈ 5% CPU** |

This is ballpark — actual measurements would be needed for tuning
beyond v1. The biggest single line item is the cubic interpolator (×2
oscs); replacing with linear when audio quality allows would knock
~30% off voice cost.

The `MAX_VOICES = 8` choice matches the M1's standard polyphony and
keeps voice cost in single-digit-percent territory. Bumping to 16+
would still fit the budget but stretches the §6 polling cost (which
is O(MAX_VOICES) per setter call).

---

## 16. Files and structure

```
pedalm1/
├── Pedal M1.NET.csproj   build (with Build §1.2 mandatory properties)
├── PedalM1.cs            main: parameters, setters, Work, voicing
├── Voice.cs              per-voice state: 2 oscs, envs, filter, LFOs
├── Wavetable.cs          snapshot cache (no PropertyChanged sub — §2)
├── Envelope.cs           ADSR with click-free retrigger (SH101 §6.2)
├── Lfo.cs                per-voice LFO with delay + fade-in
├── Filter.cs             2-pole TPT lowpass (gentle, non-self-osc — §7)
├── Interpolation.cs      linear + Catmull-Rom cubic
└── README.md             user-facing usage notes
```

`PedalM1.cs` ended up at ~700 lines, dominated by the parameter
declarations (~280 lines for 41 parameter property/method pairs) and
the per-sample inner loop (~130 lines including all the modulation
math and inline-fire). Voice.cs is ~140 lines, Wavetable.cs is ~155
lines, the rest are 60-150 lines each.

Total ~1500 lines of C#, comparable to Pedal Tracker.

---

## 17. Build status as of last test session

- **Compiles clean** — no warnings, no errors. Single CS1503 fixed
  in early build (RootNote `int` vs `byte`).
- **Loads in ReBuzz** — confirmed by user.
- **Polyphonic playback works** — confirmed by user with 24-bit/96 kHz
  M1 piano samples loaded, two octaves apart, played via Pedal Chord
  arpeggios.
- **Parameters all behave as expected** — confirmed by user.
- **Click on retrigger** — fixed via deferred-trigger pattern (§4).
- **Click on fresh trigger** — most likely caused by `PropertyChanged`
  forcing snapshot rebuilds; fixed by removing the subscription (§2).
  **Status: pending user confirmation.**

If the fresh-trigger click persists after the §2 fix, the next likely
candidate is the anti-click ramp being too fast for the early piano
attack region — bumping `FADE_STEP` from `1f / 64f` to `1f / 256f`
would give 5.3 ms fade-in instead of 1.3 ms. Documented in README as
a tuning knob.

---

## 18. v1.x roadmap

In rough priority order:

1. **6-stage amp envelope.** The M1's actual envelope is
   Attack → AttackLevel → Decay → BreakLevel → Slope → Sustain →
   Release. Most patches don't use the BreakLevel/Slope stage, but
   it's a defining feature of the M1 sound on the patches that do.
   Append-only addition (per §14).
2. **Pitch envelope.** 4 ADSR params + intensity. Append-only.
3. **Loop padding.** Copy samples beyond `LoopEnd` into the
   pre-`LoopStart` pad zone so cubic interpolation across the loop wrap
   doesn't click. Improves quality on user-supplied samples with rough
   loop points.
4. **Voice-active pruning.** Snap `CurrentGain` to 0 when env reaches
   Idle (§13). Frees up CPU on patches with lots of long releases.
5. **LFO free-run option.** Currently key-sync only (§11). A per-LFO
   "free run" parameter would let LFO phase drift across notes for
   patches that want that character. Requires per-voice phase memory
   that survives note-on rather than reset.
6. **Multi-timbral / Combi mode.** The M1 has 8-part Combis. Far out
   of scope for v1.x — would require either multiple machine instances
   or per-track patch slots.

A custom GUI was discussed and intentionally deferred — the rack works
fine for 41 parameters and a custom panel is comparable in scope to
the entire DSP build.
