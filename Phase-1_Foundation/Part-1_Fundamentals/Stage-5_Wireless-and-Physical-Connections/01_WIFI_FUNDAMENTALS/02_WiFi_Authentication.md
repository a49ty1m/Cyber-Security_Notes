# WiFi Authentication

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → WiFi Fundamentals → WiFi Authentication & Encryption

# Section 1 — What it is and where it sits

WiFi authentication is the process by which a wireless client and an access point establish whether the client is allowed to join the WLAN and, for secured networks, establish cryptographic material for protecting traffic. The important distinction is that **authentication, association, key establishment, and encryption are related but separate operations**.

For offensive security, this topic produces an understanding of what an attacker can observe during wireless authentication, what information a handshake exposes, and why **Open, WEP, WPA/WPA2-Personal, WPA2-Enterprise, and WPA3** have very different security properties.

```text
802.11 Fundamentals
       ↓
SSID / BSSID / Channel
       ↓
Authentication & Association
       ↓
Key Establishment
       ↓
Encryption
       ↓
Protected Wireless Traffic
       ↓
Wireless Security Assessment
```

If you underestimate this topic, you will confuse an open network with an unauthenticated protocol, misunderstand what a WPA handshake actually proves, or assume that capturing a handshake means you have captured the user's password.

This builds directly on the previous WiFi fundamentals topic and prepares you for wireless security assessment, authentication attacks, and analysis of captured 802.11 traffic.

# Section 2 — How attackers actually use this

## 2.1 First classify the security model

The first question is not:

> "Can I crack this WiFi?"

It is:

> "What authentication and key-management architecture is this network using?"

An attacker maps the network into one of several broad categories:

```text
WiFi Security
│
├── Open
│
├── WEP
│
├── WPA/WPA2-Personal
│     └── Pre-shared key / passphrase
│
├── WPA/WPA2-Enterprise
│     └── 802.1X / EAP
│
└── WPA3
      ├── Personal → SAE
      └── Enterprise → 802.1X/EAP
```

This classification determines what later attacks are even relevant.

---

## 2.2 Open WiFi

An open network does not require a WiFi password for admission.

Conceptually:

```text
Client
   │
   │ Association
   ▼
Access Point
   │
   └── No WiFi authentication credential required
```

That does **not** mean the network has no security whatsoever.

Higher-layer protocols can still provide encryption:

```text
WiFi:       Open
Transport:  TLS
Application: HTTPS
```

An attacker near an open AP can potentially observe wireless traffic that is not protected by higher-layer encryption.

The useful finding is therefore:

```text
"Wireless link does not provide encryption."
```

not:

```text
"Everything transmitted is automatically readable."
```

HTTPS, SSH, VPNs, and other encrypted protocols can still protect application traffic.

---

## 2.3 WEP

WEP is an obsolete wireless security mechanism based on RC4 and a relatively small initialization vector.

Its fundamental design weakness is that the IV is too small and reused frequently.

Conceptually:

```text
Secret key
    +
Initialization Vector
    ↓
RC4 keystream
    ↓
Encrypted frame
```

Repeated IVs provide attackers with statistical information about the underlying RC4 keystream.

Therefore WEP does not fail because someone simply guesses the password.

It fails because the **cryptographic design allows practical key recovery after sufficient traffic is observed**.

A WEP network should therefore be treated as fundamentally compromised.

---

## 2.4 WPA introduced TKIP

WPA was created as an interim improvement over WEP.

Its major improvement was **TKIP — Temporal Key Integrity Protocol**.

Instead of relying on WEP's static approach, TKIP introduced mechanisms including:

* per-packet key mixing
* larger packet sequence information
* stronger integrity protection
* temporal keys

Conceptually:

```text
Master Key
    ↓
TKIP key mixing
    ↓
Per-packet cryptographic material
    ↓
RC4 encryption
```

TKIP still relies on RC4 and is now considered obsolete.

Modern networks should not be configured around TKIP.

---

## 2.5 WPA2 changed the cryptographic baseline

WPA2 commonly uses **CCMP**, based on AES.

This is a major architectural improvement over WEP and TKIP.

For WPA2-Personal:

```text
WiFi Passphrase
       ↓
PSK / derived key material
       ↓
4-way handshake
       ↓
Session keys
       ↓
AES-CCMP
       ↓
Encrypted traffic
```

An attacker who captures a WPA2-Personal handshake does **not** immediately obtain the password.

Instead, the captured exchange provides material that can be used to test password guesses offline.

That distinction is critical.

---

## 2.6 Understand what the WPA2 4-way handshake actually accomplishes

