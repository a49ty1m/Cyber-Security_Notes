# Channel Planning

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → RF Survey Fundamentals → 2.4/5/6 GHz Channel Spacing, Overlap & Interference

# Section 1 — What it is and where it sits

WiFi channel planning is the process of selecting **frequency channels and channel widths** so nearby access points can operate with as little harmful interference as practical. The core idea is simple: an AP does not occupy an infinitely thin frequency point. Its channel has a **width**, and neighboring channels can overlap when their occupied frequency ranges intersect.

For a security practitioner, channel planning helps explain why an AP may have poor performance despite a strong signal, why dense environments use carefully selected channels, and why a scanner showing many nearby APs does not automatically mean all of them interfere equally.

```text
802.11 Standards
      ↓
2.4 / 5 / 6 GHz Bands
      ↓
Channels + Channel Width
      ↓
Channel Spacing
      ↓
Overlap
      ↓
Co-channel / Adjacent-channel Interference
      ↓
RF Survey
      ↓
Wireless Assessment
```

If you skip this, you may mistake **co-channel contention** for adjacent-channel interference, assume every neighboring AP is equally harmful, or choose a channel simply because it appears empty in a scan.

This follows your RF fundamentals and prepares you for wireless reconnaissance, RF troubleshooting, AP placement, and interference analysis.

# Section 2 — How attackers actually use this

## 2.1 Build a channel map

An operator first maps:

* SSID
* BSSID
* frequency
* channel
* channel width
* signal strength
* channel utilization
* neighboring APs
* overlapping channels
* noise/interference

A basic survey might produce:

```text
2.4 GHz

AP-A → Channel 1  → 20 MHz → -45 dBm
AP-B → Channel 6  → 20 MHz → -61 dBm
AP-C → Channel 11 → 20 MHz → -70 dBm
```

That is a relatively clean 2.4 GHz layout.

Compare:

```text
AP-A → Channel 1
AP-B → Channel 3
AP-C → Channel 5
```

These channels have significant spectral overlap when using typical 20 MHz operation.

The second environment is therefore much more likely to experience adjacent-channel interference.

---

## 2.2 Understand channel spacing versus channel width

This distinction is fundamental.

**Channel spacing** describes how far apart channel center frequencies are.

**Channel width** describes how much spectrum an individual transmission occupies.

For example, in the commonly used 2.4 GHz WiFi channel plan:

```text
Channel 1  center ≈ 2412 MHz
Channel 6  center ≈ 2437 MHz
Channel 11 center ≈ 2462 MHz
```

The centers are separated by:

```text
25 MHz
```

while a typical 20 MHz WiFi transmission occupies roughly 20 MHz of spectrum.

Therefore:

```text
Channel 1  ───────────────
                 25 MHz
Channel 6                 ───────────────
```

These commonly used channels provide practical separation.

But:

```text
Channel 1
Channel 2
Channel 3
```

are much closer together and can overlap significantly.

---

## 2.3 2.4 GHz: why overlap matters so much

2.4 GHz is particularly challenging because there is relatively little spectrum available for WiFi.

A simplified representation:

```text
2.4 GHz

Ch 1       Ch 6       Ch 11
  ↓          ↓          ↓
[======]  [======]  [======]
```

With 20 MHz channels, a common planning approach is to use:

```text
1 / 6 / 11
```

where permitted by the applicable regional channel plan.

The reason is not that channels 2–5 or 7–10 are "fake."

It is because using well-separated channels minimizes overlap between 20 MHz transmissions.

---

## 2.4 Understand co-channel interference

Two APs operating on the **same channel** are experiencing a form of competition called **co-channel contention/interference**.

For example:

```text
AP-A ── Channel 6
AP-B ── Channel 6
AP-C ── Channel 6
```

This is not automatically disastrous.

WiFi is designed to allow multiple stations to share a channel using mechanisms such as **CSMA/CA**.

The problem is that the devices must compete for airtime.

```text
More devices
     ↓
More contention
     ↓
More waiting/retries
     ↓
Lower effective throughput
```

Therefore, same-channel APs can be preferable to badly overlapping adjacent channels in some deployments.

This is a critical distinction:

```text
Same channel
→ contention / airtime sharing

Overlapping channels
→ adjacent-channel interference
```

---

## 2.5 Understand adjacent-channel interference

Adjacent-channel interference occurs when transmissions from different channels overlap in frequency.

For example:

```text
AP-A: Channel 6
AP-B: Channel 7
```

The occupied spectrum can overlap.

Conceptually:

```text
Frequency →
       AP-A
    [=========]
         [=========]
           AP-B
```

The receivers may have difficulty distinguishing the desired transmission from energy on the neighboring channel.

This can produce:

* retransmissions
* lower throughput
* increased latency
* reduced effective range
* unstable connections

---

## 2.6 Identify the difference between interference and congestion

These terms are often incorrectly used interchangeably.

### Congestion

Many legitimate WiFi devices compete for airtime.

```text
Channel 6
│
├── AP-A
├── AP-B
├── AP-C
└── Many clients
```

The devices may coordinate through WiFi's channel-access mechanisms, but available airtime becomes scarce.

### Interference

Unwanted RF energy disrupts reception.

It can originate from:

* overlapping WiFi transmissions
* non-WiFi devices
* Bluetooth
* microwave ovens
* other RF systems

Therefore:

```text
High AP count
≠
Automatically high RF interference
```

You need measurements.

---

## 2.7 5 GHz gives planners more room

5 GHz provides substantially more usable spectrum than 2.4 GHz, although the exact channels available depend on the regulatory domain and DFS requirements.

A simplified channel plan might look like:

```text
5 GHz

36   40   44   48
│    │    │    │
└────┴────┴────┴──

149  153  157  161
│    │    │    │
└────┴────┴────┴──
```

The exact available channel set varies by region.

5 GHz therefore provides more opportunities for spatial reuse and channel separation.

---

## 2.8 Wider channels change the planning problem

Modern WiFi can use:

```text
20 MHz
40 MHz
80 MHz
160 MHz
```

A 20 MHz channel:

```text
[====]
```

An 80 MHz channel:

```text
[================]
```

The 80 MHz channel occupies substantially more spectrum.

That can increase theoretical throughput but leaves fewer independent channels available for nearby APs.

For example:

```text
Many APs
   +
80 MHz everywhere
   ↓
Fewer independent channel choices
   ↓
More contention
```

Therefore, **wider is not automatically better**.

Dense enterprise networks often deliberately use narrower channels to increase spatial reuse.

---

## 2.9 6 GHz changes the planning possibilities

6 GHz provides substantially more contiguous spectrum for modern WiFi deployments.

This makes wider channels more practical in suitable environments.

Conceptually:

```text
6 GHz
│
├── 20 MHz
├── 40 MHz
├── 80 MHz
├── 160 MHz
└── More spectrum for modern deployments
```

However, channel availability and exact channel numbering depend on the regulatory domain.

The 6 GHz band is also designed around newer WiFi generations and does not provide the same backward compatibility as 2.4/5 GHz.

---

## 2.10 Detect a high-value RF condition

### Dead-end finding

```text
10 APs detected nearby.
```

This is almost meaningless by itself.

You do not know:

* their channels
* widths
* signal levels
* physical locations
* utilization
* whether they are even relevant

### High-value finding

```text
Target AP:
Channel 6
20 MHz
RSSI: -48 dBm

Nearby AP:
Channel 7
20 MHz
RSSI: -51 dBm

Channel utilization: high
Retries: elevated
```

Now you have evidence suggesting a potentially significant RF problem.

The next investigation becomes:

```text
Channel overlap
      ↓
RF measurements
      ↓
Channel utilization
      ↓
Retransmissions / errors
      ↓
Determine interference source
```

---

## 2.11 Use channel planning to understand attack positioning

During an authorized wireless assessment, channel planning helps determine **where to collect useful observations**.

For example:

```text
Target AP
   ↓
5 GHz / Channel 36
   ↓
Strong signal at assessment position
   ↓
Low competing-channel activity
   ↓
Good observation point
```

Conversely:

```text
Target AP
   ↓
Channel 36
   ↓
Many strong APs sharing/overlapping spectrum
   ↓
High RF contention
   ↓
Noisy measurement environment
```

