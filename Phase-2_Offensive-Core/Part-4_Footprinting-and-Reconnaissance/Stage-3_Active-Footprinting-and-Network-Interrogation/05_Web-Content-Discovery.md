# Web Content Discovery

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

Web content discovery is the active mapping of a web server's directory structure, hidden endpoints, API routes, and technology stack. Port scanning tells you port 443 is open running nginx. Web content discovery tells you what is actually running behind that port: the CMS version, hidden admin panels, exposed configuration files, API endpoint structure, and directory indexes. It is the bridge between "web server confirmed alive" and "web application attack surface fully mapped."

This is entirely active: every directory guess is an HTTP request that hits the target's web server and appears in its access logs. Unlike port scanning which can be partially disguised, web content discovery generates clean, easily identified human-readable request patterns in server logs.

```text
Stage 3 Web Recon Chain
────────────────────────────────────────────────────────────────────
Port Scan   →   [Web Content Discovery]   →   Web Route Mapping
443 open         ↑ YOU ARE HERE               (note 07)
nginx found    ffuf, gobuster, wappalyzer
               whatweb, nikto, hakrawler
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 Directory brute-forcing with ffuf

ffuf (Fuzz Faster U Fool) is the primary tool for directory and file brute-forcing. It sends HTTP requests with a wordlist replacing the `FUZZ` placeholder in a URL and filters responses by status code, response size, word count, or line count to surface real paths from 404 noise.

```bash
# Basic directory brute-force
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
       -u https://corp-target.com/FUZZ \
       -mc 200,301,302,403 \
       -t 40

# Common wordlists by scenario:
# General directory:   /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
# Common files:        /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
# API endpoints:       /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt
# Backup files:        /usr/share/seclists/Discovery/Web-Content/quickhits.txt
# PHP-specific:        /usr/share/seclists/Discovery/Web-Content/PHP.fuzz.txt

# With file extension fuzzing
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
       -u https://corp-target.com/FUZZ.FUZZ2 \
       -w2 /usr/share/seclists/Discovery/Web-Content/web-extensions.txt \
       -mc 200 -fc 404
```

**Filtering false positives:** The most important ffuf skill is filtering. A common problem is that the server returns a 200 response even for nonexistent paths (a custom 404 page with 200 status). The `-fs` (filter size), `-fw` (filter words), and `-fl` (filter lines) flags eliminate consistent false positives:

```bash
# Determine the baseline 404 size first
$ curl -s https://corp-target.com/nonexistentpath123 | wc -c
1842   ← 404 page is 1842 bytes

# Filter that exact size out of ffuf results
$ ffuf -w wordlist.txt -u https://corp-target.com/FUZZ \
       -mc 200,301,302,403 -fs 1842 -t 40
```

## 2.2 Gobuster for directory and DNS fuzzing

Gobuster is an alternative to ffuf for directory brute-forcing, with specialized modes for DNS subdomain brute-forcing and vhost enumeration. Slightly less flexible than ffuf but faster for simple directory scans.

```bash
# Directory mode
$ gobuster dir -u https://corp-target.com \
               -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
               -x php,html,txt,bak,old,zip \
               -t 50 -o gobuster_results.txt

# Results:
/admin                (Status: 301) [Size: 165] [--> /admin/]
/backup               (Status: 200) [Size: 489]      ← backup directory!
/config.php.bak       (Status: 200) [Size: 3421]     ← backup config file!
/wp-admin             (Status: 302) [Size: 0] [--> /wp-login.php]  ← WordPress
/uploads              (Status: 200) [Size: 856]
/.git                 (Status: 301) [Size: 165] [--> /.git/]   ← exposed git!

# DNS mode (subdomain brute-force — alternative to dnsx)
$ gobuster dns -d corp-target.com \
               -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
               -t 50

# Vhost mode
$ gobuster vhost -u https://203.0.113.45 \
                 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
                 --domain corp-target.com -t 50
```

A `.git` directory exposed at the web root is a critical finding. The entire git repository history — including deleted files, old configuration with credentials, and commit history — is downloadable via `.git/config`, `.git/HEAD`, and the git object database.

## 2.3 Technology fingerprinting with Wappalyzer and whatweb

Identifying the technology stack is necessary before choosing attack approaches. Wappalyzer identifies CMS, framework, JavaScript libraries, analytics, CDN, server software, and payment providers from HTTP headers, HTML source, and JavaScript file patterns. whatweb is the command-line equivalent.

```bash
# whatweb command-line fingerprinting
$ whatweb https://corp-target.com

