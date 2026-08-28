# Third-Party Scans: Shodan & Censys

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

Shodan and Censys are continuously scanning the entire public internet — every routable IP address, every port, every banner, every certificate — and indexing the results. As a recon operator you query their databases rather than scanning the target yourself. You get the equivalent of a full Nmap/banner-grab sweep of the target's IP space without sending a single packet to the target.

This is the defining characteristic of Stage 2: you are querying what third-party infrastructure has already observed. The distinction from active scanning is not semantic — it is operational. Target-side logs record connections. Shodan and Censys connections were made by *their* crawlers, not you. Querying their API is as passive as reading a webpage.

```text
Recon Chain
──────────────────────────────────────────────────────────────────────
Stage 1 (Passive)        Stage 2 (Semi-Passive)             Stage 3 (Active)
WHOIS, CT, OSINT  →  [Shodan & Censys Queries]  →  Nmap → Banner Grab
                          ↑ YOU ARE HERE             → Service enum
                       Enrich IPs from Stage 1        → Vuln validation
──────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** You either skip intelligence that identifies misconfigured services, exposed admin panels, outdated software versions, and unauthenticated interfaces — or you discover the same information through active scanning, which generates firewall logs, IDS alerts, and connection records on the target. Shodan and Censys give you that same picture without the detection risk.

Coming from Stage 1 and the external intel scouring step you have confirmed IP ranges, ASN identifiers, historical IPs, and subdomains. Shodan and Censys take those IP and domain identifiers and return service-level intelligence: what ports are open, what software is responding, what certificate names are visible, and what geographic and organizational clustering looks like.

---

# Section 2 — How attackers actually use this

## 2.1 What attackers are actually looking for

Operators querying Shodan and Censys are not running generic port scans. They are answering specific attack-surface questions using pre-collected scan data:

- What ports are open on the target's IP range and what services are responding?
- What software versions are exposed? Are any known-vulnerable versions publicly visible?
- Is there an admin panel, remote management interface, or internal application exposed directly to the internet?
- What certificates does the target serve, and what hostnames are in the SANs? (This reveals infrastructure that DNS enumeration missed.)
- Is the same application stack deployed on multiple IPs, identifiable by common certificate SAN or banner patterns?
- Are there any services using default credentials (identifiable from banner content or login page fingerprints)?
- Is the target's infrastructure geographically distributed? Which hosting providers and ASNs are involved?

## 2.2 Shodan workflow — query, filter, pivot

**Step 1 — Seed with ASN or IP range.**
From Stage 1 you know the target's ASN (e.g. `AS12345`). Start broad:
```
org:"Target Corp" OR asn:AS12345
```
This returns all IPs Shodan has observed within that organizational footprint. The initial view shows port distribution, country distribution, and top products.

**Step 2 — Facet analysis.**
Use Shodan facets to answer aggregate questions before drilling into individual hosts:
```
org:"Target Corp" country:US    → count by port to see what services dominate
org:"Target Corp" port:3389     → RDP exposed to internet
org:"Target Corp" port:22       → SSH exposure
org:"Target Corp" product:Apache httpd version:2.2   → outdated Apache
```

**Step 3 — Drill into individual hosts.**
For each interesting host, view the full Shodan record: every scanned port, banner content, HTTP response headers, certificate data, and scan timestamps. The timestamp tells you how fresh the data is — important for assessing current relevance.

**Step 4 — Certificate pivot.**
Shodan indexes TLS certificates and makes them searchable by hostname. If you found a certificate SAN of `vpn.corp-target.com` in Censys CT logs, search Shodan for it:
```
ssl:"vpn.corp-target.com"
```
This returns every IP that is currently serving a certificate containing that hostname — revealing what IP the VPN actually runs on, bypassing CDN or load balancer abstractions.

**Step 5 — Favicon hash pivot.**
Shodan crawls HTTP responses and computes a murmur3 hash of the `favicon.ico`. If the target's admin panel has a distinctive favicon, calculate its hash and search:
```
http.favicon.hash:116323821
```
This finds every IP serving that exact application, worldwide — including instances on cloud IPs or non-standard ports that DNS records do not point to.

## 2.3 Censys workflow — certificate intelligence and cross-validation

Censys maintains the most comprehensive internet-wide TLS certificate database. It indexes certificates from its own ZMap-based scans and from CT log monitoring.

**Certificate-first pivot:** If you have a subdomain from CT logs (`api.corp-target.com`), search Censys to find what IP it resolves to today and what other names share the same certificate:
```
parsed.names: corp-target.com
```
Censys returns every certificate mentioning that domain, including SAN entries. A single certificate may reveal 20 subdomains — staging environments, internal APIs, VPN endpoints — that no DNS query would surface.

**Cross-validate Shodan findings:** Shodan scans may be weeks old. Censys rescans IPv4 space roughly every week. If a Shodan finding shows a critical service, confirm it is still live in Censys before reporting or targeting it.

**Protocol coverage:** Censys scans ports that Shodan may not cover in the same depth: S/MIME certificates, SMTP TLS, FTP TLS, MQTT, and others. For email infrastructure analysis, Censys SMTP data complements the DNS-based mail posture recon from Stage 1.

## 2.4 Dead-end vs high-value finding

**Dead-end:** Shodan shows the target's IP range hosting standard nginx on 80/443 with a valid certificate, no other open ports, no unusual banners, hosted on a major cloud provider (AWS), with scan data from two days ago. The surface is clean and expected. Note it and move on.

**High-value:** Shodan shows `203.0.113.45` has port 8443 open. The banner is `HTTP/1.1 200 OK` with a `Server: Apache Tomcat/7.0.92` header — Tomcat 7 is end-of-life and has multiple CVEs including remote code execution (CVE-2019-0232). The HTTP title is `Manager Application`. The TLS certificate SAN shows `dev.corp-target.com`. This is a development Tomcat Manager UI exposed to the internet, running EOL software, with the application name in the certificate — invisible to DNS enumeration because `dev.corp-target.com` may not resolve publicly, but Shodan found it because it crawled the IP directly.

## 2.5 JARM and infrastructure clustering

JARM is a TLS fingerprint generated by probing a server's TLS handshake behavior. It identifies the underlying TLS implementation — Go's `net/tls`, Python's ssl module, Java's JSSE, Cobalt Strike's default listener, etc. Shodan indexes JARM fingerprints.

If you know a C2 framework's default JARM fingerprint (Cobalt Strike's default JARM is widely documented), searching for that fingerprint across an organization's IP space can reveal active C2 infrastructure. For defenders this is detection; for red teamers, knowing your own C2 JARM fingerprint means avoiding the ones already in threat intel.

## 2.6 Where results feed next

Shodan and Censys findings feed directly into the active phase. Identified open ports and software versions become targets for Stage 3 active fingerprinting. Exposed admin panels become priority targets for credential attacks. Certificate SAN names discovered here feed back into subdomain enumeration. Outdated software versions get queued for CVE lookup and exploit selection in Phase 3.

---

## 2.6 Censys data model and search depth

Censys is architecturally different from Shodan in ways that matter operationally. Shodan stores banners indexed per-port per-IP and updates asynchronously as its crawlers rescan. Censys organizes its data into structured **datasets**: Hosts, Certificates, and Websites \u2014 each queryable with a structured query language rather than Shodan's filter-based approach.

**The Hosts dataset** stores full protocol response bodies parsed into structured fields: TLS certificate details, HTTP response headers, service version, software identifiers, and geographic metadata. Censys uses Elasticsearch under the hood, enabling nested field queries and aggregate statistics not available in Shodan.

**The Certificates dataset** is a direct feed from Certificate Transparency logs, updated faster than Shodan's passive CT ingestion. Querying `parsed.names: corp-target.com` returns every certificate ever issued to any subdomain of the target. This is the fastest path to a complete subdomain list without running Amass or Subfinder.

```bash
# Censys search \u2014 all IPs serving corp-target.com certificate\n$ curl -s \"https://search.censys.io/api/v2/hosts/search\" \\\n  -H \"Accept: application/json\" \\\n  -u \"$CENSYS_API_ID:$CENSYS_API_SECRET\" \\\n  -G --data-urlencode 'q=services.tls.certificates.leaf_data.names: corp-target.com' \\\n  | jq '.result.hits[] | {ip: .ip, port: .services[].port, name: .services[].service_name}'\n\n# Certificates dataset \u2014 all issued certs for domain and subdomains\n$ curl -s \"https://search.censys.io/api/v2/certificates/search\" \\\n  -H \"Accept: application/json\" \\\n  -u \"$CENSYS_API_ID:$CENSYS_API_SECRET\" \\\n  -G --data-urlencode 'q=parsed.names: corp-target.com' \\\n  | jq '.result.hits[] | .parsed.names[]'\n\n# Aggregate: how many hosts per port in org's ASN\n$ curl -s \"https://search.censys.io/api/v2/hosts/aggregate\" \\\n  -u \"$CENSYS_API_ID:$CENSYS_API_SECRET\" \\\n  -d '{\"q\": \"autonomous_system.asn: 12345\", \"field\": \"services.port\", \"num_buckets\": 20}' \\\n  -H \"Content-Type: application/json\" \\\n  | jq '.result.buckets[] | {port: .key, count: .count}'\n\nport: 443, count: 47\nport: 80,  count: 47\nport: 22,  count: 38\nport: 25,  count: 3    \u2190 3 SMTP servers\nport: 3306, count: 2   \u2190 MySQL directly internet-exposed\nport: 27017, count: 1  \u2190 MongoDB internet-exposed\n```

