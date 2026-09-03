# Mobile Security Concepts

**Roadmap:** Part 1 — Fundamentals → Stage 6 — Mobile Platform Awareness → Mobile Security Concepts

# Section 1 — What it is and where it sits

Mobile security depends on multiple layers rather than a single protection mechanism. Four concepts are especially important for a security tester: **rooting/jailbreaking**, **certificate pinning**, **biometric authentication**, and **hardware-backed key storage**.

These mechanisms protect different parts of the mobile security model. Rooting/jailbreaking changes the device's privilege/trust boundary; certificate pinning attempts to prevent unauthorized interception of an application's TLS connection; biometric authentication verifies a user through a biological characteristic; and hardware-backed keystores protect cryptographic keys using dedicated or isolated hardware security capabilities.

```text
Mobile Security
│
├── Device Trust
│   └── Rooting / Jailbreaking
│
├── Network Trust
│   └── Certificate Pinning
│
├── User Authentication
│   └── Biometrics
│
└── Cryptographic Key Protection
    └── Hardware-backed Keystore
```

The security-testing chain is:

```text
Mobile App
    ↓
Device security state
    ↓
Authentication controls
    ↓
Network trust
    ↓
Cryptographic key protection
    ↓
Application security
```

These concepts build directly on your Android/iOS architecture knowledge. They also prepare you for deeper mobile security testing without prematurely jumping into exploitation.

---

# Section 2 — How attackers actually use this

## 2.1 Rooting and jailbreaking change the trust boundary

**Rooting** generally refers to obtaining elevated privileges on Android.

**Jailbreaking** generally refers to bypassing or modifying restrictions imposed by iOS.

The exact techniques vary significantly by OS version and device.

Conceptually:

```text
Normal device

Application
    ↓
Sandbox
    ↓
OS security controls
    ↓
Hardware


Modified/compromised device

Application
    ↓
Additional privileges / altered controls
    ↓
OS
    ↓
Hardware
```

For a mobile security tester, a rooted/jailbroken test device can provide greater visibility into:

* Application files
* Private application storage
* Processes
* Runtime behavior
* System configuration
* Security controls
* Debugging interfaces

But this creates an important distinction:

> **A rooted/jailbroken device does not automatically mean the application itself is vulnerable.**

It changes the testing environment and potentially weakens device-level assumptions.

---

## 2.2 Why applications care about device integrity

Applications may make security decisions based on assumptions such as:

```text
Device
 ↓
Trusted OS state
 ↓
Protected application storage
 ↓
Protected credentials
```

If the device has been modified, those assumptions may no longer hold.

An application may therefore perform **root/jailbreak detection**.

Conceptually:

```text
Application
    ↓
Device integrity checks
    ↓
Normal device → continue
Modified device → restrict functionality
```

However, device integrity detection itself is not a complete security boundary.

A robust application should avoid assuming that:

> **“The client device will always behave honestly.”**

Critical authorization decisions should be enforced server-side where appropriate.

---

## 2.3 Certificate pinning protects application-level TLS trust

Normally, an application establishes a TLS connection and validates the server certificate against a trusted certificate authority chain.

Simplified:

```text
App
 ↓
TLS
 ↓
Server Certificate
 ↓
Certificate validation
 ↓
Trusted CA
```

**Certificate pinning** adds another layer of trust.

The application may effectively require the server's certificate or public-key identity to match an expected value.

```text
Server certificate
       ↓
Normal TLS validation
       ↓
Pin validation
       ↓
Connection accepted
```

This can make unauthorized TLS interception significantly harder.

---

## 2.4 Why certificate pinning matters to mobile testers

During legitimate application testing, testers often want to inspect:

```text
Application
    ↓
HTTPS request
    ↓
Proxy
    ↓
Internet/API
```

Without pinning, installing a test CA certificate may allow the testing proxy to inspect the TLS traffic.

With pinning:

```text
Application
     ↓
TLS interception
     ↓
Certificate differs from expected pin
     ↓
Connection rejected
```

This is why pinning is important during mobile application assessments.

The security question becomes:

> **Does the application correctly enforce its intended server identity?**

---

## 2.5 Pinning is not encryption

This distinction is critical.

**TLS encryption** protects communication from passive network observation.

