# LDAP Probing

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

LDAP (Lightweight Directory Access Protocol) is the protocol used to query and modify directory services — most commonly Microsoft Active Directory. Running on TCP port 389 (plaintext) and TCP port 636 (LDAPS — TLS-encrypted), LDAP is the interface through which all domain clients authenticate, look up users and computers, apply group policies, and enumerate organizational structure.

In an Active Directory environment, LDAP is the single most information-rich enumeration target. A successful LDAP bind (authenticated or anonymous) returns: every user account with attributes (username, email, last login, password policy, group memberships), every computer object (hostnames, OS versions, last active), every group and its members, every organizational unit (OU), Group Policy Objects (GPOs), domain password policy, and service principal names (SPNs) — which identify Kerberoastable accounts.

```text
Stage 5 — LDAP Probing
────────────────────────────────────────────────────────────────────
Port Scan finds 389/TCP  →  [LDAP Probing]  →  Complete AD structure
or 636/TCP (LDAPS)            ↑ YOU ARE HERE    all users, groups
or 3268/TCP (GC)              Anonymous bind     computer objects
                              or auth bind       Kerberoast candidates
                              → ldapsearch       password policy
────────────────────────────────────────────────────────────────────
Tools: ldapsearch, ldapdomaindump, python3-ldap, nmap ldap-* scripts
```

---

# Section 2 — How attackers actually use this

## 2.1 Base DN discovery and anonymous bind test

The first step is identifying the LDAP base DN (Distinguished Name) — the root of the directory tree. The base DN follows the format `DC=corp,DC=target,DC=com` for a domain `corp.target.com`. It can be read from the LDAP rootDSE (Root Directory Service Entry) without authentication.

```bash
# Read rootDSE (no auth required — always accessible)
$ ldapsearch -H ldap://203.0.113.45 -x -s base namingContexts

# Result:
# extended LDIF
# LDAPv3
# base <> (default) with scope baseObject
# filter: (objectclass=*)
# requesting: namingContexts

dn:
namingContexts: DC=corp,DC=target,DC=com           ← base DN
namingContexts: CN=Configuration,DC=corp,DC=target,DC=com
namingContexts: CN=Schema,CN=Configuration,DC=corp,DC=target,DC=com
namingContexts: DC=DomainDnsZones,DC=corp,DC=target,DC=com
namingContexts: DC=ForestDnsZones,DC=corp,DC=target,DC=com

# Also from nmap:
$ nmap --script ldap-rootdse -p 389 203.0.113.45
| ldap-rootdse:
|   currentTime: 20240101120000.0Z
|   defaultNamingContext: DC=corp,DC=target,DC=com
|   dnsHostName: DC01.corp.target.com          ← DC hostname!
|   domainFunctionality: 7 (Windows Server 2016+)
|_  ldapServiceName: corp.target.com:dc01$@CORP.TARGET.COM
```

The rootDSE reveals the base DN (`DC=corp,DC=target,DC=com`), the domain controller's FQDN (`DC01.corp.target.com`), and the domain functional level — all without any credentials.

**Anonymous bind test:**
```bash
# Attempt anonymous bind (no username, no password)
$ ldapsearch -H ldap://203.0.113.45 -x \
  -b "DC=corp,DC=target,DC=com" \
  "(objectClass=*)" sAMAccountName 2>&1 | head -20

# Successful anonymous bind (misconfigured — increasingly rare):
# → Returns all objects matching the filter

# Failed anonymous bind (correct default configuration):
# result: 1 Operations error
# additional info: 000004DC: LdapErr: DSID-0C09075A, comment: In order to perform this
# operation a successful bind must be completed on the connection.
```

## 2.2 Authenticated bind — full domain enumeration

An authenticated LDAP bind with any valid domain user account (even the lowest-privilege domain user) returns the complete Active Directory object database. A standard domain user account can read:

```bash
# Authenticated bind
DOMAIN="corp.target.com"
DC_IP="203.0.113.45"
USER="jsmith@corp.target.com"
PASS="password123"
BASE_DN="DC=corp,DC=target,DC=com"

# Enumerate all users
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(objectClass=person)" \
  sAMAccountName userPrincipalName displayName mail pwdLastSet lastLogon \
  description memberOf \
  | tee ldap_users.txt

# Parse to readable list
$ grep "sAMAccountName:" ldap_users.txt | awk '{print $2}' | sort > domain_users.txt
$ wc -l domain_users.txt
847 users

# Sample output:
dn: CN=John Smith,OU=IT,OU=Users,DC=corp,DC=target,DC=com
sAMAccountName: jsmith
userPrincipalName: jsmith@corp.target.com
mail: j.smith@corp-target.com
description: IT Admin - has domain admin access   ← sensitive info in description!
memberOf: CN=Domain Admins,CN=Users,DC=corp,DC=target,DC=com   ← DA!
memberOf: CN=Remote Desktop Users,CN=Builtin,DC=corp,DC=target,DC=com
pwdLastSet: 133200000000000000   ← timestamp (convertible to date)
lastLogon: 133245678900000000
```

A user's `description` field containing "has domain admin access" is a direct OPSEC failure by the administrator — LDAP makes this readable by any domain user.

## 2.3 Enumerating Domain Admins and privileged groups

```bash
# Get all Domain Admin members
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "CN=Domain Admins,CN=Users,$BASE_DN" \
  "(objectClass=*)" member

member: CN=Administrator,CN=Users,DC=corp,DC=target,DC=com
member: CN=John Smith,OU=IT,OU=Users,DC=corp,DC=target,DC=com   ← jsmith is DA!
member: CN=svc_backup,CN=ServiceAccounts,OU=Users,DC=corp,DC=target,DC=com  ← service account as DA!

# Get all groups (identify high-privilege groups)
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(objectClass=group)" cn member description \
  | grep -E "^cn:|^description:|^member:" | head -60

cn: Domain Admins
cn: Enterprise Admins
cn: Schema Admins
cn: Account Operators     ← can create/modify accounts
cn: Backup Operators      ← can access all files for backup
cn: Remote Desktop Users
cn: VPN Users
cn: IT_Admins
cn: SvcAccounts           ← service accounts group — often high privilege

# Enumerate all service accounts (SPNs — Kerberoastable)
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(&(objectClass=user)(servicePrincipalName=*))" \
  sAMAccountName servicePrincipalName

sAMAccountName: svc_mssql
servicePrincipalName: MSSQLSvc/db01.corp.target.com:1433   ← MSSQL service account
sAMAccountName: svc_http
servicePrincipalName: HTTP/web01.corp.target.com:80        ← web service account
sAMAccountName: svc_backup
servicePrincipalName: backup/backup01.corp.target.com      ← backup service account
```

Every account with a `servicePrincipalName` is Kerberoastable — any domain user can request a TGS ticket for the service, and the ticket is encrypted with the service account's password hash. Offline cracking of the ticket recovers the plaintext password.

## 2.4 Computer object enumeration

```bash
# Enumerate all domain-joined computers
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(objectClass=computer)" \
  cn dNSHostName operatingSystem operatingSystemVersion lastLogonTimestamp \
  | tee ldap_computers.txt

# Results:
cn: DC01
dNSHostName: DC01.corp.target.com
operatingSystem: Windows Server 2019 Standard
operatingSystemVersion: 10.0 (17763)
lastLogonTimestamp: 133245678900000000   ← recently active

cn: FS01
dNSHostName: FS01.corp.target.com
operatingSystem: Windows Server 2012 R2 Standard   ← EOL OS!
operatingSystemVersion: 6.3 (9600)

cn: WEB01
dNSHostName: WEB01.corp.target.com
operatingSystem: Windows Server 2019 Standard

cn: DEVLAPTOP01
dNSHostName: DEVLAPTOP01.corp.target.com
operatingSystem: Windows 10 Pro
# → Developer workstation — likely has SSH keys, git credentials

# Summary
$ grep "operatingSystem:" ldap_computers.txt | sort | uniq -c | sort -rn
  12 operatingSystem: Windows Server 2019 Standard
   3 operatingSystem: Windows Server 2012 R2 Standard   ← EOL = unpatched
   5 operatingSystem: Windows 10 Pro
   2 operatingSystem: Windows 11 Pro
```

