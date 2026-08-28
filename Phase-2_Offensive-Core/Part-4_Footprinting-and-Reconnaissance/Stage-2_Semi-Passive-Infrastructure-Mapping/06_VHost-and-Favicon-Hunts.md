# VHost & Favicon Hunts

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

Virtual host (vhost) hunting is the technique of discovering web applications that are served from a shared IP address but are only reachable when the correct `Host:` header is sent in the HTTP request. Web servers, load balancers, and reverse proxies use the `Host:` header to route incoming requests to the correct backend application. If you send `Host: corp-target.com`, you get the main site. If you send `Host: admin.corp-target.com` to the same IP, you may get an entirely different application — one that DNS does not expose and that no subdomain enumeration would find.

Favicon hunting is the complementary passive technique: using the murmur3 hash of an application's `favicon.ico` to search Shodan for all IPs that serve that same favicon, revealing hidden application instances deployed on non-standard ports or IPs outside the target's primary IP space.

```text
Recon Chain
────────────────────────────────────────────────────────────────────────
Stage 2 (Semi-Passive)
WAF/CDN fingerprinting  →  [VHost & Favicon Hunts]  →  Hidden app discovery
Origin IP discovery                                   → Additional attack surface
Subdomain enumeration                                 → Phase 3 web testing
                            ↑ YOU ARE HERE
────────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** Subdomain enumeration finds names registered in DNS or CT logs. But organizations frequently run web applications that are intentionally not in DNS — internal admin panels, staging environments, or microservices served on the same IP as public applications but gated by a hostname that was never registered. VHost brute-forcing finds these hidden applications. Favicon hunting finds deployments that exist on entirely different IPs — test environments in cloud accounts, development boxes, and instances left running after migration.

This is the final note in Stage 2 and directly feeds Phase 3 web application assessment. Every application discovered here is a target for Phase 3 testing.

---

# Section 2 — How attackers actually use this

## 2.1 How virtual hosting works and why it creates hidden attack surface

When a web server receives an incoming HTTP/1.1 request, the `Host:` header tells the server which site the client is requesting. A single server at one IP can host dozens of applications — each responding only to its designated hostname:

```text
IP: 203.0.113.45

GET / HTTP/1.1
Host: corp-target.com          → Main marketing site
Host: admin.corp-target.com    → Admin panel (no DNS entry)
Host: api-internal.corp-target.com → Internal API (no DNS entry)
Host: staging.corp-target.com  → Staging environment
Host: jenkins                  → Jenkins CI (host-only, no FQDN)
```

From DNS enumeration you found `corp-target.com` resolves to `203.0.113.45`. But the server at that IP may serve four more applications that never appeared in DNS because no DNS record was created — they are accessed only by developers who know the hostname, configured in `/etc/hosts`, or reached via internal routing. VHost brute-forcing tests a wordlist of hostnames against the server IP to find which ones return a distinct response.

## 2.2 VHost brute-force mechanics — differentiating responses

The technique: send HTTP requests to the target IP with varying `Host:` header values and identify which responses are different from the default response.

The challenge: a server configured to serve a default vhost for unknown hostnames will return the same response for any unknown `Host:` value. You must differentiate:

- **Default/catch-all response:** same content for any unknown hostname — not a valid vhost
- **Valid vhost response:** different status code, content length, headers, or body — a real application

Filters to use:
```
Status code: valid vhosts may return 200, 301, 302, 401, 403 — all different from the default
Response size: different content-length indicates different content
Response headers: Server: header or Set-Cookie: may differ
Title: different `<title>` in HTML body
```

The default response is your baseline. Any response that deviates — different status, different size, different title — is a candidate vhost.

## 2.3 VHost hunting with ffuf and gobuster

**ffuf** is the primary vhost brute-force tool. It sends requests with `Host: FUZZ.<target>` (or custom host patterns) and filters by response size, status code, or word count to isolate valid vhosts from default responses.

```bash
# Basic vhost brute-force
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
     -u https://203.0.113.45/ \
     -H "Host: FUZZ.corp-target.com" \
     -t 50 -mc 200,301,302,401,403 -o ffuf_vhosts.json

# With response size filter (filter out default page)
ffuf -w wordlist.txt \
     -u https://203.0.113.45/ \
     -H "Host: FUZZ.corp-target.com" \
     -fs 1234      # filter: ignore responses with this exact size (default page size)