**Certificate pinning** strengthens the application's confidence that it is communicating with the intended server.

```text
TLS
└── Protects communication

Pinning
└── Restricts which server identity the app accepts
```

Pinning does not replace TLS.

---

## 2.6 Biometric authentication

Mobile platforms can authenticate users through:

* Fingerprints
* Face recognition
* Other supported biometric mechanisms

Conceptually:

```text
User
 ↓
Biometric sensor
 ↓
Platform biometric subsystem
 ↓
Authentication result
 ↓
Application
```

A key security principle is:

> **The application should generally receive an authentication result rather than raw biometric data.**

The operating system and secure hardware components handle much of the sensitive biometric processing.

---

## 2.7 Biometrics are not equivalent to passwords

Biometrics have different properties from traditional secrets.

| Property              | Password              | Biometric                    |
| --------------------- | --------------------- | ---------------------------- |
| User memorizes it     | Yes                   | No                           |
| Can be changed easily | Yes                   | Generally no                 |
| Can be shared         | Easily                | Not intentionally            |
| Stored as raw value   | Should not be         | Platform-controlled          |
| Typical use           | Direct authentication | Convenient user verification |

A particularly important concept is **authentication assurance**.

A biometric unlock event may authorize access to a cryptographic key or application operation without the application ever receiving the user's fingerprint or face data.

---

## 2.8 Hardware-backed keystores

A **keystore** manages cryptographic keys used by applications.

A hardware-backed keystore provides stronger protection by storing or processing key material using dedicated hardware-backed security capabilities when supported.

Conceptually:

```text
Application
     │
     │ "Use this key"
     ↓
Keystore API
     ↓
Hardware-backed security
     ↓
Cryptographic operation
```

Instead of:

```text
Application
     ↓
Raw private key
     ↓
Application memory
```

the application can request a protected cryptographic operation.

---

## 2.9 Why hardware-backed keys matter

Suppose an application uses a private key for authentication.

Weak model:

```text
App
 ↓
Private key file
 ↓
Filesystem
```

If the application storage is compromised, the key may potentially be extracted.

Stronger model:

```text
App
 ↓
Keystore API
 ↓
Protected key
 ↓
Hardware-backed operation
```

The application can use the key without necessarily being able to retrieve the raw private key material.

This is particularly valuable for:

* Device authentication
* Token protection
* Cryptographic signing
* Secure application credentials
* Key-wrapping operations

---

## 2.10 Authentication + hardware-backed keys

These mechanisms can work together.

For example:

```text
User
 ↓
Biometric authentication
 ↓
Platform verifies user
 ↓
Protected key becomes usable
 ↓
Application signs operation
 ↓
Server verifies signature
```

The application does not need to store the private key as ordinary application data.

This is a much stronger architecture than simply storing a password/token inside an APK.

---

## 2.11 Attackers look for weak links between these mechanisms

A mobile tester should not examine each mechanism in isolation.

Consider:

```text
Biometric
    ↓
Unlock
    ↓
Keystore key
    ↓
API authentication
```

Questions include:

* Is biometric authentication actually required?
* Is the key hardware-backed?
* Can the application fall back to a weaker mechanism?
* Is the cryptographic key extractable?
* Does the server independently validate authorization?
* Does a rooted device change the security assumptions?
* Does TLS pinning protect sensitive API communication?

The highest-value findings frequently occur at the **boundaries between mechanisms**.

---

## 2.12 Dead-end vs high-value findings

| Finding                                                                      | Typical value            |
| ---------------------------------------------------------------------------- | ------------------------ |
| Rooted test device detected                                                  | Low                      |
| Application detects root/jailbreak                                           | Informational            |
| TLS used without pinning                                                     | Context-dependent        |
| Sensitive API accepts weak TLS configuration                                 | High                     |
| Biometric UI exists                                                          | Low                      |
| Sensitive action bypasses biometric requirement                              | Very High                |
| Private key stored as ordinary application data                              | High                     |
| Sensitive key protected by hardware-backed keystore                          | Strong defensive control |
| Authentication key remains usable after required user verification is absent | Very High                |

Do not label a missing security mechanism a vulnerability automatically.

Always ask:

> **What security property was lost, and what can an attacker do because of it?**

---

## 2.13 Where this feeds next

