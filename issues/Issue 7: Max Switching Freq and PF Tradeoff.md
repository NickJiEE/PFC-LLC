# Issue 7: Maximum Switching-Frequency Clamp and Power-Factor Tradeoff

## Issue

The CrCM controller naturally produced a strongly varying switching frequency over each rectified AC line cycle.

At `264 VAC, 60 Hz`, the unclamped controller previously reached approximately:

```text
Average switching frequency: 494.4 kHz
Maximum switching frequency: 1.25 MHz
```

The high maximum frequency was undesirable for a practical implementation because it would increase:

```text
MOSFET switching loss
Gate-drive loss
Coss-related loss
Diode switching stress
Inductor core loss
EMI
Digital timing sensitivity
```

A maximum-frequency clamp was therefore added.

---

## Relationship to the varying-frequency behavior

The ideal CrCM switching frequency varies with instantaneous rectified input voltage:

$$
f_{\mathrm{sw,natural}}
\approx
\frac{V_{\mathrm{out}}-V_{\mathrm{in}}}
{V_{\mathrm{out}}T_{\mathrm{on}}}
$$

Near the line zero crossing:

```text
Vin approaches zero
→ inductor demagnetization becomes short
→ the next natural cycle begins quickly
→ switching frequency rises sharply
```

The varying frequency itself is normal CrCM behavior.

The issue documented here is the practical upper-frequency limit and its effect on input-current shape and PF.

---

## Maximum-frequency clamp

The controller was modified to enforce a minimum gate-to-gate period:

```matlab
Tcycle_min = 1/Fsw_max;
Ncycle_min = ceil(Tcycle_min/Ts_ctrl);
```

The next cycle begins only when both conditions are true:

```text
The inductor has returned to the valid zero-current restart state.
The minimum switching period has elapsed.
```

The restart logic is:

```matlab
period_done = cycle_count >= Ncycle_min;
restart_ready = zero_ready && period_done;
zcd_pulse_limited = rising_edge(restart_ready);
```

This preserves zero-current turn-on while limiting maximum switching frequency.

---

## Why a low clamp reduced PF

When the inductor current reached zero before the minimum period expired, the controller had to wait:

```text
iL reaches zero
→ zero_ready becomes true
→ period_done is still false
→ the converter remains idle
→ input current stays at zero temporarily
```

This forced idle interval was strongest near each AC zero crossing, where the natural CrCM frequency was highest.

The result was a visible notch and flattening in the source-current waveform near the zero crossings.

The PF reduction was therefore caused mainly by low-frequency current-envelope distortion, not by switching ripple.

This was confirmed because the source-side true PF and filtered-envelope PF remained nearly equal.

---

## 250 kHz test

The first selected limit was:

```matlab
Fsw_max = 250e3;
```

With the `100 ns` fast sample time and delayed counter reset, the measured maximum switching frequency was:

```text
243.902 kHz
```

The `264 VAC, 60 Hz` steady-state results were:

```text
Source-side true PF:   0.965344
Filtered-envelope PF:  0.965579
Modeled efficiency:    98.2883 %
Average Vout:          399.7358 V
Output ripple:         4.2866 Vpp
```

The original `PF > 0.95` requirement still passed, but zero-crossing distortion was clearly visible.

---

## 300 kHz test

The limit was increased to:

```matlab
Fsw_max = 300e3;
```

The `264 VAC, 60 Hz` results were:

```text
Source-side true PF:   0.966443
Filtered-envelope PF:  0.966787
Modeled efficiency:    98.3266 %
Average Vout:          400.0456 V
Output ripple:         4.3759 Vpp
```

The PF improvement was small:

```text
0.965344 → 0.966443
```

The converter was still frequency-limited through a large portion of the high-line waveform.

---

## 500 kHz test

The limit was increased again to:

```matlab
Fsw_max = 500e3;
```

This value is close to the previously observed natural average switching frequency while remaining far below the unclamped `1.25 MHz` maximum.

The `264 VAC, 60 Hz` results were:

```text
Source-side true PF:   0.990644
Filtered-envelope PF:  0.991444
Modeled efficiency:    98.2539 %
Average Vout:          400.0398 V
Output ripple:         4.1318 Vpp
```

The higher limit shortened the forced zero-current idle interval enough to restore a much more sinusoidal input-current envelope.

