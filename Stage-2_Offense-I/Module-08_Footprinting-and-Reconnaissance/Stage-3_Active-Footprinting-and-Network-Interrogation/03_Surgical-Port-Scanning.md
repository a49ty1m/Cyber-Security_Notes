# Surgical Port Scanning

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

Surgical port scanning is the active interrogation of TCP and UDP ports on identified live hosts to determine which services are listening, what software versions are running, and what operating system is present. Unlike a broad host sweep, surgical scanning is deliberate and targeted — you already know which hosts are live (from note 01) and now you are building a detailed service map of each.

This is the first technique in the recon chain that is unambiguously visible to the target. Every TCP SYN packet reaches the target's network interface. Every connection logged. Every IDS signature potentially triggered. Timing, technique selection, and scan scope are the operational levers between "invisible to IDS" and "instant alert."

```text
Recon Chain
──────────────────────────────────────────────────────────────────────
Stage 2 (Semi-Passive)     Stage 3 (Active)
Shodan port data  →  [Surgical Port Scanning]  →  Service enumeration
Subdomain enum         ↑ YOU ARE HERE              Vulnerability mapping
Protocol audit                                      Web app testing
                    nmap, masscan, unicornscan
──────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** Shodan gives you a stale view of open ports — what their crawler saw last time it scanned the IP, which may be weeks or months old. Services change. Ports open. Firewall rules are modified. Port scanning gives you ground truth about the current state of the attack surface, along with version information that Shodan frequently lacks. Without current version data, you cannot accurately map CVEs to targets.

---

# Section 2 — How attackers actually use this

## 2.1 Why scan type selection matters

Nmap offers multiple TCP scan types, each producing different traffic and different detection signatures. Choosing the wrong scan type on a firewalled or monitored network either produces misleading results or immediately triggers an IDS alert.

The three most operationally relevant scan types:

- **SYN scan (`-sS`):** Sends a TCP SYN, receives SYN-ACK (open) or RST (closed), then sends RST without completing the handshake. Never completes a TCP session — does not appear in application-level logs. Requires root/sudo. The stealthiest TCP scan type, but still logged by stateful firewalls.
- **Connect scan (`-sT`):** Completes the full TCP three-way handshake using the OS socket API. Works without root. Appears in the application's connection logs — the target application sees the connection. Louder than SYN scan but works through some firewalls that block half-open connections.
- **ACK scan (`-sA`):** Sends TCP ACK to probe firewall rule filtering, not to detect open ports. An ACK to a port with no prior connection is either RST'd (unfiltered — the port is reachable) or dropped (filtered). This maps firewall rules, not open services.

```text
SYN scan:      Client → SYN →  Target
               Client ← SYN-ACK (open) or RST (closed)
               Client → RST   (no handshake completion)
               → Stateful firewall logs: yes / Application logs: no

Connect scan:  Client → SYN → Target
               Client ← SYN-ACK
               Client → ACK    ← handshake complete
               Client → RST/FIN
               → Application logs: yes (connection logged)
