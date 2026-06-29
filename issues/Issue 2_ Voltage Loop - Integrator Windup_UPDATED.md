# Issue 2: Voltage-Loop Integrator Windup

## Problem

The voltage loop generated the commanded MOSFET on-time from:

```matlab
Verror = Vref - Vout_filtered;

Ton_P = Kp * Verror;
Ton_I = integral(Ki * Verror);

Ton_unlimited = Ton_ff + Ton_P + Ton_I;
```

The final command was limited between a minimum on-time and an input-dependent maximum on-time.

During startup or a large positive voltage error:

```text
Ton_unlimited exceeded the allowed maximum.
Ton_cmd was correctly saturated.
Ton_I continued increasing.
```

The saturation block limited the command sent to the timer, but it did not protect the internal integral state.

This caused integrator windup.

---

## Why saturation alone was insufficient

For example:

```text
Ton_unlimited = 1.4 us
Ton_max_dynamic = 1.0 us
Ton_cmd = 1.0 us
```

If the voltage error remained positive, the integrator continued increasing even though the actual command could not increase.

When the output approached the reference:

```text
Ton_P decreased
but
the stored Ton_I remained positive
```

The controller could therefore remain at its upper limit too long, causing:

```text
Output-voltage overshoot
Slow recovery
Long settling time
Possible low-frequency oscillation
```

---

## Final directional anti-windup logic

The integrator is blocked only when the error would push the command farther into saturation.

The final limit checks are:

```matlab
at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= Ton_min;
```

Directional blocking is:

```matlab
block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

block_integrator = block_upper || block_lower;
integrator_enable = ~block_integrator;
```

The integral input becomes:

```matlab
if integrator_enable
    error_to_integrator = Verror;
else
    error_to_integrator = 0;
end
```

Passing zero freezes the discrete integrator state. It does not reset the integral term.

---

## Why directional clamping still allows recovery

At the upper limit:

```text
Positive Verror would increase Ton_I
→ block integration
```

but:

```text
Negative Verror would decrease Ton_I
→ allow integration
```

At the lower limit:

```text
Negative Verror would decrease Ton_I
→ block integration
```

but:

```text
Positive Verror would increase Ton_I
→ allow integration
```

The integrator is therefore blocked only when it would worsen saturation.

---

## Final voltage-loop structure

The current controller uses:

```matlab
Verror = Vref - Vout_filtered;

Ton_P = Kp * Verror;
Ton_unlimited = Ton_ff + Ton_P + Ton_I;

Ton_max_dynamic = min(Ton_max_absolute, K_Ton_max * Ton_ff);

Ton_cmd = saturate_dynamic( ...
    Ton_unlimited, ...
    Ton_min, ...
    Ton_max_dynamic);
```

Final parameter values include:

```matlab
Kp = 0.05e-6;
Ki = 2e-7;

Ton_min = 0.2e-6;
Ton_max_absolute = 8e-6;

Ton_I_min = -1e-6;
Ton_I_max =  1e-6;
```

The integral contribution is itself limited:

```matlab
Ton_I = saturate( ...
    integral(Ki * error_to_integrator), ...
    Ton_I_min, ...
    Ton_I_max);
```

This additional clamp prevents an excessive stored integral correction even if another operating condition temporarily delays the directional anti-windup response.

---

## Important final-system update

An earlier version used a separate condition such as:

```text
Enable the integrator only after Vout exceeds a startup threshold.
```

That threshold gate is not part of the final anti-windup definition.

The final integrator enable is based on directional saturation logic:

```matlab
integrator_enable = ...
    ~((Ton_unlimited >= Ton_max_dynamic && Verror > 0) || ...
      (Ton_unlimited <= Ton_min && Verror < 0));
```

The controller startup itself is handled separately by `ControllerEnable`, the latch reset path, and the startup pulse.

---

## Effect on validation

Without anti-windup, a saturated `Ton_cmd` did not prove that the internal PI state was valid. The stored integral term could still cause later overshoot or slow recovery.

After directional clamping:

```text
Ton_I freezes when it would worsen saturation.
Ton_I is allowed to unwind when the error reverses.
Ton_cmd leaves saturation normally.
The output settles without a long stored-error recovery.
```

This makes startup and transient validation representative of the actual controller rather than a saturated command hiding a growing integral state.

---

## Final implementation

```matlab
Verror = Vref - Vout_filtered;

Ton_P = Kp * Verror;
Ton_unlimited = Ton_ff + Ton_P + Ton_I;

at_upper = Ton_unlimited >= Ton_max_dynamic;
at_lower = Ton_unlimited <= Ton_min;

block_upper = at_upper && (Verror > 0);
block_lower = at_lower && (Verror < 0);

integrator_enable = ~(block_upper || block_lower);

if integrator_enable
    error_to_integrator = Verror;
else
    error_to_integrator = 0;
end

Ton_I_raw = discrete_integral(Ki * error_to_integrator);
Ton_I = min(max(Ton_I_raw, -1e-6), 1e-6);

Ton_cmd = min( ...
    max(Ton_unlimited, Ton_min), ...
    Ton_max_dynamic);
```

---

## Main conclusion

The issue was not that the saturation block failed. The saturation block correctly limited the timer command.

The problem was that the integral state continued growing behind the saturation.

The final fix combines:

```text
Directional integrator clamping
+
dynamic-limit awareness
+
a bounded integral contribution
```

This prevents windup while still allowing the controller to recover from either saturation limit.
