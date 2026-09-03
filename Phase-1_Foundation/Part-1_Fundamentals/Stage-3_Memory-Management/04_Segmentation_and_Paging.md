# Segmentation & Paging

**Roadmap:** Part 1 — Fundamentals → Stage 3 — Memory Management → Segmentation & Paging

# Section 1 — What it is and where it sits

**Segmentation and paging** are two hardware-supported mechanisms used to translate addresses and enforce memory-access rules. Segmentation uses **segment descriptors** to describe logical memory segments, while paging divides virtual memory into fixed-size **pages** and physical memory into **frames**, with page tables mapping one to another.

For modern x86-64 systems, paging is the important mechanism for normal virtual-to-physical translation. Segmentation still exists architecturally, but most traditional segmentation is largely disabled or flattened in 64-bit user-space execution. You still need to understand segment descriptors because they are part of x86 protection architecture and matter when analyzing operating-system internals, compatibility modes, and kernel-level behavior.

```text
CPU
 │
 ├── Logical address
 │      ↓
 │   Segment selector
 │      ↓
 │   Segment descriptor
 │      ↓
 │   Linear address
 │
 └──────────────────────────┐
                            ↓
                     Paging hardware
                            ↓
                       Page tables
                            ↓
                     Physical address
                            ↓
                           RAM
```

```text
Memory Management
└── Stage 3
    ├── Virtual Memory
    ├── Stack
    ├── Heap
    └── Segmentation & Paging
          ├── Segment descriptors
          ├── Page tables
          ├── Address translation
          └── Memory protection
```

If you skip this, you will struggle to understand why a process can use virtual addresses safely, how the CPU determines whether an address is accessible, why kernel/user isolation works, and how low-level memory vulnerabilities interact with hardware protections.

Virtual memory introduced the problem of address translation; segmentation and paging explain the hardware structures that make that translation and protection possible.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

A low-level attacker examining memory translation is interested in:

* Which virtual addresses are mapped
* Which physical frames back those pages
* Page permissions
* User/supervisor permissions
* Read/write permissions
* Execute permissions
* Page-table hierarchy
* Kernel/user address separation
* Large-page mappings
* Shared pages
* Memory-mapped files
* Segment configuration where relevant
* Translation-related CPU state
* Whether a memory-access primitive crosses a page boundary

The practical question is:

> **What memory can this execution context address, and what does the hardware allow it to do with that memory?**

## 2.2 Logical address → linear address

On x86, a logical address conceptually consists of:

```text
Segment selector : Offset
```

The selector identifies a descriptor.

The descriptor provides information such as:

```text
Base
Limit
Access permissions
Privilege information
Type
```

Conceptually:

```text
Segment selector
       ↓
Global/Local Descriptor Table
       ↓
Segment descriptor
       ↓
Segment base + offset
       ↓
Linear address
```

Historically, this allowed an operating system to divide memory into logical regions such as:

```text
Code segment
Data segment
Stack segment
```

Modern 64-bit code generally uses a flat address-space model where segment bases are effectively zero for ordinary code/data segments.

## 2.3 Segment descriptors

A segment selector does not itself contain the complete description of the segment.

Instead:

```text
Selector
   ↓
GDT or LDT
   ↓
Descriptor
   ↓
Base + limit + permissions + privilege
```

The descriptor tells the processor how the segment can be used.

Important descriptor information includes:

* Segment base
* Segment limit
* Segment type
* Present bit
* Descriptor privilege level
* Granularity
* Access permissions

An attacker studying an OS or hypervisor may care about these because incorrect privilege configuration can create unexpected access paths.

## 2.4 Paging performs the important modern translation

After obtaining a linear address, x86-64 paging translates it through multiple page-table levels.

A simplified four-level hierarchy is:

```text
CR3
 ↓
PML4
 ↓
PDPT
 ↓
Page Directory
 ↓
Page Table
 ↓
Physical Frame
```

