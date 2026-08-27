# Social Vector Mapping

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Social Vector Mapping is the passive correlation of publicly available professional and organizational information to understand **who interacts with an organization, what responsibilities they have, and what technologies or workflows they publicly associate with**.

In a red-team engagement, the output is a relationship map rather than merely a list of names: people → roles → departments → technologies → domains → public documents → organizational relationships. OSINT collection is commonly treated as a reconnaissance activity whose processed results feed later enumeration.

```text
Recon
  └── Passive
       └── Social Vector Mapping
            ├── People
            ├── Roles
            ├── Technology signals
            ├── Organizational relationships
            └── Public artifacts
                  ↓
              Active Recon
                  ↓
             Enumeration
                  ↓
          Security Assessment
```

Skipping this produces noisy reconnaissance and weak hypotheses.

Underestimating it causes analysts to miss the human and organizational relationships that explain why a technical asset matters.

The preceding footprinting work establishes the organization and its external presence; this topic adds the **human/technology relationship layer** before active enumeration.

# Section 2 — How attackers actually use this

## 2.1 Build the organizational model

An operator begins with an organization, not an individual.

The first questions are:

- Who works there?
- Which departments exist?
- Who appears technically influential?
- Which people publish technical material?
- Which roles control infrastructure?
- Which technologies appear repeatedly?
- Which public documents connect people to systems?
- Which identities appear across multiple sources?

The objective is correlation.

A single profile saying "cloud engineer" is weak.

Five public profiles independently mentioning the same cloud platform, identity provider, CI/CD system, and security tooling create a much stronger technology hypothesis.

## 2.2 Identify useful human roles

The useful distinction is **role relevance**, not popularity.

Typical high-value roles in an authorized exercise include:

| Role                            | Recon value                                    |
| ------------------------------- | ---------------------------------------------- |
| CISO / security leadership      | Security architecture and defensive priorities |
| CIO / CTO                       | Technology strategy                            |
| Cloud engineer                  | Cloud platforms and deployment patterns        |
| DevOps engineer                 | CI/CD and infrastructure tooling               |
| Identity engineer               | Authentication and identity platforms          |
| Network engineer                | Network/security architecture                  |
| Help-desk administrator         | Operational workflows                          |
| Developer                       | Frameworks, repositories, deployment practices |
| Procurement / vendor management | Third-party technology relationships           |

This does **not** mean automatically targeting these people.

It means understanding which people can explain a technology or business dependency.

## 2.3 Extract technology signals

Look for explicit statements such as:

- "AWS"
- "Azure"
- "Kubernetes"
- "Terraform"
- "Okta"
- "Microsoft 365"
- "GitHub Actions"
- "Jenkins"
- "CrowdStrike"
- "Cisco"
- "Palo Alto"
- "Splunk"

Also record indirect evidence:

- conference presentations
- job advertisements
- engineering blog posts
- public repositories
- technical documentation
- certificates
- conference speaker biographies
- public PDFs
- architecture diagrams
- vendor case studies

The important distinction is:

```text
Observed:
"Engineer describes deploying Kubernetes."

Hypothesis:
"Organization may operate Kubernetes."

Unverified assumption:
"Organization definitely exposes Kubernetes externally."
```

Only the first statement is directly supported.

## 2.4 Correlate people with technology

A useful internal record might look like:

```text
Person: Employee-A
Role: DevOps Engineer
Source: Public professional profile
Technology: Kubernetes
Confidence: High

Person: Employee-B
Role: Cloud Engineer
Source: Conference biography
Technology: AWS
Confidence: High

Person: Employee-C
Role: Developer
Source: Public repository
Technology: GitHub Actions
Confidence: Medium
```

The pivot is then:

```text
Employee-A
   ↓
DevOps
   ↓
Kubernetes
   ↓
Possible deployment infrastructure
```

and:

```text
Employee-B
   ↓
Cloud
   ↓
AWS
   ↓
Possible cloud-hosted assets
```

These are **recon hypotheses**, not proof of exposed infrastructure.

## 2.5 Dead-end versus high-value finding

### Dead end

```text
Profile:
"Software professional | Technology enthusiast"
```

Why it matters little:

- no concrete role
- no organization-specific technology
- no useful relationship
- no corroborating source

### High-value finding

```text
Public conference biography:
"Cloud platform engineer responsible for Terraform,
Kubernetes and AWS infrastructure."

Corroborating job posting:
"AWS/Kubernetes platform team."

Corroborating repository:
Infrastructure-as-code references.
```

Why this matters:

- three independent sources
- identifiable technical function
- technology overlap
- possible infrastructure relationship
- useful pivot into authorized technical reconnaissance