The 4-way handshake allows the client and AP to prove possession of the required key material and derive fresh session keys.

Simplified:

```text
AP                              Client
│                                  │
│──── Message 1: ANonce ──────────>│
│                                  │
│<─── Message 2: SNonce + MIC ─────│
│                                  │
│──── Message 3: key information ─>│
│                                  │
│<──── Message 4: confirmation ────│
│                                  │
└────── Encrypted session ─────────┘
```

Important values include:

* **ANonce** — nonce generated by the AP
* **SNonce** — nonce generated by the client
* **MIC** — Message Integrity Code
* **PMK** — Pairwise Master Key
* **PTK** — Pairwise Transient Key
* **GTK** — Group Temporal Key

The PTK is derived from key material plus values such as the nonces and MAC addresses.

The attacker wants enough of this exchange to validate password guesses offline.

---

## 2.7 WPA2-Personal versus WPA2-Enterprise

This distinction is extremely important.

### WPA2-Personal

Usually uses a shared passphrase.

```text
Users
  ↓
Same WiFi passphrase
  ↓
WPA2-Personal AP
```

The main password-related attack surface is the strength of that shared secret.

### WPA2-Enterprise

Uses an authentication framework based on **802.1X/EAP**.

```text
Client
   ↓
AP
   ↓
Authentication infrastructure
   ↓
RADIUS
   ↓
Identity system
```

Different users can authenticate using individual credentials or certificates depending on the EAP configuration.

Therefore:

```text
WPA2-Personal
→ shared secret

WPA2-Enterprise
→ per-user / enterprise authentication
```

They should never be treated as interchangeable.

---

## 2.8 WPA3 changed Personal authentication

WPA3-Personal replaces WPA2-Personal's traditional PSK handshake model with **SAE — Simultaneous Authentication of Equals**.

The conceptual model becomes:

```text
Client                    AP
  │                        │
  │──── SAE exchange ─────>│
  │<──── SAE exchange ─────│
  │                        │
  │── key establishment ──>│
  │                        │
  └── encrypted traffic ───┘
```

SAE provides stronger protection against offline password-guessing attacks than the traditional WPA2-Personal PSK approach.

That does **not** mean a weak password magically becomes strong against every possible attack.

It means the authentication protocol is designed to make certain offline attacks substantially harder.

---

## 2.9 Identify the encryption algorithm separately

Authentication and encryption are different questions.

For example:

```text
Authentication:
WPA2-Personal

Encryption:
CCMP / AES
```

Or:

```text
Authentication:
WPA

Encryption:
TKIP
```

The attacker therefore records both.

The important encryption mechanisms are:

| Mechanism | Cryptographic basis | Security status                    |
| --------- | ------------------- | ---------------------------------- |
| WEP       | RC4                 | Broken / obsolete                  |
| TKIP      | RC4-based           | Obsolete                           |
| CCMP      | AES                 | Strong modern baseline             |
| GCMP      | AES-GCM             | Modern high-performance protection |

---

## 2.10 GCMP and modern WiFi

**GCMP — Galois/Counter Mode Protocol** uses AES-GCM.

It combines:

```text
AES
+
Counter mode encryption
+
Galois-based authentication
```

Modern high-throughput WiFi generations can use GCMP because it provides authenticated encryption efficiently.

The key point for an attacker is not merely:

> "GCMP means impossible to attack."

Instead:

> The cryptographic protection is substantially stronger than WEP/TKIP, so assessment should focus on the authentication architecture, credential management, implementation flaws, configuration weaknesses, and endpoint behavior rather than expecting the cipher itself to be trivially broken.

---

## 2.11 What does a captured handshake actually reveal?

A captured WPA2-Personal exchange may reveal information such as:

```text
SSID
BSSID
Client MAC
AP MAC
Nonces
EAPOL frames
MIC
Protocol parameters
```

It does **not** directly reveal:

```text
WiFi password
```

The attacker can instead test candidate passwords against the captured exchange.

Conceptually:

```text
Captured handshake
        +
Candidate password
        ↓
Derived key material
        ↓
Calculate expected authentication value
        ↓
Compare with captured MIC
        ↓
Match?
```

Therefore password strength matters enormously for WPA2-Personal.

---

## 2.12 Dead-end finding vs high-value finding

### Dead-end finding

```text
SSID: Cafe-WiFi
Security: WPA2
Signal: -82 dBm
```

You know the network uses WPA2, but the weak signal and lack of relevance make this mostly environmental information.

There is no reason to waste time attacking an unrelated AP.

### High-value finding