```

## 2.2 Interpreting port states

Nmap reports five port states, each with a distinct implication:

| State | Meaning | Cause |
|-------|---------|-------|
| `open` | A service is listening and accepting connections | Port is live and reachable |
| `closed` | No service listening, but port is reachable | Target responds with RST; port accessible through firewall |
| `filtered` | Nmap cannot determine state — packets are dropped | Firewall silently drops probe; no response |
| `open|filtered` | UDP scan or some firewall configs | No response received — either open or filtered |
| `unfiltered` | ACK scan — port reachable but state unknown | Firewall allows ACK through; service state not determined |

`filtered` is the most significant state for attack planning: it means a firewall is in front of the port. `closed` ports are valuable too — they confirm the host is live and the firewall allows your probe through, which means the firewall policy is permissive for that probe type.

## 2.3 OS fingerprinting mechanics

Nmap's OS fingerprinting (`-O`) works by sending a series of carefully crafted probes — TCP, UDP, ICMP — and analyzing the target's responses. Different operating systems implement the TCP/IP stack with slightly different characteristics:

- **TCP ISN (Initial Sequence Number) generation** pattern and randomness
- **TCP window size** in SYN-ACK responses
- **IP TTL** in responses (Linux starts at 64, Windows at 128)
- **TCP Options** in SYN-ACK (MSS, SACK, Timestamps, NOP ordering)
- **ICMP response behavior** to malformed packets
- **RST data** content and flag ordering

Nmap compares the observed response fingerprint against its OS detection database (~5000+ fingerprints) and returns the best match with a confidence percentage. OS detection requires at least one open and one closed port to produce reliable results.

```bash
$ nmap -O 203.0.113.45
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
OS details: Linux 5.0 - 5.4
Network Distance: 3 hops
```

OS detection is probabilistic — `Aggressive OS guesses` with `<75%` confidence should be verified. The TTL value alone often gives you rough OS family: 64 TTL = Linux/Unix, 128 TTL = Windows, 255 TTL = network device.

## 2.4 Version detection and service fingerprinting

Version detection (`-sV`) goes beyond "port 443 is open" to "port 443 is running nginx/1.18.0 (Ubuntu)." It does this by:

1. Completing the TCP handshake
2. Sending service-specific probe strings
3. Analyzing the banner or response against the nmap-service-probes database

Version data is the bridge between port mapping and vulnerability identification. `Apache httpd 2.4.38` maps to a specific CVE list. `OpenSSH 7.2p2` is a specific version with known authentication bypass issues. Without version data, you have port numbers. With version data, you have a target list for the exploitation phase.

```bash
$ nmap -sV 203.0.113.45 -p 22,80,443
PORT    STATE SERVICE VERSION
22/tcp  open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http    Apache httpd 2.4.38 ((Ubuntu))
443/tcp open  ssl/http nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## 2.5 Timing templates and IDS evasion

Nmap's timing templates control the speed and aggressiveness of scanning. Faster scans are more likely to trigger IDS/IPS rate-based alerts. Slower scans reduce detection probability at the cost of time.

| Template | Flag | Rate | IDS detection risk | Use when |
|----------|------|------|-------------------|----------|
| Paranoid | `-T0` | 1 probe per 5 min | Minimal | Stealth above all else |
| Sneaky | `-T1` | 1 probe per 15 sec | Very low | Slow engagements |
| Polite | `-T2` | Moderate, reduces bandwidth | Low | Avoiding target resource exhaustion |
| Normal | `-T3` | Default | Medium | Standard balanced scan |
| Aggressive | `-T4` | Fast, assumes reliable network | High | Lab/known clean network |
| Insane | `-T5` | Maximum | Very high | Never on production targets |

Beyond timing, specific evasion techniques:

**Decoy scan (`-D`):** Include decoy IPs alongside your real source IP. The target receives probes from multiple IPs simultaneously, making source attribution harder. Real source IP is still in the mix — not anonymization, just noise.

```bash
$ nmap -sS -D 198.51.100.1,198.51.100.2,ME 203.0.113.45
# Target receives SYN packets from 198.51.100.1, .2, and your real IP
```

**Source port manipulation (`--source-port 53`):** Some legacy firewalls allow UDP/TCP 53 (DNS) inbound assuming it's DNS traffic. Setting your source port to 53 can bypass these rules.

**Fragmentation (`-f`):** Fragment TCP packets into 8-byte chunks. Old packet-reassembly–incapable IDS systems drop fragments and miss the probe. Modern IDS reassemble before inspecting — this technique is less effective against current systems.

## 2.6 Masscan for speed + nmap for detail

For large IP ranges, nmap is too slow. Masscan can scan the entire IPv4 internet in under 6 minutes at 10M packets/second. For a corporate /16 (65,536 IPs), masscan discovers all open ports in seconds. Nmap then performs detailed version and OS detection against the discovered open ports only.

```text
Workflow:
masscan → [all open ports, all IPs]  →  nmap -sV -sC [specific ports on specific IPs]
                                        → detailed version + scripts
```

This two-pass approach: wide fast (masscan) then deep targeted (nmap) — covers the full range without the per-port overhead of running nmap's version detection across all IPs.

## 2.7 Reading open/closed/filtered for firewall inference

The combination of port states across a scan tells you about the firewall architecture in front of the target:

```text
All ports filtered except 80, 443, 22 → Stateful firewall, allowlist approach
All ports closed (RST) across all ports → No stateful firewall; host responds to all probes
Specific ports open, rest filtered/closed → Perimeter firewall with selective pass-through
Port 443 filtered but port 80 open → HTTPS forced to specific internal load balancer; HTTP externally accessible
All ports show open|filtered → Firewall dropping all probes; host may be protected by IPS in drop mode
```

