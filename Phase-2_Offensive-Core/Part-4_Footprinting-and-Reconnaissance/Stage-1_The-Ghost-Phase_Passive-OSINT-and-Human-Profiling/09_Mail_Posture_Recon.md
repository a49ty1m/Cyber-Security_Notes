# Mail Posture Recon

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Mail posture recon is the passive examination of a domain's email-authentication controls — SPF, DKIM, and DMARC — along with its MX infrastructure. The output is a spoofing-risk profile: you learn whether the domain can be impersonated in email delivery, which providers send mail on its behalf, which third-party platforms are trusted by its mail infrastructure, and how strictly the domain enforces authentication policy against non-compliant senders.

Every finding comes from public DNS records. No packets reach the target's mail servers. No request touches the target's web infrastructure. This is fully passive reconnaissance.

```text
Recon
└── Passive
     ├── Domain & Ownership
     ├── Passive DNS & CT
     ├── Organizational & Social Profiling
     ├── Breach & Paste Monitoring
     └── Mail Posture Recon  ← THIS TOPIC
          ↓
     Semi-Passive / Active Recon
          ↓
     Initial Access
          ↓
     Delivery / Phishing → Post-Exploitation
```

Skipping this step conceals one of the most exploitable passive intelligence sources. If you later construct a phishing campaign or attempt email-based initial access, you need to know whether the target domain is spoofable, which sending infrastructure is trusted, and which mail security vendors are watching. That intelligence is available entirely through DNS — and collecting it generates zero target-side events.

This topic follows Domain & Ownership and Passive DNS work. By this point you have confirmed which domains and subdomains belong to the target organization. Mail posture recon consumes those confirmed domain names and converts them into email-infrastructure intelligence, provider fingerprints, and spoofing-risk verdicts — all before a single active probe begins.

---

# Section 2 — How attackers actually use this

## 2.1 What attackers are actually looking for

An attacker querying mail posture records is not simply confirming that the domain has email. They are answering five concrete questions: which infrastructure is authorized to send mail on behalf of the domain, how strictly spoofed messages will be rejected by receiving servers, which cloud mail providers and third-party SaaS platforms are in use, whether subdomain policy is weaker than the primary domain, and whether the organization's DMARC reporting configuration reveals security tooling.

The answers directly shape initial-access planning. A domain with `p=none` DMARC and `~all` SPF softfail can be spoofed at the `From:` header level — receiving servers will not hard-reject it. A domain with `p=reject` and tightly scoped SPF hardfail requires a different approach: impersonate a look-alike domain instead, or find a trusted third-party `include:` that can be abused as a sending identity.

## 2.2 Reading SPF for infrastructure intelligence

An SPF TXT record publishes the complete list of IP addresses and hostnames authorized to send mail on behalf of the domain. The record format begins with `v=spf1` and ends with an `all` qualifier that defines the policy for any sender not explicitly listed.

```text
v=spf1 include:_spf.google.com include:sendgrid.net include:spf.protection.outlook.com ip4:198.51.100.0/24 ~all
```

From this single record an attacker extracts four pieces of intelligence: Google Workspace is a primary sending platform (`include:_spf.google.com`), a transactional mail provider is in use (`include:sendgrid.net`), there is also an Outlook-based sending path which may indicate a Microsoft 365 tenant for a subsidiary (`include:spf.protection.outlook.com`), and a dedicated IP range is separately authorized (`ip4:198.51.100.0/24`). The `~all` softfail means non-listed senders receive a soft failure — messages from unauthorized sources will not be hard-rejected at the protocol level, only flagged.

Each `include:` is a business intelligence pivot. Marketing platforms, CRM tools, helpdesk vendors, and bulk-mail services are all exposed here. This tells you what SaaS the organization depends on and potentially where a supply-chain social engineering angle exists — impersonating a trusted vendor whose domain is listed in the target's SPF record.

**SPF lookup limit:** SPF mechanisms that trigger DNS lookups (include, a, mx, ptr, exists) are capped at 10 total recursive lookups. Organizations with many `include:` entries sometimes hit this limit, which causes SPF to return `PermError`. A `PermError` domain is functionally spoofable for DMARC purposes because SPF fails in a way that prevents a valid pass result.

## 2.3 Reading DMARC for enforcement posture

