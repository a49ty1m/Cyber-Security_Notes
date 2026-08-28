# Protocol Audit

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

A protocol audit identifies which network services an organization exposes using insecure or outdated protocols — specifically FTP instead of SFTP/FTPS, and SSL/TLS versions and cipher suites that have documented cryptographic weaknesses. The output is a map of protocol-level attack surface: services where data confidentiality and integrity cannot be assumed, or where known protocol vulnerabilities are exploitable.

Protocol identification can be done semi-passively: Shodan and Censys index TLS certificate versions, cipher suites, and banner data for any port they scan. The passive phase involves reading what these platforms have already observed. The active phase — connecting to the service yourself to perform a detailed TLS audit — belongs at the boundary of Stage 2 and Stage 3, and generates direct connection events on the target.

```text
Recon Chain
──────────────────────────────────────────────────────────────────────
Stage 1 (Passive)     Stage 2 (Semi-Passive)              Stage 3 (Active)
WHOIS, CT, OSINT  →  [Protocol Audit]  →  testssl.sh → Direct TLS probe
                         ↑ YOU ARE HERE    sslscan    → Cert validation
                       Shodan TLS data,    nmap SSL   → Banner grab
                       Censys cipher data  scripts    → Service enum
──────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** Protocol weaknesses are often the quietest path to a critical finding. An organization with hardened web apps may be running FTPS on port 21 with SSLv3, or a management interface with TLS 1.0 and export-grade ciphers. These vulnerabilities are not visible through web crawling, subdomain enumeration, or port scanning alone — they require reading the TLS handshake details and comparing against known-weak configurations.

This note follows Shodan/Censys (note 02) — those tools already collect TLS version and cipher data passively. This note teaches you how to read that data, identify weaknesses, and (where active probing is authorized) perform direct TLS audits with dedicated tools.

---

# Section 2 — How attackers actually use this

## 2.1 What insecure protocols reveal

The distinction between secure and insecure protocols is not philosophical — it is operational. Using an insecure protocol means traffic between the client and server can be:

- **Intercepted in cleartext** (FTP, Telnet, HTTP): credentials, file contents, and session data transmitted without encryption.
- **Decrypted after the fact** (SSL 2.0/3.0, TLS 1.0/1.1 with broken cipher suites): protocols with known-exploitable weaknesses allow recorded sessions to be decrypted.
- **Downgraded** (servers supporting multiple protocol versions): an attacker performing a man-in-the-middle attack can negotiate the weakest mutually supported protocol, bypassing the client's TLS 1.3 capability.

For an attacker, identifying that a target exposes these protocols does three things: it identifies attack techniques (BEAST, POODLE, CRIME, FREAK, Logjam), it identifies network segments where passive interception of credentials is viable, and it prioritizes which services to target early because credential interception requires less operational complexity than exploitation.

## 2.2 FTP vs SFTP vs FTPS — the attack surface distinction

FTP transmits credentials and data in cleartext on port 21. A passive network monitor on any path between client and server sees login credentials and file contents. This is not a theoretical risk — anyone with access to a network segment, a router, a cloud provider's infrastructure, or a malicious exit node sees the data.

**The attacker workflow for exposed FTP:**
```text
Shodan: port:21 product:"vsftpd" org:"Target Corp"
                ↓
Identify FTP server IP and version
                ↓
Check if anonymous login is enabled: ftp <IP> → user: anonymous
                ↓
If authenticated FTP: check for credential reuse from breach data (Stage 1)
                ↓
