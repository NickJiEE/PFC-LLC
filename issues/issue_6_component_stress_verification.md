# Issue 6: Component-Stress Verification and Switched-Current Measurement Correction

## Verification objective

After the CrCM controller, startup sequencing, precharge network, load connection, EMI filter, and four-corner steady-state performance were verified, the next task was to determine the electrical stress on each major component.

The converter specification was:

```text
Input voltage:       90–264 VAC
Line frequency:      50/60 Hz
Output voltage:      400 VDC
Output power:        100 W
Target PF:           > 0.95
Target efficiency:   > 92%
```

The stress verification was divided into three operating conditions:

```text
Startup stress:
264 VAC, 60 Hz, cold start, 0–0.25 s

Low-line current stress:
90 VAC, 50 Hz, steady state, 4.8–5.0 s

High-line voltage/timing stress:
264 VAC, 60 Hz, steady state, 4.8–5.0 s
```

The startup and low-line cases are completed in this issue. The high-line steady-state stress case remains as the final component-stress run.

---

## Signals used for component-stress measurement

The following signals were logged through Simulink signal logging and accessed through:

```matlab
out.logsout
```

### Input, precharge, and EMI filter

```text
Vac
Iac
Iprecharge
Ibypass
Vac_bridge
Ibridge
iCx
```

### Bridge rectifier

A single bridge diode was measured as the representative diode:

```text
iBridgeD1
VBridgeD1AK
```

All four bridge diodes use identical models and operate symmetrically in steady state. Therefore, one diode is sufficient for steady-state peak, RMS, average, forward-drop, and reverse-voltage measurement.

The total bridge-path current `Ibridge` was retained because it captures the worst startup current regardless of which diagonal bridge pair conducts first.

### Boost power stage

```text
iL
Isource
Vds
iBoostDiode
VBoostDiodeAK
```

`Isource` is the current through the MOSFET source terminal and is used as the MOSFET current measurement.

### Output stage

```text
Vout
iCout
Iout
```

### Protection, sequencing, and timing

```text
OVP
OCP
BypassCmd
LoadCmd
ControllerEnable
Ton_cmd_fast
GateCmd
```

---

## Measurement windows

### Startup

The full startup-stress window was:

```matlab
startupWindow = [0.0 0.25];
```

The bypass-closing window was:

```matlab
bypassWindow = [0.145 0.170];
```

The active gate-analysis window excluded the disabled-controller period:

```matlab
gateStartupWindow = [0.161 0.25];
```

A second timing window excluded the bypass, controller-enable, startup-pulse, and load-connection events:

```matlab
postSequenceGateWindow = [0.170 0.25];
```

### Steady state

The final `0.2 s` of the five-second simulation was used:

```matlab
steadyWindow = [4.8 5.0];
```

At `50 Hz`, this contains exactly ten complete line cycles.

---

# Part 1: Startup component stress

## Startup operating condition

```text
Input:           264 VAC, 60 Hz
Initial state:   discharged output capacitor
Simulation time: 0.25 s
```

The startup sequencing was:

```text
Bypass closes:       0.1500 s
Controller enabled:  0.1600 s
Startup pulse:       0.1610 s
Load connects:       0.1651 s
```

The load-arm command occurs earlier, but the load switch waits until the output voltage reaches the hysteretic turn-on threshold.

---

## Startup protection result

The startup completed without activating either protection:

```text
OVP: no activation
OCP: no activation
```

The maximum output voltage was:

```text
Vout,max = 408.893 V
```

With a `420 V` OVP threshold:

```text
OVP margin = 11.107 V
```

The startup sequencing therefore prevented the previous 420 V overvoltage event.

---

## Precharge-resistor stress

The `82 Ω` precharge resistor experienced:

```text
Peak current:                 3.903 A
Peak instantaneous power:     1249 W
Energy before bypass:         11.692 J
Total measured startup energy:11.692 J
Current I²t:                  0.14258 A²·s
```

The total energy was equal to the energy before bypass, showing that the resistor carried negligible current after the bypass switch closed.

The peak instantaneous power is not a continuous resistor-power requirement. The important resistor-selection parameter is the pulse-energy capability.

A practical resistor should have pulse-energy capability comfortably above:

```text
11.7 J
```

The exact allowable pulse must be checked against the manufacturer’s pulse-overload curve.

---

## Bypass-switch stress

The bypass switch experienced:

```text
Peak current:        6.552 A
Peak-current time:   0.1543974 s
Bypass-window I²t:   0.043846 A²·s
```

The bypass closing event was the largest source-current transient remaining after the startup network was added.

---

