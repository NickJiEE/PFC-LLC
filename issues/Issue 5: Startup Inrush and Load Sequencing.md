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

The MOSFET current limit can only control current that flows through the active switch.

OVP can inhibit the MOSFET, but it cannot disconnect the passive diode-charging path.

This is why changing the ZCD, timer, duty command, or OCP alone could not eliminate the original startup spike.

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

The final latch gating is:

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

At controller enable, the normal ZCD, timer, OCP, and OVP behavior becomes active.

---

## Final ZCD and timer behavior during startup

The final ZCD is:

```matlab
current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;
zcd_pulse = rising_edge(zero_ready);
```

The startup pulse initiates only the first switching cycle. All later cycles are initiated by the ZCD valid-restart-state transition.

The commanded on-time path is:

```text
Ton_ff + Ton_P + Ton_I
→ dynamic saturation
→ Ton_cmd
→ Rate Transition
→ Ton_cmd_fast
→ 100 ns discrete timer
→ Ton_done
→ latch RESET
```

This is the final timer architecture used with the startup sequencer.

---

## Why bypass timing alone was insufficient

An early bypass attempt closed the switch while the output bus was still far below the rectified-line peak.

Removing the `82 ohm` resistor suddenly applied the remaining voltage difference through a low-impedance path and produced another large current pulse.

The bypass time was moved to:

```matlab
t_bypass = 0.150;
```

For a zero-phase source, `0.150 s` is a voltage zero crossing for both `50 Hz` and `60 Hz`.

This reduced the instantaneous switching step, but a large current pulse remained because the full `1600 ohm` load was still connected during precharge.

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

---

## Incorrect Vout-sensor placement

During debugging, the voltage sensor was temporarily placed across the switched load resistor.

With the load switch open:

```text
Measured Vout = 0 V
```

even though the actual bus capacitor was charged.

The controller then interpreted the bus as empty and commanded excessive on-time.

When the load switch closed, the measured voltage jumped to the actual bus value, which was approximately `455 V`, and OVP activated.

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

A separate arm time prevents premature connection:

```matlab
t_load_arm = 0.155;
```

The raw command is:

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

The additional `100 us` delay is negligible relative to the DC-bus dynamics and removes the algebraic loop.

---

## Final startup timing

```matlab
t_bypass      = 0.150;
t_load_arm    = 0.155;
t_ctrl_enable = 0.160;
t_startup     = 0.161;

Vload_on      = 370;
Vload_off     = 340;
```

Final sequence:

```text
0 to 0.150 s:
Precharge resistor active
Bypass open
Controller disabled
Load disconnected
Cout and Vout sensor connected to the bus

0.150 s:
Bypass closes near a common 50/60 Hz zero crossing

0.155 s:
Load logic is armed

0.160 s:
ControllerEnable rises

0.161 s:
Startup pulse initiates the first CrCM cycle

When Vout_filtered reaches 370 V:
LoadCmd closes after one Ts_v delay
```

In the verified high-line startup run, the load command first rose at:

```text
0.1651 s
```

---

## Final measured startup result

At `264 VAC, 60 Hz`:

```text
Source-current peak:          6.5529 A
Peak time:                    0.1543974 s
Precharge-resistor peak:      3.9031 A
Precharge-resistor energy:    11.6916 J
Maximum Vout:                 408.8928 V
OVP margin:                   11.1072 V
MOSFET startup-current peak:  1.6371 A
MOSFET OCP margin:            2.3629 A
```

The passive charging path experienced approximately:

```text
Boost-inductor peak:  6.5505 A
Bridge-path peak:     6.5505 A
Boost-diode peak:     6.5469 A
Cout charging peak:   6.5468 A
```

The passive surge is larger than the MOSFET current because it flows through the bridge, inductor, boost diode, and output capacitor without passing through the active switch.

---

## Remaining EMI-filter voltage stress

The startup event also produced bridge-side ringing.

A time-aligned comparison gave:

```text
Source voltage:       345.22 V
Bridge-side voltage:  440.25 V
Difference:            95.04 V
```

The representative bridge diode reached:

```text
Maximum reverse voltage = 447.26 V
```

This does not invalidate the startup sequence, but it must be included in component voltage-rating validation.

---

## Effect on validation

Before the startup network was added, the initial `46 A` passive spike and OVP event dominated the simulation.

This made it impossible to distinguish:

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
The steady-state controller can be validated independently of startup inrush.
```

---

## Final implementation summary

```matlab
Rprecharge = 82;

t_bypass      = 0.150;
t_load_arm    = 0.155;
t_ctrl_enable = 0.160;
t_startup     = 0.161;

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

The final solution combines:

```text
Series precharge resistance
+
zero-cross bypass timing
+
controller hold-off
+
final ZCD and discrete timer logic
+
load disconnection during precharge
+
correct bus-voltage sensing
+
hysteretic load connection
+
one-sample load-command delay
```

This reduced the source-current peak from approximately `46 A` to `6.55 A`, kept the output below the `420 V` OVP threshold, and preserved normal steady-state operation.