The second environment makes packet capture and performance measurements less reliable.

Channel knowledge therefore improves the **quality of your evidence**, not just your theoretical understanding.

---

## 2.12 Channel width can create unexpected overlap

Consider two networks:

```text
AP-A
Center: Channel 36
Width: 80 MHz

AP-B
Center: Channel 40
Width: 80 MHz
```

These are not two independent narrow 20 MHz channels.

The wide channels consume multiple underlying 20 MHz portions.

Therefore:

```text
20 MHz planning
       ↓
Fine-grained channel separation

80 MHz planning
       ↓
Fewer independent blocks
```

This is why channel width must always be recorded during an RF survey.

---

## 2.13 Pivot from channel analysis

The assessment path becomes:

```text
Discover APs
    ↓
Record channel
    ↓
Record channel width
    ↓
Measure signal
    ↓
Map neighboring APs
    ↓
Identify overlap
    ↓
Measure utilization/noise
    ↓
Determine likely interference
```

This allows you to distinguish:

```text
Weak signal
```

from:

```text
Strong signal + poor SNR
```

The second case may indicate interference rather than inadequate coverage.

# Section 3 — Core concepts and terminology

| Term                              | Meaning                                                                                      |
| --------------------------------- | -------------------------------------------------------------------------------------------- |
| **Channel**                       | Defined RF portion used by WiFi.                                                             |
| **Channel center frequency**      | Frequency at the center of a channel.                                                        |
| **Channel spacing**               | Frequency separation between channel center frequencies.                                     |
| **Channel width**                 | Amount of spectrum occupied by the WiFi transmission.                                        |
| **Overlap**                       | Situation where occupied frequency ranges of channels intersect.                             |
| **Co-channel**                    | Multiple networks operating on the same channel.                                             |
| **Co-channel contention**         | Devices sharing airtime on the same channel.                                                 |
| **Adjacent-channel interference** | Interference caused by overlapping nearby channels.                                          |
| **Airtime**                       | Time during which a channel is occupied by transmissions.                                    |
| **Channel utilization**           | Measure of how much channel airtime is occupied.                                             |
| **Interference**                  | Unwanted RF energy that disrupts communication.                                              |
| **Congestion**                    | High demand for available wireless airtime.                                                  |
| **Noise floor**                   | Background RF energy detected by a receiver.                                                 |
| **SNR**                           | Desired signal strength relative to noise.                                                   |
| **Spatial reuse**                 | Reusing the same channel in sufficiently separated locations.                                |
| **Channel bonding**               | Combining adjacent channel blocks to create a wider channel.                                 |
| **20 MHz channel**                | Common narrow channel width used for efficient channel reuse.                                |
| **40 MHz channel**                | Two 20 MHz portions combined.                                                                |
| **80 MHz channel**                | Four 20 MHz portions combined.                                                               |
| **160 MHz channel**               | Eight 20 MHz portions combined, subject to availability.                                     |
| **DFS channel**                   | Channel requiring dynamic frequency selection under applicable regulations.                  |
| **Non-WiFi interference**         | RF energy from systems other than WiFi.                                                      |
| **CCI**                           | Co-Channel Interference, commonly referring to interference/competition on the same channel. |
| **ACI**                           | Adjacent-Channel Interference.                                                               |

### Simplified channel-width relationship

```text
20 MHz
[====]

40 MHz
[========]

80 MHz
[================]

160 MHz
[================================]
```

Larger width:

```text
More spectrum consumed
        ↓
Potentially higher peak throughput
        ↓
Fewer independent channel blocks
```

### Interference comparison

| Situation            | Example                        | Main problem                   |
| -------------------- | ------------------------------ | ------------------------------ |
| Same channel         | AP-A ch. 6 + AP-B ch. 6        | Airtime contention             |
| Adjacent/overlapping | AP-A ch. 6 + AP-B ch. 7        | Spectral interference          |
| Non-WiFi             | WiFi + interfering RF source   | Noise/interference             |
| Distant reuse        | AP-A ch. 1 far from AP-B ch. 1 | Can be efficient spatial reuse |

# Section 4 — Tools and commands

