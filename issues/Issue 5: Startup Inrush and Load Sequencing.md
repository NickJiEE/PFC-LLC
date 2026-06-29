# Issue 5: Startup Inrush, OVP Triggering, and Load Sequencing

## Issue

During cold startup at `264 VAC, 60 Hz`, the converter produced a very large source-current spike and the output voltage approached the `420 V` overvoltage-protection threshold.

The original startup behavior was approximately:

```text
AC source connected
→ EMI filter and output capacitor charge through the passive power path
→ source current rises to approximately 46 A
→ Vout approaches the OVP threshold
→ active switching may be inhibited
```

This occurred even when the MOSFET was disabled.

---

## Cause

A passive charging path exists independently of the MOSFET gate command:

```text
AC source
→ EMI filter
→ bridge rectifier
→ boost inductor
→ boost diode
→ output capacitor
```

The output capacitor can therefore charge while the MOSFET is off.

This means:

```text
OCP cannot interrupt the passive charging current.
OVP can stop MOSFET switching but cannot open the diode charging path.
Changing the ZCD or on-time timer does not directly limit the initial inrush.
```

The connected `1600 ohm` load also reduced the effectiveness of precharge because it drew power while the bus was still charging.

---

## Initial precharge approach

An `82 ohm` resistor was added in series with the AC line.

A controlled switch was placed in parallel with the resistor:

```text
                         ┌──── bypass switch ────┐
AC source → Iac sensor ──┤                       ├── EMI filter → bridge
                         └──── 82 ohm resistor ──┘
```

Operation:

```text
Bypass open:
Current flows through the precharge resistor.

Bypass closed:
The resistor is bypassed for normal operation.
```

The controller was held disabled during precharge:

```matlab
SET_final = SET_existing && ControllerEnable;
RESET_final = RESET_existing || ~ControllerEnable;
```

Before `ControllerEnable` rises:

```text
SET is blocked.
RESET is forced active.
The MOSFET remains off.
```

The startup pulse is applied only after the controller is enabled.

---

## Why precharge alone was insufficient

With the full load connected during precharge, the bus could not rise close enough to the high-line rectified peak.

At approximately `300 V`, the load consumes:

$$P_{\text{load}} = \frac{300^2}{1600} \approx 56\text{ W}$$

The high-line rectified peak is approximately:

$$264\sqrt{2} \approx 373\text{ V}$$

When the bypass switch closed, the remaining voltage difference caused another charging-current pulse through the low-impedance passive path.

---

## Load-sequencing solution

A controlled switch was added only in series with the load resistor:

```text
Boost diode ───────────── Bus+
                            │
                            ├──── Cout ───────────── Bus−
                            │
                            ├──── Vout sensor ────── Bus−
                            │
                            └──── Load switch ── 1600 ohm ── Bus−
```

The output capacitor and voltage sensor remain permanently connected to the DC bus.

Only the load is disconnected during startup.

This allows the capacitor to precharge with minimal load before the full `100 W` load is connected.

---

## Vout-sensor placement correction

During debugging, the voltage sensor was temporarily placed across the switched load branch.

When the load switch was open:

```text
Measured Vout = 0 V
```

even though the bus capacitor was charged.

The controller then interpreted the bus as empty and commanded excessive power.

The correction was:

```text
Measure Vout directly across the DC bus and Cout.
Place the controlled switch only in the load branch.
```

---

## Hysteretic load control

The load command uses voltage hysteresis:

```matlab
Vload_on  = 370;
Vload_off = 340;
```

Behavior:

```text
Vout_filtered >= 370 V:
LoadReady = 1

Vout_filtered <= 340 V:
LoadReady = 0

340 V < Vout_filtered < 370 V:
Retain the previous state
```

A separate arm signal prevents the load from connecting before the intended startup stage:

```matlab
LoadCmd_raw = LoadReady * LoadArm;
```

---

## Algebraic-loop correction

The direct load-control path created an algebraic loop:

```text
Vout
→ Relay
→ LoadCmd
→ electrical load switch
→ Vout
```

A Unit Delay was inserted after the raw load command:

```text
LoadCmd_raw
→ Unit Delay
   IC = 0
   Ts = Ts_v
→ Simulink-PS Converter
→ load switch
```

with:

```matlab
Ts_v = 100e-6;
```

The one-sample delay breaks the algebraic loop without materially affecting the output-bus dynamics.

---

## Bypass-timing comparison

The bypass timing was tested at:

```text
0.150 s
0.200 s
```

Both correspond to line-voltage zero crossings for a zero-phase `50 Hz` or `60 Hz` source.

### Bypass at 0.150 s

The verified source-current peak was approximately:

```text
6.5529 A
```

The bypass-window current stress was:

```text
0.043846 A^2*s
```

### Bypass at 0.200 s

The additional `50 ms` allowed the output capacitor to precharge closer to the rectified-input peak before the resistor was bypassed.

The verified source-current peak became:

```text
4.3139 A
```

The bypass-window current stress became:

```text
0.018420 A^2*s
```

Comparison:

```text
Source-current peak:
6.5529 A → 4.3139 A

Bypass-window I^2*t:
0.043846 A^2*s → 0.018420 A^2*s
```

This is approximately:

```text
34% lower peak current
58% lower bypass-window I^2*t
```

The precharge-resistor energy changed only slightly:

```text
11.6916 J → 11.7391 J
```

The main tradeoff was an additional:

```text
50 ms of startup time
```

---

## Final startup sequence

```matlab
t_bypass      = 0.200;
t_ctrl_enable = 0.210;
t_startup     = 0.211;
t_load_arm    = 0.212;

Vload_on      = 370;
Vload_off     = 340;
```

Sequence:

```text
0 to 0.200 s:
Precharge resistor active
Bypass open
Controller disabled
Load disconnected

At 0.200 s:
Bypass switch closes

At 0.210 s:
Controller is enabled

At 0.211 s:
Startup pulse initiates the first switching cycle

At 0.212 s:
Load-control logic is armed

When Vout_filtered reaches 370 V:
The load switch closes after one Ts_v delay
```

The verified load-command transition occurred at approximately:

```text
0.2155 s
```

---

## Issue-specific result

With the final startup sequence at `264 VAC, 60 Hz`:

```text
Source-current peak:       4.3139 A
Bypass-switch peak:        4.3133 A
Precharge-resistor energy: 11.7391 J
Maximum Vout:              409.2699 V
OVP margin:                10.7301 V
OVP activation:            none
OCP activation:            none
Load connection time:      approximately 0.2155 s
```

The revised `0.200 s` bypass timing reduced the remaining startup-current spike substantially while adding only `50 ms` to the startup sequence and causing a negligible change in precharge-resistor energy.