Monitor FTP session (if MITM positioned) for cleartext credentials
```

SFTP (SSH File Transfer Protocol) runs over SSH port 22 and is fully encrypted — it is a completely different protocol from FTP, not a secured version of it. FTPS (FTP Secure) is FTP with TLS layered on top, using port 990 (implicit) or port 21 with STARTTLS (explicit). FTPS is substantially more secure than FTP but still subject to TLS configuration weaknesses.

When Shodan shows port 21 with an FTP banner, the target is running cleartext FTP. When it shows port 22 with OpenSSH, SFTP may be the file transfer mechanism — a critical security distinction.

## 2.3 SSL and TLS version attack surface

TLS versions and their known attack vectors:

| Protocol | Released | Status | Known attacks |
|----------|---------|--------|--------------|
| SSL 2.0 | 1995 | Broken — DROWN | DROWN (cross-protocol decryption via SSLv2 server) |
| SSL 3.0 | 1996 | Broken — POODLE | POODLE (CBC padding oracle), BEAST |
| TLS 1.0 | 1999 | Deprecated | BEAST (CBC), Lucky13, RC4 weaknesses |
| TLS 1.1 | 2006 | Deprecated | RC4, limited cipher suite improvement over 1.0 |
| TLS 1.2 | 2008 | Current (with care) | Safe with strong cipher suites; weak with RC4 or export ciphers |
| TLS 1.3 | 2018 | Recommended | No known practical attacks; forward secrecy mandatory |

**BEAST (TLS 1.0 + CBC):** An attacker with network access can decrypt TLS 1.0 sessions using CBC-mode ciphers by exploiting a predictable IV. Practical against browser sessions, HTTPS, and any CBC cipher on TLS 1.0.

**POODLE (SSL 3.0 + CBC):** Protocol downgrade causes the client to fall back to SSL 3.0 which uses CBC with broken padding. Session cookies and credentials can be extracted.

**DROWN (SSLv2 server present):** Even if the target's primary HTTPS server runs TLS 1.2, if *any* server sharing the same certificate also supports SSLv2, the certificate's private key can be attacked via the SSLv2 server and used to decrypt TLS 1.2 sessions captured from the primary server.

**CRIME/BREACH (TLS compression):** If TLS compression is enabled and the attacker can inject content into the request, session tokens can be extracted byte by byte.

**Cipher suite weaknesses:**
- **RC4:** Statistically biased stream cipher — broken, should not appear in any modern configuration.
- **NULL ciphers:** No encryption at all. Technically valid TLS without confidentiality protection.
- **EXPORT ciphers:** Intentionally weakened 40-bit or 56-bit ciphers mandated by 1990s US export law — **FREAK** and **Logjam** attacks exploit servers that still advertise them.
- **Anonymous DH (ADH):** No authentication — susceptible to trivial MITM with no certificate validation.
- **3DES (SWEET32):** 64-bit block cipher; birthday attacks against long sessions allow plaintext recovery.

## 2.4 Reading TLS data from Shodan and Censys passively

Shodan indexes TLS handshake data for every HTTPS port it scans. The `ssl` filter queries this data:

```
ssl.version:sslv2 org:"Target Corp"
ssl.version:tlsv1 org:"Target Corp"
ssl.cipher.name:RC4 org:"Target Corp"
ssl.cert.expired:true org:"Target Corp"
```

Censys similarly indexes TLS certificate and cipher data:
```
parsed.names: corp-target.com AND protocols: "443/https"
```
The Censys result includes `cipher_suite.name`, `version`, `certificate.parsed.validity`, and whether the certificate is trusted.

This is fully passive — you are reading Shodan/Censys data, not connecting to the target. Before any active TLS auditing tool touches the target, you already know which hosts are running broken protocol versions.

## 2.5 Dead-end vs high-value finding

**Dead-end:** Shodan shows `203.0.113.45:443` serving `TLSv1.2` with cipher `TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384`. Certificate is valid, not expired, properly signed. No SSLv2 or TLSv1.0. No EXPORT ciphers. Nothing actionable — note it and move on.

**High-value:** Shodan shows `198.51.100.5:443` with `TLSv1.0` still enabled. Censys confirms the same host also responds on port 21 with an FTP banner `220 vsftpd 2.0.8`. The host's Shodan SSL data shows `ssl.version:sslv3` is accepted. This single host has three protocol-level weaknesses: cleartext FTP credentials, SSLv3 (POODLE), and TLS 1.0 (BEAST). It is a management server for a legacy application based on the subdomain from note 03: `legacy.corp-target.com`. Credentials captured here may reuse across other services.

## 2.6 Where results feed next

Protocol weaknesses feed Phase 3 directly. An FTP server with anonymous login gets tested in Stage 3 active probing. A host with SSLv2 gets queued for DROWN analysis if it shares certificates with other hosts. TLS 1.0-only hosts become targets for BEAST testing during web application assessment. The protocol audit converts Shodan banner data into a prioritized list of technique-specific attack vectors.

---

## 2.7 SMTP, IMAP, and POP3 TLS audit

Mail protocols are routinely overlooked in protocol audits that focus exclusively on HTTPS. SMTP (ports 25, 465, 587), IMAP (ports 143, 993), and POP3 (ports 110, 995) all have TLS implementations that may be absent, misconfigured, or running weak cipher suites — particularly on self-hosted mail servers and legacy on-premise Exchange deployments identified during Stage 1 mail posture recon.

**STARTTLS downgrade attack:** SMTP with STARTTLS begins as a plaintext session and optionally upgrades to TLS when both endpoints support it. If the upgrade is not enforced by MTA-STS policy, a man-in-the-middle attacker can strip the STARTTLS advertisement and maintain a plaintext session, capturing authentication credentials and email content in transit. The vulnerability is not in the TLS implementation itself but in the unenforced upgrade mechanism.

**MTA-STS (Mail Transfer Agent Strict Transport Security)** is the SMTP equivalent of HSTS for web: a DNS-declared policy that requires STARTTLS and validates the server's certificate. Its presence or absence is determined from a TXT record at `_mta-sts.<domain>` and a policy file at `https://mta-sts.<domain>/.well-known/mta-sts.txt`. Absence of MTA-STS means STARTTLS downgrade is possible on any network path between mail servers.

