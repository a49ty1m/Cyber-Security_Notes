# WAF/CDN/TLS Fingerprinting

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

WAF/CDN/TLS fingerprinting identifies which security and delivery infrastructure sits between the attacker and the target web application. A Web Application Firewall (WAF) inspects and filters HTTP requests. A Content Delivery Network (CDN) distributes content from edge nodes, masking the origin server's IP. TLS fingerprinting characterizes the cryptographic identity of the server's TLS implementation.

The intelligence goal is to answer three questions before touching the application: Is there a WAF, and which vendor is it? Is there a CDN, and does it hide the real origin IP? What TLS fingerprint does the server present, and what does that reveal about the backend stack or C2 framework?

```text
Recon Chain
───────────────────────────────────────────────────────────────────────────
Stage 2 (Semi-Passive)               Feeds into:
Subdomain enum   →  [WAF/CDN/TLS Fingerprinting]  →  WAF bypass planning
Protocol audit                                      → Origin IP discovery
                                                    → VHost & favicon hunts
                                                    → Web app attack surface
───────────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** You send an exploit payload to a Cloudflare WAF and get blocked. You scan a CDN IP range instead of the real origin server. Your C2 infrastructure gets fingerprinted by defenders because you used a default TLS configuration. WAF/CDN/TLS fingerprinting prevents all three scenarios by establishing ground truth about the infrastructure layer before any application-level interaction.

This step follows subdomain enumeration (note 03). You now have a list of resolved hostnames — some behind CDN, some with direct IPs. This note teaches you how to determine which is which, what defensive infrastructure is in front of each target, and what the TLS behavior reveals about the backend.

---

# Section 2 — How attackers actually use this

## 2.1 Why WAF and CDN identification matters before attacking

A WAF between you and the application changes every attack technique you use. Injection payloads, path traversal strings, and authentication bypass attempts that work in a lab are blocked by WAF signature rules. If you don't know a WAF is present, you will: (a) exhaust your payload attempts and get your IP banned, (b) misattribute WAF blocks as application-level rejections, and (c) generate unnecessary noise in the target's security tooling.

A CDN in front of the application means the IP you found in DNS is not the origin server — it is a CDN edge node. Scanning or attacking the CDN IP tests the CDN's infrastructure, not the target's. Your findings will be wrong. Finding the origin IP behind the CDN is a prerequisite for accurate scanning and application testing.

## 2.2 WAF fingerprinting with wafw00f and HTTP headers

**wafw00f** sends a series of crafted HTTP requests and analyzes the response signatures to identify the WAF vendor. Each WAF vendor produces distinctive response patterns: specific HTTP headers, error page content, cookie names, and status code behaviors when receiving malformed or suspicious requests.

The workflow:
```text
Send benign request     → Baseline response (no WAF involvement)
Send WAF-probe request  → WAF response (if present)
Compare responses       → Identify WAF vendor from response signatures
```

Beyond wafw00f, HTTP response headers in *normal* responses frequently reveal CDN and WAF presence:

| Header | What it reveals |
|--------|----------------|
| `Server: cloudflare` | Cloudflare CDN/WAF |
| `CF-RAY: 12345abc-LHR` | Cloudflare Ray ID — confirmed Cloudflare, shows edge PoP |
| `X-Sucuri-ID: ...` | Sucuri WAF |
| `X-Powered-By-Plesk: ...` | Plesk hosting (WAF may be present) |
| `X-Cache: HIT from proxy...` | Generic CDN cache hit |
| `Via: 1.1 varnish` | Varnish cache/CDN |
| `X-Amz-Cf-Id: ...` | AWS CloudFront CDN |
| `Akamai-Cache-Status: ...` | Akamai CDN |
| `X-Fastly-Request-ID: ...` | Fastly CDN |
| `X-Kong-Request-Id: ...` | Kong API Gateway |

A single `curl -I` against the target reveals most of this before any WAF-probe requests are sent.

## 2.3 JA3 and JA4 TLS fingerprinting

JA3 is a method of fingerprinting a TLS client (or server) based on fields in the TLS ClientHello message: TLS version, cipher suites offered, extensions, elliptic curves, and elliptic curve point formats. These fields are hashed into a 32-character MD5 string. Because TLS implementations (Java, Python, Golang, Cobalt Strike, Metasploit) have consistent ClientHello patterns, a JA3 fingerprint identifies the *software* initiating or accepting the TLS connection.

JA4 is the successor to JA3, designed to be more stable across minor TLS library updates and produce fewer collisions. It encodes the same handshake parameters in a structured string rather than a direct hash.

**For attackers:** Defenders use JA3 fingerprints of TLS ClientHellos to detect scanning tools, C2 beacons, and malware. If your C2 framework's JA3 fingerprint matches a known malicious tool's fingerprint in the defender's JA3 block list, your beacon is blocked before it sends a single byte of C2 traffic. Knowing the target's JA3 (server-side JARM fingerprint — see note 02) tells you what TLS implementation they are running, which can reveal backend technology.

**For C2 OPSEC:** The reverse also applies — change your C2 client's JA3 fingerprint to match a common benign application (Chrome, Firefox) by modifying your listener's cipher suite and extension ordering. This prevents JA3-based detection of your implants.

## 2.4 TLS ALPN and HTTP/2 as infrastructure fingerprints

ALPN (Application-Layer Protocol Negotiation) is a TLS extension where the client and server declare which application protocol they will use after the TLS handshake completes. The values are: `h2` (HTTP/2), `http/1.1`, `spdy/3.1`, etc.

A server advertising `h2` in ALPN is running HTTP/2. HTTP/2 has different header framing from HTTP/1.1 — WAF bypass techniques that work against HTTP/1.1 may fail or succeed differently against HTTP/2 implementations. Some WAFs handle HTTP/2 differently from HTTP/1.1, creating inconsistency in filtering behavior that is exploitable for bypass.

Reading ALPN is passive — it is visible in the TLS handshake and indexed by Shodan (`ssl.alpn:h2`) and Censys. No application-level request is needed.

## 2.5 CDN origin IP discovery via favicon hash

When a CDN fronts a web application, DNS resolves to CDN edge IPs. The actual application server — the origin — sits behind the CDN at a different IP. Getting the origin IP allows direct scanning and testing that bypasses the CDN's caching, rate limiting, and sometimes WAF.

**Favicon hash technique:**
1. Retrieve the target's `favicon.ico` through the CDN (the CDN serves it).
2. Calculate the murmur3 hash of the favicon bytes.
3. Search Shodan for `http.favicon.hash:<value>` — Shodan scans IPs directly and indexes favicons it finds, regardless of DNS. If the origin server is accessible on any port to Shodan's crawler, its favicon hash is indexed against its real IP.

```text
corp-target.com → Cloudflare IP (CDN edge, not origin)
                        ↓
