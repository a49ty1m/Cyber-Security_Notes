# Wireless Monitoring Tools

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → RF Survey Fundamentals → Kismet, iw & iwconfig

# Section 1 — What it is and where it sits

Wireless monitoring tools let you observe the **802.11 environment at the radio/interface level** rather than only looking at normal IP traffic. `iw` is the modern Linux wireless configuration and inspection utility, `iwconfig` is the older Wireless Extensions interface, and **Kismet** is a wireless discovery and monitoring platform that can continuously collect and correlate information about APs, clients, channels, signal levels, and other wireless activity.

For offensive security, these tools produce different levels of visibility:

```text
WiFi Fundamentals
      ↓
Bands / Channels / RF
      ↓
Wireless Monitoring
      ↓
┌─────────────┬─────────────┬──────────────┐
│     iw      │  iwconfig   │   Kismet     │
│ Linux PHY   │ Legacy info │ Discovery /  │
│ inspection  │             │ monitoring   │
└─────────────┴─────────────┴──────────────┘
      ↓
Wireless Reconnaissance
      ↓
Authentication / Client Analysis
      ↓
Wireless Security Assessment
```

If you skip this, you become dependent on one tool's output and may not understand whether a problem comes from the adapter, driver, RF environment, or wireless protocol. The previous topics taught you **what** channels, bands, antennas, and authentication mean; these tools teach you how to **observe those things in practice**.

# Section 2 — How attackers actually use this

## 2.1 Start with the wireless interface

An operator first determines what wireless hardware Linux can actually see.

They want:

* interface name
* PHY/radio
* MAC address
* driver
* interface mode
* current channel
* supported bands
* monitor-mode capability
* regulatory configuration

The workflow is:

```text
Physical WiFi Adapter
        ↓
Linux Driver
        ↓
PHY
        ↓
Wireless Interface
        ↓
iw / iwconfig
        ↓
Kismet
        ↓
802.11 Environment Map
```

A common mistake is immediately launching a monitoring tool without first verifying that Linux actually recognizes the adapter.

---

## 2.2 Use `iw` to understand the modern Linux wireless stack

`iw` is the primary modern command-line utility for interacting with Linux's `nl80211` wireless subsystem.

An operator uses it to answer questions such as:

```text
What wireless interfaces exist?
What PHY do they belong to?
What channel am I using?
What frequency am I using?
What bands does this adapter support?
What channels are available?
What is my regulatory domain?
What AP am I currently associated with?
```

For example:

```text
phy0
 └── wlan0
      ├── managed mode
      ├── 5 GHz
      └── channel 36
```

This is foundational information before interpreting anything Kismet reports.

---

## 2.3 Use `iwconfig` to recognize legacy wireless information

`iwconfig` is older.

It exposes information through Linux's legacy **Wireless Extensions** interface.

You may encounter output like:

```text
wlan0
    IEEE 802.11
    Mode:Managed
    ESSID:"Lab-WiFi"
    Frequency:5.18 GHz
    Link Quality=62/70
    Signal level=-48 dBm
```

The important lesson is not to memorize `iwconfig` syntax.

It is to recognize that:

```text
iw
  ↓
Modern Linux wireless management

iwconfig
  ↓
Legacy wireless interface
```

For modern Linux wireless work, prefer `iw` where possible.

You should still understand `iwconfig` because it appears frequently in older tutorials, scripts, labs, and documentation.

---

## 2.4 Kismet provides the bigger picture

`iw` is excellent for interrogating your local wireless interface.

Kismet is designed to monitor and correlate the **wireless environment**.

Conceptually:

```text
               Kismet
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
      APs       Clients      RF data
       │          │          │
       └──────────┼──────────┘
                  ↓
            Wireless map
```

An operator can use Kismet to discover:

* SSIDs
* BSSIDs
* client devices
* channels
* frequencies
* signal levels
* AP/client relationships
* wireless vendors
* observed network activity
* channel changes
* device appearance/disappearance

