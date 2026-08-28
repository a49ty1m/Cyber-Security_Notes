# Defensive Profiling

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 4: Advanced Fingerprinting & Logic Analysis

# Section 1 — What it is and where it sits

Defensive profiling is the systematic identification of the security stack protecting a target environment — specifically the EDR/AV, SIEM, IDS/IPS, SOAR, and firewall products in use — before executing any payload or exploit. It is the most critical piece of intelligence before transitioning from recon to active exploitation: the defensive stack determines your operational tempo, tooling choices, and evasion requirements.

The core principle: **you cannot evade what you don't know exists.** An operator who fires a Metasploit payload without knowing CrowdStrike Falcon is running will trigger a detection within seconds of execution. An operator who identifies the defensive stack first can choose appropriate evasion before touching a target system.

```text
Stage 4 → Defensive Profiling
────────────────────────────────────────────────────────────────────
CVE Correlation  →  [Defensive Profiling]  →  Exploitation Phase
found vulns           ↑ YOU ARE HERE          choose payloads + TTPs
attack candidates     EDR/SIEM/IDS/IPS        based on defensive stack
                      identified before        configure evasion
                      first payload            techniques
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 EDR/AV identification from external indicators

Before gaining any foothold, several external signals reveal which endpoint security products are deployed:

**Job listings as a detection method:**
The most reliable pre-access method. Job postings for IT/security roles explicitly list the tools the company uses. A role requiring "experience with CrowdStrike Falcon" confirms CrowdStrike deployment. LinkedIn and Indeed are searchable.

```bash
# Google dorking for job listings mentioning security tools
site:linkedin.com "corp-target" "CrowdStrike"
site:indeed.com "corp-target" "SentinelOne" OR "Carbon Black" OR "Defender"
site:glassdoor.com "corp-target" "security engineer"
site:lever.co OR site:greenhouse.io "corp-target.com" "EDR" OR "endpoint security"
```

**Technology intelligence platforms:**
- **BuiltWith.com** — shows technology stacks from web traffic analysis (primarily web-tier tools)
- **Wappalyzer** — detects WAF vendors from HTTP response headers
- **Shodan** — HTTP banners from WAF products contain vendor-specific response headers

**WAF vendor fingerprinting from HTTP headers:**
```bash
$ curl -sk -I https://corp-target.com/ | grep -iE "server|x-powered|x-cache|x-fw|cf-ray|x-guard|via"

Server: cloudflare                          ← Cloudflare WAF
CF-RAY: 12345abc-SIN                        ← Cloudflare confirmed
x-sucuri-id: 12345                          ← Sucuri WAF
x-fw-hash: abc123                           ← Fortiweb (Fortinet WAF)
X-Squid-Error: ERR_NONE                     ← Squid proxy
Server: AkamaiGHost                         ← Akamai
X-CDN: Imperva                              ← Imperva WAF

# Deliberate 403-trigger (send bad request to elicit WAF error page)
$ curl -sk "https://corp-target.com/?id=1' OR '1'='1" | grep -iE "blocked|firewall|incapsula|sucuri|cloudflare"
# WAF error pages are vendor-specific:
# Cloudflare: "Sorry, you have been blocked"
# Sucuri: "Website Firewall - Access Denied"
# ModSecurity: "Not Acceptable" with Mod_Security header
# Imperva: "Request blocked"
```

## 2.2 IDS/IPS fingerprinting through deliberate probe responses

IDS/IPS systems inject TCP RST packets or ICMP Unreachable messages to terminate connections carrying suspicious payloads. This behavior is detectable — if your probes receive RST packets that don't correlate with the target's port state (a port that appears open suddenly sends RST when specific content is detected), an inline IPS is intercepting.

```bash
# Test: send an EICAR string over HTTP (EICAR is the standard AV test string)
$ curl -sk "https://corp-target.com/?test=X5O!P%25%40AP%5B4%5CPZ" -v 2>&1 | grep "< HTTP"
# If connection is reset mid-response → IDS/IPS is blocking

