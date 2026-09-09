# SNMP Enumeration

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

SNMP (Simple Network Management Protocol) is a protocol designed for monitoring and managing network devices — routers, switches, printers, servers, UPS systems, and industrial equipment. It runs on UDP port 161 (agent) and UDP port 162 (trap receiver). When queried with the correct community string, SNMP returns structured management data: the complete network interface table, routing table, running processes, installed software, system description, and connected neighbors.

SNMP is one of the highest-ROI enumeration targets in network reconnaissance. A single `snmpwalk` with the default community string `public` against a misconfigured device can return the complete internal routing table, all connected IP addresses, every active process, and the firmware version — in seconds, without authentication.

```text
Stage 5 — SNMP Enumeration
────────────────────────────────────────────────────────────────────
Port Scan finds 161/UDP  →  [SNMP Enumeration]  →  Network topology
or 162/UDP                   ↑ YOU ARE HERE          credential harvest
or SNMP on TCP (rare)        community string        routing tables
                             brute-force → walk      running processes
────────────────────────────────────────────────────────────────────
Tools: snmpwalk, snmpget, onesixtyone, snmp-check, nmap snmp-* scripts
```

---

# Section 2 — How attackers actually use this

## 2.1 SNMP version overview and security model

SNMP has three major versions with fundamentally different security models:

```text
Version   | Authentication    | Encryption | Risk Level
──────────┼───────────────────┼────────────┼──────────────────────────────
SNMPv1    | Community string  | None       | CRITICAL — cleartext, no auth
SNMPv2c   | Community string  | None       | CRITICAL — cleartext, no auth
SNMPv3    | Username/password | Yes (AES)  | Lower — properly configured
```

SNMPv1 and SNMPv2c use a **community string** as the only authentication mechanism. The community string is transmitted in cleartext on the wire — anyone capturing network traffic reads it directly. The default community strings (`public` for read, `private` for write) are never changed in millions of deployments.

SNMPv3 adds authentication (MD5/SHA) and encryption (DES/AES) but requires proper configuration. Many devices support SNMPv3 but administrators leave SNMPv1/v2c enabled alongside it for backward compatibility — negating all SNMPv3 security.

## 2.2 Community string brute-forcing with onesixtyone

`onesixtyone` is a fast SNMP community string brute-forcer. It sends SNMP queries in parallel across multiple IPs and community strings, making it practical for large-scale discovery.

```bash
# Install
$ sudo apt install onesixtyone snmp snmp-mibs-downloader

# Single host, common community strings
$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt 203.0.113.45

Scanning 1 hosts, 100 communities
203.0.113.45  [public] Hardware: Intel64 Family 6 Model 158, AT/AT COMPATIBLE, Version ...
203.0.113.45  [private] Hardware: Intel64 Family 6 Model 158...   ← write community found!

# Scan entire subnet for SNMP with multiple community strings
$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt \
  -i hosts.txt -w 100 | tee snmp_discovery.txt

# Common community string wordlists:
# /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt   (~33 strings)
# /usr/share/seclists/Discovery/SNMP/snmp.txt                            (~3200 strings)

# If private community string is found (read-write):
# SNMP write access allows setting values — potentially setting routing, restarting services
$ snmpset -v2c -c private 203.0.113.45 sysName.0 s "PWNED"
```

## 2.3 MIB (Management Information Base) structure

SNMP data is organized in a hierarchical tree called the MIB. Each node has an OID (Object Identifier) — a dotted-number path. Knowing which OIDs map to which data is essential for targeted queries.

```text
Key OID paths:
  1.3.6.1.2.1.1.1.0    sysDescr       OS/hardware description
  1.3.6.1.2.1.1.2.0    sysObjectID    Vendor product OID
  1.3.6.1.2.1.1.3.0    sysUptime      System uptime (since last reboot)
  1.3.6.1.2.1.1.4.0    sysContact     Admin contact information
  1.3.6.1.2.1.1.5.0    sysName        System hostname
  1.3.6.1.2.1.1.6.0    sysLocation    Physical location
  1.3.6.1.2.1.2        ifTable        Network interfaces table
  1.3.6.1.2.1.4.20     ipAddrTable    IP addresses and subnet masks
  1.3.6.1.2.1.4.21     ipRouteTable   Routing table (all routes)
  1.3.6.1.2.1.4.22     ipNetToMedia   ARP table (IP → MAC mappings)
  1.3.6.1.2.1.25.4.2   hrSWRunTable   Running processes/services
  1.3.6.1.2.1.25.6.3   hrSWInstalled  Installed software
  1.3.6.1.2.1.6.13     tcpConnTable   Active TCP connections
  1.3.6.1.4.1.77       Enterprises (Microsoft) — Windows-specific OIDs
  1.3.6.1.4.1.9        Cisco OIDs — router/switch specific data
```

