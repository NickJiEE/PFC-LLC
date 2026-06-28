# Issue 5: Startup Inrush, OVP Triggering, and Load-Sequencing Problems

## System condition

After the CrCM controller, voltage-loop feedforward, dynamic maximum on-time, and differential-mode EMI filter were working correctly, the converter passed the steady-state PF and efficiency tests.

The input filter was:

```matlab
LDM_Line    = 1e-3;
LDM_Neutral = 1e-3;
Cx          = 100e-9;
```

The output stage used:

```matlab
Cout = 220e-6;
Rload = 1600;
Vout_ref = 400;
OVP_threshold = 420;
```

Although steady-state operation was stable, the cold-start waveforms showed a large source-current spike and output-voltage overshoot.

---

## Original startup symptom

At `264 VAC, 60 Hz`, the source-side current showed a very narrow initial spike of approximately:

```text
Iac,peak ≈ 46 A
```

The output voltage also rose to approximately:

```text
Vout ≈ 420 V
```

which activated the overvoltage protection.

The observed sequence was approximately:

```text
AC source connected
→ large passive charging current
→ EMI-filter and output capacitors charge rapidly
→ Vout reaches the OVP threshold
→ PFC switching is inhibited
→ Vout later falls and the converter restarts
```

This startup event occurred even though the normal boost-inductor current limit and dynamic maximum on-time were active.

---

## Why the normal controller protections did not prevent it

The startup spike was not primarily caused by an excessive MOSFET on-time.

Even with the MOSFET disabled, a passive charging path exists:

```text
AC source
→ EMI-filter inductors
→ bridge rectifier
→ boost inductor
→ boost diode
→ output capacitor
```

Therefore:

```text
The output capacitor can charge without MOSFET switching.
```

The boost current limit only controls current created by the active switching sequence. It cannot interrupt current flowing through the passive bridge and boost-diode path.

Similarly:

```text
OVP can stop the MOSFET
but it cannot disconnect the passive diode-charging path.
```

The EMI filter also stores energy. With a nearly ideal source and low modeled resistance, the filter inductors and capacitors could ring during startup and increase both current and voltage stress.

---

## Initial fix: precharge resistor and bypass switch

An `82 Ω` resistor was added in series with the AC line to limit passive startup current.

A controlled electrical switch was connected in parallel with the resistor:

```text
                         ┌──── bypass switch ────┐
AC source → Iac sensor ──┤                       ├── EMI filter → bridge
                         └──── 82 Ω resistor ────┘
```

Switch behavior:

```text
Switch open:
Current must flow through the 82 Ω precharge resistor.

Switch closed:
The resistor is bypassed for normal operation.
```

The approximate high-line current bound was:

\[
I_{\text{precharge,max}}
\approx
\frac{264\sqrt{2}}{82}
\approx
4.55\text{ A}
\]

The output-capacitor RC time constant was approximately:

\[
\tau
=
R_{\text{precharge}}C_{\text{out}}
=
82(220\,\mu\text{F})
\approx
18\text{ ms}
\]

---

## Controller startup delay

The active PFC controller was held disabled during precharge.

The latch logic was modified as:

```matlab
SET_final = SET_existing && ControllerEnable;

RESET_final = RESET_existing || ~ControllerEnable;
```

Before controller enable:

```text
SET is forced low.
RESET is forced high.
The MOSFET remains off.
```

After controller enable:

```text
The normal startup pulse, ZCD, on-time, OCP, and OVP logic resume.
```

The startup pulse was moved to just after the controller-enable transition.

This delayed the beginning of active PFC switching, but it did not by itself eliminate passive inrush. The precharge resistor was the component that limited the passive charging current.

---

## Problem with the first bypass timing

The first timing attempt used approximately:

```matlab
t_bypass = 0.12;
t_controller_enable = 0.13;
```

The initial current was reduced, but a second current spike appeared when the bypass switch closed:

```text
Iac,peak ≈ 17 A
```

At the bypass instant:

```text
The output bus was still below the rectified input peak.
The 82 Ω resistor was suddenly removed.
The remaining voltage difference was applied through the low-impedance path.
The output capacitor charged rapidly.
```

The output voltage also made a sharp upward step.

---

## Zero-cross bypass attempt

The bypass time was moved to:

```matlab
t_bypass = 0.15;
t_controller_enable = 0.16;
t_startup_pulse = 0.161;
```