# nmap with --reason to see connection reset source
$ sudo nmap -sS --reason -p 80 203.0.113.45
PORT   STATE SERVICE REASON
80/tcp open  http    syn-ack ttl 64    ← target sent SYN-ACK
# Now try with a payload that triggers IDS:
$ nc 203.0.113.45 80 <<< "GET /../../etc/passwd HTTP/1.0\r\n\r\n"
# If RST arrives before response → IDS dropped it
# If connection completes with 400 → no IDS, just server-side filtering

# Snort RST injection tells you Snort is inline:
# Injected RST has TTL=64 while server responses have TTL=128 (Windows)
# → TTL mismatch between RST and SYN-ACK = RST was injected by IDS, not server
$ hping3 --tcp-timestamp -S -p 80 -d 500 --data "NmapVersionScanProbe" 203.0.113.45
# Compare TTL of SYN-ACK vs any subsequent RST
```

## 2.3 SIEM identification from log forwarding traffic

When you have vantage position on the network (internal or compromised host), SIEM identification is straightforward: SIEMs receive logs from all endpoints and network devices. The log forwarding traffic reveals the SIEM product.

```bash
# Common SIEM log receiver ports — watch for traffic to these
# Splunk Universal Forwarder: 9997/TCP (Splunk indexer port)
# Elastic/Logstash: 5044/TCP (Beats protocol), 5000/TCP
# QRadar: 514/UDP (syslog), 32768-65535 (QRadar-specific)
# ArcSight: 1470/TCP (ArcSight SmartConnector)
# Microsoft Sentinel: outbound HTTPS to ods.opinsights.azure.com
# Graylog: 514/UDP, 5044/TCP

# Network capture: filter for SIEM destination ports
$ tshark -r capture.pcap -Y "tcp.dstport == 9997 or tcp.dstport == 5044 or udp.dstport == 514" \
  -T fields -e ip.dst -e tcp.dstport | sort -u

10.10.0.50    9997    ← Splunk indexer at 10.10.0.50
10.10.0.50    9997    ← multiple hosts forwarding to same IP

# Identify Splunk Universal Forwarder on a compromised Windows host
$ sc query SplunkForwarder    ← Windows service
$ ps aux | grep splunk        ← Linux process
$ ls /opt/splunkforwarder/    ← Linux installation directory

# Splunk web interface (common ports)
$ curl -sk https://10.10.0.50:8000/ | grep -i "splunk"
$ curl -sk https://10.10.0.50:8089/ | grep -i "splunk"   ← management API
```

If Splunk is the SIEM: every command you run on a monitored host ships logs to `10.10.0.50`. Knowing this, you can target the Splunk instance itself (credential attacks, Splunk RCE via search app uploads) or time your operations to occur during periods of log review blindspots.

## 2.4 EDR process and driver identification (post-initial-access)

Once on a system, the EDR is identifiable from process names, services, and kernel driver files. This list covers the major EDR products:

```text
EDR Product               Process Name             Service Name
─────────────────────────────────────────────────────────────────
CrowdStrike Falcon        CSFalconService.exe      CSFalconService
                          falcon-sensor             (Linux)
SentinelOne               SentinelAgent.exe        SentinelAgent
                          SentinelStaticEngine.exe
Carbon Black (VMware)     cb.exe, cbsensor.exe     cbdefense
Cylance                   CylanceSvc.exe           CylanceSvc
Symantec (Broadcom)       Smc.exe, SmcGui.exe      ccSvcHst
Microsoft Defender ATP    MsSense.exe              Sense
                          WdFilter.sys             (kernel driver)
Palo Alto Cortex XDR      cyserver.exe             Cyserver
Elastic EDR               elastic-endpoint          elastic-endpoint
Intercept X (Sophos)      SSPService.exe            SophosSSP
Malwarebytes EDR          MBAMService.exe           MBAMService
```

```bash
# Windows: enumerate running processes for EDR
$ tasklist /svc 2>nul | findstr /i "csfalcon sentinel cylance mssense cb.exe"

# PowerShell: enumerate services matching EDR names
$ Get-Service | Where-Object {$_.Name -match "falcon|sentinel|cylance|csfalcon|mssense|cbdefense"}

