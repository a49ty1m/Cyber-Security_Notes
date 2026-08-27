# Breach & Paste Monitoring

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Breach and paste monitoring is the passive reconnaissance process of checking publicly exposed breach datasets, paste locations, code repositories, and credential-leak indexes for information associated with an authorized target. The useful output is not simply "a password was leaked"; it is a set of intelligence such as exposed email addresses, usernames, password-reuse indicators, API-key patterns, internal hostnames, VPN references, software names, repository paths, and historical organizational relationships.

This sits late in passive reconnaissance because it combines information discovered from other OSINT activities with historical exposure data.

```text
Reconnaissance
│
├── Passive
│   ├── Domain / DNS intelligence
│   ├── Certificate intelligence
│   ├── Technology fingerprinting
│   ├── Human / organizational OSINT
│   └── Breach & Paste Monitoring  ← THIS TOPIC
│
└── Active
    ├── Enumeration
    ├── Scanning
    ├── Service discovery
    └── Vulnerability validation
```

If you underestimate this skill, you may waste time attacking infrastructure when useful intelligence already exists in historical exposures.

What came before establishes **who and what belongs to the target**; this topic determines whether historical leaks add useful context before moving into active enumeration.

# Section 2 — How attackers actually use this

## 2.1 What attackers are actually looking for

An attacker is not merely searching for "breached passwords." They are looking for relationships between seemingly unrelated pieces of leaked information.

Typical targets include:

* Corporate email addresses
* Employee usernames
* Password hashes
* Password-reset information
* Reused passwords or password patterns
* API keys
* Cloud credentials
* OAuth tokens
* Database connection strings
* Internal hostnames
* VPN endpoints
* Internal IP addresses
* Repository names
* CI/CD variables
* SaaS tenant names
* Development environments
* Email naming conventions
* Old employee accounts
* Forgotten subdomains
* Software and framework versions
* Internal project names

The highest-value information is usually **context**, not the raw credential itself.

For example:

```text
employee@company.test
        │
        ├── historical breach
        │       └── reused username
        │
        ├── code exposure
        │       └── staging-api.company.test
        │
        └── public documentation
                └── VPN.company.test
```

Individually these findings may look harmless. Together they reveal the organization's naming conventions and attack surface.

## 2.2 The realistic attacker workflow

A mature workflow looks like this:

1. Establish the target's legitimate identifiers.

   * Primary domains
   * Known subsidiaries
   * Corporate email domain
   * Public usernames
   * Brand names

2. Search historical exposure sources for those identifiers.

3. Separate results into categories:

   * Identity
   * Authentication
   * Infrastructure
   * Application
   * Development
   * Third-party services

4. Normalize duplicate findings.

5. Determine whether the exposure is historical or still relevant.

6. Correlate leaked usernames with publicly visible employee naming conventions.

7. Correlate infrastructure names with DNS, certificates, documentation, or code previously discovered during passive reconnaissance.

8. Prioritize findings based on:

   * Recency
   * Privilege
   * Reuse
   * Production relevance
   * Confidence
   * Exposure type

9. Carry high-confidence infrastructure and identity intelligence into the active-enumeration phase.

The critical distinction is:

```text
Raw finding
    ↓
Validated context
    ↓
Correlation
    ↓
Attack-surface intelligence
```

A breach entry saying:

```text
user@example.test : old-password
```

does not automatically mean the password works.

But:

```text
user@example.test
old username
company VPN naming convention
staging hostname
cloud service reference
```

can reveal substantially more about the organization's architecture.

## 2.3 Dead-end finding vs high-value finding

### Dead-end

```text
old-user@example.test
Source: breach from many years ago
Status: employee no longer appears publicly
No infrastructure references
No current organization relationship
```

This has low intelligence value because there is no strong evidence connecting it to the current attack surface.

### High-value

```text
developer@example.test
Historical exposure
Repository reference
staging-api.example.test
Internal project name
Cloud service identifier
```

This is valuable because several independent identifiers connect the exposure to a potentially current development environment.

The important lesson:

> **Correlation beats volume.**

Ten thousand unrelated breach records are less useful than three strongly correlated findings.

## 2.4 How results feed the next phase

Useful findings can create pivots such as:

```text
Leaked email
    ↓
Username convention
    ↓
Public developer profile
    ↓
Repository
    ↓
Staging hostname
    ↓
DNS / certificate enumeration
    ↓
Active service discovery
```

Or:

```text
Leaked API-key pattern
    ↓
Identify affected application/service
    ↓
Determine whether key is historical
    ↓
Authorized validation
    ↓
Credential rotation / exposure remediation
```

The breach phase therefore acts as a bridge between passive intelligence and active attack-surface validation.

## 2.5 Why attackers prioritize historical data

Historical exposure can reveal things that current websites deliberately hide.

An organization may have removed:

```text
dev-old.example.test
```

from current documentation while an old repository, breach record, or paste still contains the hostname.

That creates a historical pivot:

```text
Current public surface
        +
Historical exposure
        ↓
Previously hidden relationship
```

This is particularly useful for discovering forgotten development infrastructure, legacy applications, old employee identities, and deprecated services.

# Section 3 — Core concepts and terminology

| Term                | Meaning                                                                                       |
| ------------------- | --------------------------------------------------------------------------------------------- |
| Breach              | Unauthorized exposure of information from a system or service.                                |
| Breach corpus       | A collection or index of records originating from one or more breaches.                       |
| Paste               | Publicly posted text/data, historically associated with services such as Pastebin-like sites. |
| Credential leak     | Exposure of authentication material such as usernames, passwords, tokens, or hashes.          |
| API key             | Secret value used by an application to authenticate to an API.                                |
| Password hash       | One-way representation of a password stored by an authentication system.                      |
| Credential reuse    | Using the same or similar authentication secret across multiple services.                     |
| Username convention | Organizational pattern such as `first.last` or `flast`.                                       |
| Internal hostname   | Hostname intended for internal infrastructure, such as `db01.internal.example`.               |
| Historical exposure | Information exposed in the past that may or may not remain valid.                             |
| IOC                 | Indicator of compromise; an artifact useful for detecting malicious activity.                 |
| Pivot               | Using one finding to discover another related asset or identifier.                            |
| Correlation         | Connecting multiple independent findings to establish a stronger relationship.                |
| False positive      | A result that appears relevant but does not actually belong to the target.                    |
| Secret              | Sensitive authentication material that should not be publicly exposed.                        |

### Exposure types

| Exposure        | Typical intelligence value                  |
| --------------- | ------------------------------------------- |
| Email address   | Identity and account enumeration            |
| Username        | Authentication and identity correlation     |
| Password        | Potential credential-reuse indicator        |
| Password hash   | Authentication-system intelligence          |
| API key         | Potential application/service access        |
| Token           | Potential session or service authentication |
| Hostname        | Infrastructure discovery                    |
| Repository path | Development intelligence                    |
| Internal IP     | Network architecture intelligence           |
| Project name    | Organizational/development context          |

# Section 4 — Tools and commands

Use these commands against **your own lab data, intentionally published test data, or targets for which you have explicit authorization**. Do not treat a breach index as a permission to test credentials against a live service.

| Tool         | Command                                            | What it finds/shows                         | When to use it                        |                    |                                       |                             |                        |
| ------------ | -------------------------------------------------- | ------------------------------------------- | ------------------------------------- | ------------------ | ------------------------------------- | --------------------------- | ---------------------- |
| `ripgrep`    | `rg -n -i 'api[_-]?key                             | token                                       | password                              | secret' ./corpus/` | Secret-like strings in a local corpus | Searching exported lab data |                        |
| `grep`       | `grep -RniE 'internal                              | staging                                     | dev                                   | vpn                | db[0-9]+' ./corpus/`                  | Infrastructure terminology  | Finding hostname clues |
| `awk`        | `awk -F: '{print $1}' emails.txt \| sort -u`       | Unique first fields                         | Normalizing lab records               |                    |                                       |                             |                        |
| `sort`       | `sort -u emails.txt`                               | Deduplicated identifiers                    | Cleaning collected data               |                    |                                       |                             |                        |
| `cut`        | `cut -d: -f1 credentials.txt`                      | Selected fields                             | Extracting usernames from lab records |                    |                                       |                             |                        |
| `jq`         | `jq -r '.records[]?.email' breach.json \| sort -u` | Fields from JSON exports                    | Processing structured lab datasets    |                    |                                       |                             |                        |
| `sha256sum`  | `sha256sum breach.json`                            | Dataset integrity hash                      | Recording evidence integrity          |                    |                                       |                             |                        |
| `git grep`   | `git grep -nEi 'api[_-]?key\|secret\|token'`       | Secret-like content in a Git repository     | Authorized repository auditing        |                    |                                       |                             |                        |
| `trufflehog` | `trufflehog git file://./test-repo --no-update`    | Potential secrets in a local Git repository | Secret exposure validation            |                    |                                       |                             |                        |
| `gitleaks`   | `gitleaks detect --source ./test-repo --no-banner` | Secret patterns in source history/files     | Repository leak auditing              |                    |                                       |                             |                        |

### `ripgrep`

Example:

```text
$ rg -n -i 'api[_-]?key|token|password|secret' ./corpus/
corpus/test1.txt:14:api_key=TEST_ONLY_VALUE
corpus/test2.txt:8:password_hash=TEST_HASH
```

Interpretation: the search found secret-like fields. In a real investigation, classify whether they are current, historical, fake, or revoked.

### `grep`

```text
$ grep -RniE 'internal|staging|dev|vpn|db[0-9]+' ./corpus/
./corpus/sample.txt:7:staging-api.example.test
./corpus/sample.txt:12:vpn.example.test
```

These results are infrastructure clues and should be correlated with authorized DNS or asset inventories.

### `jq`

```text
$ jq -r '.records[]?.email' breach.json | sort -u
alice@example.test
bob@example.test
dev@example.test
```