This inference tells you what scanning technique to use next: if everything is filtered, switch to TCP ACK scan to map firewall rules. If everything is closed, no firewall is present and aggressive scanning is safe. If some ports are open, the firewall is a whitelist-style stateful device.

## 2.8 Nmap Scripting Engine (NSE) for targeted enumeration

NSE scripts extend nmap beyond port detection into service-specific enumeration. After identifying open ports and versions, running targeted NSE scripts extracts deeper intelligence:

```bash
# Default scripts (safe, informational)
$ nmap -sC 203.0.113.45 -p 22,80,443

# SMB enumeration (critical for Windows targets)
$ nmap --script smb-enum-shares,smb-vuln-ms17-010 -p 445 203.0.113.45

# HTTP title and server header
$ nmap --script http-title,http-server-header -p 80,443,8080 203.0.113.45

# SSH version and auth methods
$ nmap --script ssh-auth-methods,ssh-hostkey -p 22 203.0.113.45

# FTP anonymous login
$ nmap --script ftp-anon,ftp-bounce -p 21 203.0.113.45
```

NSE scripts are categorized by risk: `safe`, `default`, `intrusive`, `vuln`, `exploit`. The `vuln` and `exploit` categories actively attempt to exploit vulnerabilities — never run these without explicit authorization. The `safe` and `default` categories are informational and appropriate for reconnaissance phases.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **TCP SYN scan** | Half-open scan — sends SYN, reads SYN-ACK or RST, never completes handshake; requires root |
| **TCP Connect scan** | Full three-way handshake using OS socket API; no root required; visible in application logs |
| **TCP ACK scan** | Probes firewall filtering rules, not open ports; determines if port is filtered or unfiltered |
| **FIN/NULL/Xmas scans** | Exploit RFC 793 behavior: send unusual flag combinations to closed ports (expect RST) and open ports (expect silence); IDS-evasive on some systems |
| **UDP scan** | Sends empty UDP packet (or protocol-specific payload); open ports are silent, closed ports return ICMP Port Unreachable |
| **OS fingerprinting** | Inferring OS type from TCP/IP stack behavioral signatures (TTL, window size, TCP options, ISN randomness) |
| **Version detection** | Banner grabbing + probe-and-response analysis to identify the specific software and version on a port |
| **NSE (Nmap Scripting Engine)** | Lua-based scripting layer for targeted service enumeration, vulnerability checking, and brute-force |
| **Masscan** | Asynchronous packet scanner capable of scanning millions of IPs per minute |
| **Timing template** | Nmap's `-T0` through `-T5` flags controlling scan speed vs. detection risk |
| **TTL (Time to Live)** | IP header field decremented by each router hop; starting value reveals OS family (64=Linux, 128=Windows, 255=network device) |
| **Banner grabbing** | Reading the text a service sends immediately upon connection (SSH version string, HTTP Server header, SMTP greeting) |
| **Port state: filtered** | Nmap received no response — firewall is silently dropping probes |
| **Port state: closed** | Target responded with TCP RST — port accessible through firewall but no service listening |
| **Decoy scan** | Injecting additional source IPs into scans to obscure the real scanner's IP |
| **Fragmentation** | Splitting probe packets into smaller fragments to evade signature-based IDS that cannot reassemble them |
| **Source port spoofing** | Setting scan source port to a trusted value (e.g. 53) to pass through legacy firewall rules |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use |
|------|---------|-------------------|------------|
| `nmap` | `nmap -sS -p- --min-rate 1000 203.0.113.45` | All 65535 TCP ports, SYN scan | Full port sweep |
| `nmap` | `nmap -sV -sC -p 22,80,443 203.0.113.45` | Version + default scripts on known ports | Service fingerprinting |
| `nmap` | `nmap -O --osscan-guess 203.0.113.45` | OS fingerprint with aggressive guess | OS identification |
| `nmap` | `nmap -sU --top-ports 100 203.0.113.45` | Top 100 UDP ports | UDP service discovery |
| `nmap` | `nmap -sA -p 22,80,443 203.0.113.45` | Firewall rule mapping (ACK scan) | Firewall architecture inference |
| `nmap` | `nmap -T2 -sS --source-port 53 -D RND:5 203.0.113.45` | Stealthy scan with decoys and fake source port | IDS evasion |
| `masscan` | `masscan 203.0.113.0/24 -p0-65535 --rate=10000` | All open TCP ports across CIDR | Fast wide scan |
| `nmap -oA` | `nmap -sV -sC -p- 203.0.113.45 -oA scan_results` | Save XML, grepable, and text output | Persistent output |