https://corp-target.com [200 OK]
  WordPress[5.8.3]           ← CMS and version
  Apache[2.4.38]             ← web server
  PHP[7.4.3]                 ← scripting language and version
  JQuery[3.6.0]              ← JS library
  Google-Analytics-UA        ← analytics (tells you Google Tag Manager is present)
  X-Powered-By[PHP/7.4.3]    ← version disclosure header
  Country[UNITED STATES]
  IP[203.0.113.45]

# More verbose (aggressive fingerprinting)
$ whatweb -v -a 3 https://corp-target.com    # -a 3 = aggressive mode

# Multiple targets from file
$ whatweb -i alive_web.txt --log-json=whatweb_results.json
```

**What version information means operationally:**
- `WordPress 5.8.3` → look up CVEs for this version; check for vulnerable plugins
- `PHP 7.4.3` → PHP 7.x has known RCE CVEs; 7.4.3 specifically may have deserialization issues
- `Apache 2.4.38` → check for CVE-2021-41773 (path traversal, Apache 2.4.49+), CVE-2017-7679 (buffer overflow in older versions)
- `X-Powered-By: PHP/7.4.3` → version disclosure misconfiguration; should be removed

## 2.4 nikto for quick web vulnerability baseline

nikto is a web scanner that checks for common misconfigurations, dangerous files, outdated server versions, and headers issues. It is not a deep vulnerability scanner — it produces a quick baseline of obvious problems.

```bash
$ nikto -h https://corp-target.com -output nikto_report.txt

- Nikto v2.1.6
+ Target IP:          203.0.113.45
+ Target Hostname:    corp-target.com
+ Target Port:        443

+ Server: Apache/2.4.38 (Ubuntu)
+ /admin/: Admin login page/area found.
+ /backup/: Backup directory found.                      ← matches gobuster
+ /config.php.bak: Backup of a configuration file found.
+ /phpinfo.php: PHP info page found.                     ← CRITICAL
+ Server leaks inodes via ETags for '/': Apache.
+ X-Frame-Options header is not included in the HTTP response to protect against clickjacking.
+ Missing X-Content-Type-Options header.
+ Allowed HTTP Methods: GET, POST, OPTIONS, HEAD
+ OSVDB-3092: /.htaccess: htaccess file found.
```

`phpinfo.php` exposed is a critical finding: it dumps the entire PHP configuration including environment variables, loaded extensions, file paths, and sometimes API keys and database connection strings set as PHP environment variables.

## 2.5 API endpoint discovery

Modern web applications expose REST or GraphQL APIs. These are not discovered by standard directory wordlists — they require API-specific wordlists and response interpretation.

```bash
# API endpoint brute-force with API wordlist
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
       -u https://corp-target.com/api/FUZZ \
       -mc 200,201,204,400,401,403,405 \
       -t 30

# Common API paths to check manually
$ for path in v1 v2 v3 api rest graphql swagger openapi docs; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" https://corp-target.com/$path)
    echo "$code https://corp-target.com/$path"
  done

200 https://corp-target.com/api
200 https://corp-target.com/graphql    ← GraphQL endpoint found
403 https://corp-target.com/swagger    ← swagger UI access restricted
301 https://corp-target.com/v2         ← API versioning

# Swagger/OpenAPI spec — if exposed, reveals ALL endpoints
$ curl -sk https://corp-target.com/swagger.json | jq '.paths | keys[]'
/api/v1/users
/api/v1/admin/users
/api/v1/auth/login
/api/v1/admin/reset-password   ← admin endpoints!

# GraphQL introspection (if enabled, dumps complete schema)
$ curl -sk https://corp-target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ __schema { types { name } } }"}' | jq .
```

An exposed swagger.json or openapi.yaml spec reveals every API endpoint, expected parameters, authentication methods, and data models. This is equivalent to handing you the complete API documentation — every endpoint in the application is listed.

## 2.6 JavaScript file analysis for hidden paths

Modern single-page applications bundle all routes, API endpoints, and configuration into JavaScript files. These JavaScript bundles often contain internal API paths, authentication endpoints, staging environment URLs, and sometimes embedded credentials.

```bash
# Extract all JavaScript files from the page
$ curl -sk https://corp-target.com/ | grep -oP 'src="[^"]+\.js"' | sed 's/src="//;s/"//'
/static/js/main.a1b2c3d4.js
/static/js/vendor.5e6f7a8b.js