The red-team skill is **correlation**, not collecting the largest number of names.

## 2.6 Pivot without losing provenance

Every pivot should retain:

```text
Source
  ↓
Observation
  ↓
Entity
  ↓
Relationship
  ↓
Confidence
  ↓
Next hypothesis
```

For example:

```text
Public profile
      ↓
Cloud Engineer
      ↓
AWS
      ↓
Organization technology hypothesis
      ↓
Check organization-owned public infrastructure
```

The supplied red-team material describes this general OSINT cycle as source identification → harvesting → processing/integration → analysis → results delivery.

The next phase is where validated organizational hypotheses become technical enumeration targets.

# Section 3 — Core concepts and terminology

| Term                   | Meaning                                                                                  |
| ---------------------- | ---------------------------------------------------------------------------------------- |
| OSINT                  | Intelligence derived from publicly available information.                                |
| Social Vector          | A relationship between a person, role, organization, technology, or artifact.            |
| Social Vector Mapping  | Correlating those relationships into a useful reconnaissance model.                      |
| Passive Reconnaissance | Collecting information without directly interacting with the target infrastructure.      |
| Human Profiling        | Building a role-oriented picture of people associated with an organization.              |
| Technology Signal      | Public evidence suggesting use of a technology or platform.                              |
| Pivot                  | Using one finding to discover a related entity or source.                                |
| Correlation            | Connecting independent observations into a stronger hypothesis.                          |
| Provenance             | The record of where an observation originated.                                           |
| Confidence             | How strongly the available evidence supports an observation.                             |
| Attack Surface         | Systems, services, identities, and interfaces potentially exposed to attack.             |
| Spear Phishing         | Phishing tailored to a specific person or organization.                                  |
| Whaling                | Spear phishing aimed at high-value executives or decision-makers.                        |
| Vishing                | Voice-based social engineering.                                                          |
| Smishing               | SMS-based phishing.                                                                      |
| Pretext                | The fabricated scenario used to make a social-engineering interaction appear legitimate. |
| Technology Stack       | Technologies and platforms used to build or operate an environment.                      |
| Entity                 | A person, organization, domain, technology, document, or other identifiable object.      |

```text
Social-engineering vectors

                    Person
                   /      \
                Role      Organization
                 |             |
             Technology       Domain
                 \             /
                  Public Artifact
```

The key distinction is:

| Evidence                                  | Interpretation               |
| ----------------------------------------- | ---------------------------- |
| One profile mentions a product            | Weak signal                  |
| Multiple employees mention it             | Stronger signal              |
| Employees + job posting mention it        | Strong signal                |
| Public technical artifact corroborates it | Strongest passive hypothesis |

# Section 4 — Tools and commands

Use these commands against domains and identities you are authorized to investigate. The supplied material specifically covers `whois`, `dig`, and `dnsmap` as reconnaissance tooling, while also identifying Maltego and other OSINT tools for relationship analysis.

| Tool      | Command                                       | What it finds/shows                           | When to use it                         |
| --------- | --------------------------------------------- | --------------------------------------------- | -------------------------------------- |
| `whois`   | `whois example.test`                          | Registration and domain information           | Establish domain context               |
| `dig`     | `dig example.test A`                          | DNS records                                   | Correlate domains with infrastructure  |
| `dig`     | `dig example.test MX`                         | Mail infrastructure                           | Understand organizational mail routing |
| `dig`     | `dig example.test TXT`                        | TXT records                                   | Identify published DNS metadata        |
| `dnsmap`  | `dnsmap example.test`                         | Candidate DNS names                           | Controlled domain enumeration          |
| `grep`    | `grep -Ei 'aws\|azure\|kubernetes' notes.txt` | Technology mentions                           | Extract signals from collected notes   |
| `sort`    | `sort -u technologies.txt`                    | Unique technology signals                     | Normalize collected observations       |
| `Maltego` | GUI entity/relationship graph                 | People, domains, organizations, relationships | Visual correlation                     |

Example:

```bash
whois example.test
```

Interpretation:

```text
Domain: example.test
Registrar: ...
Name Server: ...
```

The useful result is organizational/domain context, not a target list.

Example:

```bash
dig example.test A
```

Possible output:

```text
example.test.    300    IN    A    192.0.2.10
```

Interpretation:

```text
Domain
  ↓
IPv4 address
  ↓
Candidate infrastructure relationship
```

Example:

```bash
dig example.test MX
```

Possible output:

```text
example.test.    IN    MX    10 mail.example.test.
```

This identifies published mail-routing information.

Example:

```bash
dig example.test TXT
```

Possible output:

```text
example.test.    IN    TXT    "v=spf1 ..."
```

Treat DNS metadata as evidence about configuration, not proof of a particular employee's identity.

Example:

```bash
dnsmap example.test
```

Possible output:

```text
[+] Searching for subdomains
mail.example.test
vpn.example.test
dev.example.test
```

The important analyst action is validation.

A discovered name becomes:

```text
Candidate
   ↓
Corroborate
   ↓
Classify
   ↓
Record confidence
```

Maltego is particularly useful when the dataset becomes relational: it can connect names, email addresses, organizations, domains, DNS names, affiliations, documents, and other entities into a graph.

Example graph:

```text
Organization
 ├── Employee-A
 │     └── DevOps
 │          └── Kubernetes
 ├── Employee-B
 │     └── Cloud
 │          └── AWS
 └── example.test
       └── DNS records
```

# Section 5 — Defender detection

- Pure passive collection of public professional information generally produces **no event on the organization's servers**.
- Monitor identity-provider and web analytics for unusual automated access patterns where platform telemetry is available.
- Monitor public-facing document repositories and source-control exposure for accidental technology disclosures.
- Track unexpected aggregation of employee names, roles, technologies, and organizational relationships during authorized exposure assessments.
- Defenders commonly miss the **correlation problem**: individually harmless disclosures become sensitive when combined.
- Reduce exposure by minimizing unnecessary technology details in public profiles, job postings, presentations, screenshots, and documents.
- During a red-team exercise, operators reduce their footprint by keeping collection passive, respecting platform controls, avoiding unnecessary authentication, and recording provenance rather than repeatedly querying the same sources.

# Section 6 — Lab task

**Platform:** Local Kali VM using a deliberately fabricated organization dataset; no real employee targeting or third-party scraping.

**Objective:** Prove that you can build a defensible social-vector map from public-style data and separate observed facts from hypotheses.

**Steps:**

1. Create a directory named `social-vector-lab`.
2. Create `people.csv` containing five fictional employees, roles, and fictional technology statements.
3. Create `sources.md` containing the source URL/title, observation, date, and confidence for every record.
4. Create `technologies.txt` containing technologies extracted from the fictional dataset.
5. Normalize the list and identify repeated technology signals.
6. Create a relationship graph showing people → roles → technologies → fictional organization.
7. Add three hypotheses and label each `Observed`, `Corroborated`, or `Unverified`.
8. Verify that no real person's personal information is present.
9. Write a short `findings.md` describing the strongest and weakest correlations.

**Expected output:**

```text
social-vector-lab/
├── people.csv
├── sources.md
├── technologies.txt
├── findings.md
└── graph/
    └── social-vector-map.png
```

Success means you can trace every relationship back to a source and explain why a finding is strong, weak, or unverified.

**Git artifact:**

```bash
git add social-vector-lab/
git commit -m "Add social vector mapping reconnaissance lab"
```

# Section 7 — Common mistakes

1. **Mistake:** Treating one profile as ground truth.
   **Why:** Profiles become stale.
   **Instead:** Corroborate important findings.

2. **Mistake:** Collecting names without roles.
   **Why:** A name alone has little reconnaissance value.
   **Instead:** Record role, department, technology, source, and confidence.

3. **Mistake:** Confusing technology mention with exposure.
   **Why:** An AWS reference does not prove an AWS service is externally reachable.
   **Instead:** Keep technology evidence separate from infrastructure evidence.

4. **Mistake:** Mixing facts and assumptions.
   **Why:** This creates false pivots.
   **Instead:** Explicitly label observations and hypotheses.

5. **Mistake:** Ignoring provenance.
   **Why:** You cannot reproduce or challenge the finding later.
   **Instead:** Record source and collection date.

6. **Mistake:** Optimizing for quantity.
   **Why:** Hundreds of weak relationships create analyst noise.
   **Instead:** Prioritize corroborated relationships.

7. **Mistake:** Jumping directly from a person to social engineering.
   **Why:** It skips the analytical stage and encourages targeting based on weak evidence.
   **Instead:** Use the human map to understand organizational attack surface and validate hypotheses in the authorized technical assessment.

# Section 8 — Move-on gate

1. **Build a map:**
   Starting with a fictional organization dataset, create a people → roles → technologies graph and correctly identify at least five relationships without looking at your notes.

2. **Validate evidence:**
   Take ten observations, classify each as `Observed`, `Corroborated`, or `Unverified`, and correctly justify every classification from its recorded provenance.

3. **Perform the pivot:**
   Starting from one fictional technology signal, trace it through the dataset to an organization, role, and fictional infrastructure hypothesis, while correctly identifying which step remains unverified.
