# Organizational Hierarchy — Audience Profiling

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

**The important correction up front:** This is **not** about collecting as many employee names as possible. That is low-quality OSINT. The real objective is to reconstruct **who has access to what, who influences whom, which roles are trusted, which teams expose useful information, and where organizational privilege and trust converge**.

---

# Section 1 — What it is and where it sits

Organizational hierarchy profiling uses publicly available information to reconstruct an organization's people, departments, responsibilities, technology ownership, and trust relationships. You are turning scattered data — company pages, job descriptions, conference talks, GitHub activity, press releases, engineering blogs — into an **attack-relevant map**.

```text
Recon Phase
└── Passive OSINT
    ├── Domain / DNS intelligence
    ├── Certificate intelligence
    ├── Search engine dorking
    ├── Organizational profiling  ← YOU ARE HERE
    └── Historical / archival OSINT
         ↓
    Active Reconnaissance → Scanning → Enumeration
```

This feeds later phases: **Initial Access → Privilege Escalation → Lateral Movement → Persistence**

But the profiling itself stays passive during this stage.

**What breaks if you skip it:** You can have excellent exploitation skills and still waste an engagement because you don't understand the org. Knowing that IT uses Entra ID, that the platform team owns AWS/Terraform, and that help desk can reset accounts — before you send a single packet — dramatically narrows what matters.

For example, a job posting alone can tell you:

```text
Job posting
     ↓
"Azure AD / Entra ID administrator"
     ↓
Identity infrastructure identified
     ↓
Cloud + identity becomes an important attack surface
     ↓
Later: AD / Entra enumeration
```

Or from a public GitHub profile:

```text
Employee profile → "Senior DevOps Engineer"
     ↓
GitHub activity → Terraform / AWS / Kubernetes
     ↓
Cloud infrastructure becomes relevant
     ↓
Later: cloud enumeration
```

---

# Section 2 — How attackers actually use this

## 2.1 What are they looking for?

Don't collect information randomly. Build around four questions.

### A. Who controls identity?

Look for roles such as: Identity Administrator, IAM Engineer, Active Directory Administrator, IT Administrator, Security Administrator, Cloud Identity Engineer, Help Desk / Service Desk.

Why these specifically? Because identity is often the control plane for everything else. Compromise the identity administrator → access to every account they manage.

### B. Who controls infrastructure?

Look for: Cloud Engineers, DevOps Engineers, SREs, Infrastructure Engineers, Network Engineers, Systems Administrators, Platform Engineers.

Technology keywords from job postings are extremely useful:

```text
AWS  Azure  GCP  Kubernetes  Terraform  Ansible  Docker  Jenkins
GitLab  GitHub Actions  Active Directory  Entra ID  VMware
VPN  Cisco  Palo Alto  Fortinet
```

A job posting saying:
> "Maintain AWS IAM, Terraform infrastructure and Kubernetes clusters"

is far more valuable than "Software Engineer" — it reveals **role + technology ownership** in one sentence.

### C. Who has high business privilege?

CFO, CTO, CIO, VP Engineering, Head of Infrastructure, Head of IT, Head of Security, Procurement leadership.

But don't assume executives are automatically the best targets. A CEO may have enormous authority but little technical access. A junior IAM administrator may have significantly more useful technical privilege.

### D. Who is trusted but less hardened?

This is the "weakest link" concept — but don't reduce it to "find the dumb employee." That's amateur thinking.

**Attack value = Privilege × Access × Trust × Exposure**

| Role              | Privilege | Access    | Trust  | Exposure | Value        |
| ----------------- | --------: | --------: | -----: | -------: | -----------: |
| CEO               | High      | Medium    | High   | High     | High         |
| Help Desk         | Medium    | High      | High   | High     | High         |
| Junior Developer  | Low       | Medium    | Medium | High     | Medium       |
| IAM Administrator | Very High | Very High | High   | Medium   | **Critical** |
| HR Assistant      | Low       | Medium    | High   | High     | Medium       |

The "weakest link" may therefore be a **high-trust operational role**, not an executive. Help Desk can reset passwords. IAM Admin can create privileged accounts. These matter more than a high-ranking person with limited technical access.

---

## 2.2 Where do attackers find this information?

### Source 1 — Company website

```text
/about  /team  /leadership  /careers  /jobs
/news  /press  /security  /trust  /contact
```

Look for: names, titles, departments, technology references, locations, vendors, acquisitions, partnerships.

