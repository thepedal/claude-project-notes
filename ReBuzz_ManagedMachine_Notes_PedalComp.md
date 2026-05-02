# ReBuzz Managed Machine Development — Pedal Comp Addendum

Source: ReBuzz 1817-preview source code + Pedal Comp v1.0 build (a managed
C# effect machine: stereo soft-knee compressor with peak/RMS detection,
look-ahead, and metering).

Sections numbered locally. References to `Core §N` point to
`ReBuzz_ManagedMachine_Notes_Core.md`. No content overlap with Core.

---

## 1. ReBuzz sample values are ±32768, not ±1.0

Effect machines receive and must return samples scaled to ±32768. All dB
maths will produce wildly wrong results without normalising first. This
manifests as a compressor that crushes everything to silence at ratio 2:1,
or meters that read "full" even on near-silent signals.

```csharp
const float SCALE = 1f / 32768f;

// In Work():
float inL = input[i].L * SCALE;    // ±32768 → ±1.0
float inR = input[i].R * SCALE;

// ... all DSP at ±1.0 normalised scale ...

output[i] = new Sample(outL / SCALE, outR / SCALE);  // ±1.0 → ±32768
```

This applies to all effect machines (`bool Work(Sample[] output, Sample[] input, ...)`).
Generator machines that synthesise from scratch write directly to ±32768 without
going through the normalise/de-normalise step.

---

## 2. `ParameterDecl` MinValue must be ≥ 0

ReBuzz validates parameter declarations at load time. In older builds
(pre-1819) a negative `MinValue` caused an `Invalid MinValue` exception
during DLL scanning and the machine silently failed to appear in the browser.

**Behaviour change in 1819-preview:** the machine loads successfully with a
negative `MinValue`, but ReBuzz silently **offsets the stored parameter range**
to make it non-negative. The property setter receives values in the range
`0..(MaxValue − MinValue)` rather than `MinValue..MaxValue`. This is
worse than the previous hard failure because:

- The machine appears to work — no error is surfaced.
- All DSP formulas that treat the raw property value as the actual dB value
  compute the wrong result (wrong sign and magnitude).
- The GUI display shows wrong values for the same reason.
- The `DefValue` offset shifts too: a `DefValue = 0` with `MinValue = -48`
  stores `48` as the default, not `0` — so a "flat" default is stored as a
  large positive integer.

**Concrete example from Pedal EQ v1.2:**

```csharp
// WRONG — MinValue=-48 causes ReBuzz to offset stored range to 0-96.
// Setter receives 18 when user sets "-15.0 dB" (which is index 18 in the
// 0-96 range). DSP computes 18 * 0.5 = +9.0 dB — applying a boost where
// the user wanted a cut.
[ParameterDecl(MinValue = -48, MaxValue = 48, DefValue = 0)]
public int LSGain { get; set; } = 0;

double gainDb = LSGain * 0.5;   // WRONG: 18 * 0.5 = +9.0 dB, not -15.0 dB
```

**Correct pattern — non-negative range with explicit offset in DSP:**

```csharp
// CORRECT — MinValue=0, MaxValue=96, DefValue=48 (flat point).
// Setter receives 18; formula (18-48)*0.5 = -15.0 dB. Correct.
[ParameterDecl(
    MinValue          = 0,
    MaxValue          = 96,
    DefValue          = 48,      // 48 = 0.0 dB (the flat/zero point)
    ValueDescriptions = new[]    // 97 strings: index 0 = "-24.0 dB",
    {                            //             index 48 = "0.0 dB",
        "-24.0 dB", "-23.5 dB", //             index 96 = "+24.0 dB"
        // ...
        "0.0 dB",
        // ...
        "+24.0 dB"
    })]
public int LSGain { get; set; } = 48;   // initialise to flat

double gainDb = (LSGain - 48) * 0.5;   // CORRECT: (18-48)*0.5 = -15.0 dB
```

The same offset applies **everywhere** the raw value is used: DSP coefficient
calculations, output gain ramp (`(OutGain - 48) * 0.025f`), GUI display
formulas, and `allFlat` fast-path checks (flat is `value == 48`, not
`value == 0`).

The `ValueDescriptions` array content does not change — it was always indexed
0–96. Only the ParameterDecl min/max/default and the conversion formula change.

The same constraint applies to `DefValue` — it must be within
`[MinValue, MaxValue]`.

---

## 3. `AssemblyName` in the csproj controls the machine browser display name

The name shown in the ReBuzz machine browser comes from the DLL filename,
which is determined by `<AssemblyName>` in the `.csproj` — not by
`MachineDecl.Name`. A space in `AssemblyName` produces a space in the list.

```xml
<!-- Produces: "Pedal Comp.NET.dll" → appears as "Pedal Comp" in browser -->
<AssemblyName>Pedal Comp.NET</AssemblyName>
```

`MachineDecl.Name` is used in the machine's title bar and for internal
identification, but the browser entry comes from the filename. When the two
differ, users may see a different name when hovering over an already-placed
machine versus finding it in the browser.

---

## 4. `ValueDescriptions` for labelled toggle and enum parameters

`ParameterDecl` supports a `ValueDescriptions` array that maps parameter
values to display strings. Use this for any parameter with a small fixed set
of meaningful states — the label appears in the parameter display, pattern
editor, and automation lanes instead of a raw integer.

