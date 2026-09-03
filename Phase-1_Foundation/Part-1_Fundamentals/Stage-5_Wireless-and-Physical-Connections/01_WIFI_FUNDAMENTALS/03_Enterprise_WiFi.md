# Enterprise WiFi: EAP Variants & RADIUS

**Roadmap:** Part 1 — Fundamentals → Stage 5 — Wireless & Physical Connections → WiFi Fundamentals → Enterprise WiFi → EAP Variants (PEAP, TTLS, TLS) & RADIUS Architecture

# Section 1 — What it is and where it sits

Enterprise WiFi replaces the single shared password model of WPA2/WPA3-Personal with **802.1X-based authentication**. A wireless client authenticates through an EAP method, while the access point acts primarily as an authenticator and forwards authentication traffic to a backend **RADIUS server**. The RADIUS server decides whether the identity should be accepted and can return authorization attributes such as VLAN assignment.

The critical security boundary is therefore not just the AP. It is:

```text
Client
  ↓
802.11 Association
  ↓
Access Point
  ↓
802.1X / EAP
  ↓
RADIUS
  ↓
Identity / Authentication Backend
  ↓
Accept / Reject + Authorization
```

If you skip this, enterprise wireless testing becomes guesswork. You may attack the wrong component, misunderstand PEAP/TTLS/TLS, or mistake an encrypted outer tunnel for certificate-based client authentication.

This builds on WiFi authentication and encryption by moving from **shared PSKs** to **identity-based 802.1X/EAP authentication**, which leads directly into enterprise wireless assessment.

# Section 2 — How attackers actually use this

## 2.1 Map the enterprise authentication architecture

An attacker first determines what kind of enterprise authentication is being used.

They want to establish:

* SSID
* BSSID
* WPA2-Enterprise or WPA3-Enterprise
* EAP method
* outer identity behavior
* inner authentication method
* server certificate behavior
* authentication server
* RADIUS infrastructure
* whether credentials or certificates are used
* whether multiple SSIDs use different authentication policies

The important architecture is:

```text
                    Enterprise WLAN
                         │
                         ▼
Client ────────► Access Point
                  Authenticator
                       │
                       │ RADIUS
                       ▼
                RADIUS Server
                       │
                       ▼
              Identity Backend
              ┌────────┴────────┐
              │                 │
           LDAP/AD           Local DB
```

The AP does not normally need to understand every EAP method internally. It transports authentication information between the wireless client and the authentication infrastructure.

---

## 2.2 Understand the roles before analyzing EAP

There are three important 802.1X roles.

### Supplicant

The **supplicant** is the client attempting to obtain network access.

Examples:

```text
Laptop
Phone
Corporate workstation
IoT endpoint
```

### Authenticator

The **authenticator** controls access to the network.

In WiFi this is normally:

```text
Wireless AP
```

### Authentication Server

The authentication server validates credentials and makes the authentication decision.

Commonly:

```text
RADIUS server
```

Therefore:

```text
Client        = Supplicant
AP            = Authenticator
RADIUS        = Authentication Server
```

This three-role model is essential when tracing an enterprise authentication failure.

---

## 2.3 Determine whether the EAP method protects credentials

This is where an attacker becomes interested.

Not all EAP methods provide the same protection.

The key question is:

> "Does the method establish a protected tunnel before transmitting the user's actual authentication material?"

For example:

```text
PEAP
│
├── Outer TLS tunnel
│       ↓
└── Inner authentication
```

Whereas:

```text
EAP-TLS
│
└── Client certificate authentication
```

The distinction matters because a username/password deployment and a certificate-based deployment present very different attack surfaces.

---

## 2.4 PEAP

**PEAP — Protected Extensible Authentication Protocol** establishes a TLS-protected outer tunnel and then performs an inner authentication method.

Conceptually:

```text
Client                         RADIUS
  │                              │
  │──── TLS negotiation ─────────>│
  │<─── Server certificate ───────│
  │                              │
  │════ Encrypted TLS tunnel ════│
  │                              │
  │──── Inner authentication ────│
  │                              │
  └──────── Authentication ──────┘
```

Common deployments use:

```text
PEAP
  +
MSCHAPv2
```

The attacker therefore cares heavily about **certificate validation**.

If a client accepts an attacker's certificate without properly validating the trusted server identity, the attacker may be able to position themselves as a rogue authentication endpoint and collect authentication material.

