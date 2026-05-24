# ReBuzz Managed Machine Notes — Pedal Faze-R

Source: Pedal Faze-R v1.0 build on ReBuzz 1827-preview (an 8-voice
polyphonic phase-distortion synth, Casio CZ lineage: two PD oscillators per
voice with mix/ring/sync, a DCW "wave" envelope in place of a filter, amp +
pitch envelopes, one LFO, a gentle non-resonant tone filter, selectable
Off/2×/4× oversampling; no GUI).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`, `Build §N` to the build addendum,
`SH101 §N` and `M1 §N` to those addenda. Internal cross-references use plain
`§N`.

The genuinely Faze-R-specific findings are the phase-distortion engine and
its *contextual* DCW parameter (§§1–3) and the selectable oversampling
decimator (§4). The platform-side patterns — poly voicing, the §14
multi-track recovery, transport-stop fade — are inherited from M1/Core and
are only noted here where Faze-R does something differently (§6).

---

## 1. Phase distortion, and why DCW replaces the filter

A pure cosine read from a table at a constant rate is a sine. Read it faster
through part of its cycle and slower through the rest — *distort the phase* —
and you get harmonics. The output is always `cos(2π·μ)` where `μ` is a
distorted version of the linear phase `p ∈ [0,1)`; the per-shape distortion
function `μ = D(p, d)` is the whole instrument. `d` is the **DCW** (Digitally
Controlled Wave) amount, `0` = no distortion (pure cosine) and `→1` = full
character.

The key architectural consequence: brightness comes from `d`, not from a
filter. So the synth's main "tone-shaping" envelope is the **DCW envelope**,
not a filter-cutoff envelope. A VA-style ADSR→cutoff would be the wrong
mental model — the DCW envelope *is* the filter envelope here, just acting on
the oscillator's harmonic content directly. The included tone LP (M1 §7
topology) is deliberately gentle and wide-open by default; it's a tone
control, not the source of brightness.

`d` is computed per output sample and held constant across the oversampled
sub-samples (control-rate-ish), then passed into `PDOsc.Tick`:

```csharp
float dMod = dcwEnvAmt * denv + DcwLfoDepth * lfo;     // env (vel-scaled) + LFO
float d1   = clamp(Dcw1Base + dMod, 0f, 0.999f);
```

The `0.999` clamp matters — several distortion functions divide by a knee
width that shrinks with `d`; letting `d` reach exactly 1 puts a zero in the
denominator (§3).

## 2. The contextual DCW — knee for bend shapes, formant ratio for resonant

The single most important Faze-R design point: **`d` means different things
to different waveshapes**, mirroring how the CZ's DCW line drives different
"lines."

- **Bend shapes** (Saw, Square, Pulse, Saw-Pulse): `d` controls a *knee*
  position. As `d→1` the knee tightens, crushing one part of the cosine into
  a narrow window and stretching the rest → richer harmonics. Sine→saw,
  sine→square, etc.
- **Resonant shapes** (Reso Saw/Tri/Trap): `d` controls the **formant ratio**
  `r = 1 + d·(RESO_MAX−1)`, the multiple of the fundamental at which a sine
  "rings" inside a pitch-locked window. Sweeping `d` sweeps a resonant peak
  that tracks the played pitch — a filter-style sweep with no filter.

So **the same DCW envelope that opens a saw's brightness will sweep a reso
shape's formant.** Point the DCW envelope (or LFO→DCW) at a resonant shape
and you get the CZ's signature "resonant filter sweep" for free. This dual
meaning is invisible from the parameter list (one "DCW" knob, one "DCW Env")
and is the thing to remember when a patch behaves unexpectedly: check which
waveshape is selected before reasoning about what DCW is doing.

`r` is left **continuous**, not quantised to integers. Integer ratios give
cleaner classic formants, but continuous `r` makes DCW sweeps smooth instead
of stepped, which matters more for envelope/LFO modulation. The window forces
periodicity at the fundamental regardless, so the formant still tracks pitch.

## 3. Distortion-function guards and the resonant reset edge

Two guard constants in `PDOsc`:

- `KMIN = 0.0015` — minimum knee width for the bend shapes. At `d→1` the knee
  → 0 and `0.5·p/k` would divide by zero; KMIN caps the steepness at "very
  bright" rather than NaN.
- `WMIN = 0.03` — minimum active-window for Pulse/Saw-Pulse (which run a
  cosine cycle then hold at the peak). Same reasoning.

The **resonant shapes are `window(p) · cos(2π·frac(p·r))`**. At the cycle
boundary the window goes to 0 on one side and the next cycle restarts the
carrier at +1 — a deliberate per-fundamental **reset edge** that gives the
resonant formant its buzzy, pitch-tracked character (it's not a bug to
"smooth away"; smoothing it kills the sound). That edge is also the main
aliasing source, which is exactly what the oversampling option (§4) is for.

## 4. Selectable oversampling — windowed-sinc decimator, osc-mix only

PD aliases at high distortion + high pitch; the resonant reset edge (§3) is
the worst offender. `Oversample` (Off / 2× / 4×) trades CPU for cleanliness;
**2× is the default** as a sensible modern baseline, **Off** gives the
authentic CZ grit.

Implementation choices worth remembering:

- **Only the oscillator mix is oversampled.** Envelopes, LFO, the tone
  filter, and amp run at the base rate (per output sample). The decimator
  filters the oversampled osc-mix back down before the tone/amp stage. This
  keeps the extra cost to `os ×` the cheap oscillator work, not the whole
  voice.
- **The FIR is computed at `Configure()` time**, not a hardcoded coefficient
  table — a windowed-sinc (Blackman) low-pass, cutoff `0.5/os` of the
  oversampled Nyquist, length `8·os+1`, normalised to unity DC gain. Computing
  it in code sidesteps transcription errors and scales the kernel with the
  factor automatically.
- **Push/Read contract:** the voice `Push()`es `os` samples then `Read()`s
  once per output sample. `os == 1` bypasses the FIR entirely (pure naive PD).
- **Per-voice state.** Each voice owns its decimator (history ring), reset on
  fresh note-on during the silent moment (M1 §5) and on sample-rate change
  (Core §29).

Validated in a standalone DSP harness before first build: all 8 shapes × {1,
2, 4}× rendered finite and bounded, including a high-note (MIDI 105) DCW
sweep as an aliasing stress test.

## 5. Cosine table + FastPow2

`cos(2π·μ)` is read from an 8192-entry table with linear interpolation
(`DspMath.Cos01`), not `MathF.Cos`. With up to 2 oscs × `os` sub-samples × 8
voices the per-sample `cos` count is high enough that the software `MathF.Cos`
(~20–30 ns) was a measurable chunk; the table read (~3 ns) removes it. Table
accuracy vs `MathF.Cos` is ~2e-6 abs — inaudible. `FastPow2` (SH101 §1) is
reused for the per-sample MIDI→Hz conversion; both were the obvious hot-path
wins and needed no new analysis.

## 6. Inherited platform patterns (delta only)

- **§14 multi-track recovery applies to both Note and Velocity** (as in M1
  §6). Faze-R uses the shape-tolerant `pvalues` reader (Tracker §16.3) from
  the start, so chords don't collapse to last-track-only on ReBuzz ≥1827's
  `int[]` pvalues. Velocity is recovered for *all* tracks (it's a held value,
  not an event); Note only for siblings (the fired track got its real call).
- **Transport-stop fast-fade** (Core §27): `ForcedRelease` on the amp *and*
  DCW envelopes — fading amp alone would leave the wave frozen mid-distortion
  on the tail, so both go.
- **Output scaling:** `SoftClip(mono · volN) · 32768`, with **no** per-voice
  0.5 pre-scale. Unlike M1 §10 (fixed mixer scaling), Faze-R relies on the
  soft clip to absorb polyphonic summing — a single voice sits at a healthy
  level and big chords saturate gently rather than clipping hard (SH101 §8
  option 3). The DC blocker (`Filter.cs`) catches the offset the resonant
  shapes carry.

## 7. Files

`PedalFazeR.cs` (machine + params + §14 recovery + Work), `Voice.cs`,
`PDOsc.cs` (the eight shapes), `Envelope.cs` (ADSR + AD pitch env), `Lfo.cs`,
`Filter.cs` (tone LP + DC blocker), `Decimator.cs`, `DspMath.cs`,
`gen_presets.py` (30-preset bank, source-only). ~1100 lines of C#.

---

## Depends on

- **Build**: §1.2 (csproj properties), §1.3 (deploy to `Gear\Generators`),
  §2 (`.NET` AssemblyName), §3 (.prs.xml format), §3.5 (space-safe preset
  copy), §5 (delivery packaging), §6 (two-namespace usings).
- **Core**: §1–2 (Work / generator classification), §3 + §7 (Note type &
  encoding), §9/§11 (ParameterDecl / MachineDecl properties), §14 (multi-track
  note recovery), §15 (lazy `host.Machine` init), §27 (transport-stop fade),
  §29 (sample-rate change), §33 (return false when fully idle).
- **SH101**: §1 (FastPow2), §6.1–6.3 (mono trigger logic reused per-voice:
  wasIdle snap, click-free retrigger, pending-event drain), §8 (output
  headroom options).
- **M1**: §1 (track=voice poly model), §5 (reset state during the silent
  moment at fire time), §6 (§14 recovery for Note *and* Velocity), §7 (TPT
  tone-filter topology), §9 (gate filter coef updates on sample index), §10
  (output scaling options).
- **Tracker**: §16.3 (shape-tolerant `pvalues` reader).
- **PedalInvFFT**: §24 (defensive track-index bounds in setters).
- **PedalComp**: §2 (bipolar parameters via non-negative range + DSP offset).