## Passive boost-path surge

At the bypass event, approximately the same current flowed through the passive charging path:

```text
Bridge diode D1 peak:      6.551 A
Boost-inductor peak:       6.551 A
Boost-diode peak:          6.547 A
Output-capacitor peak:     6.547 A
```

The current path was:

```text
AC source
→ EMI filter
→ bridge rectifier
→ boost inductor
→ boost diode
→ output capacitor
```

This surge does not flow through the MOSFET and therefore cannot be interrupted by the MOSFET current limit.

---

## Inductor saturation-current requirement

The boost-inductor startup peak was:

```text
iL,startup,peak = 6.551 A
```

This value must be compared with the inductor saturation-current rating.

The normal steady-state peak current is lower, but the inductor must survive the startup bypass transient without saturating.

---

## MOSFET startup current and OCP margin

The MOSFET startup current was:

```text
Isource,peak = 1.637 A
```

The controller current limit was:

```text
Ilimit = 4.0 A
```

Therefore:

```text
OCP margin = 4.000 − 1.637 = 2.363 A
```

The earlier comparison between the `6.55 A` passive inductor surge and the `4 A` current limit was invalid because the passive surge bypasses the MOSFET.

---

## Startup voltage stress

The measured voltage stresses were:

```text
Maximum MOSFET Vds:              409.77 V
Boost-diode reverse voltage:     408.63 V
Bridge-diode D1 reverse voltage: 447.26 V
Maximum output voltage:          408.89 V
```

The bridge diode experienced the highest reverse voltage because the EMI-filter network produced bridge-side ringing.

---

## Bridge-side EMI-filter overshoot

A time-aligned comparison between the source voltage and bridge-side voltage gave:

```text
Source voltage:       345.22 V
Bridge-side voltage:  440.25 V
Difference:            95.04 V
Event time:             0.1885364 s
```

The bridge-side voltage overshoot is associated with the differential-mode input filter, switching transitions, and the low modeled parasitic resistance.

The relevant filter components were:

```text
LDM_Line    = 1 mH
LDM_Neutral = 1 mH
Cx          = 100 nF
```

The measured reverse voltage should be used when selecting the bridge voltage rating.

---

## Startup gate timing

The active startup timing results were:

```text
Ton command:
0.594–1.031 µs

Actual gate pulse:
0.700–1.200 µs
```

Post-sequence switching results were:

```text
Minimum frequency:  21.9 kHz
Average frequency: 421.4 kHz
Maximum frequency:   1.25 MHz
```

The high switching frequency occurs near the rectified-line zero crossing, where both the inductor peak current and demagnetization time become small.

The `100 ns` control sample time also limits the minimum detectable switching interval.

A practical hardware controller may require:

```text
Maximum-frequency clamp
Minimum off-time
Pulse skipping near the line zero crossing
```

---

# Part 2: Low-line steady-state current stress

## Low-line operating condition

```text
Input:              90 VAC, 50 Hz
Simulation time:    5 s
Measurement window: 4.8–5.0 s
```

This is the worst operating corner for continuous current stress because the input current is highest at minimum line voltage.

---

## Low-line protection status

During the measurement window:

```text
OVP = 0
OCP = 0
BypassCmd = 1
LoadCmd = 1
ControllerEnable = 1
```

The converter was in normal settled operation.

---

## Input and EMI-filter current

The source current was:

```text
Iac peak = 1.685 A
Iac RMS  = 1.154 A
```

The bypass-switch current was nearly identical:

```text
Ibypass peak = 1.685 A
Ibypass RMS  = 1.154 A
```

The precharge-resistor current was negligible:

```text
Iprecharge RMS ≈ 0.00014 A
```

Therefore, the bypass switch correctly carries essentially all normal operating current.

The combined modeled EMI-inductor copper loss was:

```text
Pfilter,copper = 0.133 W
```

using the combined winding resistance:

```matlab
RfilterTotal = 0.10;
```

---

## Boost-inductor current

The boost-inductor current was:

```text
Peak current: 3.365 A
RMS current:  1.348 A
Average:      1.029 A
```

The inductor thermal-current requirement is based on:

```text
1.348 A RMS
```

The inductor saturation-current requirement remains based on the larger startup result:

```text
6.551 A peak
```

---

## MOSFET current and conduction loss

The MOSFET current was:

```text
Peak current: 3.324 A
RMS current:  1.147 A
Average:      0.768 A
```

The OCP margin at low line was:

```text
4.000 − 3.324 = 0.676 A
```

This is approximately `16.9%` of the `4 A` current-limit threshold.

