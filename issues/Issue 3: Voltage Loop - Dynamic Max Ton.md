# Issue 3: High-Line Overshoot from a Fixed Maximum On-Time

## Problem

The voltage loop generated:

```text
Ton_ff + Ton_P + Ton_I
→ Ton_unlimited
→ saturation
→ Ton_cmd
```

Originally, the upper saturation limit was fixed:

```matlab
Ton_max = 8e-6;
```

The same `8 us` maximum was therefore allowed across the entire `90–264 VAC` input range.

At high line, the large startup voltage error could drive the command to this low-line-sized maximum and transfer excessive energy per switching cycle.

---

## Why the fixed limit was incorrect

Input-voltage feedforward estimates the nominal on-time as:

```matlab
Ton_ff = Kff / Vin_rms^2;
```

Using:

```matlab
Ton_nom = 7.1e-6;
Vin_nom = 90;
Kff = Ton_nom * Vin_nom^2;
```

gives:

```matlab
Kff = 0.05751;
```

Approximate feedforward values are:

```text
90 VAC  → Ton_ff ≈ 7.10 us
240 VAC → Ton_ff ≈ 1.00 us
264 VAC → Ton_ff ≈ 0.825 us
```

At high line:

```text
Vout initially below reference
→ large positive Verror
→ Ton_P requests additional on-time
→ Ton_unlimited can become much larger than the required high-line value
```

A fixed `8 us` maximum therefore allowed far too much active-switching energy at high line.

---

## Final dynamic maximum on-time

The fixed upper limit was replaced by:

```matlab
K_Ton_max = 1.25;

Ton_max_candidate = K_Ton_max * Ton_ff;
Ton_max_dynamic = min(Ton_max_absolute, Ton_max_candidate);
```

with:

```matlab
Ton_max_absolute = 8e-6;
```

The command is:

```matlab
Ton_cmd = min( ...
    max(Ton_unlimited, Ton_min), ...
    Ton_max_dynamic);
```

Equivalent signal path:

```text
Ton_ff ──> Gain 1.25 ──┐
                       Min ──> Ton_max_dynamic
8 us absolute max ─────┘

Ton_unlimited
→ Saturation Dynamic
   lower = Ton_min
   upper = Ton_max_dynamic
→ Ton_cmd
```

---

## Final dynamic limits

At `90 VAC`:

```text
Ton_ff ≈ 7.10 us
1.25 × Ton_ff ≈ 8.88 us
Ton_max_dynamic = 8.00 us
```

At `264 VAC`:

```text
Ton_ff ≈ 0.825 us
1.25 × Ton_ff ≈ 1.031 us
Ton_max_dynamic ≈ 1.031 us
```

The controller therefore retains full low-line power capability while preventing a low-line-sized pulse at high line.

---

## Anti-windup update

The anti-windup comparison must use the same dynamic maximum:

```matlab
at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= Ton_min;
```

Directional clamping is:

```matlab
block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

integrator_enable = ~(block_upper || block_lower);
```

Using the old fixed `8 us` value in the anti-windup comparison would allow the integrator to wind up while the command was already limited by the much smaller high-line dynamic maximum.

---

## Final timer integration

The dynamic saturation output is not connected directly to an analog pulse generator.

The final path is:

```text
Ton_cmd
→ Rate Transition
→ Ton_cmd_fast
→ discrete on-time timer
→ Ton_done
→ latch RESET
```

The fast timer sample time is:

```matlab
Ts_ctrl = 100e-9;
```

The timer counts while the gate latch is on and generates `Ton_done` when the elapsed on-time reaches `Ton_cmd_fast`.

Because the timer is discrete, the actual measured gate pulse is quantized in `100 ns` increments and can be approximately `0.1–0.2 us` longer than the requested command.

---

## Interaction with the final startup system

The final converter also includes:

```text
An 82 ohm precharge resistor
A bypass switch
A delayed controller enable at 0.160 s
A startup pulse at 0.161 s
A disconnected load during bus precharge
```

These additions further reduce startup stress.

However, the dynamic maximum remains necessary after the controller is enabled because it prevents the voltage loop from requesting an excessive active-switching pulse at high line.

The RMS estimator uses a `50 ms` window and has settled before the final controller-enable time, so the high-line dynamic limit is already based on the measured input when active switching starts.

---

## Effect on validation

With the fixed maximum, high-line startup behavior could be dominated by an unrealistic `8 us` active pulse. This invalidated the assessment of OVP behavior and semiconductor current stress.

With the dynamic maximum:

```text
High-line Ton_cmd is limited near the feedforward requirement.
The PI cannot request a low-line-sized pulse.
The timer receives a physically appropriate command.
High-line startup no longer depends on the fixed 8 us ceiling.
```

---

## Final implementation

```matlab
K_Ton_max = 1.25;

Ton_ff = Kff / Vin_rms_sq_limited;

Ton_max_candidate = K_Ton_max * Ton_ff;
Ton_max_dynamic = min(8e-6, Ton_max_candidate);

Ton_unlimited = Ton_ff + Ton_P + Ton_I;

Ton_cmd = min( ...
    max(Ton_unlimited, 0.2e-6), ...
    Ton_max_dynamic);

at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= 0.2e-6;

block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

integrator_enable = ~(block_upper || block_lower);
```

---

## Main conclusion

The original controller used one maximum on-time for every input voltage.

The final correction uses the feedforward term to create an input-dependent maximum:

```text
Ton_max_dynamic = min(8 us, 1.25 × Ton_ff)
```

The same limit is used by both the dynamic saturation and anti-windup logic, and the resulting command is transferred to the `100 ns` discrete on-time timer.