DMARC sits above SPF and DKIM and tells receiving mail servers what action to take when an inbound message fails both authentication checks. The policy is published at `_dmarc.<domain>` as a TXT record. DMARC also defines an alignment requirement: the domain in the authenticated SPF or DKIM result must match the visible `From:` header domain — this is what closes the gap between envelope-sender SPF checks and the user-visible address.

```text
v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s; pct=100; rua=mailto:dmarc-agg@target.com
```

The `p=` field is the operative enforcement mode: `none` is monitoring-only (no action taken on failing mail), `quarantine` routes failing mail to spam, and `reject` instructs receivers to hard-reject the message. For offensive purposes, `p=none` means the domain is spoofable at delivery. `p=quarantine` means spoofed mail may land in spam — still deliverable in some scenarios. `p=reject` with `pct=100` is the strongest posture.

The `adkim=` and `aspf=` fields control alignment strictness. `s` is strict alignment — the domain in the DKIM signature or SPF result must exactly match the `From:` header domain. `r` (relaxed, the default) allows organizational-domain matching, meaning a DKIM signature from `mail.target.com` satisfies alignment for `from: user@target.com`. Relaxed alignment opens up some bypass scenarios with subdomain-signed messages.

The `sp=` field governs subdomains independently. A `p=reject` with no `sp=` means subdomains inherit the parent policy — but this is receiver-dependent. An explicit `sp=none` means subdomain mail gets no enforcement even when the parent is `reject`. This is a common misconfiguration: the primary domain is locked down but `marketing.target.com` or `send.target.com` can be spoofed.

The `rua=` field reveals where aggregate DMARC reports go. If that address belongs to a third-party DMARC analysis service (e.g. `rua=mailto:reports@dmarcian.com`), it confirms the organization has an active mail security monitoring program. If the address is an internal mailbox, check whether it is actively monitored or a dead inbox.

## 2.4 Locating DKIM selectors and what they reveal

DKIM uses a selector to identify a specific public key at `<selector>._domainkey.<domain>`. The selector value is chosen by the sending platform and is embedded in outbound mail headers — it cannot be discovered from the domain alone without seeing a real email or guessing known patterns.

Known selector naming patterns by provider:

| Provider | Common selectors |
|----------|----------------|
| Google Workspace | `google`, `google1`, `google2` |
| Microsoft 365 | `selector1`, `selector2` |
| Mailchimp / Mandrill | `k1`, `k2`, `k3` |
| Sendgrid | `s1`, `s2`, `smtpapi` |
| Proofpoint | `proofpointmailsig`, `pp1` |
| Mimecast | `mc1`, `mimecast` |
| Amazon SES | `amazonses`, custom per-region |

Checking these selectors is passive. Each guess is a single DNS TXT query. If the selector exists, the key response confirms the sending platform. A valid DKIM key also reveals the algorithm and key length — RSA keys under 1024 bits have documented factoring vulnerabilities.

A wildcard DKIM record at `*._domainkey.<domain>` is a misconfiguration: it means any selector matches and validates, which undermines DKIM's purpose. It is worth checking.

## 2.5 Organizational domain and subdomain policy inheritance

Organizations frequently protect their primary domain with strong DMARC but leave subdomain policy unspecified or weak. The attack surface is in the subdomains.

```text
_dmarc.target.com   → p=reject; sp=none
_dmarc.mail.target.com  → (no record — inherits nothing independently)
_dmarc.send.target.com  → v=DMARC1; p=none
```

In this example, `send.target.com` is explicitly monitored-only and can be spoofed in `From:` headers. Receivers that check `_dmarc.send.target.com` find `p=none` and take no enforcement action. This is a real and common configuration error — teams lock the brand domain but forget operational subdomains used by marketing or transactional mail platforms.

## 2.6 Dead-end vs high-value finding

**Dead-end:** SPF is `v=spf1 include:_spf.google.com -all`, DMARC is `p=reject; sp=reject; pct=100; adkim=s; aspf=s`. The domain is fully hardened. Spoofing the primary domain at `From:` header level is blocked by both SPF hardfail and DMARC reject with strict alignment. Note it and move on — the attack surface here is zero for direct spoofing.

**High-value:** SPF is `v=spf1 include:_spf.google.com include:sendgrid.net include:mktomail.com include:spf.mandrillapp.com ~all` — four `include:` entries covering four different vendors, ending in softfail. DMARC is `v=DMARC1; p=none; rua=mailto:dmarc@target.com`. No `sp=` tag. You can send email from `From: ceo@target.com` with `MAIL FROM:` any authorized domain and have it arrive in inboxes without hard rejection. The four vendor includes map their entire marketing and transactional mail stack. The internal `rua=` address may be a dead mailbox.