## 2.4 snmpwalk for complete MIB dump

`snmpwalk` starts at a given OID and walks the entire subtree, printing every OID and value it finds. Walking from the root (`1.3.6.1.2.1`) dumps the entire management MIB.

```bash
# Enable MIBs for human-readable output
$ sudo apt install snmp-mibs-downloader
$ sudo download-mibs
$ echo "mibs +ALL" >> ~/.snmp/snmp.conf

# Full MIB walk (SNMPv2c)
$ snmpwalk -v2c -c public 203.0.113.45 . | tee snmp_full_walk.txt

# Filtered to interesting sections:
$ snmpwalk -v2c -c public 203.0.113.45 system
SNMPv2-MIB::sysDescr.0 = STRING: Hardware: Intel64 Family 6 Model 158...
    Software: Windows Version 10.0 (Build 17763)   ← Windows Server 2019
SNMPv2-MIB::sysUpTime.0 = Timeticks: (12345678) 142 days, 3:04:38.00  ← uptime
SNMPv2-MIB::sysContact.0 = STRING: admin@corp-target.com              ← admin email
SNMPv2-MIB::sysName.0 = STRING: WEB01                                 ← hostname
SNMPv2-MIB::sysLocation.0 = STRING: Rack 3, Server Room, Building A   ← physical loc

# Network interfaces
$ snmpwalk -v2c -c public 203.0.113.45 1.3.6.1.2.1.2
IF-MIB::ifDescr.1 = STRING: lo
IF-MIB::ifDescr.2 = STRING: eth0
IF-MIB::ifDescr.3 = STRING: eth1
IF-MIB::ifPhysAddress.2 = STRING: aa:bb:cc:dd:ee:ff
IF-MIB::ifPhysAddress.3 = STRING: 11:22:33:44:55:66
IF-MIB::ifOperStatus.2 = INTEGER: up(1)
IF-MIB::ifOperStatus.3 = INTEGER: up(1)

# IP address table
$ snmpwalk -v2c -c public 203.0.113.45 1.3.6.1.2.1.4.20
IP-MIB::ipAdEntAddr.203.0.113.45 = IpAddress: 203.0.113.45
IP-MIB::ipAdEntAddr.10.10.0.45   = IpAddress: 10.10.0.45       ← internal IP!
IP-MIB::ipAdEntNetMask.10.10.0.45 = IpAddress: 255.255.0.0     ← /16 internal network
```

A host with both a public IP and an RFC1918 address visible in the IP address table is a dual-homed host — it bridges the public internet and the internal network. This is a high-value finding: compromising this host gives access to both zones.

## 2.5 Routing table extraction

```bash
# Full routing table via SNMP
$ snmpwalk -v2c -c public 203.0.113.45 1.3.6.1.2.1.4.21

IP-MIB::ipRouteDest.0.0.0.0        = IpAddress: 0.0.0.0         (default route)
IP-MIB::ipRouteNextHop.0.0.0.0     = IpAddress: 203.0.113.1     (default gateway)
IP-MIB::ipRouteDest.10.10.0.0      = IpAddress: 10.10.0.0       (internal network)
IP-MIB::ipRouteNextHop.10.10.0.0   = IpAddress: 10.10.0.1       (internal router)
IP-MIB::ipRouteDest.10.20.0.0      = IpAddress: 10.20.0.0       ← different subnet!
IP-MIB::ipRouteNextHop.10.20.0.0   = IpAddress: 10.20.0.1       ← 10.20.0.0/24 reachable
IP-MIB::ipRouteDest.172.16.5.0     = IpAddress: 172.16.5.0      ← management VLAN
IP-MIB::ipRouteNextHop.172.16.5.0  = IpAddress: 172.16.5.1

# The routing table reveals all network segments reachable from this host:
# 10.10.0.0, 10.20.0.0, 172.16.5.0 — all internal segments
# This is the internal network topology from a single SNMP query
```

## 2.6 Running processes and installed software

