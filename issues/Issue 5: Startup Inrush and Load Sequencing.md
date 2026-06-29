# Issue 5: Startup Inrush, OVP Triggering, and Load Sequencing

## Problem

After the CrCM controller and EMI filter were operating correctly in steady state, the cold-start simulation still showed:

```text
A very large source-current spike
Rapid passive charging of the output capacitor
Vout approaching the 420 V OVP threshold
```

At `264 VAC, 60 Hz`, the original source-current spike was approximately:

```text
46 A
```

The active controller protections did not prevent this event.

---

## Why OCP and gate control could not stop the inrush

A passive charging path exists even while the MOSFET is off:

```text
AC source
→ EMI filter
→ bridge rectifier
→ boost inductor
→ boost diode
→ output capacitor
```

Therefore:

```text
The output capacitor can charge without MOSFET switching.
```

The MOSFET current limit only controls current flowing through the active switch.

OVP can inhibit the MOSFET, but it cannot disconnect the passive diode-charging path.

This is why changing the ZCD, on-time timer, or OCP alone could not eliminate the original startup spike.

---

## Precharge resistor and bypass switch

An `82 ohm` resistor was added in series with the AC line.

A controlled electrical switch was placed in parallel with the resistor:

```text
                         ┌──── bypass switch ────┐
AC source → Iac sensor ──┤                       ├── EMI filter → bridge
                         └──── 82 ohm resistor ──┘
```

Operation:

```text
Bypass open:
All source current flows through the precharge resistor.

Bypass closed:
The resistor is bypassed for normal operation.
```

Approximate ideal high-line current bound:

$$I_{\text{precharge,max}} \approx \frac{264\sqrt{2}}{82} \approx 4.55\text{ A}$$

Approximate output-capacitor RC time constant:

$$\tau = R_{\text{precharge}}C_{\text{out}} = 82 \times 220\,\mu\mathrm{F} \approx 18\text{ ms}$$

---

## Controller hold-off during precharge

The controller latch is forced off while `ControllerEnable = 0`.

The startup gating is:

```matlab
SET_existing = startup_pulse || zcd_pulse;

SET_final = SET_existing && ControllerEnable;
RESET_final = RESET_existing || ~ControllerEnable;
```

Before controller enable:

```text
SET is blocked.
RESET is forced active.
GateCmd remains low.
```

After controller enable:

```text
The startup pulse initiates the first switching cycle.
The normal ZCD, timer, OCP, and OVP logic then continue operation.
```

The precharge resistor limits the passive charging current. The controller delay prevents active PFC switching from adding power before the precharge interval is complete.

---

## Why bypass timing alone was insufficient

An early bypass attempt closed the switch while the output bus was still far below the rectified-input peak.

Removing the `82 ohm` resistor suddenly applied the remaining voltage difference through a low-impedance path and produced another large current pulse.

The bypass was first moved to:

```matlab
t_bypass = 0.150;
```

For a zero-phase source, `0.150 s` is a line-voltage zero crossing for both `50 Hz` and `60 Hz`.

This reduced the instantaneous switching step, but a substantial current pulse remained because the full `1600 ohm` load was still connected during precharge.

---

## Why the connected load prevented effective precharge

The full load consumed substantial power while the bus was still charging.

For example:

$$P_{\text{load}} = \frac{300^2}{1600} \approx 56\text{ W}$$

The precharge resistor could not raise the bus sufficiently close to the high-line rectified peak:

$$264\sqrt{2} \approx 373\text{ V}$$

After the bypass closed, the line voltage rose during the next quarter-cycle and the remaining bus-voltage difference produced a large passive charging pulse.

---

## Load-disconnect solution

A controlled electrical switch was placed only in the `1600 ohm` load branch.

Correct electrical structure:

```text
Boost diode ───────────── Bus+
                            │
                            ├──── Cout ───────────── Bus−
                            │
                            ├──── Vout sensor ────── Bus−
                            │
                            └──── Load switch ── 1600 ohm ── Bus−
```

The output capacitor and Vout sensor remain permanently connected to the DC bus.

