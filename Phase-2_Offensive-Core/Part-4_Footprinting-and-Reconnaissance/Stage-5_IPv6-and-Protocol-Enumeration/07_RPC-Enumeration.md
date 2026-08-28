# RPC Enumeration

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

RPC (Remote Procedure Call) is the mechanism Windows uses for inter-process communication across the network. The RPC Endpoint Mapper on TCP/UDP 135 acts as a directory service: clients query it to find which dynamic high-numbered port a specific RPC service is listening on. SMB (TCP 445) hosts named pipes that many RPC services use as an alternative transport.

In Windows environments, RPC is the backbone of administrative protocols: user and group enumeration (SAMR), privilege operations (LSARPC), service management (SVCCTL), task scheduling (ATSVC), and registry access (WINREG). Many of these services permit null session connections — a connection with no username and no password — allowing enumeration without credentials when the server is misconfigured or running an older Windows version.

```text
Stage 5 — RPC Enumeration
────────────────────────────────────────────────────────────────────
Port Scan finds 135/TCP  →  [RPC Enumeration]  →  Domain users/groups
or 445/TCP (named pipes)      ↑ YOU ARE HERE       SID enumeration
                              rpcclient             shared resources
                              nmap msrpc-*          server info
                              impacket              service account IDs
────────────────────────────────────────────────────────────────────
Tools: rpcclient, nmap NSE, impacket (lookupsid, samrdump), enum4linux-ng
```

---

# Section 2 — How attackers actually use this

## 2.1 Endpoint mapper enumeration (port 135)

The RPC Endpoint Mapper on TCP 135 maintains a registry of all RPC services currently listening, their protocols, and their dynamic port numbers. Querying it reveals which RPC services are active on the target.

```bash
# nmap RPC endpoint scan
$ nmap --script msrpc-enum -p 135 203.0.113.45

PORT    STATE SERVICE
135/tcp open  msrpc
| msrpc-enum:
|   100003/3: nfs on 203.0.113.45:2049       ← NFS is running!
|   100005/1: mountd on 203.0.113.45:892
|   100000/2: portmapper on 203.0.113.45:111
|   MS-RPC:
|     EP Mapper
|     LSARPC: Local Security Authority (policy, SIDs, trusts)
|     SAMR:   Security Account Manager (users, groups, passwords)
|     SVCCTL: Service Control Manager
|     WINREG: Remote Registry
|_    ATSVC:  Task Scheduler

# rpcinfo for Unix RPC (portmapper on port 111)
$ rpcinfo -p 203.0.113.45
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100000    4   udp    111  portmapper
    100003    4   tcp   2049  nfs
    100005    3   tcp    892  mountd
    100024    1   udp    892  status

# impacket endpoint mapper
$ python3 /usr/share/doc/python3-impacket/examples/rpcdump.py 203.0.113.45
```

## 2.2 Null session enumeration with rpcclient

A null session is an SMB/RPC connection established with an empty username and empty password. On older Windows systems (NT/2000/XP/Server 2003) null sessions were enabled by default and permitted full user enumeration. On modern Windows (Server 2008+) null sessions are restricted by default but are still enabled in some configurations — particularly in legacy enterprise environments.

```bash
# Connect with null session (empty username and password)
$ rpcclient -U "" -N 203.0.113.45

# If successful, prompt appears:
rpcclient $>

# Server information
rpcclient $> srvinfo
        CORP-WEB01     Wk Sv PrQ Unx NT SNT
        platform_id     : 500
        os version      : 10.0
        server type     : 0x9003
        shares comment  :

# Enumerate domain users
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[jsmith] rid:[0x44f]
user:[mwilson] rid:[0x450]
user:[svc_mssql] rid:[0x451]
user:[svc_backup] rid:[0x452]
user:[svc_http] rid:[0x453]

# Enumerate domain groups
rpcclient $> enumdomgroups
group:[Domain Admins] rid:[0x200]
group:[Domain Users] rid:[0x201]
group:[Domain Guests] rid:[0x202]
group:[IT_Admins] rid:[0x40f]

# Get members of Domain Admins group (rid=0x200 = 512 decimal)
rpcclient $> querygroupmem 0x200
        rid:[0x1f4] attr:[0x7]    ← Administrator (rid 500)
        rid:[0x44f] attr:[0x7]    ← jsmith

# Query specific user details
rpcclient $> queryuser 0x44f
        User Name   :   jsmith
        Full Name   :   John Smith
        Home Drive  :
        Dir Drive   :
        Profile Path:
        Logon Script:
        Description :   IT Admin - can access all servers
        Workstations:
        Comment     :
        Logon Time               : Fri, 28 Aug 2026 10:30:00 IST
        Logoff Time              : Wed, 31 Dec 1969 23:00:00 IST
        Kickoff Time             : Wed, 13 Sep 30828 11:48:05 IST
        Password last set Time   : Mon, 01 Jan 2024 09:00:00 IST
        Password can change Time : Mon, 01 Jan 2024 09:00:00 IST
        Password must change Time: Wed, 13 Sep 30828 11:48:05 IST
        unknown_2[0]             : 0x00000000

# Enumerate shares
rpcclient $> netshareenum
netname: ADMIN$
        remark: Remote Admin
netname: C$
        remark: Default share
netname: IPC$
        remark: Remote IPC
netname: SharedDocs
        remark: Shared Documents
        path:   C:\SharedDocs
```