# Analyze main JavaScript bundle for interesting patterns
$ curl -sk https://corp-target.com/static/js/main.a1b2c3d4.js | \
  grep -oP '["'"'"'][/a-z0-9_-]{4,50}["'"'"']' | sort -u | head -50

"/api/v1/users"
"/api/v1/admin/dashboard"
"/api/v1/auth/token"
"/api/internal/metrics"        ← internal endpoint in production JS!
"https://staging.corp-target.com/api"   ← staging URL leaked!
"AIzaSyXXXXXXXXXXXXXXX"       ← Google API key!

# LinkFinder — extract endpoints from JS files
$ python3 LinkFinder/linkfinder.py -i https://corp-target.com/static/js/main.js -o cli

# hakrawler — spider the site and collect all links/endpoints
$ echo "https://corp-target.com" | hakrawler -d 3 -plain | tee crawl_results.txt
```

A staging URL embedded in production JavaScript is a common finding: `https://staging.corp-target.com` is often less secured (no WAF, debug mode, verbose errors) and sometimes the same codebase with full debug access. Internal API paths that don't appear in the public documentation are additional attack surface.

## 2.7 Sensitive file patterns to always check

A set of common file paths should be checked on every web server engagement:

```bash
$ for path in \
  robots.txt sitemap.xml .htaccess .htpasswd \
  web.config .env .env.local .env.backup \
  config.php config.php.bak wp-config.php \
  phpinfo.php info.php test.php \
  backup.zip backup.tar.gz db.sql database.sql \
  .git/HEAD .git/config .svn/entries \
  crossdomain.xml clientaccesspolicy.xml \
  actuator/env actuator/health server-status server-info \
  console/login admin/login administrator manager; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" "https://corp-target.com/${path}")
    [ "$code" != "404" ] && echo "[$code] https://corp-target.com/${path}"
  done
```

Notable findings from this checklist:
- `.env` file returning 200 is a critical finding — PHP/Laravel/Node applications store database passwords, API keys, and secret keys here
- `actuator/env` (Spring Boot) returns all environment variables including credentials
- `server-status` (Apache) returns the current request log — you can see other users' requests in real time
- `.git/config` returning 200 means the entire git history is downloadable

## 2.8 Recursive directory brute-forcing and depth strategy

An initial brute-force sweep at the root level (`/FUZZ`) discovers top-level directories. Each discovered directory is itself a namespace that may contain further hidden content. Recursive brute-forcing systematically extends the sweep to each discovered directory.

```bash
# Step 1: Root-level discovery
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
       -u https://corp-target.com/FUZZ \
       -mc 200,301,302,403 -fs 1842 -t 40 \
       -o root_dirs.json

# Root results:
/admin          → 301
/backup         → 200
/api            → 200
/uploads        → 200

# Step 2: Recurse into each discovered directory
$ for dir in admin backup api uploads; do
    echo "=== Scanning /$dir/ ==="
    ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
         -u "https://corp-target.com/$dir/FUZZ" \
         -mc 200,301,302,403 -fs 1842 -t 20 \
         -o "scan_${dir}.json" 2>/dev/null
  done

# /backup/ recursive results:
/backup/db_backup_2023.sql       → 200   ← database dump!
/backup/config.zip               → 200   ← config archive!
/backup/index.php                → 200

# /api/ recursive results:
/api/v1                          → 200
/api/v2                          → 200
/api/internal                    → 403   ← internal API (forbidden but exists)
/api/docs                        → 200   ← API docs publicly accessible!

# Step 3: Go one level deeper for interesting paths
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt \
       -u https://corp-target.com/api/v1/FUZZ \
       -mc 200,301,302,400,401,403 -fs 1842 -t 20 \
       -o scan_apiv1.json
```

**Depth strategy:** Three levels of recursion is typically sufficient. Going deeper generates exponentially more requests and rapidly diminishing returns. The priority depth order:
1. Root level → find main directories
2. Discovered directories → find files/subdirectories
3. API paths → find versioned endpoints and documentation

