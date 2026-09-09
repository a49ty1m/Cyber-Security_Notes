# IPv6 Reconnaissance

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

IPv6 is the successor to IPv4, using 128-bit addresses compared to IPv4's 32-bit. The vast majority of enterprise networks are dual-stack — running IPv4 and IPv6 simultaneously. IPv6 introduces a critical reconnaissance asymmetry: organizations spend significant effort locking down their IPv4 perimeter (firewall rules, IDS tuning, WAF configuration) while their IPv6 infrastructure is frequently an afterthought, with no equivalent security controls applied.

IPv6 recon exploits this asymmetry: services blocked on IPv4 may be reachable on the same host's IPv6 address. Firewalls configured only for IPv4 may pass all IPv6 traffic. IPv6 scanning requires different strategies because the address space (2^128) is fundamentally unscannable with traditional sweep methods.

```text
Stage 5 — IPv6 Reconnaissance
────────────────────────────────────────────────────────────────────
IPv4 perimeter locked  →  [IPv6 Reconnaissance]  →  Bypass controls
WAF, IDS, ACLs on IPv4      ↑ YOU ARE HERE            reach unfiltered
                             Shodan/Censys IPv6         services on IPv6
                             AAAA records              dual-stack misconfig
                             EUI-64 prediction         NDP enumeration
────────────────────────────────────────────────────────────────────
Tools: nmap -6, Shodan, Censys, dig AAAA, THC-IPv6 toolkit, alive6
```

---

# Section 2 — How attackers actually use this

## 2.1 IPv6 address types and their operational significance

```text
IPv6 Address Type        Range              Scope        Significance
─────────────────────────────────────────────────────────────────────
Link-local               fe80::/10          Interface    Always present; no routing
Unique local             fc00::/7           Site-wide    Private equivalent (RFC4193)
Global unicast           2000::/3           Internet     Publicly routable
Multicast (all-nodes)    ff02::1            Link-local   Ping to discover all IPv6 hosts
Multicast (all-routers)  ff02::2            Link-local   Discover IPv6 routers
Loopback                 ::1/128            Local host   Not useful for recon
```

Operationally:
- **Link-local (fe80::/10):** Every IPv6-enabled interface automatically generates a link-local address. These are not routable — only reachable from the same network segment. Covered in note 02.
- **Global unicast (2000::/3):** The externally reachable addresses. These are what Shodan/Censys scan and what DNS AAAA records point to.
- **Unique local (fc00::/7):** Internal use only — equivalent to RFC1918 but with a 40-bit random site identifier making collision avoidance probabilistic.

## 2.2 Finding target IPv6 addresses via DNS AAAA records

The simplest IPv6 discovery method: query DNS for AAAA records (IPv6 equivalents of A records). Organizations that have deployed IPv6 typically configure AAAA records for their public-facing hosts.

```bash
# Query AAAA record for a specific host
$ dig AAAA corp-target.com +short
2001:db8:1234:5678::1    ← global IPv6 address

$ dig AAAA www.corp-target.com +short
2001:db8:1234:5678::2

$ dig AAAA mail.corp-target.com +short
2001:db8:1234:5678::5

# Query all records including IPv6
$ dig corp-target.com ANY +noall +answer
corp-target.com.  300  IN  A      203.0.113.45
corp-target.com.  300  IN  AAAA   2001:db8:1234:5678::1   ← IPv6!
corp-target.com.  300  IN  MX     10 mail.corp-target.com.

# Bulk AAAA lookup for all discovered subdomains
$ while read sub; do
    ipv6=$(dig AAAA "$sub.corp-target.com" +short 2>/dev/null | grep ":")
    [ -n "$ipv6" ] && echo "$sub.corp-target.com → $ipv6"
  done < subdomains.txt

admin.corp-target.com → 2001:db8:1234:5678::100
api.corp-target.com   → 2001:db8:1234:5678::200
vpn.corp-target.com   → 2001:db8:1234:5678::300
```

## 2.3 IPv6 enumeration via Shodan and Censys

Shodan and Censys both index IPv6 addresses. Because most organizations do not apply the same firewall rules to their IPv6 presence, Shodan IPv6 results frequently show services that are hidden on the IPv4 interface.