```bash
# Running processes (Windows host)
$ snmpwalk -v2c -c public 203.0.113.45 1.3.6.1.2.1.25.4.2

HOST-RESOURCES-MIB::hrSWRunName.1    = STRING: "System Idle Process"
HOST-RESOURCES-MIB::hrSWRunName.4    = STRING: "System"
HOST-RESOURCES-MIB::hrSWRunName.276  = STRING: "svchost.exe"
HOST-RESOURCES-MIB::hrSWRunName.512  = STRING: "CsSvr.exe"           ← CrowdStrike!
HOST-RESOURCES-MIB::hrSWRunName.720  = STRING: "sqlservr.exe"        ← SQL Server running
HOST-RESOURCES-MIB::hrSWRunName.888  = STRING: "tomcat8.exe"         ← Tomcat running
HOST-RESOURCES-MIB::hrSWRunName.1024 = STRING: "jenkins.exe"         ← Jenkins!
HOST-RESOURCES-MIB::hrSWRunName.1200 = STRING: "MsMpEng.exe"         ← Defender

# Installed software
$ snmpwalk -v2c -c public 203.0.113.45 1.3.6.1.2.1.25.6.3

HOST-RESOURCES-MIB::hrSWInstalledName.1 = STRING: "7-Zip 22.01"
HOST-RESOURCES-MIB::hrSWInstalledName.2 = STRING: "Apache Tomcat 7.0.92"
HOST-RESOURCES-MIB::hrSWInstalledName.3 = STRING: "Jenkins 2.346"
HOST-RESOURCES-MIB::hrSWInstalledName.4 = STRING: "Microsoft SQL Server 2019"
HOST-RESOURCES-MIB::hrSWInstalledName.5 = STRING: "Python 3.9.7"
```

Running processes from SNMP confirm what port scanning implied: Tomcat is running, Jenkins is running, SQL Server is running. The process list also reveals `CsSvr.exe` (CrowdStrike Falcon sensor) — feeding directly into defensive profiling (Stage 4 note 02).

## 2.7 snmp-check for formatted, human-readable output

`snmp-check` wraps snmpwalk in human-readable output tables, making large MIB dumps more navigable:

```bash
$ snmp-check 203.0.113.45 -c public -v2c

[*] Try to connect to 203.0.113.45:161 using SNMPv2c and community 'public'

[+] System information:
  Host IP address                : 203.0.113.45
  Hostname                       : WEB01
  Description                    : Hardware: Intel64 ..., Windows Version 10.0 Build 17763
  Contact                        : admin@corp-target.com
  Location                       : Rack 3, Server Room
  Uptime snmp                    : 142 days, 3:04:38.00

[+] Network information:
  IP forwarding enabled          : yes       ← this host is routing packets
  Default TTL                    : 128
  TCP segments received          : 12345
  TCP segments sent              : 23456

[+] Network interfaces:
  Interface                  : eth0
  IP Address                 : 203.0.113.45
  Netmask                    : 255.255.255.0
  IP Address                 : 10.10.0.45      ← dual-homed!
  Netmask                    : 255.255.0.0

[+] Network IP:
  Id         : 10.10.0.10 / 255.255.0.0  → DC01   ← from ARP table!
  Id         : 10.10.0.20 / 255.255.0.0  → FS01

[+] Routing information:
  (complete routing table as above)

[+] Listening UDP ports:
  161      ← SNMP itself
  53       ← DNS

[+] Listening TCP ports:
  22       ← SSH
  80, 443  ← Web
  8080     ← Tomcat
  8443     ← Tomcat HTTPS
  50000    ← Jenkins agent port!

[+] Processes:
  (complete process list)

[+] Software components:
  (complete installed software list)
```

## 2.8 SNMP trap analysis (passive SNMP intelligence)

SNMP traps are unsolicited messages sent by devices to a management station (SNMP manager) when events occur — interface up/down, high CPU, authentication failure. If you can position a listener on port 162, you receive real-time infrastructure events.

