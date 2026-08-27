# Mail Posture Recon

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling) → Mail Posture Recon

# Section 1 — What it is and where it sits

Mail posture recon examines a domain's email-authentication controls.

It produces a spoofing-risk picture.

It can reveal cloud mail providers.

It can expose third-party sending services.

It fits inside passive DNS reconnaissance.

Placement:

```text
Recon
→ Passive
→ Mail Posture
→ Active
→ Initial Access
→ Delivery
→ Post-Exploitation
```

Skipping this hides an important trust boundary.

It connects earlier DNS footprinting to later identity and phishing analysis.

# Section 2 — How attackers actually use this

## 2.1 Identify the target mail domain

Attackers start with the primary domain.

They identify MX hosts.

They inspect SPF policy.

They inspect DMARC policy.

They search for DKIM selectors.

They compare organizational domains.

The goal is not merely finding email.

The goal is learning:

who sends mail,

where mail is hosted,

and how strictly spoofing is rejected.

Google Workspace.

Microsoft 365.

Proofpoint.

Mimecast.

Marketing platforms.

Transactional mail.

Vendor includes.

Policy inheritance.

Alignment matters.

Selector reuse.

Provider fingerprints.

Authentication signals.

Trust boundaries.

Evidence correlation.

## 2.2 Read SPF for infrastructure clues

An SPF record lists authorized senders.

Those senders may be cloud providers.

It may contain vendor-specific includes.

Multiple includes can reveal SaaS dependencies.

A broad policy can increase uncertainty.

Example finding:

```text
v=spf1 include:_spf.google.com ~all
```

Google infrastructure is authorized.

`~all` represents softfail.

Another include may identify a marketing platform.

That reveals a second outbound service.

## 2.3 Read DMARC for enforcement

DMARC states what receivers should do when alignment fails.

Attackers care about the policy mode.

`p=none` means monitoring.

`p=quarantine` requests suspicious handling.

`p=reject` requests rejection.

They also inspect reporting addresses.

Reports can expose security workflows.

Example:

```text
v=DMARC1; p=reject; rua=mailto:dmarc@example.com
```

The domain has strong requested enforcement.

Aggregate reporting is configured.

## 2.4 Discover DKIM selectively

DKIM uses a selector.

The selector identifies a public-key record.

Attackers may obtain selectors from headers or public mail artifacts.

A valid key confirms signing infrastructure.

A selector name can reveal vendors.

Example:

```text
selector1._domainkey.example.com
```

The selector itself may fingerprint a provider.

## 2.5 Combine the findings

SPF answers authorized-sender questions.

DKIM provides message-signing evidence.

DMARC connects authentication to policy.

Together they show trust posture.

Dead-end finding:

```text
MX → mx.example.net
```

It identifies receiving infrastructure.

It gives limited spoofing insight.

High-value finding:

```text
SPF → several vendors
DMARC → p=none
DKIM → recognizable selectors
```

This maps mail dependencies.

It also exposes weak enforcement.

## 2.6 Feed results forward

Results inform social-engineering analysis.

They identify plausible sender infrastructure.

They help prioritize phishing simulations.

They support defensive validation.

The pivot is from DNS facts to trust-boundary analysis.

No credential attack is required.

The value comes from correlation.

# Section 3 — Core concepts and terminology

SPF — Sender Policy Framework.

Lists permitted sending infrastructure.

DKIM — DomainKeys Identified Mail.

Adds cryptographic message signatures.

DMARC — Domain-based Message Authentication.

Uses SPF and DKIM alignment.

MX — Mail exchanger record.

Identifies receiving mail servers.

TXT — DNS text record.

Common location for SPF and DMARC.

Selector — DKIM key identifier.

Selects a public-key record.

Alignment — Identity matching.

Connects authenticated domains to the visible From domain.

Envelope sender — SMTP return-path identity.

Used by SPF evaluation.

Header From — Visible sender identity.

Used by DMARC alignment.

`p=none` — Monitor only.

No requested enforcement action.

`p=quarantine` — Suspicious mail.

Requests spam-like treatment.

`p=reject` — Reject failing mail.

Strongest DMARC enforcement mode.

`rua` — Aggregate-report destination.

Receives statistical DMARC reports.

`ruf` — Failure-report destination.

May receive forensic-style reports.

Authorized sender — SPF-approved source.

DKIM public key — Verification material.

Published in DNS.

DNS propagation — Distributed record availability.

Changes may take time to appear.

| Control | Variants                         | Meaning          |
| ------- | -------------------------------- | ---------------- |
| SPF     | pass / fail / softfail / neutral | Sender result    |
| DMARC   | none / quarantine / reject       | Enforcement      |
| DKIM    | pass / fail                      | Signature result |