# Linux: check running processes
$ ps aux | grep -iE "falcon|sentinel|cb_agent|elastic-endpoint|edr"

# Check for EDR kernel drivers (Windows — highly intrusive if EDR is watching)
$ driverquery | findstr /i "CrowdStrike\|Sentinel\|CbFilter\|WdFilter"
WdFilter              Windows Defender MiniFilter Driver   ← Windows Defender active
CrowdStrike Falcon    CSAgent                               ← CrowdStrike confirmed

# Linux kernel modules
$ lsmod | grep -iE "falcon|sensor|cbd"
```

**Critical rule:** On a host where EDR is running, enumerating the EDR process itself may trigger a detection. Some EDR products alert on `tasklist` or `Get-Process` queries that include their own process names. Use indirect methods where possible.

## 2.5 Inferring SIEM rules and alert thresholds

Knowing which SIEM is deployed lets you estimate what detection rules are active. Most organizations deploy SIEM out-of-the-box with vendor-default rules, with some customization. Default Splunk ES, Sentinel, and QRadar rule sets have known gaps:

```text
Common SIEM detection rules (default Splunk ES):
  ✅ Detected: Mimikatz process name / lsass dump (by process name)
  ✅ Detected: PowerShell encoded commands with -enc flag (by command line)
  ✅ Detected: Lateral movement via net use / SMB logon events
  ✅ Detected: Large outbound data transfers (DLP rules)
  ❌ Missed:   renamed Mimikatz binary (no process name match)
  ❌ Missed:   LOLBAS usage (legitimate Windows tools used offensively)
  ❌ Missed:   slow/staged data exfil below DLP threshold
  ❌ Missed:   C2 over HTTPS to CDN-fronted infrastructure
```

This gap analysis is public knowledge — MITRE D3FEND and vendor documentation describe what their default rules catch. Once you know the SIEM, you know approximately what the gaps are.

## 2.6 Firewall and network security device fingerprinting

Network security devices (firewalls, IPS, proxy) are identifiable from packet-level behavior and HTTP proxy response patterns:

```bash
# Palo Alto firewall — sends specific RST/TCP flags
# Check if deep packet inspection is active:
$ curl -vk "https://corp-target.com/test?cmd=whoami" 2>&1 | grep -E "HTTP/|reset"

# Zscaler proxy identification (cloud proxy)
# All HTTP traffic via Zscaler includes a specific header:
$ curl -sk "http://corp-target.com/" -v 2>&1 | grep -i "x-zscaler\|via: zscaler"
Via: 1.1 zscaler.net (ZScaler/5.0)   ← Zscaler cloud proxy confirmed

# BlueCoat/Symantec proxy
$ curl -sk "http://corp-target.com/" -v 2>&1 | grep -i "via\|x-bluecoat"

# Fortinet FortiGate — specific TCP window size and options in blocked responses
# Checkpoint — "Check Point FireWall-1" in server banner on management ports

# Nmap OS detection on suspected firewall IP
$ sudo nmap -O 203.0.113.254   ← last hop before target
# Palo Alto: specific TCP behavior fingerprint
# Cisco ASA: specific ICMP unreachable response format
```

## 2.7 SOAR platform indicators

SOAR (Security Orchestration, Automation and Response) platforms respond to SIEM alerts by automating actions. Identifying SOAR presence tells you that alerts may trigger automated responses within seconds — IP blocks, account disables, or quarantine actions.

```text
SOAR indicators:
  Palo Alto XSOAR (Demisto): outbound HTTPS to demisto.* domains from security team machines
  Splunk SOAR (Phantom):     running on same host as Splunk, port 8443
  Swimlane:                  swimlane.io domain in internal DNS
  IBM Resilient:             ibm.com/mysupport outbound traffic

Implication: if SOAR is confirmed, a single triggered SIEM alert may result in:
  - Automatic firewall block of your source IP within minutes
  - Account lockout if credential spraying is detected
  - Endpoint isolation (quarantine) of the compromised host
```

## 2.8 Slowing down when defensive stack is identified

The roadmap explicitly states: *"If found, slow down your operation immediately."* This is operationally critical.

```text
Defensive stack confirmed → Adjust operational tempo:

No EDR detected           → Standard pace, standard tooling
Windows Defender only     → Use signed binaries, LOLBAS, in-memory execution
CrowdStrike Falcon        → Memory-only execution, no disk writes, custom C2 profiles
SentinelOne               → Kernel-based behavioral detection → process injection only
Splunk SIEM confirmed     → Avoid high-volume events (port scans, mass auths)
SOAR confirmed            → Single-step operations with long dwell time between steps
IPS inline confirmed      → Encrypt all C2 traffic, avoid known-bad signatures
```

## 2.9 SSL inspection proxy detection

Corporate environments frequently deploy SSL inspection (TLS interception) proxies that perform a man-in-the-middle on all HTTPS traffic — the proxy presents its own certificate to the client and re-encrypts traffic to the destination. Identifying this is critical: it means all encrypted C2 traffic originating from a compromised internal host may be decrypted and inspected by the corporate proxy before it leaves the network.

```bash
# Method 1: Check the TLS certificate issuer on an HTTPS connection
# A legitimate external HTTPS site should present a certificate signed by a public CA
# An SSL inspection proxy presents a certificate signed by the corporate internal CA
$ curl -vk https://www.google.com 2>&1 | grep "issuer"
issuer: C=US; O=Google Trust Services; CN=GTS CA 1C3   ← legitimate public CA

# If instead you see:
issuer: CN=Corp-Target Internal CA; O=Corp-Target LLC   ← corporate internal CA → SSL inspection active

# Method 2: Compare certificate fingerprint
# The certificate presented to you should match what everyone else sees
$ openssl s_client -connect www.google.com:443 2>/dev/null | openssl x509 -noout -fingerprint
# Compare with the known Google cert fingerprint from a trusted source
# Mismatch → SSL inspection proxy in the path

# Method 3: Check for X-Forwarded-For or Via headers from proxy
$ curl -sk https://httpbin.org/headers | python3 -m json.tool | grep -iE "via|forwarded|proxy"
"X-Forwarded-For": "10.10.0.100"    ← internal IP forwarded by proxy
"Via": "1.1 bluecoat.corp.target.com"   ← BlueCoat proxy identified!

# Common SSL inspection product indicators:
# Zscaler:    Certificate issuer CN contains "Zscaler"
# BlueCoat:   Via header contains bluecoat hostname
# Palo Alto:  Certificate issuer CN contains "PAN" or "Palo Alto"
# Cisco WSA:  Via header contains "WSA" or Cisco product name
# Forcepoint: Certificate issuer contains "Forcepoint"
```

If SSL inspection is confirmed, all standard HTTPS C2 channels originating from inside the corporate network are visible to the defensive team. C2 must use:
- Certificate pinning (reject inspection proxy's cert)
- Domain fronting (make traffic appear to go to a trusted CDN domain)
- Non-standard ports and protocols that bypass the proxy's rule scope

## 2.10 Network honeypot and canary token detection

Organizations deploy network honeypots (fake vulnerable services on unused IPs or ports) and canary tokens (sensitive-looking files or credentials that alert when accessed). Interacting with either generates an immediate alert — before you've done anything operationally meaningful.

```bash
# Honeypot indicators:
# 1. Hosts that appear in a broad port scan but have no legitimate business purpose
# 2. Services that respond to every connection attempt with a suspiciously "perfect" banner
# 3. IPs in the organization's range that have no DNS record and no historical Shodan data
# 4. Credentials that appear in a "configuration file" but work on every system

# Cross-referencing against historical Shodan data reveals new IPs:
$ shodan host 203.0.113.250    ← check if this IP has historical activity
Error: No information available for that IP.
# → IP is new with no history → may be a honeypot deployed recently

# Canary token detection (canarytokens.org and commercial variants):
# Canary tokens are embedded in:
# - PDF/Word documents (document open triggers HTTP request)
# - Fake credential files (AWS keys, SSH private keys)
# - Email addresses (login triggers notification)
# - DNS records (DNS query triggers notification)
# - Web bugs (image load triggers HTTP request)

# Safe approach: check Shodan/Censys for an IP BEFORE connecting to it
$ shodan host 10.10.0.250
No information available   ← no history → treat as honeypot candidate