For mail servers identified during Stage 1 (mail posture recon note) or Shodan searches, the TLS audit covers:
- Does the SMTP submission port (587) enforce TLS? Does it accept SSLv3 or TLS 1.0?
- Does the IMAP port (993) serve a valid certificate with strong ciphers?
- Is STARTTLS on port 587 enforced or optional? Is MTA-STS configured?
- What cipher suites does the mail server accept? Are there 3DES or RC4 offerings?

## 2.8 RDP security audit

Remote Desktop Protocol (RDP) on port 3389 exposed to the internet is a critical finding independent of its TLS configuration. But the security of the RDP implementation matters additionally: older Windows Server versions support NLA-less (Network Level Authentication-less) connections, weak encryption modes, and have specific CVEs enabling remote code execution without authentication.

**BlueKeep (CVE-2019-0708)** is a pre-authentication, wormable RCE vulnerability in Windows Server 2008 R2, Windows 7, and earlier versions. It affects RDP implementations that do not use NLA. Shodan explicitly indexes BlueKeep-vulnerable hosts: `port:3389 vuln:CVE-2019-0708`. If a target has RDP-exposed legacy servers, this is a maximum-severity, zero-click finding.

**DejaBlue (CVE-2019-1181/1182)** extends similar vulnerabilities to Windows Server 2019 and Windows 10 without NLA. Both BlueKeep and DejaBlue are pre-NLA-authentication vulnerabilities, meaning NLA enforcement would have mitigated them. Hosts where NLA is not enforced are exposed to both.

**Network Level Authentication (NLA):** RDP without NLA presents the Windows login screen before authentication completes, allowing username enumeration from the login UI, brute-force attacks without full session setup, and exploitation of pre-auth vulnerabilities. NLA moves authentication before the RDP session is established, requiring valid credentials before any graphical interaction. NLA enforcement is detectable passively from Shodan banner data and actively from nmap scripts.

