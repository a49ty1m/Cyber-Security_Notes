# RF Survey Fundamentals

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → RF Survey Fundamentals → Antennas, Gain, Attenuation & FSPL

# Section 1 — What it is and where it sits

RF survey fundamentals are the ability to reason about **how radio energy propagates between a transmitter and receiver**. For wireless security work, you need to understand antenna radiation patterns, antenna gain, cable/path attenuation, received power, and **Free-Space Path Loss (FSPL)**. These determine whether a wireless signal can actually reach your assessment position and how strongly it arrives.

The practical output is an RF picture: **where a signal is strong, where it becomes weak, what direction an antenna favors, and how distance and losses affect reception**.

```text
WiFi Fundamentals
      ↓
Channels / Frequencies
      ↓
RF Survey
      ↓
Antenna
  ├── Type
  ├── Gain
  └── Radiation pattern
      ↓
Transmission Power
      ↓
Path / Cable Attenuation
      ↓
FSPL
      ↓
Received Signal
      ↓
Wireless Assessment Position
```

If you skip this, you can mistake a weak signal for a nonexistent AP, misunderstand why a directional antenna changes coverage, or incorrectly estimate whether a wireless device can communicate from a particular location.

This follows your regulatory/channel knowledge and prepares you for practical wireless reconnaissance, antenna selection, signal mapping, and later RF-based assessment.

# Section 2 — How attackers actually use this

## 2.1 Establish where the target signal can actually be heard

An attacker starts by determining:

* target AP location or approximate direction
* operating frequency
* received signal strength
* noise level
* antenna type
* antenna orientation
* distance
* physical obstacles
* cable losses
* environmental attenuation

A simplified model is:

```text
AP
 │
 │ Transmit power
 ▼
Antenna
 │
 │ Antenna gain
 ▼
~~~~ RF propagation ~~~~
 │
 │ FSPL + environmental losses
 ▼
Receiver antenna
 │
 │ Antenna gain - cable losses
 ▼
Wireless adapter
```

The goal is not simply:

> "Get a stronger signal."

The goal is to understand **why** the signal has the strength you observe.

---

## 2.2 Understand omnidirectional antennas

An **omnidirectional antenna** is designed to radiate broadly around the antenna's horizontal plane.

A simplified top-down pattern looks like:

```text
             ↑
         .-------.
      .-'         '-.
    .'               '.
   /         ●         \
   \                   /
    '.               .'
      '-._________.-'

          Antenna
```

The pattern is often described as "360°," but this does **not** mean equal radiation in every direction and dimension.

Typically:

```text
Top view:
       360° coverage

Side view:
       flatter radiation pattern
```

Common uses include:

* WiFi APs
* general client coverage
* wireless sensors
* environments where clients can be located around the AP

For an attacker, an omni antenna is useful when the target direction is unknown and broad discovery is more valuable than maximum directional sensitivity.

---

## 2.3 Understand directional antennas

A **directional antenna** concentrates RF energy toward a preferred direction.

Simplified:

```text
                    Target
                      ↑
                      │
                    >>>>>
                  >>>>>>>>
                >>>>>>>>>>>
              >>>>>>>>>>>>>>
                    │
                  Antenna
```

Common directional antenna types include:

* Yagi
* patch
* panel
* parabolic dish

The practical advantage is increased gain in a particular direction.

The tradeoff is:

```text
More directional
      ↓
Narrower useful coverage
      ↓
Requires better aiming
```

This is useful during an RF survey when you already have a likely target direction and want to characterize it more precisely.

---

## 2.4 What dBi actually means

**dBi** means **decibels relative to an isotropic radiator**.

An isotropic radiator is an ideal mathematical reference that radiates equally in all directions.

Antenna gain therefore describes **how strongly an antenna concentrates energy in a particular direction relative to that reference**.

For example:

```text
2 dBi omni
     ↓
Broad radiation

12 dBi directional
     ↓
More concentrated radiation
     ↓
Higher gain in preferred direction
```

A common mistake is:

> "A 12 dBi antenna creates 12 dB more total power."

That is wrong.

Antenna gain is about **directional concentration**, not creation of additional energy.

---

## 2.5 Understand the gain/range relationship correctly

Higher antenna gain can increase usable range in a particular direction, but there is no universal:

```text
2× gain = 2× range
```

relationship.

The actual result depends on:

* frequency
* transmit power
* receiver sensitivity
* antenna radiation pattern
* cable losses
* obstacles
* interference
* polarization
* regulatory power limits

Therefore:

```text
Higher dBi
    ≠
Automatically double the range
```

Instead:

```text
Higher directional gain
       ↓
More energy concentrated in preferred direction
       ↓
Potentially stronger received signal
       ↓
Potentially longer usable link
```

---

## 2.6 Calculate Free-Space Path Loss

**FSPL** estimates the signal loss caused by propagation through ideal free space.

A commonly used formula is:

```text
FSPL(dB) =
20 log10(d)
+
20 log10(f)
+
32.44
```

when:

* `d` = distance in kilometers
* `f` = frequency in MHz

For example, suppose:

```text
Distance = 1 km
Frequency = 2400 MHz
```

Then:

```text
FSPL
= 20 log10(1)
+ 20 log10(2400)
+ 32.44

≈ 100.04 dB
```

At the same distance, higher frequency produces greater free-space path loss.

For example:

```text
2.4 GHz → lower FSPL
5 GHz   → higher FSPL
6 GHz   → higher still
```

The difference is not because 5 GHz radio is inherently "bad."

It is a mathematical consequence of frequency and wavelength in free-space propagation.

---

## 2.7 Understand distance correctly

FSPL increases with distance.

Every time distance increases by a factor of 2:

```text
Distance ×2
    ↓
FSPL increases by ≈ 6 dB
```

For example:

```text
1 m → baseline
2 m → +6 dB loss
4 m → +12 dB loss
8 m → +18 dB loss
```

This is why moving only a few meters can noticeably change a weak wireless signal.

The relationship is logarithmic, not linear.

---

## 2.8 Understand attenuation

**Attenuation** is signal power loss.

Sources include:

* free-space propagation
* walls
* concrete
* glass
* metal
* foliage
* cables
* connectors
* imperfect antenna matching
* atmospheric/environmental effects

A simplified link budget is:

```text
Received Power
=
Transmit Power
+
Transmit Antenna Gain
+
Receive Antenna Gain
-
FSPL
-
Other Losses
```

Example:

```text
Transmit power       = 20 dBm
TX antenna gain      = 5 dBi
RX antenna gain      = 3 dBi
FSPL                 = 90 dB
Cable/other losses   = 5 dB
--------------------------------
Received power       = -67 dBm
```

The calculation is useful because it explains the observed signal rather than relying entirely on a scanner's RSSI value.

---

## 2.9 Distinguish attenuation from antenna gain

These are opposite concepts in a basic link budget.

```text
Antenna gain
    ↑
Improves directional link budget

Attenuation
    ↓
Reduces received power
```

For example:

```text
10 dBm transmitter
+ 8 dBi antenna
- 3 dB cable loss
= 15 dBm effective result
```

The antenna did not create 8 dB of power.

It changed how energy is distributed spatially.

---

## 2.10 Dead-end finding vs high-value finding

### Dead-end finding

```text
Lab AP:
RSSI = -82 dBm
```

That alone does not tell you why the signal is weak.

Possible explanations include:

* distance
* wall attenuation
* antenna orientation
* cable loss
* interference
* multipath
* receiver characteristics

### High-value finding

```text
AP:
5 GHz
Directional antenna
RSSI: -82 dBm

Survey:
Strong signal toward east
Weak signal toward west
Large attenuation through concrete wall
```

Now the directional pattern and physical environment provide a plausible explanation.

This can inform the next step:

```text
RF survey
   ↓
Signal direction
   ↓
Likely AP location
   ↓
Better assessment position
   ↓
More reliable wireless observation
```

---