```text
SSID: Corp-Lab
Security: WPA2-Personal
Cipher: CCMP
Strong signal
Authorized test target
```

Now the assessment can legitimately continue toward:

```text
Authentication analysis
        ↓
Handshake capture
        ↓
Credential-strength assessment
        ↓
Password policy findings
```

The valuable discovery is not simply "WPA2 exists."

It is identifying **the exact authentication architecture and its relevant attack surface**.

---

## 2.13 Where the results feed next

The output of this topic determines the next branch:

```text
Wireless Discovery
       ↓
Security Classification
       │
       ├── Open
       │     ↓
       │   Traffic protection analysis
       │
       ├── WEP
       │     ↓
       │   Legacy cryptographic assessment
       │
       ├── WPA/WPA2-Personal
       │     ↓
       │   Handshake / credential-strength assessment
       │
       ├── WPA2-Enterprise
       │     ↓
       │   802.1X / EAP assessment
       │
       └── WPA3
             ↓
           SAE / enterprise authentication assessment
```

This is why authentication knowledge must come before attempting wireless attacks.

# Section 3 — Core concepts and terminology

| Term                | Plain meaning                                                                 |
| ------------------- | ----------------------------------------------------------------------------- |
| **Authentication**  | Process used to establish whether a device/user can obtain network access.    |
| **Association**     | Process through which a client joins an AP's BSS.                             |
| **Encryption**      | Transformation that protects data from unauthorized reading.                  |
| **WEP**             | Legacy wireless security protocol that is cryptographically broken.           |
| **WPA**             | Transitional security standard introduced to replace WEP.                     |
| **WPA2**            | Mature WiFi security standard commonly using AES-CCMP.                        |
| **WPA3**            | Newer WiFi security generation with improved authentication mechanisms.       |
| **PSK**             | Pre-Shared Key; shared secret used in personal WiFi authentication.           |
| **SAE**             | Simultaneous Authentication of Equals; WPA3-Personal authentication protocol. |
| **TKIP**            | WPA-era key-management/encryption protocol based on RC4.                      |
| **CCMP**            | AES-based authenticated encryption protocol widely used by WPA2.              |
| **GCMP**            | AES-GCM-based authenticated encryption protocol used by modern WiFi.          |
| **AES**             | Advanced Encryption Standard.                                                 |
| **RC4**             | Stream cipher used by WEP and TKIP; now obsolete.                             |
| **Nonce**           | Value intended to be used once in a cryptographic exchange.                   |
| **ANonce**          | AP-generated nonce in the WPA 4-way handshake.                                |
| **SNonce**          | Client-generated nonce in the WPA 4-way handshake.                            |
| **MIC**             | Message Integrity Code used to authenticate handshake information.            |
| **PMK**             | Pairwise Master Key from which session-specific keys are derived.             |
| **PTK**             | Pairwise Transient Key used for unicast traffic protection.                   |
| **GTK**             | Group Temporal Key used for group/broadcast traffic.                          |
| **EAP**             | Extensible Authentication Protocol framework for authentication methods.      |
| **802.1X**          | Port-based network access-control architecture used by enterprise WiFi.       |
| **RADIUS**          | Common backend protocol used by enterprise authentication systems.            |
| **EAPOL**           | EAP over LAN; carries EAP authentication messages over the local link.        |
| **4-way handshake** | WPA/WPA2 key-establishment exchange between client and AP.                    |
| **Enterprise WiFi** | WiFi using 802.1X/EAP-based authentication rather than a shared PSK.          |

### Authentication and encryption mapping

| WiFi            | Typical authentication            | Common protection           |
| --------------- | --------------------------------- | --------------------------- |
| Open            | None                              | None at WiFi layer          |
| WEP             | Shared-key/open-system mechanisms | WEP/RC4                     |
| WPA-Personal    | PSK                               | TKIP                        |
| WPA2-Personal   | PSK                               | CCMP/AES                    |
| WPA2-Enterprise | 802.1X/EAP                        | CCMP/AES                    |
| WPA3-Personal   | SAE                               | AES-based modern protection |
| WPA3-Enterprise | 802.1X/EAP                        | AES-based modern protection |

### The critical mental model

```text
Authentication
     ≠
Encryption
     ≠
Association
     ≠
Password

Example:

WPA2-Personal
     ↓
PSK authentication architecture
     ↓
4-way handshake
     ↓
PTK / GTK
     ↓
CCMP/AES
     ↓
Protected traffic
```

# Section 4 — Tools and commands

