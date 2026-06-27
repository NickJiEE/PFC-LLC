# PFC-LLC
CrCM boost PFC + LLC RC

**Issue 1:**
Constant-on-time CrCM controller

Startup pulse → SET latch
Ton_done      → RESET latch
ZCD           → starts each following cycle

The intended sequence:

MOSFET turns on
→ current rises for 7.1 µs
→ Ton_done turns MOSFET off
→ current falls to zero
→ ZCD starts the next cycle

Original symptom:

The controller switched only in bursts:

Inductor current showed groups of switching pulses.
Switching stopped near the AC zero crossings.
Output remained close to the passive rectifier voltage, approximately 127–130 V at 90 VAC.
The stage did not transfer enough continuous power to raise the bus to 400 V.

Original ZCD implementation:

zcd_signal = iL - Izcd;
zcd_pulse = DetectFallNonpositive(zcd_signal);

Izcd = 0.02;

This only generates a pulse when the current performs this transition:

iL > 0.02 A
then
iL <= 0.02 A

Near the AC zero crossing, the input voltage is very small. With constant on-time, the inductor-current peak also becomes very small:

Small Vin
→ small current ramp
→ peak iL may never exceed 0.02 A

Therefore:

iL never becomes greater than Izcd
→ no downward threshold crossing exists
→ Detect Fall Nonpositive produces no pulse
→ latch never receives another SET command
→ switching stops

It would only restart later when some other current transition happened.

Main solution:

Instead of detecting only a downward current crossing, changed the ZCD logic to detect when the converter enters the valid restart state:

Inductor current is near zero
AND
MOSFET is off

The new condition is:

current_zero = iL <= Izcd;
mosfet_off = ~GateCmdDelayed;

zero_ready = current_zero && mosfet_off;
zcd_pulse = DetectRisePositive(zero_ready);

iL
 ↓
Compare <= Izcd
 ↓
current_zero ───────────┐
                        AND
NOT(GateCmdDelayed) ────┘
                         ↓
                 Detect Rise Positive
                         ↓
                     ZCD pulse

Why:

In normal CrCM operation:

MOSFET turns off
→ current is still above zero
→ zero_ready = 0

current falls below Izcd
→ zero_ready changes 0 → 1
→ ZCD pulse starts next cycle

Near the line zero crossing:

Current may already be below Izcd
while MOSFET is on
→ zero_ready = 0 because MOSFET is on

MOSFET turns off
→ current_zero = 1
→ mosfet_off = 1
→ zero_ready changes 0 → 1
→ ZCD pulse still occurs

So the controller no longer loses the switching sequence near the line zero crossing.

Other:

Added Unit Delay blocks to remove direct algebraic loops.
