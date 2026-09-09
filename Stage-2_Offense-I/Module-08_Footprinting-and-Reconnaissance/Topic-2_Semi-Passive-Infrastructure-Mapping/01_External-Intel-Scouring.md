# External Intel Scouring

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

External intel scouring is the act of querying public threat-intelligence platforms, URL analysis services, and malware sandboxes to retrieve behavioral profiles, historical scan records, URL reputation data, and infrastructure relationships that third parties have already collected about your target. You are not generating new traffic toward the target — you are harvesting what the internet's sensor network has already observed, logged, and published.

It sits at the boundary between pure passive OSINT and active probing. Unlike Stage 1 passive OSINT (WHOIS, CT logs, breach data), which pulls registration and certificate metadata, external intel scouring focuses on the **behavioral and reputation history** of domains, IPs, files, and URLs as observed by global threat feeds, sandbox detonation services, and web crawlers.

```text
Recon Chain
─────────────────────────────────────────────────────────────────────
Stage 1 (Passive)          Stage 2 (Semi-Passive)            Stage 3 (Active)
WHOIS, CT, leaks,    →  [External Intel Scouring]  →  Shodan/Censys  →  Nmap
Breach data,             ↑ YOU ARE HERE                 → Banner grab
Passive DNS                                              → Active enum
─────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** You miss pre-existing sandbox detonations of malware tied to the target's domains, documented phishing infrastructure, reputation signals that flag whether the target's IP space has been publicly weaponized, and the cumulative research of other analysts who may have already mapped the target's infrastructure. You also lose the ability to find JavaScript-loaded API endpoints that no CT log or DNS record will reveal.

Coming from Stage 1 you have confirmed domain names, IP ranges, ASN data, and a subdomain surface. External intel scouring takes those identifiers and enriches them with behavioral intelligence — what the internet has *observed* about those assets — before you move to structured scanning tools.

---

# Section 2 — How attackers actually use this

## 2.1 What attackers are actually looking for

Attackers querying threat intelligence platforms are not simply checking whether a domain is flagged as malicious. They are building an enriched infrastructure map. The concrete questions being answered are:

- Has this domain or IP already been analyzed by any AV engine, threat feed, or researcher? What did they find?
- What IPs has this domain resolved to historically? Are any of those IPs still live?
- What files have communicated with this domain? What do their sandbox reports reveal about internal paths, C2 patterns, and network callouts?
- What does the target's web application look like from the outside, including which API endpoints and third-party scripts load when a browser renders it?
- What URLs, subdomains, and endpoints has any public crawler captured about this organization?
- Is the target's IP space clean, or has it been previously associated with phishing or malware hosting?

## 2.2 VirusTotal pivot workflow

VirusTotal (VT) is the primary infrastructure pivoting platform. It aggregates detections from 90+ security vendors and maintains passive DNS, WHOIS history, certificate data, and file associations for any domain, IP, URL, or file hash submitted to it.

**Step 1 — Domain query.** Submit the target domain. VT returns: current vendor detection ratio, passive DNS history (every IP the domain has ever resolved to), subdomains seen by VT's sensors, sibling domains registered by the same WHOIS registrant, and files that have communicated with this domain (malware samples that called home to it).

```text
corp-target.com
    ↓ passive DNS
    203.0.113.45  (current, since 2024-01)
    198.51.100.12 (previous, 2022-09 to 2023-11)   ← may still be live
    ↓ subdomains seen
    api.corp-target.com, staging.corp-target.com
    ↓ communicating files
    [SHA256: abc123] — 34/72 detections — Cobalt Strike loader
