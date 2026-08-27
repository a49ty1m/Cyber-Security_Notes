# Passive DNS & Certificate Transparency (CT)

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Passive DNS and Certificate Transparency (CT) reconnaissance identify infrastructure that may not appear in normal DNS enumeration.

Passive DNS exposes historical and observed DNS relationships.
CT exposes publicly logged TLS certificates, including names in SANs.
Together they can reveal forgotten subdomains, old infrastructure, staging systems, and shared hosting relationships.

```text
Recon
 └── Passive
      ├── DNS / WHOIS
      ├── Passive DNS + CT  ← THIS TOPIC
      ├── Metadata / OSINT
      └── Human Profiling
           ↓
      Active Enumeration
           ↓
      Validation
           ↓
      Attack Surface Mapping
```

Skipping this creates blind spots.
A current DNS zone may show only production hosts.
Historical DNS or certificates may expose older systems.

The previous reconnaissance work establishes the domain and ownership context.
This topic expands that map before active probing begins.

# Section 2 — How attackers actually use this

## 2.1 Find names that normal enumeration misses

Attackers first look for:

* `dev.example.com`
* `staging.example.com`
* `api.example.com`
* `vpn.example.com`
* `old.example.com`
* `internal.example.com`
* `mail.example.com`
* wildcard certificate names
* historical hostnames

The important question is not:
"How many subdomains exist?"

It is:
"Which infrastructure names reveal something about the target?"

A certificate may contain a hostname that is not currently visible in the target's public DNS.

A passive DNS record may show an address previously associated with a hostname.

## 2.2 Build a historical infrastructure picture

The attacker compares:

```text
Current DNS
      +
Passive DNS
      +
Certificate names
      ↓
Historical asset set
```

For every discovered hostname, they want:

* hostname
* record type
* IP address
* historical IP
* ASN
* hosting provider
* country
* certificate relationship
* first/last observation where available

This changes reconnaissance from a simple hostname list into an infrastructure graph.

## 2.3 Cluster infrastructure

Suppose the results contain:

```text
api.example.com       → 203.0.113.20
portal.example.com    → 203.0.113.21
dev.example.com       → 198.51.100.40
vpn.example.com       → 192.0.2.50
```

The first two addresses may belong to the same provider and ASN.

The third may belong to a cloud provider.

The fourth may belong to a completely different network.

That distinction matters.

A cluster can reveal:

```text
example.com
 ├── Production
 │    ├── api
 │    └── portal
 ├── Cloud / Development
 │    └── dev
 └── Remote Access
      └── vpn
```

The attacker now has hypotheses about different trust boundaries.

## 2.4 Pivot from certificates

Certificates can expose relationships that DNS does not.

For example:

```text
Certificate
 ├── example.com
 ├── www.example.com
 ├── api.example.com
 ├── staging.example.com
 └── legacy.example.com
```

`legacy.example.com` is interesting even if it no longer resolves.

It becomes a historical lead.

The attacker then checks whether the hostname:

* existed historically
* still resolves
* points to a different provider
* shares infrastructure with production
* appears in other certificates

## 2.5 Pivot from passive DNS

A historical DNS relationship can reveal infrastructure migration.

Example:

```text
api.example.com
  ↓
198.51.100.20
  ↓
old hosting provider

Current:
api.example.com
  ↓
203.0.113.20
  ↓
new cloud provider
```

The old address becomes an infrastructure-history clue.

It may identify:

* previous hosting
* forgotten services
* old naming conventions
* migration patterns
* abandoned assets

## 2.6 Dead-end versus high-value finding

Dead-end:

```text
www.example.com
→ current production IP
→ expected CDN
```

This confirms normal infrastructure but adds little new information.

High-value:

```text
staging-admin.example.com
→ historical IP
→ separate ASN
→ certificate observed
→ hostname absent from current DNS
```

The second finding matters because it suggests an asset outside the obvious production surface.

It becomes a candidate for later validation.

## 2.7 Feed results into the next phase

The final output should become:

```text
Hostname
   ↓
Resolution
   ↓
IP
   ↓
ASN / Provider
   ↓
Country
   ↓
Certificate relationship
   ↓
Priority
```

High-confidence assets move into active enumeration.

Examples:

```text
staging.example.com → HTTP probing
vpn.example.com     → service enumeration
api.example.com     → API discovery
legacy.example.com  → historical validation
```

Passive discovery does not prove that an asset is reachable.

It produces targets for validation.

# Section 3 — Core concepts and terminology

