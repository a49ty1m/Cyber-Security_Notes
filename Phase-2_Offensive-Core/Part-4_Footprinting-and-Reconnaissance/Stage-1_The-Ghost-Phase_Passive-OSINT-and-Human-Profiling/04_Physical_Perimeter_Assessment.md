# Physical Perimeter Assessment
**Roadmap:** Part 4: Footprinting and Reconnaissance → Stage 1: The "Ghost" Phase (Passive OSINT & Human Profiling)

# Section 1 — What it is and where it sits

Physical Perimeter Assessment evaluates how publicly observable information and environmental conditions could enable unauthorized physical access or disclosure. The three focus areas are **shoulder surfing** (observing sensitive information), **tailgating** (following an authorized person through a controlled entrance), and **dumpster diving** (recovering information from discarded materials).

The output is a physical attack-surface model: entrances, reception controls, visibility, discarded information, access assumptions, and opportunities for observation.

```text
Recon
  └── Passive
       └── Physical Perimeter Assessment
            ├── Visibility
            ├── Access controls
            ├── Human procedures
            └── Discarded information
                  ↓
              Active Recon
                  ↓
          Physical validation
                  ↓
          Security assessment
```

If this skill is skipped, technically strong reconnaissance can miss simple physical exposure paths. The previous OSINT stage identifies people, locations, and organizational context; this stage turns those observations into physical-security hypotheses for later authorized validation.

# Section 2 — How attackers actually use this

## 2.1 What they look for

A physical assessment looks for:

- entrances and exits
- reception arrangements
- badge-controlled doors
- visitor procedures
- security guards
- cameras and visible monitoring
- doors that appear propped open
- publicly visible screens
- exposed badges or paperwork
- unattended documents
- discarded printouts
- shipping labels
- whiteboards
- meeting-room displays
- public Wi-Fi information
- loading areas
- contractor access
- delivery procedures
- employee behavior around controlled doors

The important question is not simply **"Can someone get inside?"**

It is:

```text
Can an outsider obtain information
        ↓
without triggering a control
        ↓
that enables a later security action?
```

## 2.2 Shoulder surfing workflow

An assessor first identifies locations where sensitive information could naturally become visible: reception areas, shared workspaces, elevators, meeting rooms, cafés, or public-facing waiting areas.

The assessment then records:

1. What information is potentially visible?
2. From what distance?
3. For approximately how long?
4. Is authentication information exposed?
5. Are screens automatically locked?
6. Are documents visible to visitors?
7. Is the exposure repeatable?

A dead-end finding might be:

```text
Screen visible from reception
→ displays generic company news
→ no sensitive information
→ low value
```

A high-value finding might be:

```text
Visitor-visible workstation
→ employee opens internal dashboard
→ customer/project identifiers visible
→ repeated exposure possible
→ significant information-disclosure finding
```

The finding does not require obtaining credentials. The exposure itself is the vulnerability.

## 2.3 Tailgating assessment

Tailgating depends on a gap between **technical access control** and **human access-control behavior**.

An assessor examines:

```text
Employee → badge reader → controlled door → interior
```

and asks:

- Does the door close automatically?
- Is anti-passback implemented?
- Does reception challenge unknown visitors?
- Are visitors escorted?
- Are contractors distinguishable?
- Does staff hold doors for others?
- Are delivery personnel handled differently?
- Can one person enter while another follows?

A high-value weakness is not "someone was friendly."

It is a repeatable process failure:

```text
Controlled entrance
       ↓
Employee routinely holds door
       ↓
No identity verification
       ↓
Visitor can reach controlled area
```

The next pivot is determining **what access that physical boundary protects**, rather than immediately attempting entry.

## 2.4 Dumpster-diving workflow

The assessor looks for information discarded outside the organization's intended security boundary.

Useful categories include:

- printed reports
- meeting notes
- organizational charts
- shipping labels
- equipment inventories
- configuration printouts
- employee directories
- discarded badges
- internal phone lists
- project paperwork