The major advantage is **continuous observation and correlation** rather than manually running one command at a time.

---

## 2.5 Understand passive monitoring

A key concept is passive monitoring.

A passive observer listens to wireless transmissions without needing to associate with the target AP.

Conceptually:

```text
AP ───────────────►
       RF frames
           │
           ▼
      Monitor radio
           │
           ▼
         Kismet
```

This can reveal information contained in transmitted 802.11 management and other observable frames.

The important distinction is:

```text
Passive:
Listen and analyze

Active:
Transmit / associate / authenticate / interact
```

For reconnaissance, passive observation can provide a large amount of useful information with less interaction.

---

## 2.6 Identify APs and BSSIDs

Suppose Kismet reports:

```text
SSID: Corp-WiFi
BSSID: AA:BB:CC:11:22:33
Channel: 36
Signal: -43 dBm
```

The operator can now map:

```text
Corp-WiFi
   ↓
AA:BB:CC:11:22:33
   ↓
Channel 36
   ↓
5 GHz
   ↓
Strong signal
```

This becomes useful when several APs advertise the same SSID.

For example:

```text
Corp-WiFi
├── BSSID A → Channel 36
├── BSSID B → Channel 44
└── BSSID C → Channel 149
```

The SSID alone is insufficient for infrastructure mapping.

---

## 2.7 Identify clients

Wireless monitoring can also expose client devices.

A simplified view:

```text
AP: Corp-WiFi
      │
      ├── Client A
      ├── Client B
      └── Client C
```

An operator cares about:

* client MAC addresses
* associated AP
* signal strength
* observed channels
* vendor information
* activity over time

Client mapping becomes particularly useful later when analyzing authentication behavior and wireless infrastructure.

---

## 2.8 Track changes rather than snapshots

A single scan is a snapshot.

A monitoring platform can reveal changes:

```text
12:00 → AP visible
12:05 → Client appears
12:07 → Client moves AP
12:10 → AP changes channel
12:12 → Client disappears
```

This temporal information can reveal:

* roaming
* channel changes
* devices entering/leaving range
* AP availability
* RF instability

This is one of Kismet's major advantages over one-off interface commands.

---

## 2.9 Dead-end finding vs high-value finding

### Dead-end finding

```text
SSID: Guest-WiFi
Signal: -87 dBm
```

You have discovered a network, but:

* it may be unrelated
* signal is extremely weak
* no client relationship is known
* no useful infrastructure context exists

This is a low-value observation.

### High-value finding

```text
SSID: Corp-WiFi
BSSID: AA:BB:CC:11:22:33
Channel: 36
Signal: -43 dBm

Clients:
  11:22:33:44:55:66
  22:33:44:55:66:77

Security:
  WPA2-Enterprise
```

Now the operator has:

```text
Target WLAN
    ↓
Specific AP
    ↓
Strong RF position
    ↓
Known clients
    ↓
Enterprise authentication
    ↓
Potential next assessment stage
```

The value comes from **correlation**, not merely finding an SSID.

---

## 2.10 Use the tools together

A strong workflow does not treat Kismet, `iw`, and `iwconfig` as competing tools.

Use them for different purposes:

```text
iw
 ↓
"What is my adapter doing?"

iwconfig
 ↓
"What does this legacy interface report?"

Kismet
 ↓
"What is happening around me?"
```

Example:

```text
iw dev
   ↓
wlan0 exists

iw dev wlan0 info
   ↓
5 GHz / channel 36

Kismet
   ↓
Corp-WiFi / BSSID / clients / RF activity

iwconfig
   ↓
Legacy compatibility check
```

This triangulation prevents a single tool's output from becoming your entire understanding of the environment.

---

## 2.11 Pivot into the next phase

The monitoring results feed directly into wireless assessment:

