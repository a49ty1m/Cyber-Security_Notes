# DHCPv6 Enumeration

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

DHCPv6 (Dynamic Host Configuration Protocol version 6) is the IPv6 equivalent of DHCPv4. Unlike DHCPv4 which fully manages address assignment, DHCPv6 operates in two modes: stateless (providing supplemental information like DNS servers without assigning addresses — those come from SLAAC) and stateful (managing full address assignment). In both modes, DHCPv6 responses contain valuable network intelligence: the internal DNS server addresses, the internal domain name, NTP server addresses, and in stateful mode, the IPv6 prefix being assigned.

From a recon perspective, sending a DHCPv6 SOLICIT message and reading the ADVERTISE response is equivalent to reading the organization's internal network configuration document — it reveals what DNS servers to query, what domain name the internal hosts use, and what address ranges are in use.

```text
Stage 5 — DHCPv6 Enumeration
────────────────────────────────────────────────────────────────────
Internal segment access  →  [DHCPv6 Enumeration]  →  Internal DNS IPs
(LAN / VPN / pivot)            ↑ YOU ARE HERE          domain name
                               SOLICIT → ADVERTISE      NTP servers
                               dhcpv6-client            prefix info
                               Wireshark decode          config data
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 DHCPv6 protocol flow

The DHCPv6 exchange uses UDP, sending to the all-DHCPv6-servers multicast address (`ff02::1:2` for link-local, `ff05::1:3` for site-scoped). The standard flow:

```text
DHCPv6 Stateful Exchange:
  Client → Server:  SOLICIT (I need an address)
  Server → Client:  ADVERTISE (Here are options — address offer)
  Client → Server:  REQUEST (I want that address)
  Server → Client:  REPLY (Confirmed, here is your lease)

DHCPv6 Stateless Exchange (address from SLAAC, config from DHCPv6):
  Client → Server:  INFORMATION-REQUEST (I need config, no address needed)
  Server → Client:  REPLY (DNS servers, domain, NTP, etc.)

DHCPv6 UDP Ports:
  Client → Server:  UDP port 547
  Server → Client:  UDP port 546
```

The ADVERTISE and REPLY messages contain the intelligence:

```text
DHCPv6 ADVERTISE/REPLY options of interest:
  Option 23: DNS Recursive Name Server    → internal DNS server IPs
  Option 24: Domain Search List          → internal domain name (e.g., corp.target.com)
  Option 56: NTP Server                 → NTP server IPs
  Option 3:  IA_NA (Identity Association for Non-temporary Address)  → assigned IPv6 address
  Option 25: IA_PD (Prefix Delegation)  → delegated prefix (router use)
```

## 2.2 Sending DHCPv6 SOLICIT with dhcptest or dhcpv6-client

```bash
# Method 1: dhcptest (dedicated DHCPv6 testing tool)
$ sudo apt install dhcptest
$ sudo dhcptest --interface eth0 --dhcpv6

# Method 2: Direct dhclient for DHCPv6
$ sudo dhclient -6 -d eth0 2>&1 | tee dhcpv6_output.txt

# Sample dhclient -6 output:
XMT: Solicit on eth0, interval 10460ms.
RCV: Advertise message on eth0 from fe80::1.
    DNS Servers:
    10.10.0.10         ← internal DNS server (IPv4 mapped)
    2001:db8::10       ← internal DNS server (IPv6)
    Domain search list:
    corp.target.com    ← internal domain name!
    NTP servers:
    10.10.0.200        ← NTP server
    XMT: Request on eth0, interval 11020ms.
    RCV: Reply message on eth0 from fe80::1.
    IA_NA:
    Address: 2001:db8:1234:5678::abc  preferred lifetime 3600s, valid lifetime 7200s

# Method 3: nmap DHCPv6 discovery
$ sudo nmap -6 --script broadcast-dhcp6-discover -e eth0

Pre-scan script results:
| broadcast-dhcp6-discover:
|   Interface: eth0
|     Message type: Advertise
|     Transaction ID: 0x3f9a2d
|     DNS Servers: 10.10.0.10, 2001:db8::10
|     Domain Search: corp.target.com
|     Preferred lifetime: 1800
|_    Valid lifetime: 3600
```

## 2.3 Extracting intelligence from DHCPv6 ADVERTISE with Wireshark

When you have a packet capture containing DHCPv6 traffic (from a segment where DHCPv6 is in use), Wireshark decodes all option fields:

```bash
# Capture DHCPv6 traffic
$ sudo tcpdump -i eth0 -w dhcpv6_capture.pcap "udp port 547 or udp port 546"