A dead-end finding:

```text
Discarded advertisement
→ public marketing information
→ no additional intelligence
```

A high-value finding:

```text
Discarded document
→ internal project identifier
→ employee names
→ technology reference
→ internal contact information
```

The value comes from **correlation** with earlier OSINT.

## 2.5 Where results feed next

Physical observations can produce:

```text
Location
  ↓
Entrance / workspace
  ↓
Control weakness
  ↓
Asset or information exposed
  ↓
Technical hypothesis
  ↓
Authorized validation
```

For example, a publicly visible technology name can corroborate an earlier technology-stack hypothesis.

A discarded organizational chart can clarify reporting relationships discovered during passive OSINT.

A visible badge policy can explain how physical identity maps to logical identity.

# Section 3 — Core concepts and terminology

| Term               | Meaning                                                                                         |
| ------------------ | ----------------------------------------------------------------------------------------------- |
| Physical Perimeter | The physical boundary protecting an organization's people, systems, and information.            |
| Shoulder Surfing   | Observing information displayed or handled by another person.                                   |
| Tailgating         | Entering a controlled area by following an authorized person without independent authorization. |
| Piggybacking       | Similar to tailgating, but commonly implies the authorized person knowingly permits entry.      |
| Dumpster Diving    | Recovering useful information from discarded material.                                          |
| Access Control     | A mechanism or process determining who may enter or access something.                           |
| Visitor Management | Procedures for identifying, recording, and controlling visitors.                                |
| Security Boundary  | A physical or logical boundary separating protected resources from less-trusted areas.          |
| Clean Desk         | A policy requiring sensitive material to be secured when unattended.                            |
| Screen Lock        | Automatic or manual protection preventing unauthorized screen access.                           |
| Physical OSINT     | Publicly observable information about physical locations and security arrangements.             |
| Pretext            | A fabricated scenario used to justify an interaction.                                           |
| Badge              | Physical credential used to identify or authorize a person.                                     |
| Anti-Passback      | Access-control mechanism designed to prevent credential reuse for multiple entries.             |

| Technique        | Primary exposure                        |
| ---------------- | --------------------------------------- |
| Shoulder surfing | Visual information disclosure           |
| Tailgating       | Physical access-control failure         |
| Dumpster diving  | Information disclosure through disposal |

# Section 4 — Tools and commands

| Tool        | Command                                    | What it finds/shows      | When to use it                  |
| ----------- | ------------------------------------------ | ------------------------ | ------------------------------- |
| `exiftool`  | `exiftool photo.jpg`                       | Image metadata           | Analyze assessment photographs  |
| `strings`   | `strings evidence.bin`                     | Readable embedded text   | Examine recovered lab artifacts |
| `sha256sum` | `sha256sum evidence.bin`                   | Evidence hash            | Preserve evidence integrity     |
| `find`      | `find ./evidence -type f`                  | Collected evidence files | Inventory lab evidence          |
| `grep`      | `grep -RniE 'badge\|door\|screen' ./notes` | Relevant observations    | Search assessment notes         |

Example:

```bash
exiftool perimeter-photo.jpg
```

Possible output:

```text
File Name : perimeter-photo.jpg
Image Size: 1920x1080
Create Date: 2026:08:13 ...
```

Use the metadata to understand the evidence file; do not treat metadata as proof of where the image was physically taken.

Example:

```bash
strings discarded-document.bin
```

Possible output:

```text
Project Aurora
Internal Operations
Facilities
```

Interpretation: potentially useful information has been recovered from the lab artifact.

Example:

```bash
sha256sum discarded-document.bin
```

Possible output:

```text
8b7e... discarded-document.bin
```

Record the hash so later analysis can demonstrate that the evidence was not modified.

Example:

```bash
find ./evidence -type f
```

Possible output:

```text
./evidence/entrance.jpg
./evidence/discarded-paper.txt
./evidence/notes.md
```

This provides a reproducible inventory.

