# Environmental & Physical Security Awareness

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Physical Infrastructure → Environmental & Physical Security Awareness

# Section 1 — What it is and where it sits

**Environmental and physical security** concerns the conditions and physical boundaries surrounding computing equipment. Cybersecurity is not limited to packets and software: someone who can physically access a server, network cabinet, badge reader, workstation, or communications cable may gain opportunities that are unavailable remotely.

Two concepts are especially important here: **electromagnetic shielding** and **physical access control**. You should also recognize **TEMPEST** as a family of concepts concerning compromising electromagnetic emissions from equipment. At this stage, you need to understand what these systems protect and why they matter—not perform specialized interception or physical attacks.

```text
Physical Environment
│
├── Electromagnetic Security
│   ├── EMI
│   ├── Shielding
│   └── TEMPEST concepts
│
├── Physical Access Control
│   ├── Smart cards
│   ├── RFID credentials
│   ├── Readers
│   └── Door controllers
│
└── Environmental Controls
    ├── Power
    ├── Cooling
    ├── Fire protection
    └── Physical monitoring
```

The attack-chain position is:

```text
Physical reconnaissance
        ↓
Identify facility controls
        ↓
Identify access boundaries
        ↓
Identify exposed equipment
        ↓
Understand physical attack surface
        ↓
Physical penetration testing / hardware security
```

Detailed physical penetration testing belongs to **Part 32 (Phase 7)**, while hardware hacking belongs to **Part 30 (Phase 7)**.

---

# Section 2 — How attackers actually use this

## 2.1 Identify the physical security boundary

An attacker first wants to understand:

> **“What physically prevents me from reaching the systems I care about?”**

They may map:

* Doors
* Locks
* Badge readers
* Security desks
* CCTV
* Mantraps
* Server rooms
* Network closets
* Equipment racks
* Restricted areas
* Visitor areas
* Cable routes

The objective is not necessarily to defeat a control immediately.

The first objective is to understand the **physical trust boundary**.

```text
Public Area
    ↓
Reception
    ↓
Badge-controlled door
    ↓
Restricted floor
    ↓
Network room
    ↓
Critical infrastructure
```

---

## 2.2 Physical access control systems

A typical electronic access-control system looks like:

```text
Credential
    ↓
Reader
    ↓
Access Controller
    ↓
Authorization decision
    ↓
Door lock
```

The credential might be:

* Smart card
* RFID card
* NFC credential
* PIN
* Mobile credential
* Biometric factor

The important security distinction is:

**Credential ≠ reader ≠ authorization system.**

A card may identify a user, while the controller/backend decides whether that user is allowed through the door.

---

## 2.3 Smart cards

A smart card contains an integrated circuit capable of storing information and/or performing cryptographic operations.

Conceptually:

```text
Smart Card
   │
   ├── Identity / application data
   ├── Cryptographic functions
   └── Security credentials
          ↓
       Reader
          ↓
     Access system
```

This can be substantially stronger than a system that simply checks a static identifier.

Therefore, when assessing a physical access system, an attacker wants to determine:

* What technology is used?
* Is the credential merely an identifier?
* Is cryptographic authentication involved?
* Where is authorization performed?
* Can credentials be revoked?
* What happens when a credential is lost?

Detailed credential attacks are outside this stage.

---

## 2.4 RFID readers

RFID-based access control commonly follows:

```text
RFID Credential
      ↓
RFID Reader
      ↓
Controller
      ↓
Door
```

The attacker may initially learn useful information simply by identifying:

* Reader manufacturer/model
* Credential type
* Reader location
* Which doors are controlled
* Whether multiple doors use the same technology

This information can help build a physical attack-surface map.

---

## 2.5 Electromagnetic interference

Electronic equipment produces electromagnetic energy.

Unwanted electromagnetic energy can cause:

* Interference
* Communication errors
* Equipment malfunction
* Signal degradation

This is known as **electromagnetic interference (EMI)**.

Shielding attempts to reduce unwanted electromagnetic coupling.

```text
Without shielding

Source  ~~~~~~~~→  Nearby equipment


With shielding

Source  [Shielded enclosure/cable]
          ↓
     Reduced coupling
```

Common shielding approaches include:

* Conductive enclosures
* Shielded cables
* Proper grounding/bonding
* Physical separation
* Filtering

The key lesson:

> **EM shielding is primarily about controlling electromagnetic coupling and emissions; it is not a substitute for cryptographic security.**

---

## 2.6 Shielding does not mean “invisible”

A common misconception is:

> “If equipment is shielded, electromagnetic information cannot escape.”

Real systems are more complicated.

Potential emission/coupling paths can include:

* Cables
* Connectors
* Power lines
* Ventilation/openings
* Poorly bonded shielding
* Nearby conductive structures

Effective electromagnetic security therefore involves the **entire system**, not simply putting a metal box around a computer.

---

## 2.7 TEMPEST concepts

**TEMPEST** is commonly used as a broad term associated with protecting against compromising electromagnetic emanations from information-processing equipment.

Conceptually:

```text
Computer
   │
   ├── Display signals
   ├── Digital electronics
   ├── Cables
   └── Power systems
          ↓
     Electromagnetic emissions
          ↓
   Potential information leakage
```

The security concern is that unintended electromagnetic emissions may potentially contain information correlated with what the equipment is processing.

At this stage, you only need to understand:

* Equipment can emit electromagnetic energy.
* Some emissions may correlate with processed information.
* Shielding and controlled electromagnetic environments can reduce exposure.
* TEMPEST is a specialized security discipline.

Detailed TEMPEST measurement/interception techniques are **not part of this stage**.

---

## 2.8 Environmental security is broader than physical locks

Physical security also includes environmental threats:

```text
Threats
├── Unauthorized people
├── Fire
├── Flood
├── Excessive heat
├── Humidity
├── Power failure
├── Electrical disturbances
└── Equipment damage
```

For example, a server room with excellent authentication but inadequate cooling can still experience a security-impacting outage.

An attacker assessing a facility therefore considers both:

**intentional threats** and **environmental failure modes**.

---

## 2.9 Physical access can bypass logical controls

Consider a workstation protected by a strong password:

```text
Remote attacker
     ↓
Password required
     ↓
Blocked


Physical attacker
     ↓
Direct hardware access
     ↓
Different attack surface
```

Physical access can potentially expose:

* USB interfaces
* Storage devices
* Debug interfaces
* Console ports
* Network ports
* BIOS/UEFI interfaces
* Physical reset mechanisms

You do not need to exploit these yet. The important skill is recognizing that **physical possession changes the threat model**.

---

## 2.10 Dead-end vs high-value findings

| Finding                                         | Typical value    |
| ----------------------------------------------- | ---------------- |
| Generic badge reader                            | Low              |
| RFID technology identified                      | Medium           |
| Server room location identified                 | High             |
| Unsecured network cabinet                       | High             |
| Unrestricted equipment access                   | Very High        |
| Exposed physical management interface           | Very High        |
| Weak physical authentication                    | High             |
| Sensitive equipment accessible from public area | Critical         |
| Significant electromagnetic leakage             | Specialized/High |

The value depends heavily on what the physical asset controls.

---

## 2.11 Where this feeds next

```text
Physical reconnaissance
       ↓
Access-control identification
       ↓
Environmental/security-boundary mapping
       ↓
Equipment exposure assessment
       ↓
Physical attack surface
       ├───────────────┐
       ↓               ↓
Part 32            Part 30
Physical Pentest   Hardware Hacking
```

---

# Section 3 — Core concepts and terminology

### Physical Security

Protection of people, facilities, equipment, and physical resources against unauthorized access or damage.

### Physical Access Control

Mechanisms controlling who can enter a physical area.

### Credential

Object or information used to prove or establish identity for access.

### Smart Card

Card containing an integrated circuit capable of storing data and potentially performing cryptographic operations.

### RFID

Radio-frequency identification technology used to identify objects or credentials wirelessly.

### NFC

Short-range wireless technology commonly used for contactless credentials and payments.

### RFID Reader

Device that communicates with RFID credentials/tags.

### Access Controller

System that makes or enforces physical access decisions.

### Mantrap

Physical access-control arrangement using two doors to control entry/exit and reduce unauthorized passage.

### EMI

**Electromagnetic Interference**; unwanted electromagnetic energy that interferes with equipment or communications.

### EMC

**Electromagnetic Compatibility**; ability of equipment to operate correctly without causing or suffering unacceptable electromagnetic interference.

### Electromagnetic Shielding

Use of conductive or other engineering techniques to reduce electromagnetic coupling or emissions.

### Grounding

Connecting electrical systems to an appropriate reference/earth to support safety and electromagnetic control.

### Bonding

Creating low-impedance electrical connections between conductive components.

### TEMPEST

Specialized security discipline concerning compromising electromagnetic emanations and measures used to control them.

### Emanation

Unintended energy emitted or conducted by equipment that may potentially reveal information.

### Faraday Cage

Conductive enclosure that can reduce electromagnetic fields entering or leaving an enclosed space, depending on construction and frequency.

### Environmental Security

Protection against environmental conditions such as fire, heat, humidity, flooding, and power disturbances.

### Physical Attack Surface

Physical interfaces, equipment, spaces, and controls that could potentially be abused to affect security.

### Defense in Depth

Using multiple independent security controls so that failure of one does not automatically compromise the system.

---

## Physical access-control chain

| Component  | Main responsibility                                          |
| ---------- | ------------------------------------------------------------ |
| Credential | Presents identity/authentication information                 |
| Reader     | Reads/interacts with credential                              |
| Controller | Evaluates/enforces access decision                           |
| Lock       | Physically permits/denies entry                              |
| Backend    | May store identities, policies, logs, and authorization data |
| Monitoring | Detects and records physical events                          |

---

# Section 4 — Tools and commands

This topic is primarily **physical reconnaissance**, so many useful tools are observation/documentation tools rather than offensive software.