# Filter DHCPv6 in tshark
$ tshark -r dhcpv6_capture.pcap -Y "dhcpv6" \
  -T fields -e ip.src -e dhcpv6.msgtype -e dhcpv6.dns_servers \
  -e dhcpv6.domain_search_list -e dhcpv6.option.type

# Decode all DHCPv6 options
$ tshark -r dhcpv6_capture.pcap -Y "dhcpv6.msgtype == 2" -V 2>/dev/null \
  | grep -E "DNS|Domain|NTP|prefix|address"

DHCPv6
  Message type: Advertise (2)
  DNS recursive name server: 10.10.0.10
  DNS recursive name server: 10.10.0.11   ← secondary DNS
  Domain Search List: corp.target.com
  Domain Search List: corp.local
  NTP server: 10.10.0.200
  IA Non-temporary Address:
    IPv6 address: 2001:db8:1234:5678::100
    Preferred lifetime: 3600
    Valid lifetime: 7200
```

The domain search list `corp.target.com` confirms the internal AD domain name — the base DN for LDAP queries (note 05) and the realm for Kerberos attacks. The DNS server IPs `10.10.0.10, 10.10.0.11` are almost certainly the domain controllers in a Windows environment (DC = DNS server is the default configuration).

## 2.4 DHCPv6 INFORMATION-REQUEST for stateless config

In networks using SLAAC for addresses and DHCPv6 only for configuration options (stateless DHCPv6), the client sends an INFORMATION-REQUEST rather than SOLICIT:

```bash
# Send INFORMATION-REQUEST to get DNS/domain info without requesting an address
$ python3 - << 'EOF'
# Minimal DHCPv6 INFORMATION-REQUEST with scapy
from scapy.all import *
from scapy.layers.dhcp6 import *

# INFORMATION-REQUEST (type 11)
pkt = IPv6(dst="ff02::1:2") / \
      UDP(sport=546, dport=547) / \
      DHCP6_InfoRequest() / \
      DHCP6OptClientId(duid=DUID_LL(lladdr=get_if_hwaddr("eth0"))) / \
      DHCP6OptOptReq(reqopts=[23, 24, 56])  # Request DNS, Domain, NTP

# Send and receive
send(pkt, iface="eth0", verbose=False)
print("INFORMATION-REQUEST sent — capture response with tshark")
EOF

# Capture the reply
$ sudo tshark -i eth0 -c 5 -Y "dhcpv6.msgtype == 7" -V 2>/dev/null \
  | grep -E "DNS|Domain|NTP"
```

## 2.5 Cross-referencing DHCPv6 data with other recon findings

```text
DHCPv6 intelligence and what it unlocks:

DNS Server IPs discovered:    10.10.0.10, 10.10.0.11
→ These are almost certainly Domain Controllers in Windows (DC = DNS server by default)
→ Target for: LDAP probing (port 389), RPC enumeration (port 135/445), Kerberos
→ Add to high-priority host list

Domain name discovered:       corp.target.com
→ Base DN for LDAP: DC=corp,DC=target,DC=com
→ Kerberos realm: CORP.TARGET.COM
→ SMB FQDN for password spraying: corp.target.com\<username>

NTP Server IP:                10.10.0.200
→ NTP server is often domain controller or dedicated infrastructure server
→ Check if same IP as DNS → confirms DC dual-role

Prefix delegated:             2001:db8:1234:5678::/64
→ Global IPv6 prefix for the segment
→ Use for EUI-64 address prediction (note 01)
→ Input for alive6 and targeted nmap -6 scanning
```

## 2.6 Rogue DHCPv6 server concept (MitM intelligence)

Deploying a rogue DHCPv6 server on a segment where no legitimate DHCPv6 server exists (but hosts are configured for DHCPv6 fallback) causes clients to receive attacker-controlled DNS configuration. This forces internal DNS queries to an attacker-controlled DNS server, revealing every internal hostname being resolved.

```bash
# Informational — rogue DHCPv6 with mitm6 (requires network access)
$ pip install mitm6
# mitm6 sends crafted DHCPv6 ADVERTISE/REPLY with attacker's IP as DNS server
# → Windows hosts use attacker's DNS → WPAD/NTLM auth requests arrive at attacker
# This is an active attack, not pure recon — only in authorized testing

