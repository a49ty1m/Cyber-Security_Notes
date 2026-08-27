# Domain & Ownership

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Domain and ownership reconnaissance identifies who manages a domain, which registrar and registry are involved, when the domain was created or updated, which authoritative name servers it uses, and what registration data is publicly exposed. Traditionally this was done with WHOIS; for modern gTLD registration data, RDAP is now the preferred structured protocol. WHOIS/RDAP does **not** directly enumerate subdomains. Subdomains are normally discovered through DNS, Certificate Transparency, search indexes, and other passive sources.

The output is an ownership/infrastructure profile that can be correlated with later reconnaissance. The useful question is not simply "who owns this domain?" but "what does this registration data tell me about the organisation's infrastructure, providers, history, and likely next pivots?"

```text
Reconnaissance
├── Passive
│   ├── Domain & Ownership ← THIS TOPIC
│   │   ├── WHOIS / RDAP
│   │   ├── Registrar
│   │   ├── Registration dates
│   │   └── Name servers
│   ├── DNS intelligence
│   ├── Certificate Transparency
│   └── Search / OSINT
└── Active
    ├── Port scanning
    ├── Service enumeration
    └── Web probing
```

Skipping this skill can cause you to miss infrastructure relationships before touching the target, misidentify third-party providers, waste time attacking assets outside scope, or overlook useful historical clues.

It follows the broader passive-footprinting work and feeds the next stage by providing domain, registrar, DNS, certificate, and infrastructure pivots.

# Section 2 — How attackers actually use this

## 2.1 Establish the domain's ownership profile

An operator starts with the exact domain rather than immediately scanning hosts.

They want to determine:

* Registrar
* Registry
* Creation date
* Last-update date
* Expiration date
* Domain status
* Authoritative name servers
* Registrant organisation, if exposed
* Registrant contact information, if exposed
* Registrar abuse contact
* Registry/domain identifiers
* Historical ownership or infrastructure changes where legally available

The first useful distinction is **ownership evidence vs infrastructure evidence**.

For example:

```text
Target: example.com

Registrar: Example Registrar
Created:   2019-04-12
Updated:   2026-02-03
Expires:   2027-04-12
NS:        ns1.provider.example
           ns2.provider.example
Status:    clientTransferProhibited
```

This does not prove that `ns1.provider.example` belongs to the target. It tells the operator which DNS provider is involved and creates a pivot for understanding the target's infrastructure.

## 2.2 Extract dates as intelligence

Registration dates are more useful when correlated with other findings.

A very old domain with recent infrastructure changes can indicate:

```text
Old domain
   ↓
Recent WHOIS/RDAP update
   ↓
Recent name-server change
   ↓
New DNS provider
   ↓
Potential infrastructure migration
```

A newly registered domain associated with an established organisation may deserve additional scrutiny because it could represent:

* A new project
* A marketing campaign
* A development environment
* A newly acquired company
* A temporary operational domain
* An infrastructure migration

The date alone is never proof of any of these.

## 2.3 Use the registrar as a pivot

The registrar tells the operator where domain-management infrastructure sits.

A useful workflow is:

```text
Domain
  ↓
Registrar
  ↓
Registration metadata
  ↓
Name servers
  ↓
DNS provider
  ↓
DNS records
  ↓
Certificates
  ↓
Subdomains / hosts
```

The registrar can also reveal patterns across related domains. If several known company domains use the same registrar and similar registration periods, that correlation can help establish that they belong to the same infrastructure set.

The operator should distinguish:

```text
Registrar ≠ DNS provider ≠ Hosting provider
```

They are often different organisations.

## 2.4 Treat name servers as infrastructure pivots

Name servers are one of the most useful outputs because they connect registration information to DNS infrastructure.

For example:

```text
Target
└── example.com
    ├── Registrar: Registrar-A
    └── NS:
        ├── ns1.dns-provider.net
        └── ns2.dns-provider.net
```

The next question is not "scan the name server."

The better passive question is:

> What does this provider relationship tell me about the target's DNS architecture?

That can lead to:

* DNS provider identification
* Hosted-zone relationships
* CDN identification
* Mail-provider identification
* DNS configuration analysis
* Passive discovery of related domains

## 2.5 Understand what privacy redaction changes

Modern registration data frequently looks like:

```text
Registrant Name: REDACTED
Registrant Organization: Privacy Service
Registrant Email: REDACTED
```

This is not a failed lookup.

The useful data may still include:

* Registrar
* Creation date
* Update date
* Expiration date
* Name servers
* Domain status
* Registry identifier
* Abuse contact

An operator therefore does not stop because personal information is hidden.

They shift from **identity extraction** to **infrastructure correlation**.

## 2.6 Separate WHOIS from subdomain discovery

This is a critical distinction.

WHOIS/RDAP tells you about the domain registration.

It normally does **not** return:

```text
admin.example.com
dev.example.com
vpn.example.com
api.example.com
```

Those are discovered through other passive sources.

A realistic pivot looks like:

```text
WHOIS/RDAP
    ↓
Name servers + registrar
    ↓
DNS intelligence
    ↓
Certificate Transparency
    ↓
Subdomain candidates
    ↓
IP / ASN / hosting correlation
    ↓
Active enumeration
```

Certificate Transparency is particularly valuable because certificates can contain multiple names in their Subject Alternative Name fields.

## 2.7 Dead-end vs high-value finding

### Dead-end

```text
Registrant: Privacy Protected
Email:      Redacted
Address:    Redacted
```

This provides little direct identity information.

It is not useless, but continuing to repeatedly search for the hidden person's identity wastes time.

### High-value

```text
Creation Date: 2018-07-05
Registrar:     Registrar-A
Name Server:   ns1.dns-provider.net
Name Server:   ns2.dns-provider.net
Status:        clientTransferProhibited
```

This provides multiple pivots:

```text
Registrar
   ├── registration history
   └── related domains

Name servers
   ├── DNS provider
   └── infrastructure relationships

Dates
   └── historical change correlation

Status
   └── domain-management posture
```

## 2.8 Feed the findings into the next phase

The final output should become a small intelligence record rather than a copied WHOIS dump.

```text
Domain
├── Ownership
│   ├── Registrar
│   └── Registration dates
├── DNS
│   └── Authoritative NS
├── Infrastructure
│   └── Provider relationships
└── Passive pivots
    ├── CT certificates
    ├── Subdomains
    └── Related domains
```

The next stage can then investigate DNS records and passive subdomain sources before any active probing occurs.

# Section 3 — Core concepts and terminology

| Term                     | Plain meaning                                                                    |
| ------------------------ | -------------------------------------------------------------------------------- |
| WHOIS                    | Legacy query/response system for domain registration information.                |
| RDAP                     | Modern, HTTPS-based, structured replacement for WHOIS.                           |
| Registrar                | Company through which a domain is registered.                                    |
| Registry                 | Operator responsible for maintaining registration data for a TLD.                |
| Registrant               | Person or organisation holding registration rights to a domain.                  |
| Registration date        | Date the domain was originally registered.                                       |
| Updated date             | Date registration information was last changed.                                  |
| Expiration date          | Date the current registration period ends.                                       |
| Name server              | DNS server authoritative for a domain or zone.                                   |
| DNS                      | System that maps names such as `example.com` to network information.             |
| Domain status            | EPP state describing restrictions or conditions on a domain.                     |
| Privacy service          | Service that replaces or hides public registrant information.                    |
| RDAP endpoint            | HTTPS API endpoint providing structured registration data.                       |
| TLD                      | Top-level domain such as `.com`, `.org`, or `.in`.                               |
| gTLD                     | Generic top-level domain such as `.com` or `.org`.                               |
| Subdomain                | Domain component below a parent domain, such as `vpn.example.com`.               |
| Certificate Transparency | Public system recording issued TLS certificates.                                 |
| SAN                      | Subject Alternative Name field containing names covered by a certificate.        |
| Passive reconnaissance   | Intelligence gathering without directly probing the target's infrastructure.     |
| DNS provider             | Organisation hosting or operating DNS services for a domain.                     |
| Registrar lock           | Domain status preventing certain unauthorised registration changes or transfers. |

| Data source              | Primary value                      | Subdomain discovery |
| ------------------------ | ---------------------------------- | ------------------- |
| WHOIS                    | Registration metadata              | Usually no          |
| RDAP                     | Structured registration metadata   | Usually no          |
| DNS                      | Current DNS records                | Limited             |
| Certificate Transparency | Issued certificate names           | Yes                 |
| Search engines           | Indexed public information         | Sometimes           |
| Passive OSINT databases  | Historical/correlated intelligence | Often               |