```

**gobuster** in vhost mode:
```bash
gobuster vhost -u https://corp-target.com \
               -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
               --append-domain
```

The `--append-domain` flag constructs `WORD.corp-target.com` rather than using `WORD` as the full hostname.

## 2.4 Favicon hash hunting for hidden application instances

The favicon hunt technique was introduced in note 05. In the context of vhost/application hunting, it serves a complementary purpose: finding *deployments* of the same application on different IPs, not just the primary production instance.

Workflow:
```text
Step 1: Retrieve favicon.ico from known application endpoint
Step 2: Calculate murmur3 hash
Step 3: Search Shodan: http.favicon.hash:<value>
Step 4: Review results — IPs in target ASN not in primary DNS = hidden instances
Step 5: For each found IP, test with application-specific Host: headers
Step 6: Directly access found IPs on all open ports found in Shodan
```

Use cases:
- **Cloud test environments:** A developer stands up an EC2 instance with the same application for testing. It has the same favicon but lives at a different IP that DNS never pointed to.
- **Old application instances:** After migration, the old server is still running. DNS was updated to the new IP, but the old server still serves the application to anyone who knows the IP or finds it via favicon search.
- **Non-standard port deployments:** The same application running on port 8080 or 8443 on a non-primary IP, accessible directly without the CDN.

## 2.5 HTTP response differentiation for vhost detection

Beyond status code and response size, a valid vhost often reveals itself through:

**`<title>` tag differences:**
```
Default response: <title>Welcome to nginx!</title>
Valid vhost:      <title>Jenkins CI</title>
```

**Cookie names:**
```
Default: no Set-Cookie
Valid vhost: Set-Cookie: JSESSIONID=...; Path=/
```

**Redirect destination:**
```
Default: 302 → /
Valid vhost: 302 → /dashboard/login?from=/
```

**Authentication challenges:**
```
Default: 200 with HTML content
Valid vhost: 401 with WWW-Authenticate: Basic realm="Jenkins"
```
A 401 response to a vhost probe is a high-value finding even without knowing the credentials — it confirms an application is present and protected by basic auth, which is worth attacking separately.

## 2.6 SNI vs Host header — HTTPS vhost considerations

For HTTPS targets, there are two places hostname matters: the TLS ServerName Indication (SNI) extension in the TLS ClientHello, and the HTTP `Host:` header in the request body.

SNI tells the server which certificate to present before the HTTP request is sent. If you connect to `203.0.113.45:443` with SNI `admin.corp-target.com`, you get the certificate for `admin.corp-target.com` (if the server is configured to serve it) before you even send an HTTP request.

For vhost brute-forcing on HTTPS, both must be set correctly:
- `curl --resolve admin.corp-target.com:443:203.0.113.45 https://admin.corp-target.com/`
- ffuf handles this automatically when you set both `-u https://<IP>/` and `-H "Host: FUZZ.corp-target.com"`

If SNI does not match, many servers return a default certificate and a default response — your vhost probe gets the wrong response even though the vhost exists.

## 2.7 Dead-end vs high-value finding

**Dead-end:** ffuf tests 5000 words against `203.0.113.45`. All responses return status 200 with the same content length (the default site). No variations. The server has a catch-all vhost that serves the same content regardless of hostname. Try direct IP access: `curl https://203.0.113.45/` with no Host header — if it returns the same default, there is no hidden vhost differentiation available. Move on.

**High-value:** ffuf returns `jenkins.corp-target.com` with status 200 and content length 8924 — different from the default response of 1200. `curl -H "Host: jenkins.corp-target.com" https://203.0.113.45/` returns:
```html
<title>Dashboard [Jenkins]</title>
<a href="/login">log in</a>
```
Jenkins CI is running on the same IP as the main site, reachable only via the correct Host header, with no DNS record and no CT log entry. Favicon hash search additionally reveals `203.0.113.100` serving the same Jenkins favicon on port 8080 — a second Jenkins instance, likely the development one, accessible directly without any Host header manipulation.

## 2.8 Where results feed next

