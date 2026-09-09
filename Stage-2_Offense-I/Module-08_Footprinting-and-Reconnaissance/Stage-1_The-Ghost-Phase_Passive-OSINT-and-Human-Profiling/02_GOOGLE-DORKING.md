# Search Engine Hacking / Google Dorking

**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

**Important correction upfront:** Google Dorking does **not** bypass authentication. You are querying information a search engine has already indexed. The security issue is that the indexed information may reveal something the organization never intended to be discoverable.

---

# Section 1 — What it is and where it sits

**Search engine hacking is the deliberate use of search-engine operators to reduce a massive index into a small set of target-specific results that reveal an organization's public attack surface — documents, portals, technologies, and historical information.**

Instead of manually browsing `example.com/about`, `example.com/products`, `example.com/careers...`, you ask the search engine structured questions:

```text
site:example.com
site:example.com filetype:pdf
site:example.com intitle:"login"
```

The search engine becomes an **index-based reconnaissance interface**.

```text
Footprinting & Reconnaissance
│
├── Passive Reconnaissance
│   ├── Domain / DNS intelligence
│   ├── Certificate intelligence
│   ├── Organizational profiling
│   ├── Search Engine Hacking  ← YOU ARE HERE
│   └── Historical / archival OSINT
│
└── Active Reconnaissance → Scanning → Enumeration
```

**The important skill is not memorizing dorks.** It is learning to ask:

> *"What information would be useful at this point in the attack, and what search constraint could expose it?"*

A search result is not the end finding. It is usually a **pivot point**.

**What breaks if you skip it:** You jump straight to `nmap → gobuster → Burp` while ignoring information the organization has already published. A good passive result might tell you:

```text
Company → uses Microsoft Azure → has a public employee portal
→ portal hostname discovered → authentication technology identified
→ specific application becomes interesting
```

You discovered the path **before actively probing infrastructure**.

---

# Section 2 — How attackers actually use this

## 2.1 What attackers look for

Attackers answer concrete reconnaissance questions — not "find hacking stuff."

### Question 1 — What does the org expose?

```text
site:example.com
```

Potential discoveries:

- Public applications
- Documentation
- Old pages
- Subdomains referenced in indexed content
- Public PDFs

### Question 2 — What documents has the org published?

```text
site:example.com filetype:pdf
```

Potentially useful documents:

- Technical presentations
- Product documentation
- Public reports
- Job documents
- Architecture presentations
- Security documentation

**Key insight:** The document itself may contain information the webpage linking to it doesn't expose. A PDF buried behind a press release link may reveal infrastructure details the marketing page never mentions.

### Question 3 — Are authentication portals discoverable?

```text
site:example.com intitle:"login"
site:example.com inurl:login
```

Finding a login page does **not** mean you've found a vulnerability. It means:

> "This is an authentication surface worth investigating later."

That's a major distinction. A discovered login portal goes into your candidate list for authorized testing — not into your exploit queue.

---

## 2.2 High-value vs dead-end finding

This distinction matters more than knowing syntax.

### Dead end

```text
site:example.com
→ Home, About, Contact, Careers, Blog
```

Nothing particularly interesting. Don't waste 30 minutes staring at it — move to a narrower query.

### Medium-value

```text
site:example.com filetype:pdf
→ Annual Report.pdf, Product Brochure.pdf, Engineering Presentation.pdf
```

You open the engineering presentation. It mentions:

```text
AWS  Kubernetes  GitLab  Internal API
```

Now it gets interesting — you have technology intelligence to work with.

### High-value

The engineering presentation reveals:

```text
dev-api.example.com
staging.example.com
GitLab  Kubernetes
```

Now you have:

```text
Public document → Infrastructure naming → Potential hostname
→ Technology fingerprint → Active reconnaissance candidate
```

**The value isn't the document. It's what it allows you to discover next.**

---

## 2.3 Realistic attacker workflow

```text
Step 1 — Establish indexed footprint
→ site:example.com
→ Record: interesting pages, subdomains, apps, documents, terminology

Step 2 — Find documents
→ site:example.com filetype:pdf
→ site:example.com filetype:docx
→ site:example.com filetype:pptx
→ Prioritize: engineering, infrastructure, API, architecture, security docs
→ Don't blindly download everything — inspect titles and snippets first

Step 3 — Search authentication surfaces
→ site:example.com intitle:"login"
→ site:example.com inurl:login
→ Building candidate list: login.example.com, portal.example.com, auth.example.com

Step 4 — Generate new queries from findings (the real skill)
→ Document mentions "employee portal"? → site:example.com "employee portal"
→ Discover "GitLab" referenced? → site:example.com "GitLab"
→ Discover "API documentation"? → site:example.com "API documentation"

Step 5 — Extract infrastructure clues from everything found
→ hostname, subdomain, IP address, technology, vendor, app name,
→ employee naming convention, cloud provider

Step 6 — Pivot to active validation
→ PDF reveals dev.example.com
→ DNS validation → HTTP fingerprinting → port scanning → service enumeration
→ Pivot to: [next authorized phase]
```

