# Live Host Discovery

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

Live host discovery is the first active step in network reconnaissance: determining which IP addresses in a target range have at least one reachable service responding to probes. Before you can scan ports or enumerate services, you need to know which IPs are worth scanning. Probing every port on every IP in a /16 is 4.3 billion port-IP combinations — physically impractical without first thinning the target list to live hosts.

The output of live host discovery is a host-alive list: a file of IP addresses confirmed to be reachable. Everything downstream — port scanning, service enumeration, OS fingerprinting — works from this list.

```text
Attack Chain (Stage 3)
──────────────────────────────────────────────────────────
Stage 2 (passive)        Stage 3 (active)
  Shodan IP list    →  [Live Host Discovery]  →  Port Scanning
  DNS enum results        ↑ YOU ARE HERE           Service Enum
  ASN IP ranges                                     Exploitation
──────────────────────────────────────────────────────────
Tools: ping, arp, hping3, fping, nmap -sn, masscan
```

The technique is **not** simple — different network environments require different probe types. A target behind a firewall that blocks ICMP entirely will appear dead to a ping sweep but is alive and reachable on TCP 443. Relying on ICMP alone means missing every host behind a default-drop ICMP policy, which is most enterprise perimeters. Effective live host discovery uses multiple probe types in sequence.

---

# Section 2 — How attackers actually use this

## 2.1 Why ICMP ping sweeps are insufficient alone

The classic ping sweep — sending ICMP Echo Request to every IP in a range — is the most commonly used but least reliable live host detection method against modern corporate targets. Enterprise firewalls default to dropping ICMP Echo at the perimeter. Windows Firewall (enabled by default since Vista) blocks ICMP Echo from external sources. Cloud providers (AWS, Azure, GCP) drop ICMP at the security group level unless explicitly permitted.

What this means operationally: a ping sweep of a corporate /24 that returns 2 live hosts is not telling you that only 2 IPs are live — it's telling you that only 2 hosts have ICMP Echo permitted. The rest may be entirely live and running multiple services.

```text
Corporate perimeter firewall behavior:
  ICMP Echo → BLOCKED (host appears dead to ping sweep)
  TCP 443   → ALLOWED (host responds to HTTPS probes)
  TCP 80    → ALLOWED
  TCP 22    → BLOCKED (appears dead to SSH probe)

Result: ping sweep shows 0 hosts alive, TCP probe shows 3 hosts alive
```

## 2.2 ARP discovery — local segment only

ARP (Address Resolution Protocol) operates at Layer 2 and cannot be blocked by firewalls — it doesn't go through the router. On a local network segment, ARP is the most reliable host discovery method: if a host has the IP assigned and the network interface is up, it will respond to an ARP request. No firewall can block ARP at Layer 2.

The limitation is hard: ARP only works on the local segment. You cannot ARP-sweep a remote network. It is exclusively useful when:
- You are already on the target's internal network (post-initial-access)
- You are performing internal network mapping during a physical or VPN-access engagement

```bash
# ARP sweep using nmap (works on local segment)
$ sudo nmap -sn -PR 192.168.1.0/24
# -PR = ARP ping (default for local segment in nmap)
# Returns all hosts that responded to ARP request

# Manual ARP sweep using arping
$ sudo arping -c 1 -I eth0 192.168.1.1
ARPING 192.168.1.1 from 192.168.1.100 eth0
Unicast reply from 192.168.1.1 [AA:BB:CC:DD:EE:FF]  1.234ms
Sent 1 probes (1 broadcast(s))
Received 1 response(s)   ← host is alive
```

## 2.3 TCP and UDP probe-based discovery

For external networks where ICMP is blocked, TCP probes to commonly open ports are the most reliable alive indicator. The nmap `--PS` (TCP SYN), `--PA` (TCP ACK), and `--PU` (UDP) ping options all work without ICMP.

