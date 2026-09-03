# Storage Forensics — HDD vs SSD and Deletion vs Wiping

**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → Storage Forensics

# Section 1 — What it is and where it sits

**Storage forensics** is the analysis of persistent storage to determine what data existed, where it was stored, whether it was deleted, and whether remnants may still be recoverable. HDDs and SSDs store data differently, so forensic recovery techniques and the meaning of "deleted" differ substantially between them.

The critical forensic distinction is:

> **Deleting a file usually removes or changes filesystem metadata; it does not necessarily immediately destroy the underlying data.**

```text
Application
    ↓
Filesystem
    ↓
Storage device
    ├── HDD → magnetic platters
    └── SSD → NAND flash + controller
```

Forensic placement:

```text
Hardware
   ↓
Storage devices
   ↓
HDD vs SSD ← THIS TOPIC
   ↓
Filesystems
   ↓
Deleted data / artifacts
   ↓
Disk forensics
   ↓
Incident investigation
```

If you skip this distinction, you can make two opposite mistakes: assume deleted evidence is gone when it is still recoverable, or assume SSD deleted data remains recoverable in the same way as HDD data.

This builds on hardware fundamentals and leads into **filesystems, disk imaging, deleted-file recovery, and forensic analysis**.

# Section 2 — How attackers actually use this

## 2.1 What attackers and forensic analysts are looking for

An attacker interested in removing evidence wants to understand:

* filesystem metadata
* file allocation
* filesystem journals
* deleted directory entries
* free space
* filesystem slack
* temporary files
* application databases
* browser artifacts
* logs
* timestamps
* SSD garbage collection
* TRIM/discard behavior

A forensic analyst asks the opposite question:

> **What evidence remains after the user or attacker believes a file was deleted?**

The investigation usually follows:

```text
Storage device
    ↓
Partition
    ↓
Filesystem
    ↓
Directory metadata
    ↓
File metadata
    ↓
Allocated / deleted state
    ↓
Underlying data
```

## 2.2 HDD physical storage

A traditional HDD stores data magnetically on rotating platters.

Simplified:

```text
        HDD
┌───────────────────────┐
│ Rotating platter      │
│                       │
│  magnetic regions     │
│  representing data   │
│                       │
│        ↓              │
│   read/write head    │
└───────────────────────┘
```

The filesystem does not normally think in terms of individual magnetic regions.

Instead, the operating system sees a block-addressable storage device:

```text
Filesystem
    ↓
logical blocks
    ↓
HDD sectors
    ↓
magnetic storage
```

Historically, forensic recovery from HDDs could be relatively effective because deleting a file normally did not immediately overwrite its underlying sectors.

## 2.3 What "delete" actually means

Consider:

```text
report.txt
```

Before deletion:

```text
Directory metadata
      ↓
report.txt
      ↓
allocated blocks
      ↓
file contents
```

After a normal filesystem deletion, the filesystem may mark the file's storage as available.

Conceptually:

```text
Before:

filename → metadata → allocated blocks → DATA


After delete:

filename → removed/marked deleted
                         ↓
                  blocks become reusable
                         ↓
                     old DATA
```

The old bytes may remain until something else overwrites them.

Therefore:

```text
Delete ≠ immediate physical destruction
```

## 2.4 Dead-end finding vs high-value finding

### Dead-end finding

An analyst sees:

```text
Directory listing:
report.txt → absent
```

That proves only that the normal directory view no longer presents the file.

It does **not** prove that the contents are physically gone.

### High-value finding

A forensic image contains:

```text
Deleted file metadata
       +
recoverable file content
       +
timestamps
       +
application references
```

That is substantially more useful because multiple independent artifacts can reconstruct what happened.

For example:

```text
Deleted document
      ↓
filesystem metadata
      +
browser download record
      +
application recent-file entry
      +
recoverable content
```

The combination provides much stronger evidence than simply finding a filename.

## 2.5 SSDs change the equation

SSDs do not store data magnetically.

They use **NAND flash memory** controlled by an SSD controller.

```text
Operating System
      ↓
logical block address
      ↓
SSD controller
      ↓
flash translation layer
      ↓
NAND flash cells
```

The important forensic difference is that the logical address exposed to the operating system does not necessarily correspond permanently to one physical NAND location.

The SSD controller manages:

* wear leveling
* garbage collection
* bad-block management
* logical-to-physical mapping
* write amplification
* flash block erasure

This makes direct physical recovery considerably more complicated.

## 2.6 Flash translation layer

The **Flash Translation Layer (FTL)** maps logical block addresses to physical NAND locations.