The command extracts unique email identifiers from a JSON lab dataset.

### `git grep`

```text
$ git grep -nEi 'api[_-]?key|secret|token'
config/test.env:3:API_KEY=TEST_ONLY_VALUE
```

This identifies secret-like material tracked by Git. The next question is whether the value is fake, revoked, or still active.

### `trufflehog`

```text
$ trufflehog git file://./test-repo --no-update
...
Reason: High Entropy
Detector: Generic
```

Treat detector output as a lead, not proof. Validate the finding without attempting unauthorized access.

### `gitleaks`

```text
$ gitleaks detect --source ./test-repo --no-banner
Finding: API Key
File: config/test.env
Line: 3
```

The important result is the location and secret type, which lets the defender remove the secret and rotate it.

# Section 5 — Defender detection

* Pure historical breach searching normally produces **no server-side event on the breached organization**, because the lookup occurs against an external dataset.
* Repository-secret scanning can be detected through CI/CD security tooling, Git audit logs, and secret-scanning alerts.
* Monitor unusual authentication attempts involving accounts identified in historical breaches.
* Correlate impossible-travel, password-spray, MFA failures, token use, and anomalous VPN activity with known leaked identities.
* Defenders commonly miss old credentials because they assume "old breach = irrelevant"; attackers may exploit password reuse.
* Detect secret exposure through pre-commit hooks, CI scanners, repository scanning, and cloud-secret monitoring.
* Skilled operators performing legitimate passive reconnaissance minimize unnecessary interaction with the target; defenders therefore cannot rely on web-server logs to reveal this phase.

# Section 6 — Lab task

**Platform:** Local Kali VM with a synthetic breach corpus and a deliberately vulnerable Git repository. No real credentials or third-party breach data are required.

**Objective:** Prove that you can identify exposed identity, infrastructure, and secret-like indicators from a local corpus and correlate them into useful reconnaissance intelligence.

**Steps:**

1. Create a lab directory containing synthetic records such as emails, hostnames, hashes, and fake API keys.
2. Create a small Git repository containing deliberately fake secret values and an old staging hostname.
3. Search the corpus for email addresses and infrastructure terms.
4. Deduplicate the discovered identifiers.
5. Parse the JSON-formatted breach sample and extract unique email addresses.
6. Run a Git secret scanner against the synthetic repository.
7. Classify each finding as identity, infrastructure, secret-like, historical, or false positive.
8. Build a short correlation table connecting at least one identity finding with one infrastructure finding.
9. Hash the final corpus/report and record the hash in your notes.
10. Commit the sanitized lab artifacts to Git.

**Expected output:**

```text
Identity:
  developer@example.test

Infrastructure:
  staging-api.example.test
  vpn.example.test

Secret-like finding:
  API_KEY=TEST_ONLY_VALUE

Correlation:
  developer@example.test
       ↓
  development repository
       ↓
  staging-api.example.test
```

**Git artifact:**

```text
breach-paste-lab/
├── corpus/
│   ├── breach.json
│   ├── sample-paste.txt
│   └── emails.txt
├── test-repo/
│   └── config/
│       └── test.env
├── findings.md
└── hashes.txt
```

Use only synthetic values in the repository.

```bash
git add breach-paste-lab/
git commit -m "Add breach and paste monitoring lab"
```

# Section 7 — Common mistakes

1. **Treating every breach result as current**
   → Historical data may be obsolete or revoked.
   → Record the source date and validate relevance before drawing conclusions.

2. **Assuming a leaked password automatically works**
   → Password reuse is possible but not guaranteed.
   → Treat credentials as intelligence unless authorized validation proves otherwise.

3. **Collecting huge datasets without correlation**
   → Volume creates noise and hides useful relationships.
   → Prioritize identifiers that connect people, infrastructure, applications, and organizations.

4. **Ignoring false positives**
   → Common names and shared addresses can belong to unrelated organizations.
   → Correlate domain, username, infrastructure, and organizational context.

5. **Searching only for passwords**
   → Hostnames, repository names, API identifiers, and project terminology can be more valuable.
   → Search across identity, infrastructure, application, and development categories.

6. **Testing leaked credentials against live services**
   → This turns passive intelligence gathering into active authentication activity and may be unauthorized.
   → Keep the exercise inside your lab or explicitly authorized assessment.

7. **Failing to preserve evidence**
   → Dynamic sources change and duplicate records can make later analysis difficult.
   → Store sanitized results, timestamps, source classification, and integrity hashes.

# Section 8 — Move-on gate

1. **Run a local corpus search** and correctly extract unique emails, hostnames, and secret-like indicators without looking at your notes.

2. **Run `gitleaks` or `trufflehog` against your synthetic repository** and correctly identify which findings are genuine test secrets versus false positives.

3. **Take three independent findings from your lab corpus and build a correlation chain** that produces one defensible infrastructure or identity pivot, explaining why the pivot is relevant without attempting unauthorized access.