```

**Step 2 — IP pivot.** For each IP found in passive DNS, query that IP in VT. VT shows every domain that has ever resolved to it, the IP's ASN and hosting provider, its HTTPS certificate history (which may reveal more domains), and any files communicating with that IP.

**Step 3 — Sibling domain expansion.** VT shows other domains registered by the same registrant email or organization. These may be test domains, staging environments, or internal project names.

**Step 4 — Communicating files.** For every file hash listed under the target domain, pull the sandbox report. Sandbox reports contain: file type, behavior in execution (processes spawned, registry writes), network connections (exact IPs and domains contacted), strings extracted (internal paths, URLs hardcoded in the binary), and MITRE ATT&CK technique mapping. This is where you find internal infrastructure names and staging URLs that no public DNS record exposes.

## 2.3 urlscan.io for web application recon

urlscan.io is a web crawler that performs live browser-based scans of URLs and archives the complete result: full-page screenshots, every HTTP request the browser made, JavaScript files loaded, cookies set, CSP headers, response headers, DOM structure, and all linked third-party domains. Critically, it indexes these results and makes them searchable — meaning you can query what was captured *about* your target without triggering a new scan.

The intelligence urlscan provides that no DNS record or CT log reveals:

- **JavaScript bundle analysis:** JS bundles in production often contain hardcoded API base URLs, GraphQL endpoints, internal service names, and environment-specific configuration that was never meant to be public. urlscan executes the page and captures every network request the JS code makes.
- **Origin IP discovery through CDN bypass:** If a CDN-fronted site has its origin IP in an API call or a non-CDN resource load, urlscan's network request log shows the direct IP.
- **Third-party integrations:** Every analytics provider, CDN, identity provider, and SaaS embed loaded by the page is captured.
- **Historical application states:** urlscan archives scans over time. A subdomain that was accidentally exposed unauthenticated three months ago may have a public scan showing its full content.

## 2.4 Sandbox detonation history (any.run / Joe Sandbox)

Malware sandboxes publicly archive detonation reports for files and URLs. Searching these archives for the target's domain or IP reveals whether any malware has used that infrastructure as a C2, download server, or exfiltration point.

For red teamers, searching sandbox archives is particularly valuable for phishing kit research: if someone previously detonated a phishing kit targeting `corp-target.com`, the sandbox report shows the attacker's redirect chain, credential harvesting endpoints, the HTML form destination, and sometimes the original dropper URL — revealing the attacker's own infrastructure.

any.run allows free text search across public reports. Joe Sandbox provides API-based search. The queries use the target domain or IP as a search term.

## 2.5 urlvoid for bulk reputation screening

urlvoid aggregates the results of approximately 30 blacklisting engines simultaneously for a single query. It answers the question: has this domain been flagged by any major threat feed? A domain with 0/30 detections is clean across all major feeds. A domain with 5/30 detections is on active blocklists.

For operators, urlvoid serves as a quick confirmation pass across discovered subdomains and IPs — identifying which assets are completely clean (suitable for use as cover infrastructure in simulation) and which are already burned.

## 2.6 Dead-end vs high-value finding

**Dead-end:** VT shows 0/92 detections on `corp-target.com`. Passive DNS shows only the current IP with no historical changes. urlscan has two scans, both showing a static marketing page. No sandbox results. urlvoid shows 0 detections. This target is clean, lightly crawled, and produces no useful pivots — move on to Shodan/Censys.

**High-value:** urlscan shows 40 scans over 6 months. Drilling into the network request log of the most recent scan reveals that the page loads `https://api-internal.corp-target.com/v2/graphql` directly from the browser — a subdomain that does not appear in CT logs. VT passive DNS for that subdomain shows it resolved to `203.0.113.45` until 3 months ago, then moved — but `203.0.113.45` still has an open port 8443 on Shodan hosting an unauthenticated admin panel. VT also shows a communicating file whose sandbox report contains the string `staging.corp-target.com:8080/admin` hardcoded. You now have two live unauthenticated endpoints that were invisible to passive-only OSINT.

## 2.7 Where results feed next

Every artifact found by external intel scouring feeds a downstream task:

- Newly discovered subdomains → Shodan/Censys queries and active DNS resolution (Stage 2 and Stage 3)
- Old IPs from passive DNS → direct Shodan lookups for still-live services
- Exposed API endpoints from urlscan JS analysis → web application attack surface (Phase 3)
- Internal naming conventions from sandbox reports → wordlist construction for active fuzzing
- Identified CDN and WAF vendors → bypass planning (Stage 2 WAF/CDN fingerprinting)
- Confirmed clean IP space → C2 infrastructure selection for later phases