```bash
# Shodan: search by IPv6 network (discover a target's IPv6 prefix first)
$ shodan search --fields ip_str "net:2001:db8:1234:5678::/64" | head -20

2001:db8:1234:5678::1    corp-target.com       80/tcp 443/tcp
2001:db8:1234:5678::100  admin.corp-target.com 80/tcp 8080/tcp 8443/tcp   ← 8080 visible!
2001:db8:1234:5678::200  api.corp-target.com   443/tcp 9000/tcp           ← 9000 visible!

# 8080/tcp open on admin host via IPv6 but not shown in IPv4 scan → firewall only filters v4

# Search Shodan specifically for IPv6 hosts of a target org
$ shodan search --fields ip_str "org:\"Corp-Target LLC\" has_ipv6:true"

# Censys IPv6 search
$ curl -s "https://search.censys.io/api/v2/hosts/search" \
  -H "Authorization: Basic $(echo -n 'api_id:api_secret' | base64)" \
  -d '{"q": "autonomous_system.organization: \"Corp-Target LLC\" AND ip: /^2001:db8/"}' \
  | python3 -m json.tool

# Look for services that are exposed on IPv6 but not IPv4
# Common pattern: admin interfaces, management ports, dev endpoints
```

## 2.4 EUI-64 IPv6 address prediction from MAC addresses

Many systems use EUI-64 (Extended Unique Identifier-64) to automatically generate IPv6 interface identifiers from their MAC addresses. The formula is deterministic: insert `ff:fe` in the middle of the MAC and flip the 7th bit.

```bash
# EUI-64 conversion formula:
# MAC:  aa:bb:cc:dd:ee:ff
# → Split in half: aa:bb:cc and dd:ee:ff
# → Insert ff:fe: aa:bb:cc:ff:fe:dd:ee:ff
# → Flip 7th bit of first byte: aa XOR 0x02 = a8
# → IPv6 suffix: a8bb:ccff:fedd:eeff

# Python EUI-64 calculator
$ python3 -c "
mac = 'aa:bb:cc:dd:ee:ff'
parts = mac.split(':')
parts[0] = format(int(parts[0], 16) ^ 0x02, '02x')
eui64 = parts[:3] + ['ff', 'fe'] + parts[3:]
suffix = ':'.join(''.join(eui64[i:i+2]) for i in range(0, 8, 2))
print(f'EUI-64 suffix: ::{suffix}')
"
# Output: ::a8bb:ccff:fedd:eeff

# If you know a host's MAC (from ARP table, SNMP, network scan)
# AND you know the IPv6 prefix (from AAAA records, Shodan, or NDP)
# You can predict the host's EUI-64 IPv6 address:
prefix = "2001:db8:1234:5678"
mac = "aa:bb:cc:dd:ee:ff"
→ predicted IPv6: 2001:db8:1234:5678:a8bb:ccff:fedd:eeff

# nmap scan the predicted address to confirm
$ nmap -6 2001:db8:1234:5678:a8bb:ccff:fedd:eeff
```

If you have a list of MAC addresses from SNMP enumeration or ARP data and you know the organization's IPv6 prefix, you can predict every EUI-64 IPv6 address for every host — giving you a targeted scan list instead of sweeping the /64.

## 2.5 IPv6 scanning strategy (the /64 problem)

A typical IPv6 subnet is a /64, containing 2^64 (18 quintillion) addresses. Sequential scanning at 1M packets/second would take 500,000 years. Traditional IP sweep scanning does not work for IPv6. Alternative strategies:

```bash
# Strategy 1: Scan known addresses only (from DNS AAAA records + Shodan)
$ cat ipv6_targets.txt
2001:db8:1234:5678::1
2001:db8:1234:5678::2
2001:db8:1234:5678::100

$ nmap -6 -iL ipv6_targets.txt -sV -p 22,80,443,8080,8443

# Strategy 2: Scan the "low address" range (many admins use ::1, ::2, ::100 etc.)
$ nmap -6 2001:db8:1234:5678::/112   ← /112 = only 65536 addresses — scannable

# Strategy 3: EUI-64 prediction (if MACs are known)
# Generate IPv6 from all collected MACs (from SNMP/ARP)

# Strategy 4: Use alive6 (THC-IPv6) to find hosts via NDP
$ sudo alive6 eth0   ← sends multicast to ff02::1, listens for responses

# Strategy 5: NDP router advertisement snooping
$ sudo ndpmon -i eth0   ← capture NDP messages to discover local IPv6 hosts

# Strategy 6: masscan IPv6 (limited range only)
$ sudo masscan -6 2001:db8:1234:5678::/112 -p 80,443,22 --rate 10000
```