Every application discovered in this step is an independent attack surface entry point for Phase 3. Jenkins → unauthenticated groovy script console or weak credential attack. Admin panels → default credentials or authentication bypass. Internal APIs → authorization testing, IDOR, injection. Old application instances → outdated software CVEs. The vhost and favicon hunt is the last recon step before web application testing begins.

---

## 2.9 Wordlist selection strategy for vhost hunting

Vhost brute-forcing is only as effective as its wordlist. The wordlist must reflect how the organization names its internal services — a generic subdomain list optimized for DNS enumeration performs poorly for vhost hunting because the naming conventions differ.

**Effective wordlist sources:**

| Source | Content | Best for |
|--------|---------|----------|
| `SecLists/Discovery/DNS/subdomains-top1million-5000.txt` | Top subdomain names from public DNS | Standard FQDN vhosts |
| `SecLists/Discovery/DNS/bitquark-subdomains-top100000.txt` | 100K curated subdomain names | Thorough FQDN pass |
| `SecLists/Discovery/Web-Content/raft-large-words.txt` | Common web words | Bare hostname vhosts |
| Custom wordlist from OSINT | Names from LinkedIn job listings, GitHub repos, Shodan banners | Organization-specific |
| Tool names list | `jenkins`, `grafana`, `kibana`, `gitlab`, `jira`, `confluence`, `vault`, `consul`, `sonarqube`, `harbor`, `nexus`, `traefik`, `rabbitmq`, `redis`, `mongo`, `elastic` | Internal DevOps tools |

**Custom wordlist construction:** Pull organization-specific terms from job listings (technologies mentioned in DevOps roles), GitHub repository names, and banner content from Shodan. If a company's Shodan results show `Server: Grafana`, add `grafana`, `grafana-internal`, `monitoring`, `metrics`, `dashboards` to the wordlist.

```bash
# Build a combined wordlist for a target
$ cat /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
      /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
      custom_tools.txt \
  | sort -u > combined_vhost_wordlist.txt

# custom_tools.txt — internal DevOps and security tools:
cat << 'EOF' > custom_tools.txt
jenkins ci build deploy grafana kibana elasticsearch logstash
prometheus alertmanager vault consul nomad traefik
sonarqube nexus harbor jira confluence gitlab gitea
rabbitmq redis mongo postgres pgadmin
admin portal manage dashboard internal dev staging test qa uat
api api-v2 api-internal graphql grpc
vpn remote rdp bastion jump
mail smtp imap owa webmail
backup archive legacy old
EOF
```

**Permutation-based wordlist generation with alterx:** If you already know some vhost names from ffuf or DNS enumeration, alterx generates permutations that follow the same naming pattern. If you found `api.corp-target.com`, alterx generates `api-v2`, `api-prod`, `api-staging`, `api-internal`, `api-dev` — covering the entire naming family.

