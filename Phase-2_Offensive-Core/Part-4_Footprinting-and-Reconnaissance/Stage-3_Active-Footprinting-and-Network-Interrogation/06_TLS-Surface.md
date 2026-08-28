# TLS Surface

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

TLS surface mapping is the active interrogation of a target's TLS/SSL implementation — directly connecting to each HTTPS-serving host and probing the certificate chain, supported cipher suites, protocol versions, TLS extensions, and HTTP redirect behavior. Where Stage 2's protocol audit read Shodan's cached TLS data, Stage 3 directly connects to every host and measures the current live state.

The output is a complete TLS risk profile: weak cipher suites that enable decryption, protocol downgrades that enable man-in-the-middle, expired or misconfigured certificates, SAN entries revealing additional hosts, and HTTP/2 or ALPN support indicating protocol capabilities.

```text
Stage 3 TLS Recon Chain
────────────────────────────────────────────────────────────────────
Port Scan  →  Web Content Discovery  →  [TLS Surface]
443 open       tech stack found             ↑ YOU ARE HERE
               paths discovered          Active TLS probing
                                         testssl.sh, tlsx, nmap
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 Certificate SAN harvesting

Every TLS certificate contains a Subject Alternative Name (SAN) extension listing every domain and subdomain the certificate is valid for. A single certificate may cover dozens of hostnames — a complete inventory of that host's virtual hosts and related services. This is fully active: you must connect to the host to read its certificate.

```bash
# openssl — read SANs from a live host
$ openssl s_client -connect corp-target.com:443 -servername corp-target.com </dev/null 2>/dev/null \
  | openssl x509 -noout -text \
  | grep -A 20 "Subject Alternative Name"

X509v3 Subject Alternative Name:
    DNS:corp-target.com
    DNS:www.corp-target.com
    DNS:api.corp-target.com
    DNS:admin.corp-target.com           ← admin panel hostname confirmed
    DNS:staging.corp-target.com         ← staging environment in SAN!
    DNS:mail.corp-target.com
    DNS:vpn.corp-target.com
    DNS:corp-target-eu.com              ← related domain
    IP Address:203.0.113.45

# tlsx — fast mass SAN extraction across multiple hosts
$ echo "corp-target.com" | tlsx -san -silent
admin.corp-target.com
staging.corp-target.com
corp-target-eu.com
api.corp-target.com

# From an IP list (scan every discovered host)
$ cat alive_web.txt | tlsx -san -silent | sort -u > all_sans.txt
```

`staging.corp-target.com` in the SAN is high-value: staging environments are often deployed with debug mode enabled, verbose error messages, weaker authentication, and no WAF. The staging certificate being on the same server as production reveals that both environments share infrastructure.

## 2.2 Cipher suite enumeration

Not all cipher suites are equal. The supported cipher suites on a TLS server determine: whether forward secrecy is available, whether the session can be decrypted if the private key is obtained, and whether specific downgrade attacks are possible.

```bash
# testssl.sh — comprehensive cipher suite and protocol audit
$ testssl.sh https://corp-target.com

Testing protocols (via sockets except TLS 1.3, QUIC)
 SSLv2:      not offered (OK)
 SSLv3:      not offered (OK)
 TLS 1:      offered (DEPRECATED)         ← flagged
 TLS 1.1:    offered (DEPRECATED)         ← flagged
 TLS 1.2:    offered (OK)
 TLS 1.3:    offered (OK)

Testing cipher categories
 NULL ciphers (no encryption):    not offered (OK)
 Anonymous NULL Ciphers (no auth):not offered (OK)
 Export ciphers (<=40 bit):       not offered (OK)
 LOW: 64 Bit + DES, RC4, MD5:    not offered (OK)
 3DES Ciphers / IDEA:             offered (VULNERABLE)   ← SWEET32
 Obsoleted CBC ciphers (TLS >= 1.2): offered
 Strong encryption (AEAD ciphers): offered (OK)

Testing vulnerabilities
 Heartbleed (CVE-2014-0160):     not vulnerable (OK)
 CCS (CVE-2014-0224):            not vulnerable (OK)
 Ticketbleed (CVE-2016-9244):    not vulnerable (OK)
 SWEET32 (CVE-2016-2183):        VULNERABLE                  ← CVE!
 BEAST (CVE-2011-3389):          TLS1: CBC cipher VULNERABLE  ← CVE!
 POODLE, TLS (CVE-2015-3864):    not vulnerable (OK)
 DROWN (CVE-2016-0800/CVE-2016-0703): not vulnerable (OK)