---

## 2.8 OTX and MISP — threat intelligence enrichment

**OTX (Open Threat Exchange)**, now part of LevelBlue (formerly AlienVault), is a public threat intelligence platform where security researchers share IOC collections — malicious domains, IP addresses, file hashes, and URLs — in the form of searchable "pulses." It is freely queryable via API and updated continuously.

For recon operators, OTX serves two purposes: it shows whether the target's infrastructure has been previously documented in threat intelligence (revealing their historical vulnerability posture), and it surfaces IOC relationships linking the target's domains to malware samples, phishing campaigns, or C2 frameworks documented by third-party researchers.

```python
from OTXv2 import OTXv2, IndicatorTypes

otx = OTXv2("YOUR_OTX_API_KEY")

# Get full indicator details for a domain
result = otx.get_indicator_details_full(
    IndicatorTypes.DOMAIN,
    "corp-target.com"
)

# Passive DNS from OTX sensors
print(result['passive_dns'])
# Example output:
# [{"hostname": "corp-target.com", "address": "198.51.100.5", "last": "2023-08-15"}]

# Malware families associated with domain
print(result['malware_families'])
# [{"family": "AgentTesla", "count": 2}]   ← domain was used by AgentTesla C2

# Pulses mentioning this domain
print(result['general']['pulse_info']['pulses'])
```

OTX pulses are analyst-authored threat reports. If a pulse exists for the target's domain, it contains the full IOC set (related IPs, malware hashes, URLs), MITRE ATT&CK technique mapping, and a description of the threat activity. This may reveal that the target organization was previously compromised, that phishing campaigns impersonating them are active, or that their infrastructure was used by a threat actor.

**MISP (Malware Information Sharing Platform)** is the enterprise threat intel sharing platform, self-hosted or operated as a shared community instance. Several public-access MISP instances exist: CIRCL MISP (`https://www.misp.circl.lu/`), Malware Bazaar event feeds, and community threat sharing circles. Searching a public MISP instance for the target domain reveals threat intelligence not published to OTX.

## 2.9 theHarvester for email and hostname aggregation

theHarvester aggregates emails, hostnames, and IPs from multiple search engines (Google, Bing, DuckDuckGo, Baidu), LinkedIn, Hunter.io, Shodan, and other sources in a single automated sweep. It complements dedicated tools by providing a broad first-pass that surfaces email patterns and hostnames without individually querying each source.

```bash
# Install
$ pip install theHarvester
# Or from source: git clone https://github.com/laramies/theHarvester

# Full sweep across all sources (may take several minutes)
$ theHarvester -d corp-target.com -b all -l 500 -f harvest_results
# -b all: query all available sources
# -l 500: limit results to 500 per source
# -f: save HTML and XML output

[*] Emails found:
  john.smith@corp-target.com
  cto@corp-target.com
  helpdesk@corp-target.com
  it-support@corp-target.com

[*] Hosts found:
  mail.corp-target.com:203.0.113.5
  vpn.corp-target.com:203.0.113.50
  jenkins.corp-target.com:203.0.113.47

[*] IPs found:
  203.0.113.45
  203.0.113.50
  198.51.100.5     ← historical IP not in current DNS
```