## 2.7 Where results feed next

Mail posture results flow directly into initial-access and delivery planning. A spoofable domain enables pretexting as the target organization's own senders — far more convincing than impersonating a third party. SPF `include:` entries map the SaaS stack and reveal what platforms defenders have integrated trust relationships with. DMARC `rua=` addresses confirm whether a security team is actively watching their email authentication telemetry. Subdomain policy gaps identify specific domains to spoof in targeted pretexting.

All of this feeds Phase 4 phishing campaign design without generating a single network packet to the target.

## 2.8 BIMI as a maturity signal

BIMI (Brand Indicators for Message Identification) is a relatively recent DNS-based standard that allows organizations to display a verified brand logo in mail clients that support it (Gmail, Yahoo, Apple Mail). The DNS record is published at `default._bimi.<domain>` as a TXT record pointing to an SVG image URL and an optional Verified Mark Certificate (VMC).

For recon purposes, BIMI is a maturity signal: a VMC-backed BIMI record is issued by a commercial certificate authority (currently only DigiCert for BIMI VMCs) and requires the domain to have active `p=quarantine` or `p=reject` DMARC enforcement. A domain with a valid VMC-backed BIMI record has:

- DMARC enforcement at quarantine or reject level
- A dedicated trademarked logo registered with a CA
- A security team sufficiently mature to have deployed all three authentication layers

This means BIMI presence correlates strongly with a hardened email posture. If you find BIMI, you can immediately infer `p=quarantine` or `p=reject` without checking DMARC directly.

```bash
# Check for BIMI record
$ dig +short TXT default._bimi.target.com
"v=BIMI1; l=https://images.target.com/logo.svg; a=https://cert.target.com/bimi.pem"

# The 'a=' field points to the VMC (Verified Mark Certificate)
# If 'a=' is absent or points to an empty authority, it is a self-asserted BIMI (no CA validation)
# VMC-backed = enforced DMARC confirmed
# No VMC = BIMI is decorative, domain may still be at p=quarantine or p=reject but unverified

# Absence of BIMI record:
$ dig +short TXT default._bimi.target.com
# Empty = no BIMI — no information about DMARC enforcement level
```

For offensive planning, a BIMI-hardened domain with VMC is essentially non-spoofable at the brand domain level. The attack angle shifts to: find a related acquisition domain, regional domain, or operational subdomain (`send.target.com`) that was not brought under the same enforcement umbrella.

## 2.9 Multi-domain batch auditing workflow

Large organizations operate multiple domains — the primary brand domain, product-specific domains, regional variants, and acquired company domains. The primary `target.com` may be fully hardened while `target-emea.com`, `acquired-startup.io`, or `targetpay.com` are unenforced legacies.

The weakest domain in the portfolio is the entry point. Batch auditing every domain identified during Stage 1 (WHOIS, CT logs, Shodan) is the correct approach.

```bash
# checkdmarc multi-domain batch (pipe from domain list)
$ cat identified_domains.txt
corp-target.com
corp-target-eu.com
legacy-brand.com
acquisition-2022.io
corp-target.net

$ checkdmarc $(cat identified_domains.txt | tr '\n' ' ') -f json | tee batch_mail_audit.json

# Extract all p=none domains (spoofable)
$ cat batch_mail_audit.json | python3 -c "
import json, sys
for d in json.load(sys.stdin):
    p = d.get('dmarc', {}).get('p', 'absent')
    if p in ['none', 'absent', None]:
        print(f\"[SPOOFABLE] {d['domain']} p={p}\")
"
[SPOOFABLE] corp-target-eu.com p=none
[SPOOFABLE] acquisition-2022.io p=absent
[SPOOFABLE] corp-target.net p=none

# Three of five domains are spoofable. Primary brand is hardened but the rest are not.
```

The acquisition domain `acquisition-2022.io` has no DMARC at all. In a real phishing campaign, you would send mail with `From: ceo@acquisition-2022.io` or `From: payroll@acquisition-2022.io` and it would arrive in inboxes without any SPF/DMARC rejection. The target employees may trust emails from an acquisition they know the parent company made.