```

**Key cipher categories:**

| Category | Risk | Attack implication |
|----------|------|-------------------|
| No ECDHE/DHE forward secrecy | High | Recorded TLS sessions decryptable with private key |
| TLS 1.0 with CBC cipher | High | BEAST attack (CVE-2011-3389) — session decryption |
| 3DES (SWEET32) | Medium | CVE-2016-2183 — statistical birthday attack |
| RC4 | High | Stream cipher bias attacks |
| Anonymous DH/ECDH | Critical | No server auth — trivial MITM |
| Export-grade ciphers | Critical | FREAK (CVE-2015-0204) — downgrade to 40-bit |

## 2.3 Protocol version probing

Determining exactly which TLS protocol versions are accepted by the server:

```bash
# Manual version probing with openssl
$ openssl s_client -connect corp-target.com:443 -tls1   2>&1 | grep "Protocol"
Protocol  : TLSv1         ← TLS 1.0 accepted

$ openssl s_client -connect corp-target.com:443 -tls1_1 2>&1 | grep "Protocol"
Protocol  : TLSv1.1       ← TLS 1.1 accepted

$ openssl s_client -connect corp-target.com:443 -tls1_2 2>&1 | grep "Protocol"
Protocol  : TLSv1.2       ← TLS 1.2 accepted

$ openssl s_client -connect corp-target.com:443 -tls1_3 2>&1 | grep "Protocol"
Protocol  : TLSv1.3       ← TLS 1.3 accepted

# SSLv3 check (should be absent in any modern system)
$ openssl s_client -connect corp-target.com:443 -ssl3 2>&1 | grep -E "Protocol|CONNECTED|alert"
SSL3_GET_RECORD:wrong version number   ← SSLv3 correctly rejected
```

TLS 1.0 and 1.1 are deprecated (RFC 8996, March 2021). Their presence means the server is behind on security patching. TLS 1.0 enables the BEAST attack when paired with CBC cipher suites.

## 2.4 HTTP/2 and ALPN negotiation

ALPN (Application-Layer Protocol Negotiation) is a TLS extension where the client advertises supported protocols (`h2` for HTTP/2, `http/1.1` for HTTP/1.1) during the TLS handshake. The server responds with its preference. This determines whether HTTP/2 is supported and active.

```bash
# Check ALPN / HTTP/2 support
$ openssl s_client -connect corp-target.com:443 -alpn "h2,http/1.1" 2>&1 | grep ALPN
ALPN protocol: h2         ← HTTP/2 is supported and preferred

# curl HTTP/2 test
$ curl -sk --http2 -I https://corp-target.com/ | grep "HTTP/"
HTTP/2 200              ← server responded with HTTP/2

$ curl -sk --http2-prior-knowledge https://corp-target.com/ -I 2>&1 | head -3

# nmap HTTP/2 detection
$ nmap --script http2-request -p 443 203.0.113.45

# HTTP/2 matters for attack surface because:
# - h2c (HTTP/2 cleartext) over non-TLS port may be accessible → bypass of HTTPS-only enforcement
# - Some WAFs handle HTTP/2 differently from HTTP/1.1 → WAF bypass potential
$ curl -sk --http2-prior-knowledge http://corp-target.com:80/ -I
# If 200 → h2c (cleartext HTTP/2) is enabled — WAF may not inspect these requests
```

## 2.5 Certificate validity and chain analysis

Beyond the SANs, the certificate itself contains information about the organization's PKI management maturity:

```bash
# Full certificate inspection
$ openssl s_client -connect corp-target.com:443 -servername corp-target.com </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -E "Issuer|Subject:|Not Before|Not After|Signature"

Issuer: C=US, O=Let's Encrypt, CN=R3
Subject: CN=corp-target.com
Not Before: Jan  1 00:00:00 2024 GMT
Not After : Apr  1 00:00:00 2024 GMT     ← expires in 3 months
Signature Algorithm: sha256WithRSAEncryption

# Days until expiry check
$ echo | openssl s_client -connect corp-target.com:443 2>/dev/null \
  | openssl x509 -noout -enddate | cut -d= -f2 \
  | xargs -I{} date -d "{}" +%s \
  | xargs -I{} sh -c 'echo $(( ({} - $(date +%s)) / 86400 )) days remaining'
89 days remaining

# Certificate fingerprint (for Censys pivot from Stage 2)
$ openssl s_client -connect corp-target.com:443 </dev/null 2>/dev/null \
  | openssl x509 -noout -fingerprint -sha256 | cut -d= -f2