theHarvester is particularly valuable for email format discovery. Seeing `john.smith@` and `cto@` confirms the organization uses `firstname.lastname` format, which feeds phishing pretext construction in Phase 4 — you can now construct convincing impersonation emails using the correct format and real executive names from LinkedIn OSINT. The historical IP `198.51.100.5` not in current DNS is a pivot point back into Shodan.

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Passive DNS** | Historical log of DNS resolution records collected by sensors across the internet; shows every IP a domain has resolved to over time without querying the target |
| **IOC (Indicator of Compromise)** | Observable artifact tied to malicious activity: domain, IP, file hash, URL, mutex name, registry key |
| **Sandbox detonation** | Automated execution of a file or URL in an isolated virtual machine to observe behavior: processes, network connections, file system changes |
| **Vendor detection ratio** | Number of AV/threat engines flagging a file or URL as malicious out of total engines queried (e.g. 34/72) |
| **Infrastructure pivoting** | Using one artifact (domain, IP, cert, file hash) to discover related artifacts and expand the target map |
| **Infrastructure clustering** | Grouping IPs, domains, and certificates that share registration patterns, hosting providers, or behavioral traits |
| **Page scan** | urlscan.io crawl that captures a full DOM snapshot, all network requests, page screenshot, and header data |
| **JARM fingerprint** | Active TLS fingerprint of a server produced by probing TLS handshake responses; used to identify C2 framework infrastructure across IPs |
| **Favicon hash** | Murmur3 hash of a site's `favicon.ico` file; identical hashes across different IPs indicate the same application stack — searchable in Shodan |
| **Communicating file** | In VT context: a file (usually malware) that has been observed making network requests to the queried domain or IP |
| **Reputation score** | Aggregate risk rating for a domain or IP based on behavioral history across multiple threat feeds |

| Platform | Primary data source | Best used for |
|----------|-------------------|--------------|
| VirusTotal | 90+ AV engines, passive DNS, WHOIS history, file associations | Infrastructure pivoting, malware association, historical IP mapping |
| urlscan.io | Live browser-based crawls, HTTP archive, JS execution, screenshots | Web app recon, JS endpoint discovery, origin IP leaks |
| any.run | Interactive sandbox, process tree, network indicators, MITRE mapping | Malware behavior, phishing kit analysis |
| Joe Sandbox | Deep static + dynamic analysis, memory forensics, YARA hits | Advanced malware and APT detonation data |
| urlvoid | 30 blacklist engine aggregation | Bulk reputation check across discovered assets |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `vt-cli` | `vt domain corp-target.com` | Detections, passive DNS, subdomains, communicating files | Primary domain enrichment |
| `vt-cli` | `vt ip 203.0.113.45` | Domains resolved to IP, ASN, certificate history, communicating files | IP pivot after domain query |
| `vt-cli` | `vt url https://corp-target.com/login` | URL-specific detection and scan history | Targeted URL reputation |
| `vt-cli` | `vt file <SHA256>` | Full sandbox report, detections, network indicators for a file | Communicating file analysis |
| curl (urlscan API) | `curl "https://urlscan.io/api/v1/search/?q=page.domain:corp-target.com&size=100"` | All public scans of target domain, IPs seen, screenshots | Web app fingerprinting |
| curl (urlscan API) | `curl "https://urlscan.io/api/v1/search/?q=page.ip:203.0.113.45&size=100"` | All domains/pages hosted on target IP | IP → vhost discovery |
| curl (urlscan API) | `curl "https://urlscan.io/api/v1/result/<UUID>/" \| jq '.data.requests[].request.url'` | All network requests made during a specific page scan | JS endpoint extraction |
| Python / curl | urlvoid API: `curl "https://www.urlvoid.com/api1000/corp-target.com/?key=KEY"` | Aggregated blacklist status across ~30 engines | Bulk reputation screening |

**Installation:**
```bash
# vt-cli (Go)
go install github.com/VirusTotal/vt-cli/vt@latest
export VT_API_KEY="your_key_here"

# Alternative: Python vt library
pip install vt-py
```

**vt domain — passive DNS output:**
```bash
$ vt domain corp-target.com --format json | jq '.data.attributes'
{
  "last_analysis_stats": { "malicious": 0, "suspicious": 1, "undetected": 88 },
  "last_dns_records": [
    { "type": "A", "value": "203.0.113.45", "ttl": 300 }
  ],
  "historical_whois": [...],
  "subdomains": ["api.corp-target.com", "staging.corp-target.com"],
  "communicating_files": [{ "sha256": "abc123...", "detections": 34 }]
}
```
The subdomain `staging.corp-target.com` was not in CT logs — VT's passive sensors caught it. The communicating file has 34 detections — pull its sandbox report next.