```bash
# Shodan BlueKeep search
$ shodan search 'port:3389 org:"Target Corp" vuln:CVE-2019-0708' --fields ip_str,os,version

# Shodan all RDP hosts in ASN
$ shodan search 'port:3389 asn:AS12345' --fields ip_str,os,product,screenshot.text

# Active NLA detection with nmap (generates connection to target)
$ nmap --script rdp-enum-encryption -p 3389 203.0.113.45
PORT     STATE SERVICE
3389/tcp open  ms-wbt-server
| rdp-enum-encryption:
|   Security layer
|     CredSSP (NLA): SUCCESS   ← NLA supported
|     RDP Security Layer: SUCCESS   ← fallback to old RDP security also works
|_  Encryption level: High (128 bits)
```
If both NLA and the legacy RDP security layer succeed, the server accepts non-NLA connections as fallback — pre-auth vulnerability exposure exists even if NLA is the preferred mode. Defenders who think NLA is enforced may not have disabled the fallback.

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **FTP** | File Transfer Protocol — plaintext, port 21; credentials and data transmitted unencrypted |
| **SFTP** | SSH File Transfer Protocol — runs over SSH port 22; fully encrypted; unrelated to FTP |
| **FTPS** | FTP Secure — FTP with TLS; implicit on port 990, explicit STARTTLS on port 21 |
| **TLS (Transport Layer Security)** | Cryptographic protocol providing encrypted, authenticated transport; successor to SSL |
| **SSL (Secure Sockets Layer)** | Predecessor to TLS; all versions (2.0, 3.0) are cryptographically broken |
| **Cipher suite** | Ordered combination of key exchange, authentication, encryption, and MAC algorithms negotiated in TLS |
| **BEAST** | Browser Exploit Against SSL/TLS — CBC padding attack against TLS 1.0 |
| **POODLE** | Padding Oracle On Downgraded Legacy Encryption — CBC attack against SSL 3.0 |
| **DROWN** | Decrypting RSA with Obsolete and Weakened eNcryption — cross-protocol attack via SSLv2 |
| **CRIME** | Compression Ratio Info-leak Made Easy — exploits TLS compression to extract session tokens |
| **FREAK** | Factoring RSA Export Keys — exploits EXPORT cipher suites to downgrade to 512-bit RSA |
| **Logjam** | Exploits DHE EXPORT cipher suites to downgrade Diffie-Hellman to 512-bit; affects servers accepting DHE_EXPORT |
| **SWEET32** | Birthday attack against 64-bit block ciphers (3DES) in long TLS sessions |
| **RC4** | Broken stream cipher; statistical biases allow plaintext recovery — must not appear in any config |
| **Perfect Forward Secrecy (PFS)** | Key exchange property ensuring session keys cannot be decrypted even if the server's private key is later compromised; provided by ECDHE and DHE |
| **STARTTLS** | Protocol upgrade mechanism — starts connection in plaintext then negotiates TLS; used by SMTP, FTP, IMAP |
| **HSTS** | HTTP Strict Transport Security — browser policy preventing protocol downgrade to HTTP |
| **ALPN** | Application-Layer Protocol Negotiation — TLS extension declaring which application protocol the server supports (HTTP/1.1, HTTP/2, etc.) |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `testssl.sh` | `testssl.sh https://corp-target.com` | Full TLS audit: versions, ciphers, known vulnerabilities, cert details | Comprehensive active TLS audit |
| `testssl.sh` | `testssl.sh --severity HIGH corp-target.com:443` | Only HIGH/CRITICAL severity findings | Fast pass, prioritized output |
| `sslscan` | `sslscan corp-target.com:443` | Supported TLS versions, accepted cipher suites, cert info | Quick TLS sweep |
| `openssl` | `openssl s_client -connect corp-target.com:443 -tls1` | Checks if TLS 1.0 is accepted (connection succeeds = yes) | Version-specific quick test |
| `openssl` | `openssl s_client -connect corp-target.com:443 -ssl3` | Checks if SSL 3.0 is accepted | POODLE surface check |
| `nmap` | `nmap --script ssl-enum-ciphers -p 443 corp-target.com` | List cipher suites accepted per TLS version | Cipher suite inventory |
| `nmap` | `nmap --script ftp-anon,ftp-bounce -p 21 corp-target.com` | Anonymous FTP login and bounce checks | FTP quick assessment |
| Shodan search | `ssl.version:sslv3 org:"Target Corp"` | Hosts in org still accepting SSL 3.0 | Passive pre-assessment |
| Shodan search | `port:21 product:vsftpd org:"Target Corp"` | FTP servers in org's IP space | Cleartext FTP identification |

