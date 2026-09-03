# NFC/RFID Awareness

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → Wireless & IoT Protocol Awareness → NFC/RFID Awareness

# Section 1 — What it is and where it sits

**NFC (Near Field Communication)** and **RFID (Radio-Frequency Identification)** are short-range wireless technologies used to identify objects, exchange small amounts of data, authenticate users/devices, make payments, and track physical assets. They overlap conceptually but are not interchangeable: NFC is a family of short-range communication technologies with very specific operating modes and standards, while RFID is a broader identification technology that includes passive and active tags.

For cybersecurity, the important question is not simply *“Does this object have NFC/RFID?”* but **what information, identity, authentication mechanism, or physical access decision depends on it?**

```text
RF / Proximity Technologies
├── RFID
│   ├── Passive tags
│   └── Active tags
│
└── NFC
    ├── Reader/Writer
    ├── Card Emulation
    └── Peer-to-Peer
```

Typical attack-chain placement:

```text
Physical target
      ↓
Identify NFC/RFID technology
      ↓
Identify reader + tag/card
      ↓
Understand protocol + credential model
      ↓
Determine security dependency
      ↓
Deeper NFC/RFID assessment
```

If you skip this topic, you may completely miss a physical attack surface—for example, an access-control badge may be the actual credential protecting a building or system. Full NFC/RFID attacks are deferred to **Part 21 Stage 6**.

---

# Section 2 — How attackers actually use this

## 2.1 Identify what technology is actually being used

The first objective is classification.

An attacker wants to determine:

* NFC or another RFID technology?
* Passive or active tag?
* Approximate operating frequency/standard?
* What type of reader is present?
* What type of credential/tag is being used?
* Is the tag merely an identifier or does it contain application data?
* Is authentication involved?
* Is the credential used for access control, payment, tracking, or device configuration?

A badge reader beside a door, for example, immediately suggests a different assessment path from an NFC tag attached to a product.

---

## 2.2 Passive RFID vs active RFID

The major distinction is how the tag obtains power.

```text
Passive RFID

Reader
  │
  │ RF energy
  ↓
Tag
  │
  └── responds using harvested energy


Active RFID

Tag
  │
  ├── Battery
  └── RF transmission
          ↓
       Reader
```

**Passive tags** generally do not contain their own battery and obtain operating energy from the reader's RF field.

**Active tags** contain their own power source and can generally communicate over greater distances and/or support more functionality.

From an attacker's perspective, this affects:

* Expected range
* Device behavior
* Tag capabilities
* Battery dependency
* Discovery methodology
* Potential attack surface

---

## 2.3 NFC is extremely close-range

NFC is designed for communication at very short distances.

This makes NFC fundamentally different from Wi-Fi reconnaissance:

```text
Wi-Fi
Attacker ─────────────── AP

NFC
Attacker ── Tag/Reader
          very close
```

The short range can reduce accidental exposure, but it does **not** make the technology automatically secure.

An NFC credential may still be security-critical because possession of the physical credential can authorize:

* Building entry
* Transit access
* Payment
* Device pairing
* Identity verification
* Application actions

---

## 2.4 Understand the three NFC operating modes

NFC commonly involves three conceptual operating modes.

### Reader/Writer

A device reads or writes compatible NFC tags.

```text
Phone/Reader → NFC Tag
```

Examples include:

* Reading an NFC information tag
* Writing an NDEF record
* Reading a configured smart tag

### Card Emulation

A device behaves like an NFC card/tag.

```text
Reader → Phone/Card Emulator
```

This is important in:

* Mobile payments
* Access credentials
* Transit systems
* Digital identity systems

### Peer-to-Peer

Two NFC-capable devices exchange information.

```text
Device A ↔ Device B
```

This is less central to modern security assessments than reader/writer and card-emulation scenarios but is part of the NFC model.

---

## 2.5 Access control is the high-value target

Consider:

```text
Employee
   ↓
RFID/NFC Badge
   ↓
Door Reader
   ↓
Access Controller
   ↓
Door Unlocks
```

The attacker is interested in the **decision-making chain**.

Important questions include:

* What does the credential identify?
* Is the identifier static?
* Is cryptographic authentication used?
* Does the reader validate the credential locally or through a backend?
* What happens if an invalid credential is presented?
* Does the system distinguish authentication from authorization?

The physical card itself may be simple, while the security decision occurs elsewhere.

---

## 2.6 Payments have a different security model

NFC payment systems generally involve significantly more sophisticated security architecture than a basic proximity badge.