`Windows Server 2012 R2 Standard` — EOL since October 2023, no longer receiving security patches. Any CVE disclosed after EOL is permanently unpatched on these 3 hosts.

## 2.5 Password policy extraction

```bash
# Domain password policy
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(objectClass=domain)" \
  pwdHistoryLength minPwdLength maxPwdAge lockoutThreshold lockoutDuration

maxPwdAge: -36288000000000   (30 days in 100ns intervals)
minPwdLength: 8              ← minimum 8 characters
pwdHistoryLength: 10         ← last 10 passwords remembered
lockoutThreshold: 5          ← 5 failed attempts = lockout
lockoutDuration: -18000000000 (30 minutes)

# Password spraying safe threshold:
# lockoutThreshold = 5 → maximum 4 attempts per account before lockout
# Stay below threshold: 1 password per account per 30-minute lockout window
```

The password policy directly governs your password spraying strategy. 5 attempts before lockout with 30-minute duration means you can spray 4 passwords (leaving 1 attempt buffer) and then wait 30 minutes before the next round.

## 2.6 ldapdomaindump for automated AD enumeration

`ldapdomaindump` automates all the individual ldapsearch queries and outputs structured JSON and HTML reports covering all AD objects:

```bash
$ pip install ldapdomaindump

# Run against DC with credentials
$ ldapdomaindump -u "corp.target.com\jsmith" -p "password123" ldap://203.0.113.45

# Output files:
domain_users.json         ← all users with full attributes
domain_computers.json     ← all computers
domain_groups.json        ← all groups with members
domain_trusts.json        ← domain trusts (forest enumeration)
domain_policy.json        ← password and lockout policy
domain_users_by_group.json ← users organized by group membership

# HTML report (open in browser)
domain_users.html         ← searchable, sortable user table

# Parse JSON for specific data
$ cat domain_users.json | python3 -c "
import json, sys
for user in json.load(sys.stdin):
    if 'Domain Admins' in str(user.get('memberOf', [])):
        print(user['sAMAccountName'], '|', user.get('mail', 'no email'))
"
```

## 2.7 AS-REP Roasting candidates (no pre-auth accounts)

Accounts with the `DONT_REQUIRE_PREAUTH` flag set do not require Kerberos pre-authentication. Any unauthenticated user can request an AS-REP for these accounts, and the response contains a portion encrypted with the user's password hash — offline-crackable.

```bash
# Find accounts with pre-auth disabled (AS-REP Roasting candidates)
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" \
  sAMAccountName

# userAccountControl bitmask 4194304 = 0x400000 = DONT_REQUIRE_PREAUTH

sAMAccountName: legacy_svc_account   ← AS-REP Roastable!
sAMAccountName: temp_testuser        ← AS-REP Roastable!

# userAccountControl flags reference:
# 0x0002    ACCOUNTDISABLE
# 0x0010    LOCKOUT
# 0x0020    PASSWD_NOTREQD          ← no password required
# 0x10000   DONT_EXPIRE_PASSWORD    ← password never expires
# 0x400000  DONT_REQUIRE_PREAUTH    ← AS-REP Roastable
# 0x800000  PASSWORD_EXPIRED
```

## 2.8 Global Catalog (port 3268) for forest-wide enumeration

The Global Catalog runs on port 3268 (or 3269 for LDAPS) and contains a subset of attributes for all objects in the entire AD forest — including objects from other domains in the forest. This is valuable when the target has multiple domains (parent + child, or multi-domain forest):

```bash
# Query Global Catalog for all users across entire forest
$ ldapsearch -H ldap://203.0.113.45:3268 -x -D "$USER" -w "$PASS" \
  -b "DC=corp,DC=target,DC=com" \
  "(objectClass=person)" sAMAccountName | grep sAMAccountName | wc -l
1284   ← more users than querying the single domain (child domain users included)

# Check domain trusts
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "CN=System,$BASE_DN" \
  "(objectClass=trustedDomain)" trustPartner trustType trustAttributes

cn: partner.target.com
trustPartner: PARTNER.TARGET.COM    ← external domain trust
trustType: 2                         ← Windows domain
trustAttributes: 32                  ← forest trust

# Forest trust = bidirectional access may be possible
# → Users in corp.target.com may have access to partner.target.com resources
```