favicon.ico (served through CDN) → murmur3 hash = 116323821
                        ↓
Shodan: http.favicon.hash:116323821
                        ↓
Results: 203.0.113.45 (AWS IP, no CDN headers)  ← real origin
```

Additional origin IP discovery techniques: historical DNS before CDN was enabled (SecurityTrails, PassiveDNS), leaked IPs in email headers (`X-Originating-IP` in web forms), SSL certificate SAN names that point to non-CDN IPs, and Shodan search for the application's unique HTTP title or response content.

## 2.6 Dead-end vs high-value finding

**Dead-end:** wafw00f identifies Cloudflare. HTTP headers show `CF-RAY:` and `Server: cloudflare`. ALPN is `h2`. favicon hash search returns only the same CDN IPs. JARM fingerprint matches Cloudflare's documented JARM hash. No origin IP exposed. The target is well-protected behind CDN — you know what you're dealing with, but no bypass technique is immediately obvious. Note the WAF vendor and CDN and plan bypass accordingly in Phase 3.

**High-value:** wafw00f returns "Generic WAF detected" but cannot identify vendor. HTTP headers show `X-Powered-By: PHP/7.2.24`, `Server: Apache/2.4.38` — WAF is not intercepting all response headers (WAF is in monitoring mode or improperly configured). Favicon hash search finds `203.0.113.45` — an IP with no Cloudflare headers, serving the same favicon directly on port 8080. `203.0.113.45:8080` bypasses the CDN entirely and has no WAF signatures in its headers. Direct HTTP access to the origin with no WAF protection.

## 2.7 Where results feed next

WAF vendor identification feeds bypass payload selection in Phase 3. CDN origin IP feeds direct application scanning (Stage 3 and Phase 3). TLS ALPN/HTTP2 data feeds application fingerprinting. JA3/JARM fingerprints feed C2 OPSEC planning. VHost hunting (note 06) is a direct follow-on that uses the favicon hash and HTTP response differentiation techniques established here.

---

## 2.8 CDN IP range identification and origin bypass validation

Each major CDN and cloud provider publishes machine-readable lists of their IP address ranges. Knowing which IPs belong to Cloudflare vs. AWS CloudFront vs. Fastly vs. Akamai allows you to instantly confirm CDN presence from an IP address alone — and to distinguish "CDN edge node" from "target's own infrastructure."

| Provider | Published range list |
|----------|-----------------------|
| Cloudflare | `https://www.cloudflare.com/ips-v4` and `ips-v6` |
| AWS CloudFront | `https://ip-ranges.amazonaws.com/ip-ranges.json` (filter `CLOUDFRONT`) |
| Fastly | `https://api.fastly.com/public-ip-list` |
| Akamai | Published quarterly in Akamai support documentation |
| GCP Cloud CDN | Google publishes JSON at their IP range SPF endpoint |

