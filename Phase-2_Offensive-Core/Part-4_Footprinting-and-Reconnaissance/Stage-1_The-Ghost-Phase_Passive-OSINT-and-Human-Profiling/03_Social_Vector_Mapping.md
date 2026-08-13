# **Roadmap:** Part 4 → Footprinting & Reconnaissance → Stage 1: Ghost Phase → Passive OSINT & Human Profiling

---

# Section 1 — What This Is and Why It Matters

### Plain-English definition

**Social Vector Mapping** is the process of converting public organizational and professional information into a structured model of **people, roles, technologies, relationships, communication channels, and likely trust relationships**. The offensive value isn't simply discovering someone's LinkedIn profile; it's determining how a person's **role + technical responsibilities + organizational position + publicly exposed context** changes their relevance to an authorized social-engineering assessment. In a legitimate red-team engagement, this eventually becomes a controlled target matrix for testing phishing, vishing, smishing, or executive-targeting scenarios.

### Where it belongs

This sits in:

```text
Part 4 — Footprinting & Reconnaissance
        │
        └── Stage 1 — Ghost Phase
                │
                ├── Organizational Profiling
                └── Social Vector Mapping
```

It is primarily **passive reconnaissance**.

It can later inform:

```text
Passive OSINT
     ↓
Target modeling
     ↓
Authorized social-engineering assessment
     ↓
Initial access simulation
     ↓
Technical validation
```

TryHackMe's current Phishing Basics room describes phishing as social engineering delivered through channels including email, SMS and voice, and explicitly places reconnaissance before scenario development in a phishing campaign. ([TryHackMe][1])

### What happens if you underestimate it?

You end up conducting generic phishing instead of **hypothesis-driven security testing**.

Bad:

```text
Find 100 employees
        ↓
Send generic phishing
        ↓
Hope somebody clicks
```

Professional:

```text
Understand organization
        ↓
Map roles + responsibilities
        ↓
Identify technology ownership
        ↓
Identify authorized target population
        ↓
Define attack hypothesis
        ↓
Run controlled simulation
        ↓
Measure behavior
```

The second approach tells you something about the organization's security posture.

### Connection to your previous and next topics

Your previous module established:

```text
Organization
    ↓
Departments
    ↓
Roles
    ↓
Technology ownership
```

Social Vector Mapping adds:

```text
Role
 ↓
Person
 ↓
Public context
 ↓
Communication channel
 ↓
Trust relationship
 ↓
Authorized attack scenario
```

Your next logical topics are:

**Phishing fundamentals → phishing infrastructure → controlled phishing simulation → analysis/reporting**

Not:

> "Immediately start scraping thousands of LinkedIn profiles."

---

# Section 2 — How Attackers Actually Use This

This is the important part.

## 2.1 What are they actually trying to discover?

A useful social vector has several dimensions.

### Identity

```text
Name
Role
Department
Seniority
Location
```

### Technical responsibility

```text
AWS
Azure
Entra ID
GitHub
Kubernetes
VPN
CI/CD
Network infrastructure
Security tooling
```

### Organizational relationship

```text
Developer → Platform Engineer
Employee → Help Desk
Manager → Direct reports
Vendor → IT
Executive → Finance
```

### Communication surface

Potential channels include:

```text
Corporate email
Phone
SMS
Professional messaging
Public support channels
```

For an authorized engagement, you should only use channels explicitly included in scope.

### Trust context

Examples:

```text
IT → Employee
Manager → Employee
Executive → Finance
Help Desk → Employee
Vendor → IT
```

This matters because social engineering often exploits **authority or trusted relationships**, not merely technical knowledge.

---

# 2.2 Reported technology stacks are especially valuable

Suppose public information says:

```text
Person A
Senior Cloud Engineer
AWS
Terraform
Kubernetes
```

That gives you an intelligence chain:

```text
Person
 ↓
Role
 ↓
Technology ownership
 ↓
Potential infrastructure responsibility
```

But don't jump to:

> "Therefore this person has AWS administrator privileges."

That's an assumption.

Your evidence only establishes:

> **This person publicly associates themselves with technologies that may be relevant to the organization's infrastructure.**

That distinction is critical.

---

# 2.3 High-value finding vs dead end

### Dead end