```csharp
[ParameterDecl(
    MinValue          = 0,
    MaxValue          = 1,
    DefValue          = 0,
    ValueDescriptions = new[] { "Peak", "RMS" })]
public int Detection { get; set; }

// Three-way example
[ParameterDecl(
    MinValue          = 0,
    MaxValue          = 2,
    DefValue          = 0,
    ValueDescriptions = new[] { "Off", "Soft", "Hard" })]
public int ClipMode { get; set; }
```

The array length must match `MaxValue - MinValue + 1`. There is no partial
labelling — either all values are described or none are.

---

## 5. Fast log/exp for the DSP hot path

`MathF.Log10` and `MathF.Pow` are expensive enough to measure when called
inside a per-sample loop. Replace with IEEE 754 bit-manipulation approximations
that avoid the transcendental function entirely. Accuracy is ±0.1 dB —
sufficient for gain computation in any dynamics processor.

```csharp
// Fast LinToDb — replaces 20f * MathF.Log10(lin)
// Accuracy: ±0.1 dB over 1e-6 to 1.0
static float FastLinToDb(float lin)
{
    if (lin <= 1e-9f) return -120f;
    int   bits = BitConverter.SingleToInt32Bits(lin);
    float exp  = (bits >> 23) - 127f;
    float mant = BitConverter.Int32BitsToSingle((bits & 0x007FFFFF) | 0x3F800000) - 1f;
    float log2 = exp + mant * (1.4142f - 0.7071f * mant);
    return log2 * 6.02059f;   // log2 → dB: × (20 / log₂10)
}

// Fast DbToLin — replaces MathF.Pow(10f, db / 20f)
// Accuracy: ±0.1 dB for db in [−120, 24]
static float FastDbToLin(float db)
{
    float x  = db * 0.16609f;           // db × log₂(10) / 20
    float xi = MathF.Floor(x);
    float xf = x - xi;
    float p  = 1f + xf * (0.69315f + xf * (0.24023f + xf * 0.05550f));
    int   e  = Math.Clamp((int)xi + 127, 1, 254);
    return BitConverter.Int32BitsToSingle(e << 23) * p;
}
```

Keep the exact `MathF.Pow` / `MathF.Log10` versions for coefficient setup
(outside the loop) where accuracy matters more than throughput.

---

## 6. Coefficient caching — dirty-check pattern

Any DSP coefficient derived from a parameter or sample rate (envelope follower
time constants, linear threshold, makeup gain, etc.) should be computed once
and cached. Recompute only when an input changes. This removes all
`MathF.Exp` / `MathF.Pow` calls from the per-block hot path.

```csharp
// ── Cached fields ─────────────────────────────────────────────────────────
int   _cachedSr      = 0;
int   _cachedAttack  = -1;
int   _cachedRelease = -1;
float _attackCoef    = 0f;
float _releaseCoef   = 0f;

void UpdateCoefficients(int sr)
{
    if (sr == _cachedSr && Attack == _cachedAttack && Release == _cachedRelease)
        return;   // nothing changed — skip all Exp/Pow calls

    _cachedSr      = sr;
    _cachedAttack  = Attack;
    _cachedRelease = Release;

    _attackCoef  = MathF.Exp(-1f / (Attack  * 0.001f * sr));
    _releaseCoef = MathF.Exp(-1f / (Release * 0.001f * sr));
}

// In Work():
int sr = Global.Buzz.SelectedAudioDriverSampleRate;
UpdateCoefficients(sr);   // cheap no-op if nothing changed
```

Add every relevant parameter and `sr` to the dirty check. When parameters
change between blocks (automation, user edits), the cache misses once and
the coefficients update cleanly. The call overhead for the no-op path is
a handful of integer comparisons per block.

---

## 7. `volatile` float/int for audio→UI metering

Metering values (peak level, gain reduction, clip flags) written on the audio
thread and read on the UI timer thread do not need a lock. On x64, `volatile`
float and `volatile` int fields provide sufficient ordering guarantees for a
single-writer / single-reader scenario:

```csharp
// Audio thread writes
volatile float _meterIn  = 0f;
volatile float _grDb     = 0f;
volatile int   _clip     = 0;

// UI thread reads (DispatcherTimer at ~30 fps)
public float MeterIn => _meterIn;
public float GrDb    => _grDb;
public int   Clip    => _clip;
```

The UI may read a value that is one block behind — this is acceptable and
imperceptible at 30 fps. Do not use `Interlocked` or `lock` for float
metering; the overhead is unnecessary and `Interlocked` does not support
`float` directly anyway.

For countdown integers (e.g. a clip indicator that auto-clears), the audio
thread sets the countdown and the UI thread decrements it — one writer per
direction, no torn reads on x64:

```csharp
// Audio thread: set when clipping
if (outPeak > threshold) _clip = CLIP_HOLD;

// UI thread: decrement each tick
if (_clip > 0) _clip--;
bool lit = _clip > 0;
```

---

## 8. Sharing DSP methods between machine and GUI classes

When the GUI needs to call a DSP computation (e.g. to draw a transfer curve
or compute a display value), mark the method `internal static`. Both classes
live in the same assembly so access is unrestricted, and `static` avoids
needing a machine reference from the GUI:

```csharp
// In the machine class:
internal static float SoftKneeGR(float dbIn, float T, float W, float ratio)
{
    // ... compression curve computation ...
}

// In the GUI class — called directly by class name:
float gr = PedalCompMachine.SoftKneeGR(inDb, threshDb, kneeF, ratioF);
```

Avoid duplicating DSP formulas in the GUI class — a single `internal static`
method is the authoritative implementation for both the audio path and any
visual representation of it.