The aggregate query above \u2014 port distribution across the ASN \u2014 is one of the most efficient triage tools in recon. MySQL on port 3306 and MongoDB on port 27017 exposed to the internet in a corporate ASN are always high-severity findings. No credentials needed by default on MongoDB in older versions; MySQL exposure allows brute-force or credential-reuse attacks.

**Certificate chain pivoting:** Censys allows pivoting from a certificate SHA-256 fingerprint to all IPs presenting that exact certificate \u2014 even IPs with no DNS record. This is the most precise version of the favicon hash technique:

```bash
# Get cert fingerprint for a known host\n$ openssl s_client -connect corp-target.com:443 </dev/null 2>/dev/null \\\n  | openssl x509 -fingerprint -sha256 -noout\nSHA256 Fingerprint=AB:CD:EF:...\n\n# Search Censys for all IPs presenting this certificate\n# (in the Censys web UI: search for parsed.fingerprint_sha256:<hash>)\n# Returns all IPs using the same cert \u2014 same-cert infrastructure pivot\n```

## 2.7 Alternative scan platforms \u2014 BinaryEdge, FOFA, and ZoomEye

Shodan and Censys dominate but do not have complete coverage. BinaryEdge, FOFA, and ZoomEye each index different port ranges, geographic regions, and protocol types — making them complementary sources rather than redundant ones.