When you find an IP hosting target content — via DNS, Shodan, or favicon hash search — checking it against these published ranges tells you immediately whether it is a CDN edge (stop, find the real origin) or the actual server (proceed with scanning).

```python
import ipaddress, requests

target_ip = ipaddress.ip_address("203.0.113.45")

# Check Cloudflare
cf_ranges = requests.get("https://www.cloudflare.com/ips-v4").text.strip().split()
for cidr in cf_ranges:
    if target_ip in ipaddress.ip_network(cidr):
        print(f"{target_ip} is a Cloudflare edge node")
        break
else:
    print(f"{target_ip} is NOT a Cloudflare IP")

# Check AWS CloudFront
aws = requests.get("https://ip-ranges.amazonaws.com/ip-ranges.json").json()
cf_ips = [p['ip_prefix'] for p in aws['prefixes'] if p['service'] == 'CLOUDFRONT']
for cidr in cf_ips:
    if target_ip in ipaddress.ip_network(cidr):
        print(f"{target_ip} is an AWS CloudFront edge node")
        break
```

If the IP is not in any published CDN range and HTTP headers show no CDN fingerprints, it is either the real origin or infrastructure in a hosting provider (AWS EC2, DigitalOcean, Azure VM) that the organization manages directly.

## 2.9 SecurityTrails and historical DNS for origin IP discovery

SecurityTrails (now Recorded Future Attack Surface Intelligence) archives every DNS record change for a domain — every A record that the domain has ever pointed to, with timestamps. This is the most reliable passive technique for discovering the origin IP that predates CDN adoption.

Organizations typically add Cloudflare or another CDN after their site is already live and running directly on an IP. For weeks or months before CDN adoption, the domain pointed directly to the origin server. SecurityTrails, PassiveDNS, Robtex, and similar services archive those historical records.

```bash
# SecurityTrails API — historical A records
$ curl -s "https://api.securitytrails.com/v1/history/corp-target.com/dns/a" \
  -H "APIKEY: YOUR_KEY" \
  | jq '.records[] | {ip: .values[].ip, first: .first_seen, last: .last_seen}'

{"ip": "203.0.113.45", "first": "2024-01-15", "last": "2024-06-14"}   ← current (CDN edge)
{"ip": "198.51.100.5", "first": "2020-03-01", "last": "2024-01-14"}   ← pre-CDN, likely origin
{"ip": "192.0.2.88",  "first": "2018-06-10", "last": "2020-02-28"}   ← earlier hosting
```

The IP `198.51.100.5` served the domain consistently for nearly four years before Cloudflare was added. If the organization forgot to firewall the origin after migrating to CDN (common), this IP may still have port 443 open and serve the application directly — bypassing Cloudflare's WAF and rate limiting entirely.