| Tool          | Command                     | What it finds/shows                       | When to use it                         |
| ------------- | --------------------------- | ----------------------------------------- | -------------------------------------- |
| `nmcli`       | `nmcli device wifi list`    | SSID, channel, frequency, signal          | Quick channel survey                   |
| `iw`          | `sudo iw dev wlan0 scan`    | Detailed AP/channel/frequency information | Detailed RF inventory                  |
| `iw`          | `iw dev wlan0 link`         | Current AP/frequency/signal               | Inspect active connection              |
| `wavemon`     | `sudo wavemon`              | Signal, noise, quality and live link data | Observe RF conditions                  |
| `iw`          | `iw list`                   | Supported channel widths/frequencies      | Understand adapter capabilities        |
| `airmon-ng`   | `sudo airmon-ng`            | Wireless adapter/PHY information          | Prepare authorized wireless assessment |
| `airodump-ng` | `sudo airodump-ng wlan0mon` | APs, channels, signal levels, clients     | Channel/environment mapping            |

### `nmcli`

```text
$ nmcli device wifi list

SSID       CHAN  FREQ      SIGNAL
AP-A          1  2412 MHz    82
AP-B          6  2437 MHz    74
AP-C         11  2462 MHz    61
AP-D         36  5180 MHz    80
```

Interpretation:

```text
AP-A → 2.4 GHz / ch 1
AP-B → 2.4 GHz / ch 6
AP-C → 2.4 GHz / ch 11
AP-D → 5 GHz / ch 36
```

This gives the initial channel map.

---

### `iw dev wlan0 scan`

```text
$ sudo iw dev wlan0 scan

BSS aa:bb:cc:11:22:33
    SSID: AP-A
    freq: 2437
    signal: -48.00 dBm

BSS dd:ee:ff:44:55:66
    SSID: AP-B
    freq: 2462
    signal: -62.00 dBm
```

Interpretation:

```text
2437 MHz → Channel 6
2462 MHz → Channel 11
```

You can then determine whether their channel widths and physical locations create meaningful overlap.

---

### `iw dev wlan0 link`

```text
$ iw dev wlan0 link

Connected to aa:bb:cc:11:22:33
    SSID: AP-A
    freq: 2437
    signal: -48 dBm
```

This identifies the currently associated AP and operating frequency.

---

### `wavemon`

```text
$ sudo wavemon
```

A live display can show information such as:

```text
Signal level: -48 dBm
Noise level:  -91 dBm
Link quality: 62/70
```

Approximate SNR:

```text
-48 - (-91) = 43 dB
```

A strong RSSI combined with poor SNR can indicate significant RF noise/interference.

---

### `iw list`

```text
$ iw list

Band 2:
    Frequencies:
        * 5180 MHz [36]
        * 5200 MHz [40]

    VHT Capabilities:
        Supported Channel Width: 80 MHz
```

This tells you that the adapter supports an 80 MHz channel width in this example.

---

### `airmon-ng`

```text
$ sudo airmon-ng

PHY     Interface   Driver
phy0    wlan0       mt7921e
```

This identifies the wireless PHY and adapter driver.

---

### `airodump-ng`

```text
$ sudo airodump-ng wlan0mon

BSSID              PWR  CH  ENC   CIPHER  AUTH  ESSID
AA:BB:CC:11:22:33  -42   6  WPA2  CCMP    PSK   AP-A
DD:EE:FF:44:55:66  -51   7  WPA2  CCMP    PSK   AP-B
```

This is an important finding:

```text
AP-A → channel 6
AP-B → channel 7
```

The channels are close enough that their occupied spectrum may overlap, depending on channel width and actual RF characteristics.

# Section 5 — Defender detection