```text
Probe type selection logic:
  ┌───────────────────────────────────────────────────────┐
  │ External network, ICMP unknown    → -PS80,443,22,8080 │
  │ Windows host suspected            → -PS80,443,135,445 │
  │ External web server confirmed     → -PS80,443          │
  │ Internal network                  → -PR (ARP)          │
  │ UDP services suspected (DNS/SNMP) → -PU53,161          │
  └───────────────────────────────────────────────────────┘
```

TCP SYN probe (`--PS<ports>`): sends a SYN to the specified port. If the host responds with SYN-ACK or RST, the host is alive (RST means closed port but host is there). Nmap immediately sends RST to abort the connection.

TCP ACK probe (`--PA<ports>`): sends ACK with no prior connection. The target's TCP stack, seeing an unexpected ACK, returns RST — confirming the host is alive. Useful when firewalls block incoming SYN but allow ACK (stateful firewall configured to allow established connections inbound).

## 2.4 hping3 for custom probe crafting

hping3 gives precise control over every IP/TCP/UDP/ICMP field, making it the tool of choice when standard nmap probes are being blocked or when you need to craft specific probe types for testing firewall behavior.

```bash
# ICMP ping (equivalent to ping, but with more control)
$ hping3 -1 203.0.113.45
HPING 203.0.113.45 (eth0 203.0.113.45): icmp mode set, 28 headers + 0 data bytes
len=46 ip=203.0.113.45 ttl=64 id=12345 icmp_seq=0 rtt=2.3 ms    ← alive
DPT/FLAGS timeout  ← no response (host may be dead or ICMP blocked)

# TCP SYN probe to port 80 (alive check without completing handshake)
$ hping3 -S -p 80 203.0.113.45
HPING 203.0.113.45: S set, 40 headers + 0 data bytes
len=46 ip=203.0.113.45 ttl=64 DF id=0 sport=80 flags=SA seq=0 rtt=5.2 ms
# flags=SA = SYN-ACK → port 80 open AND host is alive

# TCP SYN to a closed port (still confirms host alive)
$ hping3 -S -p 9999 203.0.113.45
flags=RA   ← RST-ACK = port closed, but HOST IS ALIVE

# UDP probe (to check for ICMP Port Unreachable response = host alive)
$ hping3 --udp -p 33434 203.0.113.45

# Traceroute mode (increments TTL to map hops)
$ hping3 --traceroute -V -1 203.0.113.45
```

hping3 is particularly useful when you know a specific port is open (from Shodan passive recon) and want to confirm the host is still live using that specific port — because the firewall may block all other probe types.

## 2.5 fping for batch ICMP sweeps

fping sends ICMP Echo requests to multiple targets simultaneously, making it significantly faster than sequential ping for large ranges. It handles a list of IPs or a CIDR range and reports which ones responded.

```bash
# Sweep a /24 range
$ fping -a -g 203.0.113.0/24 2>/dev/null
203.0.113.1     ← alive
203.0.113.5     ← alive
203.0.113.45    ← alive
203.0.113.100   ← alive

# With timeout control (faster, fewer false negatives)
$ fping -a -g 203.0.113.0/24 -t 200 -r 1 2>/dev/null
# -t 200 = 200ms timeout per host
# -r 1 = 1 retry (default is 3, reducing to 1 speeds up dramatically)

# Count alive hosts
$ fping -a -g 203.0.113.0/24 2>/dev/null | wc -l
8   ← 8 hosts alive in the /24 (via ICMP)

# Sweep from IP list
$ fping -a -f live_candidates.txt 2>/dev/null > confirmed_alive.txt
```

## 2.6 nmap -sn for comprehensive host discovery

nmap's `-sn` flag (formerly `-sP`) performs host discovery without port scanning. By default, against external targets it uses ICMP Echo + ICMP Timestamp + TCP SYN to port 443 + TCP ACK to port 80. This multi-probe approach catches far more live hosts than ICMP alone.