```bash
# Generate permutations from known vhosts
$ echo "api.corp-target.com" | alterx -enrich | dnsx -a -resp-only -silent
api-v2.corp-target.com → 203.0.113.48     ← new discovery
api-internal.corp-target.com → 203.0.113.47
```

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Virtual host (vhost)** | A web application served from a shared IP, differentiated from other applications on the same IP by the HTTP `Host:` header value |
| **Host header** | HTTP/1.1 required header specifying the hostname the client is addressing; used by servers to route to the correct vhost |
| **Default vhost** | The catch-all application served when no vhost matches the requested `Host:` header; often a default nginx or Apache page |
| **SNI (Server Name Indication)** | TLS extension specifying the target hostname in the ClientHello; tells the server which certificate to present before HTTP begins |
| **Vhost brute-forcing** | Systematically sending requests with different `Host:` header values to find applications not visible in DNS |
| **Favicon hash** | Murmur3 hash of `/favicon.ico` bytes; same application produces same hash across all instances; Shodan-searchable |
| **Murmur3** | Non-cryptographic hash function; Shodan specifically uses it for favicon fingerprinting |
| **Response differentiation** | Identifying valid vhosts by comparing their response (status, size, title, headers) to the known default response |
| **False positive** | A vhost brute-force result where the server returned a "different" response but it is the same generic error or redirect, not a real application |
| **Catch-all vhost** | Server configured to serve the same default response for any unrecognized hostname; makes vhost brute-forcing return no useful results |
| **HTTP parameter pollution (HPP)** | Sending multiple parameters with the same name; some WAFs and backends process only one, enabling bypass |
| **Direct IP access** | Accessing a server by its IP address without sending a `Host:` header or with `Host: <IP>`; reveals the default vhost application |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `ffuf` | `ffuf -w wordlist.txt -u https://<IP>/ -H "Host: FUZZ.<domain>" -mc 200,301,302,401,403` | Vhosts returning non-default status codes | Primary vhost brute-force |
| `ffuf` | `ffuf -w wordlist.txt -u https://<IP>/ -H "Host: FUZZ.<domain>" -fs <default_size>` | Vhosts with different response body size | Filter by size to reduce false positives |
| `gobuster` | `gobuster vhost -u https://<target> -w wordlist.txt --append-domain` | Vhosts using gobuster's vhost mode | Alternative to ffuf |
| `curl` | `curl -sk -H "Host: <vhost>" https://<IP>/ -I` | Manual vhost probe — response headers | Confirm and investigate ffuf hits |
| `curl` | `curl -sk --resolve <vhost>:<port>:<IP> https://<vhost>/` | Full TLS-correct vhost probe with SNI | HTTPS vhost confirmation |
| Python `mmh3` | `python3 -c "import requests,mmh3,base64; ..."` | Calculate favicon murmur3 hash | Favicon-based application discovery |
| Shodan | `http.favicon.hash:<value> org:"Target Corp"` | All target org IPs serving this favicon | Find hidden application instances |
| Shodan | `http.favicon.hash:<value>` | All IPs worldwide serving this favicon | Cross-environment instance discovery |
| `httpx` | `cat vhosts.txt \| httpx -title -status-code -tech-detect` | Title, status, and technology stack for each found vhost | Rapid profiling of discovered vhosts |

**Installation:**
```bash
go install github.com/ffuf/ffuf/v2@latest
go install github.com/OJ/gobuster/v3@latest
go install github.com/projectdiscovery/httpx/cmd/httpx@latest
pip install mmh3 requests
```

**Wordlists:** `/usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt` for quick pass; `bitquark-subdomains-top100000.txt` for thorough pass.

**Establishing the baseline response:**
```bash
# Get default response size first
$ curl -sk -H "Host: doesnotexist.corp-target.com" https://203.0.113.45/ | wc -c
1234
# → Filter all responses of size 1234 in ffuf: -fs 1234
```

**ffuf vhost brute-force:**
```bash
$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
       -u https://203.0.113.45/ \
       -H "Host: FUZZ.corp-target.com" \
       -fs 1234 \
       -t 50 \
       -mc 200,204,301,302,307,401,403,405 \
       -o ffuf_results.json

jenkins.corp-target.com   [Status: 200, Size: 8924, Words: 312, Lines: 45]
admin.corp-target.com     [Status: 302, Size: 0, Redirect: /admin/login]
api-internal.corp-target.com [Status: 401, Size: 89, Words: 5]
staging.corp-target.com   [Status: 200, Size: 45892, Words: 1834]
```
Four valid vhosts found. `api-internal` returns 401 — application is present, protected by auth. `jenkins` is a full Jenkins instance. `staging` is the largest response — likely a development copy of the main app.

**Confirming a hit with curl:**
```bash
$ curl -sk -H "Host: jenkins.corp-target.com" https://203.0.113.45/ | grep -i title
<title>Dashboard [Jenkins]</title>

$ curl -sk -H "Host: jenkins.corp-target.com" https://203.0.113.45/script 2>&1 | grep -i "script console\|403\|login"
<h1>Script Console</h1>   ← Unauthenticated script console accessible
```
Jenkins script console is reachable without authentication — arbitrary Groovy code execution. Maximum severity.

**Favicon hash calculation and search:**
```bash
$ python3 << 'EOF'
import requests, mmh3, base64, sys

url = "https://corp-target.com/favicon.ico"
r = requests.get(url, timeout=10, verify=False)
favicon_hash = mmh3.hash(base64.encodebytes(r.content))
print(f"Hash: {favicon_hash}")
print(f"Shodan: http.favicon.hash:{favicon_hash}")
EOF
Hash: 116323821
Shodan: http.favicon.hash:116323821

$ shodan search 'http.favicon.hash:116323821' --fields ip_str,port,org,http.title
203.0.113.45   443  Target Corp    Corp-Target | Home
203.0.113.45   80   Target Corp    Corp-Target | Home
203.0.113.100  8080 Target Corp    Corp-Target Staging Environment  ← new
198.51.100.77  80   DigitalOcean   Corp-Target Dev                  ← new, different cloud
```
Two hidden instances discovered: a staging server in the same ASN on port 8080, and a development instance on DigitalOcean — entirely separate from the production infrastructure.

