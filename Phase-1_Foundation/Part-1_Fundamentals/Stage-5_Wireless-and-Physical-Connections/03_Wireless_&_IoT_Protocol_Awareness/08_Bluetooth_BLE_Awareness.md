# Bluetooth/BLE Awareness

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Wireless & IoT Protocol Awareness → Bluetooth/BLE Awareness

# Section 1 — What it is and where it sits

Bluetooth is a short-range wireless technology, but **Bluetooth Classic and Bluetooth Low Energy (BLE) are not simply two versions of the same protocol stack**. They have different architectures, terminology, connection models, and security mechanisms.

**Bluetooth Classic** is commonly associated with continuous connections such as headphones, keyboards, mice, and serial-style communication. Important components include **profiles, L2CAP, and RFCOMM**. **BLE** was designed around low-power, short-burst communication and is heavily used by sensors, wearables, smart locks, medical/fitness devices, beacons, and IoT equipment. Its important concepts include **GAP, GATT, advertising, services, characteristics, and pairing**.

```text
Bluetooth
├── Bluetooth Classic
│   ├── Profiles
│   ├── L2CAP
│   └── RFCOMM
│
└── Bluetooth Low Energy (BLE)
    ├── GAP
    ├── Advertising
    ├── Pairing / Bonding
    └── GATT
        ├── Services
        └── Characteristics
```

This topic belongs here because wireless reconnaissance is not limited to Wi-Fi. An attacker may discover a Bluetooth device, identify its role, determine which stack it uses, and then decide whether Bluetooth is worth investigating.

You should already understand basic wireless monitoring; the next step is understanding **Bluetooth discovery and protocol-specific attack surfaces**. Full BLE exploitation belongs to **Part 21 Stage 4**, so this stage focuses on recognition and architecture rather than exploitation.

---

# Section 2 — How attackers actually use this

## 2.1 Identify Bluetooth technology

The first question is:

> **“What kind of Bluetooth device and stack am I looking at?”**

An attacker wants to determine:

* Bluetooth Classic or BLE?
* Device address/identity information
* Device name
* Device type
* Services or capabilities exposed
* Whether the device is advertising
* Whether it is discoverable/connectable
* Security/pairing characteristics
* Whether the device is an IoT peripheral, computer, phone, accessory, or sensor

This classification determines what protocol concepts matter next.

---

## 2.2 Bluetooth Classic reconnaissance

Bluetooth Classic commonly appears in devices requiring relatively continuous communication.

Examples:

```text
Laptop
   │
   └── Bluetooth
        │
        ├── Headset
        ├── Keyboard
        ├── Mouse
        └── Serial-style device
```

An attacker may encounter:

* **Profiles** describing standardized application behavior
* **L2CAP** transporting higher-level Bluetooth protocols
* **RFCOMM** providing serial-port-like communication

The important point is that discovering Bluetooth does **not** immediately mean there is an exploitable service.

The attacker first determines what the device actually exposes.

---

## 2.3 BLE reconnaissance

BLE is particularly important in IoT because devices often communicate through small pieces of structured data rather than maintaining a large continuous data stream.

A common architecture looks like:

```text
BLE Peripheral
      │
      ├── Advertising
      │
      └── GATT Server
            │
            ├── Service
            │    ├── Characteristic
            │    └── Characteristic
            │
            └── Service
                 └── Characteristic
```

An attacker wants to understand:

> **What information does this device expose, and what can a connected client interact with?**

For example, a smart sensor might expose:

```text
Device
└── Environmental Service
    ├── Temperature Characteristic
    └── Battery Characteristic
```

A fitness device could expose services containing heart-rate, battery, device-information, or proprietary characteristics.

The names alone are not enough. The interesting question is **what operations and data are associated with those characteristics**.

---

## 2.4 Advertising is an important discovery mechanism

BLE devices can transmit **advertising packets** without first establishing a traditional application connection.

An attacker can therefore learn about nearby devices from their advertisements.

Potential findings include:

* Device presence
* Advertised name
* Service identifiers
* Manufacturer-specific data
* Device type indicators
* Changing or static addresses
* Signal strength
* Advertisement frequency

This creates a useful distinction:

```text
Wi-Fi reconnaissance
        ↓
Discover APs / clients

BLE reconnaissance
        ↓
Discover advertisers / peripherals
```

---

## 2.5 Pairing is not the same as application security

Bluetooth pairing establishes cryptographic relationships between devices, but that does **not automatically make the application secure**.

An attacker may encounter a system where:

```text
Bluetooth encryption
        ↓
Strong

Application authorization
        ↓
Weak
```

For example, an application may trust a paired device too broadly or expose sensitive functionality without sufficiently checking authorization.

This is why a security assessment must consider both:

```text
Radio/security layer
        +
Application/service logic
```

---

## 2.6 GATT becomes the important BLE application boundary

For BLE, attackers eventually care about the **GATT database**.