Only the load resistor is disconnected during startup.

This allows:

```text
Cout to precharge with very little load.
The PFC stage to raise the bus toward regulation.
The 100 W load to connect only after the bus is ready.
```

---

## Incorrect Vout-sensor placement

During debugging, the voltage sensor was temporarily placed across the switched load resistor.

With the load switch open:

```text
Measured Vout = 0 V
```

even though the actual bus capacitor was charged.

The controller then interpreted the bus as empty and commanded excessive power.

When the load switch closed, the measured voltage jumped to the actual bus voltage, which was approximately `455 V`, and OVP activated.

The correction was:

```text
Measure Vout directly across the DC bus and Cout.
Keep the load switch only in series with the resistor branch.
```

---

## Hysteretic load connection

The load controller uses:

```matlab
Vload_on  = 370;
Vload_off = 340;
```

Relay behavior:

```text
Vout_filtered >= 370 V:
LoadReady = 1

Vout_filtered <= 340 V:
LoadReady = 0

Between 340 V and 370 V:
Retain the previous state
```

A separate arm time prevents premature load connection:

```matlab
t_load_arm = 0.212;
```

The raw load command is:

```matlab
LoadCmd_raw = LoadReady * LoadArm;
```

---

## Algebraic-loop correction

The direct load-control path formed:

```text
Vout
→ Relay
→ LoadCmd
→ electrical load switch
→ Vout
```

A Unit Delay was inserted after the load command:

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

The additional `100 us` delay is negligible relative to the output-bus dynamics and removes the algebraic loop.

---

## Bypass-timing comparison: 0.150 s versus 0.200 s

After the load-disconnect sequence was working, the bypass timing was tested at both:

```text
0.150 s
0.200 s
```

Both times correspond to a line-voltage zero crossing for a zero-phase `50 Hz` or `60 Hz` source.

The `0.150 s` timing produced:

```text
Source-current peak: approximately 6.55 A
Bypass-window I^2*t: 0.043846 A^2*s
```

The bypass was then delayed by another `50 ms`:

```matlab
t_bypass = 0.200;
```

For a zero-phase source:

```text
50 Hz: 0.200 s = 10 complete cycles
60 Hz: 0.200 s = 12 complete cycles
```

The additional precharge time allowed the output capacitor to move closer to the passive high-line rectified voltage before the resistor was bypassed.

The `0.200 s` timing produced:

```text
Source-current peak: 4.3139 A
Bypass-window I^2*t: 0.018420 A^2*s
```

Comparison:

```text
Source-current peak:
6.5529 A → 4.3139 A

Bypass-window I^2*t:
0.043846 A^2*s → 0.018420 A^2*s
```

This corresponds to approximately:

```text
34% lower source-current peak
58% lower bypass-window I^2*t
```

The tradeoff was only an additional:

```text
50 ms of startup delay
```

The precharge-resistor energy changed only slightly:

```text
11.6916 J → 11.7391 J
```

This is an increase of approximately `0.4%`.

Therefore, the `0.200 s` bypass timing is mainly beneficial:

```text
Much lower bypass and passive-path current stress
Negligible increase in resistor energy
Only 50 ms additional startup time
```

---

## Final startup timing

```matlab
t_bypass      = 0.200;
t_ctrl_enable = 0.210;
t_startup     = 0.211;
t_load_arm    = 0.212;

Vload_on      = 370;
Vload_off     = 340;
```

Final sequence:

```text
0 to 0.200 s:
Precharge resistor active
Bypass open
Controller disabled
Load disconnected
Cout and Vout sensor connected to the bus

At 0.200 s:
Bypass switch closes near a common 50/60 Hz zero crossing

At 0.210 s:
ControllerEnable rises

At 0.211 s:
Startup pulse initiates the first CrCM switching cycle

At 0.212 s:
Load-control logic is armed

When Vout_filtered reaches 370 V:
LoadCmd closes after one Ts_v delay
```

In the verified high-line startup run, the load command first rose at:

```text
0.2155 s
```

---

## Final measured startup result

