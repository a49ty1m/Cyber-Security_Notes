# Linux File Systems

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → File Systems (Linux)

# Section 1 — What it is and where it sits

Linux presents storage through a hierarchical filesystem rooted at `/`. Files and directories are represented through filesystem metadata, with the **inode** being the core metadata structure used by traditional Unix-like filesystems to describe an object such as a regular file, directory, symbolic link, or device entry. The filename is a directory entry that points to an inode; the inode points toward the data and metadata associated with the object.

The famous **“everything is a file”** philosophy means Linux exposes many resources through file-like interfaces and file descriptors. It does **not** mean every kernel object is literally stored as an ordinary disk file. Devices, pipes, sockets, `/proc`, and `/sys` expose interfaces that programs can interact with using familiar file operations such as `open()`, `read()`, `write()`, and `close()`.

```text
Linux File-System Model
│
├── /
│   ├── /bin
│   ├── /etc
│   ├── /home
│   ├── /tmp
│   ├── /usr
│   ├── /var
│   ├── /dev
│   ├── /proc
│   └── /sys
│
├── Directory entry
│      ↓
│   inode
│      ↓
│   filesystem data blocks
│
└── Process
       ↓
   file descriptor
       ↓
   file / device / pipe / socket / pseudo-filesystem
```

If you skip this, you will misunderstand permissions, deleted-file forensics, hard links, disk usage, `/proc` investigation, device access, log locations, and a large part of Linux post-exploitation.

This builds on processes and system calls: processes access filesystem objects through file descriptors and syscalls, and the next step is understanding permissions, mounts, links, and filesystem abuse during privilege escalation.

# Section 2 — How attackers actually use this

## 2.1 What attackers map first

After obtaining a Linux foothold, an attacker wants to understand the filesystem layout quickly:

```text
Filesystem
   ↓
Where is the application?
Where are credentials/configuration files?
Where are logs?
Where are writable directories?
Where are executables?
Where are mounted filesystems?
Where are unusual devices?
```

They commonly investigate:

- `/etc`
- `/home`
- `/root`
- `/tmp`
- `/var`
- `/usr`
- `/opt`
- `/srv`
- `/dev`
- `/proc`
- `/sys`
- Mounted filesystems
- World-writable directories
- SUID/SGID binaries
- Sensitive configuration files
- Application data
- SSH material
- Service files

The objective is not "look at every file."

It is:

```text
Find the filesystem objects that
change the attacker's next decision.
```

## 2.2 Root directory structure

The starting point is `/`.

A simplified attacker-oriented view:

```text
/
├── bin      → essential user commands
├── boot     → boot-related files
├── dev      → device interfaces
├── etc      → system/application configuration
├── home     → normal users' data
├── lib*     → essential libraries
├── media    → removable-media mount points
├── mnt      → temporary mount location
├── opt      → optional third-party software
├── proc     → kernel/process information interface
├── root     → root user's home directory
├── run      → runtime state
├── sbin     → administration/system programs
├── srv      → service data
├── sys      → kernel/device information interface
├── tmp      → temporary files
├── usr      → most user-space software
└── var      → variable data such as logs and queues
```

Attackers prioritize directories based on what they are trying to achieve.

For example:

```text
Need credentials?
    ↓
/etc
/home
application configs

Need persistence?
    ↓
cron
systemd
shell startup locations
service configurations

Need privilege escalation?
    ↓
SUID/SGID
writable service paths
sudo-related configuration
capabilities
```

## 2.3 Why `/etc` is high value

`/etc` contains configuration rather than user documents.

An attacker may find:

```text
/etc/passwd
/etc/group
/etc/ssh/
/etc/systemd/
/etc/cron.*
/etc/*application*/
```

Not every file contains secrets.

The high-value cases are configuration files where administrators have embedded:

```text
passwords
API keys
database credentials
tokens
connection strings
service accounts
```

A generic application config might conceptually contain:

```text
database_host=127.0.0.1
database_user=app
database_password=...
```

That can turn a filesystem discovery step into access to another service.

## 2.4 `/home` and `/root`

User home directories can contain:

```text
SSH keys
shell history
configuration
source code
scripts
credentials
tokens
```

The attacker is not interested merely because "home contains files."

They care about objects that expose another authentication or execution path.

For example:

```text
Low-privileged foothold
       ↓
Readable application directory
       ↓
Credential/configuration discovery
       ↓
Access to another account/service
       ↓
Privilege expansion
```

## 2.5 `/tmp` and writable locations