For a typical 4-level x86-64 configuration with 4 KiB pages:

```text
Virtual address
┌────────┬───────┬───────┬───────┬────────────┐
│ PML4   │ PDPT  │ PD    │ PT    │ Offset     │
└────────┴───────┴───────┴───────┴────────────┘
```

Each index selects an entry at the corresponding level.

The final page-table entry identifies the physical frame.

The page offset remains unchanged.

## 2.5 A concrete translation

Suppose:

```text
Virtual address:

┌──────────────────────────────┐
│ page number │ offset         │
└──────────────────────────────┘
```

The page-table hierarchy determines:

```text
Virtual page 0x12345
       ↓
Physical frame 0x8ABCD
```

If the offset is:

```text
0x678
```

the resulting physical address is conceptually:

```text
0x8ABCD000 + 0x678
= 0x8ABCD678
```

The exact address sizes and canonical-address requirements depend on the CPU configuration, but the fundamental principle is:

```text
virtual page + offset
          ↓
physical frame + same offset
```

## 2.6 Permission checks

A page-table entry contains permission-related information.

Conceptually:

```text
Page
├── Present
├── Read/write
├── User/supervisor
├── Execute-disable
└── Physical frame number
```

Suppose a page is:

```text
User = 0
```

It is supervisor-only.

A user-space process attempting to access it should trigger a protection fault rather than gaining access.

Similarly:

```text
NX = 1
```

means execution is disabled for that page on systems supporting the relevant feature.

This creates an important security boundary:

```text
User process
     ↓
Virtual address
     ↓
Page-table permission check
     ↓
Allowed? ── No ──→ Page fault
     │
    Yes
     ↓
Physical memory access
```

## 2.7 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
The process uses four-level page tables.
```

That is normal on many x86-64 systems and, by itself, does not provide an exploitation path.

**High-value finding:**

```text
A privileged page-table mapping is incorrectly
accessible from an unprivileged execution context.
```

That potentially represents a serious isolation failure because the hardware memory-protection boundary itself may be compromised.

At the application level, an equally important finding can be:

```text
Attacker-controlled data causes access to a page
with unexpected permissions or mapping.
```

That can turn an ordinary memory-corruption bug into a substantially more useful primitive.

## 2.8 Page faults reveal the boundary

If software accesses:

```text
0x400000
```

and that virtual page is mapped with suitable permissions:

```text
Access → succeeds
```

If it accesses an unmapped or prohibited page:

```text
Access
  ↓
Page-table check
  ↓
Failure
  ↓
#PF (page fault)
```

The CPU records information about the fault, and the operating system's page-fault handler determines what happens next.

A fault can therefore mean:

```text
Demand paging
Copy-on-write
Guard page
Invalid address
Permission violation
```

The attacker must distinguish these cases.

## 2.9 TLB changes the performance picture

The CPU does not necessarily walk the entire page-table hierarchy for every memory access.

It uses the **Translation Lookaside Buffer (TLB)** to cache recent translations.

```text
Virtual address
      ↓
     TLB
   ┌──┴──┐
 Hit    Miss
 ↓       ↓
PA    Page-table walk
         ↓
        TLB
         ↓
        PA
```

This matters to attackers researching:

* Side channels
* Microarchitectural attacks
* Memory isolation
* Virtualization
* Translation-related performance behavior

The TLB is therefore not itself a page table. It is a cache of translation information.

## 2.10 Where results feed next

The knowledge feeds into:

```text
Segmentation
     ↓
Logical → linear address

Paging
     ↓
Linear → physical address

Permissions
     ↓
Access control

Memory isolation
     ↓
User/kernel boundary

Memory bugs
     ↓
Corruption / disclosure
     ↓
