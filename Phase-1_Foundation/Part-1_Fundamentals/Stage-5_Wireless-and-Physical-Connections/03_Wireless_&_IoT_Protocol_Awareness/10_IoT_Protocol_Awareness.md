# IoT Protocol Awareness

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Wireless & IoT Protocol Awareness → IoT Protocol Awareness

# Section 1 — What it is and where it sits

IoT devices rarely depend on Wi-Fi alone. Many use specialized wireless protocols designed around **low power, low bandwidth, automation, sensor networks, or long-distance communication**. Three important examples are **Zigbee**, **Z-Wave**, and **LoRaWAN**. They solve different problems and therefore have different architectures, ranges, communication models, and security assumptions.

The key cybersecurity skill at this stage is **protocol recognition**. When you encounter an IoT device, you should be able to identify whether it is likely using Zigbee, Z-Wave, LoRaWAN, Wi-Fi, BLE, or another technology and understand where its security boundary exists.

```text
IoT Wireless Protocols
│
├── Zigbee
│   ├── IEEE 802.15.4 foundation
│   ├── Mesh networking
│   └── Coordinator / Router / End Device
│
├── Z-Wave
│   ├── Low-power IoT networking
│   ├── Mesh networking
│   └── Controller / Nodes
│
└── LoRaWAN
    ├── Long-range LPWAN
    ├── End Devices
    ├── Gateways
    └── Network Server
```

These protocols fit after your basic wireless monitoring knowledge because IoT reconnaissance requires understanding **what kind of wireless network you are observing**. Full protocol exploitation is intentionally deferred to **Part 21 Stage 5**.

---

# Section 2 — How attackers actually use this

## 2.1 Identify the IoT protocol

The first attacker question is:

> **“What wireless technology connects this device to the rest of the IoT system?”**

The attacker wants to determine:

* Protocol
* Frequency/radio characteristics
* Network topology
* Device role
* Gateway/controller
* Addressing model
* Security mechanism
* Application purpose
* Whether communications eventually reach IP infrastructure

This classification determines the next attack path.

---

## 2.2 Zigbee reconnaissance

Zigbee is built on **IEEE 802.15.4** and is commonly used for:

* Smart lighting
* Sensors
* Home automation
* Industrial monitoring
* Smart plugs
* Security devices

Its major architectural characteristic is **mesh networking**.

```text
              Zigbee Coordinator
                     │
             ┌───────┴───────┐
             ↓               ↓
          Router            Router
          /   \              │
         ↓     ↓             ↓
      Sensor  Light        Sensor
```

An attacker therefore cares about more than a single endpoint.

A compromised or poorly secured node may potentially provide visibility into the wider Zigbee network.

Important reconnaissance questions:

* Who is the coordinator?
* Which devices are routers?
* Which devices are end devices?
* What devices are communicating?
* What network identifiers are visible?
* What security configuration appears to be in use?

---

## 2.3 Zigbee's mesh architecture changes the attack surface

In a simple point-to-point system:

```text
Device A ↔ Device B
```

In a mesh:

```text
A ↔ B ↔ C
    ↕
    D
```

Traffic may pass through intermediary routing devices.

This means an attacker should think about:

```text
Individual device
       ↓
Node role
       ↓
Mesh position
       ↓
Network-wide impact
```

A seemingly unimportant IoT device may have a strategically important position within the mesh.

---

## 2.4 Z-Wave reconnaissance

Z-Wave is another wireless technology commonly used in home/building automation.

Typical applications include:

* Smart locks
* Lighting
* Motion sensors
* Thermostats
* Alarm systems
* Home controllers

Conceptually:

```text
                 Controller
                /    |     \
               /     |      \
           Lock    Sensor   Light
              \       |      /
               ── Mesh ─────
```

The attacker wants to identify:

* Controller
* Nodes
* Device types
* Network relationships
* Security capabilities
* Commands/functions exposed by devices

The important lesson is that **Z-Wave is not Zigbee with a different name**. Their protocol architecture and security mechanisms differ.