```bash
# External network host discovery (multi-probe default)
$ sudo nmap -sn 203.0.113.0/24

Starting Nmap — Host discovery
Nmap scan report for 203.0.113.1     — Host is up (0.003s latency)
Nmap scan report for 203.0.113.45    — Host is up (0.011s latency)
Nmap scan report for 203.0.113.100   — Host is up (0.007s latency)
8 hosts up in the /24

# Custom probes: add TCP SYN to ports 80, 443, 8080, 22
$ sudo nmap -sn --PS80,443,8080,22 --PA80 --PE 203.0.113.0/24

# From a list of potential IPs (from Shodan/ASN data)
$ sudo nmap -sn -iL candidate_ips.txt -oG alive_hosts.gnmap
$ grep "Up" alive_hosts.gnmap | awk '{print $2}' > confirmed_alive.txt
```

## 2.7 Firewall-aware probe selection strategy

The right probe type depends on what the target firewall allows. The strategy for an unknown external target:

```text
Step 1: ICMP Echo sweep (fping or nmap -sn --PE)
         → Some hosts respond. Others may be alive but blocking ICMP.

Step 2: TCP SYN to common ports (--PS80,443,22,25,8080)
         → Catches hosts with ICMP blocked but web/SSH open.

Step 3: TCP ACK to port 80 (--PA80)
         → Catches hosts where inbound SYN is blocked but ACK is allowed
           (misconfigured stateful firewall or IDS bypass).

Step 4: UDP to DNS/SNMP (--PU53,161)
         → Catches DNS servers and SNMP devices undetectable by TCP.

Step 5: Merge all results → deduplicated alive host list
```

```bash
# Full multi-probe host discovery sweep
$ sudo nmap -sn \
    --PE \                  # ICMP Echo
    --PP \                  # ICMP Timestamp
    --PS80,443,22,8080 \    # TCP SYN
    --PA80 \                # TCP ACK
    --PU53,161 \            # UDP
    203.0.113.0/24 \
    -oG alive_full.gnmap

$ grep "Up" alive_full.gnmap | awk '{print $2}' | sort -u > alive_final.txt
```

## 2.8 masscan for large-scale IP range sweeping

For large IP ranges (/16 or bigger), neither fping nor nmap are practical. masscan operates asynchronously at millions of packets per second, discovering live hosts by detecting open ports at scale.

```bash
# Sweep entire ASN IP range for hosts with port 80 or 443 open
$ sudo masscan 203.0.0.0/16 -p80,443 --rate=10000 -oG masscan_alive.txt

# Parse alive hosts from masscan output
$ grep "open" masscan_alive.txt | awk '{print $4}' | sort -u > masscan_hosts.txt

# Rate-limited sweep (stealthier)
$ sudo masscan 203.0.0.0/16 -p80,443,22,8080 --rate=500 -oG masscan_alive.txt
```

## 2.9 Output normalization and deduplication

Different tools format alive host lists differently. Before feeding the list into nmap for port scanning, normalize it:

```bash
# Merge outputs from nmap, fping, masscan
$ cat confirmed_alive.txt masscan_hosts.txt fping_alive.txt | sort -u > master_alive.txt

# Verify file
$ wc -l master_alive.txt
23   ← 23 confirmed live hosts across all probe methods

# Preview
$ head -5 master_alive.txt
203.0.113.1
203.0.113.5
203.0.113.45
203.0.113.100
203.0.113.120
```

## 2.10 arp-scan for precise local segment mapping

`arp-scan` sends ARP requests directly from the network interface and captures ARP replies, returning the IP address, MAC address, and vendor identification (from the MAC OUI prefix) for every responding host. On a local segment, it is faster and more accurate than nmap's ARP probe because it operates entirely at Layer 2.