## 2.3 SID enumeration and user brute-force (lookupsid)

Windows Security Identifiers (SIDs) are the unique identifiers for domain objects. The domain SID is fixed; user/group RIDs (Relative Identifiers) follow predictable patterns. By enumerating RIDs sequentially, you can discover all domain users and groups — even without `enumdomusers` access.

```bash
# Get the domain SID
rpcclient $> lsaquery
Domain Name: CORP
Domain Sid: S-1-5-21-1234567890-9876543210-1122334455

# Enumerate by RID (SID + RID brute-force)
rpcclient $> lookupsids S-1-5-21-1234567890-9876543210-1122334455-500
S-1-5-21-1234567890-9876543210-1122334455-500 CORP\Administrator (1)

rpcclient $> lookupsids S-1-5-21-1234567890-9876543210-1122334455-1000
S-1-5-21-1234567890-9876543210-1122334455-1000 CORP\jsmith (1)

# Automated RID cycling with impacket lookupsid.py
$ python3 /usr/share/doc/python3-impacket/examples/lookupsid.py \
  corp.target.com/jsmith:password123@203.0.113.45

# With null session (no credentials):
$ python3 /usr/share/doc/python3-impacket/examples/lookupsid.py \
  anonymous:@203.0.113.45

# Output:
[*] Domain SID is: S-1-5-21-1234567890-9876543210-1122334455
500: CORP\Administrator (SidTypeUser)
501: CORP\Guest (SidTypeUser)
512: CORP\Domain Admins (SidTypeGroup)
513: CORP\Domain Users (SidTypeGroup)
...
1103: CORP\jsmith (SidTypeUser)
1104: CORP\mwilson (SidTypeUser)
1105: CORP\svc_mssql (SidTypeUser)

# Known RIDs:
# 500 = Administrator (built-in, always exists — rename doesn't change RID)
# 501 = Guest
# 512 = Domain Admins
# 513 = Domain Users
# 514 = Domain Guests
# 516 = Domain Controllers
# 519 = Enterprise Admins
# 520 = Group Policy Creator Owners
```

## 2.4 Named pipe enumeration over SMB

Many RPC services are accessible via named pipes over SMB port 445. Named pipes are identified by their pipe name (`\pipe\samr`, `\pipe\lsarpc`, etc.) and serve as transport endpoints for specific RPC interfaces.

```bash
# List available named pipes (nmap script)
$ nmap --script smb-enum-shares,smb-enum-users,smb-security-mode -p 445 203.0.113.45

| smb-security-mode:
|   account_used: <blank>          ← null session accepted!
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled      ← SMB signing disabled (MITM-able!)

| smb-enum-users:
|   CORP\Administrator (RID: 500)  ← null session user enum works
|   CORP\jsmith (RID: 1103)
|   CORP\svc_mssql (RID: 1105)

# Known named pipes and their RPC interfaces:
# \pipe\samr      → SAMR  (users, groups, passwords)
# \pipe\lsarpc    → LSARPC (policy, SIDs, trusts)
# \pipe\svcctl    → SVCCTL (service control — start/stop services)
# \pipe\atsvc     → ATSVC  (task scheduler — create scheduled tasks)
# \pipe\winreg    → WINREG (remote registry access)
# \pipe\spoolss   → Print spooler (PrintNightmare CVE-2021-34527 uses this)
# \pipe\netlogon  → Netlogon (domain authentication)
# \pipe\srvsvc    → SRVSVC (server service — share enumeration)

# Access specific RPC via pipe:
rpcclient $> pipe lsarpc
rpcclient $> lsaenumsid        # enumerate SIDs
rpcclient $> lsalookupnames "Administrator"   # resolve name to SID
```

