# Subdomain & DNS Enumeration

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 2: Semi-Passive Infrastructure Mapping

# Section 1 — What it is and where it sits

Subdomain and DNS enumeration systematically discovers the full name space of a target organization — all active subdomains, and the complete DNS record set (A, AAAA, MX, TXT, NS, CNAME, SRV) for each. A zone has one `corp-target.com` — but it may have 300 subdomains, each representing a distinct service, application, or infrastructure component. Every subdomain is a potentially independent attack surface.

DNS enumeration sits in Stage 2 because the most effective techniques query third-party aggregation services (certificate transparency logs, passive DNS databases, search engine indexes) rather than the target's authoritative DNS server directly. This separates passive/semi-passive subdomain discovery from active DNS brute-forcing and zone transfer attempts, which belong in Stage 3.

```text
Recon Chain
──────────────────────────────────────────────────────────────────────────
Stage 1 (Passive)     Stage 2 (Semi-Passive)              Stage 3 (Active)
WHOIS, CT, leaks  →  [Subdomain + DNS Enum]  →  Active DNS brute  →  Port scan
Breach data,          ↑ YOU ARE HERE            Zone transfer      → Web probing
Passive DNS            CT logs, passive DNS,    attempts           → Endpoint fuzz
                       aggregation tools
──────────────────────────────────────────────────────────────────────────
```

**What breaks if you skip this:** You enter Phase 3 with a truncated attack surface. The target's primary domain may be hardened and monitored — the vulnerable target is almost always a subdomain: a forgotten dev environment, an API endpoint without WAF protection, an admin panel on a non-standard hostname, or a microservice that was never intended to be public-facing.

This step follows external intel scouring (note 01) and Shodan/Censys queries (note 02). Certificate SANs and Shodan banners already revealed some subdomains. Subdomain enumeration consolidates those findings, applies dedicated tooling to expand coverage, and produces a complete resolved host list for the next stage.

---

# Section 2 — How attackers actually use this

## 2.1 What subdomains reveal

A subdomain name often reveals its purpose directly. Common patterns that indicate high-value targets:

| Subdomain pattern | What it likely is | Why it matters |
|-------------------|------------------|----------------|
| `dev.`, `staging.`, `test.` | Development or staging environment | Often runs without WAF, lower-security config, debug endpoints enabled |
| `api.`, `api-v2.`, `graphql.` | Backend API endpoint | May expose unauthenticated or over-privileged functionality |
| `admin.`, `portal.`, `manage.` | Admin panel | High-value credential target; often has weaker auth |
| `vpn.`, `remote.`, `rdp.` | Remote access gateway | Direct network access vector |
| `mail.`, `smtp.`, `mx.` | Mail infrastructure | Relevant for relay abuse or mail-based attacks |
| `jenkins.`, `gitlab.`, `jira.` | Internal dev tools | Credential theft, code access, secrets in repos |
| `backup.`, `archive.` | Backup or legacy systems | Old software, weak auth, forgotten credentials |
| `s3.`, `files.`, `cdn.` | Storage or CDN origin | Potential bucket misconfiguration, direct file access |

The name alone sets priority. An attacker encountering `jenkins.corp-target.com` immediately knows the attack hypothesis: unauthenticated Jenkins console, weak credentials, or script console RCE.

## 2.2 Passive subdomain discovery with Amass and Subfinder

Amass in passive mode (`-passive`) and Subfinder query CT logs, passive DNS databases, search engine indexes, threat feeds, and API services (VirusTotal, Shodan, Censys, SecurityTrails, etc.) to enumerate subdomains without generating any DNS traffic toward the target. This is the primary first-pass technique.

The workflow:
```text
Run Amass passive + Subfinder
        ↓
Merge output, deduplicate
        ↓
Resolve all names with dnsx (which generates DNS traffic — semi-active)
        ↓
Separate live hosts from dead ones
        ↓
Port scan live hosts (Stage 3)
```