## 2.11 Use directional behavior to locate an AP

Suppose a survey shows:

```text
Position A → -72 dBm
Position B → -61 dBm
Position C → -48 dBm
Position D → -68 dBm
```

The strongest reading is at Position C.

An operator can use repeated measurements to estimate the direction/location of the transmitter.

Conceptually:

```text
        C (-48)
          ↑
          │
A (-72) ──┼── B (-61)
          │
        D (-68)

        AP likely
        toward C
```

This is more useful than treating one RSSI value as a precise location.

---

## 2.12 Understand real-world propagation limitations

FSPL assumes ideal free-space propagation.

Real buildings are not free space.

A real link may experience:

```text
FSPL
 +
Wall loss
 +
Furniture
 +
Metal
 +
Multipath
 +
Interference
 +
Antenna mismatch
```

Therefore:

```text
Theoretical FSPL
       ≠
Actual measured loss
```

FSPL is a baseline model.

An RF survey measures reality.

# Section 3 — Core concepts and terminology

| Term                        | Meaning                                                                                 |
| --------------------------- | --------------------------------------------------------------------------------------- |
| **RF**                      | Radio Frequency; electromagnetic energy used for wireless communication.                |
| **Antenna**                 | Device that converts electrical signals to radio waves and vice versa.                  |
| **Omnidirectional antenna** | Antenna designed to provide broad horizontal coverage.                                  |
| **Directional antenna**     | Antenna concentrating radiation toward preferred directions.                            |
| **Antenna gain**            | Directional concentration relative to a reference antenna.                              |
| **dBi**                     | Antenna gain measured relative to an ideal isotropic radiator.                          |
| **Attenuation**             | Reduction in signal power.                                                              |
| **Path loss**               | Reduction in received signal power caused by propagation.                               |
| **FSPL**                    | Free-Space Path Loss; theoretical propagation loss in unobstructed free space.          |
| **Link budget**             | Accounting of transmitter power, antenna gains, and losses to estimate received power.  |
| **TX power**                | Power transmitted by the radio.                                                         |
| **RX sensitivity**          | Minimum signal level a receiver needs for a specified performance level.                |
| **RSSI**                    | Receiver-dependent indication of received signal strength.                              |
| **SNR**                     | Signal-to-noise ratio.                                                                  |
| **Noise floor**             | Background RF energy measured by the receiver.                                          |
| **Polarization**            | Orientation of the electromagnetic field produced by an antenna.                        |
| **Multipath**               | Multiple copies of a signal arriving through different propagation paths.               |
| **Fade**                    | Reduction in received signal caused by destructive interference or propagation effects. |
| **Cable loss**              | RF energy lost through coaxial cable and connectors.                                    |
| **Isotropic radiator**      | Ideal reference antenna radiating equally in all directions.                            |
| **Radiation pattern**       | Spatial representation of how an antenna radiates or receives energy.                   |
| **Beamwidth**               | Angular width of a directional antenna's main beam.                                     |
| **EIRP**                    | Effective radiated power accounting for transmitter output and antenna gain/loss.       |

### Antenna comparison

| Property                    | Omni                 | Directional                 |
| --------------------------- | -------------------- | --------------------------- |
| Coverage                    | Broad                | Focused                     |
| Typical horizontal coverage | ~360°                | Limited angular sector      |
| Gain                        | Usually lower        | Can be substantially higher |
| Aiming                      | Minimal              | Important                   |
| Useful when                 | Direction unknown    | Target direction known      |
| Typical examples            | Dipole/vertical omni | Yagi/patch/panel/dish       |

### Important equations

**FSPL:**

```text
FSPL(dB) =
20 log10(d km)
+
20 log10(f MHz)
+
32.44
```

**Simplified link budget:**

```text
Pr(dBm) =
Pt(dBm)
+ Gt(dBi)
+ Gr(dBi)
- FSPL(dB)
- L(dB)
```

Where:

```text
Pr = received power
Pt = transmitted power
Gt = transmit antenna gain
Gr = receive antenna gain
L  = additional losses
```