Advanced exploitation
```

# Section 3 — Core concepts and terminology

| Term               | Meaning                                                                      |
| ------------------ | ---------------------------------------------------------------------------- |
| Segmentation       | Address-management mechanism using logical segments                          |
| Segment            | Logical memory region described by a segment descriptor                      |
| Segment selector   | Value identifying a segment descriptor                                       |
| Segment descriptor | CPU structure describing a segment's properties                              |
| GDT                | Global Descriptor Table containing segment descriptors                       |
| LDT                | Local Descriptor Table containing optional per-task descriptors              |
| Segment base       | Starting address associated with a segment                                   |
| Segment limit      | Maximum permitted range for a segment                                        |
| DPL                | Descriptor Privilege Level controlling privilege relationships               |
| Logical address    | Segment selector plus offset                                                 |
| Linear address     | Address produced after segmentation                                          |
| Paging             | Mapping of virtual/linear pages to physical frames                           |
| Page               | Fixed-size virtual-memory unit                                               |
| Frame              | Fixed-size physical-memory unit                                              |
| Page table         | Translation structure mapping pages to frames                                |
| PTE                | Page Table Entry describing a mapping                                        |
| PML4               | First-level page-table structure in common x86-64 4-level paging             |
| PDPT               | Page-directory-pointer-table level                                           |
| Page Directory     | Intermediate page-table level                                                |
| Page Table         | Final table for ordinary 4 KiB mappings                                      |
| CR3                | x86 control register containing the page-table root location                 |
| TLB                | CPU cache of recent address translations                                     |
| Page fault         | CPU exception caused by failed page access/translation                       |
| User page          | Page accessible from unprivileged execution                                  |
| Supervisor page    | Page restricted to privileged execution                                      |
| NX/XD              | Execute-disable page protection                                              |
| Present bit        | Indicates whether a page-table mapping is present                            |
| Large page         | Page larger than the normal base page size                                   |
| Canonical address  | Valid x86-64 virtual address satisfying architecture-defined upper-bit rules |

### Segmentation vs paging

| Property                   | Segmentation                | Paging                           |
| -------------------------- | --------------------------- | -------------------------------- |
| Basic unit                 | Segment                     | Page                             |
| Physical backing           | Variable logical region     | Fixed-size frame                 |
| Translation                | Selector + offset           | Page-table hierarchy             |
| Main modern x86-64 role    | Limited                     | Fundamental                      |
| Protection                 | Type, privilege, limits     | Page permissions                 |
| Fragmentation concern      | External/variable regions   | Page-level internal waste        |
| Typical security relevance | Privilege/legacy mechanisms | Isolation and memory permissions |

### Common x86-64 translation model

```text
Logical address
      ↓
Segmentation
      ↓
Linear address
      ↓
TLB lookup
      │
      ├── Hit
      │    ↓
      │ Physical address
      │
      └── Miss
           ↓
       Page-table walk
           ↓
       Page permissions
           ↓
       Physical frame
           ↓
       Physical address
```

For ordinary 64-bit Linux user-space code, the segmentation stage is largely flattened:

```text
segment base ≈ 0
        ↓
linear address ≈ effective virtual address
        ↓
