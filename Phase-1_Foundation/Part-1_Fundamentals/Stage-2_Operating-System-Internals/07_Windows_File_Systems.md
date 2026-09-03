# Windows File Systems

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → File Systems (Windows)

# Section 1 — What it is and where it sits

Windows filesystem security is built heavily around **NTFS permissions**, while system configuration is exposed through the **Windows Registry**. NTFS uses security descriptors containing owners, DACLs, and SACLs to determine and audit access. Registry information is stored persistently in **hives**, while **Alternate Data Streams (ADS)** allow additional named data streams to exist alongside a file's normal `$DATA` stream.

From a red-team perspective, these mechanisms matter because filesystem permissions determine what you can modify, Registry locations reveal persistence and configuration, and ADS can conceal data from tools that only inspect the normal filename/content view.

```text
Windows File-System Attack Surface
│
├── NTFS
│   ├── Files / directories
│   ├── ACLs
│   ├── Security descriptors
│   └── ADS
│
├── Registry
│   ├── Hives
│   ├── Keys
│   └── Values
│
└── Runtime
    ├── Processes
    ├── Services
    └── User profiles
```

If you skip this, you will miss writable privileged files, dangerous Registry configuration, persistence locations, forensic artifacts, and hidden NTFS content.

This builds on Linux filesystem concepts by moving from inode/mode-bit thinking to Windows ACLs, Registry objects, and NTFS-specific features.

# Section 2 — How attackers actually use this

## 2.1 NTFS permission analysis

The first question is:

```text
Who am I?
        ↓
Which groups?
        ↓
What rights does the token have?
        ↓
Which filesystem objects can I modify?
```

Attackers specifically look for:

* Writable executables
* Writable service binaries
* Writable service directories
* Writable scheduled-task files/scripts
* Weak ACLs on privileged application directories
* Files owned by privileged users but modifiable by ordinary users

A dangerous relationship is:

```text
Low-privileged user
       ↓
Can modify privileged executable/configuration
       ↓
Privileged service executes it
       ↓
Potential privilege escalation
```

The permission alone is not the vulnerability.

The vulnerable condition is:

```text
UNTRUSTED WRITER
      +
PRIVILEGED CONSUMER
      +
EXECUTABLE / TRUSTED OBJECT
```

## 2.2 DACL reasoning

Suppose:

```text
C:\Program Files\App\service.exe
```

is executed by a service running as `SYSTEM`.

If an ordinary user can modify `service.exe`:

```text
user
 ↓ write
service.exe
 ↑ executed by
SYSTEM
```

that is a much higher-value finding than simply discovering that the file exists.

Always inspect the effective permissions rather than relying on the filename or directory name.

## 2.3 Registry hive analysis

The Registry is hierarchical:

```text
Hive
 └── Key
      └── Subkey
           └── Value
```

Attackers investigate Registry data for:

* Startup locations
* Service configuration
* Installed software
* User configuration
* Credentials/secrets
* Security settings
* Application configuration
* Execution history
* Evidence of prior activity

Important root hives include:

```text
HKLM
HKCU
HKCR
HKU
HKCC
```

An analyst should think:

```text
Registry key
      ↓
What does it control?
      ↓
Who can modify it?
      ↓
What trusted component consumes it?
```

A writable autorun-related Registry key can become a persistence mechanism.

## 2.4 Registry persistence reasoning

The dangerous pattern is:

```text
Low-privileged attacker
        ↓
Can modify startup configuration
        ↓
User logs in / program starts
        ↓
Windows executes attacker-controlled command
```

A Registry location is interesting because of its **execution semantics**, not merely because it contains suspicious text.

## 2.5 Registry hives as forensic artifacts

Common persistent hive files include:

```text
%SystemRoot%\System32\Config\SYSTEM
%SystemRoot%\System32\Config\SOFTWARE
%SystemRoot%\System32\Config\SAM
%SystemRoot%\System32\Config\SECURITY
```

User-specific Registry data is commonly associated with:

```text
NTUSER.DAT
USRCLASS.DAT
```

An investigator can correlate these hives with:

```text
User activity
Software installation
Configuration changes
Persistence
System state
```

## 2.6 Alternate Data Streams

NTFS supports multiple named streams attached to a file.

Conceptually:

```text
example.txt
│
├── $DATA          ← normal content
└── secret         ← alternate data stream
```

The normal file view may show:

```text
example.txt
```

while NTFS actually contains additional stream data.

That makes ADS useful for both legitimate applications and malicious concealment.

## 2.7 Why ADS is interesting to attackers

A poorly designed inspection workflow might check:

```text
filename
size
normal file contents
```

while failing to enumerate additional streams.

An attacker can therefore use ADS to conceal:

* Text data
* Configuration
* Staged information
* Malicious content

The important distinction is:

```text
ADS ≠ automatically malicious
```

NTFS supports streams legitimately.

The red-team concern is **unexpected streams attached to sensitive or trusted files**.

## 2.8 ADS attack chain

Conceptually:

```text
Attacker
  ↓
Chooses legitimate-looking NTFS file
  ↓
Creates named stream
  ↓
Stores concealed data
  ↓
Retrieves it later
  ↓
Normal filename remains unchanged
```

A defender inspecting only the ordinary file content may miss the extra stream.

## 2.9 Dead-end vs high-value finding

**Dead-end:**

```text
C:\Users\Alice\document.txt
```

exists.

That tells you almost nothing.

**High-value:**

```text
document.txt
 └── unexpected:$DATA-like named stream
```

combined with:

```text
untrusted user
      ↓
stream contains executable/script content
```

Now there is a concrete concealment indicator.

Likewise:

```text
SYSTEM service binary
        ↓
ordinary users have Modify
```

is far more valuable than simply finding a service executable.

## 2.10 Where the results feed next

```text
NTFS ACL analysis
      ↓
Writable privileged object
      ↓
Privilege escalation
```

```text
Registry analysis
      ↓
Execution / persistence location
      ↓
Execution investigation
```

```text
ADS discovery
      ↓
Hidden content
      ↓
Malware / forensic analysis
```

# Section 3 — Core concepts and terminology

| Term                      | Meaning                                                                                      |
| ------------------------- | -------------------------------------------------------------------------------------------- |
| **NTFS**                  | Windows filesystem supporting ACLs, metadata, journaling, ADS, and other features.           |
| **ACL**                   | Access Control List governing access to an object.                                           |
| **DACL**                  | ACL containing allow/deny access entries.                                                    |
| **SACL**                  | ACL specifying auditing rules.                                                               |
| **ACE**                   | Individual access-control entry inside an ACL.                                               |
| **Security descriptor**   | Object security metadata containing owner, DACL, and SACL information.                       |
| **Inheritance**           | Permissions propagated from a parent object to descendants.                                  |
| **Registry**              | Windows hierarchical configuration database.                                                 |
| **Hive**                  | Registry storage unit containing a subtree and its persistent data.                          |
| **Key**                   | Registry container similar conceptually to a directory.                                      |
| **Value**                 | Named data stored within a Registry key.                                                     |
| **HKLM**                  | HKEY_LOCAL_MACHINE; machine-wide configuration.                                              |
| **HKCU**                  | HKEY_CURRENT_USER; current user's configuration.                                             |
| **HKCR**                  | HKEY_CLASSES_ROOT; merged class/file-association view.                                       |
| **HKU**                   | HKEY_USERS; loaded user profiles.                                                            |
| **NTUSER.DAT**            | Per-user Registry hive file.                                                                 |
| **USRCLASS.DAT**          | Per-user class/Explorer-related Registry hive data.                                          |
| **ADS**                   | Alternate Data Stream attached to an NTFS file.                                              |
| **Named stream**          | Additional stream identified by a stream name.                                               |
| **`$DATA`**               | NTFS attribute type storing file data; the unnamed stream is the normal file content.        |
| **Persistence**           | Mechanism allowing attacker-controlled execution to survive events such as reboot or logon.  |
| **Effective permissions** | Actual permissions resulting from ACLs, group membership, inheritance, and applicable rules. |

### Important Registry hives

| Hive            | Typical role                        |
| --------------- | ----------------------------------- |
| `HKLM\SYSTEM`   | OS boot/system configuration        |
| `HKLM\SOFTWARE` | Machine-wide software/configuration |
| `HKLM\SAM`      | Local account database information  |
| `HKLM\SECURITY` | Security-policy-related information |
| `HKCU`          | Current-user configuration          |
| `HKU`           | User-profile Registry data          |