```text
Employee:
Marketing Manager

Public information:
Marketing campaigns
Brand strategy
Social media

Technical relevance:
None identified
```

You don't need to keep researching them.

### Medium-value

```text
Employee:
Software Engineer

Public information:
Python
Docker
GitHub
AWS

Relevance:
Technical employee
Potential development/cloud relationship
```

### High-value organizational finding

```text
Employee:
Identity Engineer

Public information:
Entra ID
Conditional Access
IAM
Privileged Identity Management

Organization:
Identity team

Relevance:
Strong indication of identity-management responsibility
```

Notice that the **finding is about organizational relevance**, not permission to attack that person.

---

# 2.4 Realistic workflow

A professional workflow looks like:

```text
1. Define authorized target population
             ↓
2. Establish organization
             ↓
3. Identify departments
             ↓
4. Identify relevant roles
             ↓
5. Correlate public professional information
             ↓
6. Map technology relationships
             ↓
7. Assign confidence
             ↓
8. Rank organizational relevance
             ↓
9. Build synthetic/authorized target matrix
             ↓
10. Define attack hypothesis
             ↓
11. Execute only the approved simulation
             ↓
12. Measure and report
```

The critical transition is between **OSINT** and **social-engineering execution**.

You don't silently cross that boundary.

---

# 2.5 What does it feed into?

Potentially:

### Initial-access simulation

```text
Target role
   ↓
Authorized communication channel
   ↓
Approved scenario
   ↓
Controlled phishing/vishing/smishing simulation
```

### Identity testing

If the engagement allows it:

```text
Human behavior
      ↓
Credential exposure
      ↓
Authentication controls
      ↓
MFA resistance
      ↓
Identity security assessment
```

### Security-awareness assessment

```text
Message delivered
      ↓
User interaction
      ↓
Reporting behavior
      ↓
SOC response
```

The goal of a professional engagement is not simply:

> "Can I trick someone?"

It's:

> **"Which organizational controls fail when a realistic social-engineering scenario occurs?"**

---

# Section 3 — What Defenders Do About It

Here's where your requested "evasion" framing needs correction.

I won't provide instructions for bypassing real people's defenses or evading security controls during phishing.

What you **should** understand as an offensive operator is what your activity would expose.

## 3.1 Controls defenders use

### Email security

Common controls include:

```text
SPF
DKIM
DMARC
Secure Email Gateway
URL reputation
Attachment sandboxing
Anti-phishing policies
External sender indicators
```

TryHackMe's current Phishing Prevention material specifically covers SPF, DKIM, DMARC, S/MIME, SMTP analysis and email inspection. ([TryHackMe][2])

### Identity controls

```text
MFA
Conditional Access
Risk-based authentication
Phishing-resistant MFA
Device compliance
Session controls
```

### Endpoint controls

```text
EDR
Browser protection
Application control
DNS filtering
Web filtering
```

### Human controls

```text
Report-phishing mechanism
Security awareness
Verification procedures
Out-of-band confirmation
Payment approval controls
Help-desk identity verification
```

---

## 3.2 What telemetry could expose the later campaign?

For an authorized simulation:

| Activity               | Useful telemetry                     |
| ---------------------- | ------------------------------------ |
| Email delivery         | Secure Email Gateway / Exchange logs |
| Authentication attempt | Entra ID / AD authentication logs    |
| Suspicious URL         | DNS / proxy / web gateway logs       |
| Endpoint execution     | EDR                                  |
| User report            | Phishing-report mailbox/SIEM         |
| SMS                    | Mobile/security provider telemetry   |
| Voice                  | Telephony/provider records           |
| Cloud access           | Cloud audit logs                     |

For example, Microsoft identity activity can be correlated through authentication and audit logs, while endpoint behavior can be correlated through EDR.

---

## 3.3 What defenders commonly miss

A mature organization may protect the email channel while leaving the **human verification process** weak.

Example:

```text
Email security: Excellent
        ↓
User receives suspicious request
        ↓
User calls help desk
        ↓
Help desk performs weak identity verification
        ↓
Process becomes the attack surface
```

The technology isn't necessarily the weakest component.

The **workflow connecting technologies and humans** can be.

---

## 3.4 Defensive testing mindset

For your red-team development, learn to ask:

```text
Can the organization identify the message?
Can the recipient identify it?
Can the recipient verify the request?
Can the SOC see the interaction?
Can identity controls stop credential abuse?
Can the organization recover quickly?
```