### Source 2 — Job postings (most underrated source)

A job description can reveal an entire technology chain:

```text
"Manage Entra ID, conditional access policies and privileged identity management."
```

This tells you:

```text
Identity
 ├── Entra ID
 ├── Conditional Access
 └── Privileged Identity Management
```

That's attack-surface intelligence, not just HR data. You now know the identity stack, the team responsible for it, and what tooling they use — all from a publicly posted job ad.

### Source 3 — LinkedIn / public professional profiles

Correlate across: current role, previous company, department, technologies, certifications, projects, conference presentations, public GitHub account, public personal website.

The important word is **correlate**. One profile is weak evidence. Five independent sources agreeing on the same organizational structure is strong evidence. Don't treat a single LinkedIn post as ground truth.

### Source 4 — GitHub (org repositories)

Look for public repositories associated with the organization:

```text
.github/
terraform/
ansible/
kubernetes/
docker/
helm/
.github/workflows/
README.md
SECURITY.md
```

You're not immediately hunting credentials. You're establishing:

```text
Repository → Technology → Team → Owner → Infrastructure
```

A Terraform repo tells you: this org uses infrastructure-as-code, likely on a cloud provider, managed by a specific team. That's technology ownership intelligence.

### Source 5 — Public documents

Search for: annual reports, technical whitepapers, conference presentations, engineering blogs, PDFs, procurement documents, security documentation, architecture diagrams accidentally published.

Documents often contain organizational terminology that normal webpages don't — internal team names, system names, vendor relationships, project codenames.

### Source 6 — Search engines

```text
site:example.com "CTO"
site:example.com "Head of Security"
site:example.com "IT Administrator"
site:example.com "Azure"
site:example.com "AWS"
site:example.com "Active Directory"
site:example.com "Kubernetes"
site:example.com filetype:pdf
```

You are trying to answer questions, not simply collect results.

---

## 2.3 High-value vs low-value finding

### Low-value

```text
John Smith
Software Engineer
```

No technology context, no privilege indicator. Not useful on its own.

### Medium-value

```text
John Smith
Senior DevOps Engineer
Public profile: AWS, Terraform, Kubernetes, GitHub
```

Now you have: Person → Role → Technology ownership → Potential infrastructure relationship.

### High-value

```text
Engineering organization
        ↓
Platform Team
        ↓
AWS infrastructure
        ↓
Terraform
        ↓
Senior Platform Engineer
        ↓
Public GitHub repositories
        ↓
Repository references internal naming conventions
```

This connects **human → technology → infrastructure**. That's the chain you're building toward. The value isn't the person's name — it's that you now know who owns what and where to look next.

---

## 2.4 Realistic attacker workflow

```text
1. Establish organization (name, domain, subsidiaries, brands)
         ↓
2. Identify departments (via company pages, press releases, org charts)
         ↓
3. Identify leadership (CTO, CISO, VP Eng via site search)
         ↓
4. Identify technical teams (job postings for cloud, identity, security)
         ↓
5. Identify technology owners (which team owns AWS? Entra ID? VPN?)
         ↓
6. Identify trust relationships (who can reset accounts? who approves access?)
         ↓
7. Correlate people ↔ technologies ↔ infrastructure
         ↓
8. Rank people/roles by attack relevance
         ↓
9. Identify next technical reconnaissance target
         ↓
10. Pivot to authorized active reconnaissance
```

Notice what isn't here: "Find 500 employees." That's useless without context.

**Pivot examples:**

```text
Job posting → Azure → Entra ID → Identity team → IAM administrator role
→ Authorized identity reconnaissance
```

```text
Engineering blog → Kubernetes → Platform team → Cloud infrastructure
→ Authorized infrastructure enumeration
```

The pivot is **never** "contact this employee." The correct next step is always toward the **authorized technical attack surface**.

---

## 2.5 Step-by-step engagement methodology

Engagement type: external penetration test / red-team reconnaissance. Authorization: `example.com`, passive OSINT permitted.

### Step 1 — Establish the organization

Record:
```text
Organization:
Primary domain:
Subsidiaries:
Brands:
Known locations:
Public technology references:
```

Don't immediately chase individuals.

### Step 2 — Identify leadership

```text
site:example.com leadership
site:example.com "Chief Technology Officer"
site:example.com "Chief Information Officer"
site:example.com "Chief Information Security Officer"
```