---

## 2.5 LoRaWAN has a fundamentally different architecture

LoRaWAN is designed for **long-range, low-power wide-area networking (LPWAN)**.

Instead of a local mesh where every device forwards traffic, the architecture commonly looks like:

```text
IoT End Device
      │
      │ LoRa radio
      ↓
   Gateway
      │
      │ IP network
      ↓
 Network Server
      │
      ↓
 Application Server
      │
      ↓
 Application
```

This is a major architectural distinction.

### Zigbee

```text
Local devices
     ↕
Mesh
```

### LoRaWAN

```text
End devices
     ↓
Gateways
     ↓
Network infrastructure
```

The attacker therefore looks for **different security boundaries**.

---

## 2.6 LoRaWAN creates a gateway/network-server boundary

A LoRaWAN deployment may have hundreds or thousands of low-power devices communicating through gateways.

The attacker may therefore seek:

* End-device identifiers
* Gateway presence
* Network identifiers
* Application architecture
* Device deployment locations
* Message patterns
* Backend infrastructure
* Weakly protected management interfaces

A LoRaWAN device transmitting telemetry does not necessarily expose the same attack surface as a Zigbee smart bulb.

---

## 2.7 Security models differ

At a high level:

| Protocol | Main security perspective                                                |
| -------- | ------------------------------------------------------------------------ |
| Zigbee   | Network/mesh security, device authentication, key management             |
| Z-Wave   | Node/controller security and secure command communication                |
| LoRaWAN  | End-device/network/application security and gateway/backend architecture |

The critical lesson is:

> **Do not apply the security assumptions of one IoT protocol to another.**

For example, seeing encryption does not answer:

* Who authenticated whom?
* Who possesses the keys?
* Where are keys stored?
* What does a compromised node expose?
* Is application data separately protected?
* What happens when a device is replaced?

---

## 2.8 IoT protocol → physical function

IoT attacks become particularly valuable when wireless protocol access connects to a physical action.

Example:

```text
Wireless command
      ↓
IoT controller
      ↓
Smart lock
      ↓
Physical door
```

Or:

```text
Sensor
  ↓
Zigbee
  ↓
Gateway
  ↓
Backend
  ↓
Industrial process
```

The attacker therefore prioritizes devices where compromise could affect:

* Locks
* Alarms
* Industrial controls
* Cameras
* Sensors
* Building automation
* Vehicle-related systems
* Safety-relevant equipment

---

## 2.9 Dead-end vs high-value findings

| Finding                                   | Typical value |
| ----------------------------------------- | ------------- |
| IoT wireless device detected              | Low           |
| Protocol identified                       | Medium        |
| Device role identified                    | Medium        |
| Coordinator/controller/gateway identified | High          |
| Network topology mapped                   | High          |
| Weak device authentication                | High          |
| Sensitive telemetry exposed               | High          |
| Command/control interface discovered      | Very High     |
| Wireless access reaches physical control  | Critical      |

A protocol name alone is not a vulnerability.

The useful finding is:

> **Protocol + device + security boundary + reachable functionality.**

---

## 2.10 Where this feeds next

```text
Wireless discovery
       ↓
IoT protocol identification
       ↓
Topology mapping
       ↓
Device/controller/gateway identification
       ↓
Security model
       ↓
Potential attack surface
       ↓
Part 21 Stage 5
Full IoT protocol exploitation
```

---

# Section 3 — Core concepts and terminology

### Zigbee

Low-power wireless IoT networking technology commonly used for automation and sensor networks.

### IEEE 802.15.4

Wireless standard providing the underlying low-rate wireless personal-area networking foundation used by Zigbee.

### Zigbee Coordinator

Central Zigbee network role responsible for forming/managing the network.

### Zigbee Router

Zigbee device capable of forwarding traffic within the mesh.

### Zigbee End Device

Typically a lower-power device that communicates through a parent/router rather than routing traffic for other nodes.

### Z-Wave

Low-power wireless technology designed primarily for home/building automation.

### Z-Wave Controller