**testssl.sh — reading the output:**
```bash
$ testssl.sh corp-target.com:443

 Testing protocols via sockets except NPN+ALPN
 SSLv2      not offered (OK)
 SSLv3      not offered (OK)
 TLS 1      offered (deprecated)    ← FLAGGED — BEAST possible
 TLS 1.1    offered (deprecated)    ← FLAGGED
 TLS 1.2    offered (OK)
 TLS 1.3    offered (OK)

 Testing cipher order
 TLSv1.2   ECDHE-RSA-AES256-GCM-SHA384 (server preferred) ← Strong
 TLSv1.0   DES-CBC3-SHA                                   ← SWEET32 vulnerable

 Testing vulnerabilities
 BEAST:     VULNERABLE -- but only if you use a browser (CVE-2011-3389)
 POODLE:    not vulnerable (OK)
 SWEET32:   VULNERABLE, uses 64 bit block ciphers         ← Critical finding
 FREAK:     not vulnerable (OK)

 Testing certificate
 CN:        corp-target.com
 SAN:       corp-target.com, api.corp-target.com, staging.corp-target.com
 Expiry:    2025-01-15 (120 days)
 Issuer:    Let's Encrypt
```
Two vulnerabilities: TLS 1.0 still enabled (BEAST) and 3DES in the cipher suite (SWEET32). Three subdomains exposed in SAN. The cert expires in 120 days — not a finding, but worth noting.

**sslscan — cipher suite list:**
```bash
$ sslscan corp-target.com:443
Supported Server Cipher(s):
  Preferred  TLSv1.3  256 bits  TLS_AES_256_GCM_SHA384
  Accepted   TLSv1.2  256 bits  ECDHE-RSA-AES256-GCM-SHA384
  Accepted   TLSv1.2  128 bits  ECDHE-RSA-AES128-GCM-SHA256
  Accepted   TLSv1.2  112 bits  DES-CBC3-SHA               ← 3DES, SWEET32
  Accepted   TLSv1.0  112 bits  DES-CBC3-SHA               ← Old version + weak cipher
  Accepted   TLSv1.0  128 bits  AES128-SHA                 ← No PFS (no ECDHE)
```
Sort by strength — the `112 bits DES-CBC3-SHA` on both TLS 1.0 and 1.2 is the critical finding. No PFS on the AES128-SHA cipher means sessions recorded today can be decrypted if the private key is later compromised.

**openssl — quick version check:**
```bash
$ openssl s_client -connect corp-target.com:443 -tls1 2>&1 | grep "Protocol"
Protocol  : TLSv1    ← TLS 1.0 accepted — BEAST surface

$ openssl s_client -connect corp-target.com:443 -ssl3 2>&1 | grep "handshake failure\|Protocol"
140...ssl handshake failure   ← SSLv3 rejected — good
```

**nmap cipher enumeration:**
```bash
$ nmap --script ssl-enum-ciphers -p 443 corp-target.com
PORT    STATE SERVICE
443/tcp open  https
| ssl-enum-ciphers:
|   TLSv1.0:
|     ciphers:
|       TLS_RSA_WITH_AES_128_CBC_SHA (rsa 2048) - C     ← Grade C: no PFS
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA (rsa 2048) - C    ← 3DES, SWEET32
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 - A       ← Strong
```

**FTP anonymous login check:**
```bash
$ nmap --script ftp-anon -p 21 198.51.100.5
21/tcp open  ftp
| ftp-anon: Anonymous FTP login allowed
|   drwxr-xr-x  3  ftp  ftp  4096  backups/
|   -rw-r--r--  1  ftp  ftp  18432 database_backup_2023.sql
```
Anonymous FTP access with what appears to be a database backup in a world-readable directory. Maximum severity finding.

---

**testssl.sh against mail protocols (SMTP/IMAP/POP3):**
```bash
# SMTP submission (port 587) with STARTTLS
$ testssl.sh --starttls smtp mail.corp-target.com:587
 Testing protocols via sockets (with --starttls smtp)
 SSLv2:      not offered (OK)
 SSLv3:      not offered (OK)
 TLS 1:      offered (deprecated)   ← FLAGGED
 TLS 1.1:    offered (deprecated)   ← FLAGGED
 TLS 1.2:    offered (OK)
 TLS 1.3:    offered (OK)
 STARTTLS:   available
 MTA-STS:    not configured          ← downgrade possible

# IMAP with TLS (port 993)
$ testssl.sh --starttls imap mail.corp-target.com:143
# Or implicit TLS on 993:
$ testssl.sh mail.corp-target.com:993

# POP3 TLS (port 995)
$ testssl.sh mail.corp-target.com:995

# Check MTA-STS policy
$ dig +short TXT _mta-sts.corp-target.com
# Empty = no MTA-STS = STARTTLS downgrade is possible

$ curl -s https://mta-sts.corp-target.com/.well-known/mta-sts.txt
# 404 or connection refused = MTA-STS policy file not configured
```
`TLS 1.0 offered` on SMTP port 587 combined with no MTA-STS means: (a) an attacker on the network path can downgrade the session to TLS 1.0, (b) the BEAST attack applies to TLS 1.0 CBC sessions, and (c) captured TLS 1.0 sessions with CBC ciphers can be decrypted. Credential interception is viable for organizations using this mail server for client authentication.