```bash
# Automated SPF softfail detector across all domains
$ for domain in $(cat identified_domains.txt); do
    spf=$(dig +short TXT $domain 2>/dev/null | grep spf)
    if echo "$spf" | grep -qE '~all|\+all'; then
        echo "[SOFTFAIL] $domain: $spf"
    fi
  done
[SOFTFAIL] corp-target-eu.com: v=spf1 include:_spf.google.com ~all
[SOFTFAIL] acquisition-2022.io: v=spf1 ip4:198.51.100.5 ~all
```

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **SPF (Sender Policy Framework)** | DNS TXT record listing IPs and domains authorized to send mail for a domain |
| **DKIM (DomainKeys Identified Mail)** | Cryptographic signing of outbound mail; receiver validates the signature against a public key in DNS |
| **DMARC (Domain-based Message Authentication, Reporting & Conformance)** | Policy record linking SPF/DKIM results to an enforcement action (none/quarantine/reject) and alignment requirements |
| **MX record** | DNS record identifying the mail server(s) that receive inbound mail for a domain |
| **Selector** | Identifier in a DKIM key that points to a specific public key (e.g. `selector1._domainkey.example.com`) |
| **Alignment** | DMARC requirement that the domain in SPF/DKIM result matches the `From:` header domain |
| **Organizational domain** | The registrable domain (e.g. `target.com`) used as the anchor for relaxed DMARC alignment |
| **Envelope sender** | The SMTP `MAIL FROM:` address used for SPF evaluation — may differ from the visible `From:` header |
| **`p=none`** | DMARC monitor-only mode — no enforcement action is requested for failing mail |
| **`p=quarantine`** | DMARC policy requesting receivers treat failing mail as suspicious (route to spam) |
| **`p=reject`** | DMARC policy requesting receivers hard-reject failing mail |
| **`sp=`** | DMARC subdomain policy; controls policy for subdomains when no subdomain-specific DMARC record exists |
| **`adkim=`** | DMARC DKIM alignment mode: `s` (strict) or `r` (relaxed, default) |
| **`aspf=`** | DMARC SPF alignment mode: `s` (strict) or `r` (relaxed, default) |
| **`pct=`** | Percentage of failing messages to apply the DMARC policy to; `pct=100` means all failing messages |
| **`rua=`** | Aggregate DMARC report destination — receives daily XML summary reports of pass/fail statistics |
| **`ruf=`** | Forensic DMARC report destination — may receive per-message failure reports |
| **`~all`** | SPF softfail — non-listed senders receive a soft failure result, not hard rejection |
| **`-all`** | SPF hardfail — non-listed senders are hard-rejected |
| **`+all`** | SPF pass-all — all senders are authorized; effectively disables SPF (rare but exists) |
| **SPF PermError** | SPF processing failure, typically from exceeding 10 DNS lookups; treated as SPF fail by DMARC |
| **BIMI** | Brand Indicators for Message Identification — DNS record showing a brand logo in mail clients; only issued when DMARC is at `p=quarantine` or `p=reject` |

| Control | Result variants | Attacker interpretation |
|---------|----------------|------------------------|
| SPF | pass / fail / softfail / neutral / permerror | softfail + DMARC p=none = spoofable |
| DMARC | none / quarantine / reject | none = full spoofability; quarantine = partial |
| DKIM | pass / fail | fail alone does not block if SPF passes and aligns |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|------|---------|-------------------|---------------|
| `dig` | `dig +short TXT example.com` | SPF policy, `all` qualifier, and `include:` chain | First pass — every engagement |
| `dig` | `dig +short TXT _dmarc.example.com` | DMARC policy, `sp=`, `pct=`, `rua=`, `ruf=` | DMARC enforcement check |
| `dig` | `dig +short MX example.com` | Mail server hostnames and priority weights | Receiving provider identification |
| `dig` | `dig +short TXT selector1._domainkey.example.com` | DKIM public key for a known selector | Sending platform confirmation |
| `host` | `host -t TXT _dmarc.example.com` | DMARC TXT record (cleaner output than dig in some envs) | Quick confirmation |
| `nslookup` | `nslookup -type=TXT _dmarc.example.com` | DMARC record on Windows or non-Kali systems | Cross-platform fallback |
| `dnsrecon` | `dnsrecon -d example.com -t std` | Full standard DNS record set: TXT, MX, NS, A, AAAA | Broad baseline before targeted queries |
| `checkdmarc` (Python) | `checkdmarc example.com` | Structured SPF/DMARC/SMTP TLS validation with parsed output | Automated multi-domain auditing |