Conceptually:

```text
OS thinks:

LBA 1000
   ↓
"sector 1000"


SSD controller:

LBA 1000
   ↓
FTL mapping
   ↓
NAND physical location X
```

Later, the controller may remap it:

```text
LBA 1000
   ↓
new physical NAND location Y
```

The old physical location may eventually be erased or reused.

This is one reason SSD forensics cannot simply assume:

```text
logical sector = permanent physical location
```

## 2.7 TRIM changes deleted-data behavior

Modern operating systems can send **TRIM/discard** information to SSDs.

Conceptually:

```text
File deleted
    ↓
filesystem marks blocks unused
    ↓
TRIM/discard notification
    ↓
SSD knows those logical blocks are no longer needed
    ↓
controller may reclaim them during garbage collection
```

TRIM is therefore not equivalent to overwriting the file with zeros.

Instead, it tells the SSD:

> **These logical blocks no longer contain data the filesystem needs.**

The SSD controller can subsequently manage those NAND blocks internally.

## 2.8 Why SSD recovery is unpredictable

Suppose:

```text
File A
 ↓
LBA 500
 ↓
NAND location X
```

The file is deleted.

The OS may issue TRIM.

The SSD controller may later:

```text
mark block invalid
      ↓
garbage collection
      ↓
erase NAND block
      ↓
reuse block
```

At that point, traditional undelete techniques may no longer recover the original content.

But timing matters.

```text
Delete
 ↓
TRIM
 ↓
garbage collection
 ↓
physical erase
```

These are not necessarily instantaneous.

The forensic investigator therefore needs to consider the device, firmware, filesystem, workload, and time elapsed.

## 2.9 HDD vs SSD forensic workflow

### HDD

```text
Acquire forensic image
        ↓
Analyze filesystem metadata
        ↓
Find deleted entries
        ↓
Recover unallocated data
        ↓
Validate recovered artifacts
```

### SSD

```text
Acquire forensic image
        ↓
Analyze filesystem metadata
        ↓
Check TRIM/discard evidence
        ↓
Analyze surviving logical data
        ↓
Correlate filesystem/application artifacts
        ↓
Do not assume deleted NAND contents remain recoverable
```

The SSD workflow places greater emphasis on **logical artifacts and metadata correlation**.

## 2.10 What feeds the next phase

Storage analysis can reveal:

```text
Deleted file
    ↓
Filename / metadata
    ↓
Timestamp
    ↓
Application
    ↓
User activity
    ↓
Timeline reconstruction
```

For example:

```text
2026-09-03 10:20
document downloaded

2026-09-03 10:24
document opened

2026-09-03 10:31
document deleted

2026-09-03 10:32
browser history updated
```

The individual artifacts become much more useful when correlated into a timeline.

# Section 3 — Core concepts and terminology

| Term               | Meaning                                                                                                    |
| ------------------ | ---------------------------------------------------------------------------------------------------------- |
| HDD                | Hard Disk Drive using magnetic platters for persistent storage.                                            |
| SSD                | Solid-State Drive using NAND flash memory and a controller.                                                |
| Sector             | Addressable storage unit traditionally associated with disk devices.                                       |
| Block              | Filesystem/storage allocation unit; exact meaning depends on context.                                      |
| LBA                | Logical Block Address used to identify logical storage locations.                                          |
| NAND Flash         | Non-volatile memory technology used by SSDs.                                                               |
| FTL                | Flash Translation Layer mapping logical addresses to physical NAND locations.                              |
| Wear Leveling      | SSD technique distributing writes to reduce uneven flash wear.                                             |
| Garbage Collection | SSD process reclaiming flash blocks whose data is no longer valid.                                         |
| TRIM               | Command/information allowing an SSD to know which logical blocks are no longer needed.                     |
| Discard            | Linux/filesystem terminology for informing storage that blocks are no longer required.                     |
| Filesystem         | Structure that organizes files, directories, metadata, and storage allocation.                             |
| Metadata           | Information describing a file rather than its contents.                                                    |
| Unallocated Space  | Storage currently not assigned to an active filesystem file.                                               |
| Slack Space        | Unused space within an allocated filesystem block/cluster.                                                 |
| Deleted File       | File whose active filesystem reference has been removed or marked deleted.                                 |
| Disk Image         | Forensic copy of storage data used for analysis.                                                           |
| Write Blocker      | Hardware/software mechanism preventing writes to forensic evidence media.                                  |
| Journaling         | Filesystem mechanism recording certain filesystem changes for consistency/recovery.                        |
| File Carving       | Recovering files from raw data based on file structures/signatures rather than normal filesystem metadata. |
| Overwrite          | Writing new data over existing storage locations.                                                          |
| Wiping             | Deliberately destroying or sanitizing stored information according to a defined method.                    |

