# Cable Infrastructure

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Physical Infrastructure → Cable Infrastructure

# Section 1 — What it is and where it sits

**Cable infrastructure** is the physical layer through which computers, switches, access points, servers, cameras, phones, and other network equipment communicate. In cybersecurity, understanding cabling matters because the network's logical security ultimately depends on a physical medium carrying the traffic.

The main categories you need to recognize are **copper twisted-pair Ethernet** such as UTP/STP and **fiber optics**. You also need to recognize common connectors and understand how cables are organized inside a physical plant.

```text
Endpoint
   │
   │ Copper/Fiber
   ↓
Wall Jack
   │
   ↓
Patch Panel
   │
   ↓
Horizontal / Backbone Cabling
   │
   ↓
Network Switch
   │
   ↓
Network Infrastructure
```

For an attacker, physical infrastructure can reveal **network topology, equipment locations, security zones, uplinks, unused ports, and opportunities for unauthorized physical access**.

This follows wireless/RF awareness because you are moving from radio-based connectivity to the **wired physical layer**; the next step is understanding how physical access can affect network security.

---

# Section 2 — How attackers actually use this

## 2.1 Identify the cable type

The first objective is determining:

> **“What physical medium is carrying this network?”**

An attacker performing physical reconnaissance may identify:

* Copper Ethernet
* Shielded vs unshielded twisted pair
* Fiber
* Connector type
* Cable destination
* Patch-panel organization
* Switch location
* Backbone links
* Unused network ports
* Physical security controls

A visible cable can sometimes reveal considerably more about an organization's infrastructure than its appearance suggests.

---

## 2.2 UTP vs STP

Ethernet copper cables use pairs of twisted wires.

```text
UTP
┌───────────────────────┐
│ Twisted copper pairs  │
│ No overall shielding  │
└───────────────────────┘

STP
┌───────────────────────┐
│ Shielding             │
│ Twisted copper pairs  │
│ Better EMI resistance │
└───────────────────────┘
```

### UTP — Unshielded Twisted Pair

The individual copper pairs are twisted but there is no additional conductive shielding around the cable.

Common advantages:

* Cheap
* Flexible
* Easy to install
* Very common in offices

### STP — Shielded Twisted Pair

Additional shielding reduces susceptibility to electromagnetic interference.

It is useful in environments such as:

* Industrial facilities
* Areas with significant electrical interference
* Data centers
* Specialized installations

The security lesson is not that STP is automatically "more secure." **Shielding primarily addresses electromagnetic interference, not authentication or encryption.**

---

## 2.3 Understand Ethernet cable categories

You will encounter labels such as:

* Cat5e
* Cat6
* Cat6A
* Cat7
* Cat8

These indicate cable/category specifications and supported performance characteristics.

For reconnaissance, the important skill is being able to read the label and understand that **cable category affects the physical capabilities of the link**.

Do not confuse:

```text
Cable category ≠ Security level
```

A Cat6 cable does not inherently provide stronger security than Cat5e.

---

## 2.4 Fiber optics work differently

Fiber uses **light rather than electrical signals**.

```text
Electrical Ethernet

Switch ───── copper ───── Endpoint


Fiber Ethernet

Switch ───── light pulses ───── Endpoint
```

Two major fiber types are:

### Multimode fiber

Generally designed for shorter-distance applications and commonly used within buildings/data centers.

### Single-mode fiber

Designed for much longer-distance transmission and commonly used for backbone/telecommunications links.

For physical reconnaissance, fiber can be particularly interesting because it often indicates:

```text
Building backbone
        ↓
Distribution layer
        ↓
Data center / network core
```

---

## 2.5 Connector recognition

Knowing the connector helps identify what infrastructure you are looking at.

Common connectors include:

| Connector   | Common association                   |
| ----------- | ------------------------------------ |
| RJ45 / 8P8C | Copper Ethernet                      |
| LC          | Fiber                                |
| SC          | Fiber                                |
| ST          | Fiber                                |
| MPO/MTP     | High-density multi-fiber connections |

The important practical skill is:

> **Recognize the connector and infer the likely physical medium and network role.**

For example, an LC fiber connection between network racks is more likely to represent a high-speed backbone/uplink than a normal desktop connection.

---

## 2.6 Patch panels reveal topology

A patch panel is not normally an active networking device. It provides an organized termination point for structured cabling.

```text
Office Jack
    │
    │ Permanent cable
    ↓
Patch Panel
    │
    │ Patch cable
    ↓
Switch
```

An attacker with physical access may be able to learn:

* Port numbering
* Room/location relationships
* Cable destinations
* VLAN/network documentation
* Which ports appear active
* Which cables connect different infrastructure areas

This makes physical organization valuable reconnaissance data.

---

## 2.7 Physical plant organization

A structured cabling environment commonly contains:

```text
Work Area
    ↓
Telecommunications Room
    ↓
Building Distribution
    ↓
Main Equipment Room
    ↓
Data Center / Core
```

Terminology varies by organization and cabling standard, but the general concept is:

**endpoint → access/distribution infrastructure → backbone/core**

Understanding this lets you reason about where a particular cable may lead.