| Term                     | Meaning                                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------------------- |
| Passive DNS              | Historical or observed DNS data collected without directly querying the target's authoritative infrastructure |
| DNS record               | A DNS entry mapping a name to information such as an IP address                                               |
| A record                 | Maps a hostname to an IPv4 address                                                                            |
| AAAA record              | Maps a hostname to an IPv6 address                                                                            |
| CNAME                    | Alias pointing one hostname to another                                                                        |
| Certificate Transparency | Public logging system for issued TLS certificates                                                             |
| CT log                   | Public append-only record containing certificate issuance information                                         |
| SAN                      | Subject Alternative Name; certificate field containing additional DNS names                                   |
| CN                       | Common Name; older certificate naming field                                                                   |
| crt.sh                   | Public interface for searching Certificate Transparency records                                               |
| Subdomain                | Hostname below a parent domain                                                                                |
| Passive enumeration      | Discovery using existing public information rather than direct target probing                                 |
| Historical DNS           | DNS relationships observed in the past                                                                        |
| ASN                      | Autonomous System Number identifying a routed network                                                         |
| Hosting provider         | Organization providing infrastructure or network hosting                                                      |
| Infrastructure cluster   | Related assets grouped by network, provider, ASN, IP, or naming relationships                                 |
| Shadow subdomain         | Undocumented or overlooked subdomain discovered through external evidence                                     |
| SAN discovery            | Extracting hostnames from certificate SAN fields                                                              |
| DNS pivot                | Using one DNS finding to discover related infrastructure                                                      |
| CT pivot                 | Using certificate relationships to discover additional names                                                  |
| Historical pivot         | Using old infrastructure information to discover additional assets                                            |

### Common evidence types

| Evidence            | Typical value                                        |
| ------------------- | ---------------------------------------------------- |
| Current A record    | Current infrastructure                               |
| Historical A record | Previous infrastructure                              |
| CNAME               | Hosting relationship                                 |
| CT SAN              | Additional hostname                                  |
| ASN                 | Network ownership                                    |
| Country             | Geographic infrastructure grouping                   |
| Provider            | Hosting relationship                                 |
| Certificate overlap | Shared infrastructure or organizational relationship |

# Section 4 — Tools and commands

| Tool          | Command                                                                                        | What it finds/shows                      | When to use it                 |
| ------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------- | ------------------------------ |
| `subfinder`   | `subfinder -d example.com -silent`                                                             | Passive subdomains from multiple sources | Fast passive discovery         |
| `amass`       | `amass enum -passive -d example.com`                                                           | Passive DNS/subdomain relationships      | Broader passive enumeration    |
| `curl` + `jq` | `curl -s 'https://crt.sh/?q=%25.example.com&output=json' \| jq -r '.[].name_value' \| sort -u` | CT certificate names                     | Certificate discovery          |
| `dig`         | `dig example.com A`                                                                            | Current DNS resolution                   | Validate a discovered hostname |
| `host`        | `host api.example.com`                                                                         | Simple DNS resolution                    | Quick validation               |
| `whois`       | `whois 203.0.113.20`                                                                           | Network/registration information         | Investigate an IP              |
| `asnmap`      | `asnmap -d example.com`                                                                        | ASN relationships                        | Infrastructure clustering      |
| `dnsx`        | `printf '%s\n' api.example.com dev.example.com \| dnsx -silent`                                | Resolving discovered names               | Validate candidates            |

### `subfinder`

```text
$ subfinder -d example.com -silent
api.example.com
mail.example.com
dev.example.com
```

Interpretation:
Each line is a candidate hostname.

Do not assume every result is live.

### `amass`

```text
$ amass enum -passive -d example.com
api.example.com
vpn.example.com
legacy.example.com
```

Interpretation:
Passive Amass results can reveal relationships missed by a single source.

### `crt.sh`

```text
$ curl -s 'https://crt.sh/?q=%25.example.com&output=json' \
  | jq -r '.[].name_value' \
  | sort -u
```

Example:

```text
api.example.com
example.com
legacy.example.com
staging.example.com
www.example.com
```

Interpretation:
These names appeared in publicly logged certificates.

They are leads, not proof of current reachability.

### `dig`

```text
$ dig api.example.com A +short
203.0.113.20
```

Interpretation:
The hostname currently resolves to an IPv4 address.

An empty response means it currently has no returned A record.

### `host`

```text
$ host staging.example.com
staging.example.com has address 198.51.100.40
```

Interpretation:
The hostname currently resolves.

### `whois`

```text
$ whois 198.51.100.40
...
NetName: EXAMPLE-NETWORK
...
```