This is one of the most important enterprise WiFi configuration failures to understand.

---

## 2.5 TTLS

**EAP-TTLS — EAP Tunneled TLS** is conceptually similar to PEAP.

First:

```text
TLS tunnel
```

Then:

```text
Inner authentication
```

The difference is primarily the supported inner authentication architecture and ecosystem.

Conceptually:

```text
Outer:
Client ───── TLS ───── RADIUS

Inside TLS:
Client ─── inner method ─── RADIUS
```

TTLS can support multiple inner authentication mechanisms.

Therefore, when assessing TTLS, identifying only:

```text
"EAP-TTLS"
```

is incomplete.

You also want:

```text
Outer method → EAP-TTLS
Inner method → ?
Certificate validation → ?
```

---

## 2.6 EAP-TLS

EAP-TLS takes a fundamentally different approach.

Instead of:

```text
Password
```

the client authenticates using a certificate and corresponding private key.

Conceptually:

```text
Client certificate
        +
Client private key
        ↓
TLS authentication
        ↓
RADIUS server
        ↓
Certificate validation
        ↓
Access granted
```

This creates a much stronger credential model when properly implemented.

A stolen username/password is not enough to authenticate to an EAP-TLS deployment.

The attacker would generally need access to an authorized client certificate/private-key combination or another weakness in the certificate enrollment, trust, endpoint, or authentication infrastructure.

---

## 2.7 Compare the three EAP methods

| Method           | Outer protection     | Typical credential model            | Main assessment concern                    |
| ---------------- | -------------------- | ----------------------------------- | ------------------------------------------ |
| **PEAP**         | TLS tunnel           | Password-based inner authentication | Certificate validation + inner method      |
| **EAP-TTLS**     | TLS tunnel           | Various inner methods               | Certificate validation + inner method      |
| **EAP-TLS**      | TLS                  | Client certificates                 | Certificate lifecycle, private keys, trust |
| **EAP-MSCHAPv2** | Usually inner method | Username/password                   | Credential resistance                      |
| **EAP-TLS**      | Certificate-based    | Client certificate/private key      | PKI and endpoint security                  |

The important distinction:

```text
PEAP / TTLS
→ TLS protects an inner authentication exchange

EAP-TLS
→ TLS itself authenticates the client using certificates
```

---

## 2.8 Understand the RADIUS transaction

The wireless client does not normally send a RADIUS request directly to the server.

The AP/controller acts as the RADIUS client.

```text
Client
  │
  │ EAP
  ▼
AP
  │
  │ RADIUS Access-Request
  ▼
RADIUS Server
  │
  ├── Access-Accept
  │
  ├── Access-Reject
  │
  └── Access-Challenge
```

The three important responses are:

### Access-Accept

Authentication succeeded.

```text
RADIUS
  ↓
Access-Accept
  ↓
AP allows network access
```

### Access-Reject

Authentication failed.

```text
RADIUS
  ↓
Access-Reject
  ↓
AP denies access
```

### Access-Challenge

More authentication interaction is required.

```text
RADIUS
  ↓
Access-Challenge
  ↓
AP continues EAP exchange
```

---

## 2.9 What attackers look for in RADIUS architecture

A useful architecture map might reveal:

```text
SSID: Corp-WiFi
        ↓
WPA2-Enterprise
        ↓
PEAP
        ↓
MSCHAPv2
        ↓
RADIUS
        ↓
Active Directory
```

That is far more valuable than simply knowing:

```text
"Corp-WiFi uses WPA2."
```

The attacker now understands where credentials are validated and which components deserve assessment.

Other useful findings include:

* multiple RADIUS servers
* authentication failover
* certificate authorities
* identity backend
* VLAN assignment
* accounting infrastructure
* server certificate configuration
* client trust configuration
* EAP method inconsistencies

---

## 2.10 Dead-end finding vs high-value finding

### Dead-end finding

```text
SSID: Corporate-WiFi
Security: WPA2-Enterprise
```

This establishes enterprise authentication but tells you almost nothing about the actual authentication path.

### High-value finding

```text
SSID: Corporate-WiFi
Security: WPA2-Enterprise
Outer EAP: PEAP
Inner EAP: MSCHAPv2
Server certificate: improperly validated by clients
Backend: RADIUS → AD
```

This reveals an actual attack surface:

```text
Enterprise WiFi
      ↓
PEAP
      ↓
Certificate trust weakness
      ↓
Potential credential interception opportunity
```

The important point is that the **configuration weakness**, rather than WPA2 itself, creates the interesting path.

---

## 2.11 Follow the pivot

Once the authentication architecture is identified:

```text
Wireless discovery
       ↓
Enterprise authentication
       ↓
EAP method
       ↓
Certificate validation
       ↓
Inner authentication
       ↓
RADIUS
       ↓
Identity backend
       ↓
Authorization / VLAN
```

For example:

```text
PEAP + MSCHAPv2
        ↓
Check client certificate validation
        ↓
Check inner authentication configuration
        ↓
Check RADIUS policy
        ↓
Check identity infrastructure
```

With EAP-TLS:

```text
EAP-TLS
   ↓
Certificate trust
   ↓
Certificate enrollment
   ↓
Private-key protection
   ↓
Revocation
   ↓
RADIUS certificate validation
```

The EAP method therefore determines where the assessment should go next.

# Section 3 — Core concepts and terminology

| Term                      | Meaning                                                                                                                                |
| ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| **802.1X**                | Port-based access-control framework used to control network access through authentication.                                             |
| **Supplicant**            | Client requesting network access.                                                                                                      |
| **Authenticator**         | Device controlling access, normally the AP/controller.                                                                                 |
| **Authentication Server** | Backend system that validates authentication, commonly RADIUS.                                                                         |
| **EAP**                   | Framework supporting multiple authentication methods.                                                                                  |
| **PEAP**                  | EAP method creating a TLS tunnel around an inner authentication method.                                                                |
| **EAP-TTLS**              | EAP method using a TLS tunnel with flexible inner authentication methods.                                                              |
| **EAP-TLS**               | EAP method using mutual certificate-based TLS authentication.                                                                          |
| **Inner method**          | Authentication mechanism carried inside PEAP/TTLS.                                                                                     |
| **Outer method**          | EAP method visible before the protected inner exchange.                                                                                |
| **RADIUS**                | Protocol commonly used to transport authentication and authorization information between network devices and an authentication server. |
| **Access-Request**        | RADIUS request asking the server to authenticate a client.                                                                             |
| **Access-Accept**         | RADIUS response granting authentication.                                                                                               |
| **Access-Reject**         | RADIUS response denying authentication.                                                                                                |
| **Access-Challenge**      | RADIUS response requesting additional authentication interaction.                                                                      |
| **MSCHAPv2**              | Challenge-response password authentication method commonly used inside PEAP.                                                           |
| **TLS**                   | Transport Layer Security providing encryption and authentication using certificates.                                                   |
| **PKI**                   | Public Key Infrastructure used to issue, manage, and validate certificates.                                                            |
| **CA**                    | Certificate Authority that issues or signs certificates.                                                                               |
| **Server certificate**    | Certificate used to prove the identity of the authentication server.                                                                   |
| **Client certificate**    | Certificate used to authenticate a client in methods such as EAP-TLS.                                                                  |
| **RADIUS client**         | Network device, such as an AP/controller, that communicates with the RADIUS server.                                                    |
| **RADIUS shared secret**  | Secret configured between the RADIUS server and its RADIUS clients.                                                                    |
| **VLAN assignment**       | Authorization mechanism allowing RADIUS to place an authenticated client into a particular VLAN.                                       |
| **EAPOL**                 | EAP carried over LAN, commonly used between wireless clients and APs.                                                                  |
| **AAA**                   | Authentication, Authorization, and Accounting.                                                                                         |

### Architecture variants

| Deployment        | Client               | Authenticator       | Backend               |
| ----------------- | -------------------- | ------------------- | --------------------- |
| Small enterprise  | Laptop               | AP                  | RADIUS                |
| Large enterprise  | Laptop               | WLAN controller/AP  | RADIUS cluster        |
| Directory-backed  | Laptop               | AP/controller       | RADIUS → AD/LDAP      |
| Certificate-based | Laptop + certificate | AP/controller       | RADIUS → PKI          |
| Cloud-managed     | Laptop               | AP/cloud controller | RADIUS/cloud identity |

### EAP decision tree

```text
Enterprise WiFi
      │
      ├── PEAP
      │    └── TLS tunnel → inner authentication
      │
      ├── TTLS
      │    └── TLS tunnel → inner authentication
      │
      └── EAP-TLS
           └── certificate-based client authentication
```

