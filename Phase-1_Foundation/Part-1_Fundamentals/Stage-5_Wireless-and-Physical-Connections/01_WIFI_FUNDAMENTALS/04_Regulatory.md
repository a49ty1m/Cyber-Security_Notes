# Wireless Regulatory & RF Restrictions

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Regulatory → FCC/ETSI, DFS & Regional Channel Restrictions

# Section 1 — What it is and where it sits

Wireless regulatory knowledge defines **which frequencies, channels, transmit powers, channel widths, and operating behaviors a WiFi device may legally use in a particular region**. The important point for a security practitioner is that WiFi hardware does not have unlimited freedom over the RF spectrum. Its permitted behavior depends on the regulatory domain and the rules applicable to the country or region.

This matters during wireless assessment because the same adapter can expose different channels or capabilities depending on its configured regulatory domain. **DFS (Dynamic Frequency Selection)** is especially important in 5 GHz because certain channels share spectrum with systems such as weather and aviation radar.

```text
WiFi Fundamentals
      ↓
Bands / Channels / Width
      ↓
Regional Regulatory Domain
      ↓
FCC / ETSI / Local Rules
      ↓
Allowed Channels + Power + Width
      ↓
DFS Requirements
      ↓
Wireless Assessment
```

If you skip this, you can misinterpret why a channel is unavailable, incorrectly configure an adapter, misunderstand DFS behavior, or assume that a channel permitted in one country is automatically permitted everywhere.

This follows your understanding of channels and frequency bands and provides the regulatory layer needed before doing serious RF-level wireless assessment.

# Section 2 — How attackers actually use this

## 2.1 Determine the regulatory environment

Before assessing a wireless environment, an operator establishes the relevant jurisdiction.

They care about:

* country/region
* regulatory domain
* permitted frequency ranges
* permitted channels
* maximum transmit power
* indoor/outdoor restrictions
* channel-width restrictions
* DFS requirements
* client/AP compatibility

The same 5 GHz AP might expose different channels depending on its regulatory environment.

Conceptually:

```text
Same hardware
      │
      ├── Regulatory Domain A
      │      └── Channel set A
      │
      └── Regulatory Domain B
             └── Channel set B
```

Therefore:

```text
"I don't see channel X"
```

does not necessarily mean:

```text
"The hardware cannot support channel X."
```

The channel may simply not be permitted or enabled under the current regulatory configuration.

---

## 2.2 FCC vs ETSI

**FCC** is the United States regulatory framework administered by the Federal Communications Commission.

**ETSI** is a European standards organization, and European radio requirements are implemented through the applicable European regulatory framework and national arrangements.

For practical WiFi work, you will commonly encounter regulatory-domain identifiers associated with regions such as:

```text
US
EU
IN
JP
```

The exact permitted channels and power levels depend on the applicable local rules and device certification.

An attacker does not assume:

```text
US rules = Europe rules = India rules
```

They verify the environment.

---

## 2.3 Identify channel restrictions

A channel number by itself is not enough.

The actual question is:

```text
Is this channel permitted for this device
in this regulatory domain
under these operating conditions?
```

For example:

```text
5 GHz
│
├── Lower channels
│
├── DFS channels
│
└── Higher channels
```

Different regions can have different rules for these groups.

A wireless adapter may therefore report:

```text
Channel 36 → available
Channel 52 → DFS
Channel 100 → DFS
```

while another regulatory configuration may expose a different set.

---

## 2.4 Understand DFS

DFS stands for **Dynamic Frequency Selection**.

It exists because some 5 GHz frequencies are shared with systems that have priority, including radar systems.

A DFS-capable WiFi device must be able to detect radar activity and respond appropriately.

Conceptually:

```text
AP
 │
 │ operating on DFS channel
 ▼
Monitor RF environment
 │
 ├── No radar detected
 │       ↓
 │   Continue operation
 │
 └── Radar detected
         ↓
    Vacate channel
         ↓
    Select another permitted channel
```

This is not merely a WiFi security feature.

It is a **spectrum coexistence requirement**.

---

## 2.5 What happens when an AP uses DFS

A DFS-capable AP may need to perform a **Channel Availability Check (CAC)** before beginning normal transmission on certain DFS channels.

Conceptually:

```text
AP selects DFS channel
        ↓
      CAC
        ↓
Listen for radar
        ↓
No radar?
   ┌────┴────┐
  Yes        No
   ↓          ↓
Transmit    Change channel
```