**SPF lookup and interpretation:**
```bash
$ dig +short TXT example.com
"v=spf1 include:_spf.google.com include:sendgrid.net ~all"
```
Google Workspace is the primary sending platform. Sendgrid handles transactional mail. The `~all` softfail means non-listed senders are not hard-rejected — this domain is spoofable if DMARC is also `p=none` or absent.

**SPF with PermError condition (exceeded lookup limit):**
```bash
$ dig +short TXT example.com
"v=spf1 include:a.com include:b.com include:c.com include:d.com include:e.com
         include:f.com include:g.com include:h.com include:i.com include:j.com ~all"
```
Ten `include:` entries that each resolve further sub-includes. If the recursive lookup count exceeds 10, receivers return SPF `PermError`. DMARC treats `PermError` as SPF fail — the domain may be functionally spoofable even if the intent was to protect it.

**DMARC lookup and interpretation:**
```bash
$ dig +short TXT _dmarc.example.com
"v=DMARC1; p=none; rua=mailto:dmarc@example.com"
```
Policy is `none` — monitoring only. No enforcement action is requested for failing mail. Combined with SPF softfail, this domain can be convincingly spoofed at the `From:` header level and mail will be delivered. The `rua=` address `dmarc@example.com` may or may not be monitored — that distinction matters for OPSEC.

**DMARC with full policy (hardened example):**
```bash
$ dig +short TXT _dmarc.example.com
"v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s; pct=100; rua=mailto:reports@dmarcian.com"
```
Hard `reject` on both primary domain and subdomains, strict alignment on both SPF and DKIM, 100% coverage. The `rua=` points to a professional DMARC monitoring vendor — the security team is actively watching. Spoofing this domain at the `From:` level is blocked.

**MX lookup and interpretation:**
```bash
$ dig +short MX example.com
10 aspmx.l.google.com.
20 alt1.aspmx.l.google.com.
30 alt2.aspmx.l.google.com.
```
Mail is received by Google infrastructure. Confirms Google Workspace as the receiving platform. Cross-reference with SPF: if SPF also has `include:_spf.google.com`, the send and receive paths are both Google, confirming Google Workspace as the primary platform. MX alone does not prove the sending platform.

**DKIM key lookup:**
```bash
$ dig +short TXT selector1._domainkey.example.com
"v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
```
Selector `selector1` is valid — this pattern is Microsoft 365. The key type is RSA. Estimate strength from the `p=` base64 length: a 2048-bit RSA key produces approximately 344 base64 characters; shorter values indicate weaker keys.

**DKIM selector guessing loop (Kali bash):**
```bash
domain="example.com"
for sel in google google1 selector1 selector2 k1 k2 s1 s2 mail default; do
    result=$(dig +short TXT "${sel}._domainkey.${domain}" 2>/dev/null)
    if [[ -n "$result" ]]; then
        echo "[FOUND] ${sel}: $result"
    fi
done
```
Runs 10 targeted guesses covering the most common providers. Non-existent selectors return empty — no noise, no NXDOMAIN spam.

**checkdmarc (automated structured output):**
```bash
$ pip install checkdmarc
$ checkdmarc example.com
{
  "domain": "example.com",
  "spf": { "valid": true, "record": "v=spf1 include:_spf.google.com ~all", "all": "softfail" },
  "dmarc": { "valid": true, "record": "v=DMARC1; p=none", "p": "none", "sp": null }
}
```
Structured output suitable for scripting across multiple domains in a target list.

---

# Section 5 — Defender detection

Passive DNS queries for SPF, DMARC, MX, and DKIM records generate no application-layer events on the target's infrastructure. Queries go to public DNS resolvers — the target's mail servers and web servers receive nothing. This is fully passive with no server-side detection vector.