## 2.9 BloodHound-compatible LDAP collection and attack path analysis

BloodHound maps Active Directory attack paths by collecting LDAP data and SMB session data, then graphing relationships between users, groups, computers, and permissions to find paths to Domain Admin. The collection (SharpHound/BloodHound.py) works primarily over LDAP:

```bash
# Python BloodHound collector (bloodhound-python) — runs from Kali with domain creds
$ pip install bloodhound

$ bloodhound-python -d corp.target.com -u jsmith -p password123 \
  -ns 10.10.0.10 -c All --zip

# Collection options:
# -c All:         Collect everything (sessions, ACLs, trusts, groups, computers)
# -c DCOnly:      Domain Controller only — quieter, less network traffic
# -c Session:     Active sessions only (who is logged into which computer)
# -c ACL:         Object ACLs only (which users have permissions on which objects)

# Output files:
# <timestamp>_computers.json
# <timestamp>_users.json
# <timestamp>_groups.json
# <timestamp>_domains.json
# <timestamp>_ous.json
# <timestamp>_gpos.json
# → Import into BloodHound GUI → graph attack paths to Domain Admin

# Key BloodHound queries (Cypher — run in BloodHound GUI):
# Find shortest path to Domain Admins:
#   MATCH p=shortestPath((n:User {name:"JSMITH@CORP.TARGET.COM"})-[*1..]->(g:Group {name:"DOMAIN ADMINS@CORP.TARGET.COM"})) RETURN p

# Find all users with DCSync rights:
#   MATCH (n)-[r:DCSync|AllExtendedRights|GenericAll]->(d:Domain) RETURN n, r, d

# Find computers where Domain Admin sessions are active:
#   MATCH (c:Computer)-[:HasSession]->(u:User)-[:MemberOf*1..]->(g:Group {name:"DOMAIN ADMINS@..."}) RETURN c

# Without BloodHound GUI — parse JSON to extract key findings:
$ python3 - << 'EOF'
import json, glob

for fname in glob.glob('*_users.json'):
    data = json.load(open(fname))
    for user in data.get('data', []):
        name = user.get('Properties', {}).get('name', '')
        # Find users with admincount=1 (in a high-privilege group at some point)
        if user.get('Properties', {}).get('admincount'):
            print(f"[AdminCount=1] {name}")
        # Find users with password never expires
        if user.get('Properties', {}).get('pwdneverexpires'):
            print(f"[PwdNeverExpires] {name}")
EOF

# From LDAP without BloodHound — equivalent manual queries:

# Find all users with AdminCount = 1 (ever in a privileged group)
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(&(objectClass=user)(adminCount=1))" \
  sAMAccountName memberOf \
  | grep "sAMAccountName:" | awk '{print $2}'

Administrator
jsmith          ← has adminCount=1 — was in privileged group
svc_backup      ← service account with admin history — high value target

# Find accounts with GenericAll on Domain object (DCSync candidates)
$ ldapsearch -H ldap://$DC_IP -x -D "$USER" -w "$PASS" \
  -b "$BASE_DN" \
  "(objectClass=domain)" nTSecurityDescriptor
# Parse nTSecurityDescriptor for GenericAll/DCSync rights — complex binary parsing
# → Use BloodHound for this — it handles DACL parsing automatically
```