```bash
# Install
$ sudo apt install arp-scan

# Scan your local subnet (auto-detects interface and range)
$ sudo arp-scan --localnet

Interface: eth0, type: EN10MB, MAC: aa:bb:cc:dd:ee:ff, IPv4: 192.168.1.100
Starting arp-scan 1.10.0 with 256 hosts

192.168.1.1     00:1a:2b:3c:4d:5e    Cisco Systems, Inc         ← router
192.168.1.5     00:50:56:a1:b2:c3    VMware, Inc                ← VM (vmware guest)
192.168.1.10    08:00:27:d4:e5:f6    PCS Systemtechnik GmbH     ← VirtualBox VM
192.168.1.45    b8:27:eb:12:34:56    Raspberry Pi Foundation    ← Raspberry Pi
192.168.1.100   aa:bb:cc:dd:ee:ff    (Your own MAC)

6 packets received by filter, 0 packets dropped by kernel
Ending arp-scan 1.10.0: 256 hosts scanned in 1.458 seconds (175.58 hosts/sec)

# Scan a specific range (not your direct subnet)
$ sudo arp-scan --interface=eth0 192.168.1.0/24

# Output MAC vendor analysis
# VMware MAC OUI (00:50:56, 00:0C:29) → host is a VMware virtual machine
# VirtualBox MAC OUI (08:00:27)        → VirtualBox VM
# Cisco MAC OUI                        → network device (router/switch)
# Dell, HP, Lenovo, Apple              → physical workstations
```

The MAC vendor data is operationally valuable:
- A host with a VMware OUI is a virtual machine — this changes exploitation and persistence approaches
- Cisco/Juniper/Arista MACs are network devices — high-value targets for network takeover but different attack tools
- Apple MACs are likely developer workstations — high-privilege users with SSH keys and dev credentials
- `Raspberry Pi Foundation` MACs may be IoT or physical security devices (cameras, Kali-on-Pi pentest boxes)

```bash
# On post-exploitation (internal network access): map entire segment including VLANs
$ sudo arp-scan --interface=eth1 10.10.0.0/24     ← scan internal VLAN via compromised interface
$ sudo arp-scan --interface=eth1 10.20.0.0/24     ← different VLAN
$ sudo arp-scan --interface=eth1 172.16.0.0/16    ← large internal range
```

## 2.11 Passive prefiltering with Shodan before active probing

Before sending a single active packet to the target, Shodan already knows which IPs in the target's ASN have open ports. Using Shodan as a prefilter dramatically reduces the number of IPs you need to actively probe — you send active probes only to IPs that Shodan confirms have recently had open ports.

```bash
# Query Shodan for the target org's ASN to get pre-verified live IPs
$ shodan search --fields ip_str "org:\"Corp-Target LLC\"" 2>/dev/null | head -30
203.0.113.1
203.0.113.5
203.0.113.45
203.0.113.47
203.0.113.50
# → 5 IPs with confirmed open ports from Shodan

# Query by IP range (CIDR)
$ shodan search --fields ip_str "net:203.0.113.0/24" | sort -u > shodan_alive.txt

# Query by specific port (pre-identify HTTPS-serving hosts)
$ shodan search --fields ip_str "net:203.0.113.0/24 port:443" > shodan_https.txt

# Add Shodan results to the alive host list
$ cat shodan_alive.txt >> master_alive.txt
$ sort -u master_alive.txt > master_alive_dedup.txt
```

The workflow: Shodan prefilter → active confirmation. The Shodan list is your starting point; active host discovery (nmap -sn, fping) confirms current status and catches hosts Shodan missed (recently deployed or not yet indexed).

```bash
# Recommended workflow combining passive prefilter + active confirmation
$ echo "Step 1: Shodan prefilter"
$ shodan search --fields ip_str "net:203.0.113.0/24" > shodan_candidates.txt
$ echo "$(wc -l < shodan_candidates.txt) Shodan candidates"

$ echo "Step 2: Active confirmation of Shodan candidates"
$ nmap -sn --PE --PS80,443,22 -iL shodan_candidates.txt -oG confirmed_shodan.gnmap
$ grep "Up" confirmed_shodan.gnmap | awk '{print $2}' > confirmed_from_shodan.txt

$ echo "Step 3: Sweep full range to catch Shodan gaps"
$ sudo nmap -sn --PS80,443 --PE 203.0.113.0/24 -oG full_sweep.gnmap
$ grep "Up" full_sweep.gnmap | awk '{print $2}' > full_sweep_alive.txt

$ echo "Step 4: Merge all sources"
$ sort -u confirmed_from_shodan.txt full_sweep_alive.txt > master_alive.txt
$ echo "Final: $(wc -l < master_alive.txt) live hosts"
```