Build:
```text
Leadership
├── Technology
├── IT
├── Security
├── Engineering
└── Operations
```

### Step 3 — Identify technical departments

```text
site:example.com careers AWS
site:example.com careers Azure
site:example.com careers Kubernetes
site:example.com careers "Active Directory"
site:example.com careers DevOps
```

Build:
```text
Technology stack
├── Cloud
├── Infrastructure
├── Security
├── Development
├── Networking
└── Identity
```

### Step 4 — Identify technology owners

For each technology (AWS, Azure, Kubernetes, GitHub, VPN, Active Directory, Entra ID), ask: "Which team appears to own this?"

Record:
```text
Technology:
Owner (team/role):
Evidence:
Confidence:
```

### Step 5 — Correlate people

```text
Person:      Jane Doe
Role:        Senior Cloud Engineer
Evidence:    Company profile + public conference bio + engineering article
Technology:  AWS, Terraform, Kubernetes
Confidence:  High (3 independent sources)
```

### Step 6 — Build the graph

```text
                  COMPANY
                     │
     ┌───────────────┼──────────────┐
     │               │              │
    IT          Engineering      Security
     │               │              │
  Identity          Cloud           SOC
     │               │              │
  Entra ID          AWS         SIEM / EDR
     │               │
  IAM Team         DevOps
```

Then attach publicly confirmed roles to each node.

### Step 7 — Rank findings

Score each high-value role:

```text
Identity Administrator
  Privilege: 5/5
  Access:    5/5
  Trust:     5/5
  Exposure:  2/5
  Total:     17/20
```

That is much more useful than: "Found 73 employees."

### Step 8 — Document every meaningful finding

```text
Finding ID:         ORG-007
Entity:             Cloud Platform Team
Role:               Senior Platform Engineer
Department:         Engineering → Cloud
Technology:         AWS + Terraform
Relationship:       Owns cloud infrastructure provisioning
Source:             Public job posting + engineering blog
URL:                https://example.com/blog/...
Evidence:           2 independent sources
Confidence:         HIGH
Security relevance: AWS infrastructure ownership, IaC usage
Next pivot:         Authorized cloud/infrastructure reconnaissance
```

### Step 9 — Analyze each discovery (5 questions)

Before spending more time on a finding, ask:

1. **Is it verified?** One source or multiple independent sources?
2. **Is it current?** A 2022 LinkedIn profile may be useless in 2026.
3. **Does it reveal ownership?** `AWS` < `Platform Engineering → AWS`
4. **Does it reveal privilege?** `Developer` < `IAM Administrator`
5. **Does it create a pivot?** If no → discard. This is how you avoid rabbit holes.

### Step 10 — Pivot

```text
Public job posting → Azure → Entra ID → Identity team
→ IAM administrator role → Authorized identity reconnaissance
```

```text
Engineering blog → Kubernetes → Platform team → Cloud infrastructure
→ Authorized infrastructure enumeration
```

---

## 2.6 Where this feeds next

| Discovery | What it opens |
| --------- | ------------- |
| Identity infrastructure (Entra ID, AD) | Identity enumeration, credential attacks |
| Cloud platforms (AWS, Azure, GCP) | Cloud attack-surface mapping |
| Remote access (VPN, Citrix, RDP) | Initial access candidates |
| Third-party vendors | Supply chain / trust relationship attacks |
| Help-desk processes | Social engineering (if authorized) |
| Authentication technologies (SSO, MFA type) | Bypass research |

A compromised identity may also reveal the lateral movement path:

```text
IT → Infrastructure → Cloud → Security
```

Understanding this before you have any access tells you where a foothold might lead.

---

# Section 3 — Core concepts and terminology

