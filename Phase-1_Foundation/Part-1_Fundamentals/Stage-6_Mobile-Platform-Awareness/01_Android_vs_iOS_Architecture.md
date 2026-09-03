# Android vs iOS Architecture

**Roadmap:** Part 1 — Fundamentals → Stage 6 — Mobile Platform Awareness → Android vs iOS Architecture

# Section 1 — What it is and where it sits

Android and iOS are mobile operating systems with very different architectural philosophies. **Android** is built around the Linux kernel and uses application packages such as **APK**, application sandboxing, and **SELinux** enforcement. **iOS** uses Apple's platform architecture, **IPA** application packages, **Mach-O** executables, mandatory code-signing requirements, and hardware-backed security features such as the **Secure Enclave**.

For a security professional, the important skill is not memorizing file extensions. You need to understand **where applications execute, how they are isolated, how code is trusted, where credentials/secrets are protected, and what security boundaries exist between applications and the operating system**.

```text
Mobile Platforms
│
├── Android
│   ├── Linux kernel
│   ├── Android framework/runtime
│   ├── APK
│   ├── Application sandbox
│   └── SELinux
│
└── iOS
    ├── XNU-based kernel
    ├── iOS frameworks/runtime
    ├── IPA
    ├── Mach-O
    ├── Code signing
    └── Secure Enclave
```

The attack-chain position is:

```text
Mobile reconnaissance
        ↓
Identify platform
        ↓
Identify application/package architecture
        ↓
Understand trust boundaries
        ↓
Analyze application + OS interaction
        ↓
Mobile application security testing
```

This topic comes before detailed mobile exploitation because **Android and iOS require different assumptions, tools, and analysis techniques**.

---

# Section 2 — How attackers actually use this

## 2.1 Identify the platform first

The first question is:

> **“Am I dealing with Android or iOS?”**

This affects almost everything that follows:

* Application package format
* Executable format
* Filesystem structure
* Runtime
* Permission model
* Code-signing model
* Debugging capabilities
* Application storage
* IPC mechanisms
* Security boundaries
* Available analysis tooling

A mobile tester who treats Android and iOS as identical will waste time using the wrong techniques.

---

## 2.2 Android application architecture

A simplified Android application looks like:

```text
Android App
│
├── APK
│   ├── AndroidManifest.xml
│   ├── DEX bytecode
│   ├── Resources
│   ├── Native libraries
│   └── Assets
│
└── Android OS
    ├── Application framework
    ├── Runtime
    └── Linux kernel
```

The **APK** is the application package distributed/installed on Android systems.

An attacker analyzing an Android application may therefore inspect:

* Manifest declarations
* Application components
* Permissions
* DEX bytecode
* Native libraries
* Resources
* Embedded configuration
* Network endpoints
* Local storage

The APK is therefore both an installation artifact and a valuable source of application intelligence.

---

## 2.3 Android's Linux kernel

Android uses the **Linux kernel**, but Android is not simply a normal Linux desktop distribution.

Android adds its own:

* Application framework
* Runtime
* System services
* IPC mechanisms
* Permission architecture
* Hardware abstraction components
* Security policies

Conceptually:

```text
Android Applications
        ↓
Android Framework
        ↓
Android Runtime / System Services
        ↓
Linux Kernel
        ↓
Hardware
```

The kernel provides fundamental isolation and hardware/resource management, while Android's higher layers enforce mobile-specific security controls.

---

## 2.4 Android application sandboxing

Android applications are designed to run in isolated security contexts.

Conceptually:

```text
App A ── Sandbox A
App B ── Sandbox B
App C ── Sandbox C

        ↓
   Android OS
        ↓
      Kernel
```

The goal is to prevent one application from freely accessing another application's private resources.

The sandbox is not merely a directory restriction. It works together with:

* Linux users/UIDs
* Filesystem permissions
* Android permissions
* SELinux
* Process isolation
* Framework-level security controls

Therefore:

> **Android application isolation is a layered security model.**

---

## 2.5 SELinux adds mandatory access control

Android uses **SELinux** to provide mandatory access control (MAC).

A simplified model:

```text
Application
     │
     ↓
Traditional permissions
     │
     ↓
SELinux policy
     │
     ↓
Kernel enforcement
     │
     ↓
Resource
```

Even if a process has some traditional Linux privileges, SELinux policy can restrict what it is allowed to access.

For security testing, this matters because:

> **A successful privilege escalation does not necessarily mean unrestricted access to everything.**

The resulting SELinux context and policy determine what the compromised process can actually do.

---

## 2.6 Android application components matter

Android applications are composed of standardized components such as:

* Activities
* Services
* Broadcast Receivers
* Content Providers

Conceptually:

```text
APK
│
├── Activity
├── Service
├── Broadcast Receiver
└── Content Provider
```

These components can create security boundaries and IPC interfaces.

An exposed component may therefore become an important application attack surface.

For example:

```text
Attacker-controlled input
        ↓
Exported component
        ↓
Application logic
        ↓
Sensitive operation
```

This is why Android application analysis begins with understanding the application structure rather than immediately looking at native code.

---

## 2.7 iOS application architecture

A simplified iOS application looks like:

```text
iOS App
│
├── IPA
│   └── Application bundle
│       ├── Mach-O executable
│       ├── Resources
│       ├── Frameworks
│       └── Metadata
│
└── iOS
    ├── Frameworks
    ├── Security services
    ├── XNU-based kernel
    └── Hardware security
```

An **IPA** is an iOS application distribution package.

Inside the application bundle, the primary executable is typically a **Mach-O** binary.

Therefore, when analyzing an iOS application:

```text
IPA
 ↓
Application bundle
 ↓
Mach-O executable
 ↓
Code / libraries / metadata
```

---

## 2.8 Mach-O is the executable format

**Mach-O** stands for **Mach Object**.

It is the executable/object-file format used by Apple's platforms.

From a security perspective, the distinction matters:

```text
Android
APK → DEX / native ELF libraries

iOS
IPA → Mach-O executable
```

This affects:

* Static analysis
* Disassembly
* Symbol analysis
* Binary inspection
* Reverse engineering
* Debugging workflows

---

## 2.9 iOS code signing

Code signing is a fundamental part of Apple's platform security model.

Conceptually:

```text
Application
     ↓
Code signing
     ↓
Signature validation
     ↓
Trusted execution/install process
```

The operating system can use signatures and associated trust information to determine whether executable code is authorized to run.

This creates a major difference from a simplistic model where:

```text
Download file → execute file
```

Instead, iOS strongly integrates **code provenance and integrity** into its security architecture.

For a mobile security tester, code signing affects:

* Application installation
* Application modification
* Debugging
* Repackaging
* Execution of modified binaries
* Device trust boundaries

---

## 2.10 Secure Enclave

The **Secure Enclave** is a dedicated security subsystem in Apple's hardware architecture designed to protect sensitive cryptographic operations and secrets.

Conceptually:

```text
iPhone/iPad
│
├── Main processor
│   └── iOS
│
└── Secure Enclave
    ├── Protected keys
    ├── Cryptographic operations
    └── Security-sensitive functions
```

The important security principle is:

> **Sensitive cryptographic material can be isolated from the main application processor and normal application environment.**

This is particularly important for:

* Device authentication
* Biometric-related security
* Key protection
* Hardware-backed cryptography

---

## 2.11 Android and iOS use different trust models

A simplified comparison:

```text
Android
Application
   ↓
Sandbox
   ↓
Android permissions
   ↓
SELinux
   ↓
Linux kernel


iOS
Application
   ↓
Sandbox
   ↓
Entitlements / permissions
   ↓
Code signing / platform controls
   ↓
XNU kernel
   ↓
Hardware security
```

Both platforms use isolation, but **the implementation and security boundaries differ substantially**.

---

## 2.12 Attackers look for boundary violations

The most valuable findings often occur when an application crosses a security boundary incorrectly.

Examples:

```text
App
 ↓
Sensitive API
 ↓
Authorization failure
```

or:

```text
App
 ↓
IPC
 ↓
Privileged component
 ↓
Unexpected access
```

or:

```text
Application
 ↓
Local storage
 ↓
Sensitive credential
 ↓
Weak protection
```

The specific technique differs between Android and iOS, but the fundamental question remains:

> **“Can untrusted input cross a security boundary and cause something it should not be able to do?”**

---

## 2.13 Dead-end vs high-value findings

| Finding                              | Typical value |
| ------------------------------------ | ------------- |
| Platform identified                  | Low           |
| APK/IPA obtained                     | Medium        |
| Application metadata exposed         | Medium        |
| Interesting permissions/entitlements | Medium        |
| Sensitive data stored locally        | High          |
| Exposed IPC/application interface    | High          |
| Broken authorization boundary        | Very High     |
| Security-sensitive native code flaw  | Very High     |
| OS/kernel privilege boundary bypass  | Critical      |

Obtaining an APK or IPA is **not itself a vulnerability**.

It gives you an artifact to analyze.

---

## 2.14 Where this feeds next

```text
Android/iOS identification
          ↓
Package extraction
          ↓
Manifest / metadata analysis
          ↓
Application architecture
          ↓
Runtime behavior
          ↓
IPC / storage / network analysis
          ↓
Mobile security testing
```

---

# Section 3 — Core concepts and terminology

### Android

