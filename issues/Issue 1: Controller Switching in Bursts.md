# Issue 1: CrCM Controller Switching Only in Bursts

## Original Controller Structure

The constant-on-time CrCM controller was built as:

```text
Startup pulse -> SET latch
Ton_done      -> RESET latch
ZCD           -> starts each following cycle
```

The intended switching sequence was:

```text
MOSFET turns on
-> inductor current rises for approximately 7.1 us
-> Ton_done resets the latch and turns the MOSFET off
-> inductor current falls back to zero
-> ZCD generates a new SET pulse
-> the next switching cycle begins
```

---

## Original Symptom

The controller did not switch continuously across the full rectified line cycle.

Instead:

* The inductor current appeared in groups of switching pulses.
* Switching stopped near the AC zero crossings.
* The output voltage remained close to the passive bridge-rectifier voltage.
* At 90 VAC input, the output stayed near approximately 127–130 V.
* The boost stage did not transfer enough continuous power to raise the DC bus to 400 V.

The waveform therefore looked like burst operation rather than continuous CrCM operation.

---

## Original ZCD Implementation

The first zero-current detector used:

```matlab
zcd_signal = iL - Izcd;
zcd_pulse = DetectFallNonpositive(zcd_signal);
```

with:

```matlab
Izcd = 0.02;   % A
```

This logic only generated a pulse when the inductor current completed the following transition:

```text
iL > 0.02 A
then
iL <= 0.02 A
```

In other words, the current first had to rise above the threshold and then fall back below it.

---

## Why the Original ZCD Failed

Near the AC zero crossing, the instantaneous input voltage becomes very small.

With constant on-time control:

```text
Small Vin
-> small inductor-current slope
-> small peak inductor current
```

Eventually, the current pulse could become so small that:

```text
peak iL <= Izcd
```

The current then never rose above the 0.02 A ZCD threshold.

Therefore:

```text
iL never becomes greater than Izcd
-> no downward threshold crossing exists
-> Detect Fall Nonpositive produces no pulse
-> the latch receives no new SET command
-> switching stops
```

The controller would only restart later when another current transition happened to recreate a valid falling threshold crossing.

This caused the observed groups of switching pulses and long inactive intervals.

---

## Main Solution: Detect the Valid Restart State

Instead of detecting only a downward crossing of the current threshold, the ZCD logic was changed to detect when the converter enters the valid restart state:

```text
Inductor current is near zero
AND
MOSFET is off
```

The new logic became:

```matlab
current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;
zcd_pulse = DetectRisePositive(zero_ready);
```

---

## New ZCD Block Structure

```text
iL
 |
 v
Compare <= Izcd
 |
 v
current_zero ------------------+
                               |
                               AND
NOT(GateCmdDelayed) -----------+
                               |
                               v
                     Detect Rise Positive
                               |
                               v
                           ZCD pulse
```

The delayed gate command was used because it represents the Boolean state actually being applied to the MOSFET gate path.

---

## Why the New ZCD Logic Works

### During Normal CrCM Operation

After the MOSFET turns off:

```text
MOSFET is off
inductor current is still above Izcd
-> current_zero = 0
-> zero_ready = 0
```

As the inductor current falls below the threshold:

```text
current_zero changes from 0 to 1
mosfet_off remains 1
-> zero_ready changes from 0 to 1
-> Detect Rise Positive generates a ZCD pulse
-> the next switching cycle begins
```

### Near the AC Zero Crossing

Near the line zero crossing, the current pulse may never exceed `Izcd`.

While the MOSFET is on:

```text
current may already be below Izcd
but mosfet_off = 0
-> zero_ready = 0
```

When the MOSFET turns off:

```text
current_zero = 1
mosfet_off changes to 1
-> zero_ready changes from 0 to 1
-> Detect Rise Positive generates a ZCD pulse
```

A new cycle is therefore generated even when the current never crossed downward through the 0.02 A threshold.

---

## Behavior After the Fix

Before the fix:

```text
Current pulse becomes too small near line zero
-> no falling ZCD crossing
-> no SET pulse
-> switching stops
-> burst operation
```

After the fix:

```text
MOSFET turns off while current is near zero
-> zero_ready rises
-> ZCD pulse is generated
-> switching continues
```

The controller no longer loses the switching sequence near the AC zero crossing.

As a result:

* CrCM switching continued across the full rectified line cycle.
* The inductor-current envelope followed the rectified input voltage.
* The boost stage transferred continuous average power.
* The output voltage was able to rise from the passive rectifier level toward 400 V.

---

## Final ZCD Logic

```matlab
current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;

zcd_pulse = rising_edge(zero_ready);

set_request = startup_pulse || zcd_pulse;
```

The startup pulse is still required to initiate the first switching cycle because no previous zero-current event exists at startup.

---

## Main Takeaway

The main correction was changing the ZCD method from:

```text
Detect the current falling through a threshold
```

to:

```text
Detect the transition into the valid restart state:
MOSFET off and inductor current near zero
```

The original threshold-crossing detector could miss the ZCD event when the peak current was smaller than the threshold. The replacement logic remains valid even near the AC zero crossing, so it prevents the controller from falling into burst operation.
::: 