# Section 4 — Tools and commands

| Tool             | Command                                                                                        | What it finds/shows                                               | When to use it                    |
| ---------------- | ---------------------------------------------------------------------------------------------- | ----------------------------------------------------------------- | --------------------------------- |
| `whois`          | `whois example.com`                                                                            | Registrar, dates, status, name servers, available registrant data | Legacy WHOIS inspection           |
| `curl` + RDAP    | `curl -s https://rdap.verisign.com/com/v1/domain/example.com \| jq .`                          | Structured registration JSON                                      | Modern `.com` registration lookup |
| `dig`            | `dig example.com NS`                                                                           | Authoritative name servers                                        | Pivoting from ownership to DNS    |
| `dig`            | `dig example.com SOA`                                                                          | Zone authority and serial information                             | Understanding DNS authority       |
| `dig`            | `dig example.com MX`                                                                           | Mail servers                                                      | Identifying email infrastructure  |
| `dig`            | `dig example.com TXT`                                                                          | TXT records such as SPF and verification data                     | Passive infrastructure mapping    |
| `curl` + CT logs | `curl -s 'https://crt.sh/?q=%25.example.com&output=json' \| jq -r '.[].name_value' \| sort -u` | Certificate names and potential subdomains                        | Passive subdomain discovery       |

### `whois`

```text
$ whois example.com

Domain Name: EXAMPLE.COM
Registrar: Example Registrar
Creation Date: 1995-08-14T04:00:00Z
Registry Expiry Date: 2027-08-14T04:00:00Z
Name Server: NS1.EXAMPLE.NET
Name Server: NS2.EXAMPLE.NET
Domain Status: clientTransferProhibited
```

Interpretation:

* `Registrar` identifies the registration provider.
* `Creation Date` gives domain age.
* `Name Server` identifies DNS authority.
* `Domain Status` shows registration restrictions.

Do not assume the registrar or DNS provider owns the target organisation.

### RDAP with `curl`

```text
$ curl -s https://rdap.verisign.com/com/v1/domain/example.com | jq .

{
  "ldhName": "EXAMPLE.COM",
  "status": ["clientTransferProhibited"],
  "events": [
    {"eventAction": "registration", "eventDate": "..."},
    {"eventAction": "expiration", "eventDate": "..."}
  ]
}
```

Interpretation:

RDAP provides structured fields that are easier to parse automatically than traditional WHOIS text.

### `dig NS`

```text
$ dig example.com NS

example.com.  86400  IN  NS  ns1.example.net.
example.com.  86400  IN  NS  ns2.example.net.
```

Interpretation:

These are the authoritative DNS servers. They are a pivot into DNS infrastructure, not automatically attack targets.

### `dig SOA`

```text
$ dig example.com SOA

example.com. 86400 IN SOA ns1.example.net. hostmaster.example.net. 2026081801 ...
```

Interpretation:

The SOA exposes the primary authoritative server, administrative mailbox representation, and zone serial.

### `dig MX`

```text
$ dig example.com MX

example.com. 3600 IN MX 10 mail.example.net.
```

Interpretation:

The domain's mail infrastructure is represented by the MX record. The priority value determines preference between multiple mail exchangers.

### `dig TXT`

```text
$ dig example.com TXT

example.com. 3600 IN TXT "v=spf1 include:_spf.example.net -all"
```

Interpretation:

TXT records can reveal email-security configuration, verification services, and third-party SaaS relationships.

### Certificate Transparency query

```text
$ curl -s 'https://crt.sh/?q=%25.example.com&output=json' \
  | jq -r '.[].name_value' \
  | sort -u
```

Possible output:

```text
api.example.com
dev.example.com
example.com
mail.example.com
www.example.com
```

Interpretation:

These are certificate names, not proof that every hostname is currently live. They become candidates for subsequent DNS validation.

# Section 5 — Defender detection