**Profiling discovered vhosts with httpx:**
```bash
$ cat << 'EOF' | httpx -title -status-code -tech-detect -silent
http://203.0.113.100:8080
https://203.0.113.45
https://203.0.113.45 -H "Host: jenkins.corp-target.com"
EOF
http://203.0.113.100:8080 [200] [Corp-Target Staging] [Apache][PHP/7.2][WordPress/5.8]
https://203.0.113.45 [200] [Corp-Target | Home] [nginx][React]
```
Staging is running WordPress 5.8 (outdated) on PHP 7.2 (EOL). Production is nginx + React.

---

**httpx mass vhost profiling — complete pipeline:**
```bash
# Step 1: Generate vhost candidates from ffuf results + DNS enum
$ cat ffuf_results.json | jq -r '.results[].input.FUZZ' \
  | awk '{print $1".corp-target.com"}' > vhost_candidates.txt

# Step 2: Add the origin IP as the actual address for all candidates
# (httpx will connect to this IP for each Host header)
$ cat vhost_candidates.txt | while read vhost; do
    echo "http://203.0.113.45 -H Host: ${vhost}"
  done | httpx -title -status-code -tech-detect -content-length -silent | tee vhost_profiles.txt

# Cleaner approach: use httpx's vhost mode directly
$ cat vhost_candidates.txt | httpx \
  -H "Host: {hostname}" \
  -target 203.0.113.45 \
  -title -status-code -tech-detect -content-length -location \
  -silent -o vhost_profiles.txt

# Step 3: Review by technology
$ cat vhost_profiles.txt
http://203.0.113.45 [200] [8924] [Jenkins CI] [Java] [Jetty]          jenkins.corp-target.com
http://203.0.113.45 [200] [45892] [WordPress 5.8] [Apache/2.4][PHP]   staging.corp-target.com
http://203.0.113.45 [401] [89] []                                      api-internal.corp-target.com
http://203.0.113.45 [302] [0] [] [→/dashboard/login]                  admin.corp-target.com
```

**Rapid technology-based triage of vhost results:**
```bash
# Flag Jenkins instances (script console RCE vector)
$ grep -i jenkins vhost_profiles.txt

# Flag WordPress instances (plugin vulnerability surface)
$ grep -i wordpress vhost_profiles.txt

# Flag 401/403 responses (access-controlled apps, high priority)
$ grep '\[40[13]\]' vhost_profiles.txt

# Flag redirect chains (may reveal underlying application technology)
$ grep -oP '\[.*?→.*?\]' vhost_profiles.txt

# Flag content with large body size (feature-rich apps, more attack surface)
$ awk -F'\[' '{print $3" "$0}' vhost_profiles.txt | sort -rn | head -5

# Technology summary across all discovered vhosts
$ grep -oP '\[.*?\]' vhost_profiles.txt | sort | uniq -c | sort -rn
  3 [Apache]
  2 [PHP]
  1 [Jenkins CI]
  1 [WordPress 5.8]
  1 [Jetty]
```
The technology summary shows 3 Apache-fronted apps (all require WAF/Apache version check), 1 Jenkins (highest priority — check for script console access), 1 WordPress 5.8 (outdated, check plugins), and 1 Jetty (Java app server — check for deserialization vectors).

**Complete vhost-to-attack-surface pipeline (one-liner):**
```bash
$ ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
       -u https://203.0.113.45/ \
       -H "Host: FUZZ.corp-target.com" \
       -fs $(curl -sk -H "Host: x123.corp-target.com" https://203.0.113.45/ | wc -c) \
       -mc 200,204,301,302,307,401,403 -t 30 -silent \
       -o ffuf.json && \
  cat ffuf.json | jq -r '.results[].input.FUZZ + ".corp-target.com"' | \
  httpx -title -status-code -tech-detect -silent -o vhost_profiles.txt && \
  echo "[*] Discovered vhosts:" && cat vhost_profiles.txt
```
This single command: establishes baseline size, runs ffuf, filters false positives, pipes all hits through httpx for tech detection, and outputs a final profile — the complete vhost discovery pipeline in one invocation.