AB:CD:EF:12:34:56:...

# Check certificate chain completeness
$ openssl s_client -connect corp-target.com:443 -verify_return_error 2>&1 | grep -E "verify|Verify"
Verify return code: 0 (ok)    ← complete, valid chain
# Or:
depth=0 CN=corp-target.com
verify error:num=20:unable to get local issuer certificate   ← incomplete chain (missing intermediate)
```

An incomplete certificate chain (missing intermediate CA) causes browser warnings for some clients. The issuer being Let's Encrypt vs. a commercial CA (DigiCert, Sectigo) indicates organizational maturity: Let's Encrypt is free and automated, used by smaller operations or automation-mature teams. Self-signed certificates indicate development servers or internal tools.

## 2.6 HSTS and redirect behavior

HTTP Strict Transport Security (HSTS) instructs browsers to only connect via HTTPS for a specified duration. Its presence, scope, and `max-age` reveal the security posture of the web server.

```bash
# Check HSTS header
$ curl -sk -I https://corp-target.com/ | grep -i hsts
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

# Redirect behavior — does HTTP redirect to HTTPS?
$ curl -sk -I http://corp-target.com/ | grep -E "HTTP/|Location"
HTTP/1.1 301 Moved Permanently
Location: https://corp-target.com/    ← HTTP redirects to HTTPS (correct)

# Does HTTPS redirect to www?
$ curl -sk -I https://corp-target.com/ | grep -E "HTTP/|Location"
HTTP/2 301
Location: https://www.corp-target.com/   ← canonical redirect

# Missing HSTS header (finding):
$ curl -sk -I https://corp-target.com/ | grep -i strict
# (empty) — no HSTS header, HTTP is also served without redirect
```

Missing HSTS means: (1) HTTP access is possible, (2) a network-level attacker can downgrade an HTTPS user to HTTP, (3) HSTS preload is not configured. If the server serves both HTTP and HTTPS without redirecting, content may be accessible over unencrypted HTTP — all traffic is visible to any network observer.

## 2.7 Mass TLS scan with tlsx across all discovered hosts

After collecting all discovered hostnames and IPs, running tlsx across them all extracts the complete certificate inventory in a single pass:

```bash
# Collect all web-serving IPs and domains
$ cat confirmed_alive.txt all_dns_hosts.txt \
  | sort -u > all_web_targets.txt

# Mass TLS scan
$ cat all_web_targets.txt | tlsx \
    -san -cn -so -tls-version \
    -silent -json \
    -o tls_inventory.json

# Parse results
$ cat tls_inventory.json | jq -r '{
    host: .host,
    cn: .subject_cn,
    sans: .subject_an,
    tls_ver: .tls_version,
    issuer: .issuer_cn,
    expiry: .not_after
  }'

# Find all hosts with TLS 1.0 or 1.1 still enabled
$ cat tls_inventory.json | jq -r 'select(.tls_version == "tls10" or .tls_version == "tls11") | .host'

# Find all unique SANs across all hosts (complete host inventory from certs)
$ cat tls_inventory.json | jq -r '.subject_an[]?' | sort -u > all_cert_sans.txt
```

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **SAN (Subject Alternative Name)** | X.509 certificate extension listing all valid domains/IPs for the certificate |
| **ALPN** | TLS extension for negotiating the application protocol (h2 vs http/1.1) during the handshake |
| **Cipher suite** | The combination of key exchange, authentication, encryption, and MAC algorithms used in a TLS session |
| **Forward secrecy (PFS)** | Property where session keys are not derivable from the server's private key — ECDHE/DHE key exchange provides this |
| **TLS 1.0/1.1** | Deprecated TLS versions (RFC 8996); vulnerable to BEAST and other attacks |
| **BEAST** | Browser Exploit Against SSL/TLS (CVE-2011-3389) — decrypts TLS 1.0 sessions using CBC ciphers |
| **SWEET32** | Birthday attack against 3DES (CVE-2016-2183) — after 32GB of data, session key can be recovered |
| **HSTS** | HTTP Strict Transport Security — header instructing browsers to always use HTTPS |
| **Certificate chain** | The sequence of certificates from server cert → intermediate CA → root CA; incomplete chains cause browser errors |
| **Let's Encrypt** | Free, automated, open certificate authority; issues 90-day DV certificates |
| **testssl.sh** | Comprehensive TLS auditing script — protocol versions, cipher suites, vulnerability tests |
| **tlsx** | Fast mass TLS scanner from ProjectDiscovery — extracts SANs, versions, and metadata at scale |
| **h2c** | HTTP/2 cleartext — HTTP/2 over non-TLS connection; may be enabled on port 80 and bypass WAF inspection |
| **Certificate pinning** | Mobile/thick client technique where the application validates a specific cert hash rather than trusting any CA-signed cert |
| **OCSP stapling** | Server includes a pre-fetched OCSP response in the TLS handshake, reducing latency for certificate revocation checks |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `openssl s_client` | `openssl s_client -connect host:443 -servername host` | Certificate, cipher, protocol | Manual inspection |
| `testssl.sh` | `testssl.sh https://host` | Ciphers, protocols, vulns | Full TLS audit |
| `tlsx` | `echo "host" \| tlsx -san -silent` | SANs, version, expiry | Mass scanning |
| `nmap ssl-*` | `nmap --script ssl-cert,ssl-enum-ciphers -p 443 host` | Cert + cipher enum | nmap-integrated |
| `curl --http2` | `curl -sk --http2 -I https://host` | HTTP/2 support | Protocol detection |