Device responsible for managing a Z-Wave network and its nodes.

### Z-Wave Node

An individual device participating in a Z-Wave network.

### LoRa

Long-range radio technology used for low-power wide-area communications.

### LoRaWAN

Networking protocol that manages communication between LoRa end devices and network infrastructure.

### LPWAN

**Low-Power Wide-Area Network**; networking category designed for long-range communication with low power consumption.

### LoRaWAN End Device

IoT device that transmits and/or receives LoRaWAN traffic.

### Gateway

Infrastructure that receives LoRa radio transmissions and forwards them into IP-based network infrastructure.

### Network Server

LoRaWAN infrastructure component responsible for network-level management and processing.

### Application Server

System responsible for application-level processing of IoT data.

### Mesh

Network topology where nodes can relay traffic for other nodes.

### Star-of-stars

LoRaWAN topology where end devices communicate with one or more gateways rather than forming a conventional device-to-device mesh.

### Telemetry

Data describing the state or measurements of a device or environment.

### Device provisioning

Process of registering/configuring a device and its security credentials for network operation.

### Key management

Generation, distribution, storage, rotation, and revocation of cryptographic keys.

---

## Protocol comparison

| Property                  | Zigbee                 | Z-Wave                      | LoRaWAN                        |
| ------------------------- | ---------------------- | --------------------------- | ------------------------------ |
| Primary use               | IoT/automation         | Home/building automation    | Long-range IoT                 |
| Network model             | Mesh                   | Mesh                        | Star-of-stars                  |
| Foundation                | IEEE 802.15.4          | Z-Wave protocol ecosystem   | LoRa radio + LoRaWAN           |
| Typical range             | Local                  | Local                       | Long-range                     |
| Typical power profile     | Low                    | Low                         | Very low                       |
| Common devices            | Lights, sensors, plugs | Locks, sensors, thermostats | Remote sensors/meters          |
| Major security boundary   | Mesh/network           | Controller + nodes          | End device + gateway + servers |
| Main reconnaissance focus | Nodes/topology         | Controller/nodes            | Devices/gateways/backend       |

---

# Section 4 — Tools and commands

These protocols require appropriate radio hardware; a normal Wi-Fi adapter cannot automatically monitor Zigbee, Z-Wave, or LoRaWAN traffic.

| Tool              | Command                                                 | What it finds/shows                              | When to use it                   |
| ----------------- | ------------------------------------------------------- | ------------------------------------------------ | -------------------------------- |
| `killerbee`       | `zbid`                                                  | IEEE 802.15.4/Zigbee-related adapter information | Zigbee research                  |
| `killerbee`       | `zbstumbler`                                            | Zigbee network discovery information             | Zigbee reconnaissance            |
| `killerbee`       | `zbdump`                                                | 802.15.4/Zigbee packet capture                   | Zigbee traffic analysis          |
| `Z-Wave JS`       | `zwave-js` tooling/environment                          | Z-Wave controller/node information               | Z-Wave research                  |
| `inspectrum`      | `inspectrum <capture>`                                  | Visual RF signal analysis                        | Analyzing recorded radio signals |
| `rtl_433`         | `rtl_433`                                               | Decodes many supported ISM-band RF protocols     | General RF/IoT reconnaissance    |
| `LoRaWAN` tooling | Packet capture/analyzer appropriate to your SDR/gateway | LoRa/LoRaWAN radio observations                  | LoRaWAN research                 |

### Example: Zigbee discovery

```text
$ zbstumbler
```

**Interpretation:** In a supported lab setup, this can help identify nearby Zigbee network activity.

### Example: Zigbee packet capture

```text
$ zbdump
```

**Interpretation:** Captures compatible IEEE 802.15.4 traffic for later protocol analysis.

### Example: RF signal inspection

```text
$ inspectrum capture.cfile
```

**Interpretation:** Displays signal characteristics from a recorded RF capture, helping identify modulation/timing patterns.

### Example: general ISM-band decoding

```text
$ rtl_433
```