**vt ip — pivot from domain to IP:**
```bash
$ vt ip 198.51.100.12 --format json | jq '.data.attributes.last_https_certificate'
{
  "subject": { "CN": "staging.corp-target.com" },
  "san": ["staging.corp-target.com", "dev.corp-target.com", "vpn.corp-target.com"]
}
```
The old IP's certificate SAN reveals three additional subdomains that were never publicly DNS-advertised — `dev` and `vpn` are high-value targets.

**urlscan search — find existing scans:**
```bash
$ curl -s "https://urlscan.io/api/v1/search/?q=page.domain:corp-target.com&size=50" \
  | jq '.results[] | {url: .page.url, ip: .page.ip, time: .task.time, uuid: .task.uuid}'
{
  "url": "https://corp-target.com/",
  "ip": "203.0.113.45",
  "time": "2024-06-15T08:22:00Z",
  "uuid": "f1a2b3c4-..."
}
```

**urlscan result — extract JS-loaded API endpoints:**
```bash
$ curl -s "https://urlscan.io/api/v1/result/f1a2b3c4-.../" \
  | jq '.data.requests[].request.url' | grep -E 'api|graphql|v[0-9]' | sort -u

"https://api-internal.corp-target.com/v2/graphql"
"https://api-internal.corp-target.com/v2/users/me"
"https://cdn-origin.corp-target.com/assets/main.a1b2c3.js"
```
`api-internal` is a subdomain the JS code revealed that CT logs and passive DNS never showed. `cdn-origin` is likely the real origin IP behind the CDN — query Shodan for that hostname.

**Communicating file sandbox lookup:**
```bash
$ vt file abc123... --format json | jq '.data.attributes.sandbox_verdicts'
# Then pull the network indicators:
$ vt file abc123... --format json | jq '.data.attributes.last_analysis_results'
# Check "network_infrastructure" in any.run manually for the same hash
```

**Python bulk urlvoid check:**
```python
import requests

targets = ["corp-target.com", "api.corp-target.com", "staging.corp-target.com"]
for domain in targets:
    r = requests.get(f"https://www.urlvoid.com/api1000/{domain}/", params={"key": "YOUR_KEY"})
    data = r.json()
    print(f"{domain}: {data.get('detections', 'N/A')}/30 detections")
```

---

**OTX API — domain and IP intelligence:**
```bash
# Install the OTX Python SDK
$ pip install OTXv2

# Query domain
$ python3 << 'EOF'
from OTXv2 import OTXv2, IndicatorTypes
otx = OTXv2("YOUR_API_KEY")

# Domain pulse count
general = otx.get_indicator_details_section(IndicatorTypes.DOMAIN, "corp-target.com", 'general')
print(f"Pulse count: {general['pulse_info']['count']}")
print(f"Reputation: {general.get('reputation', 'N/A')}")

# Passive DNS
passive_dns = otx.get_indicator_details_section(IndicatorTypes.DOMAIN, "corp-target.com", 'passive_dns')
for record in passive_dns.get('passive_dns', []):
    print(f"{record['hostname']} → {record['address']} (last seen: {record.get('last', 'unknown')})")
EOF

# IP query for passive DNS and malware associations
$ python3 -c "
from OTXv2 import OTXv2, IndicatorTypes
otx = OTXv2('YOUR_API_KEY')
result = otx.get_indicator_details_full(IndicatorTypes.IPv4, '198.51.100.5')
for p in result['general']['pulse_info']['pulses']:
    print(f'Pulse: {p[\"name\"]} | Tags: {p[\"tags\"]}')  
"
```

**theHarvester — targeted source queries:**
```bash
# Specific sources for faster targeted queries
$ theHarvester -d corp-target.com -b google,bing,dnsdumpster -l 200

# Email-only sweep via Hunter.io (requires API key in config)
$ theHarvester -d corp-target.com -b hunter -l 100

# LinkedIn employee name harvesting
$ theHarvester -d corp-target.com -b linkedin -l 200
# Returns: employee names that can be combined with known email format

# Output JSON for programmatic processing
$ theHarvester -d corp-target.com -b all -f harvest -o json
$ cat harvest.json | jq '.emails[]'
"john.smith@corp-target.com"
"cto@corp-target.com"

# Extract just IPs for Shodan cross-reference
$ cat harvest.json | jq '.ips[]' | sort -u | xargs -I{} shodan host {} 2>/dev/null
```
Cross-referencing theHarvester IPs with Shodan confirms which are live and what services they expose. IPs found by theHarvester that don't appear in current DNS enumeration are historical or non-DNS-advertised assets — exactly the attack surface that passive DNS-only enumeration misses.