# Section 4 — Tools and commands

| Tool       | Command                                               | What it finds/shows            | When to use it                  |
| ---------- | ----------------------------------------------------- | ------------------------------ | ------------------------------- |
| `icacls`   | `icacls "C:\path\file.exe"`                           | NTFS ACLs                      | Inspect file permissions        |
| `icacls`   | `icacls "C:\path" /inheritance:e`                     | Inheritance configuration      | Analyze inherited permissions   |
| PowerShell | `Get-Acl "C:\path\file.exe"`                          | Security descriptor/ACL        | Scriptable ACL analysis         |
| `reg`      | `reg query HKLM\SOFTWARE`                             | Registry keys/values           | Registry enumeration            |
| `reg`      | `reg query HKCU\Software`                             | User Registry                  | Per-user configuration          |
| PowerShell | `Get-ItemProperty "HKLM:\SOFTWARE\..."`               | Registry values                | PowerShell analysis             |
| `reg`      | `reg save HKLM\SYSTEM system.hiv`                     | Hive export                    | Authorized forensic acquisition |
| `dir`      | `dir /R "C:\path"`                                    | Alternate streams              | ADS discovery                   |
| PowerShell | `Get-Item -Path "C:\path\file.txt" -Stream *`         | NTFS streams                   | ADS enumeration                 |
| PowerShell | `Get-Content -Path "C:\path\file.txt" -Stream <name>` | Stream contents                | Analyze known ADS               |
| `fsutil`   | `fsutil file queryfileid <file>`                      | NTFS file ID                   | Filesystem metadata analysis    |
| `where`    | `where <program>`                                     | Executable location            | Verify which binary executes    |
| `sc`       | `sc qc <service>`                                     | Service configuration          | Identify privileged consumers   |
| `reg`      | `reg query "HKLM\SYSTEM\CurrentControlSet\Services"`  | Service Registry configuration | Service investigation           |

### `icacls`

```text
C:\> icacls "C:\Program Files\App\service.exe"

service.exe BUILTIN\Administrators:(F)
            NT AUTHORITY\SYSTEM:(F)
            Users:(RX)
```

Interpretation:

```text
Administrators → Full
SYSTEM        → Full
Users         → Read/Execute
```

If `Users` instead had `M`, `W`, or another modification-capable right, investigate whether a privileged service executes the file.

### `Get-Acl`

```powershell
PS> (Get-Acl "C:\Program Files\App\service.exe").Access
```

Example:

```text
IdentityReference : BUILTIN\Users
FileSystemRights  : ReadAndExecute
AccessControlType : Allow
```

This is useful when you need programmatic ACL analysis.

### `reg query`

```cmd
C:\> reg query HKLM\SOFTWARE\Example

HKEY_LOCAL_MACHINE\SOFTWARE\Example
    InstallPath    REG_SZ    C:\Program Files\Example
```

The key/value relationship is:

```text
Key
 └── InstallPath
       └── C:\Program Files\Example
```

### `reg save`

```cmd
C:\> reg save HKLM\SYSTEM system.hiv
The operation completed successfully.
```

This creates a hive copy suitable for authorized offline analysis.

### `dir /R`

```cmd
C:\> dir /R sample.txt

sample.txt
             42 sample.txt:hidden
```

The important observation is that the ordinary filename has an additional named stream:

```text
sample.txt:hidden
```

### PowerShell ADS enumeration

```powershell
PS> Get-Item -Path .\sample.txt -Stream *
```

Possible output:

```text
PSPath       : Microsoft.PowerShell.Core\FileSystem::...\sample.txt::$DATA
Stream       : :$DATA

PSPath       : ...\sample.txt:hidden
Stream       : hidden
```

The additional stream is separate from the normal unnamed stream.

### `sc qc`

```cmd
C:\> sc qc ExampleService
SERVICE_NAME: ExampleService
        BINARY_PATH_NAME : C:\Program Files\Example\service.exe
```

Combine this with ACL analysis:

```text
Service runs with privilege
        +
Binary path
        +
Binary ACL
```

That is the useful security relationship.

# Section 5 — Defender detection