## 2.6 Connecting to IPv6 services with nmap and curl

```bash
# nmap IPv6 port scan
$ nmap -6 -sV -p- 2001:db8:1234:5678::1

# nmap with IPv6 hostname (requires AAAA record)
$ nmap -6 corp-target.com

# curl over IPv6 (wrap IPv6 address in brackets)
$ curl -6 http://[2001:db8:1234:5678::1]/
$ curl -6 https://[2001:db8:1234:5678::1]/
$ curl -6 -H "Host: corp-target.com" https://[2001:db8:1234:5678::1]/

# SSH over IPv6
$ ssh -6 user@2001:db8:1234:5678::1

# Test if a service blocked on IPv4 is reachable on IPv6
$ curl -4 https://admin.corp-target.com/      # → Connection refused (IPv4 blocked)
$ curl -6 https://admin.corp-target.com/      # → HTTP/2 200 OK (IPv6 unfiltered!)
```

## 2.7 IPv6 in certificate SANs and CT logs

IPv6 addresses can appear in TLS certificate SANs and Certificate Transparency logs, just like hostnames and IPv4 addresses:

```bash
# Check TLS cert SANs for IPv6
$ openssl s_client -connect [2001:db8:1234:5678::1]:443 2>/dev/null \
  | openssl x509 -noout -text | grep -A 5 "Subject Alternative"

X509v3 Subject Alternative Name:
    DNS:corp-target.com
    DNS:www.corp-target.com
    IP Address:203.0.113.45
    IP Address:2001:DB8:1234:5678::1    ← IPv6 in SAN!

# crt.sh CT log search for IPv6 SANs
$ curl -s "https://crt.sh/?q=corp-target.com&output=json" \
  | python3 -c "
import json, sys
for cert in json.load(sys.stdin):
    names = cert.get('name_value', '')
    # Filter for IPv6 addresses in SANs
    if ':' in names:
        print(names)
"
```

## 2.8 IPv6 firewall bypass — reaching services blocked on IPv4

The core operational value of IPv6 recon: many organizations apply rigorous firewall policies to IPv4 but leave IPv6 traffic unrestricted, or apply a much smaller rule set to IPv6. This creates a direct bypass path.

```bash
# Test: is port 8080 blocked on IPv4?
$ curl -4 -sk --connect-timeout 5 "https://admin.corp-target.com:8080/"
curl: (7) Failed to connect to admin.corp-target.com port 8080: Connection refused

# Test: same port via IPv6 — is it reachable?
$ curl -6 -sk --connect-timeout 5 "https://[2001:db8:1234:5678::100]:8080/"
HTTP/2 200    ← open on IPv6! Firewall rule was IPv4-only

# Systematically compare IPv4 and IPv6 service exposure for each host
$ for port in 22 80 443 8080 8443 9000 9200 5601 3000 4848; do
    ipv4_result=$(nmap -p $port 203.0.113.45 2>/dev/null | grep "$port/tcp" | awk '{print $2}')
    ipv6_result=$(nmap -6 -p $port 2001:db8:1234:5678::100 2>/dev/null | grep "$port/tcp" | awk '{print $2}')
    if [ "$ipv4_result" != "$ipv6_result" ]; then
        echo "DISCREPANCY port $port: IPv4=$ipv4_result, IPv6=$ipv6_result"
    fi
  done

DISCREPANCY port 8080: IPv4=filtered, IPv6=open    ← admin panel exposed on IPv6
DISCREPANCY port 9200: IPv4=filtered, IPv6=open    ← Elasticsearch on IPv6!
DISCREPANCY port 5601: IPv4=filtered, IPv6=open    ← Kibana on IPv6!

# Common services found only on IPv6 in practice:
# - Internal admin panels (Jenkins, Grafana, Kibana)
# - Database management interfaces (phpMyAdmin, pgAdmin)
# - API management portals
# - Development/staging servers added without updating firewall rules
```

IPv6 exposes infrastructure that administrators added after the firewall rules were last reviewed — new services get global IPv6 addresses by default on dual-stack hosts, but firewall rule updates are often forgotten. Elasticsearch and Kibana exposed only on IPv6 means the entire log database is queryable from the public internet via IPv6, bypassing the IPv4 firewall that was specifically configured to block port 9200.

