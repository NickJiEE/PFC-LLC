# Issue 8: Low-Line Startup OCP and Temporary On-Time Limiting

## Issue

During the `90 VAC, 50 Hz` startup test, the converter repeatedly reached the cycle-by-cycle overcurrent-protection threshold.

The configured current limit was:

```matlab
I_limit = 4.0;
```

The measured startup currents were approximately:

```text
MOSFET branch-current peak: 4.05 A
Boost-inductor peak:        4.11 A
```

The OCP signal asserted during startup, even though:

```text
The source-side current remained near 2.2 A.
The output voltage stayed below the 420 V OVP threshold.
The converter completed startup successfully.
```

---

## Why the source current did not show 4 A

The `4 A` OCP threshold applies to the instantaneous MOSFET branch current, not the source-side AC current.

The relevant currents are different:

```text
Iac:
Source-side current before the EMI filter

iL:
Boost-inductor current

Isource:
MOSFET branch current used by the cycle-by-cycle OCP
```

During switching, the EMI filter and X capacitor supply part of the high-frequency current locally.

Therefore:

```text
Iac can remain near 2.2 A
while
iL and Isource briefly reach approximately 4 A
```

The full-window Scope also compresses thousands of microsecond-scale switching pulses, so the instantaneous current peaks are not obvious unless the waveform is viewed over a few switching cycles.

---

## Why OCP occurred during low-line startup

At `90 VAC`, passive precharge can only raise the DC bus toward the rectified input peak:

$$
90\sqrt{2} \approx 127.3\text{ V}
$$

When the controller is enabled, the bus must still rise from approximately `120–127 V` toward the `400 V` reference.

The large voltage error causes the voltage loop to demand a long on-time:

```text
Large output-voltage error
→ Ton command reaches its startup limit
→ inductor current rises toward 4 A
→ OCP resets the gate pulse
```

The first OCP event occurred before the load connected.

This showed that the event was caused by charging the output bus at low line, not by the load-connection transient.

The `500 kHz` maximum-frequency clamp was not involved because the measured startup frequency remained far below that limit.

---

## Instantaneous current limit versus average current

The OCP threshold limits the instantaneous MOSFET current.

It does not limit the line-cycle average or switching-cycle average current to `4 A`.

A typical startup switching cycle behaves as:

```text
MOSFET turns on
→ iL and Isource ramp upward
→ current reaches the 4 A threshold
→ OCP resets the gate
→ MOSFET current falls to zero
→ inductor current transfers to the boost diode and ramps downward
```

The inductor-current Scope can appear filled between zero and the peak when many switching cycles are compressed into one plot.

The upper envelope near `4.1 A` represents the instantaneous peak, not a continuous average current of `4 A`.

---

## Why the measured current exceeded 4 A slightly

The controller uses a `100 ns` fast sample time.

The current can cross the threshold between samples. OCP is detected on the next control update, and the latch then resets.

This creates a small discrete-time overshoot:

```text
OCP threshold:           4.000 A
Measured MOSFET peak:    approximately 4.052 A
Threshold overshoot:     approximately 0.052 A
Relative overshoot:      approximately 1.3%
```

The boost-inductor peak is slightly larger than the MOSFET branch-current peak because the inductor current continues through the boost diode after the MOSFET turns off.

---

## Temporary startup on-time limit

A temporary lower on-time ceiling was added for startup.

The normal and startup limits were:

```matlab
Ton_max         = 8.0e-6;
Ton_startup_max = 7.5e-6;
```

The filtered output voltage controls when the normal limit is restored.

A Relay block was configured with:

```text
Switch-on point:  390 V
Switch-off point: 370 V

Output when on:   1
Output when off:  0
```

The Relay output is:

```text
NormalTonLimitEnable
```

The selected absolute limit is:

```matlab
if NormalTonLimitEnable
    Ton_abs_max_selected = Ton_max;
else
    Ton_abs_max_selected = Ton_startup_max;
end
```

The selected value is combined with the existing feedforward-based dynamic limit:

```matlab
Ton_dynamic_max = min( ...
    Ton_abs_max_selected, ...
    K_Ton_max * Ton_ff);
```

The resulting limit is applied to the dynamic saturation block that generates `Ton_cmd`.

---

## Verified startup-limit behavior

The startup-limit signals showed:

```text
Before Vout_filtered reached 390 V:
NormalTonLimitEnable = 0
Ton_abs_max_selected = 7.5 us
Ton_dynamic_max      = 7.5 us
Ton_cmd              = 7.5 us

After Vout_filtered reached 390 V:
NormalTonLimitEnable = 1
Ton_abs_max_selected = 8.0 us
```

A Ton maximum larger than `7.5 us` over the full `0.211–0.500 s` analysis interval can occur after the controller returns to the normal `8 us` limit.

This does not mean the startup cap was bypassed.

---

## Tested startup limits

Three startup-limit settings were evaluated.

| Startup Ton limit | MOSFET peak | Inductor peak | Iac peak | Bypass-window I²t | Load connection |
|---:|---:|---:|---:|---:|---:|
| `8.0 us` | `4.0519 A` | `4.1064 A` | `2.2856 A` | `0.08142 A²s` | `0.3263 s` |
| `7.5 us` | `4.0540 A` | `4.1109 A` | `2.1657 A` | `0.07425 A²s` | `0.3314 s` |
| `7.0 us` | `4.0522 A` | `4.1069 A` | `2.1568 A` | `0.06743 A²s` | `0.3388 s` |

The internal current peak remained near the same value because OCP, rather than the commanded on-time, was setting the final peak.

Once the current reached the threshold, OCP terminated the pulse.

Lowering the startup on-time mainly changed:

```text
Source-side startup current
Bypass-window I²t
Startup duration
```

It did not materially reduce the OCP-limited MOSFET or inductor peak.

---

## Selected implementation

The selected temporary startup limit was:

```matlab
Ton_startup_max = 7.5e-6;
Ton_max         = 8.0e-6;
```

The normal limit is restored when:

```text
Vout_filtered reaches 390 V
```

The Relay hysteresis returns to startup limiting only if:

```text
Vout_filtered falls below 370 V
```

The `7.5 us` value was retained because it reduced source-side startup stress with only a small startup delay.

Reducing the limit further to `7.0 us` produced little additional source-current improvement and delayed load connection further.

---

## Why OCP was retained during startup

The brief low-line startup OCP operation was accepted because:

```text
The current remained bounded near the intended 4 A threshold.
The approximately 1.3% overshoot was consistent with sampled detection delay.
The converter completed startup.
The load connected successfully.
Vout remained below 420 V.
OVP did not activate.
Steady-state operation did not use OCP.
```

Eliminating all startup OCP events would require a more involved control method, such as:

```text
A ramped output-voltage reference
A dedicated startup current controller
A current-dependent on-time limit
```

Those changes would add control complexity and require renewed startup, transient, and four-corner validation.

The small and controlled startup OCP event did not justify that additional implementation effort.

---

## Issue-specific result

For the selected `7.5 us` startup limit at `90 VAC, 50 Hz`:

```text
OCP operation:              brief cycle-by-cycle limiting
MOSFET current peak:        approximately 4.05 A
Boost-inductor peak:        approximately 4.11 A
Source-current peak:        approximately 2.17 A
Load connection:            approximately 0.331 s
Maximum Vout:               approximately 409.2 V
OVP activation:             none
```

The inductor saturation-current rating and semiconductor pulsed-current ratings must be selected above the measured startup peaks.