Google-led mobile operating system ecosystem built around a Linux kernel and Android application framework.

### APK

**Android Package**; application package format used to distribute/install Android applications.

### Linux Kernel

Core operating-system layer responsible for processes, memory, hardware interaction, networking, and other fundamental functions.

### Android Runtime

Android execution environment responsible for running application bytecode.

### DEX

**Dalvik Executable** format containing Android application bytecode.

### Android Sandbox

Isolation mechanisms that restrict application access to other applications and protected resources.

### SELinux

**Security-Enhanced Linux**; mandatory access-control system used by Android to enforce security policies.

### Android UID

Linux user identity associated with an Android application/process for isolation and permission enforcement.

### Activity

Android application component generally associated with a user-facing interface.

### Service

Android component designed for operations that can continue without a traditional user interface.

### Broadcast Receiver

Android component that responds to broadcast messages/events.

### Content Provider

Android component providing structured data access between applications/components.

### IPA

Common distribution package format for iOS applications.

### Mach-O

Apple executable/object-file format used by iOS and other Apple platforms.

### XNU

The kernel architecture underlying Apple's operating systems, combining Mach concepts with BSD components and other Apple technologies.

### Code Signing

Cryptographic mechanism used to establish the integrity and authorized origin of executable code.

### Entitlement

Signed declaration granting an application access to specific privileged capabilities.

### Secure Enclave

Dedicated hardware security subsystem used to protect sensitive keys and security operations.

### Sandboxing

Restricting an application's access to system resources and other applications.

### IPC

**Inter-Process Communication**; mechanisms allowing processes/components to communicate.

### Privilege Boundary

Security boundary separating processes/resources with different levels of authority.

### Hardware-backed Security

Security functionality supported by dedicated hardware rather than relying entirely on software.

### Attack Surface

Collection of interfaces through which an attacker can interact with a system.

---

## Android vs iOS

| Feature                       | Android                                                 | iOS                                                       |
| ----------------------------- | ------------------------------------------------------- | --------------------------------------------------------- |
| Application package           | APK                                                     | IPA                                                       |
| Kernel foundation             | Linux kernel                                            | XNU-based                                                 |
| Common application executable | DEX + native code                                       | Mach-O                                                    |
| Application isolation         | Sandbox + Linux/Android controls                        | Sandbox + platform controls                               |
| Mandatory policy              | SELinux                                                 | Strong platform security/code-signing model               |
| Application components        | Activities, Services, Receivers, Providers              | App processes/frameworks and platform APIs                |
| Code trust                    | Package/signing mechanisms                              | Strong mandatory code-signing architecture                |
| Hardware security             | Hardware-backed security available on supported devices | Secure Enclave and other hardware-backed mechanisms       |
| Major assessment focus        | Components, permissions, IPC, storage, native code      | Signing, entitlements, sandbox, IPC, storage, native code |

---

# Section 4 — Tools and commands

| Tool          | Command                                      | What it finds/shows                         | When to use it                   |
| ------------- | -------------------------------------------- | ------------------------------------------- | -------------------------------- |
| `adb`         | `adb devices`                                | Connected Android devices/emulators         | Identify Android testing targets |
| `adb`         | `adb shell getprop ro.build.version.release` | Android OS version                          | Platform reconnaissance          |
| `aapt`        | `aapt dump badging app.apk`                  | APK package metadata                        | Initial APK analysis             |
| `apkanalyzer` | `apkanalyzer manifest permissions app.apk`   | Android permissions                         | Permission analysis              |
| `jadx`        | `jadx app.apk`                               | Decompiles DEX into readable Java-like code | Static Android analysis          |
| `apktool`     | `apktool d app.apk`                          | Decodes APK resources and smali             | APK structure/resource analysis  |
| `file`        | `file binary`                                | Identifies executable/file format           | Binary identification            |
| `otool`       | `otool -hv binary`                           | Mach-O header information                   | iOS binary analysis              |
| `codesign`    | `codesign -dvvv App.app`                     | Code-signing information                    | iOS signing analysis             |

### Example: identify Android devices

```text
$ adb devices

List of devices attached
emulator-5554    device
```

**Interpretation:** An Android emulator/device is available to the testing environment.

### Example: identify Android version

```text
$ adb shell getprop ro.build.version.release

14
```

**Interpretation:** The connected Android environment reports Android version 14.

### Example: inspect an APK

```text
$ aapt dump badging app.apk
```

**Interpretation:** Useful for quickly identifying package name, application metadata, SDK information, and related APK properties.

### Example: inspect permissions

```text
$ apkanalyzer manifest permissions app.apk
```

**Interpretation:** Displays permissions requested by the Android application, helping identify potentially security-relevant capabilities.