This is **iterative**: finding → hypothesis → new query → new finding → hypothesis.

You don't just run predefined dorks. You generate new queries from what you discover.

The search engine didn't compromise anything — it **reduced the uncertainty of your next action**.

---

## 2.4 Step-by-step practical methodology

Engagement type: external penetration test. Authorization: `example.com`, passive scope.

### Step A — Establish the indexed footprint

```text
site:example.com
```

Record:
- interesting pages
- subdomains
- applications
- documents
- terminology

### Step B — Search documents

```text
site:example.com filetype:pdf
site:example.com filetype:docx
site:example.com filetype:pptx
```

Prioritize documents related to: engineering, infrastructure, APIs, architecture, security, administration. Ignore marketing brochures.

### Step C — Search authentication surfaces

```text
site:example.com intitle:"login"
site:example.com inurl:login
```

You are building a candidate list:

```text
login.example.com
portal.example.com
auth.example.com
```

### Step D — Generate new queries from findings

This is the real skill. Each result should generate new hypotheses:

```text
Discover "employee portal" in a PDF
→ site:example.com "employee portal"

Discover "GitLab" referenced
→ site:example.com "GitLab"

Discover "API documentation" linked
→ site:example.com "API documentation"
```

Pattern: **finding → hypothesis → query → finding → hypothesis**

### Step E — Classify every result (don't treat them equally)

| Level | Category | Example |
| ----- | -------- | ------- |
| 0 | Noise | Marketing brochure |
| 1 | Context | Technology name mentioned |
| 2 | Attack-surface intelligence | Specific hostname, authentication portal |
| 3 | Security-relevant exposure | Sensitive internal info, misconfigured access |

Don't confuse **interesting** with **vulnerable**. An authentication portal at `/admin/login` is Level 2 — attack-surface intelligence. You need further authorized testing to determine if it's Level 3 or simply a properly secured login page.

### Step F — Document everything (including failures)

Don't create just a `google-dorks.txt` file. That's too shallow.

```text
recon/
└── search-engine-recon/
    ├── README.md
    ├── queries.md
    ├── findings.md
    └── screenshots/
```

For every meaningful finding record:

```text
Query:           site:example.com filetype:pdf "API"
Result:          Public engineering presentation
URL:             https://example.com/...
Category:        Level 2 — Attack-surface intelligence
Why interesting: Mentions API architecture and development environment
Confidence:      MEDIUM (single source)
Next pivot:      Validate any discovered hostnames within scope
```

Also record **failed queries** — a professional recon notebook contains:

```text
What worked
What failed
What was noise
What changed my hypothesis
```

### Step G — Pivot

```text
PDF
 ↓
dev.example.com
 ↓
DNS validation
 ↓
HTTP fingerprinting
 ↓
port/service enumeration
```

Full reconnaissance chain:

```text
Search engine → Public document → Hostname → DNS → HTTP
→ Port/service → Application → Vulnerability research
```

This is exactly why search-engine reconnaissance belongs **before** active scanning.

---

## 2.5 Where findings feed next

| Search finding | Possible next phase |
| -------------- | ------------------- |
| Login portal | Web enumeration |
| Subdomain | DNS / HTTP enumeration |
| Technology name | Fingerprinting |
| API documentation | API enumeration |
| Employee names | Social-engineering assessment |
| Old document | Historical reconnaissance |
| Cloud provider | Cloud attack-surface mapping |
| Development hostname | Active reconnaissance |
| Software/vendor name | Vulnerability research |

Before acting on any finding, establish:

```text
Is it in scope?
Is it actually exposed?
Is it relevant?
Is it vulnerable?
```

---

# Section 3 — Core concepts and terminology

| Term | Meaning |
| ---- | ------- |
| **Search engine** | System that crawls, indexes, and retrieves web content |
| **Crawler** | Automated software that discovers and retrieves web resources |
| **Index** | Search engine's stored representation of discovered content |
| **Dork** | A deliberately constructed advanced search query |
| **Search operator** | Special syntax that restricts or modifies search results |
| **Passive reconnaissance** | Intelligence gathering without directly interacting with target infrastructure |
| **OSINT** | Intelligence derived from publicly available information |
| **Attack surface** | Collection of exposed assets and interfaces an attacker could interact with |
| **Pivot** | Using one discovery to guide the next reconnaissance action |
| **Content discovery** | Finding web resources that aren't obvious from normal navigation |
| **Metadata** | Information describing a file/resource rather than its primary content |
| **Indexed content** | Content known to and retrievable through a search engine |
| **robots.txt** | A file that tells compliant crawlers which paths to avoid — NOT an access control |