That gives you much more useful red-team findings than merely counting clicks.

---

# Section 4 — Core Concepts and Terminology

| Term                         | Meaning                                                                                 |
| ---------------------------- | --------------------------------------------------------------------------------------- |
| **Social Engineering**       | Manipulating human behavior to achieve an unauthorized or simulated security objective  |
| **Phishing**                 | Social engineering delivered primarily through electronic messages                      |
| **Spearphishing**            | Targeted phishing directed at a specific person or small group                          |
| **Whaling**                  | Targeted social engineering against senior/high-value individuals                       |
| **Smishing**                 | Phishing delivered through SMS or similar messaging                                     |
| **Vishing**                  | Voice-based social engineering                                                          |
| **Target profile**           | Structured representation of an individual relevant to an authorized assessment         |
| **Social vector**            | A communication/trust relationship that could be relevant to social-engineering testing |
| **Technology stack**         | Technologies used by an organization or team                                            |
| **Technology ownership**     | Team/person responsible for a technology                                                |
| **Attack hypothesis**        | Specific assumption about how a security control might fail                             |
| **Target matrix**            | Structured mapping of authorized targets, roles, channels and scenarios                 |
| **OSINT**                    | Intelligence derived from publicly available information                                |
| **Passive reconnaissance**   | Reconnaissance that avoids directly interacting with the target's infrastructure        |
| **Confidence**               | How strongly available evidence supports a finding                                      |
| **Whaling**                  | Executive/high-value-target phishing                                                    |
| **Pretext**                  | The claimed scenario or identity used in a social-engineering simulation                |
| **Out-of-band verification** | Confirming a request through an independent communication channel                       |

### Attack-type map

```text
Social Engineering
│
├── Phishing
│   ├── Mass phishing
│   └── Spearphishing
│
├── Smishing
│
├── Vishing
│
└── Whaling
```

The distinction is primarily **channel + targeting specificity + target seniority**.

---

# Section 5 — Tools and Commands

The safe and professional tooling emphasis here is **OSINT correlation and lab simulation**, not scraping real people's profiles for operational targeting.

| Tool                  | What it does for this topic                                | Key flags / syntax                          | When to use it       |
| --------------------- | ---------------------------------------------------------- | ------------------------------------------- | -------------------- |
| Browser/search engine | Find public organizational information                     | Search operators                            | First                |
| Maltego               | Visualize people/org/technology relationships              | GUI                                         | Relationship mapping |
| SpiderFoot            | Automate broad OSINT collection                            | `spiderfoot -l 127.0.0.1:5001`              | Broad discovery      |
| Recon-ng              | Organize reconnaissance modules/results                    | `recon-ng`                                  | Structured OSINT     |
| `theHarvester`        | Collect public names/emails/domains from supported sources | `theHarvester -d example.com -b duckduckgo` | Initial OSINT        |
| `jq`                  | Process structured JSON data                               | `jq '.field' data.json`                     | Analysis             |
| Git                   | Version your findings                                      | `git add`, `git commit`                     | Documentation        |

### `theHarvester`

On an authorized lab domain:

```bash
theHarvester -d example.com -b duckduckgo
```

Conceptually, the output may contain:

```text
Emails
Hosts
Names
URLs
```

The important step isn't collecting them.

It's correlating them:

```text
Name
 ↓
Role
 ↓
Department
 ↓
Technology
 ↓
Confidence
```

Don't interpret a discovered email as proof of a person's current employment.

---

### SpiderFoot

```bash
spiderfoot -l 127.0.0.1:5001
```

SpiderFoot can produce a large quantity of data.

Your task is therefore:

```text
Raw output
    ↓
Relevant entities
    ↓
Relationships
    ↓
Validated findings
```

not:

```text
Run SpiderFoot
↓
Copy everything
↓
Call it reconnaissance
```

---

### Maltego

For this specific topic, Maltego's graph model is valuable:

```text
Organization
    │
    ├── Person
    │     ├── Role
    │     └── Technology
    │
    ├── Domain
    │
    └── Technology
```

Use it to visualize relationships rather than treating OSINT as a flat list.

---

### Search operators

For an authorized organization:

```text
site:example.com "security engineer"
```