**Full port sweep + version scan workflow:**
```bash
# Step 1: Fast port discovery with masscan
$ masscan 203.0.113.0/24 -p0-65535 --rate=5000 -oG masscan_results.txt
# Output: hosts with open ports (fast, no version)

# Step 2: Extract IPs and ports for nmap
$ grep "open" masscan_results.txt | awk '{print $4}' | sort -u > live_hosts.txt
$ grep "open" masscan_results.txt | grep -oP 'Ports: \K[0-9]+' | sort -nu | tr '\n' ',' > open_ports.txt

# Step 3: Targeted nmap for version + scripts on discovered ports
$ nmap -sV -sC -p $(cat open_ports.txt) -iL live_hosts.txt -oA stage3_port_scan
```

**SYN scan with OS detection:**
```bash
$ sudo nmap -sS -O --osscan-guess -p- --min-rate 2000 203.0.113.45

Starting Nmap scan report for 203.0.113.45
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
3306/tcp open   mysql        ← MySQL internet-exposed
8080/tcp open   http-proxy
9200/tcp filtered elasticsearch

Device type: general purpose
Running: Linux 5.X
OS CPE: cpe:/o:linux:linux_kernel:5
Aggressive OS guesses: Linux 5.0-5.4 (97%)
```
`9200/tcp filtered` — Elasticsearch is present but firewalled. `3306/tcp open` — MySQL directly accessible from the internet. This is an immediate critical finding.

**Version detection + default scripts:**
```bash
$ nmap -sV -sC -p 22,80,443,3306,8080 203.0.113.45

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.2p2 Ubuntu (protocol 2.0)
| ssh-hostkey:
|   2048 a1:b2:c3:d4:e5:f6:... (RSA)
|   256 12:34:56:78:... (ECDSA)

80/tcp   open  http    Apache httpd 2.4.38
|_http-title: Corp Target Internal Portal
|_http-server-header: Apache/2.4.38 (Ubuntu)

443/tcp  open  ssl/http nginx 1.18.0 (Ubuntu)
| ssl-cert: Subject: commonName=corp-target.com
| Not valid before: 2024-01-01
| Not valid after: 2025-01-01

3306/tcp open  mysql   MySQL 5.7.33-0ubuntu0.18.04.1
| mysql-info:
|   Status: Autocommit
|   Capacity: 1000

8080/tcp open  http    Apache Tomcat 7.0.92    ← EOL version
```
Apache Tomcat 7.0.92 on port 8080 is End of Life — CVE list covers multiple RCEs including Java deserialization via Ghostcat (CVE-2020-1938).

**Firewall mapping with ACK scan:**
```bash
$ nmap -sA -p 22,80,443,8080,3306,9200 203.0.113.45
PORT     STATE      SERVICE
22/tcp   unfiltered ssh        ← port reachable (no stateful block)
80/tcp   unfiltered http
443/tcp  unfiltered https
8080/tcp unfiltered http-proxy
3306/tcp unfiltered mysql
9200/tcp filtered   wap-wsp    ← stateful firewall blocks this one
```
Ports 22–8080 and 3306 are unfiltered by ACK scan but 9200 is filtered. This means a firewall allows traffic to those ports but blocks 9200. Combined with the SYN scan showing 9200 as `filtered`, the Elasticsearch port is protected but all others are directly accessible.

**UDP top ports scan:**
```bash
$ sudo nmap -sU --top-ports 50 --min-rate 500 203.0.113.45

PORT    STATE         SERVICE
53/udp  open          domain        ← DNS server
161/udp open          snmp          ← SNMP — check community string
500/udp open|filtered isakmp        ← VPN/IPSec
69/udp  open|filtered tftp          ← TFTP — no authentication
123/udp open          ntp
```
SNMP on 161/UDP and TFTP on 69/UDP are both critical findings. SNMP default community string `public` gives network topology information. TFTP allows unauthenticated file read/write.

---

# Section 5 — Defender detection

Port scanning is the most detectable activity in the recon chain. Every single probe generates traffic that reaches the target network.