---

## Test comparison

| Frequency-limit setting | True PF | Filtered PF | Modeled efficiency | Vout ripple |
|---:|---:|---:|---:|---:|
| `250 kHz` | `0.965344` | `0.965579` | `98.2883%` | `4.2866 Vpp` |
| `300 kHz` | `0.966443` | `0.966787` | `98.3266%` | `4.3759 Vpp` |
| `500 kHz` | `0.990644` | `0.991444` | `98.2539%` | `4.1318 Vpp` |

The modeled efficiency remained approximately unchanged.

The simulation does not include all frequency-dependent hardware losses, so the real switching-loss penalty of increasing the limit would be larger than the idealized model indicates.

---

## Considered improvement: clamp-active on-time compensation

A possible alternative was to retain a lower frequency limit and increase the current-pulse area only while the clamp was active.

A first-order compensation form considered was:

$$
T_{\mathrm{on,comp}} = \sqrt{T_{\mathrm{on,base}}T_{\mathrm{cycle,min}}\frac{V_{\mathrm{out}}-V_{\mathrm{in}}}{V_{\mathrm{out}}}}
$$

The compensation would only operate when the natural CrCM period was shorter than the enforced minimum period:

```matlab
clamp_active = Tnatural_base < Tcycle_min;
```

Conceptually:

```matlab
if clamp_active
    Ton_selected = Ton_comp;
else
    Ton_selected = Ton_base;
end
```

The compensated command would still be subject to the normal on-time, OCP, and protection limits.

---

## Additional design required for on-time compensation

The compensation was not implemented because it would require substantial additional design and validation.

### Clamp-active detection

The compensation must operate only while the frequency clamp is actually extending the natural cycle.

Applying it during normal CrCM operation would distort waveform regions that already have the correct current shape.

### Peak-current protection

A larger compensated on-time can increase peak inductor current.

The fixed OCP threshold should remain a safety limit. The compensated on-time would need to be bounded so that predicted and measured peak current stay below that limit.

The OCP threshold should not simply be relaxed to accommodate compensation.

### Model assumptions

The approximation assumes:

```text
Linear inductor-current ramps
Vout approximately constant within one switching cycle
Vin approximately constant within one switching cycle
Each cycle begins at approximately zero inductor current
Parasitic ringing and propagation delays are neglected
```

Its accuracy would decrease when parasitic ringing, nonlinear capacitance, or valley-switching behavior become important.

### Startup and transient interaction

Compensation would need to be disabled or carefully managed during:

```text
Precharge
Controller startup
Load connection
OVP or OCP events
Large voltage-loop transients
```

It would also require renewed testing of:

```text
Voltage-loop stability
Integrator behavior
Dynamic maximum on-time
Peak-current stress
Output overshoot
All line-voltage and line-frequency corners
```

---

## Why on-time compensation was not used

At the `500 kHz` limit, the measured true PF was already:

```text
0.990644
```

This is well above the design requirement:

```text
PF > 0.95
```

Implementing clamp-active on-time compensation would add:

```text
More control logic
More operating modes
Additional peak-current constraints
More transient interactions
More validation work
```

The expected improvement beyond the existing `0.99` PF result was not considered large enough to justify that added complexity.

The compensation remains a possible future option if a lower switching-frequency ceiling becomes mandatory.

---

## Valley-switching consideration

Valley switching was also considered as a future improvement.

Its main purpose would be to reduce:

```text
MOSFET turn-on loss
Coss-related loss
Switching noise
EMI
```

Valley switching would not directly restore the average input-current area removed by a hard minimum-period clamp.

A realistic valley-switching model would also require:

```text
MOSFET output capacitance
Boost-diode junction capacitance
Relevant parasitic inductance
Drain-voltage ringing
Valley detection
Blanking and timeout logic
```

It was therefore not used as the solution to the PF distortion in the present idealized model.

---

## Issue-specific result

Clamp values of `250 kHz` and `300 kHz` created excessive forced idle time near the AC zero crossings and reduced high-line PF to approximately `0.965–0.966`.

Increasing the clamp to `500 kHz` improved the measured high-line PF to:

```text
0.990644
```

while maintaining:

```text
Approximately 400 V output
Approximately 100 W output power
Modeled efficiency above 98%
No need for clamp-active on-time compensation
```