* **NTFS auditing:** SACLs and Windows security telemetry can record access to sensitive files when auditing is configured.
* **Permission monitoring:** Unexpected ACL changes on service binaries, scheduled-task resources, startup locations, or privileged application directories are high-value events.
* **Registry monitoring:** Changes to security-sensitive Registry locations can reveal persistence or configuration tampering.
* **ADS detection:** Endpoint tools can enumerate named NTFS streams and alert on unusual streams containing executable or script-like data.
* **Forensic correlation:** A suspicious ADS becomes stronger evidence when its parent file, creator process, timestamps, and subsequent execution behavior are correlated.
* **What defenders miss:** Basic directory listings and many simplistic scanners focus on the normal filename and may not expose alternate streams.
* **Operator footprint:** Skilled attackers may choose inconspicuous filenames/streams and legitimate Registry locations, so defenders should detect abnormal modification and execution relationships rather than relying only on suspicious names.

# Section 6 — Lab task

**Platform:** Windows 10/11 analysis VM.

**Objective:** Demonstrate NTFS ACL analysis, Registry inspection, and ADS discovery using only benign test data.

**Steps:**

1. Create a temporary lab directory containing a harmless text file.
2. Inspect the file's NTFS ACL and record owner, groups, and effective rights.
3. Create a benign named ADS containing the text `NTFS-LAB`.
4. Enumerate the file's streams and verify that the normal file remains unchanged.
5. Read the named stream and record its contents.
6. Query a harmless Registry key and document its key/value structure.
7. Export an appropriate non-sensitive lab Registry hive/key for offline inspection.
8. Create a second test file with deliberately different ACLs and compare the results.
9. Record which findings would be ordinary filesystem behavior versus security-relevant anomalies.
10. Remove the test ADS, temporary files, and exported Registry artifacts.

**Expected output:**

```text
test.txt
 ├── normal content
 └── hidden-lab
       └── NTFS-LAB
```

and:

```text
ACL
 ├── Owner
 ├── Allow entries
 └── Effective rights
```

You should be able to distinguish:

```text
NTFS filename
NTFS stream
Registry key
Registry value
ACL entry
```

without confusing them.

**Git artifact:**

```text
windows-filesystem/
├── README.md
├── evidence/
│   ├── icacls.txt
│   ├── ads.txt
│   ├── registry.txt
│   └── comparison.txt
└── notes.md
```

```bash
git add windows-filesystem/
git commit -m "Document NTFS ACLs Registry hives and ADS"
```

# Section 7 — Common mistakes

1. **Mistake:** Assuming `Read` means `Modify`.
   **Why it matters:** Privilege escalation often depends on a specific write/modify capability.
   **Do instead:** Inspect the actual effective rights.

2. **Mistake:** Treating every writable file as exploitable.
   **Why it matters:** A writable file is useful only when a privileged/trusted component consumes it in a dangerous way.
   **Do instead:** Find the writer → privileged consumer relationship.

3. **Mistake:** Thinking the Registry is a filesystem directory tree.
   **Why it matters:** Registry keys and values are different objects from NTFS files and directories.
   **Do instead:** Think hive → key → value.

4. **Mistake:** Assuming ADS means malware.
   **Why it matters:** Alternate streams are legitimate NTFS functionality.
   **Do instead:** Investigate unexpected streams and correlate them with process/file activity.

5. **Mistake:** Searching only normal file contents.
   **Why it matters:** Named streams can contain separate data.
   **Do instead:** Enumerate streams during forensic or malware investigations.

6. **Mistake:** Ignoring inherited permissions.
   **Why it matters:** A directory's ACL can materially affect descendants.
   **Do instead:** Inspect inheritance and calculate effective rights.

7. **Mistake:** Treating a suspicious Registry value as automatically executable.
   **Why it matters:** Registry values have different consumers and semantics.
   **Do instead:** Identify which Windows component reads the key and under what event.

# Section 8 — Move-on gate

1. **Run `icacls` against a lab file, identify its owner, inherited permissions, and one effective write-related permission without looking at your notes.**

2. **Create a benign NTFS alternate stream, enumerate it with `dir /R` or PowerShell, and correctly distinguish the normal `$DATA` stream from the named stream without referring to the note.**

3. **Query a Registry key, identify the hive → key → value hierarchy, and explain which component consumes the value before deciding whether it has security relevance.**