```text
Mobile architecture
       ↓
Device trust
       ├── Root / jailbreak state
       ↓
Application authentication
       ├── Biometrics
       ↓
Cryptographic protection
       ├── Hardware-backed keys
       ↓
Network trust
       ├── TLS + pinning
       ↓
Mobile application assessment
```

This provides the foundation for deeper mobile testing later.

---

# Section 3 — Core concepts and terminology

### Rooting

Obtaining elevated/root-level control over an Android device beyond normal application privileges.

### Jailbreaking

Modifying an iOS device to bypass or alter Apple's normal platform restrictions.

### Root Detection

Application techniques intended to identify potentially modified/rooted Android devices.

### Jailbreak Detection

Application techniques intended to identify potentially modified/jailbroken iOS devices.

### TLS

**Transport Layer Security**; cryptographic protocol used to protect network communications.

### Certificate

Cryptographically signed object binding an identity to a public key.

### Certificate Authority

Trusted entity that issues/signs certificates.

### Certificate Pinning

Application mechanism that restricts accepted server certificates/public keys to expected values.

### Public-Key Pinning

Pinning based on an expected public-key identity rather than necessarily an entire certificate.

### Biometric Authentication

User authentication using biological characteristics such as fingerprints or facial characteristics.

### Biometric Template

Protected representation derived from biometric characteristics and used by a biometric system.

### Keystore

System-managed facility for storing and using cryptographic keys.

### Hardware-Backed Keystore

Keystore functionality where key protection and/or cryptographic operations are supported by dedicated hardware security capabilities.

### Secure Element

Dedicated hardware component designed to securely store/process sensitive credentials or cryptographic operations.

### TEE

**Trusted Execution Environment**; isolated execution environment designed to protect security-sensitive code/data.

### Key Material

Information used by cryptographic algorithms, including secret/private keys.

### Key Extraction

Obtaining protected cryptographic key material in a form that allows independent use.

### Authentication

Establishing the identity of a user/device/process.

### Authorization

Determining what an authenticated entity is allowed to do.

### Device Integrity

Confidence that the operating system/device has not been improperly modified.

### Trust Boundary

Boundary separating components with different security assumptions or privileges.

---

## Mechanism comparison

| Mechanism                | Protects                     | Main security question                             |
| ------------------------ | ---------------------------- | -------------------------------------------------- |
| Root/jailbreak controls  | Device integrity assumptions | Is the device trustworthy?                         |
| Certificate pinning      | Server identity trust        | Is the app communicating with the intended server? |
| Biometrics               | User authentication          | Is the legitimate user present?                    |
| Hardware-backed keystore | Cryptographic keys           | Can sensitive keys remain protected?               |

---

# Section 4 — Tools and commands

| Tool       | Command                                      | What it finds/shows                        | When to use it                      |
| ---------- | -------------------------------------------- | ------------------------------------------ | ----------------------------------- |
| `adb`      | `adb shell id`                               | Current Android shell identity             | Inspect device privilege context    |
| `adb`      | `adb shell getenforce`                       | SELinux enforcement state                  | Android security-state inspection   |
| `adb`      | `adb shell getprop ro.build.version.release` | Android version                            | Identify test environment           |
| `openssl`  | `openssl s_client -connect example.com:443`  | TLS certificate/connection information     | Understand server TLS configuration |
| `keytool`  | `keytool -list -keystore <keystore>`         | Java keystore contents/metadata            | Inspect test keystores              |
| `codesign` | `codesign -dvvv App.app`                     | iOS signing information                    | Inspect iOS application signing     |
| `security` | `security find-identity -v -p codesigning`   | Available code-signing identities on macOS | iOS development/testing environment |

### Example: inspect Android identity

```text
$ adb shell id

uid=2000(shell) gid=2000(shell)
```

**Interpretation:** Shows the identity under which the Android shell is operating.

### Example: inspect SELinux state

```text
$ adb shell getenforce

Enforcing
```

**Interpretation:** SELinux is actively enforcing its mandatory access-control policy.

### Example: inspect TLS

```text
$ openssl s_client -connect example.com:443
```

**Interpretation:** Establishes a TLS connection and exposes certificate/handshake information useful for understanding the server's TLS configuration. It does not by itself tell you whether a particular mobile application implements certificate pinning.

