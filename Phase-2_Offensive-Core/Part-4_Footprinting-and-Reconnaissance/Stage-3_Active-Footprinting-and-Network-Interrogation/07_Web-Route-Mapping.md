# Web Route Mapping

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

Web route mapping is the final step of web reconnaissance before application testing begins. After discovering that specific paths exist (note 05) and confirming the TLS surface (note 06), route mapping builds the complete functional map of the application: how routes interconnect, what parameters each endpoint accepts, how the application handles authentication at the route level, and where hidden functionality lives in robots.txt, sitemap.xml, source maps, and parameter surfaces.

The output of route mapping is an attack surface document: a structured list of every reachable URL, its HTTP method, its parameter schema, its authentication state, and the technology handling it. Application testing in Phase 3 attacks this document directly.

```text
Stage 3 Final Web Recon Chain
────────────────────────────────────────────────────────────────────
Web Content Discovery  →  TLS Surface  →  [Web Route Mapping]
 /admin found              cert SANs         ↑ YOU ARE HERE
 tech stack known          cipher grades   robots.txt, params, JS
                                           Complete route inventory
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 robots.txt and sitemap.xml as route intelligence

`robots.txt` is intended to tell search engine crawlers which paths not to index. Its purpose is to disallow crawler access — which means every path listed is a path that exists. Operators deliberately list sensitive paths to prevent search engine indexing, inadvertently advertising them to attackers.

```bash
# Read robots.txt
$ curl -sk https://corp-target.com/robots.txt

