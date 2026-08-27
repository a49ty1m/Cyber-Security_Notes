# Metadata & Leak Analysis

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Metadata & Leak Analysis combines historical web reconstruction with public-code and document analysis. The goal is to recover information that is no longer obvious on the current website: old paths, retired applications, filenames, internal naming conventions, technology references, and accidentally published secrets. The roadmap specifically places **Wayback Machine historical analysis** alongside **GitHub/GitLab dorking** for hardcoded API keys and internal naming conventions. 

```text
Recon
  └── Passive
       └── Metadata & Leak Analysis
            ├── Historical web paths
            ├── Old files/endpoints
            ├── Metadata
            ├── Public repositories
            ├── Naming conventions
            └── Secret exposure indicators
                  ↓
             Active Mapping
                  ↓
             Enumeration
                  ↓
          Technical Validation
```

If this is skipped, current-state reconnaissance can miss retired applications, old URLs, leaked configuration, and technology clues that explain the organization's attack surface. Historical analysis is explicitly used to identify previous exposures and configuration changes. 

# Section 2 — How attackers actually use this

## 2.1 Reconstruct the historical attack surface

The operator starts with the organization's current domain and asks:

* What paths existed previously?
* Which applications disappeared?
* Were administrative interfaces once public?
* Did filenames reveal internal projects?
* Did old documents expose employee or technology information?
* Did an old application use a different hostname?
* Did infrastructure or URL structure change?

The workflow is:

```text
Current domain
    ↓
Historical snapshots
    ↓
Old URLs / files / directories
    ↓
Technology and naming clues
    ↓
Current-state correlation
```

The key finding is not "an old page existed."

It is:

```text
Old application
   ↓
Distinct hostname
   ↓
Technology fingerprint
   ↓
Current DNS / infrastructure correlation
```

That creates a hypothesis for the next reconnaissance stage.

## 2.2 Search public repositories for organizational signals

Public repositories can reveal:

* internal project names
* environment names
* hostnames
* cloud account identifiers
* CI/CD systems
* package registries
* API endpoints
* configuration filenames
* developer usernames
* branch names
* deployment terminology

The important distinction is between **technology intelligence** and **credential material**.

A repository containing:

```text
environment = staging-eu
service = billing-api
cluster = payments-prod
```

may reveal valuable internal naming conventions without containing a credential.

A repository containing something resembling a secret is a higher-priority finding, but the assessor should document the exposure without unnecessarily using the credential.

## 2.3 Dork around structure, not just secrets

Effective repository searching begins with organizational context.

Conceptually:

```text
Organization
   ↓
Repository
   ↓
Filename / language / path
   ↓
Technology indicator
   ↓
Internal naming convention
   ↓
Related repository or historical artifact
```

Useful search dimensions include:

| Dimension      | Examples                                          |
| -------------- | ------------------------------------------------- |
| Organization   | organization-owned repositories                   |
| Filename       | `.env`, configuration files, deployment manifests |
| Technology     | AWS, Azure, Kubernetes, Terraform                 |
| Naming         | `staging`, `prod`, `internal`, project codenames  |
| CI/CD          | workflow and pipeline configuration               |
| Infrastructure | Terraform, Helm, Docker, Kubernetes manifests     |

The supplied material identifies metadata extraction from documents as a reconnaissance technique and describes tools such as Metagoofil for extracting metadata from documents associated with a target domain. 

## 2.4 Separate a leak from useful intelligence

Consider two findings.

### Dead-end

```text
Historical page:
2009 company careers page
→ generic job descriptions
→ no unique paths
→ no technology information
→ no useful correlation
```

It establishes history but does not materially improve the attack-surface model.

### High-value

```text
Historical snapshot
→ /legacy-admin/
→ old deployment documentation
→ "staging-payments" hostname
→ repository contains same project identifier
→ current organization still references payments infrastructure
```

This is valuable because independent sources corroborate the same internal concept.

## 2.5 Validate before pivoting

A disciplined operator does not immediately treat every leaked-looking string as valid.

Use:

```text
Finding
  ↓
Classify
  ↓
Corroborate
  ↓
Determine freshness
  ↓
Determine sensitivity
  ↓
Create hypothesis
  ↓
Feed validated intelligence into next phase
```

For example:

```text
Old hostname
    ↓
Historical evidence
    ↓
Repository reference
    ↓
Current DNS correlation
    ↓
Candidate current asset
```

The next phase can then investigate the candidate through appropriate technical enumeration.

# Section 3 — Core concepts and terminology

| Term                       | Meaning                                                                                            |
| -------------------------- | -------------------------------------------------------------------------------------------------- |
| Metadata                   | Information describing a file, document, image, or other artifact rather than its primary content. |
| Leak                       | Information unintentionally exposed outside its intended security boundary.                        |
| Historical Analysis        | Examining previous versions of publicly accessible resources.                                      |
| Wayback Machine            | A web archive used to inspect historical versions of websites and paths.                           |
| Dorking                    | Using precise search operators to locate specific information in indexed sources.                  |
| Repository                 | A source-code project containing files, history, configuration, and development artifacts.         |
| Secret                     | Sensitive authentication or authorization material such as an API token or private credential.     |
| Hardcoded Secret           | Secret material embedded directly in source code or configuration.                                 |
| Internal Naming Convention | Repeated terminology used to identify internal systems, environments, projects, or teams.          |
| Historical Path            | A URL path discovered from an older version of a website.                                          |
| Corroboration              | Confirmation of a finding using an independent source.                                             |
| Stale Finding              | Information that existed previously but may no longer be valid.                                    |
| Exposure                   | Availability of information to an unintended audience.                                             |
| Provenance                 | Record showing where a finding originated.                                                         |

| Finding type                       | Recon value                                        |
| ---------------------------------- | -------------------------------------------------- |
| Old public page                    | Low–medium                                         |
| Old application path               | Medium–high                                        |
| Internal hostname                  | High                                               |
| Technology/configuration reference | Medium–high                                        |
| Expired credential                 | Intelligence value; do not assume current validity |
| Apparently active secret           | Critical exposure indicator                        |

# Section 4 — Tools and commands

| Tool        | Command                                               | What it finds/shows                  | When to use it                        |                             |                                |
| ----------- | ----------------------------------------------------- | ------------------------------------ | ------------------------------------- | --------------------------- | ------------------------------ |
| `curl`      | `curl -I https://example.test`                        | HTTP headers                         | Inspect current response metadata     |                             |                                |
| `wget`      | `wget --mirror --no-parent https://lab.example.test/` | Local copy of an authorized lab site | Preserve lab web content              |                             |                                |
| `exiftool`  | `exiftool document.pdf`                               | File metadata                        | Analyze downloaded lab artifacts      |                             |                                |
| `grep`      | `grep -RniE 'internal\|staging\|prod' ./repo`         | Naming conventions                   | Search an authorized repository clone |                             |                                |
| `git`       | `git log --all --oneline`                             | Repository history                   | Find historical changes               |                             |                                |
| `git`       | `git log --all -S'payments' --oneline`                | Commits containing a specific string | Trace project terminology             |                             |                                |
| `git`       | `git grep -nE 'api[_-]?key                            | token                                | secret'`                              | Potential secret references | Audit an authorized repository |
| `sha256sum` | `sha256sum artifact.pdf`                              | Evidence hash                        | Preserve evidence integrity           |                             |                                |

Example:

```bash
curl -I https://lab.example.test/
```

Possible output:

```text
HTTP/2 200
content-type: text/html
server: nginx
```

Interpretation: current HTTP metadata provides a baseline for comparing historical observations.

Example:

```bash
exiftool report.pdf
```

Possible output:

```text
Author          : Engineering
Create Date     : 2024:05:10 ...
Software        : Example Office Suite
```

Interpretation: document metadata may reveal authoring organization, software, timestamps, or other contextual clues.

Example:

```bash
grep -RniE 'internal|staging|prod' ./repo
```

Possible output:

```text
./deploy/config.yaml:8:environment: staging
./docs/architecture.md:21:payments-internal
```

Interpretation: repeated terms can expose internal naming conventions.

Example:

```bash
git log --all -S'payments' --oneline
```

Possible output:

```text
8f21a6d update payments deployment
2d7e91c initial payments service
```

Interpretation: Git history can reveal when an internal concept entered or changed within the project.

Example:

```bash
git grep -nE 'api[_-]?key|token|secret'
```

Possible output:

```text
config/example.env:4:API_KEY=<REDACTED>
```