```bash
# Validate if pre-CDN origin is still live
$ curl -sk -H "Host: corp-target.com" https://198.51.100.5/ -I
HTTP/1.1 200 OK
Server: Apache/2.4.38         ← no CDN headers, direct access confirmed
Content-Type: text/html

# Query Shodan for current port state of the historical IP
$ shodan host 198.51.100.5
Ports: 22, 80, 443, 8080  ← application still live, direct access possible
```

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **WAF (Web Application Firewall)** | Inspects and filters HTTP/HTTPS traffic based on rules; blocks common web attacks |
| **CDN (Content Delivery Network)** | Distributes web content from geographically distributed edge nodes; hides origin server IP from DNS |
| **Origin server** | The actual backend application server; its IP is masked by the CDN and is what attackers seek for direct access |
| **JA3** | MD5 hash of TLS ClientHello fields (TLS version, cipher suites, extensions, curves); fingerprints TLS client software |
| **JA4** | Successor to JA3; structured string encoding of TLS handshake parameters; more stable across versions |
| **JARM** | Active TLS server fingerprint generated by probing with multiple ClientHellos; fingerprints TLS server implementation |
| **ALPN** | Application-Layer Protocol Negotiation — TLS extension where client/server agree on application protocol (h2, http/1.1) |
| **HTTP/2 (h2)** | Binary multiplexed HTTP protocol; different framing and header format from HTTP/1.1; changes WAF bypass dynamics |
| **Favicon hash** | Murmur3 hash of favicon.ico bytes; identical across all instances of the same application; Shodan-searchable |
| **Murmur3** | Non-cryptographic hash function; Shodan uses it specifically for favicon hashing |
| **CF-RAY** | Cloudflare-specific response header containing request ID and edge location code; confirms Cloudflare fronting |
| **TLS fingerprint** | Unique identifier for a TLS implementation derived from handshake parameters |
| **WAF bypass** | Technique to evade WAF signature matching: encoding, HTTP parameter pollution, header manipulation, chunked encoding |
| **Edge node** | CDN server at a network edge location serving cached content to geographically proximate clients |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `wafw00f` | `wafw00f https://corp-target.com` | WAF vendor identification from HTTP response signatures | WAF detection first pass |
| `wafw00f` | `wafw00f -a https://corp-target.com` | Check against all known WAF fingerprints | Comprehensive WAF check |
| `curl` | `curl -sI https://corp-target.com \| grep -iE 'server\|cf-ray\|x-powered\|via\|akamai\|fastly\|x-amz'` | CDN/WAF/technology headers from response | Header-based fingerprint |
| `curl` | `curl -s --tlsv1.2 --tls-max 1.2 -I https://corp-target.com` | Force TLS 1.2, show headers | TLS version test |
| Shodan | `ssl.alpn:h2 org:"Target Corp"` | HTTP/2 enabled hosts in org | ALPN discovery |
| Shodan | `ssl.jarm:21d19d00021d21d21c21d19d21d21da4a81c32360eb31db10ae3f3d19b4 org:"Target Corp"` | Hosts with specific JARM fingerprint | C2 / framework detection |
| Python `mmh3` | `python3 -c "import requests,mmh3,base64; r=requests.get('https://target.com/favicon.ico'); print(mmh3.hash(base64.encodebytes(r.content)))"` | Calculate favicon murmur3 hash | Origin IP discovery via Shodan |
| Shodan | `http.favicon.hash:116323821` | All IPs serving this specific favicon | CDN origin IP identification |
| `tlsx` | `echo corp-target.com \| tlsx -san -cn -jarm -tls-version -cipher` | Full TLS details including JA3, JARM, SANs | Comprehensive TLS fingerprint |

**Installation:**
```bash
pip install wafw00f
go install github.com/projectdiscovery/tlsx/cmd/tlsx@latest
pip install mmh3 requests   # for favicon hash script
```

**wafw00f — reading the output:**
```bash
$ wafw00f https://corp-target.com
[*] Checking https://corp-target.com
[+] The site https://corp-target.com is behind Cloudflare (Cloudflare Inc.) WAF.
[~] Number of requests: 3
```
Cloudflare confirmed. For bypass: research Cloudflare-specific bypass techniques (header manipulation, encoding, HTTP/2 smuggling), locate origin IP via favicon hash, consider whether the origin IP is accessible without Cloudflare.

