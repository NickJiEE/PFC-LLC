# Issue 4: RMS Feedforward Error with 50/60 Hz Input

## Problem

The CrCM controller uses input-voltage feedforward:

```matlab
Ton_ff = Kff / Vin_rms^2;
```

The controller therefore requires a stable estimate of the input mean-square voltage.

Two separate errors were found:

```text
1. The original block order calculated mean(Vac)^2 instead of mean(Vac^2).

2. A line-frequency-sized moving-average window worked only when its
   configured frequency matched the actual 50 Hz or 60 Hz source.
```

---

## Incorrect block order

The first implementation was:

```text
Vac
→ Moving Average
→ Square
```

This calculates:

```matlab
mean(Vac)^2
```

For a signed sinusoidal voltage:

```text
mean(Vac) ≈ 0
```

The estimate therefore collapsed toward the minimum limit and caused the feedforward command to remain too large.

---

## Correct mean-square calculation

The correct order is:

```text
Vac
→ sampled voltage path
→ Square
→ Moving Average
→ Vin_rms_sq_est
```

Equivalent calculation:

```matlab
Vin_rms_sq_est = mean(Vac^2);
Ton_ff = Kff / Vin_rms_sq_est;
```

A square root is unnecessary in the control path because the feedforward equation already uses `Vin_rms^2`.

A square-root branch may be used only for monitoring:

```text
Vin_rms_sq_est
├── feedforward calculation
└── sqrt → displayed Vin_rms
```

---

## Why a 50 Hz or 60 Hz window did not work universally

For:

```matlab
Vac = Vpk * sin(2*pi*Fac*t);
```

the squared waveform contains:

```text
A DC component equal to Vin_rms^2
A ripple component at 2 × Fac
```

Therefore:

```text
50 Hz source → Vac^2 ripple at 100 Hz
60 Hz source → Vac^2 ripple at 120 Hz
```

A moving-average window cancels the ripple only when it contains an integer number of ripple periods.

A `20 ms` window contains:

```text
2 complete 100 Hz cycles
2.4 cycles of 120 Hz ripple
```

It therefore works at `50 Hz` but leaves residual ripple at `60 Hz`.

The same problem occurs in reverse if the window is selected only for `60 Hz`.

---

## Final fixed averaging window

The final moving-average parameter is:

```matlab
Favg = 20;
```

which corresponds to:

```text
Window length = 1 / 20 Hz = 50 ms
```

For a `50 Hz` source:

```text
Vac^2 ripple = 100 Hz
50 ms contains 5 complete ripple cycles
```

For a `60 Hz` source:

```text
Vac^2 ripple = 120 Hz
50 ms contains 6 complete ripple cycles
```

The same window therefore cancels the squared-voltage ripple at both nominal line frequencies.

No PLL or variable-frequency moving-average block is required.

---

## Final feedforward path

```text
Vac
→ sampling block
→ Square
→ Moving Average, Favg = 20 Hz
→ Vin_rms_sq_est
→ Saturation
→ Kff / Vin_rms_sq_limited
→ Ton_ff
```

Final limits and coefficient:

```matlab
Vin_ff_min = 80;
Vin_ff_max = 280;

Vin_rms_sq_min = Vin_ff_min^2;
Vin_rms_sq_max = Vin_ff_max^2;

Ton_nom = 7.1e-6;
Vin_nom = 90;

Kff = Ton_nom * Vin_nom^2;
Kff = 0.05751;
```

The feedforward calculation is:

```matlab
Vin_rms_sq_limited = min( ...
    max(Vin_rms_sq_est, Vin_rms_sq_min), ...
    Vin_rms_sq_max);

Ton_ff = Kff / Vin_rms_sq_limited;
```

The source frequency and averaging parameter remain separate:

```text
Fac  = 50 Hz or 60 Hz source frequency
Favg = 20 Hz moving-average parameter
```

The AC source must not be set to `Favg`.

---

## Initial condition

The Moving Average block input is `Vac^2`, so its initial condition must also be in volts squared.

The final initial estimate is:

```matlab
Vin_rms_init = 280;
Vin_sq_init = 280^2;
```

This produces an initial feedforward value of approximately:

```text
Ton_ff ≈ 0.733 us
```

before the first complete `50 ms` window is available.

---

## Final-system startup behavior

Before the startup sequencer was added, the conservative initial estimate could temporarily hold the low-line command near its high-line value during the first `50 ms`.

In the final system:

```text
Controller is disabled during precharge.
ControllerEnable rises at 0.160 s.
Startup pulse occurs at 0.161 s.
```

The `50 ms` RMS averaging window has therefore completed well before active PFC switching begins.

As a result:

```text
The final controller starts with a settled Vin_rms^2 estimate.
The dynamic maximum on-time is already correct for the actual input.
The initial 280 V estimate no longer causes an active-switching plateau.
```

The conservative initial condition remains useful because it prevents division by an uninitialized or unrealistically small mean-square estimate.

---

## Interaction with dynamic maximum on-time

The final feedforward value is used both for the nominal command and for the dynamic maximum:

```matlab
Ton_max_dynamic = min(8e-6, 1.25 * Ton_ff);
```

An oscillating RMS estimate would therefore disturb both:

```text
Ton_ff
and
Ton_max_dynamic
```

The fixed `50 ms` window prevents this disturbance at both `50 Hz` and `60 Hz`.

---

## Effect on validation

An incorrect or oscillating RMS estimate changes the commanded on-time even when the output-voltage loop is unchanged.

This can distort:

```text
Power factor
Line-current envelope
Output-voltage regulation
Switching frequency
High-line current stress
Startup energy
```

After the final correction:

```text
Vin_rms_sq_est is stable at both 50 Hz and 60 Hz.
Ton_ff settles to the expected value.
Ton_max_dynamic remains stable.
The same controller configuration is valid for all four input corners.
```

---

## Final implementation

```matlab
Fac = 50;              % set to 50 or 60 for the test
Favg = 20;             % fixed 50 ms averaging window

Vin_ff_min = 80;
Vin_ff_max = 280;

Vin_rms_sq_min = Vin_ff_min^2;
Vin_rms_sq_max = Vin_ff_max^2;

Vin_rms_init = 280;
Vin_sq_init = Vin_rms_init^2;

Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;

Vin_rms_sq_est = moving_average(Vac^2);

Vin_rms_sq_limited = min( ...
    max(Vin_rms_sq_est, Vin_rms_sq_min), ...
    Vin_rms_sq_max);

Ton_ff = Kff / Vin_rms_sq_limited;
Ton_max_dynamic = min(8e-6, 1.25 * Ton_ff);
```

---

## Main conclusion

The final correction is:

```text
Square the signed input voltage before averaging
+
use one fixed 50 ms averaging window
```

This produces a stable mean-square voltage estimate for both `50 Hz` and `60 Hz` inputs and supplies a correct, stable feedforward value to the voltage loop and dynamic maximum-on-time logic.