# Section 5 — Defender detection

VHost brute-forcing sends HTTP requests with rapid-fire `Host:` header variations to the same IP. This is semi-active and generates access log entries on the target's web server.

- Every ffuf request produces an access log line on the target's web server with your source IP, the timestamp, the requested URL, the `Host:` header value, and the response code.
- A burst of hundreds of requests with different `Host:` headers from one IP within seconds is detectable. WAF rate-limiting rules can match this pattern and block the source IP. IDS rules (Snort/Suricata) have signatures for vhost brute-force patterns.
- **gobuster** and **ffuf** have detectable request patterns: consistent User-Agent, identical URL paths with only the Host header varying, high request rate.
- **Mitigation:** Set a realistic User-Agent (`-H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."`), reduce concurrency (`-t 10` instead of `-t 100`), and add a random delay between requests (`-p 0.1`).
- **Favicon hash Shodan search** is fully passive — no connection to the target.
- **Direct curl confirmation** of individual hits is one request — indistinguishable from normal web traffic.
- **SNI-based detection:** Some IDS systems fingerprint TLS ClientHellos where SNI does not match the DNS-resolved hostname — this pattern is unusual for legitimate browsers and can flag vhost probe traffic.

---

# Section 6 — Lab task

**Platform:** TryHackMe — *"Virtual Host Routing"* or *"Ffuf"* room. Alternatively: set up a local Nginx with multiple vhosts in a Kali VM to practice the complete workflow.

**Local lab setup:**
```bash
# Install nginx
sudo apt install nginx -y

# Create two vhosts
sudo mkdir -p /var/www/main /var/www/admin /var/www/api
echo "<h1>Main Site</h1>" | sudo tee /var/www/main/index.html
echo "<h1>Admin Panel</h1>" | sudo tee /var/www/admin/index.html
echo '{"status": "internal"}' | sudo tee /var/www/api/index.json

# Configure nginx vhosts
sudo bash -c 'cat > /etc/nginx/sites-available/lab << EOF
server { listen 80 default_server; root /var/www/main; }
server { listen 80; server_name admin.lab.local; root /var/www/admin; }
server { listen 80; server_name api.lab.local; root /var/www/api; }
EOF'
sudo ln -s /etc/nginx/sites-available/lab /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

**Objective:** Discover all vhosts on the local nginx server using ffuf, confirm each hit manually with curl, calculate a favicon hash for one of the applications, and produce a vhost inventory.

**Steps:**

1. Establish the baseline response size: `curl -s -H "Host: doesnotexist.lab.local" http://127.0.0.1/ | wc -c`
2. Run ffuf against the local server: `ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -u http://127.0.0.1/ -H "Host: FUZZ.lab.local" -fs <baseline_size> -mc 200,301,302,401,403 -t 20 -o ffuf_results.json`
3. Review ffuf results: `cat ffuf_results.json | jq '.results[] | {host: .input.FUZZ, status: .status, size: .length}'`
4. Confirm each hit manually: `curl -s -H "Host: admin.lab.local" http://127.0.0.1/ | grep -i title`
5. Check for direct IP access (default vhost): `curl -s http://127.0.0.1/ | grep -i title`
6. Create a test favicon: `wget https://www.google.com/favicon.ico -O /var/www/admin/favicon.ico`
7. Calculate the favicon hash: `python3 -c "import requests,mmh3,base64; r=requests.get('http://127.0.0.1/favicon.ico',headers={'Host':'admin.lab.local'}); print(mmh3.hash(base64.encodebytes(r.content)))"`
8. Test the HTTPS vhost with correct SNI: `curl -sk --resolve admin.lab.local:443:127.0.0.1 https://admin.lab.local/ -I` (if TLS is configured)
9. Profile all discovered vhosts: `cat discovered_vhosts.txt | httpx -title -status-code -tech-detect`
10. Produce `vhost_inventory.md`: Vhost | IP:Port | Status | Title | Technology | Risk | Notes

