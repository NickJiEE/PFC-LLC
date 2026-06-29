# Issue 1: CrCM Controller Switching Only in Bursts

## Problem

The original CrCM controller did not switch continuously across the full rectified line cycle.

The observed behavior was:

```text
Inductor current appeared in groups of pulses.
Switching stopped near the AC zero crossings.
The boost stage transferred too little average power.
Vout remained near the passive bridge-rectifier level instead of regulating to 400 V.
```

The controller was intended to operate as:

```text
SET latch
→ MOSFET turns on
→ on-time timer reaches Ton_cmd
→ Ton_done resets the latch
→ MOSFET turns off
→ inductor current demagnetizes
→ ZCD generates the next SET pulse
```

---

## Original ZCD implementation

The first zero-current detector used a falling threshold crossing:

```matlab
zcd_signal = iL - Izcd;
zcd_pulse = DetectFallNonpositive(zcd_signal);
```

with:

```matlab
Izcd = 0.02;   % A
```

A pulse was produced only after:

```text
iL > Izcd
then
iL <= Izcd
```

---

## Why the original ZCD failed

Near the AC zero crossing, the instantaneous input voltage is small.

For constant-on-time CrCM operation:

```text
Small Vin
→ small positive inductor-current slope
→ small peak inductor current
```

The peak current could remain below the ZCD threshold:

```text
iL,peak <= Izcd
```

When this happened, the current never first rose above `Izcd`. Therefore, no falling threshold crossing existed.

The resulting sequence was:

```text
Current pulse remains below Izcd
→ falling-edge detector produces no pulse
→ latch receives no new SET request
→ switching stops
→ controller enters burst-like operation
```

---

## Final ZCD method

The ZCD was changed from a current-threshold crossing detector to a valid-restart-state detector.

The valid restart state is:

```text
Inductor current is near zero
AND
MOSFET is off
```

The final logic is:

```matlab
current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;
zcd_pulse = DetectRisePositive(zero_ready);
```

Block structure:

```text
iL
 ↓
Compare: iL <= Izcd
 ↓
current_zero ────────────────┐
                             AND
NOT(GateCmdDelayed) ─────────┘
                              ↓
                    Detect Rise Positive
                              ↓
                          zcd_pulse
```

`GateCmdDelayed` is the gate command delayed by one fast control sample using the Memory block. This delay breaks the direct Simulink feedback path between the latch output and the ZCD logic.

---

## Why the final ZCD works

### Normal CrCM cycle

After the MOSFET turns off:

```text
MOSFET off
iL still above Izcd
→ zero_ready = 0
```

When the current demagnetizes below the threshold:

```text
iL <= Izcd
MOSFET remains off
→ zero_ready changes from 0 to 1
→ rising-edge detector generates zcd_pulse
```

### Near the line zero crossing

The current may remain below `Izcd` for the entire on-time.

While the MOSFET is on:

```text
current_zero = 1
mosfet_off = 0
→ zero_ready = 0
```

When the MOSFET turns off:

```text
current_zero remains 1
mosfet_off changes to 1
→ zero_ready changes from 0 to 1
→ zcd_pulse is generated
```

The controller therefore restarts even when no downward current-threshold crossing occurred.

---

## Final latch and timer integration

The final controller uses a variable commanded on-time:

```text
Ton_ff + Ton_P + Ton_I
→ Ton_unlimited
→ dynamic saturation
→ Ton_cmd
→ Rate Transition
→ Ton_cmd_fast
→ fast on-time timer
```

The fast control sample time is:

```matlab
Ts_ctrl = 100e-9;
```

When the latch sets:

```text
GateCmd rises.
The timer counts while the MOSFET is commanded on.
Ton_done is generated when elapsed on-time reaches Ton_cmd_fast.
Ton_done resets the latch.
```

Because the timer and latch are discrete, the measured gate pulse is quantized in `100 ns` increments and can be approximately one or two samples longer than the requested `Ton_cmd`.

The final SET path is:

```matlab
set_request = startup_pulse || zcd_pulse;
SET_final = set_request && ControllerEnable;
```

The controller-disable path forces the latch reset:

```matlab
RESET_final = RESET_existing || ~ControllerEnable;
```

`RESET_existing` includes the normal timer and protection reset requests.

---

## Startup behavior

The startup pulse is still required because no previous demagnetization event exists before the first switching cycle.

In the final startup sequence:

```text
ControllerEnable rises at 0.160 s.
Startup pulse occurs at 0.161 s.
The first cycle begins.
All following cycles are initiated by the final ZCD state-transition logic.
```

---

## Effect on validation

The original burst behavior made the boost stage appear unable to regulate and made line-current and switching-frequency measurements invalid.

After the ZCD correction:

```text
Switching continues across the rectified line cycle.
The inductor-current envelope follows the rectified input.
The boost stage transfers continuous average power.
The output can regulate near 400 V.
```

---

## Final implementation

```matlab
Izcd = 0.02;

current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;
zcd_pulse = rising_edge(zero_ready);

set_request = startup_pulse || zcd_pulse;
SET_final = set_request && ControllerEnable;

RESET_final = RESET_existing || ~ControllerEnable;
```

---

## Main conclusion

The failure was caused by requiring a falling current-threshold crossing that did not exist when the peak current remained below `Izcd`.

The final solution detects entry into the valid restart state:

```text
MOSFET off
AND
inductor current near zero
```

This preserves the CrCM switching sequence near the AC zero crossing and prevents burst-like loss of switching.