Writable directories are interesting because they may become execution or substitution opportunities.

Attackers ask:

```text
Who can write here?
Who executes from here?
Who reads from here?
Can a privileged process trust contents from here?
```

A dangerous pattern is:

```text
Privileged service
      ↓
reads predictable file in writable directory
      ↓
unprivileged user can modify file
```

The vulnerability is not simply "directory is writable."

The valuable condition is:

```text
untrusted writer
       +
privileged consumer
       +
unsafe trust relationship
```

## 2.6 "Everything is a file" from an attacker perspective

Linux represents many resources with file-like interfaces.

Examples:

```text
/dev/null
/dev/random
/proc/<PID>/status
/sys/class/net/*
named pipe
Unix socket
regular file
```

Applications often interact with these through file descriptors.

Conceptually:

```text
open()
  ↓
file descriptor
  ↓
read()/write()/ioctl()/close()
```

That gives attackers a common interface for interacting with many different kernel-managed resources.

## 2.7 File descriptors

A **file descriptor (FD)** is a process-local integer referring to an open file-like object.

Typical values:

```text
0 → stdin
1 → stdout
2 → stderr
```

A process might have:

```text
FD 0 → terminal
FD 1 → terminal
FD 2 → terminal
FD 3 → log file
FD 4 → network socket
FD 5 → pipe
```

This is extremely useful during incident response.

A process can have no obvious interesting filename in its command line while still holding:

```text
open credential file
network socket
deleted executable
Unix socket
pipe
```

## 2.8 Inodes

The attacker must distinguish:

```text
filename
```

from:

```text
inode
```

A filename is a directory entry.

The inode contains filesystem metadata such as:

- File type
- Permissions
- Owner
- Group
- Size
- Timestamps
- Link count
- References to file data

A simplified relationship is:

```text
Directory
│
├── "report.txt" ──────┐
└── "report-copy.txt" ─┤
                       ↓
                    inode 84521
                       ↓
                    file data
```

Two filenames can point to the same inode.

That is the basis of **hard links**.

## 2.9 Hard-link pivot

Suppose:

```text
/etc/original
```

and:

```text
/tmp/alias
```

refer to the same inode.

Changing one changes the same underlying file object.

Conceptually:

```text
filename A ──┐
             ├── inode ── data
filename B ──┘
```

Attackers and defenders therefore need to reason about the underlying inode, not only the filename.

This is especially useful in forensic analysis where multiple names may refer to one object.

## 2.10 Deleted does not necessarily mean gone

This is one of the most important inode concepts for forensics.

Suppose a process opens a file:

```text
secret.txt
   ↓
inode
   ↓
FD 3
```

Then the filename is removed:

```text
rm secret.txt
```

The directory entry disappears, but if a process still holds the file open, the underlying data can remain allocated until the last reference is released.

Conceptually:

```text
Before deletion:

filename
   ↓
inode
   ↓
data
↑
FD 3

After unlink:

filename X

inode
   ↓
data
↑
FD 3
```

This is why an investigator can sometimes recover information from an open-but-deleted file through `/proc/<PID>/fd`.

## 2.11 High-value inode finding

**Dead-end finding:**

```text
/tmp/backup.txt
```

exists.

That tells you almost nothing.

**High-value finding:**

```text
A privileged process
    ↓
has FD 7
    ↓
/tmp/backup.txt (deleted)
```

That can indicate:

- Sensitive data still resident
- A deleted log/file being actively used
- A process retaining data after unlink
- Potential forensic evidence

The inode/FD relationship becomes more useful than the pathname alone.

## 2.12 `/proc` as an attack and analysis interface

`/proc` is a pseudo-filesystem exposing process and kernel information.

For example:

```text
/proc/<PID>/status
/proc/<PID>/cmdline
/proc/<PID>/fd/
/proc/<PID>/maps
/proc/<PID>/environ
```

An attacker can use this to understand:

```text
What process is this?
What identity does it use?
What files does it have open?
What memory is mapped?
What environment variables exist?
```

A defender can use the same interface during incident response.

## 2.13 `/dev` and device access

`/dev` contains device nodes representing interfaces to kernel/device functionality.

Examples:

```text
/dev/null
/dev/zero
/dev/random
/dev/tty
```

Some device interfaces are extremely security-sensitive.

A privileged or improperly exposed device can provide a route to operations far beyond ordinary file access.

Therefore:

```text
Device node
    ↓
Kernel driver
    ↓
Hardware/kernel functionality
```