Conceptually:

```text
Payment Device
      ↓
NFC interaction
      ↓
Payment terminal
      ↓
Payment network/backend
```

The attacker therefore needs to distinguish:

**“I found an NFC interface.”**

from:

**“I found an NFC interface that contains or exposes a useful security credential.”**

Finding NFC is reconnaissance. Understanding the credential and transaction model is what makes the finding meaningful.

---

## 2.7 Tracking creates another attack surface

RFID is also used for:

* Inventory
* Warehousing
* Logistics
* Asset tracking
* Library systems
* Industrial environments

An attacker may be interested in whether tags reveal:

* Unique identifiers
* Product identifiers
* Asset numbers
* Organization-specific information
* Persistent tracking information

Even when no direct compromise exists, exposed identifiers can provide useful intelligence about the physical environment.

---

## 2.8 Dead-end vs high-value findings

| Finding                                      | Typical value |
| -------------------------------------------- | ------------- |
| NFC/RFID technology present                  | Low           |
| Tag detected                                 | Low           |
| Public informational tag                     | Low           |
| Unique identifier exposed                    | Medium        |
| Persistent asset identifier                  | Medium        |
| Access-control credential identified         | High          |
| Weak authentication scheme                   | High          |
| Credential cloning/replay possibility        | Very High     |
| Credential directly controls physical access | Very High     |

The important distinction is:

> **A tag is not necessarily a credential, and a credential is not necessarily an authentication mechanism.**

---

## 2.9 Where this feeds next

```text
NFC/RFID discovery
       ↓
Technology classification
       ↓
Tag/card/reader identification
       ↓
Credential model
       ↓
Authentication + authorization
       ↓
Part 21 Stage 6
NFC/RFID attacks
```

At this stage, you should be able to recognize **when NFC/RFID is security-relevant**. You do not need to perform cloning, replay, emulation, or protocol attacks yet.

---

# Section 3 — Core concepts and terminology

### RFID

**Radio-Frequency Identification**; technology for identifying objects using RF communication between readers and tags.

### NFC

**Near Field Communication**; short-range communication technology based on contactless communication standards and commonly associated with the 13.56 MHz NFC ecosystem.

### ISO/IEC 14443

A family of standards for proximity cards and cards/readers operating at 13.56 MHz; widely relevant to NFC-based contactless cards.

### Tag

The RFID/NFC object that stores or provides identification/data.

### Reader

Device that communicates with a tag and retrieves or exchanges information.

### Passive tag

Tag that obtains operating energy from the RF field generated by a reader.

### Active tag

Tag containing its own power source.

### UID

Unique Identifier associated with a contactless tag/card implementation. Its security significance depends on the specific technology and system design.

### NDEF

**NFC Data Exchange Format**; standardized format commonly used to store and exchange application data on NFC tags.

### Card emulation

NFC operating mode in which a device behaves like a contactless card.

### Reader/Writer mode

NFC operating mode in which a device communicates with NFC tags.

### Peer-to-peer

NFC mode allowing two NFC devices to exchange data.

### Proximity

Communication intended for relatively short physical distances.

### Credential

Information or object used to establish identity or authorization.

### Cloning

Creating another credential that reproduces relevant properties of an original credential.

### Replay

Reusing previously captured communication or authentication information.

### Emulation

Making a device behave like another NFC/RFID device or credential.

### Access control

System that decides whether a person/device is permitted to access a resource.

### NDEF record

A structured piece of data contained within an NDEF message.

---

## NFC vs RFID

| Characteristic    | NFC                                         | RFID                                                    |
| ----------------- | ------------------------------------------- | ------------------------------------------------------- |
| Scope             | Specific short-range technology family      | Broad identification technology                         |
| Typical frequency | 13.56 MHz                                   | Multiple frequency ranges                               |
| Range             | Very short                                  | Can range from short to much longer depending on system |
| Power             | Can involve passive tags or powered devices | Passive and active systems                              |
| Common uses       | Payments, access, smartphones, tags         | Tracking, inventory, access control                     |
| Communication     | Often bidirectional                         | Depends on RFID system                                  |
| Security focus    | Cards, credentials, applications            | Tags, identifiers, readers, backend systems             |

---

# Section 4 — Tools and commands