paging
```

This is why paging deserves substantially more practical attention for modern Linux exploitation.

# Section 4 — Tools and commands

| Tool      | Command                      | What it finds/shows                    | When to use it                     |
| --------- | ---------------------------- | -------------------------------------- | ---------------------------------- |
| `gdb`     | `info registers cr3`         | Current page-table root register       | Low-level paging analysis          |
| `gdb`     | `info registers cs ds ss`    | Segment selectors                      | Inspect segmentation state         |
| `gdb`     | `info proc mappings`         | Process virtual mappings               | Relate addresses to mapped objects |
| `pagemap` | `sudo cat /proc/PID/pagemap` | Linux page-mapping information         | Advanced page/frame analysis       |
| `vmmap`   | `vmmap PID`                  | Virtual-memory regions where available | macOS-focused process analysis     |
| `pmap`    | `pmap PID`                   | Process memory mappings                | Quick Linux memory overview        |
| `readelf` | `readelf -l ./program`       | ELF memory segments and permissions    | Understand executable mappings     |
| `gdb`     | `x/16gx ADDRESS`             | Memory at a virtual address            | Inspect mapped memory              |

### `gdb` — segment registers

```bash
gdb -q ./program
```

Then:

```text
(gdb) info registers cs ds ss
```

Typical output resembles:

```text
cs    0x33
ds    0x0
ss    0x2b
```

The values are **segment selectors**, not segment base addresses.

They identify descriptor information used by the processor.

### `gdb` — CR3

```text
(gdb) info registers cr3
```

Typical output:

```text
cr3            0x0000000012345000
```

`CR3` identifies the page-table root used by the current address space on x86.

Access to this information is particularly relevant when studying kernel-level memory management rather than ordinary user-space debugging.

### `gdb` — process mappings

```text
(gdb) info proc mappings
```

Example:

```text
Start Addr   End Addr       Size      Offset  Perms
0x555555...  0x555555...    0x21000   0x0     r-xp
0x555555...  0x555555...    0x3000    0x21000 rw-p
0x7ffff7...  0x7ffff7...    ...       ...    r-xp
```

This shows which virtual regions exist and their permissions.

### `/proc/PID/pagemap`

```bash
sudo cat /proc/PID/pagemap
```

The file exposes kernel-provided page-mapping information subject to modern privilege and kernel restrictions.

It is useful when studying how virtual pages relate to physical frames, but raw output is not immediately human-readable.

### `pmap`

```bash
pmap PID
```

Example:

```text
555555554000   132K r-x-- program
555555575000    16K rw--- program
7ffff7dd0000  1800K r-x-- libc.so.6
7ffffffde000   132K rw--- [ stack ]
```

The important observation is that the process sees a **virtual layout** rather than directly exposing physical RAM.

### `readelf`

```bash
readelf -l ./program
```

Example:

```text
Type   Offset   VirtAddr        PhysAddr
LOAD   0x0000   0x00000000...  0x00000000...
LOAD   0x2000   0x00000000...  0x00000000...
```

Look at the program's `LOAD` segments and their permissions.

This connects the ELF file's logical segments with the memory regions that the loader maps into the process.

### `gdb` memory inspection

```text
(gdb) x/16gx 0x555555555000
```

Example:

```text
0x555555555000: 0x...
0x555555555008: 0x...
```

If the address is mapped and readable, GDB can display its contents. If not, the debugger will report an invalid/unreadable memory access.

# Section 5 — Defender detection

* Normal paging activity is fundamental to every modern OS, so defenders do not treat ordinary page-table translation as suspicious by itself.
* Suspicious behavior includes unexpected privileged memory access, abnormal page-permission changes, kernel memory exposure, or attempts to bypass user/kernel isolation.
* EDR and kernel telemetry can detect suspicious memory mappings, executable-memory transitions, and abnormal process behavior around memory access.
* Kernel integrity mechanisms can detect or prevent unauthorized modification of page tables or privileged memory structures.
* Defenders commonly miss the distinction between an application memory bug and a genuine page-table/privilege-boundary violation.
* Skilled operators avoid unnecessary privileged memory operations because kernel-level memory manipulation is considerably more detectable and dangerous than ordinary user-space memory access.
* Forensic analysis can correlate virtual mappings, process state, kernel events, and memory snapshots to determine whether a suspicious access crossed a privilege boundary.

# Section 6 — Lab task

**Platform:** Kali Linux VM with a local x86-64 Linux binary.

**Objective:** Demonstrate the difference between an ELF memory segment and the resulting process virtual-memory mapping, then relate the mapping to paging permissions.

**Steps:**

1. Create a small C program containing code, initialized data, and dynamically allocated heap data.
2. Compile it as an x86-64 ELF executable with debugging information.
3. Inspect its ELF program headers.
4. Run the binary and obtain its PID.
5. Inspect the process's virtual-memory mappings.
6. Identify an executable `r-x` region and writable `rw-` region.
7. Compare those mappings with the ELF `LOAD` segments.
8. Open the program in GDB and inspect its segment registers.
9. Inspect one mapped virtual address.
10. Record how the ELF segment permissions become process memory permissions.

Example:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int global_value = 42;

int main(void)
{
    char *heap = malloc(4096);

    printf("PID: %d\n", getpid());
    printf("global: %p\n", (void *)&global_value);
    printf("heap:   %p\n", (void *)heap);

    sleep(300);

    free(heap);
    return 0;
}
```

