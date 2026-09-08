# Cyber Security Notes

Structured learning notes for cyber security, organized as a progressive path from technical foundations through offensive and defensive security practice.  
Each file is written as a practice checklist — work through questions yourself, build the projects, and mark them done as you go.

---

## Phases

### Phase 1 — Foundation
Builds the technical baseline required before any offensive or defensive work:

| Part | Content |
|---|---|
| **Part-1 Fundamentals** | Hardware, CPU, pre-boot, OS internals, memory management, data representation, wireless/physical, networking core, scripting (Python, JavaScript, Bash, PowerShell, C) |
| **Part-1B Linux Admin** | Linux administration, command-line mastery, system configuration |
| **Part-1C Windows Admin** | Windows administration, Active Directory, identity, Kerberos, PowerShell |
| **Part-2 Networking** | TCP/IP, protocols, routing, DNS, packet analysis, network security |
| **Part-3 Cryptography** | Applied cryptography, PKI, digital signatures, hashing, LUKS, post-quantum |
| **Part-3B Authentication Primer** | Sessions, JWTs, OAuth 2.0, OpenID Connect (OIDC), SAML, and MFA mechanics |
| **Part-3C Web Tech Fundamentals** | HTTP request/response architecture, browser security model, SOP, CORS, REST APIs |

### Phase 2 — Offensive Core
Applies foundational knowledge to structured host and network offensive security workflows:

| Part | Content |
|---|---|
| **Part-4 Footprinting** | Passive and active reconnaissance, OSINT, target profiling, infrastructure mapping |
| **Part-5 Scanning** | Port scanning, service detection, vulnerability scanning, NSE scripting |
| **Part-6 Enumeration** | Service enumeration (SMB, RPC, NFS, SNMP, LDAP), directory and attack surface mapping |
| **Part-6B Database Security** | SQL injection enumeration, relational and NoSQL database exploitation, auditing |
| **Part-31 Password Cracking** | Cryptographic hash identification, John the Ripper, Hashcat rule engines, wordlist curation |
| **Part-7 System Hacking** | Initial access, Linux/Windows privilege escalation (9 vectors, Potato family), pivoting (Chisel/Ligolo-ng) |
| **Part-8 Malware & Weaponization** | Architecture, execution mechanisms, evasion, counter-forensics (conceptual overview) |
| **Part-9 Sniffing & Spoofing** | Passive packet capture, ARP cache poisoning, DNS spoofing, traffic interception & MITM analysis |
| **Part-10 Social Engineering** | OSINT profiling, digital pretexts, physical security assessment, phishing analysis |
| **Part-11 Denial of Service** | Network availability, Layer 4/7 mechanisms, SYN cookies, Anycast, and DDoS mitigation |

### Phase 3 — Defense Core
Covers detection engineering, security operations, and incident response:

| Part | Content |
|---|---|
| **Part-13A Detection & SOC** | Defensive architecture, TTP detection, hardening, EDR/XDR, SIEM, threat hunting, IR, forensics |
| **Part-13B SecOps Expansion** | SOAR automation, DLP fundamentals, vulnerability management programs, insider threat detection |
| **Part-14 IDS/Firewall/Honeypots** | Firewall deployment, Snort/Suricata IDS/IPS, deception traps, email & DNS security |
| **Part-15 CTI & Attack Surface** | External attack surface management (EASM), threat actor profiling, STIX/TAXII, MISP & OpenCTI |

### Phase 4 — Web & Application Security
Full-stack web application penetration testing, API auditing, and bug bounty hunting:

| Part | Content |
|---|---|
| **Part-17 Web App Hacking** | OWASP Top 10, SQLi, XSS, SSRF, IDOR, authorization bypass, SSTI, Deserialization |
| **Part-12 Session Hijacking** | Cookie security attributes, session fixation, token forgery, JWT attacks (`jwt-tool`), CSRF |
| **Part-18 Web Server Hacking** | Web server architecture, configuration reviews, directory discovery, vhost fuzzing |
| **Part-19 API Security** | REST & GraphQL security, single-packet race conditions (Turbo Intruder), cloud SSRF (IMDSv2) |
| **Part-20 Bug Bounty Methodology** | Mass external asset discovery, parameter fuzzing, triage, PoC drafting, and report writing |

### Phase 5 — Wireless & Mobile Security [POST-HIRE]
Radio frequency, physical wireless protocols, and mobile client exploitation:

| Part | Content |
|---|---|
| **Part-21 Wireless Pentesting** | 802.11 a/b/g/n/ac/ax, WPA2/WPA3 enterprise, BLE, Zigbee, NFC/RFID, and SDR analysis |
| **Part-22 Mobile Platform Pentesting** | Android & iOS architecture, APK reverse engineering, Frida hooking, Objection runtime bypass |

### Phase 6 — Enterprise Infrastructure & Active Directory
Enterprise domain dominance, cloud security, and container environments:

| Part | Content |
|---|---|
| **Part-23 Active Directory** | Kerberos exploitation (Kerberoasting, AS-REP), ADCS certificate abuse (ESC1–ESC13), DCSync |
| **Part-24 Cloud Security & IAM** | AWS/Azure IAM privilege escalation, AssumeRole chaining, CIEM, OIDC federation |
| **Part-25 Container Security** | Dockerfile hardening, `--privileged` breakouts, cgroups, `/var/run/docker.sock`, K8s RBAC |
| **Part-16 Adversary Emulation** | MITRE ATT&CK mapping, Atomic Red Team execution, purple team detection measurement |
| **Part-26 OT/ICS/SCADA** | Modbus, DNP3, PLC/HMI exploitation, safety systems, and industrial network isolation |

### Phase 7 — Advanced Offensive Security & Binary Exploitation
Low-level reverse engineering, systems programming, and modern exploit development:

| Part | Content |
|---|---|
| **Part-42 Offensive Development** | Win32 API, PE loaders, shellcode execution, NTDLL unhooking, direct syscalls (`Syswhispers3`) |
| **Part-27 Digital Forensics** | Memory extraction (Volatility 3), timeline creation (Plaso), filesystem forensics (Autopsy) |
| **Part-28 Reverse Engineering** | Static/dynamic analysis (Ghidra, x64dbg), anti-analysis tricks, unpackers, malware triage |
| **Part-29 Modern Exploitation** | Linux x86-64 stack exploitation, ROP chaining, ASLR/DEP/Canary bypass, kernel debug (WinDbg) |
| **Part-30 Hardware Hacking** | UART/JTAG pinout extraction, firmware extraction/analysis, side-channel attacks |
| **Part-32 Physical Pentesting** | Lock picking, access control bypass, HID badge cloning, rogue drop-box implants |

### Phase 8 — Security Engineering & DevSecOps
Governance, secure systems architecture, and pipeline defense:

| Part | Content |
|---|---|
| **Part-35 GRC** | NIST CSF 2.0, ISO 27001, SOC 2, risk quantification (FAIR), compliance audit workflows |
| **Part-36 Supply Chain Security** | Software Bill of Materials (SBOM), dependency confusion, typosquatting, package integrity |
| **Part-37 DevSecOps** | SAST, DAST, SCA, secrets scanning, Poison Pipeline Execution (D-PPE, I-PPE) in GitHub Actions |
| **Part-37B Secure Code Review** | Manual code review, sink-to-source tracing, custom Semgrep rule authoring |
| **Part-43 Security Architecture** | Zero Trust Architecture, network segmentation, enterprise cryptographic key management |

### Phase 9 — AI Security & Red Teaming
Security of machine learning pipelines, LLMs, and autonomous AI agents:

| Part | Content |
|---|---|
| **Part-38 AI & LLM Security** | Prompt injection (direct/indirect), jailbreaking, RAG poisoning, model extraction, Garak/PyRIT |

### Phase 10 — Operations & Career
Commercial deliverable production, adversary campaign simulation, and professional proof of work:

| Part | Content |
|---|---|
| **Part-39 Pentest Methodologies** | PTES, NIST 800-115, CVSS v3.1/v4.0 scoring, professional report architecture, executive debriefs |
| **Part-40 Red Team Operations** | C2 infrastructure (Sliver, Mythic), redirector setups, OPSEC discipline, campaign deconfliction |
| **Part-41 Proof of Work & Portfolio** | Curated GitHub profiles, reproducible technical writeups, bug bounty validations, interview prep |

---

## Naming convention

| Term | Meaning |
|---|---|
| **Phase** | High-level learning milestone |
| **Part** | Domain area within a phase |
| **Stage** | Focused sub-topic sequence within a part |
| **Markdown file** | Practice checklist or reference for a specific concept or tool |

All content is written as practice questions rather than passive notes — the goal is active recall, not reading.