# Section 4 — Tools and commands

| Tool       | Command                                                                       | What it finds/shows                                | When to use it                               |
| ---------- | ----------------------------------------------------------------------------- | -------------------------------------------------- | -------------------------------------------- |
| `iw`       | `iw dev wlan0 link`                                                           | Current AP, frequency, signal and link information | Measure an associated link                   |
| `nmcli`    | `nmcli device wifi list`                                                      | Nearby AP signal levels and channels               | Quick RF survey                              |
| `iw`       | `sudo iw dev wlan0 scan`                                                      | Detailed AP signal/frequency information           | Detailed measurements                        |
| `wavemon`  | `sudo wavemon`                                                                | Live signal/noise/link statistics                  | Interactive RF observation                   |
| `iwconfig` | `iwconfig wlan0`                                                              | Legacy wireless signal information                 | Quick legacy interface inspection            |
| `iperf3`   | `iperf3 -s`                                                                   | Application-layer throughput                       | Correlate RF quality with actual performance |
| `iperf3`   | `iperf3 -c <LAB_IP>`                                                          | Client-side throughput                             | Measure link performance                     |
| `python`   | `python3 -c "import math; print(20*math.log10(1)+20*math.log10(2400)+32.44)"` | FSPL calculation                                   | Calculate theoretical free-space loss        |

### `iw dev wlan0 link`

```text
$ iw dev wlan0 link

Connected to aa:bb:cc:11:22:33
    SSID: Lab-WiFi
    freq: 5180
    signal: -51 dBm
```

Interpretation:

```text
Frequency → 5180 MHz / 5 GHz
Signal    → -51 dBm
AP        → AA:BB:CC:11:22:33
```

A value closer to zero generally represents stronger received power.

---

### `nmcli`

```text
$ nmcli device wifi list

SSID       CHAN  FREQ      SIGNAL
Lab-WiFi     36  5180 MHz    79
Neighbor     6   2437 MHz    42
```

This provides a convenient quick survey.

The `SIGNAL` percentage is not directly equivalent to dBm and should not be treated as a universal physical measurement.

---

### `iw dev wlan0 scan`

```text
$ sudo iw dev wlan0 scan

BSS aa:bb:cc:11:22:33
    SSID: Lab-WiFi
    freq: 5180
    signal: -51.00 dBm
```

This provides the raw signal measurement reported by the wireless stack.

---

### `wavemon`

```text
$ sudo wavemon
```

A typical display contains information resembling:

```text
Interface: wlan0
Link quality: 58/70
Signal level: -52 dBm
Noise level: -92 dBm
```

You can estimate:

```text
SNR ≈ Signal - Noise
```

For the example:

```text
-52 - (-92)
= 40 dB
```

A stronger signal does not automatically mean a better link if the noise floor is also high.

---

### `iwconfig`

```text
$ iwconfig wlan0

wlan0
    Signal level=-52 dBm
    Link Quality=58/70
```

This is useful on systems that still expose wireless information through the older `iwconfig` interface.

Prefer `iw` for modern Linux wireless work.

---

### `iperf3`

Server:

```text
$ iperf3 -s
```

Client:

```text
$ iperf3 -c 192.168.1.10
```

Example:

```text
[  5]   0.00-10.00 sec   420 MBytes   352 Mbits/sec
```

This measures actual network throughput.

That lets you compare:

```text
RSSI/SNR
   ↓
Actual throughput
```

rather than assuming a strong signal automatically means high performance.

---

### FSPL calculation

```text
$ python3 -c "import math; print(20*math.log10(1)+20*math.log10(2400)+32.44)"
```

Example:

```text
100.041...
```

Therefore:

```text
FSPL ≈ 100.04 dB
```

for a 1 km, 2400 MHz free-space path.

This is a theoretical value, not a measurement of a real building.

# Section 5 — Defender detection