## 2.9 IPv6 tunnel protocol detection — 6to4, Teredo, and ISATAP

When native IPv6 routing is not available, hosts use transition mechanisms that encapsulate IPv6 packets inside IPv4. These tunnels are an often-overlooked attack surface:

```bash
# 6to4 tunnel detection (IPv6 prefix 2002::/16)
# 6to4 embeds the IPv4 address in the IPv6 address:
# 2002:cb00:712d::  = 203.0.113.45 in hex (cb=203, 00=0, 71=113, 2d=45)
# If you see 2002::/16 addresses in Shodan/DNS → this host is using 6to4

$ python3 -c "
import ipaddress
ipv4 = '203.0.113.45'
addr = ipaddress.IPv4Address(ipv4)
octets = addr.packed
prefix = f'2002:{octets[0]:02x}{octets[1]:02x}:{octets[2]:02x}{octets[3]:02x}::/48'
print(f'6to4 prefix for {ipv4}: {prefix}')
"
6to4 prefix for 203.0.113.45: 2002:cb00:712d::/48

# If you find a 2002::/16 address on a host → that host is using 6to4 tunneling
# Security implication: 6to4 relays forward traffic without authentication
# An attacker can send IPv6 traffic to the target by tunneling it in IPv4 to any 6to4 relay

# Teredo tunnel detection (IPv6 prefix 2001:0000::/32)
# Teredo encapsulates IPv6 in UDP for NAT traversal
# If a host has a Teredo address → it accepts IPv6 connections via UDP
$ nmap -sU -p 3544 203.0.113.45    ← Teredo uses UDP port 3544

# ISATAP tunnel detection (Intra-Site Automatic Tunnel Addressing Protocol)
# ISATAP addresses contain ::0:5efe: followed by the IPv4 address
# Common in Windows corporate environments — interface IDs look like:
# fe80::5efe:203.0.113.45 (link-local ISATAP)

# Detection: look for ::5efe: in IPv6 addresses from Shodan/DNS
$ shodan search "ip:203.0.113.45" | grep -i "isatap\|2001:0:"

# ISATAP implication: ISATAP routers on Windows networks answer multicast queries
# → Query the ISATAP router to discover internal IPv6 addresses
$ ping6 isatap.corp.target.com   ← ISATAP router may be reachable
```

Detecting 6to4 or ISATAP tunnel addresses on Shodan gives you two extra attack vectors: (1) contacting the host via the tunnel mechanism even if native IPv6 is blocked, and (2) the tunnel relay infrastructure itself as a target.

## 2.10 Shodan IPv6 dork cookbook for targeted enumeration

Shodan indexes IPv6 hosts independently from IPv4. The following searches reliably surface high-value services exposed on IPv6 that are missing from IPv4 scans:

```text
Shodan IPv6 dorks — copy/paste ready:

General IPv6 discovery for a target organization:
  org:"Corp-Target LLC" has_ipv6:true
  → Returns all indexed IPv6 hosts belonging to the organization

Admin and management panels on IPv6:
  org:"Corp-Target LLC" port:8080 has_ipv6:true
  org:"Corp-Target LLC" port:8443 has_ipv6:true
  org:"Corp-Target LLC" http.title:"admin" has_ipv6:true
  org:"Corp-Target LLC" http.title:"Jenkins" has_ipv6:true
  org:"Corp-Target LLC" http.title:"Grafana" has_ipv6:true

Databases exposed on IPv6 (frequently unprotected):
  org:"Corp-Target LLC" port:9200 has_ipv6:true        ← Elasticsearch
  org:"Corp-Target LLC" port:27017 has_ipv6:true       ← MongoDB
  org:"Corp-Target LLC" port:5432 has_ipv6:true        ← PostgreSQL
  org:"Corp-Target LLC" port:3306 has_ipv6:true        ← MySQL
  org:"Corp-Target LLC" port:6379 has_ipv6:true        ← Redis

Development tools on IPv6:
  org:"Corp-Target LLC" port:8888 product:"Jupyter" has_ipv6:true
  org:"Corp-Target LLC" port:3000 has_ipv6:true        ← Gitea, Grafana, Node apps
  org:"Corp-Target LLC" port:5000 has_ipv6:true        ← Docker registry, Flask apps

Network infrastructure on IPv6:
  org:"Corp-Target LLC" port:161 has_ipv6:true         ← SNMP on IPv6
  org:"Corp-Target LLC" port:22 has_ipv6:true          ← SSH on IPv6
  org:"Corp-Target LLC" port:3389 has_ipv6:true        ← RDP on IPv6 (extremely unusual externally)

By IPv6 prefix (after identifying the prefix from DNS/AAAA records):
  net:2001:db8:1234:5678::/64 port:80
  net:2001:db8:1234:5678::/64 has_ipv6:true

Historical IPv6 exposure (were services recently removed?):
  net:2001:db8:1234::/48 before:2024-01-01 port:9200
```