**Manual STARTTLS banner grab and cipher negotiation:**
```bash
# Manual STARTTLS handshake (verify the SMTP banner and STARTTLS advertisement)
$ openssl s_client -connect mail.corp-target.com:587 -starttls smtp 2>&1 \
  | grep -E "Protocol|Cipher|250|220|STARTTLS"
220 mail.corp-target.com ESMTP
250-STARTTLS                   ← server advertises STARTTLS
Protocol  : TLSv1.2
Cipher    : AES128-SHA          ← no ECDHE = no PFS

# Force TLS 1.0 to check acceptance
$ openssl s_client -connect mail.corp-target.com:587 -starttls smtp -tls1 2>&1 | grep Protocol
Protocol  : TLSv1               ← TLS 1.0 accepted on mail server

# Test RDP protocol encryption level (active probe)
$ nmap --script rdp-enum-encryption,rdp-vuln-ms12-020 -p 3389 198.51.100.5
```

# Section 5 — Defender detection

Running testssl.sh, sslscan, and openssl s_client makes **direct TCP connections** to the target. These are no longer passive — they are semi-active at minimum and generate connection records in the target's firewall and web server access logs.

- **testssl.sh** makes dozens of TLS handshake attempts (one per protocol version, one per cipher suite being tested). The source IP appears in the target's access logs for every probe. The pattern of rapid repeated TLS handshakes from one source IP is visible and recognizable to an IDS with TLS enumeration signatures.
- **sslscan** similarly makes multiple handshake attempts. It is faster and generates a shorter burst of connections but still leaves the source IP in logs.
- **nmap SSL scripts** make TCP connections and appear in logs as any other scanner.
- **Passive Shodan approach:** Before running any active TLS tool, query Shodan's already-collected TLS data. This generates zero connections to the target. Use active tools only when you need confirmation of Shodan data or Shodan's data is stale.
- **Defenders that detect:** IDS rules (Snort/Suricata) have signatures for TLS enumeration patterns. WAFs may block IPs that send many rapid TLS handshakes. Some organizations rate-limit or block IPs that attempt `ssl3` or `tls1` on ports where those versions are disabled.
- **Mitigation for operators:** Run TLS audits from a non-attributed exit IP. Use `testssl.sh --sneaky` to space probes. Check Shodan data first so you already know the TLS configuration before connecting.

---

# Section 6 — Lab task

**Platform:** Kali Linux VM. For a safe target: spin up a local Docker container with a deliberately weak TLS configuration.

```bash
docker run -d -p 4433:443 -e CERT_TYPE=rsa2048 drwetter/testssl.sh badssl.com
# Or use badssl.com directly — it is a public intentionally-broken TLS site maintained for testing
```

Alternatively, TryHackMe rooms covering TLS vulnerabilities provide authorized targets.

**Objective:** Run a complete TLS audit against a deliberately weak TLS endpoint, identify every protocol-level weakness, map cipher suite grades, and produce a prioritized finding table.

**Steps:**

1. Install tools: `apt install testssl.sh sslscan nmap` (testssl.sh: `git clone --depth 1 https://github.com/drwetter/testssl.sh.git`)
2. Passive check first — query Shodan for any existing TLS data: `shodan search 'host:testphp.vulnweb.com' --fields port,ssl.version,ssl.cert.subject.cn`
3. Run testssl.sh: `testssl.sh https://expired.badssl.com 2>&1 | tee testssl_output.txt`
4. Run sslscan: `sslscan expired.badssl.com:443 | tee sslscan_output.txt`
5. Check specific weak versions: `openssl s_client -connect expired.badssl.com:443 -tls1 2>&1 | grep -E "Protocol|Cipher|CONNECTED"` and repeat for `-tls1_1` and `-ssl3`
6. Run nmap cipher enumeration: `nmap --script ssl-enum-ciphers -p 443 expired.badssl.com | tee nmap_ciphers.txt`
7. Check for FTP on any port (if lab has it): `nmap --script ftp-anon,ftp-bounce -p 21 <target>`
8. Identify all flagged vulnerabilities from testssl output: `grep -E "VULNERABLE|WARN|NOT OK" testssl_output.txt`
9. Classify each finding: Protocol weakness / Cipher weakness / Certificate issue / FTP exposure
10. Populate `protocol_findings.md` with columns: Host | Port | Issue | CVE | Severity | Evidence