# Section 5 — Defender detection

External intel scouring queries third-party services — the target has **no server-side visibility** into these queries. There are no logs generated on the target's infrastructure from reading VT, urlscan, or any sandbox archive. The OPSEC concerns are on your side, not theirs.

- **VT API key attribution:** VirusTotal logs every query against your API key. A compromised, leaked, or subpoenaed key exposes your complete engagement target list. Use a dedicated burner API key per engagement, never your personal or company key.
- **urlscan public scans:** Searching existing urlscan results is invisible. Triggering a *new* scan via `POST /api/v1/scan/` is public by default — the scan appears in urlscan's public index and any organization monitoring their domain on urlscan will see it. Always set `"visibility": "private"` in the POST body if a new scan is necessary.
- **Sandbox submission OPSEC:** Submitting a target-specific URL or file to any.run or Joe Sandbox in *public* mode makes it visible to all users. A target with an active threat intelligence team may monitor their own domain name in sandbox feeds. Never submit target URLs to public sandboxes during an authorized engagement.
- **VT file upload:** Uploading a custom payload to VT for detection checking distributes the file to AV vendors. The file becomes part of their intelligence databases. Do this only deliberately — never accidentally submit engagement tools to VT during recon.
- **API rate limits:** Aggressive bulk VT querying from a single key can trigger account review or temporary rate limiting. Throttle bulk queries to under 4 per minute on free tier, 500 per day on public API.

---

# Section 6 — Lab task

**Platform:** TryHackMe — *"Threat Intelligence Tools"* room (covers VT and MISP workflow). If unavailable: Kali VM with free-tier VT API key (register at virustotal.com) and a urlscan.io account (free, no key needed for search).

**Target:** `testphp.vulnweb.com` — a legal, intentionally vulnerable web application maintained by Acunetix specifically for security testing. It is publicly enumerable without authorization concerns.

**Objective:** Produce a complete external intel pivot graph showing all discovered IPs, subdomains, and JS-loaded endpoints for the target domain using only VT and urlscan — without visiting the site directly.

**Steps:**

1. Register a free VT account and export your API key: `export VT_API_KEY="your_key_here"`
2. Install vt-cli: `go install github.com/VirusTotal/vt-cli/vt@latest && export PATH=$PATH:$(go env GOPATH)/bin`
3. Query the domain: `vt domain testphp.vulnweb.com -k $VT_API_KEY --format json | tee vt_domain.json`
4. Extract passive DNS IPs: `cat vt_domain.json | jq '.data.attributes.last_dns_records[] | select(.type=="A") | .value'`
5. For each discovered IP, pivot: `vt ip <IP> -k $VT_API_KEY --format json | jq '.data.attributes.last_https_certificate.san[]?' | tee vt_ip_pivot.json`
6. Query urlscan for existing scans: `curl -s "https://urlscan.io/api/v1/search/?q=page.domain:testphp.vulnweb.com&size=50" | jq '.results[] | {uuid: .task.uuid, ip: .page.ip, time: .task.time}' | tee urlscan_results.json`
7. Pull the most recent full scan result: `curl -s "https://urlscan.io/api/v1/result/<UUID>/" | jq '.data.requests[].request.url' | sort -u | tee discovered_endpoints.txt`
8. Check communicating files from VT: `cat vt_domain.json | jq '.data.relationships.communicating_files.data[].id'` — note any SHA256 hashes found.
9. Run urlvoid against the domain: `curl -s "https://www.urlvoid.com/scan/testphp.vulnweb.com/" | grep -i "detected\|engines"`
10. Compile findings into `pivot_table.md`: columns Domain/IP/Subdomain | Source | Finding | Next Action