User-agent: *
Disallow: /admin/
Disallow: /admin-backup/
Disallow: /internal/
Disallow: /staging/
Disallow: /api/v1/
Disallow: /wp-admin/
Disallow: /phpMyAdmin/         ← phpMyAdmin is present
Disallow: /old-portal/         ← legacy portal still accessible
Disallow: /*.sql$              ← SQL files exist on this server
Allow: /

# sitemap.xml reveals all public content the site wants indexed
$ curl -sk https://corp-target.com/sitemap.xml | xmllint --format - | grep "<loc>"
<loc>https://corp-target.com/</loc>
<loc>https://corp-target.com/about/</loc>
<loc>https://corp-target.com/blog/</loc>
<loc>https://corp-target.com/products/api-docs/</loc>   ← public API docs
<loc>https://corp-target.com/account/login</loc>

# Nested sitemaps (sitemap index)
$ curl -sk https://corp-target.com/sitemap_index.xml | grep "<loc>"
<loc>https://corp-target.com/page-sitemap.xml</loc>
<loc>https://corp-target.com/post-sitemap.xml</loc>
<loc>https://corp-target.com/api-sitemap.xml</loc>     ← dedicated API sitemap!

# Download and parse the API sitemap
$ curl -sk https://corp-target.com/api-sitemap.xml | grep "<loc>" | \
  sed 's/<loc>//;s/<\/loc>//' | sed 's/.*corp-target.com//'
/api/v1/products
/api/v1/users
/api/v1/orders
/api/v1/admin/reports    ← admin API route in public sitemap!
```

Every `Disallow:` in robots.txt is a direct hint. `/phpMyAdmin/` confirms phpMyAdmin is installed. `/old-portal/` suggests a legacy application. `/staging/` means a staging environment is accessible at the same domain root.

## 2.2 Parameter fuzzing for hidden parameter surfaces

Routes frequently have hidden parameters — GET parameters, POST body fields, and HTTP headers that alter application behavior but are not documented or visible in the UI. Finding these parameters is the difference between "this endpoint returns user data" and "this endpoint returns ALL users' data when you add `?admin=true`."

```bash
# arjun — automated parameter discovery
$ arjun -u https://corp-target.com/api/v1/users
[+] Scanning for GET parameters...
[+] Found: id      (param changes response size significantly)
[+] Found: limit   (param changes response size)
[+] Found: debug   (param returns different response status)    ← debug mode param!

# Manual GET parameter fuzzing with ffuf
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
       -u "https://corp-target.com/api/v1/users?FUZZ=test" \
       -mc 200 \
       -fw 45 \    # filter baseline word count for GET with no params
       -t 20

# POST parameter fuzzing
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
       -u "https://corp-target.com/api/v1/auth/login" \
       -X POST \
       -H "Content-Type: application/x-www-form-urlencoded" \
       -d "FUZZ=test" \
       -mc 200,302,400,422 \
       -fw 20 -t 20

# HTTP header parameter fuzzing (X-* headers)
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt \
       -u "https://corp-target.com/api/v1/users" \
       -H "X-FUZZ: test" \
       -mc 200 -fw 45 -t 20
# Looking for: X-Admin: true, X-Debug: 1, X-Internal: true
```

The `debug` parameter returning a different status code means the endpoint has a hidden debug mode. `?debug=true` on an API endpoint frequently enables verbose error messages with stack traces, database query details, and internal hostnames.

## 2.3 Status and size-based route triage

Not all discovered paths require equal attention. The combination of HTTP status code and response size is the primary triage signal:

```bash
# Run ffuf with size and word count output
$ ffuf -w wordlist.txt -u https://corp-target.com/FUZZ \
       -mc 200,301,302,403,401,405 -t 30 \
       -o routes.json -of json

# Triage by status code priority
$ cat routes.json | jq -r '.results[] | "\(.status) \(.length)B \(.url)"' | sort -n

# HIGH PRIORITY - 200 responses
200 48921B https://corp-target.com/admin/            ← admin panel accessible (no auth!)
200   312B https://corp-target.com/.env              ← credentials file!
200  8921B https://corp-target.com/backup/

# MEDIUM PRIORITY - 401/403 (exists but restricted)
403   489B https://corp-target.com/internal/         ← forbidden but exists
401   112B https://corp-target.com/api/v1/admin/     ← auth required

# INVESTIGATE - 405 Method Not Allowed (endpoint exists, wrong method)
405    89B https://corp-target.com/api/v1/upload/    ← try POST instead of GET

# Route method fuzzing (find accepted HTTP methods)
$ for method in GET POST PUT PATCH DELETE OPTIONS HEAD TRACE; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" -X $method https://corp-target.com/api/v1/users)
    echo "$method → $code"
  done

GET    → 200   ← read
POST   → 201   ← create (also works!)
PUT    → 405   ← not allowed
DELETE → 403   ← forbidden but recognized
OPTIONS → 200  ← check Allow header in response
```

A 403 Forbidden is not a dead end — it means the route exists and requires authorization. A 405 Method Not Allowed means the route exists but you are using the wrong HTTP method. Try the methods listed in the `Allow:` response header.

## 2.4 JavaScript source maps and bundle analysis

Production JavaScript bundles are minified and obfuscated — readable code is transformed into a compact one-liner. But many applications inadvertently ship JavaScript source maps (`.map` files) alongside the minified bundle. Source maps reconstruct the original source code — giving you readable route definitions, API calls, and application logic.

```bash
# Check for source map reference in JS bundle
$ curl -sk https://corp-target.com/static/js/main.abc123.js | tail -1
//# sourceMappingURL=main.abc123.js.map    ← source map reference at end of file!

# Download the source map
$ curl -sk https://corp-target.com/static/js/main.abc123.js.map -o main.map

# Reconstruct original source from source map (using sourcemapper tool)
$ go install github.com/denandz/sourcemapper@latest
$ sourcemapper -output ./reconstructed -url https://corp-target.com/static/js/main.abc123.js.map

# Or manually read the source map JSON
$ cat main.map | jq '.sources[]'
"src/components/App.js"
"src/api/client.js"          ← contains all API calls!
"src/routes/index.js"        ← contains all route definitions
"src/admin/AdminPanel.js"    ← admin panel code!
"src/utils/config.js"        ← may contain API keys or endpoints

$ cat main.map | jq '.sourcesContent[1]'   # Read src/api/client.js content
"const API_BASE = 'https://api-internal.corp-target.com';\n..."   ← internal API URL!
```

A JavaScript source map exposes the original, readable source code of the entire frontend application. This includes: internal API endpoint URLs not accessible from the public site, authentication token handling logic (showing how to bypass), admin functionality visible only in the source, and sometimes API keys or configuration values included in the frontend code.

## 2.5 Web crawling with hakrawler and katana

Automated crawlers follow links in HTML, JavaScript, and form actions to discover routes that no wordlist would generate. Unlike brute-forcing, crawling only finds paths the application actually links to — which means it reflects the real application flow.

```bash
# hakrawler — multi-threaded web spider
$ echo "https://corp-target.com" | hakrawler -d 3 -plain -t 10 | tee crawl_results.txt

https://corp-target.com/
https://corp-target.com/about
https://corp-target.com/login
https://corp-target.com/api/v1/products
https://corp-target.com/api/v1/products?category=electronics
https://corp-target.com/user/profile?id=1                  ← IDOR parameter visible!
https://corp-target.com/admin/dashboard                     ← admin link found in source!
https://corp-target.com/static/js/main.abc123.js.map       ← source map found!

# Katana — advanced crawler with JS rendering
$ katana -u https://corp-target.com -d 3 -jc -js-crawl -o katana_results.txt
# -jc: crawl JavaScript files
# -js-crawl: extract endpoints from JS

# Combine brute-force + crawl results
$ cat ffuf_dirs.json | jq -r '.results[].url' > brute_urls.txt
$ sort -u brute_urls.txt crawl_results.txt > all_routes.txt
$ wc -l all_routes.txt
387 unique routes discovered
```

`/user/profile?id=1` discovered by crawling is an Insecure Direct Object Reference (IDOR) candidate — change the `id` parameter to `2`, `3`, `100` and see if you can access other users' profiles. This is one of the highest-frequency web vulnerabilities and crawling reliably finds these parameterized routes.

## 2.6 Authentication state mapping

For every discovered route, the authentication requirement must be determined — not assumed from the URL structure. Some routes that look public are actually protected; some that look protected are actually accessible unauthenticated.

```bash
# Test each discovered route without any authentication header
$ while read url; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" "$url")
    size=$(curl -sk -o /dev/null -w "%{size_download}" "$url")
    echo "$code \t $size \t $url"
  done < all_routes.txt | sort -n | tee auth_states.txt

# Classify by authentication state
200 48921 https://corp-target.com/admin/          ← 200 WITHOUT auth = broken auth!
200  8921 https://corp-target.com/api/v1/users    ← user list without auth = finding!
401   112 https://corp-target.com/api/v1/admin/   ← requires auth (correct)
403   489 https://corp-target.com/internal/       ← forbidden (exists but restricted)
302     0 https://corp-target.com/account/        ← redirects to login
```

An admin panel returning 200 without any authentication header is a critical finding: the admin panel has no authentication. `api/v1/users` returning a user list without authentication is a critical data exposure — all user accounts are readable by anyone.

## 2.7 Route structure inference and hidden API versions

Modern APIs are versioned (`/api/v1/`, `/api/v2/`). Active versions are documented; deprecated versions are often still running, less maintained, and more vulnerable. Old API versions may lack: rate limiting, authentication requirements, security patches, and WAF rules (the WAF may only inspect `/api/v2/` paths).

```bash
# API version discovery
$ for ver in v1 v2 v3 v4 v5 v6 v7 v8 v9 v10 beta alpha dev test old legacy; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" "https://corp-target.com/api/$ver/")
    [ "$code" != "404" ] && echo "[$code] /api/$ver/ ← EXISTS"
  done

[200] /api/v1/ ← EXISTS (current)
[200] /api/v2/ ← EXISTS (newer)
[200] /api/beta/ ← EXISTS (unreleased features!)
[403] /api/dev/ ← EXISTS but restricted (development endpoint!)

# Check if old API version lacks auth enforcement
$ curl -sk https://corp-target.com/api/v1/users         # → 401 (requires auth)
$ curl -sk https://corp-target.com/api/beta/users       # → 200 (beta lacks auth!)
```

The beta API returning 200 without authentication while the production API v1 returns 401 is a real finding: the beta API was deployed without the authentication middleware that protects the production version.

## 2.8 Final route inventory compilation

```bash
# Compile complete route inventory from all sources
$ cat \
    <(cat robots_disallows.txt)          `# robots.txt paths`\
    <(cat sitemap_urls.txt)              `# sitemap.xml URLs`\
    <(cat ffuf_dirs.json | jq -r '.results[].url') \   `# brute-force results`\
    <(cat crawl_results.txt)             `# crawler results`\
    <(cat param_discovery.txt)           `# arjun parameter output`\
  | sort -u \
  | grep "corp-target.com" \
  > final_route_inventory.txt

$ wc -l final_route_inventory.txt
623 unique routes

# Create attack surface table
$ while read url; do
    method="GET"
    code=$(curl -sk -X $method -o /dev/null -w "%{http_code}" "$url")
    auth=$([ "$code" = "401" ] || [ "$code" = "403" ] && echo "protected" || echo "open")
    echo "$code | $method | $auth | $url"
  done < final_route_inventory.txt | tee attack_surface.txt
```

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **robots.txt** | File telling search engine crawlers which paths to exclude; inadvertently reveals sensitive paths |
| **sitemap.xml** | XML file listing all publicly intended URLs; often includes API paths and admin sections |
| **Parameter fuzzing** | Sending requests with different parameter names to discover hidden input fields that alter behavior |
| **IDOR (Insecure Direct Object Reference)** | A vulnerability where changing an object ID parameter (`?id=1` → `?id=2`) accesses another user's data |
| **arjun** | Automated tool for parameter discovery — finds hidden GET/POST parameters by comparing response differences |
| **hakrawler** | Fast web crawler that follows links and form actions to discover application routes |
| **katana** | Advanced web crawler with JavaScript rendering support; finds routes dynamically generated by JS |
| **Source map** | JavaScript file mapping minified code back to original source — reveals application logic and endpoints |
| **sourcemapper** | Tool that reconstructs original source code from JavaScript source maps |
| **HTTP method fuzzing** | Testing all HTTP methods (GET/POST/PUT/DELETE/PATCH/OPTIONS) against a route to find accepted methods |
| **Authentication state** | Whether a route returns data with or without authentication credentials — mapped by unauthenticated probing |
| **API versioning** | Practice of exposing multiple API versions simultaneously; old versions may lack modern security controls |
| **405 Method Not Allowed** | HTTP response indicating the route exists but the used HTTP method is not accepted — try other methods |
| **Burp Suite Intruder** | Burp's fuzzing module for parameter and path discovery in an authenticated browser session |
| **Route inventory** | A structured list of all discovered application routes with status, method, and authentication state |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `curl` | `curl -sk https://target/robots.txt` | Disallowed paths | Always first |
| `curl` | `curl -sk https://target/sitemap.xml \| xmllint --format -` | All intended URLs | Structured apps |
| `arjun` | `arjun -u https://target/endpoint` | Hidden GET/POST params | After route discovery |
| `ffuf` (params) | `ffuf -w params.txt -u https://target/endpoint?FUZZ=1 -fw 45` | GET parameters | Parameter fuzzing |
| `hakrawler` | `echo "https://target" \| hakrawler -d 3 -plain` | All linked routes | Spider the app |
| `katana` | `katana -u https://target -jc -js-crawl -d 3` | JS-rendered routes | Modern SPAs |
| `sourcemapper` | `sourcemapper -url https://target/main.js.map` | Original source code | Source map found |

**robots.txt + sitemap parsing pipeline:**
```bash
TARGET="https://corp-target.com"

# robots.txt
$ curl -sk $TARGET/robots.txt | grep "^Disallow:" | awk '{print $2}' | \
  while read path; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET$path")
    echo "[$code] $TARGET$path"
  done

# sitemap.xml (handle nested sitemaps)
$ curl -sk $TARGET/sitemap.xml \
  | xmllint --format - 2>/dev/null \
  | grep "<loc>" \
  | sed 's|.*<loc>||;s|</loc>.*||' > sitemap_urls.txt

# For nested sitemap index:
$ curl -sk $TARGET/sitemap.xml | grep -o 'https://[^<]*sitemap[^<]*.xml' \
  | while read smap; do
    curl -sk $smap | grep "<loc>" | sed 's|.*<loc>||;s|</loc>.*||'
  done >> sitemap_urls.txt
```

**arjun parameter discovery:**
```bash
# Install
$ pip install arjun

# GET parameter discovery
$ arjun -u https://corp-target.com/api/v1/users -m GET --stable
[*] Testing 598 parameters...
[+] Parameters found: id, limit, page, debug, format, export

# POST parameter discovery
$ arjun -u https://corp-target.com/api/v1/auth -m POST --stable
[+] Parameters found: username, password, remember_me, mfa_token, debug_bypass

# debug_bypass found → test: POST with debug_bypass=1 → may skip MFA
```

**HTTP method mapping:**
```bash
$ for route in $(cat all_routes.txt | head -20); do
    echo "=== $route ==="
    for m in GET POST PUT PATCH DELETE OPTIONS; do
        code=$(curl -sk -X $m -o /dev/null -w "%{http_code}" "$route")
        [ "$code" != "405" ] && [ "$code" != "000" ] && echo "  $m → $code"
    done
  done
```

---

# Section 5 — Defender detection

- **robots.txt and sitemap.xml requests:** These are two standard GET requests that appear in every web server's access log. They are indistinguishable from a legitimate search engine crawler's requests. Zero detection signal — request these freely.
- **Parameter fuzzing:** Sending thousands of requests with different parameter names creates a distinctive access log pattern: the same URL path repeated hundreds of times with different query string content. Web application firewalls detect parameter fuzzing by rate and pattern — identical URL with mutating query strings.
- **hakrawler/katana crawling:** Crawlers generate many GET requests in quick succession across many different paths. Unlike brute-forcing (guessing random paths), crawling follows real links — but the access log shows an automated agent (no CSS/image loading, consistent non-browser User-Agent, high request rate).
- **Source map access:** Requesting `.map` files appears in the access log. If the server returns them with 200 status, the developer has inadvertently deployed them — no attack, but the access is logged. If you repeatedly access source map content, the access log shows an unusual pattern of developer-tool-related requests.
- **Authentication state probing:** Sending requests to authenticated routes without credentials generates 401/403 responses. A burst of 401 responses across many different authenticated routes in the access log indicates systematic unauthenticated access probing. Some WAFs automatically block IPs generating too many 401 responses.
- **Mitigation for operators:** (1) Use a slow crawl rate with hakrawler: add a `--delay 500ms` equivalent. (2) Rotate User-Agent strings between requests. (3) Space parameter fuzzing across time: `ffuf -p 0.05` (50ms between requests). (4) Read static files (robots.txt, sitemap.xml) early and freely — they generate no detection signal.

---

# Section 6 — Lab task

**Platform:** TryHackMe "OWASP Top 10" room, DVWA, or any intentionally vulnerable web application with a multi-route structure. Alternatively: HackTheBox Machines with web-based attack vectors.

**Objective:** Build a complete route map of a target web application using robots.txt, sitemap, crawling, parameter fuzzing, and authentication state testing.

**Steps:**

1. **robots.txt:** `curl -sk http://target/robots.txt` — list all Disallow entries and visit each
2. **sitemap.xml:** `curl -sk http://target/sitemap.xml | xmllint --format - | grep "<loc>"` — extract all URLs
3. **Spider:** `echo "http://target" | hakrawler -d 3 -plain -t 5 | tee crawl.txt`
4. **Authentication state check:** For each discovered route, `curl -sk -o /dev/null -w "%{http_code}" <url>` — document 200/401/403/302
5. **Admin route identification:** `grep -iE "admin|manage|dashboard|control|internal" crawl.txt`
6. **Parameter fuzzing on one endpoint:** `arjun -u http://target/vulnerable-endpoint -m GET`
7. **HTTP method test:** Test GET/POST/PUT/DELETE against 3 interesting routes
8. **API version check:** `for v in v1 v2 v3 beta dev old; do curl -o /dev/null -w "$v: %{http_code}\n" -sk "http://target/api/$v/"; done`
9. **Source map check:** `curl -sk http://target/static/js/app.js | tail -1` — look for `sourceMappingURL`
10. **Compile `route_inventory.md`:** Table with URL | Method | Auth State | Notes — categorize each route by risk level (critical/high/medium/info)

```bash
git commit -m "recon(stage3): web route mapping — <N> routes mapped for <target>"
```

---

# Section 7 — Common mistakes

**1. Treating robots.txt paths as confirmed attack surface without visiting them**
_Why it matters:_ robots.txt lists paths the owner wants to hide from indexing, but some of those paths may have been deleted, moved, or may require authentication. Listing them is not the same as confirming they are accessible and exploitable.
_Fix:_ Visit every robots.txt path and check the HTTP status. Document: path | status | accessible | notes.

**2. Skipping parameter fuzzing on API endpoints**
_Why it matters:_ Hidden parameters are one of the most common ways to discover authorization flaws. `?admin=true` or `?debug=1` added to a public API endpoint sometimes grants elevated access or verbose output not available through the documented interface.
_Fix:_ Run arjun against every discovered API endpoint. Parameter discovery takes minutes and frequently finds high-value functionality.

**3. Not checking all HTTP methods against interesting routes**
_Why it matters:_ A route returning 404 for GET may return 200 for POST. An admin API endpoint returning 403 for PUT may accept DELETE and actually delete records. The HTTP method changes the access control evaluation in many frameworks.
_Fix:_ For any route that returns an interesting status code (401, 403, 405), always test all HTTP methods.

**4. Ignoring 401 and 403 responses as "access denied = nothing here"**
_Why it matters:_ 401 and 403 confirm the route exists. They are the starting point for authentication bypass and authorization testing — not the end. An IDOR vulnerability, an authentication bypass, or a parameter manipulation may turn a 401 into a 200.
_Fix:_ Document every 401 and 403 route separately from truly non-existent (404) routes. These are the authorized attack surface for Phase 3 exploitation.

**5. Not combining brute-force results with crawler results**
_Why it matters:_ Brute-forcing finds paths that the application doesn't link to. Crawling finds paths that the application does link to but the wordlist doesn't contain. Neither alone is complete. The union is.
_Fix:_ Always merge and deduplicate both source lists into `all_routes.txt` before the authentication state sweep.

**6. Missing API version enumeration**
_Why it matters:_ Deprecated API versions (v1 when v2 is current) are the most common location of missing authentication and authorization controls. They are actively maintained at the code level but rarely receive security patching after deprecation.
_Fix:_ Always enumerate API versions with a short version list (v1 through v10, plus beta/alpha/dev/test/legacy) against any discovered `/api/` path.

---

# Section 8 — Move-on gate

1. `robots.txt` for a target lists `Disallow: /internal-api/`. Without notes, describe: what this tells you about the application architecture, the first three checks you run against this path, and what constitutes a "critical finding" from this discovery vs. a "no additional risk" outcome.

2. arjun discovers a hidden `debug` parameter on `GET /api/v1/users?debug=true`. You call the endpoint with this parameter and receive a response 3x larger than without it, containing stack trace information and internal SQL queries. State what vulnerability class this is, its severity, and two attack scenarios it enables.

3. You discover `/api/v1/users` returns 401, but `/api/beta/users` returns 200 with a full user listing. Without notes, name this vulnerability class, explain why it occurs, and describe the two immediate next actions you would take to determine the full scope of the exposure.