```bash
# Shodan CLI IPv6-specific workflow
$ TARGET_ASN=$(shodan search "org:Corp-Target" --fields asn | head -1)
$ echo "Target ASN: $TARGET_ASN"

# Get all IPv6 hosts in the ASN
$ shodan search "asn:$TARGET_ASN has_ipv6:true" --fields ip_str,port,product \
  | grep ":" | tee ipv6_hosts.txt

# Parse unique IPv6 addresses
$ awk '{print $1}' ipv6_hosts.txt | grep ":" | sort -u > ipv6_unique.txt
$ echo "Unique IPv6 hosts: $(wc -l < ipv6_unique.txt)"

# For each IPv6 host, do a targeted nmap scan
$ while read ip; do
    echo "=== $ip ==="
    nmap -6 -sV -p 22,80,443,8080,8443,9200,27017 "$ip" 2>/dev/null | grep "open"
  done < ipv6_unique.txt | tee ipv6_services.txt
```

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **IPv6** | Internet Protocol version 6 — 128-bit addressing replacing IPv4's 32-bit space |
| **Dual-stack** | Host or network running both IPv4 and IPv6 simultaneously |
| **AAAA record** | DNS record mapping a hostname to an IPv6 address (equivalent of A record for IPv4) |
| **EUI-64** | Method for auto-generating a 64-bit interface identifier from a 48-bit MAC address |
| **Link-local** | IPv6 address range `fe80::/10` — auto-generated, non-routable, single segment |
| **Global unicast** | Publicly routable IPv6 address range `2000::/3` |
| **Unique local** | Private IPv6 range `fc00::/7` — equivalent to RFC1918 |
| **NDP (Neighbor Discovery Protocol)** | IPv6 replacement for ARP — discovers neighbors, routers, and address configuration |
| **/64 problem** | The standard IPv6 subnet (/64) has 2^64 addresses — sequential scanning is impossible |
| **SLAAC** | Stateless Address Autoconfiguration — automatic IPv6 address assignment using NDP |
| **alive6** | THC-IPv6 tool for discovering live IPv6 hosts on a segment via multicast probes |
| **THC-IPv6** | Toolkit for IPv6 attacks and enumeration including alive6, detect-new-ip6, fake_router6 |
| **ff02::1** | IPv6 all-nodes multicast address — ping to discover all IPv6 hosts on a segment |
| **has_ipv6:true** | Shodan filter to find hosts with indexed IPv6 addresses |
| **ndpmon** | Tool monitoring IPv6 NDP traffic for host discovery and rogue router detection |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `dig AAAA` | `dig AAAA corp-target.com +short` | IPv6 from DNS | Always first |
| `nmap -6` | `nmap -6 -sV 2001:db8::1` | Port/service scan | Known IPv6 address |
| `Shodan` | `shodan search "org:Target has_ipv6:true"` | Indexed IPv6 hosts | Passive |
| `curl -6` | `curl -6 http://[2001:db8::1]/` | HTTP via IPv6 | Service access test |
| `alive6` | `sudo alive6 eth0` | Live hosts (LAN) | Internal segment |
| EUI-64 calc | Python script (Section 2.4) | Predicted IPv6 from MAC | MAC list available |

---

# Section 5 — Defender detection

- **DNS AAAA queries:** Querying for AAAA records is indistinguishable from normal web browsing — every browser requests AAAA records before A records. Zero detection risk.
- **Shodan/Censys IPv6 search:** Passive — no traffic to target.
- **nmap -6 port scan:** Direct IPv6 connections to the target host. IPv6 traffic may bypass IPv4-focused IDS if the IDS is not configured for dual-stack inspection. Many older Snort/Suricata rules are IPv4-only — an IPv6 port scan may generate zero IDS alerts.
- **IPv6 multicast (alive6/ff02::1):** Link-local multicast pings are highly visible — every host on the segment responds. IDS systems configured for IPv6 monitoring detect multicast-based host discovery immediately.
- **EUI-64 prediction and targeted scan:** If predictions are accurate, the scan is targeted and low-volume — difficult to distinguish from legitimate connection attempts.