- **IDS signatures:** Snort and Suricata ship with default signatures for nmap scanning patterns: SYN scan bursts, XMAS packets, NULL packets, FIN scans, OS fingerprint probe sequences (the nmap T_INIT+, ECN+, etc. probes). A default Snort installation fires on nmap -sS within seconds of a standard scan.
- **Firewall connection logs:** Every TCP SYN to a port — open, closed, or filtered — reaches the target's perimeter device and generates a log entry: source IP, destination IP, destination port, timestamp, protocol. These logs are retained. If the engagement is discovered, the scan logs provide a complete timeline of your activity.
- **Rate detection:** `nmap -T4` or `nmap -T5` sends hundreds of packets per second. Any SIEM with a "high port scan rate from single source" alert fires immediately. `-T2` reduces rate below most thresholds. `-T0` is effectively undetectable by rate-based rules but takes days for a full port scan.
- **Decoys reduce attribution but not detection:** `-D RND:5` adds 5 random source IPs. The target's IDS still alerts on the scan pattern. It just cannot determine which IP is the real scanner without additional correlation (they may use BGP routing tables or reverse DNS to identify your real source).
- **SYN scan vs Connect scan detection difference:** SYN scan does not appear in application-layer logs (the handshake was never completed). Connect scan appears in every listening application's connection log. On a hardened server, a burst of Connect scan connections to dozens of ports in seconds is very obvious.
- **NSE intrusive/vuln scripts:** Scripts in the `vuln` and `exploit` categories send actual exploit payloads. These are not reconnaissance — they are attacks. Running `--script vuln` against a production target generates high-severity IDS alerts and may crash services. Never use without explicit authorization.
- **Mitigation for operators:** (1) Run masscan at low rate (`--rate=100`) on initial sweep. (2) Use `-T2` nmap for stealthy version scans. (3) Scan from a cleared IP or VPN exit not linked to your real identity. (4) Use `--source-port 53` if legacy firewall rules allow DNS source IPs. (5) Distribute scans across multiple source IPs if available.

---

# Section 6 — Lab task

**Platform:** TryHackMe — "Nmap" room or "Nmap Advanced" room. Alternatively: set up a Metasploitable2 VM in VirtualBox and scan it from your Kali VM.

**Objective:** Execute a complete surgical port scan workflow — wide port sweep, targeted version scan, OS fingerprint, UDP services, firewall mapping — and produce a full service inventory document.

**Steps:**

1. **Install targets:** Start Metasploitable2 in VirtualBox. Note its IP (`ifconfig` in the VM).
2. **Initial masscan sweep:** `sudo masscan <target_ip> -p0-65535 --rate=1000 -oG masscan_out.txt`
3. **SYN scan (verify masscan results):** `sudo nmap -sS -p- --min-rate 1000 <target_ip> -oG nmap_syn.txt`
4. **Targeted version scan on discovered ports:** `sudo nmap -sV -sC -p <comma-list-of-open-ports> <target_ip> -oA version_scan`
5. **OS fingerprint:** `sudo nmap -O --osscan-guess <target_ip>`
6. **UDP top 50:** `sudo nmap -sU --top-ports 50 <target_ip>`
7. **ACK scan for firewall mapping:** `sudo nmap -sA -p <open-ports> <target_ip>` — note unfiltered vs filtered
8. **NSE default scripts on key services:** `sudo nmap -sC --script smb-enum-shares,ftp-anon,http-title -p 21,80,139,445 <target_ip>`
9. **Parse and triage output:** `grep "open" version_scan.gnmap | awk '{print $2, $3, $5, $6, $7}' | sort`
10. **Fill `service_inventory.md`** with table: IP | Port | State | Service | Version | CVE candidates | Priority

**Expected output:** `masscan_out.txt`, `version_scan.xml/.gnmap/.nmap`, `service_inventory.md` with at least 10 services mapped, OS identified, and 3 CVE candidates listed.

**Git artifact:**
```
recon/stage3/port-scan/
├── masscan_out.txt
├── version_scan.xml
├── version_scan.gnmap
├── version_scan.nmap
└── service_inventory.md
```
```bash
git commit -m "recon(stage3): surgical port scan — service inventory for <target>"
```

---

# Section 7 — Common mistakes