$ sudo mitm6 -i eth0 -d corp.target.com
Starting mitm6 using the following configuration:
Primary adapter: eth0 [aa:bb:cc:dd:ee:ff]
IPv6 address: fe80::a8bb:ccff:fedd:1234
WPAD server: 10.8.0.5
Relaying credentials for: corp.target.com

# Every DNS query from Windows clients (WPAD lookup, authentication) arrives at mitm6
# Capture NTLM hashes with ntlmrelayx.py simultaneously
```

## 2.7 Passive DHCPv6 snooping from existing traffic

If DHCPv6 is active on the segment, clients constantly send SOLICIT messages (on join) and the server sends ADVERTISE responses. Passively capturing this traffic reveals the same intelligence as actively sending a SOLICIT:

```bash
# Passive capture of DHCPv6 exchanges on the segment
$ sudo tshark -i eth0 -Y "dhcpv6" \
  -T fields -e frame.time -e ipv6.src -e ipv6.dst -e dhcpv6.msgtype \
  -e dhcpv6.dns_servers -e dhcpv6.domain_search_list

08:30:22  fe80::a8bb::1234  ff02::1:2  1  (null)          (null)        ← SOLICIT from client
08:30:22  fe80::1           fe80::a8bb  2  10.10.0.10     corp.target.com ← ADVERTISE from server!

# 5 minutes of passive capture on a busy segment catches dozens of DHCPv6 exchanges
# No packets sent to the DHCPv6 server — completely passive
```

## 2.8 Vendor-specific options (Option 17) — deep infrastructure fingerprinting

DHCPv6 Option 17 (Vendor-Specific Information) allows DHCP servers to ship arbitrary vendor-defined data alongside standard config. Enterprise DHCP servers (Windows DHCP, Cisco IPAM, ISC dhcpd) use Option 17 to distribute infrastructure configuration — TFTP server addresses for PXE boot, VOIP configuration URLs, VPN endpoint addresses, and management platform URIs.

```bash
# Capture and decode Option 17 from ADVERTISE messages
$ tshark -r dhcpv6_capture.pcap -Y "dhcpv6.msgtype == 2" -V 2>/dev/null \
  | grep -A 10 "Vendor-specific"

DHCPv6
  Message type: Advertise (2)
  Vendor-specific Information (17):
    Enterprise ID: 311 (Microsoft)
    Sub-option: 1 (WPAD URL)
      Value: http://wpad.corp.target.com/wpad.dat   ← WPAD server!
    Sub-option: 3 (NetBIOS Name Server)
      Value: 10.10.0.10                             ← WINS/NetBIOS server
  Vendor-specific Information (17):
    Enterprise ID: 9 (Cisco Systems)
    Sub-option: 5 (TFTP Server Address)
      Value: 10.10.0.150                            ← TFTP server (PXE boot)
    Sub-option: 6 (Boot File Name)
      Value: pxeboot/wimpe.wim                      ← Windows PXE boot file

# Microsoft Enterprise ID (311) option values:
# Code 1: WPAD URL (Web Proxy Auto-Discovery) → proxy config URL
# Code 3: WINS/NetBIOS server IP
# Code 6: DNS domain suffix

# WPAD URL reveals:
# 1. Internal proxy server hostname: wpad.corp.target.com
# 2. That WPAD autodiscovery is active → WPAD poisoning (via mitm6) will redirect clients
```

The WPAD URL from Option 17 is directly actionable: if `wpad.corp.target.com` is resolvable and reachable, fetching `/wpad.dat` reveals the proxy configuration in use — which may include internal proxy credentials or proxy exclusion lists that reveal internal IP ranges.

```bash
# Fetch WPAD file (if WPAD URL is discovered from DHCPv6)
$ curl -s "http://wpad.corp.target.com/wpad.dat"

function FindProxyForURL(url, host) {
  if (isInNet(host, "10.0.0.0", "255.0.0.0")) return "DIRECT";
  if (isInNet(host, "172.16.0.0", "255.240.0.0")) return "DIRECT";
  if (isInNet(host, "192.168.0.0", "255.255.0.0")) return "DIRECT";
  return "PROXY proxy.corp.target.com:8080";
}
# → DIRECT ranges reveal all internal RFC1918 subnets in use
# → Proxy at proxy.corp.target.com:8080 — new target!
```

## 2.9 Windows-specific DHCPv6 behavior — client vendor class

Windows DHCPv6 clients include a Vendor Class option (Option 16) in SOLICIT messages identifying the client OS. This leaks OS information passively:

```bash
# Capture DHCPv6 SOLICITs to read client OS from Option 16
$ tshark -r dhcpv6_capture.pcap -Y "dhcpv6.msgtype == 1" -V 2>/dev/null \
  | grep -A 5 "Vendor Class"