### Deletion vs wiping

| Action                    | What normally happens                       | Forensic implication                                      |
| ------------------------- | ------------------------------------------- | --------------------------------------------------------- |
| Delete file               | Filesystem removes active reference         | Content may remain                                        |
| Empty recycle/trash       | References are removed                      | Data may still exist                                      |
| Overwrite                 | New data replaces old data                  | Recovery becomes difficult/impossible depending on medium |
| HDD sanitization          | Deliberate media-level sanitization         | Designed to prevent recovery                              |
| SSD discard/TRIM          | Device informed blocks are no longer needed | Controller may later reclaim/erase data                   |
| SSD secure erase/sanitize | Device-level sanitization mechanism         | Intended to remove stored data more thoroughly            |

The crucial distinction is:

```text
Delete:
"Filesystem, stop treating this as an active file."

Wipe:
"Destroy/sanitize the stored information."
```

# Section 4 — Tools and commands

| Tool       | Command                                      | What it finds/shows                         | When to use it                        |
| ---------- | -------------------------------------------- | ------------------------------------------- | ------------------------------------- |
| `lsblk`    | `lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS` | Storage topology                            | Identify disks/partitions             |
| `fdisk`    | `sudo fdisk -l`                              | Partition layout                            | Inspect disk structure                |
| `blkid`    | `sudo blkid`                                 | Filesystem identifiers                      | Identify filesystem types             |
| `smartctl` | `sudo smartctl -a /dev/sdX`                  | Drive health/device information             | Identify HDD/SSD characteristics      |
| `hdparm`   | `sudo hdparm -I /dev/sdX`                    | ATA device capabilities                     | Inspect storage-device features       |
| `nvme`     | `sudo nvme list`                             | NVMe devices                                | Identify NVMe SSDs                    |
| `mount`    | `findmnt`                                    | Mounted filesystems                         | Determine active filesystem locations |
| `df`       | `df -T`                                      | Filesystem usage/type                       | Understand mounted storage            |
| `debugfs`  | `sudo debugfs /dev/sdXN`                     | ext filesystem metadata                     | Low-level ext analysis                |
| `fls`      | `fls -r image.dd`                            | Files/directories including deleted entries | Forensic filesystem analysis          |
| `fsstat`   | `fsstat image.dd`                            | Filesystem metadata                         | Identify filesystem structure         |
| `mmls`     | `mmls image.dd`                              | Partition layout                            | Analyze forensic images               |
| `icat`     | `icat image.dd <inode>`                      | Extract file content from an inode          | Recover identified file data          |
| `strings`  | `strings image.dd`                           | Printable strings                           | Initial raw-data triage               |

Example:

```bash
$ lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS

NAME   SIZE TYPE FSTYPE MOUNTPOINTS
sda     50G disk
├─sda1  512M part vfat   /boot/efi
└─sda2 49.5G part ext4   /
```

This establishes:

```text
sda  → disk
sda1 → EFI partition
sda2 → ext4 filesystem
```

Example:

```bash
$ sudo smartctl -a /dev/sda
Device Model:     ...
Rotation Rate:    7200 rpm
```

A non-zero rotation rate is an indication that the device is a rotating HDD.

For an SSD you may instead see information identifying it as solid-state/NVMe depending on the interface and tool.

Example:

```bash
$ sudo nvme list

Node             Model                    Namespace Usage
/dev/nvme0n1     Example NVMe SSD         1         50.00 GB
```

This identifies an NVMe storage device.

Example:

```bash
$ sudo fdisk -l
Device       Start      End  Sectors  Size Type
/dev/sda1     2048  1050623  1048576  512M EFI System
/dev/sda2  1050624 ...              49.5G Linux filesystem
```

This reveals the partition structure.

Example:

```bash
$ mmls evidence.dd

Slot    Start       End         Length      Description
00:     0000000000  ...
01:     0000002048  ...                     Linux
```

`mmls` identifies partitions inside a forensic image.

Example:

```bash
$ fsstat evidence.dd
FILE SYSTEM INFORMATION
--------------------------------------------
File System Type: Ext4
...
```

This establishes filesystem properties before deeper analysis.

Example:

```bash
$ fls -r evidence.dd

r/r 12-128-1: report.txt
d/d 15-128-1: Documents
-/r * 20-128-1: deleted.txt
```