**Full TLS surface audit for one host:**
```bash
HOST="corp-target.com"

# Cipher suite + protocol + vulnerability scan
$ testssl.sh --color 0 --jsonfile tls_audit.json https://$HOST

# Extract SANs
$ openssl s_client -connect $HOST:443 -servername $HOST </dev/null 2>/dev/null \
  | openssl x509 -noout -text | grep -A 5 "Subject Alternative Name"

# Days to expiry
$ echo | openssl s_client -connect $HOST:443 2>/dev/null \
  | openssl x509 -noout -enddate

# HSTS check
$ curl -sk -I https://$HOST/ | grep -i strict

# HTTP/2 check
$ curl -sk --http2 -I https://$HOST/ | head -1

# h2c on port 80 check
$ curl -sk --http2-prior-knowledge http://$HOST:80/ -o /dev/null -w "%{http_code}\n"

# HTTP redirect check
$ curl -sk -I http://$HOST/ | grep -E "HTTP/|Location"
```

**nmap TLS scripts:**
```bash
$ nmap --script ssl-cert,ssl-enum-ciphers,ssl-dh-params,ssl-heartbleed -p 443 203.0.113.45

PORT    STATE SERVICE
443/tcp open  https
| ssl-cert:
|   Subject: commonName=corp-target.com
|   Subject Alternative Name: DNS:corp-target.com, DNS:admin.corp-target.com, DNS:staging.corp-target.com
|   Not valid after: 2024-04-01T00:00:00
| ssl-enum-ciphers:
|   TLSv1.0:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA - A          ← ECDHE = PFS ✓
|       TLS_RSA_WITH_3DES_EDE_CBC_SHA - C                ← 3DES = SWEET32 ✗
|   TLSv1.2:
|     ciphers:
|       TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 - A       ← Strong
|_  least strength: C
| ssl-heartbleed:
|_  VULNERABLE: Heartbleed                               ← CRITICAL CVE!
```

---

# Section 5 — Defender detection

- **TLS handshake connections:** Every TLS probe (testssl.sh, tlsx, openssl s_client) completes or attempts a TLS handshake. The server's TLS stack logs each connection with source IP, timestamp, SNI (Server Name Indication) hostname requested, and negotiated cipher suite.
- **testssl.sh detection:** testssl.sh sends a large number of TLS ClientHello packets with different cipher suites and protocol versions in rapid succession. This is clearly identifiable in TLS session logs — no legitimate browser changes cipher suite preferences 50 times in a row from the same IP.
- **Cipher-forcing probes:** Forcing specific cipher suites with `-cipher` in openssl creates unusual ClientHello messages with non-standard cipher orderings. TLS inspection devices (SSL inspection proxies, WAF) may flag these as anomalous.
- **SNI probing:** When you probe a host using an IP address with different `-servername` values, each unique SNI value appears in the TLS log. Probing 20 different domain names against the same IP in quick succession is a vhost and SAN enumeration signature.
- **Mitigation for operators:** (1) Use tlsx with `-c` (concurrency) limited to 5-10 for mass scanning. (2) Add delays between testssl.sh probes for multiple hosts. (3) Prefer reading SAN data from CT logs (passive, crt.sh) before actively connecting. Active TLS probing only for hosts that CT logs didn't cover.

---

# Section 6 — Lab task

**Platform:** Kali Linux. Target: any HTTPS-serving public host you're authorized to examine (hackthebox.com, badssl.com — specifically designed for TLS testing).

**Objective:** Complete a full TLS surface audit including SANs, cipher suites, protocol versions, HSTS, and HTTP/2 support.