**wafw00f — no WAF detected:**
```bash
$ wafw00f https://api.corp-target.com
[*] Checking https://api.corp-target.com
[!] The site https://api.corp-target.com seems to be not protected by any WAF.
```
API subdomain has no WAF. This is common — API endpoints are often not behind a WAF even when the main site is. Direct application testing is possible without bypass planning.

**Header fingerprinting:**
```bash
$ curl -sI https://corp-target.com | grep -iE 'server|cf-ray|x-powered|via|akamai|fastly|x-cache'
Server: cloudflare
CF-RAY: 7f1a2b3c4d5e6f78-LHR
X-Cache: HIT

$ curl -sI https://staging.corp-target.com | grep -iE 'server|x-powered|via'
Server: Apache/2.4.38 (Ubuntu)
X-Powered-By: PHP/7.2.24
```
Production domain is Cloudflare. Staging subdomain shows raw Apache and PHP version — no CDN, no WAF, version exposure. The staging subdomain bypasses the WAF entirely.

**Favicon hash → origin IP discovery:**
```bash
$ pip install mmh3 requests
$ python3 << 'EOF'
import requests, mmh3, base64

response = requests.get("https://corp-target.com/favicon.ico", timeout=5)
hash_val = mmh3.hash(base64.encodebytes(response.content))
print(f"Favicon hash: {hash_val}")
print(f"Shodan query: http.favicon.hash:{hash_val}")
EOF
Favicon hash: 116323821
Shodan query: http.favicon.hash:116323821
```
```bash
$ shodan search 'http.favicon.hash:116323821' --fields ip_str,port,org,hostnames
203.0.113.45  80  AS12345 Target Corp  dev.corp-target.com
203.0.113.45  8080  AS12345 Target Corp  [none]
```
The origin IP is `203.0.113.45`. Port 8080 is accessible directly without Cloudflare.

**tlsx — comprehensive TLS fingerprint:**
```bash
$ echo corp-target.com | tlsx -san -cn -jarm -tls-version -cipher -silent
corp-target.com [203.0.113.45:443] [TLSv13] [TLS_AES_256_GCM_SHA384]
  CN: corp-target.com
  SAN: corp-target.com, api.corp-target.com, staging.corp-target.com
  JARM: 27d40d40d29d40d1dc42d43d00041d4689ee210389f4f6b4b5b1b93f92252d
```
JARM fingerprint `27d40d40...` — cross-reference against known JARM databases. This fingerprint matches nginx + OpenSSL on Ubuntu. SANs reveal additional subdomains.

---

**Cloud provider IP range check — CDN confirmation:**
```bash
# Download all major CDN IP ranges to local files
$ curl -s https://www.cloudflare.com/ips-v4 > cf_ips.txt
$ curl -s https://api.fastly.com/public-ip-list | jq -r '.addresses[]' > fastly_ips.txt
$ curl -s https://ip-ranges.amazonaws.com/ip-ranges.json \
  | jq -r '.prefixes[] | select(.service=="CLOUDFRONT") | .ip_prefix' > cloudfront_ips.txt

# Check a specific IP against all CDN ranges in one script
$ cat > check_cdn.py << 'EOF'
import ipaddress, sys
target = ipaddress.ip_address(sys.argv[1])
for cdn, fname in [("Cloudflare", "cf_ips.txt"), ("Fastly", "fastly_ips.txt"), ("CloudFront", "cloudfront_ips.txt")]:
    for cidr in open(fname):
        if target in ipaddress.ip_network(cidr.strip()):
            print(f"{target} = {cdn} edge node")
            break
    else:
        continue
    break
else:
    print(f"{target} = NOT a known CDN IP")
EOF

$ python3 check_cdn.py 203.0.113.45
203.0.113.45 = NOT a known CDN IP   ← real server or unknown CDN

$ python3 check_cdn.py 104.16.123.96
104.16.123.96 = Cloudflare edge node   ← CDN edge, find origin
```