is an important attack-surface relationship.

## 2.14 Where the results feed next

Filesystem findings can unlock:

```text
Filesystem enumeration
        ↓
Credential/config discovery
        ↓
Privilege escalation
        ↓
Persistence
        ↓
Lateral movement
```

or:

```text
/proc + /dev + inodes + FDs
        ↓
Runtime analysis
        ↓
Malware/process investigation
        ↓
Forensics
```

# Section 3 — Core concepts and terminology

| Term | Meaning |
|---|---|
| **Filesystem** | Structure and rules used to organize, store, and retrieve data. |
| **Root directory `/`** | Top of the Linux directory hierarchy. |
| **Directory** | Filesystem object containing mappings from names to filesystem objects/inodes. |
| **Directory entry (dentry)** | Kernel representation of a directory/name relationship. |
| **Inode** | Filesystem metadata structure describing an object and its data references. |
| **Filename** | Name used to refer to an object through a directory entry. |
| **File descriptor** | Integer handle a process uses to access an open file-like object. |
| **Hard link** | Additional directory entry referencing the same inode. |
| **Symbolic link** | File containing a pathname reference to another object. |
| **Link count** | Number of directory entries referencing an inode. |
| **Mount** | Attaching a filesystem to a directory in the existing hierarchy. |
| **Mount point** | Directory where another filesystem becomes accessible. |
| **Pseudo-filesystem** | Kernel-provided filesystem interface whose contents are not ordinary disk files. |
| **`/proc`** | Pseudo-filesystem exposing process and kernel state. |
| **`/sys`** | Sysfs interface exposing kernel/device/system information. |
| **`/dev`** | Device-node hierarchy for device and pseudo-device interfaces. |
| **Device node** | Filesystem entry representing a device interface. |
| **Unlink** | Removing a directory entry from a pathname. |
| **Open file** | File-like object currently referenced through a process's file descriptor. |
| **SUID** | Permission bit causing an executable to run with the file owner's effective user ID under supported conditions. |
| **SGID** | Permission bit affecting execution/group behavior or directories. |
| **Permissions** | Mode bits controlling read, write, and execute access. |
| **Owner** | User associated with a filesystem object. |
| **Group** | Group associated with a filesystem object. |
| **Filesystem metadata** | Information such as owner, permissions, size, timestamps, and link count. |
| **Filesystem block** | Storage unit used by the filesystem for data/metadata allocation. |
| **File offset** | Current read/write position associated with an open file description. |

### Root-directory map

| Path | Attacker-relevant purpose |
|---|---|
| `/etc` | Configuration, service settings, authentication-related files |
| `/home` | User data, SSH material, histories, application configuration |
| `/root` | Root user's home data |
| `/tmp` | Temporary writable area; potential trust-boundary issues |
| `/var` | Logs, caches, queues, application/runtime data |
| `/usr` | Major user-space programs and libraries |
| `/opt` | Optional/third-party software |
| `/srv` | Service data |
| `/proc` | Process/kernel runtime information |
| `/sys` | Kernel/device information and controls |
| `/dev` | Device interfaces |
| `/boot` | Bootloader/kernel-related files |

### Inode vs filename

```text
Filename
   ↓
Directory entry
   ↓
Inode
├── owner
├── group
├── permissions
├── size
├── timestamps
├── link count
└── data references
```

The important forensic principle is:

```text
Names can change.
The inode relationship explains the underlying object.
```

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|---|---|---|---|
| `ls` | `ls -la /` | Root directory and permissions | Initial filesystem map |
| `find` | `find / -xdev -type f -perm -4000 2>/dev/null` | SUID files | Privilege-escalation enumeration |
| `find` | `find / -xdev -type d -writable 2>/dev/null` | Writable directories | Identify writable attack surfaces |
| `stat` | `stat <file>` | Inode and metadata | Detailed object analysis |
| `ls` | `ls -li <file>` | Inode number + filename | Link/inode investigation |
| `file` | `file <file>` | File type | Identify suspicious/unknown files |
| `df` | `df -hT` | Filesystems, mount types, usage | Understand storage layout |
| `findmnt` | `findmnt` | Mounted filesystems | Mount analysis |
| `mount` | `mount` | Current mounts | Inspect mounted resources |
| `du` | `du -sh <path>` | Disk usage | Locate large data areas |
| `lsof` | `lsof -p <PID>` | Files/FDs open by a process | Process/file relationship analysis |
| `lsof` | `lsof +L1` | Open files with removed directory links | Deleted-file investigation |
| `readlink` | `readlink /proc/<PID>/fd/<FD>` | What an FD points to | Inspect open/deleted files |
| `ps` | `ps aux` | Running processes | Select processes for filesystem analysis |
| `ls` | `ls -la /proc/<PID>/fd` | Process FD directory | Runtime file relationships |
| `cat` | `cat /proc/<PID>/status` | Process metadata | Process identity/state |
| `cat` | `cat /proc/<PID>/mountinfo` | Process mount view | Namespace/mount analysis |
| `namei` | `namei -l /path/to/file` | Path components and permissions | Find permission problems along a path |
| `getfacl` | `getfacl <file>` | POSIX ACLs | Analyze access beyond mode bits |
| `realpath` | `realpath <path>` | Resolved path | Analyze links/path resolution |

