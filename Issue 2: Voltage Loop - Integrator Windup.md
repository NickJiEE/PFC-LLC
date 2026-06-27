# Voltage-Loop Issue: Integrator Windup

## Original Voltage-Loop Structure

The voltage loop was built as:

```text
Verror = 400 - Vout_filtered
```

with:

```text
Ton_P = Kp * Verror
Ton_I = integral(Ki * Verror)
```

The requested on-time was then:

```text
Ton_unlimited = Ton_bias + Ton_P + Ton_I
```

and limited by:

```text
Ton_cmd = Saturation(Ton_unlimited, Ton_min, Ton_max)
```

For this model:

```text
Ton_bias = 7.1 us
Ton_max  = 8.0 us
```

---

## Original Symptom

During startup, or whenever `Vout` was well below 400 V:

- The voltage error was strongly positive.
- `Ton_P` was large.
- `Ton_unlimited` exceeded 8 us.
- The Saturation block correctly held `Ton_cmd` at 8 us.
- However, `Ton_I` continued increasing even though the actual command could not increase.

The waveform showed:

```text
Ton_cmd = constant at 8 us
Ton_I   = continuously increasing
```

This is known as **integrator windup**.

---

## Why the Saturation Block Was Not Enough

The Saturation block only limits the signal sent to the on-time timer:

```text
Ton_unlimited = 9 us
Ton_cmd       = 8 us
```

But it does not affect the internal integral state.

While the error remains positive, the integrator continues storing correction:

```text
Ton_I = 0
Ton_I = 0.1 us
Ton_I = 0.2 us
Ton_I = 0.3 us
...
```

The hidden unlimited command can therefore continue growing even though the converter is already operating at the maximum allowed on-time.

For example:

```text
Ton_bias = 7.1 us
Ton_P    = 0.5 us
Ton_I    = 1.0 us

Ton_unlimited = 8.6 us
Ton_cmd       = 8.0 us
```

When the output finally reaches 400 V, the proportional term falls toward zero, but the stored integral term remains:

```text
Ton_bias = 7.1 us
Ton_P    = 0
Ton_I    = 1.0 us

Ton_unlimited = 8.1 us
Ton_cmd       = 8.0 us
```

So the converter can continue applying too much power even though the voltage error is already near zero.

---

## Resulting Problems

Without anti-windup clamping, the stored integral correction can cause:

- Output-voltage overshoot
- Slow recovery after reaching 400 V
- Long settling time
- `Ton_cmd` remaining saturated for too long
- Possible low-frequency oscillation

The integral term must first unwind through negative voltage error before the on-time command can decrease normally.

---

## Main Solution: Conditional Integrator Clamping

The solution was to freeze the integrator whenever:

1. The on-time command is already at a limit.
2. The voltage error would push it farther into that limit.

The upper-limit condition is:

```text
Ton_unlimited >= Ton_max
AND
Verror > 0
```

The lower-limit condition is:

```text
Ton_unlimited <= Ton_min
AND
Verror < 0
```

The complete blocking condition is:

```matlab
block_upper = ...
    (Ton_unlimited >= Ton_max) && ...
    (Verror > 0);

block_lower = ...
    (Ton_unlimited <= Ton_min) && ...
    (Verror < 0);

block_integrator = ...
    block_upper || block_lower;
```

---

## Integrator-Enable Logic

The integral action was also disabled until the output approached the target:

```matlab
near_target = Vout_filtered >= 350;
```

The final integrator-enable condition became:

```matlab
integrator_enable = ...
    near_target && ...
    ~block_integrator;
```

A Switch was placed before the integral path:

```matlab
if integrator_enable
    error_to_integrator = Verror;
else
    error_to_integrator = 0;
end
```

The integral path is therefore:

```text
Verror
  |
  v
Switch:
- pass Verror when enabled
- pass 0 when blocked
  |
  v
Ki
  |
  v
Discrete-Time Integrator
  |
  v
Ton_I
```

Passing zero freezes the integral state:

```text
Ton_I remains at its present value
```

It does not reset it.

---

## Why the Clamp Still Allows Recovery

The clamp only blocks integration when it would worsen saturation.

At the upper limit:

```text
Ton_cmd at maximum
Verror positive
-> freeze integrator
```

But if the voltage rises above 400 V:

```text
Ton_cmd at maximum
Verror negative
-> allow integration
```

The negative error reduces `Ton_I` and helps the controller leave upper saturation.

Similarly, at the lower limit:

```text
Ton_cmd at minimum
Verror negative
-> freeze integrator
```

but:

```text
Ton_cmd at minimum
Verror positive
-> allow integration
```

because positive integration helps move the command back upward.

---

## Behavior After the Fix

Before clamping:

```text
Ton_cmd = 8 us
Ton_I continuously increases
```

After clamping:

```text
Ton_cmd = 8 us
Ton_I remains constant
```

Once the proportional term decreases enough that:

```text
Ton_unlimited < 8 us
```

the clamp releases:

```text
block_integrator = 0
integrator_enable = 1
```

and `Ton_I` begins adjusting normally.

The updated waveform showed:

```text
Ton_cmd initially saturated at 8 us
Ton_I held at zero
Ton_P decreases
Ton_cmd leaves saturation
Ton_I begins increasing
Ton_cmd settles toward the required steady-state value
```

---

## Final Voltage-Loop Logic

```matlab
Verror = Vref - Vout_filtered;

Ton_P = Kp * Verror;

Ton_unlimited = ...
    Ton_bias + Ton_P + Ton_I;

at_upper = Ton_unlimited >= Ton_max;
at_lower = Ton_unlimited <= Ton_min;

block_integrator = ...
    (at_upper && Verror > 0) || ...
    (at_lower && Verror < 0);

integrator_enable = ...
    (Vout_filtered >= Vint_start) && ...
    ~block_integrator;

if integrator_enable
    error_to_integrator = Verror;
else
    error_to_integrator = 0;
end

Ton_I = integral(Ki * error_to_integrator);

Ton_cmd = saturate( ...
    Ton_bias + Ton_P + Ton_I, ...
    Ton_min, Ton_max);
```

---

## Main Takeaway

The main fix was changing the integral path from:

```text
Always integrate the voltage error
```

to:

```text
Integrate only when the integral action can move Ton_cmd in a useful direction
```

The Saturation block protects the actual on-time command, while the clamping logic protects the internal PI state.