| Tool               | Command         | What it finds/shows                                  | When to use it              |
| ------------------ | --------------- | ---------------------------------------------------- | --------------------------- |
| `nfc-list`         | `nfc-list`      | Detects and displays supported NFC tag information   | Basic NFC tag discovery     |
| `nfc-poll`         | `nfc-poll`      | Polls for an NFC tag and reports detected properties | Inspecting a nearby tag     |
| `nfc-mfclassic`    | `nfc-mfclassic` | Works with compatible MIFARE Classic cards/tags      | Later-stage card assessment |
| `rfidler`          | `rfidler`       | RFID analysis/emulation platform                     | Advanced RFID research      |
| `libnfc` utilities | `nfc-list -v`   | More verbose NFC/tag information                     | Detailed tag enumeration    |

### Example: detect an NFC tag

```text
$ nfc-list
NFC device: PN532
ISO/IEC 14443A target detected
```

**Interpretation:** An NFC reader has detected a compatible ISO/IEC 14443A target.

### Example: poll for a tag

```text
$ nfc-poll
ISO/IEC 14443A
UID: XX XX XX XX
```

**Interpretation:** A nearby compatible tag/card responded and exposed identification information.

### Example: verbose enumeration

```text
$ nfc-list -v
```

**Interpretation:** Useful when you need more detailed information about the detected NFC target and supported properties.

### Example: RFID research platform

```text
$ rfidler
```

**Interpretation:** `Rfidler` is intended for deeper RFID experimentation and research. Its relevance becomes much greater in the dedicated NFC/RFID exploitation stage.

---

# Section 5 — Defender detection

* **Reader logs:** Access-control readers can record credential presentations, successful authentications, failures, and unusual access patterns.
* **Repeated credential attempts:** Multiple failed or abnormal presentations can indicate reconnaissance or credential testing.
* **Unexpected access locations:** A credential appearing at physically impossible or suspicious locations can indicate credential misuse.
* **Backend correlation:** Correlating reader events with employee/device identity can expose anomalous usage.
* **Physical monitoring:** Cameras and access-control telemetry can complement digital logs because proximity attacks require physical access.
* **Credential inventory:** Unknown, duplicated, expired, or unauthorized credentials should be detected and revoked.
* **Strong cryptographic credentials:** Systems should avoid relying solely on easily reproducible identifiers where stronger authentication is available.

---

# Section 6 — Lab task

**Objective:** Use an NFC-capable reader and a tag you own to identify the NFC technology, retrieve its publicly exposed identification/data, and document what the reader can observe.

### Steps

1. Connect an NFC reader supported by your Kali system.
2. Place your own NFC tag/card near the reader.
3. Confirm that the reader detects a target.
4. Record the detected technology and tag identifier.
5. Check whether the tag contains an NDEF message.
6. Record any publicly readable information.
7. Classify the tag as informational, identification-oriented, or security/credential-related.
8. Save the terminal output and observations without attempting cloning or authentication attacks.

### Expected output

```text
NFC Target
├── Technology: ISO/IEC 14443A
├── UID: XX XX XX XX
├── NDEF: Present/Absent
└── Purpose: Test tag
```

### Git artifacts

```text
nfc-rfid-awareness/
├── README.md
├── notes/
│   └── nfc-vs-rfid.md
└── evidence/
    └── nfc-tag-enumeration.txt
```

Commit:

```bash
git add nfc-rfid-awareness/
git commit -m "Add NFC and RFID awareness lab"
```

---

# Section 7 — Common mistakes

1. **Treating NFC and RFID as synonyms** → RFID is a broader technology family → Understand NFC as a specific short-range ecosystem within the broader contactless/RF landscape.

2. **Assuming every RFID tag is passive** → Active RFID exists and behaves differently → Determine how the tag is powered.

3. **Assuming a UID is automatically a secure credential** → An identifier and cryptographic authentication are different things → Determine what the access-control system actually validates.

4. **Assuming short range means secure** → Physical proximity can still be obtained or deliberately approached → Evaluate the credential and authentication mechanism.

5. **Confusing NFC data with authentication** → An NDEF URL or text record is not inherently an authentication credential → Identify the actual security dependency.

6. **Jumping directly to cloning/replay** → Without understanding the technology, you may attack the wrong protocol or credential → First classify the tag, reader, protocol, and security model.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Classify:** Given a wireless credential system, distinguish **NFC, RFID, passive RFID, and active RFID** and explain the practical differences.

2. **Map:** Given an NFC access-control system, identify the roles of **tag/card → reader → access controller → backend → physical resource**.

3. **Assess:** Given a discovered NFC/RFID credential, determine whether the finding is merely an identifier, application data, or a security-critical authentication credential—and explain why that distinction matters before proceeding to **Part 21 Stage 6**.