BloodHound's "Find Shortest Paths to Domain Admins" query is the single most impactful query in Active Directory recon — it converts raw LDAP data into a visual attack path that shows exactly which accounts and permissions are needed to escalate from `jsmith@corp.target.com` to Domain Admin. This turns hours of manual LDAP analysis into a 5-second graph traversal.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **LDAP** | Lightweight Directory Access Protocol — queries and modifies directory services on TCP 389/636 |
| **Active Directory (AD)** | Microsoft's directory service for Windows networks; uses LDAP as its query protocol |
| **Base DN** | Distinguished Name forming the root of the LDAP search scope (e.g., `DC=corp,DC=target,DC=com`) |
| **Anonymous bind** | LDAP connection without credentials; returns limited data in properly configured systems |
| **Authenticated bind** | LDAP connection with valid domain credentials; returns full AD object data |
| **sAMAccountName** | Windows login name attribute (e.g., `jsmith`) |
| **userPrincipalName** | UPN login format (e.g., `jsmith@corp.target.com`) |
| **Distinguished Name (DN)** | Full LDAP path to an object (e.g., `CN=jsmith,OU=IT,DC=corp,DC=target,DC=com`) |
| **SPN (Service Principal Name)** | Attribute on service accounts identifying the service — Kerberoastable |
| **Kerberoasting** | Requesting TGS tickets for SPN accounts and cracking the hash offline |
| **userAccountControl** | Bitmask attribute controlling account properties (disabled, locked, pre-auth required, etc.) |
| **AS-REP Roasting** | Retrieving encrypted AS-REP for accounts with DONT_REQUIRE_PREAUTH; offline-crackable |
| **Global Catalog** | AD service on port 3268 containing partial attributes for all objects in the entire forest |
| **ldapdomaindump** | Tool automating AD enumeration via LDAP; produces JSON and HTML reports |
| **Domain trust** | Relationship between two domains allowing resource access across boundaries |
| **Password policy** | AD-enforced rules for minimum length, lockout threshold, history — governs spraying strategy |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `ldapsearch` | `ldapsearch -H ldap://DC -x -s base namingContexts` | Base DN (no auth) | First query |
| `ldapsearch` | `ldapsearch -H ldap://DC -x -D user@domain -w pass -b base "(objectClass=person)"` | All users | Auth available |
| `ldapdomaindump` | `ldapdomaindump -u "domain\user" -p pass ldap://DC` | Full AD dump | Auth available |
| `nmap ldap-*` | `nmap --script ldap-rootdse,ldap-search -p 389 DC` | rootDSE + basic enum | No auth |

---

# Section 5 — Defender detection

- **Anonymous bind attempts:** Modern Active Directory denies anonymous binds by default. An anonymous LDAP bind attempt generates an event (Event ID 4776 — failed credential validation or LDAP authentication failure) in Windows Security logs. The rootDSE read (namingContexts) is genuinely unauthenticated and is not logged as a security event — it is designed to be public.
- **Authenticated LDAP queries:** Successful LDAP binds are logged (Event ID 4624 — logon) and LDAP queries can be logged with LDAP search auditing enabled. By default, LDAP query auditing is not active in standard Windows Server configurations. However, SIEM rules can detect unusually broad LDAP queries (e.g., requesting all attributes of all users in a single query).
- **ldapdomaindump volume:** ldapdomaindump generates a relatively high volume of LDAP queries to pull all objects. On a monitored domain with LDAP query auditing, this is visible as hundreds of queries from a single source within seconds.
- **Mitigation for operators:** (1) Use targeted ldapsearch queries for specific OUs rather than a full base-level query. (2) Space queries over time. (3) Use legitimate user account credentials and query patterns that blend with normal domain operation — domain users read LDAP constantly for normal functions.

---

# Section 6 — Lab task

**Platform:** Kali Linux + TryHackMe "Active Directory Basics" or "Attacktive Directory" room, or a local Windows Server 2019 AD lab.

**Objective:** Enumerate a target AD domain via LDAP — discover all users, Domain Admins, Kerberoastable accounts, and the password policy.

**Steps:**