A deleted entry marked by the tool is an important lead.

It does **not** automatically prove that the complete file content remains recoverable.

Example:

```bash
$ icat evidence.dd 20 > recovered.bin
```

If the inode still contains recoverable content, `icat` extracts it for examination.

# Section 5 — Defender detection

* Monitor filesystem and endpoint telemetry for suspicious mass deletion, log clearing, unusual file destruction, or sudden changes to evidence-bearing directories.
* For forensic acquisition, use a **write-blocked or otherwise controlled acquisition process** so investigators do not accidentally modify evidence.
* Preserve original media/images and calculate hashes so subsequent analysis can be compared against the acquired evidence.
* SSD investigations should consider TRIM/discard behavior because logical deletion and physical NAND persistence are not equivalent.
* Defenders commonly miss application-level artifacts that survive file deletion, such as databases, caches, recent-file records, thumbnails, and logs.
* Skilled attackers may attempt to delete files, clear logs, or overwrite storage, but deletion does not guarantee removal of every correlated artifact.
* On SSDs, defenders should not promise recoverability simply because a file was deleted; TRIM, garbage collection, and controller behavior can eliminate physical remnants.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM using a disposable virtual disk image. Create, delete, and investigate files without touching your host's real storage.

**Objective:** Demonstrate that deleting a file changes filesystem state and investigate whether recoverable artifacts remain in a controlled disk image.

**Steps:**

1. Create a small disposable virtual disk image and format it with a forensic-friendly filesystem such as ext4.
2. Mount it through a controlled loop-device setup and create several ordinary test files.
3. Record the filenames and file contents before deletion.
4. Delete one test file while leaving another file intact.
5. Unmount the filesystem and preserve the resulting image as your forensic evidence copy.
6. Use `mmls` and `fsstat` to identify the partition/filesystem structure.
7. Use `fls` to enumerate active and deleted filesystem entries.
8. Attempt to extract the deleted file using its recovered filesystem metadata.
9. Compare the recovered result with the original test file and document what survived.

**Expected output:**

You should see evidence resembling:

```text
Active file:
normal.txt

Deleted entry:
deleted.txt

Recovery:
deleted.txt → recovered content
```

The important forensic conclusion is:

```text
filesystem no longer presents the file normally
                ≠
underlying evidence necessarily destroyed
```

**Git artifact:**

```text
storage-forensics/
├── README.md
├── evidence/
│   └── hashes.txt
├── analysis/
│   ├── partition-layout.txt
│   ├── filesystem-info.txt
│   └── deleted-files.txt
└── notes/
    └── deletion-vs-wiping.md
```

Do not commit the actual disk image or recovered potentially sensitive content.

```bash
git add storage-forensics/
git commit -m "Add storage deletion and recovery forensics lab"
```

# Section 7 — Common mistakes

1. **Treating delete as wipe** → Deletion usually changes filesystem metadata rather than immediately destroying every underlying byte → Investigate unallocated space and related artifacts.

2. **Assuming HDD and SSD recovery are identical** → SSD controllers, FTL, TRIM, and garbage collection fundamentally change physical-data persistence → Identify the storage technology before deciding what recovery techniques make sense.

3. **Assuming every deleted file is recoverable** → Data may have been overwritten or become unavailable through device-level behavior → Treat recoverability as an empirical finding, not an assumption.

4. **Analyzing the original evidence directly** → Mounting or modifying evidence can change filesystem metadata → Work from a forensic image and preserve the original.

5. **Looking only for the deleted filename** → Valuable evidence can survive in application databases, caches, logs, and metadata → Correlate multiple artifacts.

6. **Assuming TRIM is equivalent to secure erase** → TRIM tells the SSD which logical blocks are no longer needed; device behavior afterward varies → Distinguish filesystem deletion, discard/TRIM, and device-level sanitization.

7. **Trusting timestamps without correlation** → Filesystem timestamps can be changed or interpreted differently across filesystems → Correlate timestamps with application, system, and other forensic artifacts.

# Section 8 — Move-on gate

1. **Create and delete a test file on your forensic disk image, then use `fls` to identify the deleted filesystem entry and correctly interpret what the result proves without looking at your notes.**

2. **Take a disposable ext4 disk image, identify its partition/filesystem structure with `mmls` and `fsstat`, then recover a deleted test file with `icat` and compare its recovered content against the original.**

3. **Given an HDD and an SSD deletion scenario, determine whether you would expect the same recovery workflow, then identify the specific role of FTL, TRIM, and garbage collection in the SSD case without looking at your notes.**