This hybrid approach is more efficient than either passive-only or active-only: Shodan reduces the active scan surface, and the active sweep catches what Shodan missed. For a /16 network, Shodan may return 200 known-live hosts from 65,536 possible — the active sweep of the full /16 is still needed but you already have a confirmed-alive starter list.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **ICMP Echo Request / Reply** | The packet exchange behind `ping` — OS and Layer 3 aware, frequently blocked by firewalls |
| **ARP (Address Resolution Protocol)** | Layer 2 protocol mapping IP addresses to MAC addresses; cannot be blocked by IP-layer firewalls; local segment only |
| **Host discovery** | Determining which IPs in a range are live (reachable and responding to at least one probe type) |
| **TCP SYN probe** | Sending a SYN to a port; any response (SYN-ACK or RST) confirms the host is alive |
| **TCP ACK probe** | Sending an ACK to a port; RST response confirms the host is alive and the port is reachable; used when SYN is blocked |
| **UDP probe** | Sending a UDP packet to a port; ICMP Port Unreachable in response confirms the host is alive |
| **fping** | Parallel ICMP Echo tool for fast alive sweeps of large ranges |
| **hping3** | Low-level packet crafter for custom TCP/UDP/ICMP probes with full control over headers |
| **masscan** | Asynchronous packet scanner capable of scanning millions of IPs/ports per minute |
| **nmap -sn** | Nmap host discovery mode (no port scan); uses multiple probe types by default |
| **Alive host list** | The output of host discovery — a deduplicated list of IP addresses confirmed reachable |
| **RFC 1918** | Private IP address ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) — not routable on the public internet |
| **TTL (Time to Live)** | IP field decremented at each hop; reaching 0 causes ICMP Time Exceeded response — useful for traceroute and OS inference |
| **ICMP Type 3 (Port Unreachable)** | ICMP response to a UDP probe for a closed port — confirms the host is alive even when UDP port is closed |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `ping` | `ping -c 3 203.0.113.45` | ICMP Echo response | Quick single-host alive check |
| `fping` | `fping -a -g 203.0.113.0/24 2>/dev/null` | Batch ICMP sweep | Fast alive check across /24 |
| `nmap -sn` | `nmap -sn 203.0.113.0/24` | Multi-probe host discovery | Reliable external network sweep |
| `nmap -sn` | `nmap -sn --PS80,443 -PE 203.0.113.0/24` | Custom probe combo | ICMP-blocked networks |
| `nmap -PR` | `nmap -sn -PR 192.168.1.0/24` | ARP discovery | Local segment only |
| `hping3` | `hping3 -S -p 80 203.0.113.45` | TCP SYN alive check on port 80 | Firewall-specific testing |
| `arping` | `arping -c 1 -I eth0 192.168.1.1` | ARP single-host | Local segment alive check |
| `masscan` | `masscan 203.0.0.0/16 -p80,443 --rate=5000` | Large-range port-based alive check | /16 and larger sweeps |

**External /24 sweep — multi-probe (recommended default):**
```bash
$ sudo nmap -sn --PE --PS80,443,22,8080 --PA80 203.0.113.0/24 -oG host_sweep.gnmap
$ grep "Up" host_sweep.gnmap | awk '{print $2}' > alive.txt
$ cat alive.txt
203.0.113.1
203.0.113.5
203.0.113.45     ← primary target IP from earlier Shodan recon confirmed alive
203.0.113.100
```

