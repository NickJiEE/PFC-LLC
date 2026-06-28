# Issue 3: High-Line Startup Overshoot from a Fixed Maximum On-Time

## Controller structure

The voltage loop generates the commanded MOSFET on-time:

```text
Ton_ff + Ton_P + Ton_I
          ↓
     Ton_unlimited
          ↓
       Saturation
          ↓
        Ton_cmd
```

Originally, the final saturation limits were fixed:

```matlab
Ton_min = 0.2e-6;
Ton_max = 8.0e-6;

Ton_cmd = min(max(Ton_unlimited, Ton_min), Ton_max);
```

The same maximum on-time of `8 µs` was therefore allowed at every input voltage.

---

## Original symptom

At low line, around `90 VAC`, the converter operated correctly with an on-time near:

```text
Ton ≈ 7.1 µs
```

However, at high line, around `240 VAC`, the output voltage showed an undesirable startup transient:

```text
Vout rose rapidly
→ approached or exceeded the 420 V OVP threshold
→ switching stopped
→ the output capacitor discharged
→ switching restarted later
```

The output waveform could rise toward approximately `420 V`, fall toward the passive rectifier region, and then recover.

Even when OVP did not fully trigger, the startup overshoot was too close to the protection threshold.

---

## Why the fixed maximum caused the problem

Input-voltage feedforward produced the approximate nominal on-time:

```matlab
Ton_ff = Kff / Vin_rms^2;
```

Using:

```matlab
Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;
```

the expected feedforward values were approximately:

```text
90 VAC  → Ton_ff ≈ 7.1 µs
240 VAC → Ton_ff ≈ 1.0 µs
264 VAC → Ton_ff ≈ 0.83 µs
```

At startup, the output-voltage error is initially large:

```matlab
Verror = Vref - Vout_filtered;
```

When `Vout` is low:

```text
Verror is large and positive
→ Ton_P becomes large
→ Ton_unlimited becomes much greater than the required high-line on-time
```

For example, at `240 VAC`:

```text
Ton_ff ≈ 1.0 µs
Ton_P can initially request many additional microseconds
Ton_unlimited may exceed 8 µs
```

The fixed saturation only limited this command to:

```text
Ton_cmd = 8 µs
```

An `8 µs` pulse is reasonable near `90 VAC`, but it is far too large near `240–264 VAC`.

Therefore:

```text
Large startup voltage error
→ PI requests excessive on-time
→ fixed 8 µs limit allows too much energy per cycle
→ inductor current and transferred power become excessive
→ output overshoots
→ OVP may activate
```

---

## Main solution

The fixed maximum on-time was replaced by an input-voltage-dependent maximum:

```matlab
K_Ton_max = 1.25;

Ton_max_candidate = K_Ton_max * Ton_ff;
Ton_max_dynamic = min(Ton_max, Ton_max_candidate);
```

The final command is now:

```matlab
Ton_cmd = min( ...
    max(Ton_unlimited, Ton_min), ...
    Ton_max_dynamic);
```

Equivalent block structure:

```text
Ton_ff
  ↓
Gain = 1.25
  ↓
Ton_max_candidate ───────────┐
                             ↓
                         MinMax: min
                             ↑
Ton_max = 8 µs ──────────────┘
                             ↓
                    Ton_max_dynamic
                             ↓
              upper limit of Saturation Dynamic
```

The complete command path becomes:

```text
Ton_ff ─────┐
Ton_P ──────┼──> Sum ──> Ton_unlimited ──> Saturation Dynamic ──> Ton_cmd
Ton_I ──────┘                               ↑                 ↑
                                            │                 │
                                  Ton_max_dynamic          Ton_min
```

---

## Expected dynamic limits

With:

```matlab
K_Ton_max = 1.25;
Ton_max = 8e-6;
```

the approximate upper limits become:

```text
90 VAC:
Ton_ff ≈ 7.10 µs
1.25 × Ton_ff ≈ 8.88 µs
Ton_max_dynamic = 8.00 µs

240 VAC:
Ton_ff ≈ 1.00 µs
1.25 × Ton_ff ≈ 1.25 µs
Ton_max_dynamic ≈ 1.25 µs

264 VAC:
Ton_ff ≈ 0.83 µs
1.25 × Ton_ff ≈ 1.03 µs
Ton_max_dynamic ≈ 1.03 µs
```

This preserves the required low-line power capability while preventing excessively long pulses at high line.

---

## Anti-windup update

The integrator-clamping logic must use the dynamic upper limit.

The old comparison:

```matlab
at_upper = Ton_unlimited >= Ton_max;
```

was replaced by:

```matlab
at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= Ton_min;
```

Directional clamping remains:

```matlab
block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

block_integrator = block_upper || block_lower;
integrator_enable = ~block_integrator;
```

This prevents the integrator from winding up while the command is limited by the smaller high-line maximum.

---

## Why the new method works

At high line:

```text
PI requests a long on-time
→ Ton_unlimited exceeds Ton_max_dynamic
→ Saturation Dynamic limits Ton_cmd near 1.0–1.3 µs
→ excessive startup energy is prevented
→ Vout rises without approaching OVP
```

At low line:

```text
Ton_ff is already near 7.1 µs
→ 1.25 × Ton_ff exceeds the fixed 8 µs absolute limit
→ Ton_max_dynamic remains 8 µs
→ low-line behavior is mostly unchanged
```

The dynamic limit does not force the converter to operate at `1.25 × Ton_ff`.

It only prevents:

```text
Ton_cmd > Ton_max_dynamic
```

The voltage PI can still reduce the command below this limit during normal regulation.

---

## Result after the fix

At `240 VAC`:

```text
The previous startup overshoot disappeared.
Vout rose smoothly from the rectified-input level.
The output approached approximately 400 V without reaching 420 V.
No large OVP shutdown and discharge cycle was observed.
```

At `90 VAC`:

```text
The converter still reached the intended operating region.
The startup remained stable.
The absolute maximum remained 8 µs.
```

The low-line waveform showed an initial plateau near:

```text
90 × sqrt(2) ≈ 127 V
```

because the RMS estimator initially assumed a conservative high-line voltage. After the RMS averaging window completed, the dynamic limit increased and normal boosting began.

---

## Final implementation

```matlab
K_Ton_max = 1.25;

Ton_max_candidate = K_Ton_max * Ton_ff;
Ton_max_dynamic = min(Ton_max, Ton_max_candidate);

Ton_unlimited = Ton_ff + Ton_P + Ton_I;

Ton_cmd = min( ...
    max(Ton_unlimited, Ton_min), ...
    Ton_max_dynamic);

at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= Ton_min;

block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

integrator_enable = ~(block_upper || block_lower);
```

---

## Main conclusion

The original controller used one fixed maximum on-time for the entire `90–264 VAC` input range.

That allowed the voltage loop to command a low-line-sized pulse at high line.

The main correction was:

```text
Use Ton_ff to create an input-dependent maximum on-time
→ apply that value to a Saturation Dynamic block
→ use the same dynamic limit in the anti-windup logic
```

This directly reduced high-line startup overshoot while preserving low-line operation.