## 2.5 enum4linux-ng — automated RPC/SMB enumeration

`enum4linux-ng` is the modern successor to `enum4linux`, combining rpcclient, nmblookup, smbclient, and ldapsearch into a single automated enumeration tool.

```bash
# Install
$ pip install enum4linux-ng
# or: git clone https://github.com/cddmp/enum4linux-ng

# Full enumeration (attempt null session and authenticated)
$ enum4linux-ng -A 203.0.113.45 2>/dev/null | tee enum4linux_results.txt

# With credentials
$ enum4linux-ng -A -u "jsmith" -p "password123" 203.0.113.45 | tee enum4linux_auth.txt

# Output sections:
# [*] SMB dialect check
# [*] RPC session check  
# [*] Domain/workgroup info
# [*] OS/build info
# [*] Users via RPC
# [*] Groups via RPC
# [*] Shares via RPC
# [*] Password policy via RPC
# [*] Users via LDAP (if available)

# JSON output for scripting
$ enum4linux-ng -A -oJ results.json 203.0.113.45
$ cat results.json | python3 -c "
import json, sys
data = json.load(sys.stdin)
print('Users:', len(data.get('users', {})))
print('Groups:', len(data.get('groups', {})))
print('Shares:', len(data.get('shares', {})))
"
```

## 2.6 SAMR password policy extraction

The SAMR (Security Account Manager Remote) protocol exposes the domain password policy over RPC without requiring Domain Admin privileges:

```bash
# Password policy via rpcclient
rpcclient $> getdompwinfo
min_password_length: 8
password_properties: 0x00000001
        DOMAIN_PASSWORD_COMPLEX

rpcclient $> getusrdompwinfo 0x44f
    &info: struct samr_PwInfo
        min_password_length      : 8
        password_properties      : 0x00000001

# Password policy via nmap
$ nmap --script smb-security-mode -p 445 203.0.113.45

# impacket samrdump (authenticated)
$ python3 /usr/share/doc/python3-impacket/examples/samrdump.py \
  corp.target.com/jsmith:password123@203.0.113.45

# Output includes:
# Domain password info:
#   Minimum password length: 8
#   Password history length: 10
#   Maximum password age: 30 days
#   Minimum password age: 1 day
#   Lockout threshold: 5
#   Lockout observation window: 30 mins
#   Lockout duration: 30 mins
```

## 2.7 SVCCTL — remote service control interrogation

The SVCCTL (Service Control Manager) RPC interface allows listing and managing Windows services remotely. With sufficient privileges, it can start/stop/create services — a common lateral movement vector. For recon, it reveals running services:

```bash
# List services via rpcclient (requires appropriate privileges)
rpcclient $> enumservices
svc:      AmazonSSMAgent     title: Amazon SSM Agent
svc:      CSFalconService    title: CrowdStrike Falcon Sensor Service   ← EDR confirmed!
svc:      MpsSvc             title: Windows Defender Firewall
svc:      Schedule           title: Task Scheduler
svc:      Sense              title: Windows Defender Advanced Threat Protection ← Defender ATP!
svc:      MSSQLSERVER        title: SQL Server (MSSQLSERVER)
svc:      W3SVC              title: World Wide Web Publishing Service    ← IIS running

# impacket services.py
$ python3 /usr/share/doc/python3-impacket/examples/services.py \
  corp.target.com/jsmith:password123@203.0.113.45 list
```

`CSFalconService` (CrowdStrike) and `Sense` (Defender ATP) in the service list feeds directly into Stage 4 defensive profiling (note 02). RPC-based service enumeration achieves the same result as post-access `tasklist`, but remotely.

## 2.8 RPC error interpretation

Not all RPC errors mean enumeration failed — error codes reveal the security configuration:

```text
rpcclient error codes and meanings:
  NT_STATUS_ACCESS_DENIED         → null session blocked, need credentials
  NT_STATUS_LOGON_FAILURE         → wrong credentials
  NT_STATUS_ACCOUNT_LOCKED_OUT    → account locked — stop spraying immediately!
  NT_STATUS_PASSWORD_EXPIRED      → password expired but account valid
  NT_STATUS_CONNECTION_REFUSED    → port closed or firewall blocking
  NT_STATUS_NOT_SUPPORTED         → this RPC function not available
  WERR_ACCESS_DENIED              → operation requires higher privileges
  STATUS_PIPE_DISCONNECTED        → named pipe timeout — retry
```