Example:

```bash
grep -RniE 'badge|door|screen' ./notes
```

Possible output:

```text
./notes/observations.md:12:badge reader visible at entrance
./notes/observations.md:19:screen visibility from waiting area
```

Interpretation: quickly locate observations requiring further analysis.

# Section 5 — Defender detection

- **Access-control logs:** repeated badge events, rejected entries, anti-passback violations, and unusual entry patterns.
- **CCTV/video analytics:** following behavior, unauthorized presence, abandoned doors, and activity around disposal areas.
- **Reception records:** unexplained visitors, missing escorts, or inconsistent visitor procedures.
- **DLP/document controls:** sensitive paperwork repeatedly appearing in unsecured disposal.
- **Clean-desk audits:** exposed documents, unlocked workstations, and visible credentials.
- Defenders commonly miss **behavioral weaknesses** because badge readers may be functioning perfectly while employees defeat the control operationally.
- Skilled assessors reduce footprint by observing without touching, photographing only where authorized, avoiding real credential collection, and documenting exposure rather than exploiting it.

# Section 6 — Lab task

**Platform:** Local Kali VM plus a home-office mock perimeter using a spare laptop, printed fictional documents, a locked interior door, and a disposal box.

**Objective:** Identify and document three physical information-security weaknesses without accessing a real organization or collecting real credentials.

**Steps:**

1. Create a mock reception area and mark one doorway as the controlled boundary.
2. Place a laptop displaying fictional non-sensitive project information at a realistic viewing angle.
3. Place fictional discarded documents in the disposal box.
4. Have a second lab participant simulate normal employee movement through the doorway.
5. Observe the setup from the designated visitor position and record visibility and access-control observations.
6. Photograph only the staged environment and record each observation with a timestamp.
7. Classify each finding as shoulder surfing, tailgating exposure, dumpster diving, or no finding.
8. Assign severity based on information sensitivity, repeatability, and control failure.
9. Write remediation for each finding.

**Expected output:**

```text
physical-perimeter-lab/
├── observations.md
├── findings.md
├── remediation.md
└── evidence/
    └── staged-perimeter.jpg
```

Success means three distinct observations are documented with evidence, impact, and remediation.

**Git artifact:**

```bash
git add physical-perimeter-lab/
git commit -m "Add physical perimeter assessment lab"
```

# Section 7 — Common mistakes

1. **Mistake:** Treating visibility as automatically exploitable.
   **Why:** Seeing a screen is not necessarily sensitive exposure.
   **Instead:** Identify exactly what information is visible.

2. **Mistake:** Confusing tailgating with legitimate visitor access.
   **Why:** A controlled process may intentionally permit guests.
   **Instead:** Determine whether authorization and identity verification actually occurred.

3. **Mistake:** Searching discarded material indiscriminately.
   **Why:** It creates unnecessary collection and noise.
   **Instead:** Categorize the disposal weakness and collect only the minimum evidence required.

4. **Mistake:** Ignoring environmental context.
   **Why:** A weakness may exist only at a particular time or location.
   **Instead:** Record location, timing, visibility, and repeatability.

5. **Mistake:** Reporting "bad security" without impact.
   **Why:** Defenders cannot prioritize vague findings.
   **Instead:** Map exposure → information → consequence.

6. **Mistake:** Focusing only on technology.
   **Why:** Physical controls frequently fail through human procedure.
   **Instead:** Assess technical controls and operational behavior together.

# Section 8 — Move-on gate

1. **Shoulder-surfing test:** From a designated visitor position in your lab, identify exactly which staged information is visible and record the exposure without interacting with the workstation.

2. **Access-control test:** Observe ten staged doorway events and correctly classify each as authorized entry, tailgating exposure, piggybacking exposure, or correctly controlled entry without looking at your notes.

3. **Evidence test:** Analyze a staged disposal artifact with the Section 4 tools, produce a hash, extract its relevant information, and write a finding containing evidence, impact, and remediation.