- **DNS query logs:** DNS recursive resolvers (Google 8.8.8.8, Cloudflare 1.1.1.1, ISP resolvers) retain query logs including source IP, queried name, record type, and timestamp. These logs are held by the resolver operator, not the target. The target organization cannot see who queried their SPF record. The only party that could theoretically correlate your queries is the resolver operator.
- **Authoritative DNS logs:** Some organizations configure their authoritative DNS server to log incoming queries. In that case, queries for `_dmarc.target.com` or `selector1._domainkey.target.com` would be visible in those logs with your resolver's source IP. This is uncommon but technically possible. Mitigate by using a major public resolver (Google, Cloudflare) as an intermediary — the query the authoritative server sees comes from the resolver, not from you.
- **DKIM selector guessing noise:** A short, targeted list of 10–20 provider-specific selectors generates minimal query volume — indistinguishable from normal administrative or verification queries. Running a wordlist of 500+ selector guesses from a single source creates a detectable burst of NXDOMAIN responses in resolver telemetry and authoritative server logs. Stick to provider-specific targeted lists.
- **SPF `include:` chain queries:** When you resolve an SPF record, your resolver may trigger recursive DNS lookups for each `include:` domain (e.g. `_spf.google.com`, `sendgrid.net`). These go to the respective third-party's authoritative DNS servers, not the target's. No target-side visibility.
- **Intelligence exposure the defender cannot control:** The SPF `include:` chain, DMARC `rua=` address, DKIM selector confirmations, and MX provider identifiers are all public by design. Defenders who understand this exposure periodically audit what their DNS records disclose. Most do not. The SPF record is a free SaaS inventory for any attacker who reads it.
- **OPSEC for the operator:** Use DNS-over-HTTPS or a non-attributed resolver for sensitive engagement phases. Avoid querying from engagement-specific infrastructure IPs when those IPs should not be linked to the target. Standard DNS queries from a Kali VM behind a home ISP are unattributable in practice.
- **BIMI and VMC lookups:** Querying `default._bimi.target.com` and following the `a=` URL to fetch the VMC makes an outbound HTTPS request to the target's certificate server. That single request would appear in their CDN or web server logs. Use a public certificate transparency viewer or crt.sh to inspect the VMC content instead.

---

# Section 6 — Lab task

**Platform:** Kali Linux VM with `dig` and `dnsrecon` (both installed by default). Install `checkdmarc` via pip. Target: use your own domain if you have one, or enumerate a live real-world domain you are authorized to examine — `tutorialspoint.com`, `hackthebox.com`, or any similarly public target whose mail posture is representative.

**Objective:** Produce a complete mail posture profile and deliver a written spoofing-risk verdict with full justification from DNS evidence.

**Steps:**

1. Run a broad baseline first: `dnsrecon -d <domain> -t std | tee dnsrecon_output.txt` — capture all TXT, MX, NS records in one pass.
2. Extract and parse the SPF record: `dig +short TXT <domain> | grep spf` — record the `all` qualifier and list every `include:` and `ip4:`/`ip6:` entry.
3. For each `include:` entry, identify which product or vendor it belongs to — add each to a technology-map note.
4. Count the total DNS lookups the SPF chain requires — flag if approaching the 10-lookup limit.
5. Extract DMARC: `dig +short TXT _dmarc.<domain>` — record `p=`, `sp=`, `pct=`, `adkim=`, `aspf=`, `rua=`, `ruf=`.
6. Extract MX records: `dig +short MX <domain>` — identify the receiving infrastructure from hostname patterns.
7. Run the DKIM selector guessing loop from Section 4 against all common selectors for the identified sending providers.
8. For any found DKIM key, estimate the key strength from the base64 `p=` length.
9. Check subdomain DMARC independently: `dig +short TXT _dmarc.mail.<domain>` and `dig +short TXT _dmarc.send.<domain>`.
10. Compile all findings into a `findings.md` with a table (Record | Value | Attacker Insight) and a final paragraph delivering a spoofing risk verdict: high / medium / low with specific evidence.

**Expected output:** `dnsrecon_output.txt`, filled `findings.md` with the complete record table, DKIM selector results showing at least one found or all-miss with reasoning, and a spoofing risk paragraph citing specific record values.

**Git artifact:**
```
mail-posture-recon/
├── dnsrecon_output.txt
├── spf_record.txt
├── dmarc_record.txt
├── mx_record.txt
├── dkim_selectors.txt
└── findings.md
```
```bash
git commit -m "recon(stage1): mail posture recon — SPF/DMARC/DKIM analysis and spoofing risk verdict for <domain>"
```

---

# Section 7 — Common mistakes

**1. Treating MX records as proof of the sending platform**
_Why it matters:_ MX identifies where inbound mail is *received*, not where outbound mail is *sent from*. An organization can receive mail at an on-premise Exchange server while sending through Google Workspace or Sendgrid. Declaring "they use Microsoft 365" based solely on `mx.protection.outlook.com` while the SPF record shows `include:_spf.google.com` would be wrong.
_Fix:_ Always cross-reference MX with SPF `include:` values. SPF shows the authorized senders; MX shows the receivers. Both together paint the real picture.