1. **rootDSE read:** `ldapsearch -H ldap://<DC_IP> -x -s base namingContexts` — identify base DN
2. **Anonymous bind test:** `ldapsearch -H ldap://<DC_IP> -x -b "DC=corp,DC=target,DC=com" "(objectClass=*)" cn | head -20` — document result
3. **User enumeration (auth):** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "DC=corp,DC=target,DC=com" "(objectClass=person)" sAMAccountName | grep sAMAccountName | wc -l`
4. **Domain Admin group members:** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "CN=Domain Admins,CN=Users,DC=corp,DC=target,DC=com" "(objectClass=*)" member`
5. **Kerberoastable accounts (SPN):** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "DC=..." "(&(objectClass=user)(servicePrincipalName=*))" sAMAccountName servicePrincipalName`
6. **AS-REP Roastable accounts:** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "DC=..." "(&(objectCategory=person)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName`
7. **Password policy:** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "DC=..." "(objectClass=domain)" lockoutThreshold minPwdLength maxPwdAge`
8. **Computer objects:** `ldapsearch -H ldap://<DC_IP> -x -D "user@domain" -w "pass" -b "DC=..." "(objectClass=computer)" cn operatingSystem | grep -E "^cn:|^operatingSystem:"`
9. **ldapdomaindump full run:** `ldapdomaindump -u "domain\user" -p "pass" ldap://<DC_IP>` — review all output files
10. **Compile `ldap_intel.md`:** Total users | Domain Admins (list) | Kerberoast candidates (list) | AS-REP targets | Password policy | EOL OS count | Domain trusts

```bash
git commit -m "recon(stage5): LDAP probing — AD structure enumerated for <target>"
```

---

# Section 7 — Common mistakes

**1. Treating anonymous bind failure as "LDAP has nothing"**
_Why it matters:_ Anonymous bind is denied by default on modern AD. A failed anonymous bind is not a failed LDAP probe — it just means you need credentials. The rootDSE read (which IS available anonymously) has already given you the base DN and DC hostname.
_Fix:_ Always read rootDSE first regardless of anonymous bind status. Then use any valid domain credential for the authenticated bind.

**2. Only querying the users OU and missing service accounts**
_Why it matters:_ Service accounts with SPNs (Kerberoastable) are often in separate OUs (`ServiceAccounts`, `ManagedServiceAccounts`). A query scoped to `OU=Users` misses them.
_Fix:_ Always search from the root base DN (`DC=corp,DC=target,DC=com`) with base scope. Never scope to a specific OU for initial enumeration.

**3. Not reading the `description` attribute**
_Why it matters:_ Active Directory administrators frequently put sensitive information in the `description` field of user accounts and groups — "Password: Welcome1", "Admin account for legacy system", "Has DA access". These are readable by any domain user.
_Fix:_ Always request the `description` attribute in user and group queries. Grep the results for: `password`, `admin`, `cred`, `access`, `key`.

**4. Forgetting to check AS-REP Roastable accounts**
_Why it matters:_ AS-REP Roasting doesn't require any domain credentials — it only requires network access to the DC on port 88. Accounts with `DONT_REQUIRE_PREAUTH` are crackable without even doing an authenticated bind. The LDAP query to find them requires credentials but the attack itself does not.
_Fix:_ Always query for `userAccountControl` bitmask `4194304`. Even one AS-REP Roastable account may provide a credential without any prior foothold.

**5. Not checking the Global Catalog for forest-wide enumeration**
_Why it matters:_ Querying the DC on port 389 only returns objects from the local domain. If the AD forest has multiple domains, users and computers from child/sibling domains are not returned. The Global Catalog (port 3268) contains partial data for all forest objects.
_Fix:_ After standard LDAP enumeration, always run a Global Catalog query: `ldapsearch -H ldap://DC:3268 ...` — compare the user count to the standard port 389 result.

---

# Section 8 — Move-on gate

1. `ldapsearch` returns a user object with attributes: `sAMAccountName: svc_backup`, `servicePrincipalName: backup/backup01.corp.target.com`, `memberOf: CN=Domain Admins`. Without notes, state: what attack this account enables, what tool you use to execute it, what credential you get, and why the `memberOf` attribute makes this a critical finding.

2. The password policy from LDAP shows `lockoutThreshold: 3` and `lockoutDuration: -18000000000` (30 minutes). You have a list of 500 domain users from LDAP. Without notes, describe your password spraying strategy: how many passwords per window, how long to wait between windows, and what the maximum number of passwords you can try in 24 hours is.

3. A `ldapsearch` for `(objectClass=person)` returns 200 results. A Global Catalog query on port 3268 returns 450 results. Without notes, explain what the discrepancy means about the AD forest structure and what your next investigative step is.