**Operator reference:**

| Operator | Purpose | Mental shortcut |
| -------- | ------- | --------------- |
| `site:` | Restrict results to a domain | **Where?** |
| `filetype:` | Restrict by file extension | **What kind of resource?** |
| `intitle:` | Term must appear in page title | **What does the page call itself?** |
| `inurl:` | Term must appear in URL | **What path pattern?** |
| `intext:` | Term must appear in page body | **What does the content say?** |
| `"phrase"` | Exact phrase match | **Exact wording** |
| `-term` | Exclude a term | **Not this** |

**Build queries, don't memorize lists.** Formula:

```text
TARGET + RESOURCE TYPE + INTERESTING TERM + EXCLUSION

site:example.com + filetype:pdf + "API" + -cloud
```

This formula is transferable to every target. Memorizing 500 dorks is not.

---

# Section 4 — Tools and commands

Google Dorking's primary tool is a **search engine** — you don't need Kali for the core technique.

| Tool | Command | What it does | When |
| ---- | ------- | ------------ | ---- |
| Google | `site:`, `filetype:`, `intitle:` | Primary dorking | Always first |
| Bing | Similar operators | Alternative index (different results) | Cross-checking |
| `curl` | `curl -I https://example.com/report.pdf` | Validate resource is accessible | After finding a URL |
| `wget` | `wget https://example.com/report.pdf` | Download authorized public resource | Document analysis |
| `exiftool` | `exiftool report.pdf` | Inspect document metadata | After downloading |
| `sha256sum` | `sha256sum report.pdf` | Hash file for evidence integrity | Evidence chain |
| Wayback Machine | `web.archive.org` | Historical web content | Historical recon |

### Full document inspection workflow

```bash
# 1. Validate it's accessible
curl -I https://example.com/public/report.pdf
```

```text
HTTP/2 200
content-type: application/pdf
content-length: 482931
```

Interpretation:
- `200` → resource is accessible
- `content-type` → server confirms it's a PDF
- `content-length` → approximately 470KB

```bash
# 2. Download it (where permitted)
wget https://example.com/public/report.pdf

# 3. Extract metadata
exiftool report.pdf
```

```text
Creator:     Microsoft PowerPoint
Author:      Engineering
Producer:    ...
Create Date: ...
```

The author field and creator app aren't a vulnerability. They are **another intelligence point** — the document was created by the Engineering team using PowerPoint, which tells you who might own the information it contains and what tooling they use.

---

# Section 5 — Defender detection

**Google Dorking itself generally does not create a server-side detection event.** When you run `site:example.com filetype:pdf`, Google receives the query. `example.com` may receive **no request whatsoever**. There is no "Google Dork detected" event in the target's web logs.

### What defenders actually monitor

They focus on the **exposure being discovered** and on subsequent activity:

| Activity | Log source that catches it |
| -------- | ------------------------- |
| Following a discovered URL (`GET /admin/login`) | Nginx / Apache / IIS / CDN / WAF logs |
| Automated path probing | WAF behavioral detection |
| Abnormal request rates | WAF / CDN rate limiting |
| Correlation of follow-up events | SIEM (fed by WAF, proxy, CDN, IdP) |

### What defenders commonly miss

**1. They remove the page but leave the file**

```text
old-page.html → removed
technical.pdf → still publicly accessible
```

**2. They assume robots.txt is access control**

`robots.txt` tells compliant crawlers what to avoid. It does **not** provide authentication or authorization. The path is still accessible to anyone who knows it.

```text
User-agent: *
Disallow: /admin/
```

doesn't mean `/admin/` is protected — it means compliant crawlers are asked not to crawl it.

**3. They forget historical exposure**

A current website may be clean while an older indexed or archived document still reveals useful information. The Wayback Machine doesn't forget.

**4. They focus on secrets and ignore architecture**

A document doesn't need to contain credentials to be dangerous. A document revealing:

```text
AWS  Kubernetes  GitLab  internal-api  staging
```

can significantly improve reconnaissance even without credentials present.

### Attacker evasion for this technique

There is no magic evasion technique. The professional lesson is:

> Passive search-engine queries are inherently difficult for the target to detect because the interaction occurs primarily with the search engine, not the target.

The defensive response therefore isn't "detect every Google query." It's:

```text
Minimize unnecessary public exposure
        +
Control access
        +
Continuously inspect indexed/public content
        +
Monitor subsequent active reconnaissance
```

---

# Section 6 — Lab task