| Tool        | Command                    | What it finds/shows                 | When to use it                        |
| ----------- | -------------------------- | ----------------------------------- | ------------------------------------- |
| `lsusb`     | `lsusb`                    | USB-connected hardware              | Inventory physical interfaces         |
| `lspci`     | `lspci`                    | PCI/PCIe hardware                   | Identify installed hardware           |
| `dmidecode` | `sudo dmidecode -t system` | System/motherboard information      | Hardware inventory                    |
| `ethtool`   | `sudo ethtool eth0`        | Ethernet link characteristics       | Correlate physical network connection |
| `rfkill`    | `rfkill list`              | Wireless radio hardware/block state | Hardware/radio inventory              |

### Example: inspect USB devices

```text
$ lsusb

Bus 001 Device 003: ID xxxx:xxxx USB Device
```

**Interpretation:** The system has a USB-connected device. This can help identify externally connected hardware.

### Example: inspect PCI hardware

```text
$ lspci
```

**Interpretation:** Displays PCI/PCIe hardware such as network controllers, storage controllers, and other system components.

### Example: system hardware information

```text
$ sudo dmidecode -t system
```

**Interpretation:** Provides system manufacturer/model and related firmware-exposed information.

### Example: inspect wired connection

```text
$ sudo ethtool eth0
```

**Interpretation:** Shows the negotiated Ethernet state, including link detection, speed, and duplex.

---

# Section 5 — Defender detection

* **Badge/access logs:** Record successful and failed credential presentations and correlate them with authorized users.
* **Door alarms:** Detect forced doors, doors held open, or abnormal access events.
* **CCTV and physical sensors:** Provide evidence when digital access logs alone cannot explain an event.
* **Environmental monitoring:** Temperature, humidity, smoke, water, and power sensors detect physical conditions threatening infrastructure.
* **Equipment-room monitoring:** Restrict and monitor access to server rooms, network closets, and sensitive hardware.
* **EM security controls:** Specialized environments can use shielding, filtering, grounding/bonding, and controlled equipment layouts to reduce electromagnetic leakage.
* **Skilled operators minimize interaction:** Physical reconnaissance may initially consist of observation and mapping, producing little or no conventional network telemetry.

---

# Section 6 — Lab task

**Objective:** Perform a physical-security survey of your own lab environment and document the relationship between access controls, network equipment, physical interfaces, and environmental protections.

### Steps

1. Select your own workstation, home lab, or authorized equipment area.
2. Identify the physical boundary protecting the equipment.
3. Record the access-control mechanism, if one exists.
4. Identify exposed interfaces such as Ethernet, USB, console, or removable storage.
5. Identify network equipment and cable-access points.
6. Record environmental protections such as cooling, power backup, smoke detection, or equipment enclosure.
7. Identify where electromagnetic shielding is present, such as shielded cables or conductive equipment enclosures.
8. Draw the physical-security boundary.
9. Document the highest-value physical exposure you found.

### Expected output

```text
Physical Boundary
│
├── Access Control
│   └── Locked room
│
├── Network Equipment
│   └── Switch / Router
│
├── Physical Interfaces
│   ├── Ethernet
│   └── USB
│
└── Environmental Controls
    ├── Cooling
    └── UPS
```

### Git artifacts

```text
physical-security-awareness/
├── README.md
├── notes/
│   └── environmental-physical-security.md
├── diagrams/
│   └── physical-security-boundary.md
└── evidence/
    └── hardware-interface-inventory.txt
```

Commit:

```bash
git add physical-security-awareness/
git commit -m "Add physical security awareness lab"
```

---

# Section 7 — Common mistakes

1. **Thinking physical security only means locks** → Environmental failures and exposed interfaces also create security risk → Assess the entire physical environment.

2. **Assuming RFID/smart cards automatically provide strong authentication** → Security depends on the credential technology and how the backend validates it → Identify the complete access-control architecture.

3. **Confusing shielding with encryption** → Shielding controls electromagnetic energy; encryption protects information logically → Treat them as separate security mechanisms.

4. **Assuming a Faraday cage blocks everything** → Effectiveness depends on construction, openings, grounding, frequency, and implementation → Think in terms of electromagnetic engineering rather than a magic barrier.

5. **Ignoring cables and connectors** → They can provide electromagnetic coupling paths and physical access opportunities → Include the entire equipment installation in the assessment.

6. **Ignoring environmental threats** → Fire, heat, water, and power failures can cause major security and availability impacts → Include environmental controls in physical reconnaissance.

7. **Jumping into hardware/TEMPEST attacks** → Specialized attacks require substantially deeper knowledge and equipment → Build the physical-security model first; detailed hardware work belongs to Part 30 and physical pentesting to Part 32.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Map:** Given a physical facility, identify its **physical security boundary, access-control mechanism, equipment areas, exposed interfaces, and environmental controls**.

2. **Explain:** Given a system using smart cards/RFID, correctly map **credential → reader → controller → authorization decision → physical lock**, and explain why each component matters.

3. **Assess:** Given an electronic system, explain the difference between **EMI, electromagnetic shielding, EMC, and TEMPEST concepts**, and identify which physical-security questions should be investigated later in **Part 30 or Part 32**.
