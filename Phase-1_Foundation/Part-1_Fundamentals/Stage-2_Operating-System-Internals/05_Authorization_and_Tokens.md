# Authorization & Tokens

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → Authorization & Tokens

# Section 1 — What it is and where it sits

Authorization is the operating system's decision about **what an identity is allowed to do**. In Windows, the core objects are **Security Identifiers (SIDs)**, **access tokens**, security descriptors, and access checks. A process normally carries an access token containing its security identity, groups, privileges, and related security information. **Token impersonation** allows a thread to temporarily act using another security context.

Linux uses a different model: credentials such as UID/GID and capabilities are combined with **namespaces** and **cgroups** to isolate and control processes. Namespaces create separate views of resources; cgroups control and account for resource consumption. Together they are fundamental building blocks behind Linux containers.

```text
OS Internals
│
├── Privilege Levels
├── System Calls
├── Process & Thread Mechanics
├── PEB
│
└── Authorization & Tokens
      │
      ├── Windows
      │    ├── SID
      │    ├── Access Token
      │    └── Token Impersonation
      │
      └── Linux
           ├── Namespaces
           └── Cgroups
                    ↓
              Containers
```

If you misunderstand this topic, privilege escalation becomes guesswork: you will see "Administrator", "SYSTEM", "root", containers, or impersonation and not understand **which security identity actually authorizes the operation**.

This builds on processes, threads, and system calls and leads directly into Windows privilege escalation, access-control abuse, Linux container security, and identity-based defense evasion.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for on Windows

An attacker who already has code execution wants to answer:

```text
Who am I?
What groups am I in?
What privileges does my token contain?
Which process has a more valuable token?
Can I cause code to execute under that token?
```

The useful information includes:

- User SID
- Group SIDs
- Restricted SIDs
- Enabled/disabled privileges
- Integrity level
- Token type
- Impersonation level
- Whether the token belongs to a service, administrator, or `SYSTEM`
- Processes carrying interesting tokens

The practical chain is:

```text
Initial foothold
      ↓
Inspect token
      ↓
Identify current identity/privileges
      ↓
Find more privileged security context
      ↓
Obtain or impersonate token
      ↓
Execute under stronger identity
```

## 2.2 SIDs — the identity Windows actually checks

A Windows username is a human-readable label.

A **SID** is the machine-readable security identifier used in authorization.

Conceptually:

```text
Username
   ↓
SID
   ↓
Groups / privileges
   ↓
Access check
```

For example:

```text
BUILTIN\Administrators
        ↓
S-1-5-32-544
```

An attacker therefore cares less about the display name and more about the underlying SID and the token containing it.

## 2.3 Access-token analysis

A token effectively represents:

```text
"This thread/process is operating with this security identity."
```

A simplified token might contain:

```text
User SID
 └── S-1-5-21-...

Group SIDs
 ├── Administrators
 ├── Users
 └── ...
 
Privileges
 ├── SeDebugPrivilege
 ├── SeImpersonatePrivilege
 └── ...

Integrity level
 └── High
```

The attacker checks whether the current token has something valuable.

Examples of high-value privileges can include:

- `SeDebugPrivilege`
- `SeImpersonatePrivilege`
- `SeAssignPrimaryTokenPrivilege`
- `SeBackupPrivilege`
- `SeRestorePrivilege`
- `SeTakeOwnershipPrivilege`

Having a privilege does not automatically mean it is enabled or that exploitation is possible. The exact token state matters.

## 2.4 The privilege-escalation pivot

Suppose an attacker controls:

```text
Medium-integrity user process
```

but discovers a service running as:

```text
NT AUTHORITY\SYSTEM
```

The interesting question is not simply:

> "Can I access the process?"

It is:

> "Can I obtain, duplicate, impersonate, or otherwise abuse a security token associated with that stronger identity?"

The conceptual path becomes:

```text
Low-privileged process
       ↓
Identify privileged process
       ↓
Inspect token/security boundary
       ↓
Token abuse
       ↓
Higher-privileged thread/process
```

This is one of the core concepts behind Windows post-exploitation.

## 2.5 Token impersonation

**Impersonation** means a thread can temporarily perform operations using a client's security identity instead of only its own process identity.

A common server pattern is:

```text
Client
  ↓
Server
  ↓
Server impersonates client
  ↓
Server accesses resource as client
```

That's legitimate operating-system functionality.

Attackers are interested when a privileged service has dangerous impersonation opportunities.

Conceptually:

```text
Attacker-controlled input
        ↓
Privileged service
        ↓
Security token becomes available
        ↓
Thread impersonates privileged identity
        ↓
Privileged operation
```

The security impact depends heavily on which token and which privileges are involved.

## 2.6 Why `SeImpersonatePrivilege` matters

`SeImpersonatePrivilege` is particularly relevant to Windows privilege-escalation research because it allows a process to impersonate a security context under supported conditions.

The attacker looks for situations where:

```text
Current process
     ↓
has SeImpersonatePrivilege
     ↓
can interact with a privileged service/token source
     ↓
can cause a privileged token to be represented
     ↓
executes under stronger identity
```

The important lesson is not to memorize one exploit.

The reusable skill is:

```text
Token privilege
     +
Service interaction
     +
Impersonation primitive
     =
Potential escalation path
```

## 2.7 Dead-end vs high-value Windows finding

**Dead-end:**

```text
User:
DOMAIN\alice

Groups:
Users
```

This identifies the current context but does not establish an escalation path.

**High-value:**

```text
Process token:
User: low-privileged account
Privilege:
SeImpersonatePrivilege: Enabled

Target:
privileged service
```

Now there is a concrete authorization boundary worth investigating.

The privilege itself is not the exploit; it is the **enabling condition**.

## 2.8 Linux namespaces — the attacker perspective

Linux namespaces change what a process can **see and interact with**.

Instead of every process seeing one universal environment:

```text
Host
├── Process A
├── Process B
└── Process C
```

a namespace can provide an isolated view:

```text
Host
│
├── Namespace A
│    ├── PID 1
│    └── processes visible here
│
└── Namespace B
     ├── PID 1
     └── different process view
```

Attackers care because container isolation depends heavily on namespaces.

They investigate:

- PID namespace
- Mount namespace
- Network namespace
- User namespace
- IPC namespace
- UTS namespace
- Cgroup namespace

A container escape often involves finding a way to cross an isolation boundary into the host.

## 2.9 Namespaces do not create a tiny virtual machine

This is a common misconception.

A container usually shares the host kernel.

Conceptually:

```text
Container
  └── isolated user-space view
          │
          ▼
       Host kernel
```

Therefore:

```text
VM:
Guest kernel
     ↕
Hypervisor
     ↕
Host

Container:
Container processes
     ↕
Shared host kernel
```

This is why kernel vulnerabilities can become especially serious for container environments.

## 2.10 Cgroups — what attackers and defenders care about

**Control groups (cgroups)** organize processes and apply resource accounting/control.

They can control or track resources such as:

- CPU
- Memory
- PIDs
- I/O

Conceptually:

```text
cgroup
├── Process A
├── Process B
└── Process C

Limits/accounting
├── CPU
├── Memory
├── PIDs
└── I/O
```

For an attacker, cgroups can reveal that they are inside a container and help fingerprint the environment.

For defenders, cgroups help enforce resource boundaries and constrain damage from a compromised workload.

## 2.11 Namespaces + cgroups = container foundation

A simplified container model is:

```text
Container runtime
       │
       ├── Namespaces
       │     └── Isolation
       │
       ├── Cgroups
       │     └── Resource control
       │
       ├── Filesystem
       │     └── Root filesystem
       │
       └── Capabilities / seccomp
             └── Reduced privilege
```

Namespaces answer roughly:

> "What can this process see?"

Cgroups answer roughly:

> "How much of the system can this process consume?"

Capabilities and other security mechanisms answer:

> "Which privileged operations can it perform?"

## 2.12 Container escape reasoning

Suppose an attacker gets code execution in a container:

```text
Container process
      ↓
root inside container
```

That does **not** automatically mean:

```text
root on host
```

The attacker next examines:

```text
User namespace
Capabilities
Mounted host paths
Docker/container runtime socket
Devices
Kernel version
Seccomp restrictions
Namespace configuration
Cgroups
```

A high-value finding might be:

```text
Container
  ↓
Privileged capability
  ↓
Host filesystem/device exposure
  ↓
Host interaction
  ↓
Potential container escape
```