* Traditional WHOIS/RDAP queries are generally **passive with respect to the target**, because the query is made against registration infrastructure rather than the target's application or host.
* There is normally no target-side HTTP, firewall, or IDS event generated simply because someone performs a public WHOIS/RDAP lookup.
* Registrars and registry operators may retain query/access telemetry, but this is different from telemetry on the target organisation's servers.
* Certificate Transparency monitoring can reveal new certificates and therefore help defenders identify unexpected or forgotten subdomains.
* Defenders commonly miss historical exposure: an old certificate, old DNS record, or old registration relationship can remain discoverable after infrastructure is retired.
* Skilled operators minimise attribution by relying on public registration, DNS, and CT datasets rather than directly querying target infrastructure.
* The strongest defensive control is reducing unnecessary public metadata and continuously monitoring registration, DNS, and certificate changes.

# Section 6 — Lab task

**Platform:** TryHackMe — **Passive Reconnaissance** room. It directly covers WHOIS, RDAP, DNS, and passive subdomain discovery.

**Objective:** Build a registration-and-DNS intelligence record for `tryhackme.com` and correctly distinguish WHOIS/RDAP ownership data from DNS and certificate-based subdomain data.

**Steps:**

1. Start the TryHackMe Passive Reconnaissance room and open the AttackBox or connect your Kali VM through the provided VPN.
2. Query the domain's registration information with the `whois` tool and save the raw result.
3. Identify the registrar, creation date, expiration date, status, and authoritative name servers.
4. Query the modern RDAP record and compare its structured events with the legacy WHOIS output.
5. Query the domain's NS, MX, SOA, and TXT records and record what each reveals.
6. Search Certificate Transparency data for names associated with the domain and deduplicate the discovered hostnames.
7. Separate your findings into `ownership`, `DNS`, and `subdomains`; do not label certificate names as confirmed live hosts.
8. Write a short assessment identifying the three most useful passive pivots discovered during the exercise.

**Expected output:**

```text
domain: tryhackme.com

ownership:
  registrar: [identified registrar]
  creation:  [identified date]
  expiration:[identified date]

dns:
  nameservers: [identified NS]
  mail:        [identified MX]

passive_subdomains:
  [deduplicated certificate/DNS names]

key_pivots:
  1. [finding]
  2. [finding]
  3. [finding]
```

**Git artifact:**

```text
recon/
└── domain-ownership/
    └── tryhackme.com/
        ├── whois.txt
        ├── rdap.json
        ├── dns.txt
        ├── ct-subdomains.txt
        └── notes.md
```

```bash
git add recon/domain-ownership/tryhackme.com/
git commit -m "Add passive domain ownership reconnaissance"
```

# Section 7 — Common mistakes

1. **Mistake:** Treating WHOIS as a subdomain-enumeration tool.
   **Why it matters:** Registration records normally do not list the target's subdomains.
   **Do instead:** Use DNS and Certificate Transparency as separate passive sources.

2. **Mistake:** Assuming the registrant must be publicly visible.
   **Why it matters:** Privacy services frequently redact personal registration data.
   **Do instead:** Extract registrar, dates, status, and name-server information that remains available.

3. **Mistake:** Confusing registrar, DNS provider, and hosting provider.
   **Why it matters:** They can be three completely different organisations.
   **Do instead:** Record each relationship separately.

4. **Mistake:** Treating a certificate name as a live host.
   **Why it matters:** Certificates can contain expired, unused, or historical names.
   **Do instead:** Treat CT results as candidates and validate them through passive DNS data first.

5. **Mistake:** Copying the entire WHOIS response without interpretation.
   **Why it matters:** A raw dump does not identify useful pivots.
   **Do instead:** Extract dates, registrar, status, NS, and meaningful historical clues.

6. **Mistake:** Assuming a recent update automatically means compromise.
   **Why it matters:** Registration changes happen for legitimate administrative reasons.
   **Do instead:** Correlate registration changes with DNS, certificate, and infrastructure changes.

# Section 8 — Move-on gate

1. **Run `whois` and RDAP against a permitted domain and correctly extract the registrar, creation date, expiration date, status, and authoritative name servers without looking at your notes.**

2. **Run passive DNS queries for NS, SOA, MX, and TXT records against the same domain and correctly explain what each returned record reveals.**

3. **Perform a Certificate Transparency lookup for the domain, produce a deduplicated subdomain list, and correctly separate certificate-discovered names from confirmed live hosts without looking at your notes.**