Interpretation:
Use registration/network information as infrastructure context.

Do not treat WHOIS ownership alone as proof of application ownership.

### `asnmap`

```text
$ asnmap -d example.com
203.0.113.0/24
```

Interpretation:
The result can help associate discovered infrastructure with network ownership.

### `dnsx`

```text
$ printf '%s\n' api.example.com staging.example.com \
  | dnsx -silent
api.example.com
staging.example.com
```

Interpretation:
Returned names successfully resolved during the check.

## Minimal passive workflow

```text
1. Discover subdomains
2. Extract CT names
3. Deduplicate names
4. Resolve candidates
5. Record IPs
6. Map ASN/provider
7. Group related assets
8. Prioritize unusual findings
9. Pass candidates to active validation
```

## Useful evidence table

```text
hostname              source       IP              ASN       priority
api.example.com       DNS          203.0.113.20    AS64500   normal
staging.example.com   CT           198.51.100.40   AS64510   high
legacy.example.com    CT/history   unknown         unknown   medium
vpn.example.com       passive DNS  192.0.2.50      AS64520   high
```

# Section 5 — Defender detection

* Passive DNS collection is generally invisible to the target because the analyst is querying third-party datasets rather than the target's DNS servers.
* CT searches normally generate no application-server event because certificate logs are publicly accessible.
* DNS validation performed later can appear in authoritative DNS query telemetry.
* Defenders commonly miss certificates containing forgotten development or staging names.
* Certificate issuance monitoring can detect unexpected new names before attackers discover them elsewhere.
* DNS analytics can identify unusual resolver behavior when reconnaissance transitions from passive collection to direct resolution.
* Skilled operators reduce footprint by separating passive collection from later validation and minimizing direct queries during the Ghost phase.

# Section 6 — Lab task

**Platform:** Kali Linux VM using a domain you own/control and its publicly issued certificates.

**Objective:** Prove that you can combine passive subdomain discovery, CT SAN discovery, DNS validation, and ASN clustering into one evidence set.

**Steps:**

1. Create a working directory named `passive-dns-ct`.
2. Run passive subdomain discovery against your authorized domain.
3. Query CT records and extract unique certificate names.
4. Combine and deduplicate both hostname lists.
5. Resolve the discovered hostnames and record successful results.
6. Map the resulting IPs to network/ASN information.
7. Separate current production assets from historical or unusual findings.
8. Save the final evidence table with source, hostname, IP, ASN, and priority.
9. Identify one dead-end result and one high-value result.
10. Commit the finished evidence to Git.

**Expected output:**

```text
passive-dns-ct/
├── README.md
├── data/
│   ├── subdomains.txt
│   ├── ct-names.txt
│   └── resolved.txt
└── evidence/
    └── infrastructure.md
```

`infrastructure.md` should contain:

```text
Hostname
Discovery source
Current IP
ASN/provider
Country
Historical indication
Priority
Reason for priority
```

**Git artifact:**

```bash
git add passive-dns-ct/
git commit -m "Add passive DNS and CT reconnaissance evidence"
```

# Section 7 — Common mistakes

1. **Treating CT names as live assets**
   → Certificates can contain retired names.
   → Validate current DNS separately.

2. **Using only one passive source**
   → One dataset can miss historical or provider-specific observations.
   → Compare multiple passive sources.

3. **Ignoring historical results**
   → Old infrastructure can reveal naming and hosting history.
   → Preserve historical evidence separately from current assets.

4. **Assuming every subdomain belongs to production**
   → Development, SaaS, CDN, and third-party infrastructure can coexist.
   → Cluster by IP, ASN, provider, and purpose.

5. **Failing to deduplicate SAN results**
   → CT searches often return repeated names across certificates.
   → Normalize and sort unique hostnames.

6. **Jumping directly into active scanning**
   → You lose the advantage of passive reconnaissance and create unnecessary traffic.
   → Build the candidate set first, then validate selectively.

7. **Recording only hostnames**
   → A hostname without infrastructure context is weak intelligence.
   → Record DNS, IP, ASN, provider, source, and priority together.

# Section 8 — Move-on gate

1. **Run passive discovery against your lab domain and identify at least five candidate hostnames, then explain which source produced each result without looking at your notes.**

2. **Extract CT names from the domain, identify one SAN-derived hostname that is absent from your initial subdomain list, and correctly determine whether it currently resolves.**

3. **Take three discovered IP addresses, map their network/ASN relationships, and correctly group them into infrastructure clusters before moving to active enumeration.**