# Check if a "discovered" credential works universally — instant honeypot sign
# If username:password123 works on SSH, RDP, web panel, AND VPN → canary credential
# Real credentials have specific access scope; universal credentials are canaries

# Timing anomalies:
# Honeypot services respond instantly and uniformly to all queries
# Real services have variable response times based on backend load
# If every probe to every service responds in exactly the same millisecond window → suspicious
$ for port in 22 80 443 8080 3389; do
    time nc -zw 1 10.10.0.250 $port 2>&1
  done
# All identical response times → honeypot fingerprint
```

The rule: never interact with a host that has no DNS record, no Shodan history, and no apparent business function before verifying it against the authorized IP list in the engagement scope.

## 2.11 DLP fingerprinting from network behavior

Data Loss Prevention (DLP) products inspect outbound traffic for sensitive content (PII, source code, credit card numbers, classified keywords). They are often the least-understood component of the defensive stack and are identifiable from specific network behaviors:

```bash
# DLP products intercept outbound HTTP/HTTPS:
# Signs of active DLP inspection:
# 1. Outbound HTTPS shows corporate CA cert (SSL inspection — also used by DLP)
# 2. Large file transfers to external sites are blocked or delayed
# 3. Email attachments containing specific patterns are rejected with NDR

# Test for DLP-triggered blocks (do NOT use in production without auth):
# DLP commonly blocks: SSN patterns, credit card numbers, source code keywords
# A safe test: send a benign file with the MIME type that triggers DLP rules

# Symantec DLP (Vontu) — identifies itself in block pages:
$ curl -sk "http://corp-target.com/" --data "file_content=CONFIDENTIAL_SECRET_KEY" \
  -v 2>&1 | grep -i "vontu\|symantec\|DLP"

# ForcePoint DLP — specific HTTP response headers
$ curl -sk "https://corp-target.com/" -v 2>&1 | grep -i "forcepoint\|websense"

# Netskope (cloud DLP) — redirects through Netskope cloud:
$ traceroute corp-target.com | grep -i "netskope"

# Common DLP indicators:
# - Response to data-containing POST: "Content blocked by DLP policy" page
# - Email gateway NDR: "Message rejected by content inspection policy"
# - Upload blocked with error: HTTP 403 + X-DLP-Policy header
# - SSL inspection proxy cert from DLP vendor (ForcePoint, Zscaler, Netskope)

# DLP operational implications:
# → Data exfiltration must avoid DLP-monitored channels:
#   Use encrypted C2 over HTTPS to avoid content inspection
#   Use steganography or encoding to bypass keyword scanning
#   Use DNS exfiltration (base64 in DNS TXT queries) to bypass HTTP DLP
#   Fragment data into small chunks below DLP alerting thresholds

# Identifying DLP threshold from probe responses:
$ for size in 1000 5000 10000 50000 100000; do
    echo "Testing ${size}B payload..."
    dd if=/dev/urandom bs=$size count=1 2>/dev/null | base64 | \
      curl -sk -X POST -d @- "https://httpbin.org/post" | grep -c "data"
  done