A database dump (`db_backup_2023.sql`) returning 200 from `/backup/` is an immediate critical finding — it contains the full database schema, user records, and potentially hashed passwords. Download it, parse the schema, and look for credentials tables.

## 2.9 Exposed git repository dumping

When `.git/HEAD` returns HTTP 200, the server is exposing the entire git object database through the web server. Every committed file, every deleted file, and every version of every file since the repository's first commit is downloadable.

```bash
# Confirm git exposure
$ curl -sk https://corp-target.com/.git/HEAD
ref: refs/heads/main     ← git repo exposed!

# Method 1: git-dumper (recommended)
$ pip install git-dumper
$ git-dumper https://corp-target.com/.git ./dumped_repo/
# Downloads all git objects and reconstructs the repository locally

# Method 2: Manual reconstruction
$ mkdir dumped && cd dumped && git init
$ curl -sk https://corp-target.com/.git/config > .git/config
$ curl -sk https://corp-target.com/.git/COMMIT_EDITMSG > .git/COMMIT_EDITMSG
$ curl -sk https://corp-target.com/.git/logs/HEAD > .git/logs/HEAD
$ git log --oneline   # see commit history if HEAD/logs downloaded

# After dumping: search for secrets in commit history
# truffleHog — finds secrets across all commits, not just current files
$ pip install trufflehog
$ trufflehog filesystem ./dumped_repo/ --only-verified

🔍 Found verified secret!
  File:   config/database.php
  Commit: a3b4c5d (2023-08-15: "fix DB connection string")
  Secret: DB_PASSWORD=Sup3rS3cr3tP4ss!   ← credential committed to history, later deleted

# GitLeaks — alternative scanner with more rule coverage
$ gitleaks detect --source ./dumped_repo/ --report-format json --report-path leaks.json

# Manual commit history triage
$ cd dumped_repo
$ git log --all --oneline | head -30      # all commits including orphaned branches
$ git show <commit-hash>                   # view a specific commit diff
$ git diff HEAD~10 HEAD -- config/         # changes to config directory over last 10 commits
```

The key insight: deleted files and removed credentials are still in the git history. A developer who committed `DB_PASSWORD=mysecret` and then removed it in the next commit has not actually secured the password — it is permanently in commit history and fully downloadable from the exposed `.git` directory. truffleHog and GitLeaks scan the entire history including deleted content.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Directory brute-forcing** | Systematically guessing directory and file paths on a web server by sending HTTP requests for each guess |
| **ffuf** | Fuzz Faster U Fool — fast HTTP fuzzer with flexible filtering options; primary tool for web content discovery |
| **Gobuster** | Directory/file/DNS/vhost brute-forcing tool; alternative to ffuf |
| **Wappalyzer** | Browser extension and CLI tool that identifies web technologies from HTTP headers and HTML content |
| **whatweb** | Command-line equivalent of Wappalyzer; identifies CMS, server, scripting language, JS libraries |
| **nikto** | Web server scanner for common misconfigurations, dangerous files, and outdated versions |
| **Response code filtering** | Using HTTP status codes (200=found, 301/302=redirect, 403=forbidden) to identify real paths |
| **False positive filtering** | Removing results that appear valid but are actually the server's custom 404 page with 200 status |
| **Wordlist** | A file of common directory/file names used as input for brute-forcing tools |
| **API endpoint** | A URL path that accepts HTTP requests and returns data; may be REST, GraphQL, SOAP, or other |
| **Swagger/OpenAPI** | API documentation standard; if exposed, reveals all endpoints, parameters, and data models |
| **GraphQL introspection** | Query that returns the complete GraphQL schema — all types, queries, mutations, and fields |
| **hakrawler** | Web spider that crawls a site following links and collecting all discovered URLs |
| **LinkFinder** | Python tool that extracts URLs and API endpoints from JavaScript files |
| **Spring Boot Actuator** | Management API exposed by Java Spring Boot applications; `/actuator/env` may expose credentials |
| **`.env` file** | Environment variable file used by PHP, Node.js, Python apps; typically contains database passwords and API keys |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `ffuf` | `ffuf -w wordlist.txt -u https://target/FUZZ -mc 200,301,403` | Directories, files | Primary discovery tool |
| `gobuster dir` | `gobuster dir -u https://target -w wordlist.txt -x php,txt,bak` | Files with extensions | File hunting |
| `whatweb` | `whatweb -v https://target` | Technology stack | After first 200 response |
| `nikto` | `nikto -h https://target` | Misconfigs, sensitive files | Quick baseline scan |
| `curl` | `curl -sk -I https://target/path` | Single path check | Manual verification |
| `hakrawler` | `echo "https://target" \| hakrawler -d 3 -plain` | All linked URLs | Dynamic content discovery |