**Expected output:** `testssl_output.txt` with at least two flagged findings, `protocol_findings.md` with a correctly graded finding table, and a one-paragraph attack narrative explaining which finding you would prioritize and why.

**Git artifact:**
```
recon/stage2/protocol-audit/
├── testssl_output.txt
├── sslscan_output.txt
├── nmap_ciphers.txt
└── protocol_findings.md
```
```bash
git commit -m "recon(stage2): protocol audit — TLS/FTP weakness analysis for <target>"
```

---

# Section 7 — Common mistakes

**1. Treating TLS 1.2 as automatically safe**
_Why it matters:_ TLS 1.2 is the current supported standard, but it only provides security when configured correctly. TLS 1.2 with RC4, NULL, EXPORT, or ADH ciphers is as broken as older protocol versions. The protocol version alone is not sufficient — the cipher suite matters equally.
_Fix:_ Always check cipher suite details alongside protocol version. A TLS 1.2 server using `TLS_RSA_WITH_RC4_128_SHA` is broken regardless of the version number.

**2. Confusing SFTP with FTPS and treating both as equivalently secure to SFTP**
_Why it matters:_ SFTP runs over SSH and is fully encrypted with modern cryptography by default. FTPS is FTP with TLS — it is more secure than plaintext FTP but subject to all TLS configuration weaknesses. Recommending FTPS as equivalent to SFTP misses TLS misconfiguration risks.
_Fix:_ Treat SFTP as a distinct and preferred protocol. For FTPS findings, apply the same TLS version and cipher analysis as HTTPS.

**3. Not checking whether SSLv2 is supported on any host sharing a certificate**
_Why it matters:_ DROWN requires only that *any* server sharing the certificate's private key supports SSLv2 — even on a different port or different hostname. An organization's primary HTTPS may be TLS 1.2 clean, but if their mail server on port 465 still accepts SSLv2 using the same wildcard certificate, the HTTPS sessions are vulnerable.
_Fix:_ After identifying all hosts and certificates in the org, query Shodan for `ssl.version:sslv2` across the entire ASN and check whether any found certificates share subjects or SANs with the primary domain.

**4. Ignoring cipher suite ordering**
_Why it matters:_ Even if a server supports strong ciphers, if weak ciphers are listed first (server preferred order), clients may negotiate weaker ciphers depending on their own capability. A server that lists `DES-CBC3-SHA` before `AES-256-GCM-SHA384` will negotiate 3DES with any client that supports it.
_Fix:_ Record which cipher is the server's *preferred* choice, not just which ciphers it accepts. testssl.sh shows the server-preferred cipher explicitly.

**5. Running active TLS tools without noting the detection risk**
_Why it matters:_ testssl.sh makes tens of TLS handshake attempts in rapid succession. On an engagement, doing this from an uncleared IP during a stealth phase wastes the element of surprise and may trigger IDS/WAF responses that alert the target before the active phase begins.
_Fix:_ Complete all Shodan/Censys passive TLS analysis before running any active tool. Only run testssl.sh and sslscan when the active reconnaissance phase has officially begun and your IP is operationally cleared.

---

# Section 8 — Move-on gate

1. Run testssl.sh against `expired.badssl.com` without looking at notes, identify all flagged vulnerabilities in the output, and for each: state the vulnerability name, the CVE if applicable, and a one-sentence explanation of why it matters operationally.

2. Given a testssl.sh output showing `TLS 1.0 offered`, `DES-CBC3-SHA accepted`, and `RC4 accepted`, rank these three findings by severity and justify the ranking by explaining the specific attack that each enables and the conditions required.

3. Explain from memory the difference between FTP, FTPS, and SFTP — including which port each uses, whether credentials are encrypted, and which TLS attacks apply to FTPS that do not apply to SFTP.