# If requests below threshold succeed but large ones are blocked → DLP threshold identified
```

Knowing DLP is active and its approximate threshold is directly actionable: exfiltration payloads are sized and encoded to stay below the DLP detection boundary, data is exfiltrated over channels DLP doesn't inspect (DNS, ICMP, IPv6), and the DLP product's own SSL inspection cert reveals the vendor's identity.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **EDR (Endpoint Detection & Response)** | Security software monitoring endpoint process behavior, memory, and network activity in real time |
| **SIEM (Security Information and Event Management)** | Aggregates logs from all infrastructure sources and applies detection rules to generate alerts |
| **SOAR (Security Orchestration, Automation & Response)** | Automated response platform triggered by SIEM alerts; can quarantine hosts, block IPs, disable accounts |
| **IDS (Intrusion Detection System)** | Passive network monitor that generates alerts on suspicious traffic patterns |
| **IPS (Intrusion Prevention System)** | Inline network device that can actively block or reset suspicious connections |
| **DLP (Data Loss Prevention)** | Software monitoring for unauthorized data transfer; commonly deployed as agent on endpoints |
| **WAF (Web Application Firewall)** | Proxied or agent-based protection filtering HTTP/HTTPS traffic for attack signatures |
| **Universal Forwarder** | Lightweight Splunk agent running on endpoints to ship logs to Splunk indexer |
| **Kernel driver** | OS-level code loaded by EDR for deep system visibility; harder to evade than user-space agents |
| **LOLBAS (Living off the Land Binaries and Scripts)** | Legitimate Windows binaries used offensively to evade EDR tools that whitelist signed binaries |
| **Behavioral detection** | EDR technique that flags suspicious action sequences (process injection, credential dumping) regardless of binary name |
| **Operational tempo** | The pace and volume of attacker activity — reduced to stay below detection thresholds |
| **Deep Packet Inspection (DPI)** | Firewall capability to inspect payload content, not just headers |
| **Log forwarding** | Process by which endpoints ship event logs to the SIEM |

---

# Section 4 — Tools and commands

| Tool / Method | Command | What it finds | When to use |
|---------------|---------|--------------|------------|
| Google dork | `site:linkedin.com "corp-target" "CrowdStrike"` | EDR from job posts | Pre-access |
| curl header | `curl -sk -I https://target/ \| grep -i "server\|cf-ray\|x-fw"` | WAF vendor | Pre-access |
| curl 403-trigger | `curl -sk "https://target/?id=1' OR '1'='1"` | WAF error page | WAF fingerprint |
| tshark | `tshark -r cap.pcap -Y "tcp.dstport==9997"` | Splunk forwarder traffic | Internal capture |
| tasklist | `tasklist /svc \| findstr csfalcon` | CrowdStrike on Windows | Post-access |
| Get-Service | `Get-Service \| Where {$_.Name -match "sentinel\|falcon"}` | EDR services | Post-access |
| ps aux | `ps aux \| grep -i "falcon\|sentinel\|elastic-endpoint"` | EDR on Linux | Post-access |
| lsmod | `lsmod \| grep -i "falcon\|sensor"` | EDR kernel module | Post-access |

---

# Section 5 — Defender detection

- **Job listing and OSINT research:** Entirely passive — no target systems are touched.
- **HTTP header inspection and 403-trigger:** The WAF fingerprinting requests appear in the WAF's own logs. Triggering a 403 via a SQL injection string generates a WAF alert — the defender's WAF dashboard shows your source IP as an attacker. This is a trade-off: you confirm the WAF exists, but you also reveal your IP.
- **IPS probe (EICAR/known-bad strings):** Inline IPS systems log every payload they block. Sending EICAR strings to probe for IPS generates IPS block log entries with your source IP. Do not do this unless you are comfortable with attribution.
- **Post-access EDR enumeration:** `tasklist` and `Get-Service` are standard commands — most EDR products do not alert on them individually. However, querying for the EDR's own process name (e.g., `tasklist | findstr csfalcon`) may trip behavioral rules in advanced EDR products that monitor self-inspection attempts.
- **Log forwarding capture (internal):** Requires vantage on the network. The capture itself is passive — no packets are sent to the SIEM. SIEM operators cannot see that you observed their log forwarding traffic.

---

# Section 6 — Lab task

**Platform:** Kali Linux. Target: TryHackMe or HackTheBox rooms with "Security Tools" or "Blue Team" themes. For WAF testing: use your own web server with ModSecurity installed, or test against `waf.testfire.net`.

**Objective:** Identify the defensive stack of a target from external signals only, without triggering any security alerts.

**Steps:**