# Section 4 — Tools and commands

| Tool         | Command                                        | What it finds/shows                | When to use it                     |
| ------------ | ---------------------------------------------- | ---------------------------------- | ---------------------------------- |
| `nmcli`      | `nmcli connection show`                        | Existing enterprise WiFi profiles  | Inspect client configuration       |
| `nmcli`      | `nmcli connection show "Corp-WiFi"`            | EAP/security settings              | Identify configured authentication |
| `iw`         | `sudo iw dev wlan0 scan`                       | AP security advertisements         | Identify enterprise WLANs          |
| `tshark`     | `tshark -r capture.pcapng -Y "eapol"`          | EAPOL exchanges                    | Analyze authentication traffic     |
| `tshark`     | `tshark -r capture.pcapng -Y "eap"`            | EAP packets                        | Identify EAP conversations         |
| `tcpdump`    | `sudo tcpdump -ni any port 1812 or port 1813`  | RADIUS traffic                     | Observe a lab RADIUS exchange      |
| `ss`         | `sudo ss -lunp`                                | Listening UDP services             | Verify local RADIUS service        |
| `freeradius` | `sudo freeradius -X`                           | RADIUS authentication/debug events | Debug a lab RADIUS server          |
| `radtest`    | `radtest user password 127.0.0.1 0 testing123` | Test RADIUS authentication         | Validate a local lab server        |

### `nmcli`

```text
$ nmcli connection show "Corp-WiFi"

connection.id:                  Corp-WiFi
802-1x.eap:                    peap
802-1x.phase2-auth:            mschapv2
802-1x.ca-cert:                /etc/ssl/certs/ca.pem
```

Interpretation:

```text
Outer EAP: PEAP
Inner authentication: MSCHAPv2
Trusted CA: explicitly configured
```

This is exactly the kind of client-side configuration information that matters during an enterprise WiFi assessment.

---

### `iw`

```text
$ sudo iw dev wlan0 scan

BSS aa:bb:cc:11:22:33
    SSID: Corp-WiFi
    freq: 5180
    signal: -44.00 dBm
```

This confirms the AP's wireless presence.

The scan itself normally does **not** reveal the complete EAP method. Client configuration and authentication traffic may be required to identify it.

---

### `tshark`

```text
$ tshark -r capture.pcapng -Y "eapol"
```

Example:

```text
1  Client → AP  EAPOL
2  AP → Client    EAPOL
3  Client → AP  EAPOL
```

To focus on EAP:

```text
$ tshark -r capture.pcapng -Y "eap"
```

Example:

```text
EAP Identity
EAP Request
EAP Response
EAP-TLS
```

This can help determine which EAP conversation is taking place.

---

### `tcpdump`

On a controlled RADIUS lab:

```text
$ sudo tcpdump -ni any 'udp port 1812 or udp port 1813'
```

Example:

```text
IP 10.0.0.10.43012 > 10.0.0.20.1812: UDP
IP 10.0.0.20.1812 > 10.0.0.10.43012: UDP
```

Interpretation:

```text
10.0.0.10 → RADIUS client
10.0.0.20 → RADIUS server
1812      → authentication
```

RADIUS accounting commonly uses UDP 1813.

---

### `ss`

```text
$ sudo ss -lunp

UNCONN 0 0 0.0.0.0:1812  0.0.0.0:*
UNCONN 0 0 0.0.0.0:1813  0.0.0.0:*
```

This indicates that something is listening on the standard RADIUS UDP ports.

It does not by itself prove that the service is correctly configured.

---

### FreeRADIUS

Run the lab server in debug mode:

```text
$ sudo freeradius -X
```

Example:

```text
Ready to process requests
Received Access-Request
User-Name = "labuser"
...
Sent Access-Accept
```

Interpretation:

```text
Access-Request
     ↓
Credentials processed
     ↓
Access-Accept
```

This is extremely useful for understanding the RADIUS transaction because the server exposes its decision process.

---

### `radtest`

Against a deliberately configured local RADIUS lab:

```text
$ radtest labuser 'LabPassword123!' 127.0.0.1 0 testing123
```

Example:

```text
Sent Access-Request
Received Access-Accept
        User-Name = "labuser"
```

If authentication fails:

```text
Received Access-Reject
```

This allows you to validate the RADIUS backend independently of the WiFi layer.

# Section 5 — Defender detection