**SecurityTrails historical DNS via API:**
```bash
# Free account: 50 queries/month at https://securitytrails.com
$ export ST_KEY="your_securitytrails_key"

# Historical A records
$ curl -s "https://api.securitytrails.com/v1/history/corp-target.com/dns/a" \
  -H "APIKEY: $ST_KEY" | jq '.records[] | {ip: .values[].ip, last_seen: .last_seen}'

# Historical MX records (confirm mail provider transitions)
$ curl -s "https://api.securitytrails.com/v1/history/corp-target.com/dns/mx" \
  -H "APIKEY: $ST_KEY" | jq '.records[] | {mx: .values[].hostname, last: .last_seen}'

# List all subdomains SecurityTrails has ever seen
$ curl -s "https://api.securitytrails.com/v1/domain/corp-target.com/subdomains" \
  -H "APIKEY: $ST_KEY" | jq '.subdomains[]'
```
Combining SecurityTrails subdomain list with crt.sh and Amass/Subfinder output maximizes coverage — each source has different data gaps and overlaps. Merging and deduplicating all three is the most complete passive subdomain enumeration strategy available without active DNS queries.

# Section 5 — Defender detection

wafw00f sends crafted HTTP requests including some that trigger WAF rules intentionally. These requests appear in the target's WAF logs as detected probes. The source IP is logged for every wafw00f attempt.

- **wafw00f requests** are recognizable: they send specific payloads designed to trigger WAF detection signatures. Security teams monitoring their WAF logs for probe patterns will see them.
- **Header reading via `curl -I`** is a single HTTP HEAD request — effectively invisible among normal web traffic. No WAF logs this as suspicious.
- **Favicon hash calculation** requires one GET request to `/favicon.ico` — completely normal browser behavior. Indistinguishable from a legitimate visitor loading the page.
- **Shodan favicon hash search** is fully passive — no connection to the target.
- **tlsx** makes direct TLS connections but only completes the handshake — no HTTP request is sent. It appears in connection logs (TCP SYN → TLS handshake → close) without any HTTP access log entry. Less visible than a full HTTP probe.
- **JARM fingerprinting** is active — it requires sending multiple crafted TLS ClientHello packets to the server. The source IP appears in firewall connection logs. Not an HTTP-layer event, but visible at the network level.
- **Mitigation:** Complete all Shodan-based passive fingerprinting before running wafw00f. Use wafw00f only when active probing has been cleared. Use header reading and favicon hash as the primary passive fingerprinting techniques.

---

# Section 6 — Lab task

**Platform:** Kali Linux VM. Target: any Cloudflare-fronted public site you can legally probe for educational purposes (e.g. your own domain on Cloudflare, or a TryHackMe lab target with a WAF in place).

**Objective:** Fingerprint the WAF and CDN of a target, calculate the favicon hash, search for the origin IP via Shodan, and confirm whether the origin is accessible directly — bypassing the CDN.

**Steps:**

1. Install tools: `pip install wafw00f mmh3 requests`
2. Run header fingerprint first (passive): `curl -sI https://<target> | grep -iE 'server|cf-ray|x-powered|via|akamai|fastly|set-cookie'`
3. Run wafw00f: `wafw00f -a https://<target> 2>&1 | tee wafw00f_output.txt`
4. Extract ALPN and TLS version from Shodan: `shodan search 'hostname:<target>' --fields ip_str,port,ssl.alpn,ssl.version`
5. Calculate favicon hash: `python3 -c "import requests,mmh3,base64; r=requests.get('https://<target>/favicon.ico'); print(mmh3.hash(base64.encodebytes(r.content)))"`
6. Search Shodan for favicon hash: `shodan search 'http.favicon.hash:<value>' --fields ip_str,port,org,hostnames | tee favicon_hits.txt`
7. For each non-CDN IP found: `curl -sI http://<ip>:<port> -H "Host: <target>" | head -20` — compare headers to CDN response
8. Run tlsx for full TLS fingerprint: `echo <target> | tlsx -san -cn -jarm -tls-version -cipher -silent | tee tlsx_output.txt`
9. Cross-reference JARM against known JARM databases (e.g. `https://github.com/salesforce/jarm`)
10. Document in `waf_cdn_findings.md`: WAF vendor, CDN vendor, origin IP (if found), TLS version, JARM, ALPN, subdomains from SAN

**Expected output:** `wafw00f_output.txt` identifying WAF vendor, `favicon_hits.txt` with at least one Shodan result (the known CDN IP), `tlsx_output.txt` with JARM and SAN data, and `waf_cdn_findings.md` with the complete fingerprint profile.