```text
Kismet / iw
      ↓
AP discovery
      ↓
BSSID mapping
      ↓
Channel mapping
      ↓
Client discovery
      ↓
Signal / RF analysis
      ↓
Authentication identification
      ↓
Wireless security assessment
```

For example:

```text
Kismet:
Corp-WiFi
   ↓
BSSID discovered
   ↓
Client observed
   ↓
WPA2-Enterprise
   ↓
EAP analysis
   ↓
RADIUS architecture
```

The tool is not the objective.

**The objective is turning observations into an accurate wireless attack-surface map.**

# Section 3 — Core concepts and terminology

| Term                   | Meaning                                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------------------- |
| **Kismet**             | Wireless discovery, monitoring, and network detection platform.                                           |
| **iw**                 | Modern Linux command-line utility for configuring and inspecting wireless devices.                        |
| **iwconfig**           | Legacy Linux wireless configuration/inspection utility.                                                   |
| **PHY**                | Physical wireless radio represented by Linux.                                                             |
| **Wireless interface** | Linux network interface associated with a wireless PHY.                                                   |
| **Managed mode**       | Normal client mode where the adapter connects to an AP.                                                   |
| **Monitor mode**       | Mode allowing a wireless adapter to receive 802.11 frames without normal client association requirements. |
| **nl80211**            | Modern Linux kernel/userspace wireless interface.                                                         |
| **802.11 frame**       | Protocol frame used by WiFi devices.                                                                      |
| **Beacon**             | Management frame periodically advertising an AP/BSS.                                                      |
| **BSSID**              | Identifier for a specific wireless BSS, commonly a MAC address.                                           |
| **SSID**               | Human-readable WLAN name.                                                                                 |
| **Client/STA**         | Wireless station communicating with an AP.                                                                |
| **RF survey**          | Systematic observation of wireless signals and RF conditions.                                             |
| **Channel**            | Specific frequency allocation used for WiFi communication.                                                |
| **RSSI**               | Receiver-reported signal strength indicator.                                                              |
| **SNR**                | Ratio of desired signal to noise.                                                                         |
| **Passive monitoring** | Observing wireless transmissions without actively interacting with the target.                            |
| **Active scanning**    | Sending requests or otherwise transmitting to discover/respond to wireless devices.                       |
| **Packet capture**     | Recording transmitted frames for later analysis.                                                          |
| **Channel hopping**    | Monitoring different channels over time to discover more wireless activity.                               |
| **WIDS**               | Wireless Intrusion Detection System.                                                                      |
| **WIPS**               | Wireless Intrusion Prevention System.                                                                     |

### Tool comparison

| Capability                         |      `iw` | `iwconfig` |    Kismet |
| ---------------------------------- | --------: | ---------: | --------: |
| Interface inspection               | Excellent |       Good |       Yes |
| Channel inspection                 | Excellent |      Basic |       Yes |
| PHY capabilities                   | Excellent |    Limited |  Indirect |
| Regulatory information             | Excellent |    Limited |  Indirect |
| Nearby AP discovery                |       Yes |    Limited | Excellent |
| Client discovery                   |   Limited |         No | Excellent |
| Continuous monitoring              |        No |         No |       Yes |
| Historical/correlated observations |        No |         No |       Yes |
| Modern Linux interface             |       Yes |     Legacy |       Yes |

# Section 4 — Tools and commands

| Tool       | Command                  | What it finds/shows                          | When to use it                        |
| ---------- | ------------------------ | -------------------------------------------- | ------------------------------------- |
| `iw`       | `iw dev`                 | Wireless interfaces and PHY relationships    | Start every wireless assessment       |
| `iw`       | `iw dev wlan0 info`      | Interface mode/channel/frequency             | Inspect one interface                 |
| `iw`       | `iw dev wlan0 link`      | Associated AP and signal                     | Inspect current connection            |
| `iw`       | `iw list`                | Adapter capabilities and supported channels  | Hardware capability assessment        |
| `iw`       | `iw reg get`             | Regulatory domain                            | Verify regional RF configuration      |
| `iwconfig` | `iwconfig`               | Legacy wireless interface information        | Read older tooling/tutorials          |
| `iwconfig` | `iwconfig wlan0`         | Mode, ESSID, frequency, signal               | Inspect a legacy-compatible interface |
| `Kismet`   | `sudo kismet`            | Starts wireless monitoring service/interface | Continuous wireless discovery         |
| `nmcli`    | `nmcli device wifi list` | Nearby APs and signal levels                 | Quick comparison against Kismet       |