```bash
# Listen for SNMP traps on port 162
$ sudo snmptrapd -f -Lo -c /dev/null -C 0.0.0.0

# Or using tcpdump to capture trap traffic
$ sudo tcpdump -i eth0 -w snmp_traps.pcap "udp port 162"

# Analyze captured traps
$ snmptranslate -Td -OS < snmp_traps.pcap

# Common trap types:
# coldStart (0):          device rebooted
# warmStart (1):          device restarted config
# linkDown (2):           network interface went down
# linkUp (3):             network interface came up
# authenticationFailure (4): wrong community string used
# enterpriseSpecific (6): vendor-specific alerts (Cisco, HP, etc.)

# authenticationFailure traps reveal:
# 1. That someone else is probing SNMP with wrong community strings
# 2. The SNMP agent is active and sending traps to a management station
# 3. The IP of the SNMP manager (destination of the trap)
```

## 2.9 SNMPv3 user enumeration and Cisco-specific high-value OIDs

SNMPv3 requires authentication, but the user discovery process has a vulnerability: an SNMP engine responds differently to queries with a valid username versus an invalid username, allowing username enumeration before authentication is cracked.

```bash
# SNMPv3 username enumeration (nmap script)
$ nmap -sU -p 161 --script snmp-brute \
  --script-args snmp-brute.communitiesdb=/usr/share/seclists/Discovery/SNMP/snmp.txt \
  203.0.113.45

# Try known SNMPv3 usernames (common defaults)
$ for user in admin administrator cisco Monitor netman operator readonly; do
    result=$(snmpwalk -v3 -u $user -l noAuthNoPriv 203.0.113.45 sysDescr.0 2>&1)
    if echo "$result" | grep -q "Unknown user"; then
        echo "[$user] INVALID"
    else
        echo "[$user] VALID USERNAME — try auth cracking"
    fi
  done

# Cisco-specific high-value OIDs (Cisco devices respond to SNMP with rich data)
$ CISCO="203.0.113.254"   # suspected Cisco router
$ COMMUNITY="public"

# Cisco version string (IOS version + hardware)
$ snmpget -v2c -c $COMMUNITY $CISCO 1.3.6.1.4.1.9.2.1.3.0
Cisco IOS Software, Version 15.7(3)M4, RELEASE SOFTWARE
ROM: System Bootstrap, Version 12.4(13r)T, RELEASE SOFTWARE

# Cisco running configuration (if SNMP write access available — CVE-level finding)
$ snmpwalk -v2c -c $COMMUNITY $CISCO 1.3.6.1.4.1.9.2.1.55
# Returns chunks of the running config — contains all passwords, ACLs, routing

# Cisco CDP (Cisco Discovery Protocol) neighbor table
$ snmpwalk -v2c -c $COMMUNITY $CISCO 1.3.6.1.4.1.9.9.23.1.2.1.1.6
# Returns: neighboring device hostnames (maps entire Cisco network topology!)

# Cisco interface descriptions (admins often put info in descriptions)
$ snmpwalk -v2c -c $COMMUNITY $CISCO 1.3.6.1.2.1.31.1.1.1.18
# Returns: interface descriptions configured by admins
# e.g., "Link to DC01", "Connection to PCI zone", "Management VLAN"

# Cisco IP SLA (network performance monitoring data)
$ snmpwalk -v2c -c $COMMUNITY $CISCO 1.3.6.1.4.1.9.9.42.1.2.10
# Returns: round-trip times and jitter to monitored destinations
# → Reveals all IPs the network team monitors = all important internal hosts
```

Cisco CDP neighbor data from SNMP is particularly powerful: a single query to a Cisco router returns the hostname, IP address, platform type, and IOS version of every directly connected Cisco device. Recursively querying each neighbor builds a complete Cisco network topology map without ever using Cisco management protocols directly.