**Git artifact:**
```
recon/stage2/waf-cdn-fingerprinting/
├── wafw00f_output.txt
├── favicon_hits.txt
├── tlsx_output.txt
└── waf_cdn_findings.md
```
```bash
git commit -m "recon(stage2): WAF/CDN/TLS fingerprinting — vendor ID and origin IP for <target>"
```

---

# Section 7 — Common mistakes

**1. Assuming CDN presence means the origin IP is completely hidden**
_Why it matters:_ CDNs mask the origin IP in DNS — but the origin IP is often discoverable through favicon hash Shodan searches, historical DNS records, leaked IP headers in email forms, and misconfigured direct-IP access. Treating CDN as an impenetrable barrier causes you to stop investigating before finding the real target.
_Fix:_ Always attempt origin IP discovery via favicon hash, SecurityTrails historical DNS, and Shodan HTTP title/content searches before accepting "protected by CDN" as the final answer.

**2. Treating wafw00f output as definitive**
_Why it matters:_ wafw00f identifies WAFs from signature patterns, but signatures can be wrong, outdated, or overridden by custom configurations. A "no WAF detected" result from wafw00f does not guarantee no WAF — it means no recognized signature was triggered.
_Fix:_ Always cross-reference wafw00f with manual header analysis. Look for WAF-specific cookies (`__cfduid`, `incap_ses_*`), unusual response codes, or error page formats that are WAF-specific but not in wafw00f's database.

**3. Ignoring the ALPN/HTTP2 implication for attack planning**
_Why it matters:_ HTTP/2 uses binary framing and has different header behavior from HTTP/1.1. Some WAFs handle HTTP/2 differently — H2C (HTTP/2 cleartext upgrade) smuggling is a WAF bypass technique that only works when the backend supports HTTP/2 but the WAF processes HTTP/1.1. Not knowing ALPN means missing this attack vector.
_Fix:_ Always record ALPN from Shodan or tlsx. If `h2` is advertised, add H2C smuggling to your bypass candidates for Phase 3.

**4. Not checking staging/dev subdomains for WAF bypass**
_Why it matters:_ Production sites are behind WAFs. Development and staging subdomains frequently are not — they may be on the same origin IP but served without WAF protection because the team didn't put them behind the CDN. This is one of the most common WAF bypass opportunities found in real engagements.
_Fix:_ Run wafw00f and header analysis against every subdomain discovered in note 03, not just the primary domain. The WAF bypass may already be there.

**5. Using a stale favicon hash after a site redesign**
_Why it matters:_ Favicons change when organizations redesign their sites. A favicon hash from six months ago may no longer match the current favicon — Shodan searches with the old hash will find no current results, making you think the origin is hidden when it's actually discoverable with the current hash.
_Fix:_ Always retrieve the favicon fresh from the current live site, not from cached versions. Recalculate the hash each engagement.

**6. Confusing JA3 (client fingerprint) with JARM (server fingerprint)**
_Why it matters:_ JA3 fingerprints the TLS *client* (browser, scanner, C2 beacon). JARM fingerprints the TLS *server* (nginx, Apache, Cobalt Strike listener). Confusing these leads to incorrect conclusions about what is being identified.
_Fix:_ JA3 = what your tool looks like to the server. JARM = what the server looks like to you. Use JARM when identifying server-side TLS implementations. Consider JA3 when thinking about how your scanning tools appear to the target's IDS.

---

# Section 8 — Move-on gate

1. Run wafw00f and header analysis against a target with a known WAF — identify the vendor, confirm it with at least two independent signals (wafw00f output + a specific header), and name one bypass technique specific to that vendor that you would attempt in Phase 3.

2. Calculate the favicon murmur3 hash of a target website, run the Shodan search, and interpret what it means when you get: (a) only CDN IPs in results, (b) a non-CDN IP in a different ASN, and (c) no results at all — for each scenario, what is your next step?

3. From a `tlsx` output showing a specific JARM fingerprint and HTTP/2 ALPN, explain what each value tells you about the server's technology stack and how you would use that information in the next attack phase — without looking at notes.