| Term | Meaning |
| ---- | ------- |
| **Organizational hierarchy** | Formal structure: CEO → CTO/CIO → Engineering/IT → teams |
| **Stakeholder** | Person or group affected by a system/process (Engineering, IT, HR, Legal, Finance, vendors) |
| **Privileged role** | Domain Admin, Cloud Admin, IAM Admin, DB Admin — elevated access |
| **Trust relationship** | Entity A accepts requests/credentials/decisions from Entity B (Employee → Help Desk, Vendor → IT) |
| **Technology ownership** | Which team/role is responsible for a specific technology (e.g. IAM Team owns Entra ID) |
| **Attack path** | Public info → technical employee → technology → privileged account → critical system |
| **Weakest link** | Lowest-resistance route — NOT "least intelligent" — can be excessive privilege, poor process, or over-trusted role |
| **Confidence level** | How many independent sources confirm a finding (1 source = LOW, 3+ independent = HIGH) |
| **Organizational OSINT** | Intel focused on people, departments, technologies, infrastructure, business relationships |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
| ---- | ------- | ------------- | ----------- |
| Browser | Search operators | Primary OSINT collection | Always first |
| `theHarvester` | `theHarvester -d example.com -b duckduckgo` | Emails, domains, names from public sources | Initial collection |
| `recon-ng` | `recon-ng` → `marketplace search` | Modular OSINT workflows | Structured collection |
| `maltego` | GUI | Relationship graphs (org → person → tech) | Visualizing complex relationships |
| `spiderfoot` | `spiderfoot -l 127.0.0.1:5001` | Broad automated OSINT across many sources | Wide-net collection |
| `amass` | `amass enum -passive -d example.com` | Subdomains + asset mapping | After org OSINT reveals domain |
| `subfinder` | `subfinder -d example.com -silent` | Passive subdomain discovery | Technical pivot |

### theHarvester

```bash
theHarvester -d example.com -b duckduckgo
```

Conceptually:

```text
example.com
     ↓
public search sources
     ↓
emails / names / domains / hosts
```

Don't treat every returned email or name as authoritative. Validate against a second source before recording as a confirmed finding.

### recon-ng

```bash
recon-ng
> marketplace search
```

The workflow pattern:

```text
Source → Entity → Relationship → New entity → New source
```

Each module takes an entity type as input and produces another entity type. You chain them. The skill isn't memorizing every module — it's understanding what entity you have and what entity you need.

### Maltego — why it's specifically useful here

This is fundamentally a **relationship problem**:

```text
Company
   ├── Person → Job → Email → GitHub
   ├── Domain → Subdomain
   └── Technology → Infrastructure
```

That's exactly the graph Maltego builds. Use it when your spreadsheet of findings becomes too complex to reason about manually — when you need to see paths and connections, not just rows.

### SpiderFoot

```bash
spiderfoot -l 127.0.0.1:5001
```

Then access the local interface. Use for broad automated discovery, but don't blindly trust automated output. Automation produces leads — you perform validation.

### Passive domain pivot (after identifying org domains)

```bash
amass enum -passive -d example.com
# or
subfinder -d example.com -silent
```

These aren't organizational-hierarchy tools themselves. They become useful **after** org OSINT reveals a domain or infrastructure relationship worth pursuing.

---

# Section 5 — Defender detection

**Passive OSINT usually does not generate a useful event in the target's SIEM.** If you read a public company webpage, the target generally cannot see it in Microsoft Defender. That's why passive reconnaissance is powerful.

### What defenders actually monitor

They monitor their **external exposure**, not your individual session:

| Activity after OSINT turns active | Useful log source |
| --------------------------------- | ----------------- |
| Web requests to discovered URLs | Web server / CDN / WAF logs |
| Authentication attempts | Entra ID / AD sign-in logs |
| VPN access | VPN logs |
| DNS queries | DNS resolver logs |
| Port scanning | Firewall / IDS / network telemetry |
| Web enumeration | WAF / web logs |
| Cloud access | CloudTrail / Azure Activity Logs |
| Endpoint activity | EDR |

External monitoring tools defenders use: Google Search Console, Shodan monitoring, Censys, GitHub secret scanning, Have I Been Pwned domain monitoring, external ASM platforms.

### What defenders commonly miss

> "Our network is secure, therefore our organization is secure."

An attacker may already know who runs AWS, who manages identity, who manages VPN, what cloud provider is used, what technologies are deployed, and what vendors are trusted — before sending a single packet. The network perimeter is irrelevant until the attacker chooses to cross it.

### Staying passive — operational discipline

The strongest approach is simply **remaining passive**:

- Use public sources only
- Don't contact employees during passive recon
- Don't trigger password-reset workflows
- Don't authenticate to target systems
- Don't scrape aggressively
- Don't bypass access controls
- Maintain clear scope
- Distinguish verified facts from assumptions

Once you begin probing systems or interacting with personnel, you've crossed into active reconnaissance and potentially a different authorization requirement.

---

# Section 6 — Lab task