**Complete web content discovery pipeline:**
```bash
TARGET="https://corp-target.com"

# Step 1: Technology fingerprint
$ whatweb -v $TARGET | tee whatweb_results.txt

# Step 2: Determine 404 baseline size for filtering
$ BASELINE=$(curl -sk "$TARGET/path_that_definitely_doesnt_exist_xyz123" | wc -c)
$ echo "Baseline 404 size: $BASELINE bytes"

# Step 3: Directory brute-force (filter baseline size)
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
       -u "$TARGET/FUZZ" -mc 200,301,302,403 -fs $BASELINE -t 40 \
       -o ffuf_dirs.json -of json

# Step 4: File hunting with extensions
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt \
       -u "$TARGET/FUZZ" \
       -e ".php,.html,.txt,.bak,.old,.zip,.sql,.xml,.json,.env,.config,.log" \
       -mc 200 -fs $BASELINE -t 30 -o ffuf_files.json -of json

# Step 5: Sensitive file checklist
$ for path in robots.txt .env phpinfo.php .git/HEAD actuator/env server-status wp-config.php; do
    code=$(curl -sk -o /dev/null -w "%{http_code}" "$TARGET/$path")
    [ "$code" != "404" ] && [ "$code" != "000" ] && echo "[$code] $TARGET/$path"
  done

# Step 6: nikto scan
$ nikto -h $TARGET -output nikto_results.txt -Format txt

# Step 7: JS endpoint extraction
$ JS_URL=$(curl -sk $TARGET | grep -oP 'src="[^"]+\.js"' | head -1 | sed 's/src="//;s/"//')
$ curl -sk "$TARGET$JS_URL" | grep -oP '"(/[a-z0-9/_-]{3,})"' | sort -u

# Step 8: API discovery
$ ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt \
       -u "$TARGET/api/FUZZ" -mc 200,201,400,401,403,405 -t 20 -o ffuf_api.json
```

---

# Section 5 — Defender detection

- **Access log entries:** Every ffuf or gobuster request appears in the web server's access log with your source IP, the requested path, HTTP method, status code, and timestamp. A burst of requests across hundreds of paths in seconds is an obvious brute-force pattern: `203.0.113.0 - - [27/Aug/2024:22:15:01] "GET /admin HTTP/1.1" 404` × 5000 times in 2 minutes.
- **WAF detection:** Most WAFs (Cloudflare, AWS WAF, ModSecurity) detect directory brute-forcing within seconds and either block the source IP, return a CAPTCHA, or start returning fake 200 responses to confuse the scanner. If ffuf results suddenly show hundreds of 200 responses to obscure paths, the WAF has entered deception mode.
- **nikto signatures:** Nikto is well-known and WAFs have specific signatures for its User-Agent string (`Nikto/2.1.6`) and request patterns. Always change the nikto User-Agent: `nikto -h $TARGET -useragent "Mozilla/5.0 (compatible)"`.
- **Rate limiting:** Web servers and WAFs commonly rate-limit by source IP. After a threshold (e.g. 100 requests/minute), subsequent requests are either slowed or blocked. Use `-rate` in ffuf or `--delay` in gobuster to reduce request rate.
- **Mitigation for operators:** (1) Set a realistic User-Agent in ffuf: `-H "User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"`. (2) Add a short delay: `-p 0.1` in ffuf adds 100ms between requests. (3) Use fewer threads: `-t 10` instead of `-t 100`. (4) Use Burp Suite's Intruder in Sniper mode for slow, authenticated, session-aware brute-forcing.

---

# Section 6 — Lab task

**Platform:** TryHackMe "Content Discovery" room or "ffuf" room. Alternatively: DVWA, Metasploitable, or OWASP WebGoat running locally.

