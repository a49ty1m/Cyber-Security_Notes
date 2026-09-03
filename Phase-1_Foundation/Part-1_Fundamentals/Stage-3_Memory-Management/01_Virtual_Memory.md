# Virtual Memory

**Roadmap:** Part 1 — Fundamentals → Stage 3 — Memory Management → Virtual Memory

# Section 1 — What it is and where it sits

Virtual memory is the OS mechanism that gives each process its own **virtual address space** while the CPU and MMU translate those virtual addresses into physical RAM locations.

A process therefore works with addresses such as `0x7fffffffe000`, not with raw RAM locations.
The OS maintains page tables, while the CPU's **MMU** performs translations.

```text
CPU instruction
      ↓
Virtual address
      ↓
TLB / Page Table
      ↓
Physical address
      ↓
RAM
```

```text
Memory Management
└── Virtual Memory
    ├── Virtual addresses
    ├── Pages
    ├── Page tables
    ├── TLB
    ├── Page faults
    ├── Physical frames
    └── Process isolation
```

If you underestimate this skill, memory corruption, ASLR, information leaks, privilege boundaries, and exploitation techniques become difficult to reason about.

It follows process/memory fundamentals and leads directly into understanding memory corruption and exploitation.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers care about the relationship between:

* Virtual addresses
* Memory permissions
* Loaded libraries
* Stack locations
* Heap locations
* Shared memory
* Page boundaries
* ASLR randomization
* Kernel/user separation
* Memory mappings
* Leaked addresses

The key question is:

> **"What virtual memory exists, where is it, and what can I do with it?"**

## 2.2 A realistic attacker workflow

1. Obtain execution inside a process.
2. Inspect its memory layout.
3. Identify executable regions.
4. Identify writable regions.
5. Locate shared libraries.
6. Look for predictable addresses.
7. Determine whether ASLR changes addresses.
8. Use an information leak if addresses are randomized.
9. Combine the discovered layout with a memory-corruption primitive.
10. Redirect execution toward useful code or data.

The important point is that exploitation rarely depends on knowing "the RAM address."

The attacker normally works with **virtual addresses belonging to the compromised process**.

## 2.3 Why memory permissions matter

A mapping may have permissions such as:

```text
r-xp
```

Meaning:

```text
r = readable
x = executable
p = private
```

A writable executable region is particularly interesting:

```text
rwxp
```

It potentially allows code to be written into a region that can subsequently execute.

Modern systems try to avoid this using protections such as:

```text
NX / DEP
W^X
ASLR
```

## 2.4 ASLR changes the attacker's problem

Without ASLR, an attacker might repeatedly encounter predictable addresses.

With ASLR:

```text
Run 1 → libc = 0x7ffff7...
Run 2 → libc = 0x7f21...
Run 3 → libc = 0x7fa4...
```

The attacker therefore may need an **information disclosure** vulnerability before reliably using an address-dependent exploitation technique.

## 2.5 Page faults are not automatically errors

A page fault means the CPU attempted to access a virtual page whose current mapping cannot immediately satisfy the access.

Possible causes include:

* Page not currently mapped
* Permission violation
* Demand paging
* Copy-on-write
* Access to invalid memory

Therefore:

```text
Page fault ≠ automatically exploitable bug
```

A normal program can generate legitimate page faults.

## 2.6 Process isolation is the security boundary

Suppose:

```text
Process A
Virtual address 0x400000
        ↓
Physical frame 8123
```

and:

```text
Process B
Virtual address 0x400000
        ↓
Physical frame 9917
```

The same virtual address can refer to completely different physical memory.

This is one reason one process normally cannot simply read another process's memory.

## 2.7 High-value finding vs dead end

**Dead-end finding:**

```text
A process has a stack at 0x7ffffff...
```

This alone is weak because ASLR may make the address different every execution.

**High-value finding:**

```text
An information leak reveals a stable pointer
into libc despite ASLR.
```

That can provide an address from which other useful addresses can be calculated.

The second finding changes exploitation possibilities.

## 2.8 Where the results feed next

Virtual-memory knowledge feeds directly into:

```text
Memory layout
      ↓
Memory corruption
      ↓
Address disclosure
      ↓
ASLR analysis
      ↓
Control-flow manipulation
      ↓
Code execution
```

It also matters when analyzing:

```text
Buffer overflows
Use-after-free
Format-string bugs
Heap corruption
ROP
JOP
Sandbox escapes
Kernel exploitation
```