Compile:

```bash
gcc -g -O0 segment_demo.c -o segment_demo
```

Inspect:

```bash
readelf -l ./segment_demo
```

Run it and inspect:

```bash
pmap PID
```

Then inspect the process in GDB:

```bash
gdb -q -p PID
```

**Expected output:**

You should be able to correlate:

```text
ELF LOAD segment
       ↓
process virtual mapping
       ↓
page permissions
```

For example:

```text
ELF executable segment
        ↓
r-xp process mapping

ELF writable segment
        ↓
rw-p process mapping
```

The exact addresses will vary because of ASLR and the executable's configuration.

**Git artifact:**

```text
segmentation-paging-lab/
├── README.md
├── src/
│   └── segment_demo.c
└── evidence/
    ├── elf-program-headers.txt
    ├── process-mappings.txt
    └── observations.md
```

Commit:

```bash
git add segmentation-paging-lab/
git commit -m "Add segmentation and paging analysis lab"
```

# Section 7 — Common mistakes

1. **Mistake → Treating segmentation and paging as interchangeable.**
   **Why it matters →** They solve different parts of address translation and protection.
   **Instead →** Remember: segmentation produces a linear address; paging translates it toward physical memory.

2. **Mistake → Assuming traditional segmentation is the main mechanism in modern x86-64 Linux.**
   **Why it matters →** Ordinary 64-bit Linux uses a largely flat segmentation model and relies heavily on paging.
   **Instead →** Learn segmentation for architecture and privilege understanding, but prioritize paging operationally.

3. **Mistake → Thinking a segment selector is the physical address of a segment.**
   **Why it matters →** The selector identifies a descriptor; the descriptor contains the relevant segment information.
   **Instead →** Follow selector → descriptor → linear address.

4. **Mistake → Confusing a page with a physical frame.**
   **Why it matters →** A page belongs to virtual memory; a frame belongs to physical memory.
   **Instead →** Think page → page-table mapping → frame.

5. **Mistake → Assuming every virtual address has a physical frame behind it.**
   **Why it matters →** Virtual address spaces contain unmapped regions, guard pages, and special mappings.
   **Instead →** Check whether the page is mapped and whether the requested access is permitted.

6. **Mistake → Assuming a page-table entry only contains a physical address.**
   **Why it matters →** Permission and state bits are essential to memory protection.
   **Instead →** Analyze both the frame address and access-control bits.

7. **Mistake → Confusing the TLB with the page table.**
   **Why it matters →** The TLB caches translations; the page tables are the authoritative translation structures used for page walks.
   **Instead →** Think `TLB = translation cache`, `page table = translation structure`.

# Section 8 — Move-on gate

1. **Trace an address:** take a virtual address from your lab process and explain the complete path from process virtual address → page → page-table translation → physical frame without looking at your notes.

2. **Analyze segmentation state:** use GDB on an x86-64 binary, inspect `CS`, `DS`, and `SS`, and correctly identify them as segment selectors rather than physical addresses.

3. **Correlate executable structure with memory:** run `readelf -l` and `pmap` against the same binary and correctly match at least one ELF `LOAD` segment to its process mapping and permissions.

