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

Physical observations feed the technical phases directly. Every physical OSINT finding maps to a technical hypothesis:

```text
Physical finding                  → Technical follow-on
────────────────────────────────────────────────────────
Visible technology brand (Cisco)  → Version-specific CVE search in Phase 3
Badge reader model (HID, Lenel)   → Physical access control platform identification
Discarded org chart               → Hierarchy + target names for social engineering
WiFi SSID visible on device       → Wireless network for Phase 3 active recon
Printed IP range on whiteboard    → Internal network segmentation hypothesis
Delivery personnel routine        → Timing window for physical access attempt
```

A discarded organizational chart clarifies reporting relationships and identifies who has administrator-level access. A visible badge policy explains how physical identity maps to logical identity (username format, domain structure). A publicly readable technology label corroborates an earlier technology-stack hypothesis from job listings or OSINT.

The output of physical perimeter assessment is a list of observations with corroborating evidence, not a list of "things I saw." Each observation must connect to a technical or social engineering implication to be reportable.

**Correlating physical finds with prior OSINT:** The real power of physical perimeter assessment is not standalone discovery — it is corroboration. A document recovered from a dumpster that mentions an internal hostname (`FILESERVER01`) is far more valuable when combined with a PTR record from Stage 2 that resolves to the same name. A visible network equipment brand (Juniper on the building's comms room door label) confirms what a Shodan query returned for the organization's IP space. Physical intelligence validates and enriches digital intelligence, and vice versa.

```text
Walkthrough sequence:
1. OSINT hypothesis:  "They likely use VMware based on job listings for vSphere admins."
2. Physical confirm:  Server room vent visible from parking — VMware logo on visible equipment.
3. Digital confirm:   Shodan shows ESXi management port 443 on 203.0.113.8, confirmed vendor.
4. Combined finding:  VMware ESXi version X exposed on management interface at specific IP.
```

## 2.6 Satellite and aerial imagery OSINT

Satellite imagery from Google Earth, Bing Maps, Google Maps, and Apple Maps provides overhead views of target facilities that are public, high-resolution, and timestamped. This is fully passive — requesting satellite imagery generates no event on the target's side.

**What overhead imagery reveals:**

- **Roof infrastructure:** HVAC units, satellite dishes, antennas, generator exhaust ports, and raised floor ventilation grills identify critical infrastructure locations. Server rooms require significant HVAC — large rooftop HVAC clusters above a specific section of a building indicate a data center or server room below.
- **Parking lot behavior:** Executive vehicle patterns, shift-change timing (how many cars arrive/depart at specific hours), delivery bay activity, and security vehicle presence.
- **Physical perimeter:** Fencing type and coverage, security cameras mounted on exterior walls, guard post locations, loading docks, emergency exits, and secondary entrances not visible from street level.
- **Adjacent facilities:** What organizations share the building, parking, or access roads. Adjacent businesses may share physical access points that reduce security assumptions.
- **Historical imagery (Google Earth Pro / Wayback):** Google Earth Pro allows time-slider navigation through historical satellite images. A facility that has added fencing, cameras, or guard infrastructure in recent years was likely reacting to a previous incident. Construction visible in old imagery may reveal new building wings, underground cable runs, or access modifications.

```text
Google Earth time-slider workflow:
1. Open Google Earth Pro (free)
2. Navigate to target facility address
3. Click the clock icon → enable historical imagery
4. Scroll timeline back 5-10 years
5. Document: fence changes, HVAC additions, new structures, parking patterns
```

**Geospatial OSINT tools:**
- **Google Earth Pro** (free, Windows/Mac/Linux) — historical imagery, measurement tools, GPS coordinate export
- **Bing Maps Bird's Eye** — oblique 45-degree aerial photography that provides a near-3D view of building facades and roofs not visible from straight overhead
- **Sentinel Hub EO Browser** (`apps.sentinel-hub.com`) — free ESA satellite imagery, updated roughly every 5 days for cloudy/clear passes; useful for large industrial or campus facilities
- **SkyFi / Planet Labs** (commercial) — tasked satellite imagery on demand; for high-priority targets in critical infrastructure sectors

## 2.7 Social media as physical intelligence

Employees and visitors post photographs, videos, and check-ins that inadvertently reveal physical security details. This is passive OSINT that requires no physical presence at the site.

**LinkedIn background photographs:** Job posting photos, office tour videos on company culture pages, and "day in the life" posts routinely show:
- Security badge designs and lanyard colors
- Badge reader models and locations
- Server room and NOC photographs (sometimes accidentally)
- Desk layouts revealing dual-monitor setups, laptop docking stations, and classified label positioning
- Whiteboard photographs with internal architecture diagrams

**Instagram and X (Twitter) geotagged posts:** Searching a facility's geolocation on social platforms (Instagram location tags, Twitter geosearch) finds employee and visitor posts from inside the building. Photographs taken indoors often contain:
- Interior floor layout visible in background
- Network equipment cabinets
- Interior access card readers and door configurations
- Meeting room naming conventions (which may match internal system hostnames)

**Job listing analysis for physical security details:** Job listings for physical security roles describe the technology stack: "Lenel OnGuard experience required" identifies the physical access control platform, "CCTV experience with Milestone XProtect" identifies the VMS (Video Management System). These are the same systems an insider or social engineer would need to understand.

```text
LinkedIn search → Company → Posts → Filter by media
Instagram → Location search → Facility address
Indeed/LinkedIn → Jobs → Security → "Physical Security Analyst" at target
                                 → Description reveals: badge system, CCTV brand, alarm vendor
```

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