This is why "container root" and "host root" must never be treated as synonyms.

# Section 3 — Core concepts and terminology

| Term | Meaning |
|---|---|
| **SID** | Windows security identifier representing a user, group, or other security principal. |
| **Security principal** | Entity that can be authenticated and authorized, such as a user or group. |
| **Access token** | Windows security context containing identity, groups, privileges, and related authorization information. |
| **Primary token** | Token normally associated with a process and used for its default security context. |
| **Impersonation token** | Token associated with a thread that causes it to act as another security identity. |
| **Token impersonation** | Using an impersonation token so a thread performs access checks as another identity. |
| **Privilege** | Special Windows capability that grants a class of sensitive operations. |
| **Integrity level** | Windows security level used to constrain interactions between processes. |
| **Security descriptor** | Object metadata defining ownership and access-control information. |
| **ACL** | Access Control List containing access-control entries. |
| **ACE** | Individual allow/deny entry in an ACL. |
| **Access check** | Kernel decision determining whether a token can perform a requested operation. |
| **`SeImpersonatePrivilege`** | Windows privilege allowing supported impersonation operations. |
| **`SeDebugPrivilege`** | Privilege that permits powerful debugging/access operations against processes. |
| **Namespace** | Linux isolation mechanism providing processes with a separate view of selected system resources. |
| **PID namespace** | Namespace providing an isolated process-ID view. |
| **Mount namespace** | Namespace isolating filesystem mount views. |
| **Network namespace** | Namespace isolating network interfaces, routes, and related network state. |
| **User namespace** | Namespace isolating user/group ID mappings and capabilities. |
| **IPC namespace** | Namespace isolating System V IPC and POSIX message-queue resources. |
| **UTS namespace** | Namespace isolating hostname/domain-name information. |
| **Cgroup namespace** | Provides an isolated view of cgroup paths for processes. |
| **cgroup** | Linux mechanism for grouping processes and controlling/accounting for resources. |
| **Container** | Isolated application environment built using kernel mechanisms such as namespaces and cgroups. |
| **Container escape** | Moving from container isolation into the underlying host or another security boundary. |
| **Capability** | Fine-grained Linux privilege replacing some aspects of all-powerful root authority. |
| **Seccomp** | Linux mechanism for restricting the system calls available to a process. |

### Windows token components

| Component | Security significance |
|---|---|
| User SID | Identifies the primary user |
| Group SIDs | Determine group-based authorization |
| Privileges | Grant special operating-system capabilities |
| Integrity level | Controls trust boundaries between processes |
| Token type | Distinguishes primary/impersonation behavior |
| Impersonation level | Determines how an impersonating thread can use another identity |

### Linux namespace variants

| Namespace | Isolates |
|---|---|
| PID | Process ID view |
| Mount | Filesystem mount tree |
| Network | Interfaces/routes/network stack |
| User | UID/GID mappings and capabilities |
| IPC | IPC objects |
| UTS | Hostname/domain name |
| Cgroup | Cgroup view |

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|---|---|---|---|
| `whoami` | `whoami` | Current Windows identity | Quick identity check |
| `whoami` | `whoami /user` | Current SID | Identify user SID |
| `whoami` | `whoami /groups` | Group SIDs and memberships | Analyze token groups |
| `whoami` | `whoami /priv` | Available Windows privileges | Identify interesting token privileges |
| `whoami` | `whoami /all` | Combined identity/group/privilege information | Fast token baseline |
| PowerShell | `Get-Process` | Running processes | Find interesting processes |
| Sysinternals Process Explorer | `procexp64.exe` | Process/security details | Investigate tokens and processes |
| Sysinternals Process Hacker | `procexp` / GUI | Processes, tokens, handles | Interactive token analysis |
| `Get-Process` | `Get-Process -Id <PID> \| Select-Object *` | Detailed process properties | Baseline target process |
| `Get-Process` | `Get-Process -IncludeUserName` | Process username | Identify process owner |
| `id` | `id` | Linux UID/GID/groups | Identity baseline |
| `lsns` | `lsns` | Active Linux namespaces | Identify namespace isolation |
| `nsenter` | `nsenter -t <PID> -m -u -i -n -p` | Enter target process namespaces | Authorized container/lab analysis |
| `cat` | `cat /proc/<PID>/cgroup` | Process cgroup membership | Identify cgroup placement |
| `cat` | `cat /proc/self/status` | UID, capabilities, namespace-related process state | Container/process inspection |
| `capsh` | `capsh --print` | Current Linux capabilities | Analyze privilege inside containers |
| `unshare` | `unshare --pid --fork --mount-proc sh` | Creates isolated namespaces | Build a namespace lab |
| `systemd-cgls` | `systemd-cgls` | Cgroup hierarchy | Inspect resource-control tree |