The hierarchy is:

```text
GATT
│
├── Service
│     ├── Characteristic
│     │      └── Descriptor
│     │
│     └── Characteristic
│
└── Service
      └── Characteristic
```

A characteristic may represent:

* Sensor data
* Device configuration
* Status
* Commands
* Authentication state
* Control functions

A finding becomes much more valuable when a characteristic exposes **write/control functionality**, especially when access control is weak.

Full enumeration and exploitation of these interfaces are intentionally deferred to **Part 21 Stage 4**.

---

## 2.7 Dead-end vs high-value findings

| Finding                         | Typical value |
| ------------------------------- | ------------- |
| Bluetooth device exists         | Low           |
| Device name identified          | Low–Medium    |
| Classic profile identified      | Medium        |
| BLE advertising detected        | Medium        |
| Service identifiers discovered  | Medium        |
| GATT services identified        | High          |
| Readable sensitive data         | High          |
| Writable control characteristic | Very High     |
| Weak/incorrect authorization    | Very High     |
| Proprietary command interface   | Very High     |

The key lesson:

> **Device discovery is reconnaissance; exposed functionality is the attack surface.**

---

## 2.8 Where this feeds next

Bluetooth awareness lets you recognize when a wireless target deserves deeper investigation:

```text
Bluetooth discovery
       ↓
Classic vs BLE classification
       ↓
Services / profiles / GATT
       ↓
Security model
       ↓
Potential attack surface
       ↓
Part 21 Stage 4: BLE exploitation
```

---

# Section 3 — Core concepts and terminology

### Bluetooth Classic

Traditional Bluetooth stack optimized for applications requiring relatively continuous communication.

### Bluetooth Low Energy (BLE)

Bluetooth architecture designed for low-power communication and short, efficient exchanges.

### Profile

A Bluetooth specification defining how a particular application/service should operate.

### L2CAP

**Logical Link Control and Adaptation Protocol**; provides multiplexing and transport functions for higher Bluetooth protocols.

### RFCOMM

A Bluetooth protocol that provides serial-port-like communication over Bluetooth Classic.

### GAP

**Generic Access Profile**; defines device discovery, advertising, connection behavior, and related access procedures.

### GATT

**Generic Attribute Profile**; defines how BLE applications organize and exchange structured data.

### Advertising

BLE mechanism for broadcasting small amounts of information to nearby devices.

### Peripheral

A BLE device that generally provides functionality or data to a central device.

### Central

A BLE device that generally initiates connections to peripherals.

### Service

A logical collection of related BLE functionality.

### Characteristic

A GATT data or control endpoint associated with a service.

### Descriptor

Metadata associated with a characteristic.

### Pairing

Procedure used to establish security credentials between Bluetooth devices.

### Bonding

Persistent storage of security information so devices can recognize each other later.

### Authentication

Process of establishing confidence in the identity of the communicating device.

### Authorization

Determining what an authenticated/connected device is actually allowed to do.

### BSSID vs Bluetooth address

A Wi-Fi BSSID identifies a wireless LAN interface/AP. Bluetooth uses its own addressing and identity mechanisms; do not treat Bluetooth addresses as Wi-Fi BSSIDs.

### RSSI

Received Signal Strength Indicator; provides an indication of received radio signal strength.

---

## Bluetooth Classic vs BLE

| Feature                   | Bluetooth Classic                        | BLE                                        |
| ------------------------- | ---------------------------------------- | ------------------------------------------ |
| Primary design            | Continuous/general-purpose communication | Low-power communication                    |
| Important concepts        | Profiles, L2CAP, RFCOMM                  | GAP, GATT, advertising                     |
| Common devices            | Headsets, keyboards, mice                | Sensors, wearables, IoT                    |
| Discovery model           | Classic discovery mechanisms             | Advertising/scanning                       |
| Application model         | Profile-oriented                         | Service/characteristic-oriented            |
| Power characteristics     | Generally higher                         | Designed for low power                     |
| Security assessment focus | Profiles/services/connections            | Advertising/GATT/pairing/application logic |

---

# Section 4 — Tools and commands

| Tool           | Command                   | What it finds/shows                       | When to use it                         |
| -------------- | ------------------------- | ----------------------------------------- | -------------------------------------- |
| `bluetoothctl` | `bluetoothctl`            | Interactive Bluetooth discovery/control   | Basic Bluetooth reconnaissance         |
| `bluetoothctl` | `bluetoothctl show`       | Local Bluetooth controller information    | Identify local adapter                 |
| `bluetoothctl` | `bluetoothctl scan on`    | Nearby discoverable Bluetooth devices     | Discovery                              |
| `bluetoothctl` | `bluetoothctl devices`    | Discovered Bluetooth devices              | Review discovered devices              |
| `bluetoothctl` | `bluetoothctl info <MAC>` | Information about a discovered device     | Inspect a target                       |
| `btmgmt`       | `sudo btmgmt info`        | Bluetooth controller information          | Lower-level Linux Bluetooth inspection |
| `btmgmt`       | `sudo btmgmt power on`    | Enables Bluetooth controller              | Prepare adapter                        |
| `rfkill`       | `rfkill list`             | Whether Bluetooth hardware is blocked     | Troubleshoot adapter visibility        |
| `hciconfig`    | `hciconfig -a`            | Classic Bluetooth HCI adapter information | Legacy adapter inspection              |