This can create an important observation during assessment:

```text
"Why isn't the AP immediately transmitting on this channel?"
```

One possible reason is DFS-related channel availability checking.

---

## 2.6 DFS affects scanning

DFS can make wireless reconnaissance less straightforward.

An AP may:

* delay transmission
* move channels
* disappear temporarily
* change operating channel after radar detection
* behave differently from a non-DFS AP

Therefore, a scan that produces:

```text
Channel 36 → AP visible
Channel 100 → nothing visible
```

does not prove that channel 100 has no relevant wireless activity.

The observation could be affected by:

* regulatory configuration
* DFS behavior
* adapter support
* scanning method
* driver behavior
* AP operating state

This is why experienced operators understand the RF conditions before drawing conclusions.

---

## 2.7 Dead-end finding vs high-value finding

### Dead-end finding

```text
Adapter:
5 GHz supported

Scan:
No networks on channel 100
```

That is not enough information to conclude:

```text
"No channel-100 networks exist."
```

The adapter may not be scanning DFS channels correctly, or the AP may not currently be transmitting there.

### High-value finding

```text
Regulatory domain: appropriate local domain
Adapter: DFS-capable
Channel: 100
AP: observed
DFS behavior: channel changes after radar simulation in controlled lab
```

This establishes an actual understanding of how regulatory restrictions and DFS behavior affect the wireless environment.

---

## 2.8 Understand regional channel planning

A useful mental model is:

```text
Region
  ↓
Allowed frequency ranges
  ↓
Allowed channels
  ↓
Channel width limits
  ↓
Transmit power limits
  ↓
Indoor/outdoor requirements
  ↓
DFS requirements
```

For example, an enterprise deployment might have:

```text
Office AP
   ↓
Country configuration
   ↓
5 GHz channel plan
   ↓
DFS-capable channels available
   ↓
Controller selects channels automatically
```

Changing the country configuration is therefore not simply a harmless software preference.

It can change what RF behavior the device is permitted to perform.

---

## 2.9 How this becomes useful during an assessment

Suppose a wireless assessment reports:

```text
Target AP
Channel: 100
Frequency: 5500 MHz
```

The operator should immediately ask:

```text
Is channel 100 permitted here?
Is it DFS?
Is my adapter capable of operating on it?
Does my driver expose it?
Does the AP have DFS requirements?
```

This prevents incorrect conclusions.

The pivot is:

```text
Observed RF channel
      ↓
Regulatory classification
      ↓
DFS / non-DFS
      ↓
Adapter capability
      ↓
AP behavior
      ↓
Accurate wireless assessment
```

# Section 3 — Core concepts and terminology

| Term                           | Meaning                                                                                                                                   |
| ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **FCC**                        | U.S. Federal Communications Commission, which regulates radio use in the United States.                                                   |
| **ETSI**                       | European Telecommunications Standards Institute, which develops European telecommunications standards including relevant radio standards. |
| **Regulatory domain**          | Region-specific rules governing how wireless hardware may operate.                                                                        |
| **DFS**                        | Dynamic Frequency Selection; mechanism for detecting radar and vacating affected channels.                                                |
| **CAC**                        | Channel Availability Check performed before operation on applicable DFS channels.                                                         |
| **Radar detection**            | Detection of signals from protected radar systems sharing spectrum.                                                                       |
| **TPC**                        | Transmit Power Control; mechanism used in applicable bands to control transmission power.                                                 |
| **Channel restriction**        | Rule limiting which channels may be used in a particular region or situation.                                                             |
| **Transmit power**             | RF power used by a transmitter; subject to regulatory limits.                                                                             |
| **EIRP**                       | Equivalent Isotropically Radiated Power; effective radiated power accounting for transmitter output and antenna gain/loss.                |
| **Indoor restriction**         | Requirement limiting certain frequencies or power levels to indoor operation.                                                             |
| **Outdoor restriction**        | Rules governing operation outside buildings.                                                                                              |
| **Non-DFS channel**            | Channel that does not require DFS operation under the applicable rules.                                                                   |
| **DFS channel**                | Channel subject to radar-detection and channel-vacation requirements.                                                                     |
| **CAC**                        | Pre-transmission listening period required on applicable DFS channels.                                                                    |
| **Channel Availability Check** | Full form of CAC.                                                                                                                         |
| **Radar event**                | Detection of a signal interpreted as protected radar activity.                                                                            |
| **Channel move**               | AP changing away from a DFS channel after a radar event.                                                                                  |