### Windows — `whoami /user`

```text
C:\> whoami /user

USER INFORMATION
----------------
User Name       SID
DOMAIN\alice    S-1-5-21-...
```

The important value is the SID.

The account name is human-readable; the SID is the security identifier used by Windows authorization mechanisms.

### Windows — `whoami /groups`

```text
C:\> whoami /groups

GROUP INFORMATION
...
BUILTIN\Users
S-1-5-32-545
...
```

This reveals group membership associated with the current token.

### Windows — `whoami /priv`

```text
C:\> whoami /priv

Privilege Name                  State
SeChangeNotifyPrivilege        Enabled
SeImpersonatePrivilege         Disabled
...
```

The distinction between:

```text
Privilege exists
```

and:

```text
Privilege is enabled
```

matters during analysis.

### Windows — `whoami /all`

```text
C:\> whoami /all

USER INFORMATION
GROUP INFORMATION
PRIVILEGES INFORMATION
...
```

This is useful as the first token baseline.

### Windows — PowerShell process enumeration

```text
PS> Get-Process | Select-Object Id, ProcessName

Id    ProcessName
--    -----------
4     System
...
```

This gives a process inventory before deeper token analysis.

### Linux — `id`

```text
$ id
uid=1000(user) gid=1000(user) groups=1000(user)
```

This is the Linux identity baseline.

Inside a container, UID 0 may appear as root while being mapped through a user namespace.

### Linux — `lsns`

```text
$ lsns
NS         TYPE   NPROCS   PID
402653...  mnt    ...      1
402653...  pid    ...      1
402653...  net    ...      1
```

This identifies namespace instances.

The important question is whether your process belongs to the same namespace as the host processes.

### Linux — `/proc/<PID>/cgroup`

```text
$ cat /proc/1234/cgroup
0::/some/container/path
```

This shows the process's cgroup placement.

### Linux — `capsh --print`

```text
$ capsh --print
Current: cap_chown,cap_setuid,...
Bounding set = ...
```

This reveals Linux capabilities available to the current process.

### Linux — `unshare`

```text
$ unshare --pid --fork --mount-proc sh
# echo $$
1
```

Inside the new PID namespace, the shell can appear as PID 1 within that namespace.

That demonstrates that namespace PIDs are relative to the namespace rather than automatically equal to host PIDs.

### Linux — `nsenter`

```text
$ sudo nsenter -t 1234 -m -u -i -n -p
```

This enters selected namespaces associated with an existing process.

In a lab, it is useful for understanding exactly what namespace isolation changes.

### Linux — `systemd-cgls`

```text
$ systemd-cgls
Control group
├─system.slice
│ ├─...
└─user.slice
   └─...
```

This displays the cgroup hierarchy on systems using systemd.

# Section 5 — Defender detection

- **Windows token analysis:** EDR and security tooling can inspect process identities, token privileges, integrity levels, and suspicious impersonation behavior.
- **Privilege use:** Unexpected use of powerful privileges such as `SeDebugPrivilege` or `SeImpersonatePrivilege` should receive attention when associated with unusual processes.
- **Token anomalies:** A process running as one identity but performing operations using an unexpected impersonation context can be highly significant.
- **Process lineage:** Correlate token changes with the process tree; a network-facing service suddenly producing a `SYSTEM` execution context is more meaningful than the token alone.
- **Linux container telemetry:** Monitor namespace/cgroup membership, capabilities, privileged containers, host-path mounts, and access to container runtime sockets.
- **What defenders miss:** They may equate `root` inside a container with host root, or monitor UID changes while ignoring Linux capabilities and namespace boundaries.
- **Skilled operators:** Attackers may abuse legitimate tokens or inherited security contexts rather than creating a conspicuous new administrator account, so token lineage and access patterns matter.