For a zero-phase source, `0.15 s` is a line-voltage zero crossing for both nominal frequencies:

```text
50 Hz: 0.15 s = 7.5 cycles
60 Hz: 0.15 s = 9 cycles
```

Closing the bypass near zero voltage reduced the spike from approximately:

```text
17 A → 14 A
```

but did not eliminate it.

---

## Why zero-cross bypass was not sufficient

The `1600 Ω` load was still connected during precharge.

At approximately `300 V`, the load consumed:

\[
P_{\text{load}}
=
\frac{300^2}{1600}
\approx
56\text{ W}
\]

Because the full load was continuously drawing power, the precharge resistor could not raise the bus close enough to the high-line rectified peak:

\[
264\sqrt{2}
\approx
373\text{ V}
\]

Even if the bypass closed exactly at a zero crossing:

```text
The line voltage rose during the next quarter-cycle.
The output capacitor was still well below the line peak.
The bridge and boost diode began conducting strongly.
A large charging-current pulse still occurred.
```

Thus:

```text
Zero-cross switching reduced the instantaneous step
but did not solve the remaining bus-voltage mismatch.
```

---

## Load-disconnect approach

A controlled electrical switch was added in series with only the `1600 Ω` load branch.

Correct electrical connection:

```text
Boost diode ───────────── Bus+
                            │
                            ├──── Cout ───────────── Bus−
                            │
                            ├──── Vout sensor ────── Bus−
                            │
                            └──── Load switch ── 1600 Ω ── Bus−
```

The output capacitor and voltage sensor remain connected to the PFC bus at all times.

Only the load resistor is disconnected during startup.

This allows:

```text
The bus capacitor to precharge with little load.
The PFC to raise the bus toward regulation.
The full 100 W load to connect only after the bus is ready.
```

---

## Incorrect Vout-sensor placement discovered during debugging

During one test, the Vout sensor was accidentally connected across the load resistor on the switched side.

When the load switch was open:

```text
Measured Vout = 0 V
```

even though the output capacitor and actual DC bus were charged.

The controller therefore interpreted the bus as empty:

```text
Measured Vout ≈ 0 V
→ Verror ≈ 400 V
→ Ton command saturates
→ unloaded bus is overcharged
```

When the load switch later closed, the measured voltage suddenly jumped to approximately:

```text
Vout ≈ 455 V
```

This activated OVP. After the bus discharged below the OVP threshold, the controller restarted, which appeared as an unexplained gate-command re-enable.

The correction was:

```text
Measure Vout directly across Cout and the DC bus.
Place the load switch only in series with the resistor branch.
```

---

## Load-enable hysteresis

A voltage-based load controller was added using a Relay block.

Settings:

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

340 V < Vout_filtered < 370 V:
Retain the previous state
```

A minimum arm time was also retained:

```matlab
t_load_arm = 0.155;
```

The final load command was:

```matlab
LoadCmd_raw = LoadReady * LoadArm;
```

This ensured that the load could not connect before the precharge sequence reached the intended stage.

---

## Algebraic-loop issue

The direct load-control path created an algebraic loop:

```text
Vout
→ filtered Vout
→ Relay
→ LoadCmd
→ electrical load switch
→ Vout
```

Because the load command immediately changed the same bus voltage used to generate the command, Simulink reported an algebraic loop containing a discontinuity.

The loop was broken by inserting a Unit Delay after the load command:

```text
Vout_filtered
      ↓
Relay
      ↓
LoadReady × LoadArm
      ↓
Unit Delay
IC = 0
Ts = Ts_v
      ↓
Simulink-PS Converter
      ↓
Load switch
```

Using:

```matlab
Ts_v = 100e-6;
```

the load switch responds one voltage-loop sample later:

```text
Delay = 100 µs
```

This delay is negligible relative to the output-bus dynamics and prevents the algebraic loop.

---

## Final startup sequence

The final timing and logic were:

```matlab
t_bypass      = 0.150;
t_load_arm    = 0.155;
t_ctrl_enable = 0.160;
t_startup     = 0.161;

Vload_on      = 370;
Vload_off     = 340;
```

Sequence:

```text
0 to 0.150 s:
Precharge resistor active
Bypass switch open
PFC controller disabled
Load disconnected
Cout and Vout sensor remain connected to the bus