`NT_STATUS_ACCOUNT_LOCKED_OUT` is the most critical error to monitor during password spraying. If you receive this, the target account has been locked. Stop immediately — further attempts extend the lockout and alert the security team.

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **RPC (Remote Procedure Call)** | Protocol allowing programs to call functions on remote systems as if they were local |
| **Endpoint Mapper** | RPC service on TCP 135 that maps RPC interfaces to their current listening ports |
| **Null session** | SMB/RPC connection with no username and no password; restricted by default on modern Windows |
| **SAMR** | Security Account Manager Remote — RPC interface for user, group, and password management |
| **LSARPC** | Local Security Authority RPC — exposes security policy, SIDs, trust relationships |
| **SVCCTL** | Service Control Manager RPC — exposes running services; can start/stop/create services |
| **ATSVC** | Task Scheduler RPC — can create/enumerate/delete scheduled tasks remotely |
| **WINREG** | Remote Registry RPC — allows reading/writing the Windows registry remotely |
| **Named pipe** | Windows IPC mechanism used as transport for RPC over SMB |
| **SID (Security Identifier)** | Unique identifier for a Windows security principal (user, group, computer) |
| **RID (Relative Identifier)** | The final portion of a SID that identifies a specific object within a domain |
| **SID brute-force** | Enumerating users by iterating RIDs from 500 upward |
| **rpcclient** | Linux tool for interactive RPC communication with Windows SMB servers |
| **enum4linux-ng** | Automated Windows enumeration tool combining RPC, SMB, and LDAP queries |
| **SMB signing** | Cryptographic signing of SMB packets preventing relay attacks; if disabled, NTLM relay is possible |
| **lookupsid.py** | Impacket script for SID/RID enumeration to discover users |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `nmap msrpc-enum` | `nmap --script msrpc-enum -p 135 target` | RPC endpoints | Discovery |
| `rpcclient` | `rpcclient -U "" -N target` | Null session | Initial access |
| `rpcclient` | `rpcclient -U "user%pass" target` | Auth enumeration | With credentials |
| `enum4linux-ng` | `enum4linux-ng -A target` | Full RPC/SMB dump | Quick sweep |
| `lookupsid.py` | `lookupsid.py domain/user:pass@target` | SID/user brute-force | User enumeration |
| `samrdump.py` | `samrdump.py domain/user:pass@target` | Full SAMR dump | AD user mapping |

**Complete RPC enumeration pipeline:**
```bash
TARGET="203.0.113.45"

# Step 1: Check endpoint mapper
$ nmap --script msrpc-enum -p 135 $TARGET

# Step 2: Test null session
$ rpcclient -U "" -N $TARGET -c "srvinfo" 2>&1 | grep -v "^$"

# Step 3: If null session works — full enumeration
$ rpcclient -U "" -N $TARGET -c "enumdomusers" | tee rpc_users.txt
$ rpcclient -U "" -N $TARGET -c "enumdomgroups" | tee rpc_groups.txt
$ rpcclient -U "" -N $TARGET -c "netshareenum" | tee rpc_shares.txt
$ rpcclient -U "" -N $TARGET -c "getdompwinfo" | tee rpc_policy.txt

# Step 4: SID brute-force
$ python3 /usr/share/doc/python3-impacket/examples/lookupsid.py anonymous:@$TARGET

# Step 5: Automated full scan
$ enum4linux-ng -A $TARGET -oJ rpc_results.json 2>/dev/null

# Parse results
$ python3 -c "
import json
data = json.load(open('rpc_results.json'))
users = data.get('users', {})
for rid, info in users.items():
    print(f'[{rid}] {info.get(\"username\", \"?\")}')
"
```

---

# Section 5 — Defender detection

- **Port 135 probe (Endpoint Mapper):** Connecting to port 135 and querying the endpoint mapper generates a Windows Security event (Event ID 4656 — object handle requested) on the target. Large-scale nmap scans of port 135 across many hosts are identifiable as automated scanning in Windows event logs.
- **Null session attempts:** Modern Windows (Server 2016+) denies null sessions and generates Event ID 4625 (failed logon) with logon type 3 (network) for each attempt. A burst of null session attempts from one source is an alert trigger in SIEM rules.
- **rpcclient enumdomusers:** Successful user enumeration via SAMR generates no specific event by default in standard Windows logging. However, SAMR enumeration can be logged with object access auditing enabled — it then generates Event ID 4661 for each user object accessed.
- **SID brute-force (lookupsid):** Sequential RID lookups (500, 501, 502...) generate a pattern of LSARPC requests that is distinctly automated. EDR behavioral rules flag sequential SID lookups as a user enumeration attack.
- **SMB signing disabled:** SMB signing disabled is itself a finding, not a detection. But exploiting it (NTLM relay attack) is highly visible in Windows Security logs.