```text
SNMP enumeration output → intelligence conversion table:

  sysDescr        →  OS version, patch level, product name
  ifDescr         →  Network topology (interface names reveal connected networks)
  hrSWInstalledName → Full software inventory (feed into Version-to-CVE note)
  hrNetworkAddress  → Additional IP addresses on multi-homed hosts
  hrProcessorLoad   → CPU usage patterns (reveals batch job schedules)
  hrStorageUsed     → Disk usage (large /tmp or unusual mounts are interesting)
  ciscoCdpCache     → Full Cisco neighbor map
  Cisco config OID  → Running configuration (if write community is "private")
  ipCidrRouteTable  → Routing table (reveals all internal subnets the device routes)
  tcpConnTable      → Active TCP connections at time of query
  udpTable          → Active UDP listeners
```

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **SNMP** | Simple Network Management Protocol — protocol for monitoring and managing network devices |
| **Community string** | SNMP v1/v2c authentication mechanism — essentially a cleartext password |
| **OID (Object Identifier)** | Dotted-number path identifying a specific value in the MIB tree |
| **MIB (Management Information Base)** | Hierarchical database of manageable objects for a device |
| **snmpwalk** | Command that retrieves all values in an OID subtree by iterative GETNEXT requests |
| **snmpget** | Command that retrieves a specific OID value |
| **onesixtyone** | Fast SNMP community string brute-forcer |
| **snmp-check** | SNMP enumeration tool that formats output into readable tables |
| **SNMP trap** | Unsolicited alert message sent by a device to the SNMP manager when an event occurs |
| **hrSWRunTable** | SNMP MIB table listing all currently running processes/services |
| **hrSWInstalled** | SNMP MIB table listing all installed software packages |
| **ipRouteTable** | SNMP MIB table containing the device's full routing table |
| **ipAddrTable** | SNMP MIB table listing all IP addresses configured on the device |
| **ipNetToMedia** | SNMP MIB ARP cache table — maps IP addresses to MAC addresses |
| **Dual-homed host** | A host with interfaces on two or more networks — bridges security zones |
| **SNMPv3** | SNMP version with authentication and encryption — `authPriv` mode is secure |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `onesixtyone` | `onesixtyone -c strings.txt 203.0.113.45` | Community strings | Discovery sweep |
| `snmpwalk` | `snmpwalk -v2c -c public target .` | Full MIB dump | After string found |
| `snmpwalk` | `snmpwalk -v2c -c public target 1.3.6.1.2.1.4.21` | Routing table | Topology mapping |
| `snmpwalk` | `snmpwalk -v2c -c public target 1.3.6.1.2.1.25.4.2` | Running processes | Post-discovery |
| `snmp-check` | `snmp-check target -c public` | Formatted full dump | Quick assessment |
| `nmap` | `nmap -sU -p 161 --script snmp-info target` | SNMP detection | Initial scan |

**Complete SNMP enumeration pipeline:**
```bash
TARGET="203.0.113.45"

# Step 1: Confirm SNMP is open (UDP 161)
$ sudo nmap -sU -p 161 $TARGET --open
161/udp open  snmp

# Step 2: Community string brute-force
$ onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt \
  $TARGET | tee snmp_strings.txt

# Step 3: Full walk with found community string
$ COMMUNITY=$(grep "$TARGET" snmp_strings.txt | awk '{print $2}' | tr -d '[]')
$ snmpwalk -v2c -c $COMMUNITY $TARGET . > snmp_full.txt 2>&1

# Step 4: Extract key sections
$ grep "sysDescr\|sysName\|sysContact\|sysLocation\|sysUpTime" snmp_full.txt
$ grep "ipAdEntAddr" snmp_full.txt | grep -v "127.0.0.1"
$ grep "ipRouteDest\|ipRouteNextHop" snmp_full.txt
$ grep "hrSWRunName" snmp_full.txt | grep -v "STRING: " | head -30

# Step 5: Human-readable summary
$ snmp-check $TARGET -c $COMMUNITY -v2c > snmp_report.txt
```

---

# Section 5 — Defender detection

- **UDP port 161 is logged:** Every SNMP query arrives on UDP port 161. Network-aware firewalls and SNMP agents log incoming queries with source IP. A burst of onesixtyone community string brute-force attempts (33–3200 packets in seconds) appears in SNMP agent logs as repeated authentication failures.
- **SNMPv2c community string attempts send authenticationFailure traps:** When the wrong community string is used, SNMPv2c agents send an `authenticationFailure` SNMP trap to the configured management station. This directly alerts the network management team that SNMP probing is occurring.
- **snmpwalk generates many GETNEXT requests:** A full MIB walk generates hundreds to thousands of GETNEXT packets against the target's SNMP agent. On a monitored network, this volume from a single source IP is identifiable as an SNMP enumeration scan.
- **SNMPv3 auth failures are even more visible:** SNMPv3 devices log authentication failures with the attempted username — directly revealing your probe attempt.
- **Mitigation for operators:** (1) Start with just the two default strings (`public`, `private`) before running the full wordlist — if defaults work, the rest is unnecessary noise. (2) Use `-w` (wait) in onesixtyone to slow the rate. (3) Query specific OIDs rather than snmpwalk for a more surgical approach.

---

# Section 6 — Lab task