---

## 2.8 Cable labeling is intelligence

Well-organized infrastructure often uses labels such as:

```text
TR-02-P24
SW03-Gi1/0/24
FBR-A-CORE-07
```

These may encode:

* Room
* Rack
* Patch panel
* Switch
* Port
* Backbone segment

From an attacker's perspective, this can provide a physical map without touching the logical network.

---

## 2.9 Physical access can become logical access

Consider:

```text
Physical access
      ↓
Network jack
      ↓
Switch port
      ↓
VLAN
      ↓
Internal network
      ↓
Authentication / services
```

The existence of a network port does **not** automatically mean an attacker can access the network. Controls such as:

* 802.1X
* Port security
* NAC
* Disabled unused ports
* VLAN segmentation

can prevent or restrict unauthorized access.

But identifying an exposed or poorly controlled port is still a valuable finding.

---

## 2.10 Fiber introduces different physical concerns

Fiber does not radiate electrical Ethernet signals in the same way copper does, but it is **not magically immune to interception or disruption**.

Physical attackers may care about:

* Accessible fiber routes
* Exposed patch panels
* Unsecured equipment rooms
* Cable cuts
* Unauthorized patching
* Physical taps
* Backbone redundancy

At this stage, you only need to recognize these as attack surfaces. Detailed fiber interception belongs in a deeper physical-security assessment.

---

## 2.11 Dead-end vs high-value findings

| Finding                                             | Typical value |
| --------------------------------------------------- | ------------- |
| Cable type identified                               | Low           |
| Cable category identified                           | Low           |
| Connector identified                                | Low           |
| Patch-panel numbering exposed                       | Medium        |
| Network room identified                             | Medium        |
| Backbone cable identified                           | High          |
| Unused live network port                            | High          |
| Unsecured patch-panel access                        | High          |
| Unprotected network equipment room                  | Very High     |
| Physical path into sensitive network infrastructure | Very High     |

The important lesson:

> **A cable itself is usually not the vulnerability; the physical access surrounding it is.**

---

## 2.12 Where this feeds next

```text
Physical reconnaissance
       ↓
Cable identification
       ↓
Connector identification
       ↓
Patch-panel mapping
       ↓
Network equipment identification
       ↓
Physical access opportunities
       ↓
Layer-2 / physical security assessment
```

---

# Section 3 — Core concepts and terminology

### Twisted Pair

Copper conductors twisted together to reduce electromagnetic interference and crosstalk.

### UTP

**Unshielded Twisted Pair**; twisted copper pairs without additional shielding.

### STP

**Shielded Twisted Pair**; twisted-pair cabling with additional shielding.

### Ethernet

Common LAN networking technology frequently carried over twisted-pair copper or fiber.

### Cable Category

Specification defining characteristics such as frequency/performance capabilities of twisted-pair cabling.

### Cat5e

Enhanced Category 5 cabling commonly used for Gigabit Ethernet.

### Cat6

Higher-performance twisted-pair cabling with improved crosstalk characteristics.

### Cat6A

Enhanced Category 6 cabling designed for higher-performance Ethernet applications including 10 Gb/s over appropriate distances.

### Fiber Optic

Cable that transmits information using light through optical fiber.

### Multimode Fiber

Fiber designed to support multiple propagation modes and commonly used for shorter-distance links.

### Single-mode Fiber

Fiber designed primarily around a single propagation mode and commonly used for longer-distance links.

### RJ45 / 8P8C

Common modular connector used for Ethernet copper cabling. Strictly speaking, many connectors casually called "RJ45" are 8P8C modular connectors.

### LC

Small-form-factor fiber connector commonly used in modern network equipment.

### SC

Larger push-pull fiber connector.

### ST

Bayonet-style fiber connector.

### MPO/MTP

High-density multi-fiber connector used for high-bandwidth fiber systems.

### Patch Panel

Passive termination/organization point for structured network cabling.

### Patch Cable

Short cable used to connect equipment to patch panels or other interfaces.

### Backbone

Cabling connecting major network infrastructure areas, floors, buildings, or distribution points.

### Horizontal Cabling

Cabling generally connecting telecommunications rooms to work-area outlets within a floor/area.

### Telecommunications Room

Physical room containing network termination and distribution equipment for an area.

### Rack

Physical frame/cabinet used to mount network and computing equipment.

### Cable Management

Physical organization system used to route, label, secure, and separate cables.

### Crosstalk

Unwanted signal coupling from one communication pair into another.

### EMI

**Electromagnetic Interference** that can disrupt electrical communication signals.

### 802.1X

Port-based network access-control mechanism commonly used to authenticate devices before granting network access.

### NAC

**Network Access Control**; controls network connectivity based on device/user/security policy.

---

# Section 4 — Tools and commands

Physical cable identification is primarily a **visual/physical reconnaissance task**, but Linux can help correlate a physical port with its logical network interface.