**Interpretation:** With compatible SDR hardware, attempts to decode supported RF IoT/device protocols. It is a broad RF tool rather than a dedicated Zigbee/Z-Wave/LoRaWAN tool.

---

# Section 5 — Defender detection

* **IoT gateway/controller logs:** Unexpected node joins, removals, authentication failures, or configuration changes can expose attacks.
* **Network topology changes:** Unexpected Zigbee/Z-Wave nodes or routing changes can indicate unauthorized devices.
* **LoRaWAN backend telemetry:** Abnormal device behavior, message frequency, locations, or identifiers can indicate compromise.
* **Gateway monitoring:** Unexpected gateway traffic or management access should be investigated.
* **RF monitoring:** Specialized wireless monitoring can identify unauthorized transmissions or unexpected IoT infrastructure.
* **Behavioral anomalies:** A sensor suddenly transmitting at unusual intervals or issuing unexpected commands is often more useful than looking for a single signature.
* **Key/provisioning events:** Unexpected device provisioning or credential changes are high-value indicators.
* **Skilled operators reduce footprint:** Passive RF observation can reveal infrastructure without necessarily generating application-layer logs; defenders therefore need RF awareness in addition to conventional IP monitoring.

---

# Section 6 — Lab task

**Objective:** Build a small authorized IoT test network and identify its protocol, topology, device roles, and security boundary without performing exploitation.

### Steps

1. Choose one protocol you can legally reproduce with your available hardware, preferably Zigbee.
2. Deploy a coordinator and at least one test end device.
3. Confirm the radio adapter and coordinator are functioning.
4. Observe the network and identify participating devices.
5. Determine each device's role: coordinator, router, or end device where applicable.
6. Draw the resulting topology.
7. Record which component represents the primary security boundary.
8. Document what information is observable before authentication.
9. Save your observations and packet/RF evidence.

### Expected output

```text
IoT Test Network

Coordinator
    │
    ├── Router
    │     └── Sensor
    │
    └── End Device

Security boundary:
Coordinator / network key / application layer
```

### Git artifacts

```text
iot-protocol-awareness/
├── README.md
├── notes/
│   └── zigbee-zwave-lorawan.md
├── diagrams/
│   └── test-topology.md
└── evidence/
    └── protocol-observations.txt
```

Commit:

```bash
git add iot-protocol-awareness/
git commit -m "Add IoT protocol awareness lab"
```

---

# Section 7 — Common mistakes

1. **Treating Zigbee, Z-Wave, and LoRaWAN as interchangeable** → Their architectures and security boundaries differ → Identify the protocol before selecting an assessment technique.

2. **Assuming every IoT network is a mesh** → LoRaWAN uses a different gateway-oriented architecture → Learn the topology before reasoning about attack paths.

3. **Thinking 802.15.4 means “Zigbee”** → IEEE 802.15.4 is a lower-layer foundation used by multiple technologies → Distinguish the radio/link foundation from the higher-level protocol.

4. **Ignoring the gateway/controller** → It may be the bridge between the wireless network and valuable IP infrastructure → Map the complete architecture.

5. **Focusing only on encryption** → Encryption does not automatically provide correct authentication or authorization → Identify the complete security model.

6. **Assuming a sensor is low impact** → A small sensor may sit inside a strategically important mesh or control system → Assess its network position and downstream consequences.

7. **Jumping into exploitation immediately** → You may attack the wrong radio technology or device role → First identify protocol, topology, nodes, and security boundaries.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Classify:** Given an IoT wireless system, correctly distinguish **Zigbee, Z-Wave, and LoRaWAN** and state their fundamental architectural differences.

2. **Map:** Given a deployment, identify **Zigbee coordinator/router/end device**, **Z-Wave controller/nodes**, or **LoRaWAN end device/gateway/network server/application server** as appropriate.

3. **Assess:** Given an IoT architecture, identify the most valuable security boundary and explain how compromise of a wireless node, controller, gateway, or backend could change the attack surface—without relying on the assumption that all IoT protocols have the same security model.