### Simplified regulatory comparison

| Property                       | FCC/US context                                 | ETSI/European context                                   |
| ------------------------------ | ---------------------------------------------- | ------------------------------------------------------- |
| Regulatory authority/framework | FCC + applicable U.S. rules                    | European regulatory framework + national implementation |
| Channel availability           | Region-specific                                | Region-specific                                         |
| DFS                            | Required on applicable DFS channels            | Required on applicable DFS channels                     |
| Power limits                   | Region-specific                                | Region-specific                                         |
| Indoor/outdoor rules           | Frequency-dependent                            | Frequency-dependent                                     |
| Exact channel plan             | Must verify current rules/device certification | Must verify current rules/device certification          |

The important lesson is:

> **FCC and ETSI are not interchangeable "WiFi standards." They are part of the regulatory/standards ecosystem governing radio operation.**

# Section 4 — Tools and commands

| Tool        | Command                  | What it finds/shows                           | When to use it                          |
| ----------- | ------------------------ | --------------------------------------------- | --------------------------------------- |
| `iw`        | `iw reg get`             | Current regulatory domain                     | Determine regional RF restrictions      |
| `iw`        | `iw list`                | Supported bands/channels and regulatory flags | Determine adapter capabilities          |
| `iw`        | `iw phy phy0 channels`   | Channel/frequency information                 | Inspect PHY channel support             |
| `iw`        | `sudo iw dev wlan0 scan` | AP frequencies/channels                       | Observe actual RF environment           |
| `nmcli`     | `nmcli device wifi list` | Nearby networks and channels                  | Quick channel survey                    |
| `rfkill`    | `rfkill list`            | Radio blocking state                          | Troubleshoot disabled wireless hardware |
| `airmon-ng` | `sudo airmon-ng`         | Wireless PHY/interface information            | Identify wireless assessment hardware   |

### `iw reg get`

```text
$ iw reg get

global
country IN:
    (2402 - 2482 @ ...)
    (5150 - 5350 @ ...)
```

The important observation is:

```text
country IN
```

The regulatory configuration is currently associated with India.

The exact channel and power permissions should then be interpreted from the device/driver and applicable rules rather than assuming every possible WiFi channel is available.

---

### `iw list`

```text
$ iw list

Wiphy phy0
    Band 2:
        Frequencies:
            * 5180 MHz [36]
            * 5200 MHz [40]
            * 5260 MHz [52] (radar detection)
            * 5280 MHz [56] (radar detection)
```

The important part is:

```text
5260 MHz [52] (radar detection)
```

That indicates channel 52 is treated as a DFS-related channel by the wireless stack.

---

### `iw phy phy0 channels`

```text
$ iw phy phy0 channels

Band 2:
    * 5180 MHz [36]
    * 5200 MHz [40]
    * 5260 MHz [52] (radar detection)
    * 5280 MHz [56] (radar detection)
```

This is useful when you need a more direct view of the PHY's channel information.

---

### `iw dev wlan0 scan`

```text
$ sudo iw dev wlan0 scan

BSS aa:bb:cc:11:22:33
    SSID: Lab-5G
    freq: 5500
    signal: -48.00 dBm
```

5500 MHz corresponds to:

```text
5 GHz band
Channel 100
DFS-related channel
```

The scan tells you what was observed; regulatory interpretation tells you why that frequency has particular operating requirements.

---

### `nmcli`

```text
$ nmcli device wifi list

SSID       CHAN  FREQ      SIGNAL
Lab-2G       6   2437 MHz    75
Lab-5G      36   5180 MHz    82
Lab-DFS    100   5500 MHz    61
```

This gives a quick RF inventory:

```text
Lab-2G  → 2.4 GHz
Lab-5G  → 5 GHz
Lab-DFS → 5 GHz / DFS channel
```

---

### `rfkill`

```text
$ rfkill list

0: phy0: Wireless LAN
    Soft blocked: no
    Hard blocked: no
```

This confirms that the adapter is not being blocked by the operating system or hardware switch.

---

### `airmon-ng`

```text
$ sudo airmon-ng

PHY     Interface   Driver
phy0    wlan0       mt7921e
```

This identifies the PHY and driver used by the adapter.