1. **Job post OSINT:** Search LinkedIn/Indeed for `"<target company>" "CrowdStrike" OR "SentinelOne" OR "Splunk" OR "QRadar"` — document findings
2. **WAF header inspection:** `curl -sk -I https://target/ | grep -iE "server|x-fw|cf-ray|via|x-cache|x-sucuri"` — identify WAF from headers
3. **WAF error-page fingerprinting:** Send `?id=1' OR '1'='1` and `?../../etc/passwd` — document the error page vendor signature
4. **Shodan WAF check:** `shodan host <target-ip>` — does Shodan show any WAF product in the HTTP banner?
5. **SSL/TLS inspection proxy check:** `curl -vk https://target/ 2>&1 | grep -E "issuer|subject"` — if cert is issued by Zscaler/BlueCoat CA, corporate SSL inspection is active
6. **Simulate internal log forwarding check (lab only):** On a local VM with Splunk UF installed, capture: `tcpdump -i eth0 -n port 9997 -w splunk_traffic.pcap`
7. **EDR process check (on lab VM):** `tasklist /svc` — document which security processes are running
8. **Slow-down assessment:** Based on findings, write a one-paragraph operational tempo recommendation — what pace and tooling changes would this defensive stack require?
9. **Document in `defensive_profile.md`:** WAF vendor | IDS/IPS | EDR | SIEM | SOAR | Operational impact
10. **Compile evasion requirements:** For each identified tool, list one specific evasion technique

```bash
git commit -m "recon(stage4): defensive profiling — security stack identified for <target>"
```

---

# Section 7 — Common mistakes

**1. Skipping defensive profiling and going straight to exploitation**
_Why it matters:_ The most common cause of detection in red team engagements. Running a standard Metasploit payload against a host protected by CrowdStrike Falcon is detected within 3 seconds of execution in default configuration.
_Fix:_ Defensive profiling is mandatory before any payload execution. It takes 20–30 minutes externally and saves the entire engagement.

**2. Sending EICAR strings to probe for IPS**
_Why it matters:_ EICAR payload probes trigger IPS block events with your source IP logged. You confirm IPS exists but burn your IP address in the process.
_Fix:_ Infer IPS from other signals: response timing anomalies, RST injection (TTL mismatch), job listings, and Shodan data. Only probe directly if you can rotate your source IP or are operating under cover.

**3. Assuming Windows Defender = no EDR**
_Why it matters:_ Windows Defender ATP (Microsoft Defender for Endpoint) is a full enterprise EDR product — not just the consumer antivirus. Organizations with E5 licensing have behavioral detection, EDR telemetry, and integration with Microsoft Sentinel.
_Fix:_ Distinguish between Windows Defender Antivirus (consumer, basic) and Microsoft Defender for Endpoint (enterprise EDR). Check for `MsSense.exe` (Sense service) — that's Defender ATP, not just basic AV.

**4. Not cross-referencing WAF vendor with known bypass techniques**
_Why it matters:_ Identifying Cloudflare as the WAF without knowing that Cloudflare WAF bypass techniques exist (header injection, direct origin IP access, h2c smuggling) leaves capability on the table.
_Fix:_ For each identified WAF/IPS product, immediately look up: (1) known bypass methods, (2) whether the origin IP is discoverable, (3) whether h2c or HTTP/2 bypass applies.

**5. Treating defensive profiling as a one-time check**
_Why it matters:_ Defensive stack changes during engagements. An organization may deploy additional tooling in response to detecting early-stage scanning (incident response triggers deployment of endpoint tooling).
_Fix:_ Re-evaluate the defensive stack at each engagement phase. If your scanning triggered an alert, the next time you access the network the defensive posture may be elevated.

---

# Section 8 — Move-on gate

1. A job listing for Corp-Target mentions "experience with CrowdStrike Falcon preferred." HTTP headers from the target show `CF-RAY: abc123`. A curl request with a SQL injection string returns an error page with "Sorry, you have been blocked by Cloudflare." Without notes, describe the complete defensive stack you've identified, state what each component means for your exploitation approach, and describe your adjusted operational tempo.

2. You have a packet capture from a vantage point on the target's internal network. You filter for TCP destination port 9997 and see traffic from 15 different IPs all going to `10.10.0.50`. What SIEM product does this confirm, where is the SIEM server located, and what attack surface does the SIEM server itself represent?

3. Post-exploitation: you run `tasklist /svc` on a compromised host and see `SentinelAgent.exe` running. Without notes, state what this means for: (1) which actions will immediately trigger a detection, (2) what execution method you should use instead, and (3) whether you should continue operating on this host or pivot.