DHCPv6
  Message type: Solicit (1)
  Vendor Class (16):
    Enterprise ID: 311 (Microsoft)
    Vendor class data: MSFT 5.0   ← Windows client (MSFT 5.0 = Windows Vista+)

# MSFT 5.0 in Vendor Class = Windows operating system (Vista, 7, 8, 10, 11, Server 2008+)
# Linux/macOS clients typically omit Option 16 or use different enterprise IDs

# Implications: if all SOLICITs include "MSFT 5.0" → environment is predominantly Windows
# → Confirms Active Directory environment
# → Prioritize Windows-specific attack vectors (NTLM, Kerberos, LOLBAS)

# Collect OS evidence from passive DHCPv6 traffic:
$ tshark -i eth0 -c 300 -Y "dhcpv6.msgtype == 1" \
  -T fields -e ipv6.src -e dhcpv6.vendorclass.data 2>/dev/null | sort -u

fe80::a8bb:ccff:fedd:1234  MSFT 5.0   ← Windows
fe80::a8bb:ccff:feaa:bbcc  MSFT 5.0   ← Windows
fe80::1234:5678:9abc:def0  (empty)     ← Linux/macOS (no vendor class)
```

Passively monitoring DHCPv6 SOLICITs from the segment builds a per-host OS fingerprint table: Windows hosts include `MSFT 5.0`, Linux/macOS hosts typically omit the vendor class. Combined with MAC OUI (from link-local discovery, note 02) this gives a comprehensive view of the segment's OS distribution before any active scanning.

## 2.10 DHCPv6 lease map and address range reconstruction

DHCPv6 REPLY messages contain the assigned address and lease parameters. By passively collecting REPLY messages from the segment over time, you can reconstruct the entire DHCPv6 lease database — every host that has received a DHCPv6 address, its current IPv6 address, and its link-local address:

```bash
# Passive collection of DHCPv6 REPLY messages to build lease map
$ sudo tshark -i eth0 -c 500 -Y "dhcpv6.msgtype == 7" \
  -T fields -e frame.time -e ipv6.src -e dhcpv6.iaid \
  -e dhcpv6.option.type -e dhcpv6.iana.addr \
  -w dhcpv6_replies.pcap 2>/dev/null &
sleep 120 && kill %1   # collect for 2 minutes

# Parse the collected REPLYs
$ tshark -r dhcpv6_replies.pcap -Y "dhcpv6.msgtype == 7" \
  -T fields -e ipv6.dst -e dhcpv6.iana.addr | sort -u

# Output: (server → client, assigned address)
fe80::a8bb::5678  2001:db8:1234:5678::100   ← host 100 has this IPv6
fe80::a8bb::9abc  2001:db8:1234:5678::101   ← host 101
fe80::a8bb::def0  2001:db8:1234:5678::102   ← host 102

# Infer total number of DHCPv6 clients from IAID (Identity Association ID)
# IAIDs are typically assigned sequentially per-interface
# IAID 1000 on a host means it's the 1000th interface that requested DHCPv6
# Large IAIDs → organization has deployed many DHCPv6 clients

# Calculate address range in use from observed assignments:
$ tshark -r dhcpv6_replies.pcap -Y "dhcpv6.msgtype == 7" \
  -T fields -e dhcpv6.iana.addr | sort -u > assigned_addresses.txt

# First and last assigned addresses define the DHCPv6 pool bounds:
$ head -1 assigned_addresses.txt   ← lowest assigned address
$ tail -1 assigned_addresses.txt   ← highest assigned address
# Pool size = (highest - lowest) + currently assigned count

# nmap scan of the entire observed pool range:
$ nmap -6 2001:db8:1234:5678::100-200 -p 22,80,443 --open 2>/dev/null
# This is a focused, justified scan — you know hosts are in this range from DHCPv6 data

# Cross-reference assigned addresses with Shodan:
$ while read addr; do
    result=$(shodan host "$addr" 2>/dev/null | head -3)
    [ -n "$result" ] && echo "=== $addr ===" && echo "$result"
  done < assigned_addresses.txt