### `ls -li`

```text
$ ls -li report.txt report-copy.txt
84521 -rw-r--r-- 2 user user 1200 ... report.txt
84521 -rw-r--r-- 2 user user 1200 ... report-copy.txt
```

Both names have inode:

```text
84521
```

Therefore they are hard links to the same underlying inode.

### `stat`

```text
$ stat report.txt
  File: report.txt
  Size: 1200
  Inode: 84521
  Links: 2
  Access: (0644/-rw-r--r--)
  Uid: 1000
  Gid: 1000
```

Important fields include:

```text
Inode
Links
Permissions
UID
GID
Size
Timestamps
```

### `find` — SUID files

```text
$ find / -xdev -type f -perm -4000 2>/dev/null
/usr/bin/passwd
/usr/bin/su
...
```

A SUID executable deserves investigation because execution can occur with the file owner's effective UID.

The presence of SUID is **not automatically a vulnerability**.

### `find` — writable directories

```text
$ find / -xdev -type d -writable 2>/dev/null
/tmp
...
```

The useful follow-up question is:

```text
Who writes here?
Who trusts files here?
What process consumes them?
```

### `df -hT`

```text
$ df -hT
Filesystem     Type  Size  Used  Avail  Use%  Mounted on
/dev/sda1      ext4   40G   ...    ...    ...  /
```

This reveals the filesystem type and mount layout.

### `findmnt`

```text
$ findmnt
TARGET SOURCE    FSTYPE OPTIONS
/      /dev/sda1 ext4   rw,...
/proc  proc      proc   rw,...
```

This makes the relationship between mount points and filesystem types visible.

### `lsof -p`

```text
$ lsof -p 4200
COMMAND PID USER  FD  TYPE DEVICE SIZE/OFF NODE NAME
server  4200 user cwd DIR  ...          ... /opt/server
server  4200 user  3u REG  ...       84521 /tmp/output.log
server  4200 user  4u IPv4 ...            TCP ...
```

A single process may hold:

```text
directories
regular files
pipes
sockets
devices
```

through its file descriptors.

### `lsof +L1`

```text
$ lsof +L1
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NLINK NODE NAME
server 4200 user 7w REG ...  4096     0 84521 /tmp/output.log (deleted)
```

This is a classic forensic clue.

The pathname is gone, but the process still has the inode open.

### `/proc/<PID>/fd`

```text
$ ls -l /proc/4200/fd
0 -> /dev/pts/0
1 -> /dev/pts/0
2 -> /dev/pts/0
3 -> /tmp/output.log
4 -> socket:[12345]
```

This shows what the process's file descriptors currently reference.

### `readlink`

```text
$ readlink /proc/4200/fd/3
/tmp/output.log
```

For a deleted file, it may show:

```text
/tmp/output.log (deleted)
```

The process can still hold the underlying open file.

### `namei`

```text
$ namei -l /opt/app/config/settings.conf
drwxr-xr-x root root /
drwxr-xr-x root root opt
drwxr-xr-x root root app
-rw-r----- app  app  config
```

This is useful because a file can appear restrictive while one of its parent directories allows unwanted traversal or modification.

### `getfacl`

```text
$ getfacl settings.conf
# file: settings.conf
user::rw-
group::r--
other::---
```

ACLs can add permissions that are not obvious from a simple `ls -l`.

# Section 5 — Defender detection