Amass produces larger, richer output because it queries more sources. Subfinder is faster and more reliable for CT-log-heavy enumeration. Running both and merging results maximizes coverage.

## 2.3 DNS record type intelligence

Beyond A records, the full DNS record set for each domain and subdomain provides infrastructure intelligence that pure subdomain enumeration misses:

**MX records** reveal the mail provider. `aspmx.l.google.com` confirms Google Workspace. `mail.protection.outlook.com` confirms Microsoft 365. A self-hosted MX hostname (`mail.corp-target.com`) may indicate on-premise Exchange or Postfix — a separate attack surface.

**TXT records** are the most information-dense DNS record type. They contain: SPF policy (see note 09 in Stage 1), DMARC policy, DKIM selectors, domain verification tokens for SaaS platforms (e.g. `google-site-verification=`, `MS=`, `docusign-`, `dropbox-`, `atlassian-domain-verification=`), and sometimes misconfigured internal data. Each verification token identifies a SaaS platform the organization uses — free technology mapping.

**NS records** identify the DNS provider (Route53, Cloudflare, GoDaddy). The NS record for a subdomain may differ from the parent domain's NS records, indicating a delegated zone — possibly managed by a third party or forgotten.

**CNAME records** are critical for dangling subdomain detection. A CNAME pointing to a service that no longer exists (e.g. a deleted S3 bucket, a removed Heroku app, a cancelled CDN endpoint) is a takeover vulnerability — you can register that external resource and serve content from the target's subdomain.

**SRV records** expose internal services and their ports: `_ldap._tcp.corp-target.com` confirms Active Directory LDAP; `_kerberos._tcp` confirms a Kerberos environment; `_sip._tcp` reveals VoIP infrastructure.

## 2.4 CNAME dangling and subdomain takeover surface

A dangling CNAME is a CNAME record that points to a hostname that resolves to NXDOMAIN or belongs to an unclaimed cloud resource. The attack: register the external resource, point it to your content, and the target's subdomain now serves your page under the target's trusted domain.

```text
old-app.corp-target.com  CNAME  old-app.azurewebsites.net
                                        ↓
                            [Azure App Service deleted]
                                        ↓
                            azurewebsites.net returns NXDOMAIN
                                        ↓
         Register old-app.azurewebsites.net → serve phishing page
                                        ↓
         old-app.corp-target.com now points to your content
```

Platforms known to be takeover-vulnerable: GitHub Pages, Heroku, S3 static hosting, Azure App Service, Netlify, Fastly, Pantheon, and others.

## 2.5 Dead-end vs high-value finding

**Dead-end:** Amass returns 12 subdomains. `dnsx` resolves 10 of them. All 10 resolve to the same CDN IP range (Cloudflare). All serve the same corporate website with identical HTTP fingerprints. No record type variation beyond A and CNAME. This is a standard CDN-fronted deployment — the subdomains exist but are all the same application. Move to WAF fingerprinting.