# Section 3 — Core concepts and terminology

| Term                  | Meaning                                                     |
| --------------------- | ----------------------------------------------------------- |
| Virtual address       | Address used by a process                                   |
| Physical address      | Actual address in physical memory                           |
| Virtual address space | Addresses available to a process                            |
| Page                  | Fixed-size block of virtual memory                          |
| Frame                 | Fixed-size block of physical memory                         |
| Page table            | Data structure mapping virtual pages to physical frames     |
| PTE                   | Page-table entry describing one mapping                     |
| MMU                   | CPU hardware translating virtual addresses                  |
| TLB                   | Fast cache of recent virtual-to-physical translations       |
| Page fault            | CPU exception caused by an invalid/unavailable access       |
| Demand paging         | Loading/mapping memory when it is needed                    |
| Copy-on-write         | Sharing pages until a process modifies one                  |
| ASLR                  | Randomizes important memory locations                       |
| NX/DEP                | Prevents execution from non-executable memory               |
| Memory mapping        | Region connecting virtual addresses to an object/file/frame |
| Heap                  | Dynamic process memory                                      |
| Stack                 | Memory used for function calls and local data               |
| Shared library        | Library mapped into a process address space                 |
| Kernel space          | Privileged memory region                                    |
| User space            | Restricted application memory region                        |
| Memory isolation      | Preventing unauthorized memory access between domains       |

### Address translation

A simplified virtual address contains:

```text
Virtual Address
┌──────────────┬────────────┐
│ Page Number  │ Page Offset│
└──────────────┴────────────┘
```

The page number selects a page-table entry.

The offset remains unchanged.

```text
Virtual page 42
      ↓
Page table
      ↓
Physical frame 913
      +
Same page offset
      ↓
Physical address
```

### Common page states

| State          | Meaning                              |
| -------------- | ------------------------------------ |
| Present        | Page currently mapped                |
| Read-only      | Reads allowed, writes rejected       |
| Writable       | Writes allowed                       |
| Executable     | Instruction execution allowed        |
| Non-executable | Instruction execution blocked        |
| Shared         | Mapping can be shared                |
| Private        | Mapping belongs privately to process |

### Translation path

```text
CPU
 ↓
Virtual Address
 ↓
TLB lookup
 ├── Hit → Physical address
 └── Miss
       ↓
   Page-table walk
       ↓
   Valid mapping?
    ├── Yes → TLB update
    └── No → Page fault
```

# Section 4 — Tools and commands

| Tool      | Command              | What it finds/shows               | When to use it              |
| --------- | -------------------- | --------------------------------- | --------------------------- |
| `pmap`    | `pmap -x PID`        | Process memory mappings           | Quick layout inspection     |
| `/proc`   | `cat /proc/PID/maps` | Exact mappings and permissions    | Detailed Linux inspection   |
| `gdb`     | `gdb -q -p PID`      | Debugger-level memory information | Controlled lab analysis     |
| `vmstat`  | `vmstat 1`           | Memory and paging activity        | Observe system behavior     |
| `free`    | `free -h`            | RAM and swap usage                | System-level overview       |
| `getconf` | `getconf PAGESIZE`   | System page size                  | Understand page granularity |

### `pmap`

```bash
pmap -x $$ | head
```

Typical output:

```text
Address           Kbytes     RSS   Dirty Mode  Mapping
00005555...         164     120       0 r-x-- bash
00007fff...         132      48      12 rw--- [ stack ]
```

Interpretation:

```text
r-x → executable mapping
rw- → writable mapping
[stack] → process stack
```

### `/proc/PID/maps`

```bash
cat /proc/$$/maps
```

Typical output:

```text
555555554000-555555575000 r--p ... /usr/bin/bash
555555575000-55555562e000 r-xp ... /usr/bin/bash
7ffff7dd0000-7ffff7fa0000 r-xp ... libc.so.6
7ffffffde000-7ffffffff000 rw-p ... [stack]
```

The important fields are:

```text
start-end permissions offset device inode pathname
```

### `gdb`

```bash
gdb -q -p PID
```

Then:

```text
(gdb) info proc mappings
```

Typical result:

```text
Start Addr   End Addr     Size       Offset  objfile
0x555555...  0x555555...  ...        ...     /bin/program
0x7ffff7...  0x7ffff7...  ...        ...     libc.so.6
```

This lets you correlate debugger-visible addresses with mapped objects.

### `vmstat`