```

The DHCPv6 lease map gives you a ground-truth host list: every IP that appears in an REPLY message is an active, assigned, DHCPv6-managed host. Unlike Shodan (which scans from outside and may miss hosts), and unlike nmap IPv6 scanning (which faces the /64 problem), the lease map is derived directly from the DHCP server's assignments — the most authoritative possible host inventory source.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **DHCPv6** | Dynamic Host Configuration Protocol version 6 — IPv6 address and configuration management |
| **Stateful DHCPv6** | DHCPv6 managing full IPv6 address assignment (similar to DHCPv4) |
| **Stateless DHCPv6** | DHCPv6 providing configuration (DNS, NTP, domain) only; addresses come from SLAAC |
| **SOLICIT** | DHCPv6 client message requesting address assignment or configuration |
| **ADVERTISE** | DHCPv6 server response to SOLICIT containing offered configuration |
| **REQUEST** | DHCPv6 client message accepting an ADVERTISE offer |
| **REPLY** | DHCPv6 server confirmation message completing the exchange |
| **INFORMATION-REQUEST** | DHCPv6 client requesting configuration only (no address) — stateless mode |
| **ff02::1:2** | DHCPv6 all-DHCP-servers multicast address (link-local scope) |
| **Option 23** | DHCPv6 option carrying DNS recursive name server addresses |
| **Option 24** | DHCPv6 option carrying the domain search list (internal domain name) |
| **IA_NA** | Identity Association for Non-temporary Address — carries the assigned IPv6 address |
| **IA_PD** | Identity Association for Prefix Delegation — prefix assigned to a router for downstream clients |
| **mitm6** | Tool sending rogue DHCPv6 to redirect DNS queries to attacker-controlled server |
| **Prefix delegation** | DHCPv6 mechanism for ISPs/routers to assign IPv6 prefixes to downstream networks |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `dhclient -6` | `sudo dhclient -6 -d eth0` | Full DHCPv6 response | Active enumeration |
| `nmap broadcast-dhcp6` | `nmap -6 --script broadcast-dhcp6-discover -e eth0` | DNS, domain, prefix | Quick scan |
| `tshark` | `tshark -r cap.pcap -Y "dhcpv6.msgtype == 2"` | ADVERTISE responses | Pcap analysis |
| `tshark` | `tshark -i eth0 -Y "dhcpv6"` | Live DHCPv6 traffic | Passive snooping |
| `mitm6` | `sudo mitm6 -i eth0 -d corp.target.com` | NTLM via DNS redirect | Authorized testing only |

---

# Section 5 — Defender detection

- **DHCPv6 SOLICIT:** The SOLICIT message is sent to the all-DHCP-servers multicast address — visible to the DHCPv6 server and any device monitoring multicast traffic. Enterprise DHCPv6 servers log all SOLICIT messages with the source MAC and link-local address. A SOLICIT from an unknown MAC generates a DHCP lease assignment log entry.
- **INFORMATION-REQUEST:** Slightly less logged than SOLICIT (no lease is created) but still reaches the DHCPv6 server and is logged by full-featured DHCP servers.
- **Passive snooping:** Completely silent — reading existing DHCPv6 traffic generates no traffic. The only risk is the presence of a packet capture tool on the machine.
- **mitm6 rogue server:** Highly detectable. DHCPv6 guard on enterprise switches (similar to DHCP snooping for IPv4) blocks unauthorized DHCPv6 server responses. Legitimate DHCPv6 deployments almost universally have DHCPv6 guard configured.
- **Mitigation for operators:** Prefer passive snooping (tshark) over active SOLICIT when possible. Passive capture of existing DHCPv6 exchanges reveals the same intelligence without generating any DHCPv6 log entries on the server.

---

# Section 6 — Lab task

**Platform:** Kali Linux on a LAN with DHCPv6 configured (or a lab with a DHCPv6 server — can set up `dnsmasq` with DHCPv6 on a lab VM).

**Objective:** Extract internal DNS server addresses, domain name, and IPv6 prefix from DHCPv6 responses.

**Steps:**

1. **Check for existing DHCPv6 assignment:** `ip -6 addr show eth0` — is there a global IPv6 address? If so, DHCPv6/SLAAC is active
2. **Passive capture:** `sudo tshark -i eth0 -c 200 -Y "dhcpv6" -w dhcpv6.pcap` — wait 2 minutes
3. **Analyze capture:** `tshark -r dhcpv6.pcap -Y "dhcpv6.msgtype == 2 or dhcpv6.msgtype == 7" -V 2>/dev/null | grep -E "DNS|Domain|NTP|prefix"`
4. **Active SOLICIT:** `sudo dhclient -6 -d eth0 2>&1 | grep -E "DNS|Domain|search|NTP"`
5. **nmap broadcast:** `sudo nmap -6 --script broadcast-dhcp6-discover -e eth0`
6. **Compare DNS server IPs with known hosts:** Do the DNS server IPs match the domain controller IPs from other enumeration?
7. **Confirm domain name:** Does the domain search list match the LDAP base DN / Kerberos realm?
8. **Map extracted prefix:** Document the IPv6 prefix for use in targeted nmap -6 scanning
9. **Cross-reference with LDAP:** Use discovered DNS IPs as DC targets for LDAP probing (note 05)
10. **Compile `dhcpv6_intel.md`:** DNS server IPs | Domain search list | NTP servers | IPv6 prefix | Lease time | Cross-references to other notes

```bash
git commit -m "recon(stage5): DHCPv6 enumeration — DNS servers and domain name extracted for <target>"
```

---

# Section 7 — Common mistakes

**1. Not running passive capture first**
_Why it matters:_ Active DHCPv6 SOLICIT generates log entries on the DHCPv6 server. Passive snooping reveals the same information from existing client traffic with zero log impact.
_Fix:_ Always run `tshark -i eth0 -c 200 -Y "dhcpv6"` first for 1-2 minutes before sending active SOLICIT. If existing traffic contains ADVERTISE/REPLY, you have the data for free.

**2. Not recognizing that DHCPv6 DNS servers are likely Domain Controllers**
_Why it matters:_ In Windows environments, the default DNS server for all clients is the Domain Controller(s). DHCPv6 DNS server IPs are therefore likely the most valuable targets in the environment — they run AD, DNS, Kerberos, LDAP, and RPC.
_Fix:_ Every DNS server IP from DHCPv6 should immediately be added to the high-priority host list and targeted with LDAP probing, RPC enumeration, and port scanning.

**3. Treating domain search list as just supplemental info**
_Why it matters:_ The domain search list from DHCPv6 is the internal AD domain name — the exact string needed to construct the LDAP base DN (`DC=corp,DC=target,DC=com`), the Kerberos realm, and the SMB domain. This single string connects all subsequent Windows attack techniques.
_Fix:_ Document the domain search list as a primary finding. It directly feeds LDAP, RPC, Kerberos, and SMB attack workflows.

**4. Confusing DHCPv6 SOLICIT with DHCPv4 discover**
_Why it matters:_ Different ports, different multicast addresses, different message types. `dhclient` (without -6) sends DHCPv4. `dhclient -6` sends DHCPv6. A confusing mistake when working quickly.
_Fix:_ DHCPv6 is always: `dhclient -6`, UDP 546/547, multicast `ff02::1:2`. DHCPv4 is: `dhclient`, UDP 67/68, broadcast `255.255.255.255`.

**5. Not correlating the IPv6 prefix from DHCPv6 with EUI-64 prediction**
_Why it matters:_ The prefix from DHCPv6 ADVERTISE combined with MAC addresses from SNMP/ARP allows predicting every EUI-64 IPv6 address for every known host. This is a direct pipeline to IPv6 host inventory.
_Fix:_ After extracting the prefix from DHCPv6, feed it into the EUI-64 prediction workflow from note 01.

---

# Section 8 — Move-on gate

1. `dhclient -6 -d eth0` output includes `DNS Servers: 10.10.0.10, 10.10.0.11` and `Domain search list: corp.target.com`. Without notes, state: what these DNS server IPs likely represent in a Windows environment, how you use `corp.target.com` in your next LDAP query, and what the Kerberos realm value would be.

2. You are on an internal segment with IPv6 enabled. `tshark -i eth0 -Y "dhcpv6"` captures an ADVERTISE message immediately. Without notes, explain why the passive capture approach is preferable to sending a SOLICIT, state what the ADVERTISE message contains that gives you the same intelligence as a SOLICIT response, and describe the risk of active SOLICIT that passive capture avoids.

3. DHCPv6 REPLY shows `IA_NA Address: 2001:db8:1234:5678::abc`. The DHCPv6 ADVERTISE shows `Domain Search: corp.target.com` and `DNS Servers: 2001:db8::10`. Without notes, extract the /64 prefix from the assigned address, write the nmap command to scan the /112 range, and write the ldapsearch command using the discovered domain name.