* **AP/controller telemetry:** Monitor channel utilization, retries, packet errors, client SNR, and channel changes.
* **RF sensors:** Dedicated sensors can identify neighboring transmitters and non-WiFi interference that normal IP monitoring cannot see.
* **Co-channel congestion:** Multiple legitimate APs sharing a channel should be evaluated through airtime utilization rather than simply counting APs.
* **Adjacent-channel problems:** Strong neighboring APs operating on overlapping frequencies can produce elevated retries and poor performance.
* **Common miss:** Defenders often increase transmit power when clients experience poor performance, which can make channel contention and spatial-reuse problems worse.
* **Channel-width monitoring:** Unexpected 80/160 MHz operation in a dense environment can consume spectrum that could otherwise support additional independent channels.
* **Operator footprint reduction:** Passive channel observation is less intrusive than deliberately generating interference or forcing clients to change channels; unnecessary RF disruption creates obvious operational symptoms.

# Section 6 — Lab task

**Platform:** Local Kali Linux + two or more WiFi APs that you control. Configure them on 2.4 GHz using 20 MHz channels.

**Objective:** Demonstrate the difference between clean channel separation, co-channel contention, and overlapping adjacent-channel operation.

**Steps:**

1. Configure AP-A on 2.4 GHz channel 1 with 20 MHz width.
2. Configure AP-B on channel 6 with 20 MHz width.
3. Place both APs within normal RF range of your Kali adapter.
4. Record their channels, frequencies, signal levels, and channel widths.
5. Reconfigure AP-B to channel 3 and repeat the measurements.
6. Compare the RF environment and link behavior between the two configurations.
7. Configure both APs on channel 6 and observe the difference between same-channel operation and the previous separated-channel configuration.
8. Use `wavemon` and/or throughput measurements to compare signal/noise and performance.
9. Record which configuration demonstrates channel separation, adjacent-channel overlap, and co-channel contention.
10. Save the results as a Markdown channel-planning report.

**Expected output:**

```text
Configuration A:
AP-A → Ch 1 / 20 MHz
AP-B → Ch 6 / 20 MHz
→ Well separated

Configuration B:
AP-A → Ch 1 / 20 MHz
AP-B → Ch 3 / 20 MHz
→ Overlapping channels

Configuration C:
AP-A → Ch 6 / 20 MHz
AP-B → Ch 6 / 20 MHz
→ Co-channel operation / airtime contention
```

The important result is not simply which configuration gives the highest throughput.

You should be able to explain **why the RF behavior differs**.

**Git artifact:**

```text
channel-planning/
├── README.md
├── notes/
│   └── channel-overlap-lab.md
└── evidence/
    └── measurements.csv
```

```bash
git add channel-planning/
git commit -m "Add WiFi channel overlap lab"
```

# Section 7 — Common mistakes

1. **Thinking neighboring channel numbers are automatically independent** → channel numbers are center frequencies, while transmissions occupy bandwidth → consider channel width and occupied spectrum.

2. **Treating same-channel APs as automatically worse than adjacent-channel APs** → same-channel networks can coordinate through WiFi's contention mechanisms, while overlapping adjacent channels can cause harmful interference → distinguish CCI from ACI.

3. **Using 1/6/11 as a universal rule for every WiFi deployment** → that pattern applies to common 2.4 GHz 20 MHz planning, not every band, width, or regulatory domain → choose channels based on the actual environment.

4. **Assuming wider channels are always better** → 80/160 MHz can increase peak throughput but reduce the number of independent channel blocks → balance throughput against spatial reuse and density.

5. **Counting APs instead of measuring RF conditions** → ten weak distant APs can be less significant than one extremely strong overlapping AP → consider signal strength, width, location, and utilization.

6. **Confusing congestion with interference** → contention for airtime and unwanted RF energy are different problems → inspect utilization, SNR, retries, and spectrum conditions.

7. **Ignoring 6 GHz because the channel plan looks unfamiliar** → 6 GHz has a different channel structure and newer client requirements → analyze it separately from 2.4 and 5 GHz.

# Section 8 — Move-on gate

1. **Scan a real or lab RF environment and produce a channel map containing SSID, BSSID, band, channel, channel width, and signal strength for at least five APs without looking at your notes.**

2. **Given APs operating on channels 1, 3, 6, and 11 at 20 MHz, identify which pairs are likely to overlap and distinguish adjacent-channel interference from same-channel contention without looking at your notes.**

3. **Configure two lab APs first on separated channels and then on the same/overlapping channels, measure signal/noise and throughput in both configurations, and correctly explain the observed performance difference without looking at your notes.**