**hping3 — testing ICMP vs TCP probe response:**
```bash
# Test 1: ICMP (may be blocked)
$ hping3 -1 -c 2 203.0.113.45
ICMP timeout   ← ICMP is blocked

# Test 2: TCP SYN to port 443 (HTTPS likely open)
$ hping3 -S -p 443 -c 2 203.0.113.45
flags=SA   ← SYN-ACK: port 443 open, host IS alive despite ICMP block

# Conclusion: host is alive, ICMP is filtered, TCP 443 is the probe to use
```

**fping range sweep with timing:**
```bash
$ fping -a -g 203.0.113.0/24 -t 200 -r 1 2>/dev/null | tee alive_icmp.txt
203.0.113.1
203.0.113.45
203.0.113.100

# Cross-reference: run TCP probe sweep to catch ICMP-blocked hosts
$ sudo nmap -sn --PS80,443 -n 203.0.113.0/24 2>/dev/null \
  | grep "Nmap scan report" | awk '{print $5}' > alive_tcp.txt

# Merge and deduplicate
$ sort -u alive_icmp.txt alive_tcp.txt > alive_final.txt
```

**masscan large-range with rate limiting:**
```bash
# Discover hosts in a /16 with rate limiting for stealth
$ sudo masscan 203.0.0.0/16 -p80,443,22 --rate=200 --randomize-hosts \
  -oG masscan_results.txt

# Parse IPs from masscan output
$ grep "open" masscan_results.txt | awk '{print $4}' | sort -u > masscan_hosts.txt
$ wc -l masscan_hosts.txt
47   ← 47 hosts with at least one of ports 80/443/22 open across the /16
```

---

# Section 5 — Defender detection

Host discovery is fully visible on the target's network. Every probe generates traffic on their wire.

- **ICMP:** Perimeter firewalls log (and often drop) ICMP Echo Requests. A sweep of a /24 generating 254 ICMP Echo Requests in rapid succession will appear in firewall logs with the source IP. Rate-limited sweeps (spreading pings over minutes) reduce the signature but don't eliminate it.
- **TCP SYN probes:** A burst of TCP SYN packets to port 80 or 443 from a single source IP across multiple destination IPs in the same /24 is a classic port-scan/host-sweep signature. Snort and Suricata have default rules for this pattern: `alert tcp any any -> any 80 (detection_filter: track by_src, count 20, seconds 10;)` — 20 connections in 10 seconds from a single source triggers the alert.
- **ARP sweeps (internal only):** ARP is broadcast traffic and appears on the LAN switch's port logging and in any network packet capture. IDS sensors on the internal network (not just perimeter) will detect an ARP sweep. Not blockable but fully visible.
- **masscan detection:** masscan's asynchronous mode sends packets in bursts without full TCP stack semantics. Its traffic pattern — many SYNs with no corresponding ACKs, from a single source — is distinctively different from normal browser traffic and trivial to detect.
- **Mitigation for operators:** (1) Use a VPN or proxy that doesn't link back to your real identity. (2) Rate-limit masscan to `--rate=50` or below for external scanning. (3) Spread probes over time: `nmap -sn --PE --PS80,443 --min-hostgroup 1 --scan-delay 1s` adds 1 second between host probes, reducing the scan rate signature. (4) Use OSINT (Shodan) to prefilter which IPs are likely alive before running active probes.

---

# Section 6 — Lab task

**Platform:** TryHackMe "Nmap Live Host Discovery" room, or local VirtualBox network with Metasploitable2 and Kali.

**Objective:** Complete a full multi-probe host discovery workflow, document which probe types succeeded and which failed, and produce a deduplicated alive host list.

**Steps:**

