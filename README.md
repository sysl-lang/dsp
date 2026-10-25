# dsp

Audio DSP for [sysl](https://github.com/sysl-lang/sysl), written once and run at `q15`, `q31` and
`f32`: biquads with the RBJ cookbook's designs, FIR filters, gain and mixing that clip rather than
wrap, a phase-accumulator oscillator, an ADSR envelope, level meters, and windows that lead into
[`sh.sysl.fft`](https://github.com/sysl-lang/fft).

```
sh/sysl/dsp/
    sample.sysl       the Sample trait, and q15, q31 and f32 as samples
    biquad.sysl       the RBJ cookbook designs, and a biquad that rounds once per sample
    fir.sysl          FIR filtering over a delay line the caller supplies
    gain.sysl         gain and mixing, clipped at full scale
    oscillator.sysl   sine (a const-built table, interpolated), saw, square and triangle
    envelope.sysl     ADSR, each stage exact to the sample
    meter.sysl        peak, RMS and dBFS
    window.sysl       Hann and Hamming, into the complex buffer fft transforms
    fault.sysl        what a construction refuses
    tests.sysl        what all of it claims, run by `sysl test .`
sysl.hocon            who this package is, and what it needs of the machine
```

The module is **`sh.sysl.dsp`**. It needs no heap: every block keeps its state in its own fields,
and anything longer, such as an FIR's delay line or a spectrum's buffer, is storage the caller
passes in.

## Using it

```
dependencies {
  dsp { git = "github.com/sysl-lang/dsp", version = "0.1.0" }
}
```

```
import sh.sysl.dsp.{Biquad, Fault, Oscillator, Sine, low_pass}

val osc: Result[Oscillator[q15], Fault] = Oscillator.of(Sine, 440.0, 48000.0)
val lp: Result[Biquad[q15], Fault] = Biquad.of(low_pass(1000.0, 48000.0, 0.7071)?)

var o = osc?
var f = lp?
var block: [64]q15 = [q15(0.0); 64]

o.process(block[..])
f.process_in_place(block[..])
```

Every block is built by `of`, which returns `Result[_, Fault]`: a frequency outside zero to Nyquist,
a Q or a rate that is not positive, a coefficient the sample type cannot hold, or storage of the
wrong length is refused there. Once a block is built, processing it cannot fail.
`process(*self, input, out)` writes the first `input.len` samples of `out`. `step` and `next` work
one sample at a time.

## The `Sample` trait

```
trait Sample: Zero + Add + Sub + Mul + Neg + Ord + Eq
    type Coef           // a coefficient or a gain
    type Acc: Add       // where a coefficient times a sample is summed
    ...
```

A coefficient has a wider range than a sample. A biquad's `a1` approaches 2 and a shelf's `b0` goes
past it, and a fixed-point sample cannot hold 1.0, so unity gain is not a sample either. For both
fixed-point types **`Coef` is `i32q28`**, which holds -8 up to just under 8.

**`Acc` is the exact product.** `q15 × i32q28` is exactly an `i48q43`, and `q31 × i32q28` is exactly
an `i64q59`. Each fixed-point sample therefore sums its products in that format, held in 64 bits and
made `wrapping`, so every product goes in unrounded. An overflow partway through a sum cancels when
the sum comes back into range, and the only rounding is the conversion back to a sample, which
saturates. For `f32`, `Acc` is `f32`. No language feature had to be added for this. sysl's
fixed-point product takes its format from the accumulator it is added into, and `S::Acc` is an
ordinary associated type.

## The blocks

- **Biquad.** One transposed-direct-form-II body serves all three types. At fixed point it computes
  exactly what direct form I computes, with one rounding per sample, and it carries the bits each
  rounding drops into the next sum (first-order error feedback). Without that feedback, a `q15`
  200 Hz high-pass at 48 kHz settles about 700 units away from zero on DC, because the feedback
  path's DC gain is 1486. With it, the filter settles within one unit. The designs are `low_pass`,
  `high_pass`, `band_pass`, `notch`, `peaking`, `low_shelf` and `high_shelf`. They are computed in
  `real` and quantized only when the filter is built.
- **FIR.** The delay line is twice as long as the taps, and each sample is written into both halves,
  so the most recent samples always form one contiguous run. For `q15` the sum is taken two lanes at
  a time and becomes `SMLAD` on a Cortex-M4.
- **Gain and mix.** These go through the exact accumulator and clip: `0.75 + 0.75` at `q15` is
  `0x7FFF`, never a wrap to the other sign.
- **Oscillator.** The phase is a `u32q32` that wraps, so the end of each cycle needs no comparison.
  The sine is a 257-entry `q15` table that a `const` function builds while compiling, read with
  linear interpolation. Its harmonic distortion is below -80 dB.
- **ADSR.** Each level is `from + (to - from) * k / n` in integers, so every stage ends exactly on
  its last sample.
- **Meters.** `peak`, `rms` (summed in `real`) and `dbfs`.
- **Windows.** `window_at` gives numpy's symmetric Hann and Hamming. `windowed` writes into a
  `[]Complex[F]` for `sh.sysl.fft.fft` to transform in place.

## What the tests check it against

The tests use 26 cases, and none of them checks this package against a value it produced itself.
One of them is the example above, compiled exactly as it is written.

- **Coefficients:** the RBJ formulas worked in numpy, to 1e-12.
- **Frequency response:** the RMS ratio of a sine through each design, at chosen frequencies and all
  three types, against `|H(e^jw)|` from numpy.
- **Direct form I:** the transposed form against direct form I written out by hand, sample by sample,
  with saturation included.
- **FIR:** `q15` and `q31` bit for bit against numpy's integer `convolve` rounded by hand, and `f32`
  against the float convolution.
- **Oscillator:** zero-crossing counts for every shape and type, and distortion measured with `fft`.
- **Clipping:** exactly at `0x7FFF`/`-0x8000`, with no wraparound.
- **Envelope:** each stage ends on its last sample.
- **Meters:** closed forms.
- **Edge cases:** empty input, one-sample input, taps at the ends of `q15`, DC, and every refusal.

## On a board

A probe running a `q15` oscillator, biquad and FIR, built with `sysl build-c --target
thumbv7em-freestanding`:

- **Biquad:** the per-sample loop is 46 instructions with no branches except the loop's own. Each of
  the five products is one `SMULL`/`SMLAL`, and the state stays in registers.
- **FIR:** the inner loop is `LDR`, `LDR`, `SMLAD`, two taps per step. An odd tap ends with one
  `SMLABB` and the output with `SSAT`.
- **Sine table:** 514 bytes in `.rodata`, with nothing run before `main`.
- **No heap:** the archive references no allocator. Its undefined symbols are the soft-double helpers
  and `sin`/`cos`, which only the coefficient design in `real` needs, plus `exit`/`putchar` for a
  failed `unwrap`.

On the host at `-O2`, the `f32` biquad runs at the speed of the same filter in C, and the `q15` one
within 1.3× of it.

## Who uses it

The first consumers will be [musicbox-pico](https://github.com/sysl-lang/musicbox-pico) and
[tuner](https://github.com/sysl-lang/tuner).

## Licence

ISC. See `LICENSE`.