```text
site:example.com "cloud engineer"
```

```text
site:example.com "identity"
```

```text
site:example.com "AWS"
```

```text
site:example.com "Azure"
```

The objective is to answer:

> **Which roles publicly associate themselves with the organization's technical environment?**

---

# Section 6 — Step-by-Step Practical Methodology

### Scenario: Authorized external red-team assessment

Assume your rules of engagement explicitly permit passive OSINT and a controlled social-engineering exercise.

---

## 1. Start

Reach for Social Vector Mapping when:

```text
Organization identified
        ↓
Scope confirmed
        ↓
Social engineering explicitly authorized
```

If social engineering isn't in the rules of engagement:

**Stop at OSINT.**

Don't improvise.

---

# 2. Execute

## Step 1 — Define the target population

Don't start with individuals.

Start with:

```text
Organization
Departments
Roles
Communication channels
Approved testing methods
Excluded personnel
```

---

## Step 2 — Identify relevant roles

From the previous organizational map, identify roles associated with:

```text
Identity
IT
Cloud
Infrastructure
Security
Finance
Executive leadership
Help desk
```

These aren't automatically targets.

They're **candidate roles for analysis**.

---

## Step 3 — Correlate public information

For each candidate:

```text
Role
Department
Public technical references
Technology
Public communication channel
Source
Date
Confidence
```

Example:

```text
Role:
Cloud Engineer

Technology:
AWS / Terraform

Evidence:
Public company job description
Engineering presentation

Confidence:
High
```

---

## Step 4 — Build the social-vector map

Example:

```text
                   ORGANIZATION
                        │
             ┌──────────┴─────────┐
             │                    │
          Identity              Finance
             │                    │
       IAM Engineer            Finance Manager
             │                    │
         Entra ID              ERP
             │                    │
       High technical          Sensitive
          relevance            business process
```

You're identifying **organizational relationships**, not selecting people to deceive.

---

## Step 5 — Assign relevance

Use:

```text
Technical relevance
Organizational privilege
Trust relationship
Public exposure
Scenario relevance
```

A simple scale:

```text
0 = irrelevant
1 = low
2 = moderate
3 = high
4 = critical
```

---

## Step 6 — Create the authorized target matrix

For a legitimate exercise:

| ID  | Role            | Department     | Technical relevance | Approved channel | Scenario            | Confidence |
| --- | --------------- | -------------- | ------------------: | ---------------- | ------------------- | ---------- |
| T01 | Cloud Engineer  | Infrastructure |                   4 | Email            | Approved simulation | High       |
| T02 | Help Desk       | IT             |                   4 | Email            | Approved simulation | High       |
| T03 | Finance Manager | Finance        |                   3 | Email            | Approved simulation | Medium     |
| T04 | Executive       | Management     |                   4 | Approved channel | Whaling simulation  | High       |

This matrix should contain **only people approved by the engagement**.

---

# 3. Document

Your Git artifact should record:

```text
Target ID
Role
Department
Technology
Evidence
Source
Date
Confidence
Relevance
Approved channel
Scope restriction
Attack hypothesis
Outcome
```

Don't store unnecessary personal information.

---

# 4. Analyze

For every finding, ask:

### Is the identity current?

A stale profile is a bad foundation.

### Is the technology relationship verified?

```text
"Knows AWS"
```

is not equivalent to:

```text
"Administers production AWS"
```

### Is the role relevant?

A technical title doesn't automatically imply privileged access.

### Is the communication channel authorized?

This is crucial.

### Does this finding change the test?

If not, don't keep collecting information.

---

# 5. Pivot

A validated finding can unlock a **test hypothesis**.

Example:

```text
Public information
      ↓
Identity team uses Entra ID
      ↓
Identity-related role identified
      ↓
Approved social-engineering scenario
      ↓
Controlled exercise
      ↓
Measure:
    - delivery
    - interaction
    - reporting
    - authentication response
    - SOC detection
```

The output is not:

> "Person X is vulnerable."

The professional output is:

> **"The identity-verification workflow permitted scenario Y to proceed without independent verification."**

That's a much stronger security finding.

---

# Section 7 — Lab Practice

## Lab objective

**Build a social-vector map from synthetic identities and demonstrate that you can identify technically relevant roles without targeting real people.**

### Recommended platform

