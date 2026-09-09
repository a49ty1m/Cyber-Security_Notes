# Network Security Protocols — TryHackMe Notes

[TryHackMe — Network Security Protocols](https://tryhackme.com/room/networksecurityprotocols?utm_source=chatgpt.com)

## 1. Room Overview

### Main objective

Understand how networking protocols can provide:

* **Confidentiality**
* **Integrity**
* **Authentication**
* **Secure communication**

The room specifically organizes protocols according to the **OSI model** and covers secure protocols at different layers. ([TryHackMe][1])

### Prerequisites

You should already understand:

* OSI model
* HTTP and HTTP methods
* Basic security principles

([TryHackMe][1])

---

# 2. Why Do We Need Secure Protocols?

This is the most important concept in the room.

In the previous **Networking Core Protocols** room, you learned protocols such as:

```text
HTTP
FTP
SMTP
POP3
IMAP
TELNET
```

The problem is that many traditional implementations can transmit information without adequate encryption.

For example:

```text
User
 |
 | username: alice
 | password: password123
 ↓
Server
```

If that traffic is observable by an attacker, sensitive information may be exposed.

The three major security properties are:

### Confidentiality

Only authorized parties should be able to read the data.

```text
Alice ─── encrypted data ───> Server
              ↑
          Attacker sees
          unreadable data
```

### Integrity

Data should not be modified without detection.

```text
Original:
Transfer ₹100

Attacker tries:
Transfer ₹900

↓

Integrity protection detects
the modification.
```

### Authentication

You need confidence that you are communicating with the intended party.

```text
Client
  |
  | "Are you really example.com?"
  ↓
Server
  |
  | Certificate / authentication
  ↓
Client
```

These three concepts are central to secure networking.

---

# 3. Plaintext vs Encrypted Communication

## Plaintext

Data is sent in a readable form.

```text
HTTP
FTP
TELNET
```

Potential problem:

```text
Client
   |
   | USER bob
   | PASS hunter2
   ↓
Server

        ↑
    Attacker
    sniffing traffic
```

The attacker may be able to read the credentials.

---

## Encrypted communication

```text
Client
   |
   | encrypted traffic
   ↓
Server

        ↑
    Attacker
    sees encrypted
    packets
```

The attacker may still see that communication is happening, but the contents should be protected.

**Important:** Encryption does not magically make a protocol secure against every attack. Authentication, certificate validation, configuration, endpoint security, and protocol implementation still matter.

---

# 4. TLS — Transport Layer Security

TLS is one of the most important technologies in this room.

## What is TLS?

**Transport Layer Security (TLS)** is a cryptographic protocol used to secure communication.

It provides:

```text
        TLS
         |
   +-----+-----+
   |     |     |
Conf.  Integrity Authentication
```

TryHackMe explains that TLS can be added to existing application protocols to protect confidentiality, integrity, and authenticity. ([TryHackMe][2])

---

# 5. TLS Handshake

Before encrypted application data is exchanged, the client and server establish a secure connection.

Simplified:

```text
Client                         Server
  |                              |
  |------ ClientHello ---------->|
  |                              |
  |<----- ServerHello -----------|
  |<----- Certificate -----------|
  |                              |
  |==== Key establishment =======|
  |                              |
  |<==== Encrypted traffic =====>|
```

The actual TLS handshake is more complicated and differs between TLS versions, but this simplified model is enough to understand the concept.

---

# 6. Digital Certificates

A certificate helps a client determine whether a server is who it claims to be.

For example:

```text
You connect to:

https://example.com
```

The server provides a certificate containing information associated with the domain and cryptographic identity.

A trusted **Certificate Authority (CA)** signs the certificate.

Conceptually:

```text
Certificate Authority
          |
          | signs
          ↓
     Server Certificate
          |
          ↓
       Browser
```

The browser checks things such as:

* Certificate validity
* Domain name
* Expiration
* Trust chain
* Signature

---

# 7. Certificate Authority

A **Certificate Authority (CA)** is an entity trusted to issue/sign digital certificates.

Simplified trust chain:

```text
Root CA
   |
   ↓
Intermediate CA
   |
   ↓
Server Certificate
   |
   ↓
example.com
```

Your browser/OS contains a collection of trusted root certificates.

This is one reason simply generating your own certificate doesn't automatically make your website trusted.

---

# 8. HTTPS

HTTPS =

```text
HTTP + TLS
```

Instead of:

```text
HTTP
TCP/80
```

HTTPS commonly uses:

```text
TCP/443
```

Conceptually:

```text
HTTP:

Client ───── HTTP ─────> Server


HTTPS:

Client ── TLS ──> Server
           |
           └── HTTP inside encrypted connection
```

TryHackMe explicitly describes HTTPS as HTTP secured with TLS. ([TryHackMe][2])

---

# 9. HTTP vs HTTPS

| Feature                     | HTTP      | HTTPS                  |
| --------------------------- | --------- | ---------------------- |
| Encryption                  | ❌         | ✅                      |
| Integrity protection        | ❌         | ✅                      |
| Server authentication       | ❌/limited | ✅ via TLS certificates |
| Common port                 | 80        | 443                    |
| Uses TLS                    | No        | Yes                    |
| Suitable for sensitive data | No        | Yes                    |

### Important

HTTPS does **not** mean:

> "The website is trustworthy."

It means the connection has TLS protection and the server presented a certificate that passed the relevant validation checks.

A phishing website can also use HTTPS.

That's a very important distinction.

---

# 10. Secure Email Protocols

From the previous room:

```text
SMTP  → Send email
POP3  → Retrieve email
IMAP  → Access/synchronize email
```

Their TLS-secured variants include:

```text
SMTPS
POP3S
IMAPS
```

TryHackMe specifically covers these secure versions in the networking security sequence. ([TryHackMe][2])

---

# 11. SMTPS

SMTP is used to send email.

TLS can protect SMTP communication.

Commonly encountered ports include:

```text
25
465
587
```

Don't blindly associate all three with exactly the same security mode.

Modern email deployments commonly use:

```text
587 → message submission + STARTTLS
465 → implicit TLS submission
25  → server-to-server SMTP, often with STARTTLS
```

---

# 12. STARTTLS

This is an important concept.

STARTTLS allows an existing plaintext protocol connection to be upgraded to TLS.

Conceptually:

```text
Client
  |
  | Plain connection
  ↓
Server
  |
  | STARTTLS
  ↓
TLS negotiation
  |
  ↓
Encrypted communication
```

This is different from **implicit TLS**, where TLS is established immediately after connecting.

### Remember

```text
STARTTLS
→ Start plaintext protocol
→ Request TLS upgrade
→ Continue securely
```

---

# 13. POP3S

POP3 is used for retrieving email.

Traditional:

```text
POP3
TCP/110
```

TLS-protected:

```text
POP3S
TCP/995
```

Conceptually:

```text
Client
  |
  | TLS
  ↓
POP3 server
  |
  ↓
Email
```

---

# 14. IMAPS

IMAP is used for accessing and synchronizing mailboxes.

Traditional:

```text
IMAP
TCP/143
```

TLS-protected:

```text
IMAPS
TCP/993
```

---

# 15. Email Protocol Cheat Sheet

Memorize this table:

| Protocol | Purpose          | Common secure port |
| -------- | ---------------- | -----------------: |
| SMTP     | Send mail        |          465 / 587 |
| POP3     | Retrieve mail    |                995 |
| IMAP     | Access/sync mail |                993 |

And:

```text
SMTP  → SEND
POP3  → PULL
IMAP  → SYNCHRONIZE
```

---

# 16. TELNET → SSH

This is another major concept.

TELNET allows remote access to another machine.

But traditional Telnet communication is not properly protected against eavesdropping.

Example:

```text
Attacker
   |
   | sniffing
   ↓
TELNET traffic

Username: admin
Password: password123
```

That's obviously bad.

---

# 17. SSH — Secure Shell

SSH was designed to provide secure remote access.

Default port:

```text
TCP/22
```

Instead of:

```text
Client ─── plaintext ───> Server
```

you have:

```text
Client
   |
   | encrypted SSH connection
   ↓
Server
```

TryHackMe specifically identifies SSH as the secure replacement for remote access previously provided by Telnet. ([TryHackMe][2])

---

# 18. What SSH Provides

SSH can provide:

### Confidentiality

Commands and data are encrypted.

### Integrity

Tampering with packets should be detected.

### Authentication

The client and server can authenticate each other.

### Secure remote shell

```bash
ssh username@TARGET_IP
```

Example:

```bash
ssh student@10.10.10.10
```

---

# 19. SSH Authentication

Two common approaches are:

### Password authentication

```text
Username
   +
Password
   ↓
SSH Server
```

### Public-key authentication

You have:

```text
Private key → kept secret
Public key  → placed on server
```

Conceptually:

```text
Client                         Server
Private Key                     Public Key
    |                               |
    +-------- Authentication -------+
```

The private key should **never be shared**.

---

# 20. SFTP

SFTP means:

**SSH File Transfer Protocol**

It provides file transfer functionality over SSH.

Typical port:

```text
TCP/22
```

Example:

```bash
sftp username@TARGET_IP
```

You may see commands such as:

```text
ls
cd
get
put
pwd
```

---

# 21. SFTP vs FTP

| Feature                   | FTP                          | SFTP      |
| ------------------------- | ---------------------------- | --------- |
| Security                  | Plaintext by default         | Encrypted |
| Underlying technology     | FTP                          | SSH       |
| Typical port              | 21                           | 22        |
| Authentication protection | Weak/plain depending on mode | SSH       |
| Encryption                | No                           | Yes       |

**Critical distinction:**

SFTP is **not simply "FTP with encryption."**

It is a different protocol that operates over SSH.

---

# 22. FTPS

FTPS is different from SFTP.

```text
FTPS
=
FTP + TLS
```

while:

```text
SFTP
=
SSH-based file transfer protocol
```

### Comparison

|                     | FTP  | FTPS                       | SFTP |
| ------------------- | ---- | -------------------------- | ---- |
| Security technology | None | TLS                        | SSH  |
| Base protocol       | FTP  | FTP                        | SSH  |
| Common port         | 21   | 21 / 990 depending on mode | 22   |
| Encrypted           | ❌   | ✅                         | ✅   |

This distinction is frequently tested in networking and cybersecurity interviews.

---

# 23. VPN — Virtual Private Network

Now we move beyond individual application protocols.

A VPN creates a secure logical connection across an untrusted network.

Imagine:

```text
Office A
   |
   |
Internet
   |
   |
Office B
```

Without protection, traffic crosses an untrusted environment.

A VPN creates a tunnel:

```text
Office A
   |
   |=========================|
   |      VPN tunnel         |
   |=========================|
   |
Internet
   |
   |
Office B
```

---

# 24. VPN Security

A VPN can provide:

* Confidentiality
* Integrity
* Authentication
* Secure connectivity across untrusted networks

Example:

```text
Laptop
   |
   | encrypted tunnel
   ↓
VPN Gateway
   |
   ↓
Internal Network
```

---

# 25. VPN Types

Two broad concepts you should know:

### Remote-access VPN

Used by an individual user to connect to an organization/network.

```text
Employee
   ↓
Internet
   ↓
VPN Gateway
   ↓
Company Network
```

### Site-to-site VPN

Connects entire networks.

```text
Office A
   |
VPN tunnel
   |
Office B
```

---

# 26. Network Layer Security

The room also covers protocols at the **Network layer**. ([TryHackMe][1])

A major technology here is:

**IPsec — Internet Protocol Security**

IPsec can secure IP communications.

It is commonly associated with:

* Authentication
* Integrity
* Confidentiality
* VPNs

Conceptually:

```text
Application
     ↓
TCP / UDP
     ↓
IPsec
     ↓
IP
     ↓
Ethernet
```

---

# 27. IPsec

IPsec is a suite of protocols used to secure IP communication.

Important concepts include:

### Authentication

Verifies communicating parties/data origin.

### Integrity

Detects modification.

### Confidentiality

Encrypts traffic.

### Key management

Handles cryptographic keys used for secure communication.

---

# 28. AH vs ESP

This is worth knowing.

### AH — Authentication Header

Provides:

* Authentication
* Integrity

But does **not** provide encryption/confidentiality.

### ESP — Encapsulating Security Payload

Can provide:

* Confidentiality
* Integrity
* Authentication

In modern VPN deployments, **ESP is much more important than AH**.

Easy memory:

```text
AH
→ Authentication
→ Integrity

ESP
→ Encryption
→ Security
→ Protection
```

---

# 29. OSI Layer View

This room becomes much easier if you organize it by OSI layer.

| OSI Layer      | Examples                                          |
| -------------- | ------------------------------------------------- |
| 7 Application  | HTTPS, SMTPS, POP3S, IMAPS, SSH, SFTP             |
| 6 Presentation | Encryption / encoding concepts                    |
| 5 Session      | Session establishment/management                  |
| 4 Transport    | TCP / UDP + TLS commonly operates above transport |
| 3 Network      | IPsec                                             |
| 2 Data Link    | Ethernet / Wi-Fi security mechanisms              |

The exact OSI classification of modern protocols can be nuanced, so don't treat the table as a rigid statement about where every technology "lives" in an implementation.

---

# 30. Plain vs Secure Protocols

This is probably the most important revision table.

| Insecure / plaintext | Secure alternative    |
| -------------------- | --------------------- |
| HTTP                 | HTTPS                 |
| FTP                  | FTPS / SFTP           |
| TELNET               | SSH                   |
| POP3                 | POP3S                 |
| IMAP                 | IMAPS                 |
| SMTP                 | SMTP with TLS / SMTPS |

Think:

```text
HTTP       → HTTPS
FTP        → FTPS / SFTP
TELNET     → SSH
POP3       → POP3S
IMAP       → IMAPS
SMTP       → TLS-protected SMTP
```

TryHackMe explicitly uses this progression to explain why TLS and SSH matter. ([TryHackMe][2])

---

# 31. The Most Important Difference: Encryption vs Authentication

Don't make this beginner mistake:

> "It's encrypted, therefore it's safe."

No.

Consider:

```text
Encryption
    ↓
Protects the contents
```

But:

```text
Authentication
    ↓
Helps establish who you're communicating with
```

And:

```text
Integrity
    ↓
Helps detect modification
```

A secure protocol needs to address the relevant security properties, not just encryption.

---

# 32. MITM Connection

This is where secure protocols become particularly important.

**MITM = Man-in-the-Middle**

Without proper authentication:

```text
Client
  |
  ↓
Attacker
  |
  ↓
Server
```

The attacker can potentially:

* Read traffic
* Modify traffic
* Inject traffic
* Impersonate a party

TLS certificate validation helps defend against server impersonation.

SSH uses its own host-key mechanisms to help establish server identity.

VPN technologies similarly rely on authentication and cryptographic mechanisms.

---

# 33. Security Comparison

| Protocol | Main problem                        | Secure technology |
| -------- | ----------------------------------- | ----------------- |
| HTTP     | Traffic can be observed/modified    | TLS               |
| FTP      | Credentials/data exposed            | FTPS/SFTP         |
| Telnet   | Remote sessions exposed             | SSH               |
| POP3     | Mail/credentials exposed            | POP3S             |
| IMAP     | Mail/credentials exposed            | IMAPS             |
| SMTP     | Email transmission can be exposed   | TLS               |
| IP       | No built-in general confidentiality | IPsec             |

---

# 34. Commands You Should Know

### HTTPS

```bash
curl https://example.com
```

Inspect TLS:

```bash
openssl s_client -connect example.com:443
```

### SSH

```bash
ssh username@TARGET_IP
```

### SFTP

```bash
sftp username@TARGET_IP
```

### Check TLS certificate

```bash
openssl s_client -connect example.com:443
```

You can inspect:

* Certificate
* Issuer
* Subject
* TLS version
* Cipher information

---

# 35. Pentesting Perspective

This room is particularly important for your cybersecurity path because you need to recognize secure vs insecure services during enumeration.

Suppose Nmap gives you:

```text
21/tcp   ftp
22/tcp   ssh
25/tcp   smtp
80/tcp   http
110/tcp  pop3
143/tcp  imap
443/tcp  https
```

You shouldn't just think:

> "Port 80 = website."

You should think:

```text
80
 ↓
HTTP
 ↓
Is it plaintext?
 ↓
Can credentials/data be exposed?
 ↓
Is HTTP redirecting to HTTPS?
 ↓
What TLS configuration does 443 use?
```

For email:

```text
110 → POP3
 ↓
Is TLS supported?
 ↓
995 → POP3S?
 ↓
Are credentials protected?
```

For remote access:

```text
23 → Telnet
 ↓
Why is plaintext remote access enabled?
 ↓
Potentially serious finding

22 → SSH
 ↓
Check authentication/configuration
```

That's the mindset you want.

---

# 36. Quick Enumeration Commands

For authorized labs:

```bash
nmap -sV TARGET_IP
```

Check common secure services:

```bash
nmap -sV -p 22,443,465,587,993,995 TARGET_IP
```

Check HTTPS:

```bash
curl -I https://TARGET_IP
```

Inspect TLS:

```bash
openssl s_client -connect TARGET_IP:443
```

SSH:

```bash
ssh user@TARGET_IP
```

SFTP:

```bash
sftp user@TARGET_IP
```

---

# 37. Exam / Interview Revision

### What does TLS provide?

**Confidentiality, integrity, and authentication/authenticity.**

### What does HTTPS mean?

```text
HTTP + TLS
```

### HTTPS default port?

```text
443
```

### SSH default port?

```text
22
```

### SFTP uses what underlying technology?

```text
SSH
```

### FTPS uses what?

```text
TLS
```

### POP3S?

```text
POP3 + TLS
```

### IMAPS?

```text
IMAP + TLS
```

### What replaced Telnet for secure remote administration?

```text
SSH
```

### What is a VPN?

A mechanism that creates a secure logical tunnel across an untrusted network.

### What is IPsec?

A suite of protocols used to secure IP communication.

---

# 38. Final Cheat Sheet

```text
┌──────────────────────────────────────────┐
│       NETWORK SECURITY PROTOCOLS         │
└──────────────────────────────────────────┘

TLS
│
├── Confidentiality
├── Integrity
└── Authentication

APPLICATION
│
├── HTTP  → HTTPS
├── SMTP  → TLS/SMTPS
├── POP3  → POP3S
├── IMAP  → IMAPS
├── FTP   → FTPS
└── FTP   → SFTP (SSH-based)

REMOTE ACCESS
│
└── TELNET → SSH

NETWORK
│
└── IP → IPsec

SECURE NETWORKING
│
└── VPN
     ├── Remote Access
     └── Site-to-Site
```

## What you should actually remember

Don't memorize this room as a collection of port numbers.

Build this chain in your head:

```text
CORE PROTOCOLS
      ↓
"What does the protocol do?"
      ↓
SECURITY PROBLEM
      ↓
"Can someone read/change/impersonate?"
      ↓
SECURE PROTOCOL
      ↓
TLS / SSH / IPsec / VPN
      ↓
SECURITY TESTING
      ↓
"Is the service actually configured securely?"
```

That's the useful takeaway from this room. TryHackMe places **Network Security Protocols** as the security-focused networking module, following the earlier networking rooms, and its current learning module also points learners toward Wireshark, tcpdump, and Nmap afterward. ([TryHackMe][3])

For your pentesting progression, **don't move on until you can look at `21/22/23/25/80/110/143/443/465/587/993/995` and immediately explain what service is likely there, whether it's normally encrypted, and what the secure alternative is.**

[1]: https://tryhackme.com/room/networksecurityprotocols "TryHackMe | Network Security Protocols"
[2]: https://tryhackme.com/room/networkingsecureprotocols?utm_source=chatgpt.com "TryHackMe | Networking Secure Protocols"
[3]: https://tryhackme.com/module/networking?utm_source=chatgpt.com "TryHackMe | Networking"