**Platform:** TryHackMe — [Google Dorking](https://tryhackme.com/room/googledorking)

The room covers crawlers, indexing, `site`, `filetype`, `intitle`, and advanced Google searching.

**Objective:** Use search operators to discover, classify, document, and pivot from public information — without treating every result as a vulnerability.

**Steps:**
1. Complete the crawler/indexing section — understand: `crawler → content discovery → index → search result`
2. Practice exact phrases: compare `"passive reconnaissance"` vs `passive reconnaissance` — observe how precision changes results
3. Run `site:tryhackme.com` — record what kinds of results appear
4. Run `site:tryhackme.com filetype:pdf` — understand what changed and why
5. Run `site:tryhackme.com intitle:"login"` — record what appears
6. Build 5 queries yourself using `TARGET + TYPE + TERM + EXCLUSION` — don't copy a list, construct them from scratch
7. Classify every result Level 0–3 with written reasoning for each classification
8. Turn your best Level 2+ finding into a full pivot chain showing the next 3 authorized actions

**Expected output:**
```text
5 self-constructed queries (not copied)
All results classified with reasoning
1 complete pivot chain from a Level 2+ finding
1 documented failure (query that returned noise — equally important)
```

**Git artifact:**
```bash
mkdir -p recon/search-engine-recon
# Files: README.md, queries.md, findings.md, screenshots/
git add recon/search-engine-recon/
git commit -m "feat(recon): google dorking session — tryhackme.com"
```

**Optional follow-up:** TryHackMe — [Content Discovery](https://tryhackme.com/room/contentdiscoveryx) — combines dorking, Wappalyzer, Wayback Machine, GitHub, S3 discovery, and automated tools in one room.

---

# Section 7 — Common mistakes

**1. Treating every result as a vulnerability**

Finding:

```text
/admin/login
```

doesn't mean:

```text
/admin/login = vulnerability
```

It means: "authentication surface discovered." You need further authorized testing to determine if it's exploitable.

**2. Blindly copying "Google Dork lists"**

Bad methodology:

```text
500 dorks from GitHub → paste all → hope something pops up
```

Professional methodology:

```text
Target intelligence → question → query → result → hypothesis → new query
```

The methodology matters more than the list.

**3. Assuming Google indexes everything**

`Google finds nothing` ≠ `Target has nothing`. A resource can exist and be completely absent from Google's index. This is why recon eventually transitions to DNS, Certificate Transparency, Wayback, GitHub, and active discovery.

**4. Treating robots.txt as a security control**

`Disallow: /admin/` tells compliant crawlers not to crawl the path. It does NOT protect it. Authentication and authorization are security controls — robots.txt is a courtesy directive to well-behaved bots.

**5. Crossing passive into active mid-exercise**

```text
Google result → click → server receives request
```

If your engagement requires passive-only reconnaissance, don't start browsing every discovered URL. You may generate server-side logs and cross your scope boundary without realizing it.

**6. Testing outside scope**

Scope says `example.com`. Don't assume `example-bank.com`, `examplecdn.com`, or third-party vendors are authorized. Scope determines what you're allowed to test.

**7. Ignoring search-engine differences**

Operators and indexing behavior aren't perfectly consistent across search engines or over time. A query that worked six months ago may return fewer results, different results, or stop being supported. Treat dork syntax as a tool, not gospel.

**8. Searching for real secrets on random organizations**

Don't turn a learning exercise into finding someone's credentials. Use TryHackMe, HTB, your home lab, or explicitly authorized client scope. The fact that information is publicly indexed does **not** mean you have authorization to exploit it.

---

# Section 8 — Move-on gate

1. Given an authorized domain, construct queries using `site:`, `filetype:`, `intitle:`, `inurl:`, exact phrases, and exclusions to answer 5 different reconnaissance questions — **without consulting a cheat sheet**.

2. Given 10 search results, classify each as Level 0–3 and explain the reasoning for every single classification.

3. Given a public document containing a previously unknown hostname and technology name, extract the relevant intelligence, document it using the finding template, and produce the next 3 authorized reconnaissance actions in the correct order.

4. Perform a complete search-engine reconnaissance exercise against a lab target and commit an artifact containing: queries, results, screenshots, classified findings, failed queries, risk classification, and next pivots — such that someone else can reproduce your reasoning from the repository.

---

# Mental Model

Don't think:

> **"Which Google Dork finds a vulnerability?"**

Think:

> **"What do I need to know about this target right now, and can the search index answer it before I touch the target?"**

Full reconnaissance chain:

```text
Search → Discover → Validate → Correlate → Document → Pivot → Active reconnaissance
```

[1]: https://tryhackme.com/room/googledorking "TryHackMe | Google Dorking"
[2]: https://tryhackme.com/room/passiverecon "TryHackMe | Passive Reconnaissance"
[3]: https://tryhackme.com/room/contentdiscoveryx "TryHackMe | Content Discovery"
[4]: https://tryhackme.com/room/searchskills "TryHackMe | Search Skills"