### Example: inspect a test keystore

```text
$ keytool -list -keystore test.keystore
```

**Interpretation:** Lists metadata/entries in a Java keystore when you have legitimate access to that test keystore.

### Example: inspect iOS signing

```text
$ codesign -dvvv App.app
```

**Interpretation:** Displays code-signing information associated with an iOS application bundle.

---

# Section 5 — Defender detection

* **Device-integrity telemetry:** Enterprise mobile-management systems can identify devices with abnormal/rooted/jailbroken security states.
* **TLS monitoring:** Network telemetry can identify unexpected TLS behavior, although encrypted traffic limits application-level visibility.
* **Authentication events:** Repeated biometric failures, authentication fallbacks, and unusual account/device associations can indicate abuse.
* **Key usage:** Applications should monitor abnormal cryptographic authentication patterns and unexpected device registrations.
* **Application integrity:** Unauthorized modification or repackaging should be detected where appropriate.
* **Server-side authorization:** Critical operations should be independently authorized by backend systems rather than trusting only client-side biometric or integrity checks.
* **Skilled attackers minimize observable changes:** Static analysis and offline inspection can reveal application weaknesses without necessarily triggering mobile runtime telemetry.

---

# Section 6 — Lab task

**Objective:** On an Android test emulator/device you control, inspect the device security state and analyze a test application to determine how its authentication, TLS, and cryptographic-key protections are structured.

### Steps

1. Connect your Android test device/emulator.
2. Record the Android version and current security-policy state.
3. Obtain a deliberately vulnerable mobile application for testing.
4. Inspect its requested permissions and authentication-related functionality.
5. Identify whether it uses biometric APIs or a simpler application-level authentication mechanism.
6. Inspect the application's network-security configuration and determine whether TLS is being used.
7. Identify whether cryptographic keys are handled through the Android Keystore rather than ordinary application storage.
8. Document which protections are enforced by the platform versus the application.
9. Record the security boundaries and potential weaknesses without attempting to bypass them yet.

### Expected output

```text
Test Application
│
├── Authentication
│   └── Biometric / Password
│
├── Network
│   └── TLS
│
├── Key Storage
│   └── Android Keystore
│
└── Device Trust
    └── SELinux / Integrity State
```

### Git artifacts

```text
mobile-security-concepts/
├── README.md
├── notes/
│   └── rooting-pinning-biometrics-keystore.md
├── analysis/
│   └── security-controls.md
└── evidence/
    └── device-security-state.txt
```

Commit:

```bash
git add mobile-security-concepts/
git commit -m "Add mobile security concepts lab"
```

---

# Section 7 — Common mistakes

1. **Thinking rooting/jailbreaking is itself an application vulnerability** → It primarily changes the device trust environment → Distinguish device compromise from application flaws.

2. **Confusing TLS with certificate pinning** → TLS provides encrypted/authenticated transport; pinning adds application-specific server identity restrictions → Treat them as separate controls.

3. **Assuming biometrics are sent directly to applications** → Modern mobile platforms generally isolate biometric processing → Focus on the authentication result and protected key usage.

4. **Assuming a keystore automatically means hardware-backed protection** → Hardware-backed capabilities depend on device/platform support and key configuration → Verify how the key is actually protected.

5. **Treating client-side root/jailbreak detection as a complete defense** → A modified client can potentially alter its own behavior → Enforce critical security decisions on trusted backend infrastructure.

6. **Assuming a biometric prompt automatically protects every sensitive operation** → Applications can implement authentication flows incorrectly → Determine exactly which operation the biometric result authorizes.

7. **Jumping directly into bypass techniques** → You need to understand what security property the control provides before attempting to defeat it → Map the trust boundary first; detailed mobile exploitation comes later.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Differentiate:** Given a mobile security architecture, correctly explain what **rooting/jailbreaking, certificate pinning, biometric authentication, and hardware-backed keystores** each protect.

2. **Trace:** Given a sensitive mobile operation, trace the security chain from **user authentication → application → protected key → TLS/API → server authorization** and identify each trust boundary.

3. **Assess:** Given a mobile application's security design, determine whether a control is merely present or actually protecting a meaningful security property—and identify whether the relevant weakness belongs to **device integrity, authentication, network trust, key protection, or backend authorization**.