That matters because DFS support is not simply a property of "WiFi." It depends on the regulatory domain, chipset, firmware, driver, and operating mode.

# Section 5 — Defender detection

* **Controller/AP telemetry:** Enterprise controllers record channel changes, DFS events, radar detections, and AP radio state changes.
* **DFS events:** Unexpected repeated channel moves or radar-detection events can be investigated against known RF conditions.
* **RF sensors:** Dedicated wireless sensors can detect unauthorized transmitters operating on restricted or unexpected frequencies.
* **Configuration monitoring:** Compare AP regulatory configuration against the organization's physical location and approved deployment configuration.
* **Common miss:** Defenders may treat frequent DFS channel changes as ordinary WiFi instability without checking whether radar events or configuration problems explain the behavior.
* **Operator footprint reduction:** RF activity that deliberately interferes with or manipulates wireless operation is considerably more observable than passive spectrum observation; skilled operators avoid unnecessary transmission.
* **Important distinction:** A channel appearing in a device's capability list does not automatically mean unrestricted operation is permitted in every country or operating condition.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM + compatible USB WiFi adapter + your own AP; use a non-production environment and do not intentionally interfere with real radar or protected radio services.

**Objective:** Identify your adapter's regulatory domain, enumerate its available 5 GHz channels, and correctly distinguish DFS from non-DFS channels.

**Steps:**

1. Connect the WiFi adapter to Kali and verify the wireless interface.
2. Check the current regulatory domain.
3. Enumerate the adapter's supported 5 GHz frequencies.
4. Identify channels marked for radar detection/DFS.
5. Configure your lab AP using a permitted non-DFS 5 GHz channel.
6. Scan the environment and record the AP's frequency and channel.
7. If your hardware/AP supports a DFS lab configuration, observe its normal DFS behavior without attempting to simulate real radar.
8. Compare the AP's observed channel with the adapter's regulatory channel list.
9. Record which channels are non-DFS and which require DFS handling.
10. Save the results as a Markdown regulatory inventory.

**Expected output:**

```text
Regulatory domain: IN

5 GHz:
Channel 36 → 5180 MHz → Non-DFS
Channel 40 → 5200 MHz → Non-DFS
Channel 52 → 5260 MHz → DFS
Channel 56 → 5280 MHz → DFS
Channel 100 → 5500 MHz → DFS
```

The exact list depends on your hardware, driver, regulatory configuration, and current rules. Success means you can explain **why** a particular channel is DFS-restricted rather than merely memorizing channel numbers.

**Git artifact:**

```text
wireless-regulatory/
├── README.md
├── notes/
│   └── regulatory-domain.md
└── evidence/
    └── iw-channel-output.txt
```

```bash
git add wireless-regulatory/
git commit -m "Add WiFi regulatory and DFS lab"
```

# Section 7 — Common mistakes

1. **Treating FCC/ETSI as WiFi protocols** → they belong to the regulatory/standards ecosystem rather than being authentication or encryption protocols → separate regulatory rules from 802.11 functionality.

2. **Assuming the same channels are legal everywhere** → channel availability and operating conditions vary by region → check the applicable regulatory domain.

3. **Thinking DFS means "secret WiFi channels"** → DFS exists primarily for spectrum coexistence with protected users such as radar → understand the reason behind the restriction.

4. **Assuming every 5 GHz channel is DFS** → many 5 GHz channels are non-DFS → identify the specific channel and its regulatory status.

5. **Changing the regulatory domain just to unlock channels** → software visibility does not override legal/device certification requirements → operate according to the actual jurisdiction and certified hardware configuration.

6. **Assuming missing DFS networks don't exist** → scanning support can depend on drivers, firmware, hardware, and DFS behavior → distinguish "not observed" from "does not exist."

7. **Treating signal/channel behavior as purely technical** → regulatory restrictions can directly explain why an AP changes channel, disappears, or refuses a configuration → include regulatory context when interpreting RF observations.

# Section 8 — Move-on gate

1. **Run `iw reg get` and `iw list` on your Kali adapter and identify the active regulatory domain plus at least three 5 GHz channels, correctly marking which are DFS-related without looking at your notes.**

2. **Scan your lab AP and, from its reported frequency/channel, correctly determine its band and whether DFS requirements are relevant to that channel without looking at your notes.**

3. **Given three wireless configurations from different regions, identify which channel assignments require additional regulatory/DFS consideration and explain exactly what information you would verify before operating the hardware.**