* **RF sensors:** Wireless monitoring systems can detect unexpected transmitters, signal-strength changes, channel occupancy, and suspicious RF activity.
* **AP telemetry:** Enterprise APs expose client RSSI, SNR, channel utilization, retransmissions, and roaming behavior.
* **Physical survey correlation:** Unexpected strong signals outside expected coverage can indicate rogue APs or incorrectly positioned equipment.
* **Common miss:** Teams often monitor IP traffic but ignore the RF layer, missing devices that are physically transmitting even before they establish normal network sessions.
* **Directional anomalies:** A signal unexpectedly appearing strongly from a restricted or unusual direction can justify physical investigation.
* **Operator footprint reduction:** Passive RF observation generally produces less network-layer evidence than actively associating, authenticating, or transmitting test traffic.
* **Important limitation:** RSSI alone cannot reliably identify an attacker's physical distance; antenna gain, transmit power, obstacles, and propagation conditions all affect the measurement.

# Section 6 — Lab task

**Platform:** Local Kali Linux + your own WiFi AP + USB WiFi adapter. Place the AP at a known location and perform measurements from several known positions.

**Objective:** Measure signal strength at multiple positions and use antenna/path-loss concepts to explain the observed RF pattern.

**Steps:**

1. Place your lab AP at a fixed location and record its operating frequency.
2. Start from a known position approximately 2 m from the AP.
3. Record RSSI, noise level if available, channel, and frequency.
4. Repeat measurements at approximately 4 m, 8 m, and 16 m.
5. Keep the AP and client orientation approximately constant.
6. Calculate the theoretical FSPL for the frequency and each distance.
7. Compare the theoretical values with the measured RSSI changes.
8. Repeat one measurement with a significant physical obstacle between AP and client.
9. Record the difference between free-space prediction and real-world observation.
10. Save the measurements and interpretation as Markdown.

**Expected output:**

```text
Distance    RSSI       FSPL trend
2 m         -42 dBm    baseline
4 m         -48 dBm    ~6 dB additional FSPL
8 m         -54 dBm    ~12 dB additional FSPL
16 m        -60 dBm    ~18 dB additional FSPL
```

Your actual measurements will differ because a building is not free space.

The important result is demonstrating that **doubling distance adds approximately 6 dB of free-space path loss**, while real obstacles introduce additional attenuation and multipath effects.

**Git artifact:**

```text
rf-survey/
├── README.md
├── notes/
│   └── rf-measurements.md
└── evidence/
    └── signal-measurements.csv
```

```bash
git add rf-survey/
git commit -m "Add RF survey path loss measurements"
```

# Section 7 — Common mistakes

1. **Thinking high dBi means more transmitter power** → antenna gain concentrates energy rather than creating energy → understand gain as directional concentration.

2. **Assuming RSSI directly tells you distance** → transmit power, antenna gain, obstacles, and multipath change RSSI → use multiple measurements and physical context.

3. **Treating FSPL as real-world path loss** → FSPL assumes ideal free space → add environmental attenuation when interpreting buildings and outdoor environments.

4. **Assuming directional antennas are always better** → high-gain antennas have narrower coverage and require accurate aiming → choose omni or directional based on the survey objective.

5. **Ignoring cable and connector losses** → RF energy can be lost before reaching the antenna → include cable/connector attenuation in the link budget.

6. **Confusing dBm and dBi** → dBm measures power while dBi describes antenna gain relative to an isotropic reference → never substitute one for the other.

7. **Looking only at signal strength** → a strong signal can coexist with significant noise/interference → measure SNR and, where useful, correlate RF measurements with throughput.

# Section 8 — Move-on gate

1. **Measure your lab AP at four different distances, calculate FSPL for each distance, and correctly explain the approximately 6 dB loss introduced by every doubling of distance without looking at your notes.**

2. **Use an omni and directional antenna in a controlled lab measurement and map the signal at multiple positions, correctly identifying the directional antenna's stronger preferred direction without looking at your notes.**

3. **Given a transmitter power, two antenna gains, frequency, distance, and cable loss, calculate the expected received power using a link budget and compare it against the measured RSSI from your lab.**