At `264 VAC, 60 Hz` with the final startup timing:

```text
Source-current peak:          4.3139 A
Peak time:                    0.2044658 s

Precharge-resistor peak:      3.9031 A
Precharge peak power:         1249.2 W
Precharge-resistor energy:    11.7391 J
Precharge current I^2*t:      0.143159 A^2*s

Bypass-switch peak:           4.3133 A
Bypass-window I^2*t:          0.018420 A^2*s

Maximum Vout:                 409.2699 V
OVP margin:                   10.7301 V

MOSFET startup-current peak:  1.6366 A
MOSFET OCP margin:            2.3634 A
```

The passive charging path experienced:

```text
Boost-inductor peak:  4.3120 A
Bridge-path peak:     4.3120 A
Bridge-diode peak:    4.3120 A
Boost-diode peak:     4.3083 A
Cout charging peak:   4.3083 A
```

The passive surge is larger than the MOSFET current because it flows through the bridge, inductor, boost diode, and output capacitor without passing through the active switch.

The protection signals remained inactive:

```text
OVP = 0
OCP = 0
```

---

## Remaining EMI-filter voltage stress

The longer precharge interval reduced the passive current surge, but the startup event still produced bridge-side ringing.

A time-aligned comparison gave:

```text
Source voltage:       -329.81 V
Bridge-side voltage:  -427.76 V
Difference:             97.96 V
```

The measured semiconductor voltage stresses were:

```text
Maximum MOSFET Vds:              410.19 V
Boost-diode reverse voltage:     408.95 V
Bridge-diode reverse voltage:    444.99 V
```

The bridge-diode reverse-voltage requirement remains the largest semiconductor voltage stress in the startup run.

This does not invalidate the startup sequence, but the ringing must be included when selecting component voltage ratings.

---

## Effect on validation

Before the startup network was added, the initial `46 A` passive spike and OVP event dominated the simulation.

This made it difficult to distinguish:

```text
Normal CrCM controller behavior
from
uncontrolled capacitor-charging behavior
```

After the correction:

```text
The passive inrush is limited.
The controller begins only after precharge.
The load connects only after the bus is ready.
Vout remains below the 420 V OVP threshold.
OCP and OVP remain inactive.
The startup transient no longer dominates normal controller validation.
```

The final `0.200 s` timing further reduced the remaining bypass-event current without introducing a meaningful startup penalty.

---

## Final implementation summary

```matlab
Rprecharge = 82;

t_bypass      = 0.200;
t_ctrl_enable = 0.210;
t_startup     = 0.211;
t_load_arm    = 0.212;

Vload_on  = 370;
Vload_off = 340;

SET_existing = startup_pulse || zcd_pulse;
SET_final = SET_existing && ControllerEnable;

RESET_final = RESET_existing || ~ControllerEnable;

LoadReady = relay( ...
    Vout_filtered, ...
    Vload_on, ...
    Vload_off);

LoadCmd_raw = LoadReady * LoadArm;
LoadCmd = unit_delay(LoadCmd_raw, Ts_v, 0);
```

Electrical structure:

```text
AC source
→ Iac sensor
→ 82 ohm precharge resistor with parallel bypass switch
→ differential-mode EMI filter
→ bridge rectifier
→ boost PFC
→ DC bus and Cout

DC bus
├── Vout sensor permanently connected
├── Cout permanently connected
└── controlled load switch → 1600 ohm load
```

---

## Main conclusion

The startup problem was caused by passive capacitor charging, not by the active MOSFET command alone.

The final startup correction combines:

```text
Series precharge resistance
+
zero-cross bypass timing
+
0.200 s precharge interval
+
controller hold-off
+
load disconnection during precharge
+
correct bus-voltage sensing
+
hysteretic load connection
+
one-sample load-command delay
```

Testing showed that increasing the bypass time from `0.150 s` to `0.200 s` reduced the source-current peak from approximately `6.55 A` to `4.31 A`.

The extra `50 ms` of startup time caused only a negligible increase in precharge-resistor energy, so the longer timing is a favorable tradeoff.