| Tool      | Command                | What it finds/shows                                         | When to use it                             |
| --------- | ---------------------- | ----------------------------------------------------------- | ------------------------------------------ |
| `ethtool` | `sudo ethtool eth0`    | Ethernet link state, speed, duplex, negotiated capabilities | Identify active wired link                 |
| `ethtool` | `sudo ethtool -i eth0` | Driver and hardware information                             | Identify Ethernet adapter                  |
| `ip`      | `ip link`              | Network interfaces and link state                           | Correlate logical interfaces               |
| `ip`      | `ip -br link`          | Compact interface/link summary                              | Quick inventory                            |
| `lldpctl` | `sudo lldpctl`         | LLDP neighbor/device information                            | Identify connected network equipment       |
| `nmcli`   | `nmcli device status`  | Network devices and connection state                        | Quick physical/logical interface inventory |

### Example: inspect Ethernet link

```text
$ sudo ethtool eth0

Speed: 1000Mb/s
Duplex: Full
Link detected: yes
```

**Interpretation:** The interface has a live 1 Gb/s full-duplex Ethernet connection.

### Example: inspect interfaces

```text
$ ip -br link

lo       UNKNOWN
eth0     UP
wlan0    DOWN
```

**Interpretation:** `eth0` is currently operational while `wlan0` is not.

### Example: identify a connected switch

```text
$ sudo lldpctl
```

**Interpretation:** If LLDP is enabled, the host may learn information about the directly connected network device, such as its system name, port, and capabilities.

### Example: inspect Ethernet hardware

```text
$ sudo ethtool -i eth0
```

**Interpretation:** Shows the Ethernet driver's and adapter's hardware information, useful when troubleshooting or identifying the physical network interface.

---

# Section 5 — Defender detection

* **Physical inspections:** Regularly inspect racks, patch panels, cable routes, and network closets for unauthorized changes.
* **Port monitoring:** Switch telemetry can reveal unexpected link activation on previously unused ports.
* **802.1X/NAC events:** Unauthorized devices attempting to use physical Ethernet ports can generate authentication failures.
* **LLDP/CDP monitoring:** Unexpected topology changes can reveal unauthorized equipment or cabling.
* **Cable labeling:** Proper documentation makes unauthorized patching easier to detect.
* **Physical access control:** Network rooms, patch panels, and backbone infrastructure should be protected because physical access can bypass many logical controls.
* **Skilled operators minimize interaction:** A physical reconnaissance operator may gather substantial topology information through observation and documentation without immediately generating network events.

---

# Section 6 — Lab task

**Objective:** Map the physical-to-logical path of an Ethernet connection on your own lab network and identify the cable, connector, patching point, switch port, and negotiated link characteristics.

### Steps

1. Select a wired device and trace its Ethernet cable from the endpoint to your switch/patch panel.
2. Record whether the cable is UTP or STP and its printed category.
3. Identify the connector type at the endpoint.
4. Record the patch-panel port if one exists.
5. Identify the corresponding switch port.
6. Use Linux to inspect the interface's link state and negotiated speed.
7. Use LLDP where available to correlate the host with the connected network device.
8. Draw the physical-to-logical path.
9. Save the observations and terminal evidence.

### Expected output

```text
Laptop
  ↓
Cat6 UTP
  ↓
RJ45/8P8C
  ↓
Patch Panel P24
  ↓
Switch SW01 Port 24
  ↓
1 Gb/s Full Duplex
```

### Git artifacts

```text
physical-infrastructure/
├── README.md
├── notes/
│   └── cable-infrastructure.md
├── diagrams/
│   └── physical-to-logical-path.md
└── evidence/
    └── ethernet-link.txt
```

Commit:

```bash
git add physical-infrastructure/
git commit -m "Add cable infrastructure reconnaissance lab"
```

---

# Section 7 — Common mistakes

1. **Thinking UTP/STP is primarily a security distinction** → Shielding mainly addresses interference → Do not confuse electromagnetic protection with network security.

2. **Calling every modular Ethernet connector “RJ45” without qualification** → The common connector is technically an 8P8C modular connector → Recognize the terminology but understand what you physically have.

3. **Ignoring cable labels** → Category and labeling can reveal capabilities and topology → Read the cable jacket and document its markings.

4. **Treating patch panels as switches** → Patch panels are generally passive termination/organization hardware → Follow the connection from patch panel to active network equipment.

5. **Assuming fiber is impossible to intercept** → Fiber still has physical attack and disruption possibilities → Treat exposed backbone infrastructure as sensitive.

6. **Assuming an Ethernet port automatically provides network access** → 802.1X, NAC, VLANs, and port security can restrict access → Assess the logical controls associated with the physical port.

7. **Ignoring physical organization** → Rack/patch-panel layout can reveal the network architecture → Treat physical plant documentation and labeling as reconnaissance information.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Identify:** Given several cables and connectors, correctly distinguish **UTP, STP, multimode fiber, single-mode fiber, RJ45/8P8C, LC, SC, ST, and MPO/MTP**.

2. **Trace:** Starting from an endpoint, physically trace the path through **cable → wall jack → patch panel → switch port → network infrastructure** and document it.

3. **Correlate:** Given a physical Ethernet connection, use Linux tooling to correlate the physical connection with its **interface, link state, speed, duplex, and directly connected network device**.