**1. Running nmap -T4 or -T5 against production targets**
_Why it matters:_ Aggressive timing sends hundreds of packets per second. Every SIEM with a basic port scan rule fires within seconds. Your engagement IP is blacklisted, the client's security team is alerted, and the engagement timeline is compressed. `-T4` is a lab timing — never use it on production targets without specific clearance.
_Fix:_ Default to `-T2` (polite) for production scanning. Use `-T3` only when the engagement specifically permits aggressive scanning. Reserve `-T4`/`-T5` for isolated lab VMs.

**2. Running nmap without sudo for SYN scans**
_Why it matters:_ SYN scan (`-sS`) requires raw socket access which requires root. Without sudo, nmap silently falls back to Connect scan (`-sT`), which completes full TCP handshakes and appears in application logs. Many operators don't notice the fallback and don't realize their "stealthy" SYN scan was actually a noisy Connect scan.
_Fix:_ Always run nmap with `sudo` for SYN scans. Confirm scan type in the nmap output header: `Scan type: SYN Stealth Scan` vs `Scan type: TCP Connect Scan`.

**3. Scanning all 65535 ports with nmap's default timing (no rate limit)**
_Why it matters:_ Nmap's default full-port scan on a slow network link is extremely slow (can take hours per host) and on a fast network it is rate-aggressive enough to trigger IDS. Neither extreme is useful.
_Fix:_ Use `--min-rate 1000` for a balanced speed that covers all ports in a reasonable time without hitting rate-based IDS thresholds. For stealthy scans, use masscan for the initial wide sweep and nmap only for targeted follow-up.

**4. Not scanning UDP**
_Why it matters:_ UDP scanning is slower and noisier, so many operators skip it. But critical services run on UDP: DNS (53), SNMP (161), TFTP (69), NTP (123), DHCP (67/68), RADIUS (1812), VPN protocols (500, 4500). A target with no interesting TCP services may have SNMP with default community string on UDP 161 — a complete network topology disclosure.
_Fix:_ Always run `nmap -sU --top-ports 100` against confirmed live hosts. Focus on the top 100 UDP ports rather than the full 65535 (UDP full scan is prohibitively slow and unreliable).

**5. Treating `filtered` as definitively no service**
_Why it matters:_ `filtered` means nmap received no response — but this is because a firewall dropped the probe, not because there's no service. The port may be wide open behind the firewall with a vulnerable service listening. Treating `filtered` as "nothing here" misses firewall-protected attack surface.
_Fix:_ Follow up on filtered ports with ACK scan to confirm the firewall state. Then check if alternative probe techniques (different source port, fragmented packets, ICMP vs TCP) can elicit a response. Document filtered ports as "protected service — unknown state" rather than "closed."

**6. Running NSE `--script vuln` without authorization**
_Why it matters:_ The `vuln` script category actively tests for vulnerabilities, sends exploit payloads, and may crash services. Running `--script vuln` against a production target without explicit authorization is an exploit attempt, not reconnaissance — it is illegal without written permission in most jurisdictions.
_Fix:_ Stick to `--script safe` and `--script default` for reconnaissance. Use `--script discovery` for service enumeration. Never run `vuln`, `exploit`, or `brute` scripts without the specific written authorization to do so.

**7. Not saving scan output in multiple formats**
_Why it matters:_ Nmap scans take time and generate irreplaceable data. Running a 4-hour scan and not saving the output means re-scanning if you need to reference a port state or version later. XML output is required for import into Metasploit, OpenVAS, and most vulnerability management tools.
_Fix:_ Always use `-oA <filename>` which saves `.nmap` (human-readable), `.xml` (machine-parseable), and `.gnmap` (grepable) simultaneously. Store all scan output in the Git artifact directory immediately after the scan completes.

---

# Section 8 — Move-on gate

1. Run a full TCP port scan (`-p-`) on Metasploitable2 using SYN scan, version detection, and OS fingerprinting in one nmap command — then state from memory the difference between `filtered`, `closed`, and `open` port states and name one attack implication of each state.

2. Given a nmap result showing `9200/tcp filtered elasticsearch` — explain what the `filtered` state tells you about the network, describe what an ACK scan against port 9200 would tell you that the SYN scan didn't, and name one technique to determine if there is an actual service behind the filter.

3. You need to scan a /24 range quickly but stealthily. Describe the two-tool workflow (masscan + nmap), state the exact flags you would use for each, and explain why running nmap directly on the full /24 all-ports is the wrong approach for this scenario.