# Section 4 — Tools and commands

| Tool     | Command                                          | What it finds/shows | When to use it             |
| -------- | ------------------------------------------------ | ------------------- | -------------------------- |
| dig      | `dig +short TXT example.com`                     | SPF TXT data        | SPF lookup                 |
| dig      | `dig +short TXT _dmarc.example.com`              | DMARC policy        | DMARC lookup               |
| dig      | `dig +short MX example.com`                      | Mail exchangers     | Provider discovery         |
| dig      | `dig +short TXT selector._domainkey.example.com` | DKIM key            | Known selector             |
| host     | `host -t TXT example.com`                        | TXT records         | Quick confirmation         |
| nslookup | `nslookup -type=TXT _dmarc.example.com`          | DMARC TXT           | Alternative DNS client     |
| dnsrecon | `dnsrecon -d example.com -t std`                 | DNS records         | Broader passive DNS review |

Example SPF output:

```text
"v=spf1 include:_spf.google.com ~all"
```

Google infrastructure is authorized.

`~all` is a softfail policy.

Example DMARC output:

```text
"v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

DMARC is strongly enforced.

Aggregate reports are configured.

Example MX output:

```text
10 mx.example.net.
```

Mail reception points to that host.

The MX alone does not prove the provider.

Example DKIM output:

```text
"v=DKIM1; k=rsa; p=MIIBIj..."
```

The selector has a published key.

The public key supports verification.

Example `host` output:

```text
example.com has no TXT record
```

That query found no TXT data.

It does not prove no email controls exist.

Example `nslookup` output:

```text
_dmarc.example.com text = "v=DMARC1; p=none"
```

The domain publishes DMARC.

Its policy is monitoring only.

Example `dnsrecon` output:

```text
[*] MX: mx.example.net
[*] TXT: v=spf1 ...
```

The tool corroborates DNS findings.

Use targeted queries for exact posture.

# Section 5 — Defender detection

Passive DNS queries usually create no application log.

Resolvers may retain DNS query telemetry.

Watch for unusual DNS-enumeration volume.

Monitor repeated TXT and MX lookups.

Behavioral rules can flag broad discovery.

EDR generally will not see remote DNS queries.

Defenders commonly miss DKIM-selector discovery.

They may also ignore SPF vendor sprawl.

Skilled operators reduce their footprint by querying selectively.

They avoid unnecessary brute-force activity.

They correlate public data before probing further.

# Section 6 — Lab task

**Platform:** Kali Linux VM + local test DNS zone with SPF, DMARC, MX, and one DKIM selector.

**Objective:** Prove you can map a test domain's mail posture and classify its spoofing risk.

**Steps:**

1. Create or use a local test zone.

2. Publish one SPF TXT record.

3. Publish one DMARC TXT record.

4. Publish one MX record.

5. Publish one DKIM TXT record.

6. Query each record from Kali.

7. Record provider and enforcement clues.

8. Classify the posture as weak, moderate, or strong.

**Expected output:**

You can identify MX infrastructure.

You can quote the SPF policy.

You can identify DMARC enforcement.

You can verify a DKIM public key.

You can justify the risk rating.

**Git artifact:**

```text
mail-posture-recon/
├── README.md
├── evidence/
│   ├── spf.txt
│   ├── dmarc.txt
│   ├── mx.txt
│   └── dkim.txt
└── notes.md
```

Commit with:

```bash
git add mail-posture-recon/
git commit -m "Add mail posture reconnaissance lab"
```

# Section 7 — Common mistakes

1. Treating MX as proof of the mail provider.

Why: MX only identifies receiving infrastructure.

Instead: correlate MX with SPF and DKIM evidence.

2. Calling `p=none` secure enforcement.

Why: monitoring is not rejection.

Instead: distinguish visibility from enforcement.

3. Assuming SPF protects the visible From address.

Why: SPF authenticates the envelope sender.

Instead: evaluate DMARC alignment.

4. Guessing DKIM selectors blindly.

Why: selectors are not universal.

Instead: obtain selectors from legitimate artifacts or lab data.

5. Treating every SPF include as malicious exposure.

Why: vendors can be legitimate.

Instead: map business dependencies first.

6. Ignoring DNS caching.

Why: stale records can mislead analysis.

Instead: compare authoritative and recursive results when needed.

# Section 8 — Move-on gate

1. Run the SPF and DMARC lookups against your lab domain and correctly classify both policies without looking at notes.

2. Identify the MX host and explain what it proves versus what it does not prove.

3. Query a known DKIM selector, interpret the public-key record, and produce a one-paragraph spoofing-risk assessment from all three controls.