Using the modeled MOSFET on-resistance:

```matlab
RdsOn = 0.30;
```

the conduction-loss estimate was:

```text
MOSFET conduction loss ≈ 0.395 W
```

This does not include:

```text
Turn-on loss
Turn-off loss
Output-capacitance loss
Gate-drive loss
Reverse-recovery-related loss
```

---

## Representative bridge-diode stress

The representative bridge diode D1 experienced:

```text
Peak forward current:  3.365 A
RMS current:           0.954 A
Average current:       0.515 A
Maximum reverse voltage:145.60 V
Maximum forward drop:  1.610 V
```

The measured D1 values agreed closely with the estimates derived from the total bridge-path current:

```text
Estimated per-diode RMS:     0.953 A
Estimated per-diode average: 0.515 A
```

This confirms that one diode is sufficient for representative steady-state bridge stress when all four devices are identical.

The worst bridge reverse voltage still comes from startup:

```text
447.26 V
```

---

## Boost-diode stress

The measured boost-diode values were:

```text
Peak current:             3.361 A
Raw RMS current:          0.707 A
Maximum reverse voltage:  401.49 V
Maximum forward drop:     1.608 V
```

The raw average current initially appeared as:

```text
0.26157 A
```

which was inconsistent with output-stage charge balance.

The charge-balanced average was:

```text
0.24980 A
```

which agrees with the load average current:

```text
Iout,average = 0.24981 A
```

For conservative component selection, the larger raw RMS value was retained:

```text
Boost-diode RMS for rating = 0.707 A
```

---

## Output-capacitor stress

The raw output-capacitor measurements were:

```text
Peak current: 3.111 A
Raw RMS:      0.65718 A
Raw average:  0.01176 A
```

A settled capacitor should have approximately zero average current. The raw `11.76 mA` average would imply a large output-voltage drift that was not present.

The output bus was settled near:

```text
Vout ≈ 399.7 V
```

with ripple:

```text
Vout ripple = 4.472 Vpp
```

The expected capacitor average from the measured bus-voltage endpoints was:

```text
−7.97 µA
```

Therefore, the raw switched-current waveform contained a small accumulated DC-area error.

---

# Switched-current integration problem

## Original symptom

The raw currents satisfied instantaneous output-stage KCL:

```text
iCout = iBoostDiode − Iout
```

The raw averages also satisfied:

```text
0.26157 A − 0.24981 A = 0.01176 A
```

However, the capacitor voltage did not increase as the `11.76 mA` current average would predict.

This showed that the branch-current logs were mutually consistent but their integrated switching-edge area was slightly biased.

---

## Why direct integration was unreliable

The Simscape current signals contain abrupt switching transitions and irregular solver time steps.

Direct trapezoidal integration can connect the left-side and right-side values of a switching transition with an artificial triangular ramp.

This effect is very small per switching event, but the `0.2 s` measurement window contained:

```text
23,486 switching cycles
```

The small edge-area error accumulated into a measurable DC-current bias.

---

## Uniform previous-value resampling

The script was changed to:

```text
1. Remove duplicate timestamps.
2. Keep the final sample at each repeated timestamp.
3. Create a uniform 100 ns time grid.
4. Use previous-value interpolation.
5. Calculate mean and RMS from the uniform samples.
```

The statistics sample time was:

```matlab
statsSampleTime = 100e-9;
```

This avoided creating artificial linear ramps across ideal switching edges.

However, the capacitor average still contained approximately the same small bias, showing that the error was embedded in the logged switching-event data rather than caused only by `trapz`.

---

## Charge-balance correction

Because rerunning the five-second simulation was expensive, the existing logged data was corrected through output-capacitor charge balance.

The expected average capacitor current was calculated as:

$$I_{\text{C,expected}} = C_{\text{out}}\frac{V_{\text{out,end}}-V_{\text{out,start}}}{t_{\text{end}}-t_{\text{start}}}$$

The inferred DC-area bias was:

```text
Raw iCout average:       0.011762649 A
Expected capacitor avg: −0.000007965 A
Inferred DC bias:        0.011770614 A
```

The corrected waveform estimate was formed by subtracting only this inferred DC component:

```matlab
iCout_corrected = iCout_uniform - dcBias;
```

The measured current peaks were not changed.

---

## Corrected capacitor result

After charge-balance correction:

```text
Corrected iCout average:  −7.97 µA
Corrected iCout RMS:       0.657079 A
Capacitor ripple RMS:      0.657079 A
```

The correction changed the RMS value by only approximately:

```text
0.000105 A
```

Therefore, the original capacitor RMS stress was already essentially correct.