**Platform:** TryHackMe — [Red Team Recon](https://tryhackme.com/room/redteamrecon)

The room covers passive recon, advanced searching, specialized search engines, Recon-ng, and Maltego — including finding company-related emails, infrastructure, and public information.

**Also useful:** [Sakura](https://tryhackme.com/room/sakura) — challenges you to use passive OSINT only, no active contact with account owners.

**Objective:** Reconstruct a fictional organization's human/technical hierarchy from passive OSINT and produce a defensible attack-surface map without contacting any employee or target system.

**Steps:**
1. Establish org: name, domain, subsidiaries, brands, known locations
2. Identify leadership using `site:example.com "Chief Technology Officer"` style queries
3. Identify technical departments via job-posting searches (`careers AWS`, `careers Kubernetes`)
4. For each technology discovered, identify the owning team — record Technology / Owner / Evidence / Confidence
5. Correlate at least 5 public people to their role + technology using 3+ independent sources each
6. Rank 3 roles by `Privilege × Access × Trust × Exposure` — justify each ranking; don't just pick executives
7. Document 3 dead ends — findings that led nowhere (equally important to record)
8. Build the final graph: `Org → Department → Role → Technology → Infrastructure`

**Expected output:**
```text
1 organization profile
5+ person/role/technology relationships (each with confidence rating)
3 high-value roles (ranked and justified)
3 dead ends (documented)
1 organizational attack-surface graph
```

**Git artifact:**
```bash
mkdir -p recon/organizational-hierarchy
# Files: README.md, hierarchy.md, findings.md, sources.md
git add recon/organizational-hierarchy/
git commit -m "feat(recon): org hierarchy OSINT — example.com"
```

Don't commit personal data unnecessarily. Store only what's needed to demonstrate the technique within the lab's rules.

---

# Section 7 — Common mistakes

**1. Collecting names instead of relationships**
Bad: 73 employee names with no context. Good: `Sarah → IAM Administrator → Entra ID → Identity infrastructure`. The chain is the value — names without context are worthless.

**2. Assuming job title = privilege**
"Director" ≠ Administrator. "Junior Engineer" ≠ low privilege. Always investigate the responsibilities and technology ownership listed in the role — not the title on the business card.

**3. Trusting one source**
Profiles can be outdated, incorrect, incomplete, duplicated, or scraped incorrectly. Use multiple independent sources and record confidence explicitly for every finding.

**4. Ignoring job descriptions**
Job postings can expose remarkably detailed technology information — cloud stack, identity tooling, networking, EDR, CI/CD, databases, containers, security tooling. Often more useful than employee social-media profiles.

**5. OSINT rabbit holes**

```text
Employee → personal website → old employer
→ old GitHub → old project → another employee → another company → ...
```

Stop and ask: **Does this change my understanding of the authorized target?** If not, discard it. OSINT can run forever if you let it.

**6. Treating OSINT as automatically harmless**
Passive doesn't mean "anything goes." You still need authorization and must respect: scope, privacy, terms of service, applicable law, platform restrictions, engagement rules. Especially avoid contacting employees, triggering password-reset workflows, impersonation, or credential testing unless explicitly authorized.

**7. Confusing CTF behavior with professional recon**

CTF: `Target → find flag → exploit`

Real engagement:
```text
Scope → Objective → Evidence → Confidence → Risk → Authorization → Controlled testing
```

Your documentation discipline matters as much as your tool knowledge.

---

# Section 8 — Move-on gate

1. **Given only a company name and domain**, produce Leadership / Technical departments / Technology ownership / Relevant roles using only passive sources — without notes.

2. **Given a realistic job description**, extract: Department, Technology, Likely ownership, Potential privilege, Next reconnaissance pivot — without repeating the JD back verbatim.

3. **Given 10 organizational roles**, rank the top 3 using `Privilege × Access × Trust × Exposure` and defend each ranking with evidence. Selecting the CEO without justification fails this test.

4. **Produce a complete passive recon report** from an authorized domain: Org → Departments → People/roles → Technology ownership → Infrastructure relationships → High-value attack surfaces → Next authorized step — with sources, confidence ratings, and documented dead ends.

---

# Mental Model

```text
Don't ask: "Who works here?"

Ask: "Who controls what?"
  → Who owns identity?
  → Who owns infrastructure?
  → Who controls access?
  → Who supports users?
  → Who manages vendors?
  → Who has privileged responsibilities?

Then: "Which relationships create the shortest authorized path to a valuable technical asset?"
```

That is the real purpose of organizational hierarchy profiling in the Ghost Phase.