| Tool          | Command                                                                       | What it finds/shows                                              | When to use it                                |
| ------------- | ----------------------------------------------------------------------------- | ---------------------------------------------------------------- | --------------------------------------------- |
| `nmcli`       | `nmcli -f SSID,BSSID,CHAN,FREQ,SECURITY device wifi list`                     | AP security advertisements                                       | Quickly classify nearby networks              |
| `iw`          | `sudo iw dev wlan0 scan`                                                      | Detailed AP information and security-related capabilities        | Inspect authorized AP advertisements          |
| `airodump-ng` | `sudo airodump-ng wlan0mon`                                                   | APs, clients, channels, encryption/authentication indicators     | Wireless assessment and handshake observation |
| `airodump-ng` | `sudo airodump-ng --bssid AA:BB:CC:DD:EE:FF --channel 36 -w capture wlan0mon` | Focused capture for one authorized AP/channel                    | Capture authentication traffic in a lab       |
| `tshark`      | `tshark -r capture-01.cap -Y "eapol"`                                         | EAPOL frames from a capture                                      | Verify whether a WPA handshake is present     |
| `Wireshark`   | `wireshark capture-01.cap`                                                    | GUI packet analysis                                              | Detailed 802.11/EAPOL inspection              |
| `aircrack-ng` | `aircrack-ng capture-01.cap`                                                  | Determines whether a capture contains crackable WPA/WEP material | Validate capture structure in a lab           |

### `nmcli`

```text
$ nmcli -f SSID,BSSID,CHAN,FREQ,SECURITY device wifi list

SSID       BSSID              CHAN  FREQ      SECURITY
Lab-WiFi   AA:BB:CC:11:22:33   36   5180 MHz  WPA2
Guest      DD:EE:FF:44:55:66    6   2437 MHz  WPA2
```

Interpretation:

```text
Lab-WiFi
→ WPA2 advertised
→ 5180 MHz
→ Channel 36
```

The scan alone does not tell you the user's password.

---

### `iw dev wlan0 scan`

```text
$ sudo iw dev wlan0 scan

BSS aa:bb:cc:11:22:33
    SSID: Lab-WiFi
    freq: 5180
    signal: -42.00 dBm
```

This provides lower-level wireless information than a simple network-manager listing.

---

### `airodump-ng`

```text
$ sudo airodump-ng wlan0mon

BSSID              PWR  CH  ENC   CIPHER  AUTH  ESSID
AA:BB:CC:11:22:33 -42  36  WPA2  CCMP    PSK   Lab-WiFi
```

The important interpretation is:

```text
ENC    → WPA2
CIPHER → CCMP
AUTH   → PSK
```

That tells you the target is using WPA2-Personal-style authentication with CCMP protection.

---

### Focused `airodump-ng` capture

```text
$ sudo airodump-ng \
  --bssid AA:BB:CC:11:22:33 \
  --channel 36 \
  -w capture \
  wlan0mon
```

A successful authorized lab capture may show:

```text
BSSID              PWR  CH  ENC   CIPHER  AUTH  ESSID
AA:BB:CC:11:22:33 -40  36  WPA2  CCMP    PSK   Lab-WiFi

STATION            PWR   BSSID
11:22:33:44:55:66  -38   AA:BB:CC:11:22:33
```

The AP/client relationship is now visible.

---

### `tshark`

```text
$ tshark -r capture-01.cap -Y "eapol"
```

Example:

```text
1  0.000000  AP → Client  EAPOL
2  0.014221  Client → AP  EAPOL
3  0.020514  AP → Client  EAPOL
4  0.029813  Client → AP  EAPOL
```

Four EAPOL frames strongly indicate that the WPA 4-way handshake was captured.

Do not interpret this as:

```text
"Password captured."
```

It means:

```text
"Authentication exchange captured."
```

---

### Wireshark

```text
$ wireshark capture-01.cap
```

Useful display filters include:

```text
eapol
```

and:

```text
wlan.fc.type_subtype == 0x08
```

The first isolates EAPOL traffic.

The second identifies beacon frames.

In the packet details, inspect fields such as:

```text
802.1X Authentication
Key Information
Replay Counter
Nonce
Key MIC
```

---

### `aircrack-ng`

```text
$ aircrack-ng capture-01.cap
```

A WPA-related capture may produce output indicating:

```text
WPA (1 handshake)

SSID: Lab-WiFi
```

This verifies that the capture contains material recognized as a WPA handshake.

For an authorized lab, you can then assess the strength of the lab password against a controlled wordlist.

# Section 5 — Defender detection