The final capacitor ripple-current requirement is:

```text
Cout ripple-current RMS = 0.657 A
```

The direct measured peak remains:

```text
Cout peak current = 3.111 A
```

The startup charging peak remains:

```text
Cout startup peak = 6.547 A
```

---

## Corrected boost-diode result

The charge-balanced boost-diode average was calculated from:

```text
IboostDiode,average = Iout,average + iCout,expected
```

The corrected result was:

```text
Raw boost-diode average:       0.261574 A
Charge-balanced average:       0.249803 A
Raw boost-diode RMS:           0.707345 A
Bias-adjusted RMS estimate:     0.703077 A
```

Because the exact logging error likely occurs around switching edges rather than as a physically uniform DC current, the raw RMS value was retained as the conservative rating value:

```text
Boost-diode RMS for rating = 0.707 A
```

---

# Low-line gate timing

The low-line on-time command was:

```text
Ton command minimum: 6.571 µs
Ton command maximum: 6.612 µs
Ton command average: 6.592 µs
```

The actual gate pulse was:

```text
Minimum: 6.7 µs
Average: 6.731 µs
Maximum: 6.8 µs
```

The difference between commanded and actual on-time is consistent with the `100 ns` sampled timing logic and gate-state delay.

The switching-frequency range was:

```text
Minimum: 100.0 kHz
Average: 117.4 kHz
Maximum: 147.1 kHz
```

This is the expected variable-frequency behavior of a CrCM boost PFC.

---

# Final verified stress values so far

## Startup worst-case values

| Component or quantity | Verified stress |
|---|---:|
| Precharge resistor peak current | 3.903 A |
| Precharge resistor pulse energy | 11.692 J |
| Bypass-switch surge current | 6.552 A |
| Boost-inductor startup peak | 6.551 A |
| Bridge-diode startup peak | 6.551 A |
| Boost-diode startup peak | 6.547 A |
| Output-capacitor charging peak | 6.547 A |
| MOSFET startup current peak | 1.637 A |
| Maximum output voltage | 408.89 V |
| Maximum MOSFET `Vds` | 409.77 V |
| Maximum boost-diode reverse voltage | 408.63 V |
| Maximum bridge-diode reverse voltage | 447.26 V |

## Low-line continuous values

| Component or quantity | Peak | RMS used for rating |
|---|---:|---:|
| AC input / EMI inductors | 1.685 A | 1.154 A |
| Bypass switch | 1.685 A | 1.154 A |
| Boost inductor | 3.365 A | 1.348 A |
| Bridge diode D1 | 3.365 A | 0.954 A |
| MOSFET | 3.324 A | 1.147 A |
| Boost diode | 3.361 A | 0.707 A |
| Output capacitor | 3.111 A | 0.657 A ripple RMS |
| X-capacitor | 1.716 A | 0.704 A |
| Load | 0.251 A | 0.250 A |

---

# Current verification conclusion

The completed startup and low-line tests show:

```text
Startup sequencing works correctly.
OVP does not activate.
OCP does not activate.
The precharge resistor limits the initial current.
The bypass and load sequencing operate correctly.
The MOSFET remains below the 4 A current limit.
The output capacitor remains below the 420 V OVP threshold.
The main passive startup surge is approximately 6.55 A.
The low-line continuous-current stresses are within the simulated limits.
```

The main design requirements identified by the verification are:

```text
Precharge resistor:
At least 11.7 J pulse-energy capability, with design margin.

Boost inductor:
At least 6.55 A transient saturation capability.
At least 1.35 A RMS thermal-current capability.

MOSFET:
At least 3.32 A peak and 1.15 A RMS current capability.
Voltage rating with substantial margin above 410 V.

Bridge:
At least 6.55 A surge capability.
Voltage rating with margin above 447 V.

Boost diode:
At least 6.55 A surge capability.
At least 0.71 A RMS continuous capability.
Voltage rating with margin above 409 V.

Output capacitor:
At least 0.657 A ripple-current capability.
At least 6.55 A startup charging-pulse capability.
Voltage rating above the 408.9 V measured maximum.
```

---

# Remaining verification

The remaining component-stress run is:

```text
264 VAC, 60 Hz
5 s simulation
4.8–5.0 s measurement window
cfg.caseName = "steady_high"
```

The high-line run should focus on:

```text
Maximum MOSFET Vds
Boost-diode reverse voltage
Bridge-diode reverse voltage
Minimum on-time
Actual gate-pulse width
Switching-frequency range
High-line switched-current RMS values
EMI-filter ringing
```

After the high-line run, the component-stress verification will be complete.