1. **Set up lab network:** Ensure Kali and Metasploitable2 are on the same VirtualBox Host-Only network. Note the network range (`ip a` on Kali).
2. **ICMP sweep:** `fping -a -g <network_range>/24 2>/dev/null > alive_icmp.txt`
3. **nmap multi-probe:** `sudo nmap -sn --PE --PS80,443,22,8080 --PA80 <network_range>/24 -oG nmap_sweep.gnmap`
4. **Extract nmap alive list:** `grep "Up" nmap_sweep.gnmap | awk '{print $2}' > alive_nmap.txt`
5. **ARP sweep (local segment):** `sudo nmap -sn -PR <network_range>/24 -oG nmap_arp.gnmap`
6. **hping3 test on one host:** `sudo hping3 -1 -c 3 <target_ip>` (ICMP), then `sudo hping3 -S -p 80 -c 3 <target_ip>` (TCP). Document which probe type returned a response.
7. **Merge results:** `sort -u alive_icmp.txt alive_nmap.txt > master_alive.txt`
8. **Verify count:** `wc -l master_alive.txt`
9. **Document per-host probe response table:** For each alive IP, record which probe type (ICMP/TCP-SYN/ARP) confirmed it.
10. **Write `host_discovery.md`** with: network range, hosts found per method, comparison table, and notes on which probe types were blocked by the target OS/firewall.

**Expected output:** `alive_icmp.txt`, `alive_nmap.txt`, `master_alive.txt`, `host_discovery.md` showing at least 2 hosts discovered with probe method documented.

```bash
git commit -m "recon(stage3): live host discovery — <N> hosts confirmed alive in <range>"
```

---

# Section 7 — Common mistakes

**1. Using only ICMP ping sweep and concluding "no hosts alive"**
_Why it matters:_ Enterprise firewalls block ICMP by default. Windows Firewall blocks ICMP Echo from external sources. A /24 ping sweep returning 0 results means ICMP is blocked, not that the network is empty.
_Fix:_ Always follow ICMP sweeps with TCP SYN probes to common ports (80, 443, 22, 8080) and merge the results.

**2. Running masscan at maximum rate on external targets**
_Why it matters:_ Masscan at full speed sends millions of packets per second from a single source. This is immediately detected and blocked by any perimeter device. It also risks disrupting target services.
_Fix:_ Cap masscan at `--rate=200` or below for external targets. Use `--randomize-hosts` to spread the traffic. Prefer nmap for targeted scans.

**3. Confusing "no response to probe" with "host is dead"**
_Why it matters:_ A host with a firewall dropping all incoming packets returns no response to any probe — ICMP, TCP SYN, or UDP. It still has live services internally; they are just unreachable externally. Marking it dead removes it from scope incorrectly.
_Fix:_ Document no-response hosts as "unknown state — all probes blocked" rather than dead. Use multiple probe types before concluding. Check Shodan to see if any service was indexed.

**4. Not using ARP on local network segments**
_Why it matters:_ Hosts on the same local segment may have ICMP blocked by their host firewall even locally. ARP bypasses this — it's Layer 2 and cannot be blocked by IP-layer rules. Missing ARP means missing hosts that block all IP traffic but are still on the segment.
_Fix:_ Always run `nmap -sn -PR` or `arp-scan` when operating on a local segment.

**5. Forgetting to save and normalize output across tools**
_Why it matters:_ fping, nmap, masscan, and hping3 all format output differently. Running all four and then manually comparing the results is error-prone. Missing a host from one tool because its IP format didn't match is a real risk.
_Fix:_ Extract IP addresses to plain text files immediately after each tool run. Sort and deduplicate with `sort -u`. Keep all intermediate files in the engagement directory.

---

# Section 8 — Move-on gate

1. A target /24 returns zero responses to an fping sweep. Without any other information, state whether the network is definitively empty and describe exactly which follow-up probe types you would run to determine if hosts are alive behind an ICMP-blocking firewall.

2. Explain why ARP is the most reliable host discovery method on a local segment and the hard limit that makes it useless for external network scanning. Name the nmap flag that performs ARP-based host discovery.

3. You need to sweep a /16 network range (65,536 IPs) for live hosts. Compare the trade-offs between using `nmap -sn`, `fping`, and `masscan` for this task — state which you would use first, why, and what follow-up tool you would use for hosts the first tool missed.