**BinaryEdge** indexes a broader port range than Shodan's default scan set and explicitly covers ICS and IoT protocols: MQTT (port 1883), CoAP (port 5683), Modbus (port 502), BACnet (port 47808), and EtherNet/IP (port 44818). For targets with operational technology or connected device infrastructure, BinaryEdge data is frequently more complete. It also maintains an RDP screenshot archive and SSH fingerprint index, enabling visual credential assessment without active connections.

**FOFA** (developed by BAIMAOHUI) has substantially stronger coverage of Chinese and APAC IP space. If the target organization operates in Asian markets or has infrastructure on Chinese cloud providers — Alibaba Cloud, Tencent Cloud, Huawei Cloud — FOFA surfaces findings that Shodan and Censys miss. FOFA query syntax: `domain="corp-target.com"`, `cert="corp-target.com"`, `header="X-Powered-By: PHP/7.2"`. The free tier allows 10 queries per day.

**ZoomEye** (Knownsec) focuses on network device fingerprinting — routers, switches, IP cameras, and industrial controllers. Its `app:` operator searches for application names: `app:"Apache httpd" country:CN`. For targets with distributed network hardware or APAC-hosted infrastructure, ZoomEye provides coverage that Western-centric scanners miss.

**Practical workflow:** Run Shodan for the primary sweep — it has the broadest coverage for US/EU infrastructure and the best API tooling. Cross-validate critical findings with Censys (weekly rescan vs. Shodan's variable frequency). For IoT/OT or port-heavy targets, supplement with BinaryEdge. For targets with confirmed APAC presence, add a FOFA query for the primary domain and ASN.

## 2.8 Shodan Alerts for continuous monitoring

For longer engagements or persistent red team operations, Shodan Alerts convert the platform from point-in-time queries into a continuous intelligence feed. When Shodan's crawler detects a new open port, new service banner, or a new CVE signature in a monitored IP range, it triggers a notification. This is operationally significant: if the target organization deploys a new service or accidentally exposes a management interface during the engagement window, Shodan's alert fires before your next scheduled recon phase.

```bash
# Create an alert for target IP CIDR
$ shodan alert create "Target Corp Monitoring" 203.0.113.0/24

# List configured alerts
$ shodan alert list
ID         Name                    IP Range           Triggers
aabbccdd   Target Corp Monitoring  203.0.113.0/24     new-port,vuln,ssl-expired

# Available trigger types:
# new-port      → new open port detected on any IP in range
# vuln          → Shodan identifies a new CVE on a host in range
# ssl-expired   → TLS certificate expires
# new-ip        → new IP added to monitored range
```

Shodan Alerts also detect *defensive* changes: if the organization adds a new IP to the monitored CIDR (spinning up additional infrastructure), the `new-ip` trigger fires. If they patch a service and close a port, the absence of a trigger confirms the change. Continuous monitoring turns Shodan into a near-real-time infrastructure feed for the target's IP space.

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Banner** | The text a service sends when a connection is established; contains software name, version, and sometimes configuration details |
| **ASN (Autonomous System Number)** | Number identifying a network operated by one organization; used to scope Shodan/Censys queries to a target's full IP space |
| **Facet** | Shodan aggregation filter that groups results by a field (port, country, org, product) to show distribution |
| **JARM** | Active TLS fingerprint based on ordered server responses to crafted TLS ClientHello probes; identifies the TLS implementation |
| **Favicon hash** | Murmur3 hash of a `favicon.ico` file; identical hashes across IPs indicate the same application — searchable in Shodan |
| **Censys perspective** | Censys term for a single scan result from one of its scan vantage points |
| **Certificate transparency (Censys)** | Censys monitors CT logs and indexes all certificates, making them searchable by domain name or organization |
| **Product** | Shodan field identifying the software name extracted from the banner (e.g. `Apache httpd`, `OpenSSH`, `nginx`) |
| **CVE** | Common Vulnerabilities and Exposures — identifier for a known vulnerability; Shodan can filter by `vuln:CVE-XXXX-XXXXX` for some vulnerabilities |
| **EOL (End of Life)** | Software version no longer receiving security patches — any known vulnerability in it is permanently unpatched |
| **HTTP title** | Text content of the `<title>` tag of a web page, indexed by Shodan — useful for identifying login pages and admin panels |
| **Port/service distinction** | Port number (e.g. 8443) identifies the channel; service identifies the protocol (HTTPS); banner identifies the software |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `shodan` CLI | `shodan search 'org:"Target Corp"'` | All hosts in org's IP space with open ports and banners | Initial broad sweep |
| `shodan` CLI | `shodan search 'asn:AS12345 port:3389'` | RDP-exposed hosts within the ASN | Remote access surface discovery |
| `shodan` CLI | `shodan host 203.0.113.45` | All open ports, banners, and certificates for one IP | Drill down on a specific host |
| `shodan` CLI | `shodan search 'ssl:"corp-target.com" product:Apache'` | Apache servers serving certs with the target domain | Certificate + software pivot |
| `shodan` CLI | `shodan search 'http.favicon.hash:116323821'` | Every IP serving a specific favicon globally | Application fingerprint pivot |
| `shodan` CLI | `shodan search 'org:"Target Corp" vuln:CVE-2021-44228'` | Log4Shell-vulnerable hosts in the org | Vulnerability-specific query |
| Censys CLI | `censys search 'parsed.names: corp-target.com'` | All certificates mentioning the domain name | Certificate and SAN expansion |
| Censys CLI | `censys search 'autonomous_system.asn: 12345'` | All hosts in ASN indexed by Censys | ASN-scoped sweep |
| Censys web | `https://search.censys.io/search?q=corp-target.com` | Interactive certificate and host pivot | Quick manual exploration |

**Installation:**
```bash
pip install shodan censys
shodan init YOUR_API_KEY
export CENSYS_API_ID="your_id"
export CENSYS_API_SECRET="your_secret"
```

**Shodan org search — reading the output:**
```bash
$ shodan search 'org:"Target Corp"' --fields ip_str,port,product,version
203.0.113.45    443   nginx          1.18.0
203.0.113.45    22    OpenSSH        8.9p1
198.51.100.5    8443  Apache Tomcat  7.0.92
198.51.100.5    22    OpenSSH        7.4
203.0.113.46    3306  MySQL          5.7.38
```
MySQL on port 3306 is exposed directly to the internet with no indication of authentication restriction. Apache Tomcat 7.0.92 is EOL — both are immediate priorities. OpenSSH 7.4 has known CVEs.

**Shodan host drill-down:**
```bash
$ shodan host 198.51.100.5
IP:          198.51.100.5
Organization: Target Corp
ASN:          AS12345
Ports:        8443, 22
Last update:  2024-06-14

Port 8443 / tcp
  Transport: tcp
  Product:   Apache Tomcat
  Version:   7.0.92
  HTTP Title: Apache Tomcat/7.0.92
  Server:    Apache-Coyote/1.1
  Certificate:
    CN: dev.corp-target.com
    SAN: dev.corp-target.com, staging.corp-target.com
```
The certificate SAN exposes two subdomains. The HTTP title confirms the Tomcat Manager UI is the default page — meaning the Manager endpoint may be accessible.

**Censys certificate pivot:**
```bash
$ censys search 'parsed.names: corp-target.com' --fields parsed.names,parsed.subject_dn,metadata.updated_at
{
  "parsed.names": ["corp-target.com", "api.corp-target.com", "dev.corp-target.com",
                   "staging.corp-target.com", "vpn.corp-target.com", "admin.corp-target.com"],
  "parsed.subject_dn": "CN=corp-target.com",
  "metadata.updated_at": "2024-06-10T14:22:00Z"
}
```
Six hostnames from one certificate lookup — `admin` and `vpn` are high priority. Query each in Shodan to find their current IPs.

**Favicon hash — finding hidden instances:**
```python
import requests, mmh3, base64

r = requests.get("https://corp-target.com/favicon.ico")
favicon_hash = mmh3.hash(base64.encodebytes(r.content))
print(f"http.favicon.hash:{favicon_hash}")
# → http.favicon.hash:116323821
# Then query Shodan: shodan search 'http.favicon.hash:116323821'
```

---

**BinaryEdge CLI — IoT and protocol-specific queries:**
```bash
$ pip install python-binaryedge
# Configure: export BINARYEDGE_API_KEY="your_key"

# Query all open ports for a specific IP
$ binaryedge host 203.0.113.45

# Search for MQTT brokers in target org's CIDR
$ binaryedge search 'ip:"203.0.113.0/24" port:1883'
# MQTT on port 1883 = unauthenticated IoT message broker — critical severity

# Search for Modbus (ICS) exposure
$ binaryedge search 'ip:"203.0.113.0/24" port:502'
# Modbus without authentication = direct PLC/SCADA access vector
```
MQTT on port 1883 in a corporate IP range is a maximum-severity finding — it indicates an IoT/OT broker accessible without TLS or authentication, allowing any client to subscribe to all message topics including sensor data, actuator commands, and automation telemetry.

**Shodan CVE sweep across ASN — multi-vulnerability query:**
```bash
# Log4Shell (CVE-2021-44228) across entire ASN
$ shodan search 'asn:AS12345 vuln:CVE-2021-44228' --fields ip_str,port,product,version

# EternalBlue (MS17-010) — SMB RCE
$ shodan search 'asn:AS12345 vuln:CVE-2017-0144' --fields ip_str,port

# ProxyLogon (CVE-2021-26855) — Exchange Server
$ shodan search 'asn:AS12345 vuln:CVE-2021-26855' --fields ip_str,port

# Multiple CVEs in one query
$ shodan search 'asn:AS12345 (vuln:CVE-2021-44228 OR vuln:CVE-2021-26855 OR vuln:CVE-2017-0144)' \
  --fields ip_str,port,product,version
```
Shodan only indexes vulnerabilities where its scanner can confirm the version fingerprint — absence from results is not a clean bill of health. But a positive match is high-confidence: `port:445 vuln:CVE-2017-0144` means SMB with confirmed EternalBlue exposure — unauthenticated RCE without authentication. Treat any CVE hit as a verified pre-engagement finding.

**Shodan historical data — past banners for an IP:**
```bash
$ shodan host 203.0.113.45 --history
# Output: every timestamp Shodan scanned this IP and what it found
# Shows services that were open in the past but are now closed
# Old admin panels, test services, and debug endpoints visible in history

Timestamp: 2023-09-15
  Port: 8080, Product: Apache Tomcat, Version: 7.0.92   ← now closed, was EOL
  Port: 9200, Product: Elasticsearch, Version: 6.8.0    ← was unauthenticated

Timestamp: 2024-06-14 (current)
  Port: 443, Product: nginx, Version: 1.18.0
  Port: 22, Product: OpenSSH, Version: 8.9p1
```
Historical data reveals the *remediation gap*: services that Shodan saw open are now closed. This confirms the organization is actively hardening its surface — but also shows what vulnerabilities existed in the past, which may inform social engineering narratives or indicate which CVEs were likely present before patching.

# Section 5 — Defender detection

Shodan and Censys scan your target continuously regardless of whether you query their databases. The target's IPs appear in Shodan and Censys because Shodan's and Censys's crawlers already connected to them. Your query against the Shodan or Censys API generates no new connection to the target.

- **Shodan crawler IPs are documented and blockable.** Organizations that block Shodan's known scanner ranges (`198.20.69.74`, `198.20.70.114`, etc.) will have incomplete or stale Shodan data. This is worth noting — if Shodan shows no data for an IP range, it may mean the target actively blocks scanners, not that no services are running.
- **Censys crawler IPs are similarly documented.** Some organizations add Censys scanner IPs to blocklists. Censys publishes its scanner IP list at `censys.io/scanners`.
- **Data staleness is a real risk.** Shodan data can be days to months old depending on the scan frequency for that IP range. Always note the `last_update` timestamp. Treat old findings as hypotheses to validate — not confirmed vulnerabilities.
- **Shodan Favicon scan detection:** Shodan's HTTP crawler accesses `GET /favicon.ico` — this connection *was* made to the target by Shodan's crawler, not by you. But if a target monitors web access logs for Shodan scanner IPs, they see Shodan's crawler, not yours.
- **Your query is your only OPSEC exposure.** Use a Shodan API key registered to an account not tied to your identity or employer. Shodan API queries are logged against the key.

---

# Section 6 — Lab task

**Platform:** Shodan free account (register at shodan.io — the free tier allows searches and the CLI). Censys free account for certificate queries. Kali Linux for CLI tools.

**Target:** Use Shodan to enumerate the internet-facing services of a publicly known vulnerable/intentionally exposed organization. TryHackMe's "Shodan.io" room targets are pre-authorized. Alternatively: query `org:"HackTheBox"` or any organization that publicly acknowledges their security research infrastructure is intentionally exposed.

**Objective:** Produce a host inventory table for the target ASN showing all exposed services, identify at least one high-priority finding (EOL software, exposed admin panel, or unexpected service), and pivot from a certificate SAN to a new subdomain not found in Stage 1 OSINT.

**Steps:**

1. Install tools: `pip install shodan censys && shodan init YOUR_KEY`
2. Identify the target's ASN from Stage 1 WHOIS data: `shodan search 'org:"Target Name"' --limit 1 --fields org,asn`
3. Broad sweep: `shodan search 'asn:ASXXXXX' --fields ip_str,port,product,version,http.title | tee shodan_broad.txt`
4. Count exposed ports: `cat shodan_broad.txt | awk '{print $2}' | sort | uniq -c | sort -rn`
5. Check for remote access protocols: `shodan search 'asn:ASXXXXX port:3389 OR port:5900 OR port:23'` — RDP, VNC, Telnet
6. Check for EOL products: `shodan search 'asn:ASXXXXX product:"Apache Tomcat" version:7'`
7. Drill into one interesting host: `shodan host <IP> | tee host_detail.txt`
8. Extract certificate SANs: `shodan search 'asn:ASXXXXX' --fields ssl.cert.subject.cn,ssl.cert.subject_alt_name | grep target-domain`
9. Cross-validate in Censys: `censys search 'autonomous_system.asn: XXXXX' --fields ip,services.port,services.software.product`
10. Record all findings in `host_inventory.md` with columns: IP | Port | Product | Version | Risk | Source | Timestamp

**Expected output:** `shodan_broad.txt` with at least 5 distinct hosts, `host_detail.txt` for one high-interest host showing certificate SAN data, `host_inventory.md` with a complete risk-graded table, and at least one subdomain discovered from certificate SAN data not found in Stage 1.

**Git artifact:**
```
recon/stage2/third-party-scans/
├── shodan_broad.txt
├── host_detail.txt
├── censys_certs.json
└── host_inventory.md
```
```bash
git commit -m "recon(stage2): Shodan + Censys sweep — host inventory and CVE surface for <target>"
```

---

# Section 7 — Common mistakes

**1. Trusting stale Shodan data without checking the timestamp**
_Why it matters:_ Shodan data can be weeks or months old. A service visible on Shodan six weeks ago may have been patched, decommissioned, or moved behind a firewall. Acting on stale findings without validation wastes time and produces false positives in reports.
_Fix:_ Always check the `last_update` field on every Shodan host record. Treat findings older than 30 days as leads to validate, not confirmed vulnerabilities.

**2. Confusing "querying Shodan" with "actively scanning the target"**
_Why it matters:_ Operators sometimes include Shodan queries in engagement reports under "active scanning" methodology. This is incorrect and undersells the technique. It is also the reverse misunderstanding — assuming Shodan queries leave logs on the target.
_Fix:_ Querying Shodan or Censys is passive intelligence collection. The scan connections were made by Shodan's crawlers. Your API query generates zero target-side events.

**3. Ignoring Shodan facets — drilling into hosts before understanding the big picture**
_Why it matters:_ Looking at individual hosts before understanding the full org profile leads to tunnel vision. You may spend time on one host while a more critical service on a different IP is overlooked.
_Fix:_ Run a faceted summary first (`port`, `product`, `country`, `vuln`) to understand the overall surface. Identify the highest-risk categories before drilling into individual hosts.

**4. Using only `org:` and missing hosts registered under different org names**
_Why it matters:_ Large organizations may have subsidiaries, cloud infrastructure, or acquired companies registered under different organization names in IP registration records. `org:"Target Corp"` misses `org:"Target Corp Europe"`, `org:"Target Payments Ltd"`, etc.
_Fix:_ Cross-reference the ASN list from Stage 1 WHOIS and use `asn:` queries in addition to `org:` queries. Query each known ASN individually.

**5. Not correlating Shodan banners with known CVEs**
_Why it matters:_ Seeing `Apache httpd 2.4.49` in a banner is only half the value. That version is affected by CVE-2021-41773 (path traversal/RCE). Without cross-referencing the version against CVE databases, you produce a host list with no actionable severity grading.
_Fix:_ For each software version found, check NVD or `shodan search 'org:"..." vuln:CVE-XXXX-XXXXX'` to confirm exposure. Build version→CVE mappings as part of the host inventory table.

**6. Forgetting to check Censys for cross-validation when Shodan data seems incomplete**
_Why it matters:_ Shodan and Censys scan different port ranges on different schedules. A service visible on Censys may not appear in Shodan at all, and vice versa. Using only one source produces gaps.
_Fix:_ For high-value targets, always run both Shodan and Censys queries and merge the findings. Discrepancies (a port visible in one but not the other) are also intelligence — they may indicate selective blocking of one scanner's IP range.

---

# Section 8 — Move-on gate

1. Given a target ASN number, construct the complete Shodan query sequence — from broad org sweep to certificate pivot to specific vulnerability filter — without looking at notes, and explain what each filter is selecting and why you run them in that order.

2. Retrieve a Shodan host record for a specific IP and correctly identify: the software name and version, whether that version is EOL, the TLS certificate CN and SAN names, and the timestamp indicating how fresh the data is — all from the single `shodan host <IP>` output.

3. Calculate the murmur3 favicon hash for a target web application, run the Shodan favicon hash search, and explain what it means if multiple unrelated IPs return the same hash versus if only the known target IP matches.