### Example: identify the local adapter

```text
$ bluetoothctl show

Controller XX:XX:XX:XX:XX:XX
    Name: kali
    Powered: yes
```

**Interpretation:** The Bluetooth controller is available and powered.

### Example: discover nearby devices

```text
$ bluetoothctl
[bluetooth]# scan on

Device AA:BB:CC:DD:EE:FF Sensor-01
Device 11:22:33:44:55:66 Keyboard
```

**Interpretation:** Two Bluetooth devices have been discovered. Their addresses and advertised names become reconnaissance data.

### Example: inspect a discovered device

```text
[bluetooth]# info AA:BB:CC:DD:EE:FF
```

**Interpretation:** This retrieves information known to the local Bluetooth stack about that device.

### Example: inspect controller state

```text
$ sudo btmgmt info
```

**Interpretation:** Useful for determining controller capabilities and current Bluetooth management state.

### Example: check whether Bluetooth is blocked

```text
$ rfkill list
```

**Interpretation:** If Bluetooth is blocked, discovery tools may appear broken even though the adapter exists.

---

# Section 5 — Defender detection

* **Bluetooth monitoring:** Enterprise environments can monitor Bluetooth-capable endpoints and identify unexpected nearby devices.
* **Unexpected device associations:** New or unusual Bluetooth pairings can generate endpoint or device-management events.
* **BLE advertising:** IoT/BLE sensors may continuously advertise identifiable information that can be observed externally.
* **Unauthorized peripherals:** Unknown keyboards, headsets, sensors, or custom BLE devices can indicate an unmanaged wireless attack surface.
* **Behavioral anomalies:** Unexpected connections, repeated pairing attempts, or unusual device interactions may indicate reconnaissance or abuse.
* **Application-layer controls:** Strong Bluetooth encryption does not compensate for weak authorization in GATT services or the application using them.
* **Skilled operators reduce footprint:** Passive observation generally reveals less to the target than actively attempting connections or repeatedly triggering pairing workflows.

---

# Section 6 — Lab task

**Objective:** Use a Kali Bluetooth adapter to discover a Bluetooth/BLE device you own and distinguish basic discovery information from application-level functionality.

### Steps

1. Connect a Bluetooth-capable adapter to Kali.
2. Confirm the adapter is available and not blocked.
3. Start Bluetooth discovery.
4. Place your own Bluetooth/BLE device within range.
5. Record its address, advertised name, and visible device information.
6. Determine whether the device behaves as a Classic Bluetooth or BLE target.
7. Inspect the discovered device information.
8. Record what information was visible **without attempting exploitation**.
9. Save the observations as Markdown and terminal evidence.

### Expected output

```text
Device discovered
├── Address: AA:BB:CC:DD:EE:FF
├── Name: Sensor-01
├── Technology: BLE
└── Visible information: advertising/device metadata
```

### Git artifacts

```text
bluetooth-awareness/
├── README.md
├── notes/
│   └── bluetooth-classic-vs-ble.md
└── evidence/
    └── bluetooth-discovery.txt
```

Commit:

```bash
git add bluetooth-awareness/
git commit -m "Add Bluetooth and BLE awareness lab"
```

---

# Section 7 — Common mistakes

1. **Treating Classic and BLE as the same stack** → Their architecture and security models differ → Always classify the target first.

2. **Thinking Bluetooth discovery equals exploitation** → A discovered device may expose nothing useful → Identify actual services and functionality before judging risk.

3. **Confusing GAP and GATT** → They serve different purposes → Remember: **GAP = access/discovery behavior; GATT = application data/services**.

4. **Assuming pairing means complete security** → Pairing protects the communication relationship, not necessarily application authorization → Assess authorization separately.

5. **Ignoring advertising** → BLE can reveal useful information before a connection exists → Treat advertising data as reconnaissance.

6. **Jumping directly into BLE exploitation** → You can waste time without understanding the target's architecture → Build discovery and GATT fundamentals first; exploitation comes in Part 21 Stage 4.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Classify:** Given a Bluetooth target, explain whether you are dealing primarily with Bluetooth Classic or BLE and identify the relevant stack concepts.

2. **Enumerate:** Discover a Bluetooth/BLE device and record its address, name, advertising/discovery information, and basic role.

3. **Map:** Given a BLE architecture, correctly map **GAP → advertising/discovery** and **GATT → services → characteristics → data/control functionality**, then identify which findings would justify deeper investigation.