---

# Section 6 — Lab task

**Platform:** Kali Linux. Any lab environment with IPv6 enabled. TryHackMe rooms with dual-stack VPN connections.

**Objective:** Discover all IPv6 addresses for a target from passive and semi-passive sources; confirm services accessible on IPv6 that are not accessible on IPv4.

**Steps:**

1. **AAAA record check:** `dig AAAA corp-target.com +short` — document any IPv6 addresses
2. **All subdomain AAAA check:** Loop through discovered subdomains and check each for AAAA records
3. **Shodan IPv6 search:** `shodan search "org:\"Target\" has_ipv6:true" --fields ip_str` — document results
4. **Identify IPv6 prefix:** From the collected addresses, identify the /48 or /64 prefix
5. **Low-range scan:** `nmap -6 <prefix>::/112 -p 80,443,22,8080 --open -T4`
6. **EUI-64 prediction:** For each known MAC (from SNMP/ARP), calculate the predicted EUI-64 address and scan it
7. **Service comparison:** For each discovered service on IPv6, test if the same port is accessible on IPv4 — document discrepancies
8. **TLS cert SAN check:** `openssl s_client -connect [<ipv6>]:443 | openssl x509 -noout -text | grep "Subject Alt"`
9. **CT log IPv6 check:** Query crt.sh for IPv6 addresses in SANs
10. **Compile `ipv6_intel.md`:** All IPv6 addresses | Source (DNS/Shodan/CT) | Services per address | IPv4 vs IPv6 service comparison | EUI-64 predictions made

```bash
git commit -m "recon(stage5): IPv6 reconnaissance — <N> addresses found for <target>"
```

---

# Section 7 — Common mistakes

**1. Assuming no IPv6 because IPv4 firewall is strong**
_Why it matters:_ The two stacks are independent. Excellent IPv4 security does not imply any IPv6 security. Many organizations have mature IPv4 controls but default or no IPv6 firewall rules — the IPv6 side of the same host is effectively unprotected.
_Fix:_ Always check for IPv6 independently, regardless of IPv4 security posture. AAAA record presence is the fastest check.

**2. Trying to scan a full /64 with nmap**
_Why it matters:_ A /64 has 18 quintillion addresses. nmap on a /64 never finishes — it is computationally impossible in human timescales.
_Fix:_ Only scan known addresses (from DNS, Shodan, EUI-64 prediction) or a restricted range (/112 = 65536 addresses, which is scannable).

**3. Not wrapping IPv6 addresses in brackets for curl/URLs**
_Why it matters:_ `curl http://2001:db8::1/` fails — the colons in IPv6 are misinterpreted as URL port separators.
_Fix:_ Always use brackets: `curl http://[2001:db8::1]/`. SSH does not require brackets: `ssh user@2001:db8::1`.

**4. Forgetting to specify `-6` in nmap**
_Why it matters:_ `nmap 2001:db8::1` without `-6` fails with "Failed to resolve host" or interprets the address incorrectly.
_Fix:_ IPv6 targets in nmap always require the `-6` flag: `nmap -6 2001:db8::1`.

---

# Section 8 — Move-on gate

1. `dig AAAA admin.corp-target.com` returns `2001:db8:1234:5678::100`. `nmap -p 8080 admin.corp-target.com` (IPv4) shows the port as `filtered`. `nmap -6 -p 8080 2001:db8:1234:5678::100` shows it as `open`. Without notes, explain why this discrepancy exists, what the security implication is, and what your next action is.

2. You have a MAC address `b8:27:eb:12:34:56` from SNMP enumeration, and you know the target's IPv6 prefix is `2001:db8:1234:5678::/64`. Without notes, calculate the predicted EUI-64 IPv6 address for this host and describe the three-step process for generating any EUI-64 address from a MAC.

3. You attempt `nmap -6 2001:db8:1234:5678::/64` and nmap runs for 10 minutes with no results. Without notes, explain why this scan fails, state the correct scannable prefix length for targeted IPv6 scanning, and describe three alternative strategies for discovering IPv6 hosts in this /64.