### `iw dev`

```text
$ iw dev

phy#0
    Interface wlan0
        ifindex 3
        addr aa:bb:cc:dd:ee:ff
        type managed
```

Interpretation:

```text
phy0
  ↓
Physical wireless radio

wlan0
  ↓
Linux wireless interface

managed
  ↓
Normal client mode
```

This is the first thing to establish before troubleshooting wireless tools.

---

### `iw dev wlan0 info`

```text
$ iw dev wlan0 info

Interface wlan0
    type managed
    channel 36 (5180 MHz), width: 80 MHz
```

This immediately gives:

```text
Channel: 36
Frequency: 5180 MHz
Width: 80 MHz
```

Therefore the adapter is currently operating in the 5 GHz band.

---

### `iw dev wlan0 link`

```text
$ iw dev wlan0 link

Connected to aa:bb:cc:11:22:33
    SSID: Lab-WiFi
    freq: 5180
    signal: -47 dBm
```

Interpretation:

```text
AP:
AA:BB:CC:11:22:33

SSID:
Lab-WiFi

Signal:
-47 dBm
```

This describes the current client connection.

---

### `iw list`

```text
$ iw list

Wiphy phy0

Band 1:
    Frequencies:
        * 2412 MHz [1]
        * 2437 MHz [6]
        * 2462 MHz [11]

Band 2:
    Frequencies:
        * 5180 MHz [36]
        * 5200 MHz [40]
```

This tells you what the adapter reports as supported.

The important distinction is:

```text
iw list
→ Adapter capabilities

Kismet scan
→ Observed wireless environment
```

---

### `iw reg get`

```text
$ iw reg get

global
country IN:
```

This identifies the current regulatory-domain configuration.

That matters because channel availability and transmit behavior depend on the applicable regional rules.

---

### `iwconfig`

```text
$ iwconfig

wlan0
    IEEE 802.11
    Mode:Managed
    ESSID:"Lab-WiFi"
    Frequency:5.18 GHz
    Link Quality=60/70
    Signal level=-48 dBm
```

This is useful for recognizing output from older wireless tooling.

For current Linux systems:

```text
Prefer:
iw

Understand:
iwconfig
```

---

### Kismet

Start Kismet:

```text
$ sudo kismet
```

Depending on the installation, Kismet opens its web interface.

A discovered device might conceptually appear as:

```text
SSID:       Lab-WiFi
BSSID:      AA:BB:CC:11:22:33
Channel:    36
Frequency:  5180 MHz
Signal:     -47 dBm
```

Client observations may look like:

```text
Client:
11:22:33:44:55:66

Associated:
AA:BB:CC:11:22:33
```

Interpretation:

```text
AP
 ↓
AA:BB:CC:11:22:33
 ↓
Client
11:22:33:44:55:66
```

Kismet becomes particularly useful when many APs and clients must be correlated over time.

# Section 5 — Defender detection

