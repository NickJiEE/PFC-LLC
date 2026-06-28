# Issue 4: RMS Feedforward Error with 50/60 Hz Input Frequency

## Feedforward goal

The CrCM controller uses input-voltage feedforward to estimate the nominal MOSFET on-time:

```matlab
Ton_ff = Kff / Vin_rms^2;
```

Using:

```matlab
Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;
```

the expected values are approximately:

```text
90 VAC  → Ton_ff ≈ 7.1 µs
240 VAC → Ton_ff ≈ 1.0 µs
264 VAC → Ton_ff ≈ 0.83 µs
```

Therefore, the controller must obtain an accurate estimate of `Vin_rms^2`.

---

## Initial incorrect implementation

The first implementation used:

```text
Vac
 ↓
Moving Average
 ↓
Square
 ↓
Vin_rms²
```

This calculated:

```matlab
mean(Vac)^2
```

For a signed sinusoidal voltage:

```text
mean(Vac) ≈ 0
```

Therefore, the RMS estimate was forced to its minimum saturation limit and `Ton_ff` remained at its maximum value.

This caused both `90 VAC` and `240 VAC` tests to produce approximately:

```text
Ton_ff = 8 µs
```

---

## Correct RMS-squared calculation

The RMS relationship is:

```matlab
Vin_rms = sqrt(mean(Vac^2));
```

Because the feedforward equation already uses `Vin_rms^2`, the square root is unnecessary.

The corrected structure is:

```text
Vac
 ↓
Zero-Order Hold
 ↓
Square
 ↓
Moving Average
 ↓
Vin_rms² estimate
 ↓
Saturation
 ↓
Kff / Vin_rms²
 ↓
Ton_ff
```

Equivalent expression:

```matlab
Vin_rms_sq_est = mean(Vac^2);
Ton_ff = Kff / Vin_rms_sq_est;
```

A square-root branch may be added only for monitoring:

```text
Vin_rms² estimate
      ├──> feedforward calculation
      │
      └──> sqrt ──> Scope: Vin_rms
```

---

## New symptom with a fixed 50 Hz averaging setting

With the Moving Average block configured for `50 Hz`, the feedforward worked correctly for a `50 Hz` input.

However, when the source frequency was changed to `60 Hz`, the estimated RMS voltage oscillated.

The on-time command also oscillated:

```text
Vin_rms estimate oscillated
→ Vin_rms² oscillated
→ Ton_ff oscillated
```

When the Moving Average block was changed from `50 Hz` to `60 Hz`, the oscillation disappeared.

---

## Why the frequency mismatch caused ripple

For a sinusoidal input:

```matlab
Vac = Vpk * sin(2*pi*Fac*t);
```

After squaring:

```matlab
Vac^2 = Vin_rms^2 * (1 - cos(4*pi*Fac*t));
```

The squared waveform contains:

```text
DC component = Vin_rms²
AC ripple frequency = 2 × Fac
```

Therefore:

```text
50 Hz input → Vac² ripple at 100 Hz
60 Hz input → Vac² ripple at 120 Hz
```

A moving-average window cancels the ripple only when it contains an integer number of ripple cycles.

Using a `50 Hz` line-cycle window:

```text
Window length = 1 / 50 = 20 ms
```

For a `60 Hz` source:

```text
Vac² ripple period = 1 / 120 = 8.33 ms
20 ms contains 2.4 ripple cycles
```

Because the window does not contain an integer number of cycles:

```text
The 120 Hz component does not average to zero
→ the RMS estimate oscillates
→ Ton_ff oscillates
```

---

## Considered solution: automatic frequency tracking

A PLL and a variable-frequency moving-average block were tested.

The intended structure was:

```text
Vac
 ├──> PLL ──> frequency estimate ──> Gain ×2 ──┐
 │                                             │
 └──> Square ──> Moving Average Variable Freq ─┘
```

The factor of two was required because the moving-average input was `Vac²`, whose ripple frequency is twice the line frequency.

However, the PLL output showed a large startup transient before settling near the correct frequency.

For a `60 Hz` input:

```text
PLL output eventually approached 60 Hz
Gain ×2 eventually approached 120 Hz
```

But during startup:

```text
PLL frequency estimate changed sharply
→ averaging-window length changed sharply
→ Vin_rms² estimate experienced large transients
→ Ton_ff became unstable during startup
```

The PLL-based method worked in principle but added unnecessary complexity.

---

## Main solution

A fixed averaging window of `50 ms` was selected:

```matlab
Favg = 20;
```

because:

```text
1 / 20 Hz = 50 ms
```

This window works for both nominal line frequencies.

For `50 Hz` input:

```text
Vac² ripple frequency = 100 Hz
50 ms contains 5 complete ripple cycles
```

For `60 Hz` input:

```text
Vac² ripple frequency = 120 Hz
50 ms contains 6 complete ripple cycles
```

Both are integer numbers of cycles.

Therefore:

```text
The AC ripple in Vac² averages to zero
→ Vin_rms² becomes nearly constant
→ Ton_ff becomes nearly constant
→ no frequency detector or PLL is required
```

---

## Final feedforward structure

```text
Vac
 ↓
Zero-Order Hold
 ↓
Square
 ↓
Moving Average
Fundamental frequency = Favg = 20 Hz
 ↓
Vin_rms² estimate
 ↓
Saturation [Vin_ff_min², Vin_ff_max²]
 ↓
Kff / Vin_rms²
 ↓
Saturation [Ton_ff_min, Ton_ff_max]
 ↓
Ton_ff
```

Recommended variables:

```matlab
Fac = 50;          % actual source frequency: 50 or 60 Hz
Favg = 20;         % fixed 50 ms averaging window

Vin_ff_min = 80;
Vin_ff_max = 280;

Vin_rms_sq_min = Vin_ff_min^2;
Vin_rms_sq_max = Vin_ff_max^2;

Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;
```

The source frequency and averaging frequency are separate variables:

```text
Fac  = actual AC source frequency
Favg = moving-average window parameter
```

The AC source must not be set to `Favg`.

---

## Initial-condition behavior

The Moving Average input is `Vac²`, so the initial value must also be in volts squared.

A conservative high-line initial estimate was used:

```matlab
Vin_rms_init = 280;
Vin_sq_init = Vin_rms_init^2;
```

For the first `50 ms`:

```text
Vin_rms estimate = 280 V
Ton_ff ≈ Kff / 280²
Ton_ff ≈ 0.733 µs
```

At the end of the averaging window, the estimate updates to the measured input voltage.

At `240 VAC`:

```text
0 to 50 ms:
Vin_rms ≈ 280 V
Ton_ff ≈ 0.733 µs

After 50 ms:
Vin_rms ≈ 240 V
Ton_ff ≈ 1.0 µs
```

At `90 VAC`:

```text
0 to 50 ms:
Vin_rms ≈ 280 V
Ton_ff ≈ 0.733 µs

After 50 ms:
Vin_rms ≈ 90 V
Ton_ff ≈ 7.1 µs
```

The first `50 ms` is not a dead time.

The controller still operates using the conservative initial estimate.

---

## Why the delay was not initially obvious

The moving-average block does not leave its output blank while the window fills.

Instead, it outputs the configured initial value.

Therefore:

```text
The converter begins operating immediately
→ the first 50 ms uses Vin_rms_init
→ the measured estimate replaces it after one complete window
```

The delay appears as a step in the RMS estimate and feedforward command at approximately:

```text
t = 0.05 s
```

At high line, the change is small:

```text
280 V estimate → 240 V measured
0.733 µs → approximately 1.0 µs
```

At low line, the change is much larger:

```text
280 V estimate → 90 V measured
0.733 µs → approximately 7.1 µs
```

This explains the temporary low-line plateau near the passive rectifier voltage.

---

## Result after the fix

With `Favg = 20 Hz`:

```text
Vin_rms estimate became constant at 90 VAC.
Ton_ff settled near 7.1 µs.

Vin_rms estimate became constant at 240 VAC.
Ton_ff settled near 1.0 µs.
```

The same averaging configuration worked for both `50 Hz` and `60 Hz` source tests.

The PLL and variable-frequency moving average were no longer required.

---

## Final implementation

```matlab
% AC source
Fac = 50;              % set to 50 or 60 for the test

% RMS-squared averaging
Favg = 20;             % 50 ms window, valid for both 50 and 60 Hz

% Feedforward limits
Vin_ff_min = 80;
Vin_ff_max = 280;

Vin_rms_sq_min = Vin_ff_min^2;
Vin_rms_sq_max = Vin_ff_max^2;

% Conservative moving-average initial condition
Vin_rms_init = 280;
Vin_sq_init = Vin_rms_init^2;

% Feedforward coefficient
Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;

% Calculation
Vin_rms_sq_est = moving_average(Vac^2);
Vin_rms_sq_limited = min( ...
    max(Vin_rms_sq_est, Vin_rms_sq_min), ...
    Vin_rms_sq_max);

Ton_ff = Kff / Vin_rms_sq_limited;
```

---

## Main conclusion

The RMS feedforward problem had two separate causes:

```text
1. The original block order calculated mean(Vac)² instead of mean(Vac²).

2. A line-frequency-sized averaging window only worked when its configured
   frequency matched the actual 50/60 Hz source.
```

The final correction was:

```text
Square the input voltage before averaging
+
use one fixed 50 ms averaging window
```

This produced a stable RMS-squared estimate and correct feedforward on-time for both nominal `50 Hz` and `60 Hz` operation without requiring frequency tracking.