- **Filesystem auditing:** Linux audit or EDR telemetry can record sensitive file access, execution, permission changes, and modifications to important paths.
- **SUID/permission monitoring:** Unexpected creation or modification of SUID/SGID binaries, service files, startup scripts, or privileged configuration files is high-value telemetry.
- **Open-but-deleted files:** Processes retaining deleted sensitive files can be investigated through FD/inode relationships.
- **`/proc` and `/sys` access:** Unusual access to sensitive process or kernel interfaces can provide behavioral context, especially from untrusted applications.
- **Mount monitoring:** Unexpected mounts, writable host filesystem exposure, or unusual device access are important in container/server environments.
- **What defenders miss:** Searching only filenames can miss hard links, deleted-but-open files, unusual mount points, or files whose contents are accessible through pseudo-filesystems.
- **Reducing footprint:** Skilled operators may use existing writable locations, legitimate binaries, or open file descriptors rather than creating obviously suspicious files; defenders therefore need metadata, process, and filesystem-behavior correlation.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM.

**Objective:** Build a controlled filesystem investigation demonstrating the relationship between filenames, inodes, hard links, file descriptors, and deleted-but-open files.

**Steps:**

1. Create a temporary lab directory under `/tmp`.
2. Create one test file and write harmless sample data into it.
3. Display its inode number and detailed metadata.
4. Create a hard link to the same file and verify that both names report the same inode.
5. Start a process that keeps the test file open.
6. Identify the process and inspect its open file descriptors.
7. Remove one pathname while keeping the file open through the process.
8. Use `lsof` and `/proc/<PID>/fd` to identify the deleted-but-open file.
9. Compare the filename state, inode state, and process FD state.
10. Record the results as a forensic diagram and clean up the lab files/process.

**Expected output:**

```text
report.txt ────────┐
                   ├── inode 84521 ── data
report-link.txt ───┘

after unlink:

report.txt       X
report-link.txt ───── inode 84521 ── data
                                ↑
                              FD 3
```

You should demonstrate that a pathname and the underlying inode are not the same thing, and that an open file can remain accessible through a process after a directory entry is removed.

**Git artifact:**

```text
linux-filesystem/
├── README.md
├── evidence/
│   ├── inode.txt
│   ├── stat.txt
│   ├── hard-link.txt
│   ├── lsof.txt
│   └── proc-fd.txt
└── notes.md
```

```bash
git add linux-filesystem/
git commit -m "Document Linux filesystem inodes and file descriptors"
```

# Section 7 — Common mistakes

1. **Mistake:** Thinking a filename is the file itself.  
   **Why it matters:** Names are directory entries pointing to filesystem objects/inodes.  
   **Do instead:** Track filename → inode → data.

2. **Mistake:** Assuming `rm` immediately destroys all file data.  
   **Why it matters:** Removing a directory entry is not the same as immediately erasing storage, and open references can keep the inode/data alive.  
   **Do instead:** Check link count and open file descriptors.

3. **Mistake:** Treating `/proc` as ordinary disk storage.  
   **Why it matters:** `/proc` is a kernel-backed pseudo-filesystem.  
   **Do instead:** Treat it as a live interface to process/kernel state.

4. **Mistake:** Thinking "everything is a file" literally.  
   **Why it matters:** Many resources are file-like interfaces, not ordinary disk files.  
   **Do instead:** Interpret the phrase as a unified file-descriptor-oriented interface model.

5. **Mistake:** Searching only `/home` for sensitive information.  
   **Why it matters:** Credentials and execution-relevant configuration frequently exist under `/etc`, `/var`, application directories, and service-specific paths.  
   **Do instead:** Map the entire root hierarchy first, then prioritize.

6. **Mistake:** Treating every SUID binary as exploitable.  
   **Why it matters:** SUID is a privilege property, not proof of a vulnerability.  
   **Do instead:** Identify what the binary actually does and whether unsafe input/control exists.

7. **Mistake:** Ignoring mount points and filesystem types.  
   **Why it matters:** Permissions, device exposure, namespace isolation, and mount options can materially change attackability.  
   **Do instead:** Inspect mounts with `findmnt` and reason about the filesystem boundary.

# Section 8 — Move-on gate

1. **Create two hard links to one file, inspect their inode numbers with `ls -li`, and correctly explain why modifying either pathname modifies the same underlying object without looking at your notes.**

2. **Keep a lab file open, unlink its pathname, and use `lsof` plus `/proc/<PID>/fd` to identify the remaining open inode without referring to the note.**

3. **On a Kali VM, map `/`, `/etc`, `/var`, `/proc`, `/sys`, and `/dev`, then identify one attacker-relevant object in each and explain whether it is ordinary filesystem data, a pseudo-filesystem interface, or a device interface without looking at your notes.**