At 0.150 s:
Bypass switch closes near a common 50/60 Hz zero crossing

At 0.155 s:
Load control becomes armed

At 0.160 s:
PFC controller is enabled

At 0.161 s:
Startup pulse begins the first CrCM switching cycle

When Vout_filtered reaches 370 V:
Load switch closes after one Ts_v delay

If Vout_filtered later falls below 340 V:
Load switch opens

When Vout_filtered rises above 370 V again:
Load switch reconnects
```

At high line, the load typically connected soon after active PFC startup.

At low line, the converter could raise the bus with the load disconnected, then connect the full load after the bus reached the required threshold.

---

## Final measured startup result

After correcting the bus-voltage sensing and adding hysteretic load control, the bypass-event source-current peak was measured as:

```text
Bypass-event peak current = 6.5529 A
Peak time                 = 0.1543974 s
```

Compared with the original startup:

```text
Original source-current spike ≈ 46 A
Final measured spike          ≈ 6.55 A
```

This represents a major reduction in startup current.

The output-voltage behavior also improved:

```text
Vout no longer reached the 420 V OVP threshold.
The large startup jump was greatly reduced.
The bus rose toward regulation without an OVP shutdown cycle.
```

The load command switched on once the filtered bus voltage reached approximately `370 V`, and the load current settled near:

\[
I_{\text{out}}
=
\frac{400}{1600}
=
0.25\text{ A}
\]

No load-switch chatter was observed because of the `370 V / 340 V` hysteresis band.

---

## Gate-command gap investigation

Small apparent gaps remained visible in the gate waveform near startup and near the rectified-line peaks.

The protection signals remained inactive:

```text
OVP = 0
OCP = 0
ControllerEnable = 1
```

The boost-inductor current, ZCD pulse, Ton-done pulse, and gate command were inspected at a much smaller time scale.

The gaps were consistent with normal CrCM variable-frequency behavior:

```text
Near the rectified-line peak:
Vin is close to Vout
→ inductor demagnetization is slower
→ off-time becomes longer
→ switching frequency decreases
```

The detailed cycle sequence remained correct:

```text
Gate rises
→ boost-inductor current rises
→ Ton_done resets the gate
→ inductor current falls
→ ZCD pulse occurs near zero current
→ next gate pulse begins
```

Therefore, the remaining sparse gate regions were not caused by OVP, OCP, or a missing startup trigger.

---

## Steady-state impact

The startup additions did not meaningfully change steady-state performance.

For `90 VAC, 60 Hz`, before and after the final startup changes:

```text
PF:
0.99964 → 0.99964

Efficiency:
96.186% → 96.173%

Average Vout:
399.7728 V → 399.8053 V

Output ripple:
3.8872 Vpp → 3.8674 Vpp
```

The differences were negligible.

Therefore:

```text
The precharge, bypass, controller-delay, and load-hysteresis circuitry
improved startup behavior without degrading normal operation.
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

SET_final =
    SET_existing && ControllerEnable;

RESET_final =
    RESET_existing || ~ControllerEnable;

LoadReady =
    relay(Vout_filtered, Vload_on, Vload_off);

LoadCmd_raw =
    LoadReady * LoadArm;

LoadCmd =
    unit_delay(LoadCmd_raw, Ts_v, 0);
```

Electrical structure:

```text
AC source
→ Iac sensor
→ 82 Ω precharge resistor with parallel bypass switch
→ differential-mode EMI filter
→ bridge rectifier
→ boost PFC
→ DC bus and Cout

DC bus
├── Vout sensor permanently connected
├── Cout permanently connected
└── controlled load switch → 1600 Ω load
```

---

## Main conclusion

The original startup problem was caused by uncontrolled passive charging of the EMI-filter and output capacitors, not only by the active PFC gate command.

The successful correction required several coordinated changes:

```text
Add a series precharge resistor
+
bypass it after a controlled delay near a line zero crossing
+
hold the PFC controller disabled during precharge
+
disconnect the full load during bus startup
+
measure Vout directly across the DC bus and Cout
+
connect the load using voltage hysteresis
+
insert one Unit Delay to remove the load-control algebraic loop
```

The final approach reduced the measured source-current spike from approximately `46 A` to `6.55 A`, prevented the output from reaching the `420 V` OVP threshold, and preserved the converter's steady-state PF, efficiency, and output regulation.