* **AP/controller authentication logs:** Repeated association and authentication failures can reveal clients attempting to connect with incorrect credentials.
* **EAPOL monitoring:** Wireless IDS/IPS systems can identify unusual authentication activity, repeated handshakes, and abnormal client/AP behavior.
* **Deauthentication monitoring:** Unexpected bursts of deauthentication/disassociation frames can indicate attempts to force clients to reconnect and generate authentication exchanges.
* **RADIUS logs:** Enterprise WiFi provides much richer authentication telemetry because authentication attempts are commonly visible through the RADIUS infrastructure.
* **Common miss:** Defenders sometimes treat a successful WPA2 handshake capture as proof that the password was stolen. The handshake contains authentication material, not the plaintext PSK.
* **Footprint reduction:** An operator trying to stay unobtrusive favors passive observation over unnecessary active authentication or disruption, because forced reconnections and repeated authentication attempts generate detectable wireless events.
* **Configuration detection:** Controllers should flag deprecated WEP/TKIP configurations and unexpected downgrade/legacy compatibility behavior.

# Section 6 — Lab task

**Platform:** Local Kali VM + a second WiFi-capable test device + your own lab AP configured with WPA2-Personal.

**Objective:** Capture and identify a WPA2 4-way handshake and distinguish the authentication exchange from the actual WiFi password.

**Steps:**

1. Configure your lab AP as WPA2-Personal using a deliberately controlled test password.
2. Connect the test client to the AP and verify normal connectivity.
3. Put the authorized Kali adapter into monitor mode using your normal wireless assessment setup.
4. Identify the AP's BSSID and operating channel.
5. Capture traffic focused on that AP and channel while the test client reconnects.
6. Stop the capture after the client has completed authentication.
7. Inspect the capture with Wireshark or `tshark` and filter for EAPOL.
8. Identify the four handshake messages and inspect their nonce/MIC/key-information fields.
9. Verify that the capture is recognized as containing a WPA handshake.
10. Record the SSID, BSSID, cipher, authentication type, and handshake evidence in Markdown.

**Expected output:**

```text
SSID:       Lab-WiFi
Security:   WPA2-Personal
Cipher:     CCMP
Authentication: PSK

EAPOL:
Message 1 → AP → Client
Message 2 → Client → AP
Message 3 → AP → Client
Message 4 → Client → AP
```

Success means you can point to the exchange in the capture and explain:

```text
"These packets prove that the WPA2 authentication/key-establishment
exchange occurred; they do not contain the plaintext WiFi password."
```

**Git artifact:**

```text
wifi-authentication/
├── README.md
├── notes/
│   └── wpa2-handshake.md
└── evidence/
    └── eapol-summary.txt
```

Do **not** commit real passwords or captures containing sensitive third-party traffic.

```bash
git add wifi-authentication/
git commit -m "Add WPA2 authentication handshake lab"
```

# Section 7 — Common mistakes

1. **Thinking a handshake is the WiFi password** → a WPA2 handshake contains cryptographic exchange material, not the plaintext PSK → learn to identify EAPOL frames and their purpose.

2. **Treating WPA, WPA2, and WPA3 as the same thing** → their authentication and cryptographic mechanisms differ significantly → identify the exact generation and authentication mode.

3. **Confusing authentication with encryption** → WPA2-Personal describes an authentication/key-management architecture while CCMP describes traffic protection → record both separately.

4. **Calling TKIP modern encryption** → TKIP is an obsolete transitional mechanism → treat WEP and TKIP configurations as legacy findings.

5. **Assuming WPA3 means "nothing can be attacked"** → WPA3 improves authentication, but implementations, credentials, enterprise configuration, clients, and surrounding infrastructure can still contain weaknesses → assess the complete architecture.

6. **Ignoring WPA2-Enterprise** → enterprise WiFi does not revolve around one shared PSK → identify 802.1X/EAP and the backend authentication infrastructure before choosing an assessment path.

7. **Capturing packets without understanding what they represent** → collecting a `.cap` file is not analysis → identify the AP, client, EAPOL exchange, authentication model, and cipher before drawing conclusions.

# Section 8 — Move-on gate

1. **Capture a WPA2-Personal handshake from your lab AP and identify all four EAPOL messages, including which side sent each message, without looking at your notes.**

2. **Inspect a wireless scan and correctly classify three networks as Open, WPA2-Personal/CCMP, or WPA2-Enterprise, then identify the authentication mechanism and encryption method for each.**

3. **Given a packet capture containing a WPA2 handshake, locate the ANonce, SNonce, MIC, and key-information fields in Wireshark and explain what each field contributes to the key-establishment process without referring to your notes.**