* **Kismet-like monitoring:** Wireless IDS/monitoring systems can continuously detect APs, clients, channels, and unexpected wireless infrastructure.
* **Rogue AP detection:** Compare observed BSSIDs against an approved inventory to identify unauthorized wireless devices.
* **Client anomaly detection:** Unexpected clients, unusual roaming, or clients associated with unauthorized APs can indicate wireless security problems.
* **Channel monitoring:** Unexpected channel changes and unusual channel utilization can be correlated with RF events.
* **Common miss:** Defenders may inventory SSIDs but fail to inventory individual BSSIDs, making rogue infrastructure using a legitimate SSID harder to detect.
* **Passive vs active visibility:** Passive monitoring itself does not require normal network authentication, so network-layer logs may contain no corresponding client session.
* **Operator footprint reduction:** Passive observation minimizes transmission compared with active scanning or authentication attempts, reducing wireless events generated by the assessor.

# Section 6 — Lab task

**Platform:** Kali Linux + your own WiFi AP + compatible USB WiFi adapter.

**Objective:** Use `iw` and Kismet together to produce a basic wireless inventory and correctly correlate an AP, BSSID, channel, frequency, and client.

**Steps:**

1. Connect the WiFi adapter to Kali and verify that Linux detects it.
2. Use `iw` to identify the wireless interface and PHY.
3. Inspect the adapter's supported channels and bands.
4. Verify the current regulatory domain.
5. Start Kismet with the authorized lab adapter as its capture source.
6. Locate your lab AP in Kismet.
7. Record its SSID, BSSID, channel, frequency, and signal level.
8. Connect your second test device to the lab AP.
9. Confirm that Kismet observes the test client's wireless activity and associate it with the correct BSSID.
10. Compare the Kismet observations with `iw` output and save the inventory.

**Expected output:**

```text
PHY:       phy0
Interface: wlan0

AP:
SSID:      Lab-WiFi
BSSID:     AA:BB:CC:11:22:33
Channel:   36
Frequency: 5180 MHz
Signal:    -47 dBm

Client:
MAC:       11:22:33:44:55:66
Associated AP:
           AA:BB:CC:11:22:33
```

Success means the three tools no longer look like unrelated commands:

```text
iw
 ↓
Local adapter state

iwconfig
 ↓
Legacy representation

Kismet
 ↓
Wireless environment
```

**Git artifact:**

```text
wireless-tools/
├── README.md
├── notes/
│   └── kismet-iw-inventory.md
└── evidence/
    └── wireless-inventory.txt
```

```bash
git add wireless-tools/
git commit -m "Add wireless monitoring tools lab"
```

# Section 7 — Common mistakes

1. **Using `iwconfig` as the primary modern tool** → it is legacy technology and does not expose the full capabilities of modern Linux wireless stacks → use `iw` first and understand `iwconfig` for compatibility.

2. **Starting Kismet before checking the adapter** → driver, firmware, regulatory, or interface problems can make Kismet appear broken → verify `iw dev`, `iw list`, and radio state first.

3. **Confusing adapter capabilities with observed networks** → `iw list` describes your hardware while Kismet observes the environment → never interpret the two outputs as the same thing.

4. **Treating an SSID as a unique AP** → multiple BSSIDs can advertise one SSID → map BSSID, channel, and radio individually.

5. **Assuming one scan represents the entire environment** → clients move, APs change channels, and signals fluctuate → continuous monitoring can reveal information a snapshot misses.

6. **Ignoring signal and channel context** → finding a client or AP without frequency, channel, and signal information makes the finding difficult to interpret → record the RF context with every important observation.

7. **Thinking Kismet is automatically an attack tool** → its core value here is discovery, monitoring, and correlation → use the observations to determine the next assessment stage.

# Section 8 — Move-on gate

1. **Run `iw dev`, `iw list`, and `iw dev wlan0 link` on your Kali adapter and correctly explain the PHY, interface, mode, channel, frequency, and current AP without looking at your notes.**

2. **Run Kismet against your lab environment and identify one AP plus one associated test client, correctly correlating the SSID, BSSID, channel, frequency, and signal information without looking at your notes.**

3. **Given output from `iw`, `iwconfig`, and Kismet describing the same wireless environment, reconcile the three outputs and identify which information describes your local adapter versus the observed wireless environment without looking at your notes.**