**Platform:** Kali Linux. TryHackMe "Network Services" room (includes SNMP section) or Metasploitable2 which runs SNMP with the `public` community string.

**Objective:** Enumerate a target via SNMP — extract the routing table, process list, installed software, and network topology.

**Steps:**

1. **Discover SNMP:** `sudo nmap -sU -p 161 <target> --open` — confirm UDP 161 is open
2. **Community string brute-force:** `onesixtyone -c /usr/share/seclists/Discovery/SNMP/common-snmp-community-strings.txt <target>`
3. **System info:** `snmpwalk -v2c -c public <target> system`
4. **IP address table:** `snmpwalk -v2c -c public <target> 1.3.6.1.2.1.4.20` — note all IPs (look for RFC1918)
5. **Routing table:** `snmpwalk -v2c -c public <target> 1.3.6.1.2.1.4.21` — document all routes
6. **Running processes:** `snmpwalk -v2c -c public <target> 1.3.6.1.2.1.25.4.2` — grep for interesting services
7. **Installed software:** `snmpwalk -v2c -c public <target> 1.3.6.1.2.1.25.6.3`
8. **ARP table:** `snmpwalk -v2c -c public <target> 1.3.6.1.2.1.4.22` — map internal IPs to MACs
9. **snmp-check summary:** `snmp-check <target> -c public -v2c | tee snmp_report.txt`
10. **Compile `snmp_intel.md`:** System description | All IPs | Routes | Interesting processes | Installed software | ARP table | Security tools detected

```bash
git commit -m "recon(stage5): SNMP enumeration — routing table and process list for <target>"
```

---

# Section 7 — Common mistakes

**1. Only checking TCP 161 — SNMP is UDP**
_Why it matters:_ `nmap -sT -p 161 target` performs a TCP connect scan. SNMP runs on UDP 161. A TCP scan reports port 161 closed even when SNMP is fully functional. This is one of the most frequent SNMP enumeration failures.
_Fix:_ Always scan with `nmap -sU -p 161 target` for SNMP discovery. The `-sU` flag enables UDP scanning.

**2. Not checking SNMPv1 when SNMPv2c fails**
_Why it matters:_ Older devices only support SNMPv1. If onesixtyone with `-v2c` gets no results but the port is open, the device may only accept v1 queries.
_Fix:_ Try both: `snmpwalk -v1 -c public target .` if v2c produces no results.

**3. Stopping after the system OID — not reading the routing table**
_Why it matters:_ `sysDescr` and `sysName` are the obvious targets. The routing table (OID 1.3.6.1.2.1.4.21) is where the real topology intelligence lives. It reveals every network segment the device knows how to reach, including internal subnets invisible from external scanning.
_Fix:_ Always walk the full ipRouteTable. The routing table from a single border device may reveal the entire internal network topology.

**4. Forgetting that the `private` community string allows writes**
_Why it matters:_ `private` is the default SNMP write community string. Write access allows setting MIB values — including sysName, interface admin status, and on some devices, installing software or changing routing configuration.
_Fix:_ When `private` is confirmed, document write access as a critical finding separately from read access. Do not exercise write capabilities without explicit authorization.

**5. Not cross-referencing SNMP process list with defensive profiling**
_Why it matters:_ The `hrSWRunTable` (running processes) from SNMP is the same data as `tasklist` from a compromised host — without needing shell access. If `CsSvr.exe` (CrowdStrike) appears in the process list, defensive profiling is complete before gaining any foothold.
_Fix:_ After snmpwalk, grep the process list specifically for EDR product process names.

---

# Section 8 — Move-on gate

1. You run `nmap -p 161 <target>` and see the port as closed. You then run `nmap -sU -p 161 <target>` and see it open. Without notes, explain why the first scan missed SNMP, what the `-sU` flag does differently, and why SNMP specifically requires this flag.

2. `snmpwalk` of the IP address table returns two entries: `203.0.113.45` (public IP) and `10.10.0.45` (RFC1918). The routing table shows routes to `10.10.0.0/16`, `10.20.0.0/24`, and `172.16.5.0/24`. Without notes, describe what this tells you about the host's network position, what it means for post-exploitation, and why this is a higher-priority target than a single-homed host.

3. `onesixtyone` returns `[private]` community string against a target. Without notes, state what write access means operationally (beyond reading data), name two actions you would NOT perform with write access during a reconnaissance engagement, and explain why the distinction between read and write community strings is a critical finding in a penetration test report.