Interpretation: this identifies a potential secret location without printing or using a credential.

Example:

```bash
sha256sum report.pdf
```

Possible output:

```text
b61f...  report.pdf
```

Interpretation: record the hash alongside the evidence so the artifact can be verified later.

# Section 5 — Defender detection

* Historical archive queries themselves generally produce **no event on the target**, because the analyst is consulting a third-party archive.
* Repository searches may similarly be invisible to the organization unless repository, identity-provider, or platform telemetry records the activity.
* Monitor public repositories for secret-scanning alerts, including newly committed credentials and secrets appearing in historical commits.
* CI/CD platforms should alert when exposed credentials are detected and automatically support revocation/rotation workflows.
* Defenders commonly miss **Git history**: deleting a secret from the current branch does not necessarily remove it from historical commits.
* Defenders also miss **metadata leakage** from documents, images, and build artifacts.
* Skilled operators minimize footprint by using passive archives and public repository information, avoiding credential use, and recording findings without unnecessary access attempts.

# Section 6 — Lab task

**Platform:** Local Kali VM with a deliberately vulnerable Git repository and local static website containing historical-style artifacts.

**Objective:** Recover one historical path, one internal naming convention, and one simulated secret exposure without using a real credential.

**Steps:**

1. Create `metadata-leak-lab` and initialize a Git repository containing fictional application files.
2. Add a fake historical directory such as `legacy-admin/` and commit it.
3. Add a fictional configuration value using `API_KEY=LAB_ONLY_REDACTED_VALUE`.
4. Make a later commit removing the configuration value.
5. Use Git history to locate the previous occurrence and record the commit identifier.
6. Create a fictional PDF containing author/software metadata and inspect it with `exiftool`.
7. Build `findings.md` containing the historical path, naming convention, simulated secret exposure, provenance, and confidence.
8. Verify that every credential-like value is explicitly marked as a laboratory placeholder.
9. Hash the final evidence files and record the hashes.

**Expected output:**

```text
metadata-leak-lab/
├── findings.md
├── metadata/
│   └── lab-report.pdf
├── repository/
│   └── .git/
└── evidence/
    └── hashes.txt
```

Success means you can distinguish **current exposure**, **historical exposure**, and **intelligence-only findings**.

**Git artifact:**

```bash
git add metadata-leak-lab/
git commit -m "Add historical metadata and leak analysis lab"
```

# Section 7 — Common mistakes

1. **Mistake:** Assuming archived content is still live.
   **Why:** Historical pages are snapshots, not current infrastructure.
   **Instead:** Treat them as hypotheses requiring current-state validation.

2. **Mistake:** Searching only for API keys.
   **Why:** Naming conventions and architecture clues often provide more durable intelligence.
   **Instead:** Search for project, environment, hostname, and deployment terminology.

3. **Mistake:** Treating every secret-looking string as valid.
   **Why:** Examples, placeholders, and expired credentials are common.
   **Instead:** classify freshness and confidence before reporting impact.

4. **Mistake:** Forgetting Git history.
   **Why:** A removed credential can remain in previous commits.
   **Instead:** inspect repository history during authorized leak assessments.

5. **Mistake:** Using a discovered credential to "prove" the finding.
   **Why:** Credential use changes passive reconnaissance into active access.
   **Instead:** document the exposure and recommend revocation/rotation.

6. **Mistake:** Collecting everything from a repository.
   **Why:** Noise hides important relationships and increases unnecessary data handling.
   **Instead:** preserve only evidence needed to establish the finding.

7. **Mistake:** Failing to correlate sources.
   **Why:** One stale hostname or project name may be meaningless.
   **Instead:** connect archive evidence, repository history, metadata, and current reconnaissance before promoting a hypothesis.

# Section 8 — Move-on gate

1. **Historical reconstruction:** Analyze a lab website's historical artifacts and recover three previous paths, then correctly classify which are merely historical and which warrant current-state validation.

2. **Repository analysis:** Run Git history searches against the lab repository and identify the commit containing the simulated secret, internal project name, and deployment environment without using the simulated credential.

3. **Leak triage:** Take five findings from the lab, classify each as `Historical`, `Current`, `Intelligence`, or `Secret Exposure`, and produce a prioritized findings table with provenance and confidence without looking at your notes.