### Example: decompile an APK

```text
$ jadx app.apk
```

**Interpretation:** Converts DEX bytecode into a Java-like representation that can be searched for application logic, endpoints, secrets, and security-sensitive operations.

### Example: decode APK resources

```text
$ apktool d app.apk
```

**Interpretation:** Extracts application resources and translates Android bytecode into smali, useful for deeper static analysis.

### Example: identify a binary

```text
$ file binary

Mach-O 64-bit executable arm64
```

**Interpretation:** The executable is a 64-bit ARM Mach-O binary, strongly indicating an Apple-platform executable.

### Example: inspect Mach-O headers

```text
$ otool -hv binary
```

**Interpretation:** Displays Mach-O header information useful during iOS binary analysis.

### Example: inspect code signing

```text
$ codesign -dvvv App.app
```

**Interpretation:** Displays signing information associated with the application bundle.

---

# Section 5 — Defender detection

* **Application integrity:** Monitor application installation and integrity controls to identify unauthorized or modified applications.
* **Android security telemetry:** Look for unusual application installation, permission usage, privileged behavior, and security-policy violations.
* **iOS code-signing controls:** Unauthorized or improperly signed application code should be prevented from executing by platform security mechanisms.
* **Enterprise MDM telemetry:** Mobile-device-management systems can identify unsupported applications, configuration changes, and compromised device states.
* **Privilege anomalies:** Unexpected access to sensitive APIs, services, storage, or device capabilities can indicate an application escaping its intended security boundary.
* **Runtime behavior:** Monitor unusual network connections, excessive permissions, suspicious background activity, and abnormal application behavior.
* **Skilled attackers reduce footprint:** Static analysis can be performed against application artifacts without generating runtime telemetry on the target device, so defenders cannot rely exclusively on device logs.

---

# Section 6 — Lab task

**Objective:** Obtain a deliberately vulnerable Android application in an authorized lab and map its APK structure, permissions, components, and executable code before performing deeper mobile exploitation.

### Steps

1. Set up an Android emulator or dedicated test device.
2. Obtain a deliberately vulnerable/test APK.
3. Record the package name and basic metadata.
4. Extract the APK structure.
5. Review the manifest and requested permissions.
6. Identify Activities, Services, Broadcast Receivers, and Content Providers.
7. Decompile the application and locate its main application logic.
8. Identify any native libraries and determine their executable format.
9. Document the application's major trust boundaries.

### Expected output

```text
Test APK
│
├── Manifest
│   ├── Permissions
│   └── Components
│
├── DEX
│   └── Application logic
│
├── Resources
└── Native libraries
    └── ELF
```

### Git artifacts

```text
mobile-platform-awareness/
├── README.md
├── notes/
│   └── android-vs-ios-architecture.md
├── analysis/
│   └── android-apk-inventory.md
└── evidence/
    └── apk-metadata.txt
```

Commit:

```bash
git add mobile-platform-awareness/
git commit -m "Add mobile platform architecture lab"
```

---

# Section 7 — Common mistakes

1. **Thinking APK means “Android executable”** → An APK is a package containing application components and resources → Distinguish the package from DEX and native executable formats.

2. **Treating Android as ordinary desktop Linux** → Android adds its own runtime, framework, services, permissions, and security model → Learn the Android layers separately.

3. **Assuming sandboxing is only filesystem isolation** → Android uses multiple layers including UIDs, permissions, SELinux, and process isolation → Analyze the complete security boundary.

4. **Thinking iOS IPA is the executable** → IPA is a distribution package containing an application bundle → Locate and analyze the Mach-O executable inside it.

5. **Assuming code signing is merely an application-store feature** → Code signing is deeply integrated into iOS platform trust → Understand its role in installation and execution.

6. **Treating Secure Enclave as a general-purpose application sandbox** → It is a dedicated hardware security subsystem for specific security-sensitive operations → Keep hardware-backed key protection separate from application sandboxing.

7. **Using the same analysis workflow for both platforms** → Their package formats, executable formats, security controls, and tooling differ → Classify the platform first and select the appropriate analysis path.

---

# Section 8 — Move-on gate

You can move on when you can perform all three tasks:

1. **Identify:** Given an unknown mobile application artifact, determine whether it is an **APK or IPA**, identify its executable format, and describe the major platform layers underneath it.

2. **Map:** Given an Android application, trace **APK → manifest/components → DEX/native code → Android framework → Linux kernel** and explain where sandboxing and SELinux fit.

3. **Compare:** Given Android and iOS security scenarios, correctly identify whether the relevant boundary involves **Android sandbox/SELinux/permissions** or **iOS sandbox/code signing/entitlements/Secure Enclave**, and explain why the distinction changes the security-testing approach.