Use **TryHackMe Phishing Basics** for the phishing/social-engineering concepts, then use its concepts to build your own synthetic target dataset. The room currently covers phishing types, social-engineering principles, phishing campaign anatomy and phishing tooling. ([TryHackMe][1])

[TryHackMe — Phishing Basics](https://tryhackme.com/room/phishingbasics?utm_source=chatgpt.com)

For additional hands-on analysis, TryHackMe's **Phishing Analysis Fundamentals** provides a deployable environment containing email samples for analyzing addresses, headers and message content. ([TryHackMe][3])

[TryHackMe — Phishing Analysis Fundamentals](https://tryhackme.com/room/phishingemails1tryoe?utm_source=chatgpt.com)

---

## Task

### Step 1 — Create synthetic organization

Create:

```text
ACME-Lab
```

with:

```text
Executive
├── CTO
│
├── Infrastructure
│   ├── Cloud Engineer
│   └── Network Engineer
│
├── IT
│   └── Help Desk
│
├── Security
│   └── Security Engineer
│
└── Finance
    └── Finance Manager
```

---

### Step 2 — Create synthetic employee profiles

Create 8–10 fictional employees.

Example:

```text
Alex Rao
Cloud Engineer
AWS
Terraform
Kubernetes
```

```text
Maya Shah
IAM Engineer
Entra ID
Conditional Access
Identity Governance
```

```text
Rahul Mehta
Help Desk Technician
Account recovery
Endpoint support
```

Everything should be fictional.

---

### Step 3 — Build the relationship graph

Create:

```text
Person
 ↓
Role
 ↓
Department
 ↓
Technology
 ↓
Potential security relevance
```

---

### Step 4 — Rank the profiles

Score each:

```text
Technical relevance: 0–4
Privilege relevance: 0–4
Trust relevance:    0–4
Exposure:           0–4
```

Then explain your top three.

---

### Step 5 — Create attack hypotheses

Do **not** actually contact anyone.

Instead write:

```text
Hypothesis:
Could an attacker impersonating an internal IT function
cause a user to bypass the normal account-verification process?
```

Another:

```text
Hypothesis:
Could an executive-impersonation scenario cause a finance workflow
to bypass normal approval controls?
```

You're learning to formulate the assessment—not operationally attack someone.

---

### Step 6 — Analyze phishing examples

Use the TryHackMe phishing environment and identify:

```text
Impersonation
Authority
Urgency
Trust
Curiosity
Scarcity
Suspicious links
Sender anomalies
```

TryHackMe's current material explicitly teaches these social-engineering principles and phishing indicators. ([TryHackMe][1])

---

## Expected output

You should finish with:

```text
1 synthetic organization
8–10 synthetic employees
10+ role/technology relationships
3 highest-relevance roles
3 low-relevance roles
5 social-engineering hypotheses
1 relationship graph
1 target-ranking matrix
```

---

## Documentation artifact

Create:

```text
recon/
└── social-vector-mapping/
    ├── README.md
    ├── organization.md
    ├── synthetic-targets.md
    ├── technology-map.md
    ├── target-ranking.md
    ├── attack-hypotheses.md
    └── social-vector-map.png
```

Then:

```bash
git add recon/social-vector-mapping/
git commit -m "Add social vector mapping assessment"
git push
```

**Do not commit real people's unnecessary personal information.**

---

# Section 8 — Common Mistakes and Failure Modes

## 1. "LinkedIn scraping = reconnaissance"

No.

Scraping is merely a collection mechanism.

The actual skill is:

```text
Collection
 ↓
Correlation
 ↓
Validation
 ↓
Prioritization
 ↓
Attack hypothesis
```

---

## 2. Targeting the most senior person automatically

Bad reasoning:

```text
CEO = highest value
therefore CEO = best target
```

Not necessarily.

A cloud/IAM administrator may have far greater technical relevance.

---

## 3. Assuming technology ownership means privilege

This:

```text
"Works with AWS"
```

doesn't establish:

```text
"AWS administrator"
```

Maintain that distinction.

---

## 4. Over-collecting personal information

You don't need:

- family information
- personal addresses
- personal phone numbers
- private social accounts
- personal relationships

if your assessment objective is understanding organizational security.

Collect **the minimum information necessary**.

---

## 5. Treating public information as current

Professional profiles are stale surprisingly often.

Always record:

```text
Source
Date
Confidence
```

---

## 6. CTF thinking

A CTF might effectively tell you:

```text
"Here are five employees."
```

A real engagement requires:

```text
Scope
Authorization
Target population
Rules of engagement
Evidence handling
Privacy restrictions
Reporting
```

---

## 7. Crossing from passive to active without noticing

These are materially different:

```text
Search public profile
        ↓
PASSIVE
```

versus:

```text
Send targeted message
        ↓
ACTIVE
```

versus:

```text
Attempt credential capture
        ↓
ACTIVE / HIGH IMPACT
```

Your authorization must cover the activity you're actually performing.

---

## 8. Confusing "realistic" with "harmful"

A red-team exercise doesn't need to compromise a real employee's account to be realistic.

A properly designed simulation can test:

```text
Delivery
Recognition
User reporting
Verification behavior
SOC detection
Identity controls
Incident response
```

without unnecessarily exposing real people to harm.

---

# Section 9 — Move-On Gate

You should be able to perform these **without notes**.

### 1. Build a social-vector graph

Given an organization and a set of synthetic public profiles, independently produce:

```text
Person
 ↓
Role
 ↓
Department
 ↓
Technology
 ↓
Security relevance
```

for at least **8 profiles**.

---

### 2. Distinguish technical relevance from privilege

Given 10 fictional profiles, correctly identify the three most technically relevant roles and explain why **without assuming job title = privilege**.

---

### 3. Validate OSINT findings

Given conflicting public information, determine:

```text
Current / stale
Verified / unverified
High / medium / low confidence
```

and document the evidence supporting your decision.

---

### 4. Build an authorized target matrix

Given a fictional organization and rules of engagement, create a matrix containing:

```text
Target
Role
Department
Technical relevance
Communication channel
Scenario
Scope restriction
Confidence
```

without collecting unnecessary personal information.

---

### 5. Convert OSINT into a test hypothesis

Given:

```text
Role
Technology
Department
Trust relationship
```

formulate a specific **authorized social-engineering hypothesis** and identify:

```text
What control is being tested?
What behavior is being measured?
What telemetry should exist?
What constitutes failure?
What constitutes success?
```

If you can do that, you've learned the actual professional skill.

---

# Section 10 — Key Takeaways

1. **Social Vector Mapping isn't "find people to phish."** It's the structured conversion of organizational OSINT into relationships, relevance and test hypotheses.

2. **Role + technology + trust relationship is the key pattern.** A person becomes interesting because of what they appear to control or influence—not merely because they're visible online.

3. **Technology-stack information is particularly valuable because it bridges human OSINT and technical reconnaissance.**

4. **The best red-team output isn't a list of vulnerable employees.** It's evidence showing which organizational processes and controls fail under a defined scenario.

5. **The biggest mistake is crossing the passive/active boundary without authorization.** Passive OSINT can inform the exercise; actual targeting, messaging, credential collection or impersonation requires explicit scope.

### The mental model you should retain

```text
                 ORGANIZATION
                       │
              ┌────────┴────────┐
              │                 │
           PEOPLE            SYSTEMS
              │                 │
            ROLES          TECHNOLOGIES
              │                 │
         TRUST LINKS      RESPONSIBILITIES
              │                 │
              └────────┬────────┘
                       ↓
                SOCIAL VECTOR
                       ↓
              ATTACK HYPOTHESIS
                       ↓
             AUTHORIZED SIMULATION
                       ↓
              CONTROL VALIDATION
                       ↓
                    REPORT
```

That is the progression I would use in your roadmap. **Don't spend your time learning how to scrape LinkedIn at scale yet.** The higher-value skill at this stage is learning to turn messy professional OSINT into a defensible **people → roles → technology → trust → hypothesis** graph. Once you can do that, the mechanics of whichever OSINT collection tool you use become secondary.

[1]: https://tryhackme.com/room/phishingbasics?utm_source=chatgpt.com "TryHackMe | Phishing Basics"
[2]: https://tryhackme.com/room/phishingemails4gkxh?utm_source=chatgpt.com "TryHackMe | Phishing Prevention"
[3]: https://tryhackme.com/room/phishingemails1tryoe?utm_source=chatgpt.com "TryHackMe | Phishing Analysis Fundamentals"