**Expected output:** `vt_domain.json` with at least one passive DNS IP, `discovered_endpoints.txt` with JS-loaded URLs distinct from the homepage, `pivot_table.md` showing the complete discovery chain, and at least one subdomain or endpoint found through intel scouring that would not appear in passive-only OSINT.

**Git artifact:**
```
recon/stage2/external-intel/
├── vt_domain.json
├── vt_ip_pivot.json
├── urlscan_results.json
├── discovered_endpoints.txt
├── urlvoid_output.txt
└── pivot_table.md
```
```bash
git commit -m "recon(stage2): external intel scouring — VT + urlscan pivot for testphp.vulnweb.com"
```

---

# Section 7 — Common mistakes

**1. Using the primary engagement API key for VT queries**
_Why it matters:_ VT logs all queries against your key. A breach, insider, or subpoena exposes your full target list. Your personal key ties findings to your identity.
_Fix:_ Create a dedicated burner key per engagement. Rotate after use. Never query VT targets from your personal account.

**2. Triggering new urlscan public scans instead of searching existing results**
_Why it matters:_ A new public urlscan scan appears in the global index and is immediately visible to anyone monitoring that domain — including the target's threat intelligence team.
_Fix:_ Always search existing results first (`/api/v1/search/`). Only trigger new scans when absolutely necessary, and always set `"visibility": "private"` in the API POST body.

**3. Treating 0 VT detections as "nothing interesting"**
_Why it matters:_ Zero detections means no AV vendor has analyzed or flagged the domain — not that the domain is clean or unimportant. A new phishing domain used for only 48 hours may have zero VT detections and still be your highest-value finding.
_Fix:_ Treat 0 detections as "not yet analyzed" — still pivot to passive DNS, subdomains, and urlscan crawl history. Absence of detection is not absence of information.

**4. Stopping at the top-level domain**
_Why it matters:_ The primary domain is typically hardened and maintained. The real attack surface is in subdomains, expired IPs, staging environments, and old infrastructure visible in passive DNS and urlscan JS call logs.
_Fix:_ Always pivot: domain → historical IPs → sibling domains → subdomains → communicating files → sandbox reports → internal naming conventions. Depth, not breadth.

**5. Ignoring communicating files in VT**
_Why it matters:_ Files that have communicated with the target domain are often malware samples or phishing kits. Their sandbox reports contain internal paths, staging server hostnames, hardcoded credentials, and network callout sequences that reveal infrastructure invisible to DNS and CT log analysis.
_Fix:_ For every communicating file hash shown in VT, pull the sandbox report (VT sandbox tab, or search the hash on any.run/Joe Sandbox). Extract all network indicators and string artifacts.

**6. Not recording timestamps alongside findings**
_Why it matters:_ Passive DNS records and urlscan entries are time-stamped. An IP that hosted the domain two years ago may be irrelevant, or it may still be a live staging server. Without timestamps you cannot assess whether a finding is current.
_Fix:_ Always record `last_seen` or `task.time` for every DNS record, scan entry, and sandbox report. Include dates in your pivot table.

**7. Submitting target-specific URLs or files to public sandboxes**
_Why it matters:_ Public sandbox submissions are indexed and searchable. If you submit `https://corp-target.com/vpn-login` to any.run in public mode, that URL appears in any.run's public search feed — the target's threat team monitoring their brand in threat intel feeds will see it.
_Fix:_ Search existing sandbox archives instead of submitting. If a submission is operationally necessary, use private mode only.

---

# Section 8 — Move-on gate

1. Query `testphp.vulnweb.com` on VirusTotal without looking at notes — extract all passive DNS IPs, identify which IP is no longer the current resolver, and explain in one sentence what your next step would be with that old IP and which tool you would use.

2. Search urlscan.io for an existing scan of `testphp.vulnweb.com`, pull the full JSON result for the most recent scan, and extract at least three distinct network request URLs that the browser loaded during page render — without visiting the site yourself.

3. Given a VT domain result showing 3 communicating files, 2 historical IPs, and a sibling domain identified via WHOIS registrant match, write out the exact pivot sequence you would follow (each step: what artifact, what tool, what you expect to find) — from memory, without notes.