**Objective:** Complete a full web content discovery pipeline against a target web server and produce an annotated attack surface map.

**Steps:**

1. **Technology fingerprint:** `whatweb -v http://target-ip/ | tee whatweb.txt` — note CMS, server, and scripting language
2. **Baseline 404 size:** `curl -s http://target-ip/pathdoesntexist | wc -c`
3. **Directory brute-force:** `ffuf -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://target-ip/FUZZ -mc 200,301,302,403 -fs <baseline> -t 40 -o dirs.json`
4. **File hunt (common extensions):** `ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-medium-words.txt -u http://target-ip/FUZZ -e .php,.txt,.bak,.html,.zip,.sql -mc 200 -fs <baseline> -t 30 -o files.json`
5. **Sensitive file checklist:** Run the manual curl loop from Section 4 — document all non-404 results
6. **nikto scan:** `nikto -h http://target-ip -useragent "Mozilla/5.0" -output nikto.txt`
7. **Review access log (if accessible):** Log into the target VM and `tail /var/log/apache2/access.log` — observe your scan traffic
8. **API discovery:** `ffuf -w /usr/share/seclists/Discovery/Web-Content/api/api-endpoints.txt -u http://target-ip/api/FUZZ -mc 200,201,400,401,403 -t 20`
9. **JS endpoint extraction:** Find a `.js` file in the page source, download it, and grep for path patterns
10. **Compile `web_surface.md`:** Table of all discovered paths classified by category (admin panel, backup file, API endpoint, config file, exposed git) with HTTP status, size, and risk level

```bash
git commit -m "recon(stage3): web content discovery — attack surface map for <target>"
```

---

# Section 7 — Common mistakes

**1. Not filtering false positives (custom 404 pages with 200 status)**
_Why it matters:_ Some web servers return a 200 OK response even for nonexistent paths, serving a "page not found" page with 200 status instead of 404. ffuf with `-mc 200` treats every one of these as a valid finding — thousands of false positives.
_Fix:_ Always test a random nonexistent path first and note its response size. Use `-fs <size>` to filter that size from results. Alternatively, use `-fw` (filter words) or `-fl` (filter lines) if the size varies.

**2. Using weak or generic wordlists**
_Why it matters:_ The default wordlist `common.txt` (only ~4000 entries) misses the vast majority of real paths. WordPress installs, Laravel applications, and Spring Boot services all have specific directory structures not covered by generic lists.
_Fix:_ Match wordlist to technology. Use `wp-plugins.fuzz.txt` for WordPress, `spring-boot.txt` for Java Spring, `raft-large-directories.txt` for thorough general enumeration.

**3. Not checking for exposed git repositories**
_Why it matters:_ An exposed `.git` directory is one of the highest-value web findings. The entire codebase, commit history, and any credentials ever committed are accessible. This is consistently found and consistently overlooked.
_Fix:_ Always check `.git/HEAD` explicitly. If it returns `ref: refs/heads/main` (a 200 response), use `git-dumper` to download the entire repository.

**4. Running nikto without changing User-Agent**
_Why it matters:_ Nikto sends `Nikto/2.1.6` as its User-Agent. Every WAF has signatures for this. The scan is detected and blocked within the first few requests.
_Fix:_ Always use `-useragent "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"` or similar innocuous User-Agent.

**5. Ignoring JavaScript files for endpoint discovery**
_Why it matters:_ Modern React/Angular/Vue SPAs bundle all routes and API endpoints into JavaScript files. Standard directory brute-forcing only finds server-side paths — client-side routes are invisible without JavaScript analysis.
_Fix:_ Always download and analyze the main JavaScript bundle. Use LinkFinder or grep for path patterns.

---

# Section 8 — Move-on gate

1. A web server returns a 200 status response for every path you request including `/randomnonexistent123`. Without notes, describe the problem, state how you detect it before running a full scan, and name the exact ffuf flag and its value that solves it.

2. ffuf discovers `/admin/` returning 403 Forbidden. State whether this is a useful finding and explain what three follow-up actions you would take from this discovery before concluding whether admin access is achievable.

3. You discover a path `/.git/HEAD` returning HTTP 200. Without notes, describe what this means, name the tool you would use to exploit it, and state what categories of sensitive data you might find inside the downloaded repository.