**Steps:**

1. **SAN harvest:** `openssl s_client -connect badssl.com:443 -servername badssl.com </dev/null 2>/dev/null | openssl x509 -noout -text | grep -A 20 "Subject Alternative Name"`
2. **Expiry check:** `echo | openssl s_client -connect badssl.com:443 2>/dev/null | openssl x509 -noout -enddate`
3. **Full testssl.sh audit:** `testssl.sh https://badssl.com | tee testssl_results.txt`
4. **TLS 1.0 test:** `openssl s_client -connect badssl.com:443 -tls1 2>&1 | grep -E "Protocol|handshake"`
5. **TLS 1.3 test:** `openssl s_client -connect badssl.com:443 -tls1_3 2>&1 | grep Protocol`
6. **HSTS header:** `curl -sk -I https://badssl.com/ | grep -i strict`
7. **HTTP redirect:** `curl -sk -I http://badssl.com/ | grep -E "HTTP/|Location"`
8. **HTTP/2 support:** `curl -sk --http2 -I https://badssl.com/ | head -1`
9. **nmap TLS scripts:** `nmap --script ssl-cert,ssl-enum-ciphers,ssl-heartbleed -p 443 badssl.com`
10. **Compile `tls_surface.md`:** Table with: SANs found | TLS versions accepted | Cipher grade (A-F from testssl) | HSTS present | HTTP/2 supported | Vulnerabilities found | Certificate expiry

```bash
git commit -m "recon(stage3): TLS surface audit — cipher/protocol/cert inventory for <target>"
```

---

# Section 7 — Common mistakes

**1. Only checking port 443 for TLS**
_Why it matters:_ TLS runs on many ports: 465 (SMTPS), 993 (IMAPS), 995 (POP3S), 636 (LDAPS), 8443 (HTTPS alt), 4443. A server may have weak TLS only on the SMTP port, not the HTTPS port.
_Fix:_ After identifying all open ports from the port scan, run tlsx or testssl.sh against every port that served TLS in the nmap -sV output.

**2. Treating "no HSTS" as a low-finding**
_Why it matters:_ Without HSTS, a network attacker can downgrade an HTTPS user to HTTP via SSL stripping. This is a real attack (sslstrip) that works against users on the same network segment. No HSTS = HTTPS is not enforced.
_Fix:_ Document missing HSTS as medium-severity finding: "HTTP Strict Transport Security not configured — HTTP downgrade attack possible via network interception."

**3. Not comparing SAN hosts against already-discovered inventory**
_Why it matters:_ SANs in a certificate are frequently the most complete inventory of hosts associated with that server. Operators who extract SANs but don't compare them to their existing host list miss new discoveries.
_Fix:_ After extracting SANs, diff them against `all_dns_hosts.txt`. New entries are previously unknown hosts — add them to the target scope and scan them.

**4. Confusing cipher suite grade with actual exploitability**
_Why it matters:_ testssl.sh grades a cipher suite as "F" for being theoretically weak. "Theoretically weak" and "actively exploitable today" are not the same thing. 3DES SWEET32 requires 32GB of data in a single session — practically very difficult.
_Fix:_ Document cipher weaknesses with their actual exploit complexity. Grade "F" is a finding but must be reported with practical context.

**5. Skipping the HTTP/2 cleartext (h2c) check**
_Why it matters:_ Some servers enable HTTP/2 on port 80 (h2c — cleartext). WAFs may inspect HTTP/1.1 traffic but pass h2c traffic uninspected. A WAF bypass via h2c can make previously blocked attack payloads succeed.
_Fix:_ Always check `curl --http2-prior-knowledge http://<host>:80/` after confirming HTTP/2 support on HTTPS. If port 80 responds with HTTP/2, test WAF bypass via h2c.

---

# Section 8 — Move-on gate

1. You extract the SAN from a certificate and find `staging.corp-target.com` listed alongside the production domain. Without notes, describe two reasons this is a high-value finding and state exactly what you would check first on the staging host.

2. testssl.sh reports `TLS 1.0: offered` and `3DES cipher: offered (VULNERABLE - SWEET32)`. Explain what SWEET32 is, why TLS 1.0 presence compounds it, and what the practical exploitation requirement is (why it's not trivially exploitable).

3. A target serves HTTPS on port 443 with only TLS 1.2 and TLS 1.3 and strong AEAD ciphers. curl returns HTTP/2 on port 443. A check of port 80 with `--http2-prior-knowledge` returns a 200 response. What is the security significance of this finding and what attack does it enable?