* **RADIUS logs:** Monitor Access-Request, Access-Accept, Access-Reject, and Access-Challenge activity for unusual authentication patterns.
* **AP/controller logs:** Correlate wireless client MAC addresses with authentication attempts and EAP failures.
* **Certificate monitoring:** Alert on expired, untrusted, unexpected, or incorrectly issued certificates used by enterprise authentication infrastructure.
* **EAP method monitoring:** Detect unexpected changes from EAP-TLS to password-based methods or unauthorized enterprise SSIDs using weaker configurations.
* **Common miss:** Organizations configure PEAP/TTLS but fail to ensure clients validate the RADIUS server certificate correctly.
* **Credential monitoring:** Repeated authentication failures against many accounts can indicate password spraying or rogue authentication infrastructure.
* **Operator footprint reduction:** Passive observation and configuration analysis create less authentication telemetry than repeatedly attempting to authenticate clients or disrupt an enterprise WLAN.

# Section 6 — Lab task

**Platform:** Kali Linux running a local FreeRADIUS server; no real enterprise credentials or production WLAN required.

**Objective:** Build a miniature 802.1X/RADIUS environment and trace an authentication attempt from client → authenticator → RADIUS → authentication decision.

**Steps:**

1. Install FreeRADIUS on the Kali lab VM.
2. Configure one test RADIUS client representing the AP.
3. Create a dedicated lab user with a test credential.
4. Start FreeRADIUS in foreground debug mode.
5. Use `radtest` to send a controlled authentication request.
6. Observe the RADIUS `Access-Request`.
7. Observe whether the server returns `Access-Accept` or `Access-Reject`.
8. Capture the local RADIUS exchange with `tcpdump`.
9. Record the roles of supplicant, authenticator, and authentication server.
10. Repeat once with an intentionally incorrect password and compare the server-side result.

**Expected output:**

Successful authentication:

```text
Access-Request
     ↓
User-Name = "labuser"
     ↓
Authentication succeeds
     ↓
Access-Accept
```

Failed authentication:

```text
Access-Request
     ↓
Authentication fails
     ↓
Access-Reject
```

Your final diagram should look like:

```text
Lab Client
   ↓
Supplicant
   ↓
Authenticator
   ↓
RADIUS Access-Request
   ↓
FreeRADIUS
   ↓
Access-Accept / Reject
```

**Git artifact:**

```text
enterprise-wifi/
├── README.md
├── notes/
│   └── radius-eap-architecture.md
└── evidence/
    └── radius-debug.txt
```

```bash
git add enterprise-wifi/
git commit -m "Add enterprise WiFi RADIUS authentication lab"
```

# Section 7 — Common mistakes

1. **Treating PEAP, TTLS, and TLS as interchangeable** → their authentication models differ → identify both outer and inner methods.

2. **Thinking the AP authenticates the user's password itself** → enterprise WiFi normally delegates authentication through the RADIUS architecture → trace the complete client → AP → RADIUS path.

3. **Ignoring certificate validation** → PEAP/TTLS security depends heavily on correctly validating the TLS server identity → verify trusted CA and expected server identity.

4. **Assuming EAP-TLS is just PEAP with certificates** → EAP-TLS uses certificate-based client authentication as the core authentication mechanism → analyze certificates, private keys, PKI, and revocation.

5. **Looking only at the SSID security label** → "WPA2-Enterprise" does not tell you whether the network uses PEAP, TTLS, or EAP-TLS → inspect client configuration and authentication traffic.

6. **Ignoring the RADIUS backend** → the wireless network is only one component of enterprise authentication → map RADIUS policies and the identity infrastructure behind them.

7. **Testing authentication without understanding the roles** → confusing supplicant, authenticator, and authentication server leads to bad troubleshooting and bad attack assumptions → explicitly map all three before proceeding.

# Section 8 — Move-on gate

1. **Build a local FreeRADIUS lab, submit one valid and one invalid authentication request, and correctly identify the resulting Access-Accept and Access-Reject messages without looking at your notes.**

2. **Inspect an authorized enterprise WiFi client configuration and identify the outer EAP method, inner authentication method, trusted CA, and server identity without looking at your notes.**

3. **Given a PEAP, TTLS, and EAP-TLS authentication trace, identify each method and correctly map the supplicant, authenticator, RADIUS server, TLS tunnel, and credential/certificate mechanism involved in each exchange.**