**2. Treating `p=none` as "DMARC is in place"**
_Why it matters:_ `p=none` is monitoring mode — it requests zero enforcement action from receivers. Calling this a protective control in a risk assessment is incorrect. A domain with `p=none` is as spoofable at the `From:` header level as a domain with no DMARC record at all, from a delivery perspective.
_Fix:_ Only `p=quarantine` and `p=reject` represent enforcement. Document `p=none` explicitly as "no enforcement" and flag it as a spoofing enabler.

**3. Assuming SPF authenticates the visible `From:` header**
_Why it matters:_ SPF authenticates the *envelope sender* (`MAIL FROM:` in the SMTP session), not the `From:` header the recipient sees in their mail client. An attacker can craft a message where `MAIL FROM:` passes SPF for a throwaway domain they control, while `From:` displays `ceo@target.com`. This phish lands if DMARC is absent or `p=none`, because DMARC is the only control that enforces alignment between the authenticated domain and the visible `From:` header.
_Fix:_ Always evaluate DMARC alignment as the operative spoofing barrier — SPF alone does not protect the user-visible identity.

**4. Guessing DKIM selectors with a large generic wordlist**
_Why it matters:_ DKIM selectors are not a publicly enumerable namespace. Running hundreds of selector guesses generates NXDOMAIN noise for every miss, creates unnecessary query volume visible in resolver telemetry, and wastes time — selectors are vendor-specific and predictable with a short targeted list.
_Fix:_ Use a short, provider-specific list (10–20 guesses) derived from the SPF `include:` values and MX provider. If the SPF confirms Google Workspace, try `google` and `google1`. If M365, try `selector1` and `selector2`.

**5. Ignoring SPF `include:` entries as intelligence**
_Why it matters:_ Every `include:` in an SPF record discloses a third-party SaaS platform or mail vendor the organization trusts to send mail on their behalf. Treating these as noise loses your clearest window into the target's SaaS stack — marketing platforms, helpdesk tools, CRM integrations, and transactional mail services are all enumerated here for free.
_Fix:_ For each `include:` value, identify what product or vendor it belongs to and record it in your technology map. This feeds both social engineering pretext construction and SaaS attack-surface analysis.

**6. Not checking subdomain DMARC policy independently**
_Why it matters:_ A well-configured primary domain with `p=reject` may have explicit `sp=none` — or worse, no `sp=` at all and subdomains with their own weaker `_dmarc.` records. Marketing subdomains (`send.target.com`, `mail.target.com`, `em.target.com`) are routinely misconfigured with `p=none` while the brand domain is locked down.
_Fix:_ Always check `_dmarc.<domain>` for the `sp=` tag. Then independently query `_dmarc.mail.<domain>`, `_dmarc.send.<domain>`, and other operational subdomains discovered from SPF includes and MX records.

**7. Stopping at one domain when the target has multiple**
_Why it matters:_ Large organizations operate multiple domains — brand domains, product domains, regional domains, acquisition domains. The primary `target.com` may be hardened while `target-eu.com`, `targetpay.com`, or a recently acquired company's domain has `p=none` and softfail SPF. The weakest domain in the group is your entry point.
_Fix:_ Run SPF/DMARC checks across every domain identified during Stage 1 OSINT — not just the primary brand domain. Use `checkdmarc` in batch mode: `checkdmarc domain1.com domain2.com domain3.com -f json > results.json`.

---

# Section 8 — Move-on gate

1. Query SPF, DMARC, and MX for a live domain without looking at notes, then deliver a one-sentence spoofing risk verdict explaining exactly *why* the domain is or is not spoofable — citing the specific `all` qualifier, `p=` value, and `sp=` value from the DNS records you retrieved.

2. Given the DMARC record `v=DMARC1; p=none; sp=quarantine; rua=mailto:reports@example.com` — without notes, state what happens to spoofed mail targeting the primary domain, what happens to spoofed mail targeting a subdomain, and what the `rua=` address tells you about the organization's security monitoring maturity.

3. Run DKIM selector guessing against a confirmed Google Workspace or Microsoft 365 domain using the provider-specific selector list from memory, retrieve one valid DKIM public key, and state the algorithm, estimated key strength, and what that strength means for an attacker.