---

# Section 6 — Lab task

**Platform:** Kali Linux + Metasploitable2 (runs Samba with null sessions enabled) or TryHackMe "Network Services 2" room.

**Objective:** Enumerate users, groups, and shares via RPC — discover all domain users using null session and SID brute-force.

**Steps:**

1. **Endpoint mapper:** `nmap --script msrpc-enum -p 135 <target>` — document all RPC services
2. **SMB security mode:** `nmap --script smb-security-mode -p 445 <target>` — check for null session and SMB signing
3. **Null session test:** `rpcclient -U "" -N <target> -c "srvinfo"`
4. **Server info:** `rpcclient -U "" -N <target> -c "srvinfo"` — document OS version
5. **User enumeration:** `rpcclient -U "" -N <target> -c "enumdomusers"` — list all users
6. **Group enumeration:** `rpcclient -U "" -N <target> -c "enumdomgroups"` — list all groups
7. **Share enumeration:** `rpcclient -U "" -N <target> -c "netshareenum"` — list all shares
8. **SID brute-force:** `python3 /usr/share/doc/python3-impacket/examples/lookupsid.py anonymous:@<target>`
9. **enum4linux-ng full scan:** `enum4linux-ng -A <target> 2>/dev/null | tee enum_results.txt`
10. **Compile `rpc_intel.md`:** All discovered users (with RIDs) | Groups (with members) | Shares (with paths) | Password policy | Services (if visible) | Security posture (null session Y/N, SMB signing Y/N)

```bash
git commit -m "recon(stage5): RPC enumeration — users/groups/shares enumerated for <target>"
```

---

# Section 7 — Common mistakes

**1. Only testing null session and giving up when it fails**
_Why it matters:_ Null sessions fail on modern Windows. But any valid domain credential — even a low-privilege user — enables the same SAMR enumeration via rpcclient. The same commands work with `-U "user%password"`.
_Fix:_ If null session fails, retry with credentials. Even `jsmith:Welcome1` from a password spray gives full SAMR access.

**2. Not checking the RID 500 user after renaming Administrator**
_Why it matters:_ Organizations rename the `Administrator` account to `admin`, `sysadmin`, or something custom. The account name changes but the RID remains 500. SID brute-force always finds the administrator regardless of the rename.
_Fix:_ Always resolve `S-1-5-21-<domain-SID>-500` explicitly. This is the built-in administrator regardless of what it's named.

**3. Missing SMB signing status**
_Why it matters:_ If SMB signing is disabled, NTLM relay attacks are possible — you can relay captured NTLM authentication hashes to authenticate against any target on the network, without cracking the hash. This is a critical finding independent of the enumeration data.
_Fix:_ Always run `nmap --script smb-security-mode -p 445` and document the `message_signing` field explicitly in your report.

**4. Not reading the `description` field from rpcclient queryuser**
_Why it matters:_ Like LDAP, administrators put passwords and notes in the SAMR description field. `rpcclient queryuser` includes the `Description` field in its output.
_Fix:_ After `enumdomusers`, run `queryuser <RID>` for all discovered accounts. Grep output for `Description:` and check for embedded credentials.

**5. Treating `NT_STATUS_ACCOUNT_LOCKED_OUT` as a failed login and retrying**
_Why it matters:_ Retrying a locked account extends the observation window and generates additional lockout events. Security teams are alerted on consecutive lockout events. Continuing to retry after a lockout confirmation is an operational mistake that increases detection risk.
_Fix:_ The moment `NT_STATUS_ACCOUNT_LOCKED_OUT` appears, stop all authentication attempts for that account. Wait the full lockout duration before attempting again. Document the locked account for the engagement report.

---

# Section 8 — Move-on gate

1. `rpcclient -U "" -N 203.0.113.45 -c "enumdomusers"` returns a list of 12 users. Without notes, explain what "null session" means, why this is a security misconfiguration, and name two pieces of information about each user you would then retrieve using `queryuser <RID>`.

2. SID brute-force returns `S-1-5-21-XXXX-500 CORP\sysadmin`. Without notes, state what the RID 500 tells you about this account regardless of the username, what attack implication this has, and why renaming the account does not provide meaningful security.

3. `nmap --script smb-security-mode` returns `message_signing: disabled`. Without notes, state what attack this enables, what tool implements it, and what the attacker receives at the end of the attack without cracking any hash.