**Expected output:** ffuf output identifying at least 2 vhosts (admin, api), manual curl confirmation for each, favicon hash calculated and verified, and `vhost_inventory.md` with complete inventory.

**Git artifact:**
```
recon/stage2/vhost-favicon-hunts/
├── ffuf_results.json
├── favicon_hash.txt
├── curl_confirmations.txt
└── vhost_inventory.md
```
```bash
git commit -m "recon(stage2): vhost + favicon hunts — hidden app discovery for <target>"
```

---

# Section 7 — Common mistakes

**1. Not establishing the baseline response size before running ffuf**
_Why it matters:_ Without knowing the default response size, ffuf has no filter baseline and returns thousands of results — every request that gets *any* response. This produces an unworkable list where valid hits are buried in noise.
_Fix:_ Always send one probe with a random non-existent hostname first: `curl -s -H "Host: xyzabc12345.target.com" http://<IP>/ | wc -c`. Use the result as the `-fs` filter value.

**2. Ignoring 401 and 403 responses as "not found"**
_Why it matters:_ A 401 or 403 response to a vhost probe is a confirmation that a vhost exists and is protected. It is arguably a higher-value finding than a 200 — it tells you there is an access-controlled application at that hostname. Many operators filter these out and miss the finding entirely.
_Fix:_ Always include `401` and `403` in the `-mc` filter. Investigate every non-200 hit to determine what application is behind it before deciding it is out of scope.

**3. Not filtering by response size — treating all non-404 responses as valid hits**
_Why it matters:_ Some servers return 200 for every hostname but with different body content (wildcard vhost that serves a generic message). Without size filtering, every probe appears to be a valid hit.
_Fix:_ Use both `-mc` (match by status) and `-fs` (filter by size). When the server uses a wildcard 200 response, filter the size of that default 200. Check response word count (`-fw`) as an additional discriminator.

**4. Not testing the SNI separately from the Host header on HTTPS targets**
_Why it matters:_ On HTTPS, if the SNI does not match the vhost, the server presents the wrong certificate and may serve the wrong application or return a TLS error. A vhost that exists may appear non-existent because SNI was set to the target IP rather than the vhost hostname.
_Fix:_ For HTTPS vhost confirmation, always use `curl --resolve <vhost>:443:<IP> https://<vhost>/` to set both SNI and `Host:` correctly. ffuf handles this automatically when given `https://<IP>/` and `-H "Host: FUZZ.<domain>"`.

**5. Using only DNS-based wordlists for vhost brute-forcing**
_Why it matters:_ Vhosts don't have to be DNS names — `jenkins`, `admin`, `monitoring`, `grafana`, `kibana`, `rabbitmq` are common internal hostnames accessed by developers via `/etc/hosts` that will never appear in a DNS-based wordlist because they are not FQDN subdomains.
_Fix:_ Use two wordlists: a subdomain wordlist for FQDN-style probes (`FUZZ.corp-target.com`) and a service/tool name wordlist for bare hostname probes (`FUZZ` as the full Host value). SecLists has both.

**6. Trusting that favicon hash uniqueness guarantees application identity**
_Why it matters:_ Many applications use generic favicons (the default browser favicon, a stock icon set, or a framework default). A search for a generic favicon hash returns thousands of unrelated results — the hash is not unique to the target's application.
_Fix:_ Before running a Shodan favicon search, verify that the favicon is application-specific: open it in a browser and confirm it is the organization's branded icon, not a generic icon. If it is generic, the favicon hunt is not applicable for this target.

---

# Section 8 — Move-on gate

1. Set up a local nginx with two vhosts (main and admin), run ffuf against the local IP with correct baseline filtering, discover both vhosts, and confirm each with a manual curl command that correctly sets the `Host:` header — without referring to notes.

2. Given a target IP that returns identical `200 OK` responses for every `Host:` value you send, explain what this tells you about the server configuration, name two techniques you would use to still find hidden vhosts, and describe what response characteristic — beyond status code — you would use to differentiate them.

3. Calculate the favicon murmur3 hash of a live web application using the Python script from memory (without checking the code), run the Shodan search, and for each result returned: classify it as production CDN, origin server, or unrelated — based on the ASN and HTTP headers shown in the Shodan result.