```bash
vmstat 1
```

Example:

```text
procs  memory        swap
r b   swpd free      si so
1 0   0    4200000   0  0
```

`si` and `so` show swap-in and swap-out activity.

### `free`

```bash
free -h
```

Example:

```text
              total   used   free
Mem:           15Gi    7Gi    4Gi
Swap:           4Gi    0Gi    4Gi
```

This is system-level information, not a process's virtual layout.

### `getconf`

```bash
getconf PAGESIZE
```

Example:

```text
4096
```

A `4096` result means the system uses 4 KiB pages in this configuration.

# Section 5 — Defender detection

* Normal virtual-memory inspection is not necessarily malicious; defenders correlate it with process behavior.
* Linux process-memory access can be investigated through audit telemetry, `/proc` access, debugger activity, and EDR behavior.
* Suspicious patterns include unexpected debuggers, memory reads from unrelated processes, executable anonymous mappings, and abnormal permission changes.
* Defenders commonly miss the significance of **memory permission transitions**, especially writable-to-executable behavior.
* EDR products can detect suspicious allocation, protection changes, thread creation, and execution from unusual memory regions.
* Skilled operators reduce obvious artifacts by avoiding unnecessary debugging, minimizing suspicious memory operations, and using existing executable mappings when appropriate.
* Kernel-level memory activity requires stronger telemetry because user-space logs alone may not reveal the full operation.

# Section 6 — Lab task

**Platform:** Kali Linux VM targeting a local Linux test program.

**Objective:** Prove that a process uses virtual addresses and that its memory mappings have different permissions.

**Steps:**

1. Create a small C program that allocates heap memory and sleeps.
2. Compile it with debugging information.
3. Execute the program locally.
4. Record its PID.
5. Inspect its memory mappings.
6. Identify the executable, heap, shared-library, and stack regions.
7. Compare the permissions of those regions.
8. Restart the program and compare addresses.
9. Record whether important addresses changed.
10. Save the findings as an Obsidian-compatible Markdown note.

A suitable C test program:

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(void)
{
    void *memory = malloc(4096);

    printf("PID: %d\n", getpid());
    printf("Heap allocation: %p\n", memory);

    sleep(300);

    free(memory);
    return 0;
}
```

**Expected output:**

```text
PID: 4217
Heap allocation: 0x55556a...
```

The mapping inspection should reveal regions resembling:

```text
... r-xp ... program
... rw-p ... [heap]
... r-xp ... libc.so.6
... rw-p ... [stack]
```

Restarting the program should demonstrate that some addresses can change because of ASLR.

**Git artifact:**

```text
virtual-memory-lab/
├── README.md
├── src/
│   └── memory_demo.c
└── evidence/
    ├── mappings.txt
    └── observations.md
```

Commit with:

```bash
git add virtual-memory-lab/
git commit -m "Add virtual memory mapping lab"
```

# Section 7 — Common mistakes

1. **Mistake → Treating virtual addresses as physical RAM addresses.**
   **Why it matters →** The MMU translates them.
   **Instead →** Always reason through page → frame translation.

2. **Mistake → Assuming one virtual address always points to one physical location.**
   **Why it matters →** Different processes can map the same virtual address differently.
   **Instead →** Analyze the mapping in the specific process.

3. **Mistake → Thinking every page fault means a crash.**
   **Why it matters →** Demand paging and copy-on-write legitimately generate faults.
   **Instead →** Distinguish recoverable faults from protection violations.

4. **Mistake → Ignoring memory permissions.**
   **Why it matters →** `r-x`, `rw-`, and `rwx` have very different security implications.
   **Instead →** Always inspect mapping permissions.

5. **Mistake → Assuming ASLR makes exploitation impossible.**
   **Why it matters →** Information leaks can disclose randomized addresses.
   **Instead →** Treat ASLR as an obstacle that attackers may bypass.

6. **Mistake → Looking only at RAM usage.**
   **Why it matters →** RAM consumption does not explain the process's virtual layout.
   **Instead →** Inspect actual mappings and permissions.

# Section 8 — Move-on gate

1. **Run a process-memory inspection:** inspect a live Linux process and correctly identify its executable, heap, stack, and libc mappings without looking at your notes.

2. **Demonstrate ASLR:** start the same test program multiple times and record changing virtual addresses, then identify which mappings changed.

3. **Interpret permissions:** inspect a process mapping and correctly identify one readable-only, one writable, and one executable region, explaining what operation each permission permits.