# Section 6 — Lab task

**Platform:** Two-part local lab using a Windows analysis VM and a Linux VM. The task is one authorization investigation covering the two OS models.

**Objective:** Enumerate the Windows security token and Linux namespace/cgroup context of your own lab processes and explain exactly what identity and isolation boundaries apply.

**Steps:**

1. On Windows, record your current SID, group membership, and privileges.
2. Start a harmless process and identify its PID and owning account.
3. Inspect the process in Process Explorer and examine its token-related information.
4. Record one important difference between user SID, group SID, privilege, and integrity level.
5. On Linux, record your current UID/GID and capabilities.
6. Identify the namespaces used by your shell with `lsns`.
7. Inspect your shell's cgroup membership through `/proc/self/cgroup`.
8. Create a small PID namespace with `unshare` and verify that your shell receives a namespace-local PID.
9. Compare the namespace/cgroup state before and after isolation.
10. Save the evidence and write a diagram showing Windows token authorization versus Linux namespace/cgroup isolation.

**Expected output:**

```text
WINDOWS

User
 ↓
SID
 ↓
Groups + Privileges + Integrity
 ↓
Access Check
 ↓
Resource

LINUX

Process
 ↓
UID/GID + Capabilities
 ↓
Namespaces
 ↓
Cgroups
 ↓
Isolated resource view/control
```

You should be able to explain why:

```text
Windows Administrator
```

is not simply the same concept as:

```text
Linux root
```

and why:

```text
container UID 0
```

does not automatically imply host-level authority.

**Git artifact:**

```text
authorization-tokens/
├── README.md
├── windows/
│   ├── whoami-user.txt
│   ├── whoami-groups.txt
│   ├── whoami-priv.txt
│   └── token-notes.md
├── linux/
│   ├── id.txt
│   ├── namespaces.txt
│   ├── cgroup.txt
│   ├── capabilities.txt
│   └── namespace-lab.txt
└── notes.md
```

```bash
git add authorization-tokens/
git commit -m "Document Windows tokens and Linux isolation primitives"
```

# Section 7 — Common mistakes

1. **Mistake:** Treating a username as the Windows authorization identity.  
   **Why it matters:** Access checks use security identifiers and token information, not merely the displayed name.  
   **Do instead:** Track the SID and token.

2. **Mistake:** Assuming every privilege shown by `whoami /priv` is actively usable.  
   **Why it matters:** Token privileges can be disabled or otherwise constrained.  
   **Do instead:** Inspect the privilege state and execution context.

3. **Mistake:** Thinking impersonation changes the entire process identity.  
   **Why it matters:** Impersonation is primarily associated with a thread's security context.  
   **Do instead:** Distinguish primary process tokens from thread impersonation tokens.

4. **Mistake:** Treating `root` in a container as host root.  
   **Why it matters:** User namespaces and other isolation mechanisms can separate container identity from host identity.  
   **Do instead:** Check namespace mappings, capabilities, devices, and host exposure.

5. **Mistake:** Thinking namespaces alone provide complete container security.  
   **Why it matters:** Namespace isolation is only one layer.  
   **Do instead:** Evaluate namespaces together with capabilities, seccomp, filesystem restrictions, and cgroups.

6. **Mistake:** Treating cgroups as a security identity mechanism.  
   **Why it matters:** Cgroups primarily provide resource accounting and control.  
   **Do instead:** Use namespaces for isolation and cgroups for resource governance, while analyzing credentials/capabilities separately.

7. **Mistake:** Looking for only explicit password-based privilege escalation.  
   **Why it matters:** Token abuse and inherited security contexts can produce privilege changes without new credentials.  
   **Do instead:** Inspect how processes obtain and use their security context.

# Section 8 — Move-on gate

1. **Run `whoami /all` on Windows and correctly identify your SID, one group SID, one privilege, and its current state without looking at your notes.**

2. **On Linux, run `lsns`, inspect `/proc/self/cgroup`, and identify which namespace provides PID isolation and which mechanism controls resource membership without referring to the note.**

3. **Create a PID namespace with `unshare`, observe the namespace-local PID, then explain from your recorded evidence why that PID namespace does not by itself grant or remove UID-based privileges.**