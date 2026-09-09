# 🛡️ Cyber Security Notes

Structured learning notes and hands-on laboratory documentation for offensive and defensive security operations.  
Organized into **5 linear execution stages (Modules 01–30)** plus a post-hire **Shelf**, matching the Master Roadmap.

Each module directory contains practice checklists, tool commands, lab output, network captures, and structured markdown summaries.

---

## 📑 Stages Overview

| Stage | Focus Area | Modules | Status |
|:-----:|------------|:-------:|:------:|
| **[Stage 1](#stage-1--foundation)** | System Internals, Networking, Crypto, Auth, Web | 01–07 | 🟡 Substantially Complete |
| **[Stage 2](#stage-2--offense-i)** | Host Recon, Scanning, Enumeration, Cracking, Initial Access | 08–13 | 🟢 Active Focus |
| **[Stage 3](#stage-3--web--application-security)** | Full-Stack Web Pentesting, APIs, JWTs, Bug Bounty, Defense Side-Track | 14–18 | ⚪ Pending |
| **[Stage 4](#stage-4--enterprise-infrastructure--identity)** | Active Directory, Cloud IAM, Containers, Adversary Emulation, Reporting | 19–26 | ⚪ Pending |
| **[Stage 5](#stage-5--advanced--specialized-operations)** | Custom C2 Development, Binary Exploitation, AI Red Teaming, Portfolio | 27–30 | ⚪ Pending |
| **[Shelf](#-shelf--post-hire--elective-specializations)** | Specialized & Post-Hire Tracks (Wireless, Mobile, Forensics, GRC, etc.) | S01–S17 | 📦 Off-Sequence |

---

### Stage 1 — Foundation
*Core engineering baselines required before offensive engagement.*

| Module | Directory | Key Focus |
|---|---|---|
| **01** | [Module-01_Fundamentals](Stage-1_Foundation/Module-01_Fundamentals/) | Hardware, CPU execution, OS internals, memory, data representation, basic scripting |
| **02** | [Module-02_Linux-Administration](Stage-1_Foundation/Module-02_Linux-Administration/) | Linux CLI mastery, permissions, process control, networking, services, system hardening |
| **03** | [Module-03_Windows-Administration](Stage-1_Foundation/Module-03_Windows-Administration/) | Windows management, Event Viewer, PowerShell, registry, Kerberos prerequisites |
| **04** | [Module-04_Networking](Stage-1_Foundation/Module-04_Networking/) | TCP/IP stack, routing, switching, DNS, Wireshark packet dissection, protocol analysis |
| **05** | [Module-05_Cryptography](Stage-1_Foundation/Module-05_Cryptography/) | Applied crypto, hashing, symmetric/asymmetric ciphers, TLS handshake, PKI |
| **06** | [Module-06_Authentication-Standards](Stage-1_Foundation/Module-06_Authentication-Standards/) | Sessions, JWTs, OAuth 2.0, OpenID Connect (OIDC), SAML, and MFA mechanics |
| **07** | [Module-07_Web-Technology-Fundamentals](Stage-1_Foundation/Module-07_Web-Technology-Fundamentals/) | HTTP/1.1 & HTTP/2, cookies, SOP, CORS, REST APIs, JSON data structures |

> **🏁 Exit Gate:** [Foundation Proof Gate](../Cyber-Security/Roadmap/Stage-1_Foundation.md#foundation-proof-gate) *(10 PCAPs, admin baselines, 3 automation scripts, lab topology report)*

---

### Stage 2 — Offense I
*Host reconnaissance, port scanning, service enumeration, credential acquisition, and initial shells.*

| Module | Directory | Key Focus |
|---|---|---|
| **08** | [Module-08_Footprinting-and-Reconnaissance](Stage-2_Offense-I/Module-08_Footprinting-and-Reconnaissance/) | Passive & active recon, OSINT, ASN profiling, DNS enumeration, TLS analysis |
| **09** | [Module-09_Scanning](Stage-2_Offense-I/Module-09_Scanning/) | Nmap scan techniques, timing templates, service fingerprinting, NSE scripting |
| **10** | [Module-10_Enumeration](Stage-2_Offense-I/Module-10_Enumeration/) | Service interrogation (SMB, RPC, NFS, SNMP, LDAP), directory busting (`ffuf`) |
| **11** | [Module-11_Database-Security](Stage-2_Offense-I/Module-11_Database-Security/) | SQLi enumeration, database privilege escalation, relational & NoSQL auditing |
| **12** | [Module-12_Password-Cracking](Stage-2_Offense-I/Module-12_Password-Cracking/) | Cryptographic hash identification, John the Ripper, Hashcat rule engines, wordlists |
| **13** | [Module-13_System-Hacking](Stage-2_Offense-I/Module-13_System-Hacking/) | Initial access, Linux/Windows privesc (SUID, Potato exploits), pivoting (`Ligolo-ng`) |

> **🏁 Exit Gate:** [Stage Gate 1](../Cyber-Security/Roadmap/Stage-2_Offense-I.md#stage-gate-1) *(Root a box cold, dump & crack hashes, demonstrate Linux & Windows privesc)*

---

### Stage 3 — Web & Application Security
*Full-stack web application penetration testing, API auditing, and bug bounty workflows.*

| Module | Directory | Key Focus |
|---|---|---|
| **14** | [Module-14_Web-Application-Hacking](Stage-3_Web-and-App-Sec/Module-14_Web-Application-Hacking/) | OWASP Top 10, SQLi, XSS, SSRF, IDOR, SSTI, Deserialization, race conditions |
| **15** | [Module-15_Session-Hijacking](Stage-3_Web-and-App-Sec/Module-15_Session-Hijacking/) | Session fixation, cookie hijacking, JWT forgery (`jwt-tool`), CSRF tokens |
| **16** | [Module-16_Web-Server-Hacking](Stage-3_Web-and-App-Sec/Module-16_Web-Server-Hacking/) | Web server misconfigurations, virtual hosts, directory traversal, HTTP smuggling |
| **17** | [Module-17_API-Security](Stage-3_Web-and-App-Sec/Module-17_API-Security/) | OWASP API Top 10, REST, GraphQL introspection, gRPC, cloud metadata SSRF |
| **18** | [Module-18_Bug-Bounty-Methodology](Stage-3_Web-and-App-Sec/Module-18_Bug-Bounty-Methodology/) | Mass attack surface discovery, parameter fuzzing, triage, PoC authoring |
| **Side-Track** | [Side-Track_Defense-Awareness](Stage-3_Web-and-App-Sec/Side-Track_Defense-Awareness/) | Detection Engineering (SIEM/Sigma), IDS/Firewalls (Snort/Suricata), CTI/OSINT |

> **🏁 Exit Gate:** [Stage Gate 2](../Cyber-Security/Roadmap/Stage-3_Web-and-App-Sec.md#stage-gate-2) *(3+ HTB writeups, OWASP practitioner labs, manual Burp exploit delivery)*

---

### Stage 4 — Enterprise Infrastructure & Identity
*Enterprise domain domination, cloud security posture, container breakout, and professional reporting.*

| Module | Directory | Key Focus |
|---|---|---|
| **19** | [Module-19_Active-Directory](Stage-4_Enterprise/Module-19_Active-Directory/) | Kerberos (Kerberoasting, AS-REP), ADCS abuse (ESC1–ESC13), BloodHound, DCSync |
| **20** | [Module-20_Cloud-Security](Stage-4_Enterprise/Module-20_Cloud-Security/) | AWS/Azure IAM privilege escalation, role assumption chaining, CIEM, metadata abuse |
| **21** | [Module-21_Container-Security](Stage-4_Enterprise/Module-21_Container-Security/) | Docker socket breakouts, `--privileged` escape, cgroups, Kubernetes RBAC auditing |
| **22** | [Module-22_Adversary-Emulation](Stage-4_Enterprise/Module-22_Adversary-Emulation/) | MITRE ATT&CK mapping, Atomic Red Team execution, purple team metric evaluation |
| **23** | [Module-23_Sniffing-and-Spoofing](Stage-4_Enterprise/Module-23_Sniffing-and-Spoofing/) | ARP poisoning, DNS spoofing, traffic manipulation, Responder, Bettercap |
| **24** | [Module-24_Social-Engineering](Stage-4_Enterprise/Module-24_Social-Engineering/) | Phishing infrastructure, pretexts, credential harvesting, physical assessments |
| **25** | [Module-25_Malware-Architecture](Stage-4_Enterprise/Module-25_Malware-Architecture/) | Architectural patterns, execution primitives, payload delivery, evasion concepts |
| **26** | [Module-26_Pentest-Reporting](Stage-4_Enterprise/Module-26_Pentest-Reporting/) | PTES/CVSS standards, executive debriefs, technical remediation documentation |

> **🏁 Exit Gate:** [Stage Gate 3](../Cyber-Security/Roadmap/Stage-4_Enterprise.md#stage-gate-3) *(Multi-forest AD compromise, BloodHound attack path, commercial pentest report)*

---

### Stage 5 — Advanced & Specialized Operations
*Offensive development, AI red teaming, adversary campaign operations, and portfolio validation.*

| Module | Directory | Key Focus |
|---|---|---|
| **27** | [Module-27_Offensive-Development](Stage-5_Specialized/Module-27_Offensive-Development/) | Win32 API, PE loaders, shellcode injection, NTDLL unhooking, AMSI/ETW bypass |
| **28** | [Module-28_AI-Red-Teaming](Stage-5_Specialized/Module-28_AI-Red-Teaming/) | LLM prompt injection, jailbreaking, RAG poisoning, model extraction, PyRIT/Garak |
| **29** | [Module-29_Red-Team-Operations](Stage-5_Specialized/Module-29_Red-Team-Operations/) | Multi-tier C2 infrastructure (Sliver/Mythic), redirectors, OPSEC, campaign management |
| **30** | [Module-30_Portfolio](Stage-5_Specialized/Module-30_Portfolio/) | Public security research, tool publishing, verified writeups, interview defense |

> **🏁 Final Gate:** [Mastery Capstone](../Cyber-Security/Roadmap/Stage-5_Specialized.md#final-gate) *(Custom C2 implant, published original AI exploit research, OSCP certification)*

---

### 📦 Shelf — Post-Hire & Elective Specializations
*Specialized and compliance tracks reserved for post-employment study.*

All post-hire notes live in [`Shelf_Post-Hire/`](Shelf_Post-Hire/):
- **S01–S02:** Wireless & Mobile Platform Pentesting
- **S03:** OT / ICS / SCADA Security
- **S04–S06:** Digital Forensics, Reverse Engineering & Modern Binary Exploitation
- **S07–S10:** Hardware Hacking, Physical Pentesting, Telecom (VoIP/5G), Blockchain
- **S11–S15:** GRC (ISO 27001/SOC 2), Supply Chain, DevSecOps, Secure Code Review, Security Architecture
- **S16–S17:** Security Operations Expansion & DoS Resilience

---

## 📁 Repository Directory Hierarchy

```text
Cyber-Security_Notes/
├── Stage-1_Foundation/
│   ├── Module-01_Fundamentals/
│   ├── Module-02_Linux-Administration/
│   ├── Module-03_Windows-Administration/
│   ├── Module-04_Networking/
│   ├── Module-05_Cryptography/
│   ├── Module-06_Authentication-Standards/
│   └── Module-07_Web-Technology-Fundamentals/
├── Stage-2_Offense-I/
│   ├── Module-08_Footprinting-and-Reconnaissance/
│   ├── Module-09_Scanning/
│   ├── Module-10_Enumeration/
│   ├── Module-11_Database-Security/
│   ├── Module-12_Password-Cracking/
│   └── Module-13_System-Hacking/
├── Stage-3_Web-and-App-Sec/
│   ├── Module-14_Web-Application-Hacking/
│   ├── Module-15_Session-Hijacking/
│   ├── Module-16_Web-Server-Hacking/
│   ├── Module-17_API-Security/
│   ├── Module-18_Bug-Bounty-Methodology/
│   └── Side-Track_Defense-Awareness/
├── Stage-4_Enterprise/
│   ├── Module-19_Active-Directory/
│   ├── Module-20_Cloud-Security/
│   ├── Module-21_Container-Security/
│   ├── Module-22_Adversary-Emulation/
│   ├── Module-23_Sniffing-and-Spoofing/
│   ├── Module-24_Social-Engineering/
│   ├── Module-25_Malware-Architecture/
│   └── Module-26_Pentest-Reporting/
├── Stage-5_Specialized/
│   ├── Module-27_Offensive-Development/
│   ├── Module-28_AI-Red-Teaming/
│   ├── Module-29_Red-Team-Operations/
│   └── Module-30_Portfolio/
└── Shelf_Post-Hire/
```