**High-value:** Amass returns `jenkins.corp-target.com`. `dnsx` resolves it to `203.0.113.47` — a direct IP (not CDN, confirmed because the IP is in the target's own ASN from Shodan). TXT record lookup for `corp-target.com` reveals `atlassian-domain-verification=abc123` — they also use Atlassian/Jira. CNAME check on `old-portal.corp-target.com` resolves to `old-portal.azurewebsites.net` which returns NXDOMAIN — takeover opportunity. MX records show `mail.corp-target.com` as a self-hosted server — separate attack surface from Google Workspace.

## 2.6 Where results feed next

The resolved subdomain list feeds every subsequent stage. Live subdomains with direct IPs go to Stage 3 active scanning. Subdomains behind CDN/WAF go to Stage 2 WAF fingerprinting (note 05) and VHost hunting (note 06). CNAME dangles go into a takeover-opportunity tracker. SRV records confirming AD/Kerberos feed Phase 3 Active Directory enumeration planning. TXT verification tokens feed the SaaS technology map.

---

## 2.7 crt.sh and DNSdumpster manual workflow

**crt.sh** is a public Certificate Transparency log search interface. Every certificate issued to any domain and its subdomains is logged to CT logs within 24 hours of issuance. crt.sh makes those raw logs searchable by domain name. Unlike Amass and Subfinder which use CT as one of many sources, querying crt.sh directly gives you the unfiltered CT log data without external API key dependencies.

crt.sh is particularly valuable for finding subdomains that existed briefly, used Let's Encrypt certificates, and are no longer resolving. CT records are permanent even when the subdomain is gone — making crt.sh a historical subdomain discovery tool in addition to a current-state one.

```bash
# Web query: https://crt.sh/?q=%25.corp-target.com
# The % wildcard matches all subdomains

# API-based query for automation
$ curl -s "https://crt.sh/?q=%25.corp-target.com&output=json" \
  | jq -r '.[].name_value' \
  | tr ',' '\n' \
  | sed 's/^\*\.//g' \
  | sort -u | tee crtsh_output.txt

api.corp-target.com
dev.corp-target.com
internal.corp-target.com       ← not in Amass output
staging.corp-target.com
vpn.corp-target.com
*.corp-target.com
```

crt.sh frequently surfaces `internal.*`, `dev.*`, and `staging.*` subdomains that passive DNS aggregators miss — particularly short-lived certificates for development environments that were exposed via Let's Encrypt before being taken offline. The fact that a certificate was issued means the subdomain existed and had a valid web service; even if DNS no longer resolves it, the certificate record is permanent and the IP may still be live.

**DNSdumpster** (`dnsdumpster.com`) queries multiple DNS aggregation sources and presents a visual DNS map — MX, NS, A, and TXT records for the domain and its known subdomains, plotted as a relationship graph showing IP-to-hostname connections and hosting providers. It also shows the AS organization name and geolocation for each IP. The HTML export gives a ready-made infrastructure diagram. DNSdumpster is a fast manual tool for an initial visual overview before running automation.

## 2.8 PTR record sweeping for IP-to-hostname mapping

PTR records are the reverse of A records: they map an IP address to a hostname. For IP ranges where the target has configured reverse DNS, sweeping the full CIDR converts raw IP addresses into meaningful hostnames — often revealing internal naming conventions and services that forward DNS enumeration never surfaces.

Organizations configure PTR records primarily for their own operational convenience (identifying machines in logs) and rarely restrict or audit them. PTR records are publicly resolvable for any IP with configured reverse DNS, and the naming conventions they expose were designed for internal use, not external visibility.

```bash
# dnsx PTR sweep of a /24 CIDR (from Stage 1 WHOIS and Shodan data)
$ seq 1 254 | xargs -I{} echo "203.0.113.{}" | dnsx -ptr -resp-only -silent | tee ptr_results.txt

203.0.113.1   → gateway.corp-target.com
203.0.113.5   → mail.corp-target.com
203.0.113.10  → vpn.corp-target.com
203.0.113.45  → web01.corp-target.com
203.0.113.50  → db-primary.corp-target.com
203.0.113.51  → db-replica.corp-target.com     ← primary-replica DB architecture
203.0.113.100 → monitoring.corp-target.com     ← monitoring system hostname
203.0.113.200 → dc01.corp-target.com           ← domain controller
```

The PTR sweep reveals internal architecture in one pass: a primary-replica database cluster (`db-primary`, `db-replica`), a domain controller (`dc01`), a monitoring stack (`monitoring`), and the VPN server. None of these appeared in forward DNS enumeration because no A records point to them from the public zone. This feeds directly into Phase 3 AD enumeration (the `dc01` hostname confirms an AD environment) and data exfiltration targeting (the `db-primary`/`db-replica` pair confirms a relational database cluster).

```bash
# For a /16 CIDR (larger range), use prips to generate the list
$ sudo apt install prips
$ prips 203.0.0.0 203.0.255.255 | dnsx -ptr -resp-only -silent | grep 'corp-target' | tee ptr_large.txt

# Filter PTR results to only confirmed-target hostnames
$ cat ptr_results.txt | grep -i 'corp-target' | sort
```

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Zone transfer (AXFR)** | DNS operation that copies the entire zone file from an authoritative server — only works if the server is misconfigured to allow it from arbitrary IPs |
| **Wildcard DNS** | A record like `*.corp-target.com → IP` that matches all subdomains; tools must detect and handle this to avoid false positives (all names appear to resolve) |
| **Passive DNS enumeration** | Querying third-party aggregated databases without generating traffic to the target's authoritative DNS server |
| **Active DNS brute-forcing** | Sending DNS queries for wordlist-based names directly to a resolver; generates traffic visible to recursive resolvers |
| **CNAME takeover** | Claiming an external resource pointed to by a dangling CNAME, allowing content delivery under the target's domain |
| **Dangling CNAME** | A CNAME record pointing to an external hostname that resolves to NXDOMAIN or an unclaimed cloud resource |
| **SOA record** | Start of Authority — identifies the primary nameserver and administrative contact for a DNS zone |
| **SRV record** | Service locator record specifying hostname and port for a named service (e.g. `_ldap._tcp`) |
| **PTR record** | Reverse DNS — maps an IP to a hostname; can reveal internal hostnames when reverse DNS is publicly configured |
| **DNS resolver** | Server that performs recursive DNS lookups on behalf of clients; `dnsx` uses public resolvers to resolve discovered names |
| **Wildcard detection** | Technique to identify if a zone has a wildcard record by querying a random non-existent subdomain and checking if it resolves |
| **Permutation enumeration** | Generating subdomain candidates by combining known words: `api-v2`, `staging-api`, `dev2`, etc. |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `amass` | `amass enum -passive -d corp-target.com -o amass_out.txt` | Subdomains from CT, passive DNS, search indexes (no direct DNS traffic) | Primary passive discovery |
| `subfinder` | `subfinder -d corp-target.com -o subfinder_out.txt` | Subdomains from CT logs, Shodan, VT, ThreatCrowd, and others | Complementary passive discovery |
| `dnsx` | `cat all_subs.txt \| dnsx -a -cname -txt -mx -ns -resp -o resolved.txt` | Resolve names, return A/CNAME/TXT/MX/NS records | Post-enumeration resolution |
| `dnsx` | `dnsx -l all_subs.txt -a -resp-only -o live_ips.txt` | Only live A-record results | Filter dead names |
| `dig` | `dig +short A target.corp-target.com` | Current A record for one name | Spot-check resolution |
| `dig` | `dig +short TXT corp-target.com` | All TXT records (SPF, DMARC, SaaS verifications) | Technology mapping |
| `dig` | `dig +short NS corp-target.com` | Authoritative nameservers | DNS provider identification |
| `dig` | `dig +short SRV _ldap._tcp.corp-target.com` | LDAP SRV — confirms AD environment | Internal service discovery |
| `dnsrecon` | `dnsrecon -d corp-target.com -t brt -D /usr/share/dnsrecon/namelist.txt` | DNS brute-force with built-in wordlist (active) | Stage 3 active brute-force |
| `alterx` | `echo corp-target.com \| alterx \| dnsx -a -resp-only` | Permutation-based subdomain generation + resolution | Expanding coverage beyond CT |

**Installation:**
```bash
go install -v github.com/owasp-amass/amass/v4/...@master
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/alterx/cmd/alterx@latest
```

**Amass passive — reading the output:**
```bash
$ amass enum -passive -d corp-target.com -o amass_out.txt
[Found] api.corp-target.com
[Found] staging.corp-target.com
[Found] jenkins.corp-target.com
[Found] admin.corp-target.com
[Found] mail.corp-target.com
[Found] old-portal.corp-target.com
```
Amass labels the source for each result (CT, VirusTotal, PassiveDNS etc.) if you add `-v`. Priority names immediately visible: `jenkins`, `admin`, `old-portal`.

**Subfinder — complementary discovery:**
```bash
$ subfinder -d corp-target.com -silent -o subfinder_out.txt
vpn.corp-target.com
api-v2.corp-target.com
dev.corp-target.com
```
Subfinder found `vpn`, `api-v2`, and `dev` that Amass missed. Always run both.

**Merge and resolve:**
```bash
$ cat amass_out.txt subfinder_out.txt | sort -u > all_subs.txt
$ cat all_subs.txt | dnsx -a -cname -resp -silent | tee resolved.txt
api.corp-target.com [A] [203.0.113.45]
staging.corp-target.com [A] [198.51.100.5]
jenkins.corp-target.com [A] [203.0.113.47]
admin.corp-target.com [CNAME] [corp-target.azurewebsites.net]   ← check if live
old-portal.corp-target.com [CNAME] [old-portal.azurewebsites.net] [NXDOMAIN]  ← dangling!
vpn.corp-target.com [A] [203.0.113.50]
```
`old-portal` resolves to NXDOMAIN via its CNAME target — potential takeover. `admin` resolves to Azure App Service — check if it's actively in use or another dangling CNAME.

**TXT record mining for SaaS tech mapping:**
```bash
$ dig +short TXT corp-target.com
"v=spf1 include:_spf.google.com include:sendgrid.net ~all"
"v=DMARC1; p=quarantine; rua=mailto:dmarc@corp-target.com"
"google-site-verification=abc123xyz"
"MS=ms12345678"
"atlassian-domain-verification=xyz789"
"docusign=abc123"
"dropbox-domain-verification=xyz"
```
Technology map extracted: Google Workspace, Sendgrid, Microsoft 365 (or Azure AD), Atlassian (Jira/Confluence), DocuSign, Dropbox — all from one DNS TXT lookup.

**SRV record for Active Directory detection:**
```bash
$ dig +short SRV _ldap._tcp.corp-target.com
0 100 389 dc01.corp-target.com.
$ dig +short SRV _kerberos._tcp.corp-target.com
0 100 88 dc01.corp-target.com.
```
Active Directory domain controller at `dc01.corp-target.com` on ports 389 (LDAP) and 88 (Kerberos) — confirmed AD environment, feeds Phase 3 AD enumeration planning.

---

**crt.sh API — subdomain extraction automation:**
```bash
# Simple bash one-liner for crt.sh
$ curl -s "https://crt.sh/?q=%25.corp-target.com&output=json" \
  | python3 -c "
import json,sys
data = json.load(sys.stdin)
names = set()
for cert in data:
    for name in cert['name_value'].split('\\n'):
        name = name.strip().lstrip('*.')
        if name and 'corp-target.com' in name:
            names.add(name)
for n in sorted(names): print(n)
" | tee crtsh_subdomains.txt

# Count unique results
$ wc -l crtsh_subdomains.txt
47 crtsh_subdomains.txt   ← 47 unique subdomains from CT logs alone

# Cross-reference with Amass/Subfinder results to find crt.sh-exclusive findings
$ comm -23 crtsh_subdomains.txt all_subs.txt
internal.corp-target.com       ← only in crt.sh, not in Amass or Subfinder
sso.corp-target.com            ← SSO portal, not in other sources
```
`internal.corp-target.com` appearing exclusively in crt.sh means it had a certificate at some point but was never indexed by passive DNS or search engines. If it still resolves, it is a hidden endpoint. If it does not resolve, check Shodan's history for the IP the cert pointed to.

**PTR record sweep commands:**
```bash
# Install dnsx if not present
$ go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest

# Sweep a /24 range
$ seq 1 254 | awk '{print "203.0.113."$1}' | dnsx -ptr -resp-only -silent | tee ptr_24.txt

# Sweep a /16 range using prips
$ prips 203.0.0.0 203.0.255.255 | dnsx -ptr -resp-only -silent -c 100 | grep corp-target | tee ptr_16.txt

# Extract just the hostname from PTR results and resolve forward to confirm
$ cat ptr_24.txt | dnsx -a -resp -silent | tee ptr_forward_confirmed.txt

# PTR results that have no forward A record = stale PTR = old infrastructure
$ diff <(cat ptr_24.txt | awk '{print $3}') <(cat ptr_forward_confirmed.txt | awk '{print $1}') 
```
Stale PTR records — where the reverse DNS points to a hostname but the hostname no longer resolves forward — indicate decommissioned infrastructure. The IP may still be reachable even though DNS no longer routes to it. Check the IP directly in Shodan to confirm current port state.

# Section 5 — Defender detection

Passive subdomain enumeration using Amass passive mode and Subfinder queries third-party services (CT logs, passive DNS APIs, search engines) — the target's authoritative DNS server receives no queries. This is invisible to the target.

- **Wildcard detection DNS queries:** When dnsx or amass tests for wildcard records (querying `randomname-doesnotexist.corp-target.com`), that query resolves via public resolvers and is visible in the target's authoritative server query logs — if they have DNS query logging enabled. This is a very low-volume signature, rarely monitored.
- **Amass active mode** (`-active` flag, NOT used in Stage 2) sends zone transfer attempts and DNS brute-force queries directly to the authoritative server. These generate events. Amass passive mode does not.
- **dnsx resolution** generates DNS traffic for each resolved name — but this is indistinguishable from normal internet DNS traffic. Public resolvers (8.8.8.8, 1.1.1.1) forward queries to the target's authoritative server, which sees them as coming from those resolver IPs, not from you.
- **Defenders monitoring their subdomains in external scan tools** (e.g. via Shodan alerts or Censys notifications) may see your resolution activity indirectly if those tools also scan the IPs you discover.
- **Rate limiting:** If you brute-force thousands of subdomains via dnsx at high concurrency, the burst of DNS queries from a single resolver IP can trigger rate limiting at the authoritative server. Keep dnsx concurrency moderate (`-c 50` rather than `-c 500`) for stealth.

---

# Section 6 — Lab task

**Platform:** TryHackMe — *"DNS in Detail"* or *"Passive Reconnaissance"* rooms cover DNS fundamentals. For subdomain enumeration specifically: TryHackMe *"Subdomain Enumeration"*. Alternatively, use Kali VM with Amass and dnsx against `testphp.vulnweb.com` (authorized target maintained by Acunetix).

**Objective:** Enumerate all discoverable subdomains for the target using Amass and Subfinder (passive only), resolve them with dnsx, extract DNS record intelligence, and identify at least one dangling CNAME or high-value subdomain.

**Steps:**

1. Install tools: `go install github.com/owasp-amass/amass/v4/...@master` and `go install github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest` and `go install github.com/projectdiscovery/dnsx/cmd/dnsx@latest`
2. Configure API keys for Subfinder (VT, Shodan, Censys): create `~/.config/subfinder/provider-config.yaml` with your keys.
3. Run Amass passive: `amass enum -passive -d vulnweb.com -o amass_out.txt` (use the parent domain for broader results)
4. Run Subfinder: `subfinder -d vulnweb.com -silent -o subfinder_out.txt`
5. Merge results: `cat amass_out.txt subfinder_out.txt | sort -u | tee all_subs.txt && wc -l all_subs.txt`
6. Resolve all names: `cat all_subs.txt | dnsx -a -cname -resp -silent | tee resolved.txt`
7. Check for NXDOMAIN or dangling CNAMEs: `grep -i "nxdomain\|CNAME" resolved.txt`
8. Extract TXT records for the root domain: `dig +short TXT vulnweb.com` — map every token to a SaaS product.
9. Check for SRV records: `dig +short SRV _ldap._tcp.vulnweb.com` and `_kerberos._tcp.vulnweb.com`
10. Classify each discovered subdomain by risk level and populate `subdomain_inventory.md`: columns Subdomain | IP | Source | DNS Type | Risk | Notes

**Expected output:** `resolved.txt` with at least 5 live subdomains, `subdomain_inventory.md` with risk classifications, a SaaS tech map from TXT records, and at least one CNAME entry documented with its resolution chain.

**Git artifact:**
```
recon/stage2/subdomain-enum/
├── amass_out.txt
├── subfinder_out.txt
├── all_subs.txt
├── resolved.txt
└── subdomain_inventory.md
```
```bash
git commit -m "recon(stage2): subdomain + DNS enum — resolved host inventory and tech map for <target>"
```

---

# Section 7 — Common mistakes

**1. Running Amass active mode during Stage 2**
_Why it matters:_ `amass enum -active` attempts zone transfers and sends DNS queries directly to the target's authoritative nameserver. This generates events in the target's DNS query logs. Active DNS enumeration belongs in Stage 3 after deliberate scoping.
_Fix:_ Always use `-passive` flag in Stage 2. Confirm you are running passive mode by checking that `amass` does not show zone transfer attempts in its verbose output.

**2. Not handling wildcard DNS records**
_Why it matters:_ A wildcard record (`* → IP`) makes every subdomain you query appear to resolve, even invented ones. This floods your resolved list with false positives — `doesnotexist.corp-target.com` resolves to the same IP as real subdomains.
_Fix:_ Detect wildcards first: `dig +short randomxyz123456.corp-target.com`. If it resolves, the zone has a wildcard. Use dnsx's `-wc` flag or filter results where all IPs match the wildcard IP.

**3. Using only one subdomain source**
_Why it matters:_ No single tool or source has complete coverage. CT logs miss subdomains that never had certificates. Passive DNS misses names that resolved only briefly. Amass and Subfinder use different source sets. Using only one produces significant gaps.
_Fix:_ Always run at least two tools (Amass + Subfinder) and merge results. Add `assetfinder` or `chaos` if scope allows.

**4. Ignoring CNAME chains and not checking for dangling CNAMEs**
_Why it matters:_ A CNAME pointing to an unclaimed external resource is a subdomain takeover vulnerability — one of the most impactful web findings, achievable entirely through passive recon. Ignoring CNAMEs in resolved output leaves this on the table.
_Fix:_ For every CNAME in `resolved.txt`, resolve the final target hostname and check if it returns NXDOMAIN or a generic "no content" page from a cloud provider. Use `subjack` or `nuclei` for automated CNAME takeover checking.

**5. Skipping non-A record types**
_Why it matters:_ Running dnsx with `-a` only gives you hostnames and IPs. TXT records reveal SaaS stack. SRV records reveal AD/Kerberos. NS records reveal DNS delegation. MX records reveal mail infrastructure. All of this is free intelligence from the same tool.
_Fix:_ Always run dnsx with at minimum `-a -cname -txt -mx -ns` flags. The additional query time is negligible; the intelligence gain is significant.

**6. Not resolving subdomains before reporting them as discovered**
_Why it matters:_ CT logs and passive DNS contain historical names — subdomains that existed years ago and no longer resolve. Reporting `old-admin.corp-target.com` as an active attack surface when it resolves to NXDOMAIN misleads the engagement scope.
_Fix:_ The resolution step (dnsx) is mandatory before any subdomain is counted as an active finding. Separate live-resolving names from historical-only names in your output.

---

# Section 8 — Move-on gate

1. Run Amass passive and Subfinder against a target domain, merge the results, resolve them with dnsx, and produce a correctly formatted `subdomain_inventory.md` — without looking at notes — including live host count, CNAME chain results, and wildcard detection outcome.

2. From a dnsx resolved output, identify a dangling CNAME (NXDOMAIN resolution at the final target), name the specific cloud platform it points to, and explain the exact steps to claim that resource and complete a subdomain takeover — without consulting documentation.

3. Run a full TXT record lookup on a target domain, identify at least three distinct SaaS or cloud services from the verification tokens and SPF includes, and explain how each service could be relevant to a subsequent attack phase.
