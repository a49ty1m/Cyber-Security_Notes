# Memory Protection

**Roadmap:** Part 1 — Fundamentals → Stage 3 — Memory Management → Memory Protection

# Section 1 — What it is and where it sits

**Memory protection** is the hardware-enforced mechanism that controls what code can do with mapped memory. At the page level, the CPU uses permission information such as **Read/Write/Execute (R/W/X)** and **User/Supervisor (U/S)** to decide whether a memory access is permitted.

This creates two critical boundaries: **what operation is allowed** on a page and **which privilege level is allowed** to access it.

```text
Memory Management
└── Stage 3
    ├── Virtual Memory
    ├── Stack
    ├── Heap
    ├── Segmentation & Paging
    └── Memory Protection
         ├── R/W/X permissions
         ├── User / Kernel separation
         ├── NX / DEP
         └── Page faults
```

```text
Process
   ↓
Virtual address
   ↓
Page-table lookup
   ↓
Permission check
 ┌─┴──────────────────┐
Allowed              Denied
  ↓                    ↓
Access             Page fault
```

If you skip this, you will misunderstand why a writable buffer cannot normally be executed, why user programs cannot simply read kernel memory, and why memory-corruption vulnerabilities do not automatically produce code execution.

Paging taught you **how addresses are translated**; memory protection now teaches you **how the CPU decides whether the requested access is legal**.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers examine:

* Readable pages
* Writable pages
* Executable pages
* Writable + executable mappings
* Non-executable stack/heap
* User-accessible versus supervisor-only pages
* Memory-permission transitions
* Shared mappings
* Guard pages
* Memory-mapped executable files
* Information leaks that reveal protected memory addresses
* Vulnerabilities that cross privilege boundaries

The key question is:

> **Can attacker-controlled execution obtain a memory primitive that defeats or works around the target's protection model?**

## 2.2 R/W/X controls

Think of the permissions independently:

```text
R → Can data be read?
W → Can data be modified?
X → Can instructions execute?
```

Examples:

```text
r--  → read
rw-  → read + write
r-x  → read + execute
rwx  → read + write + execute
```

A typical Linux process may contain:

```text
Code       → r-x
Read-only  → r--
Data       → rw-
Heap       → rw-
Stack      → rw-
```

The absence of `X` on heap and stack is intentional.

## 2.3 Why writable + executable memory matters

Suppose an attacker can write arbitrary bytes into:

```text
rw-
```

That does **not** automatically mean those bytes can execute.

The CPU's execute permission must also permit instruction fetches.

The attacker therefore encounters:

```text
attacker-controlled data
        ↓
writable memory
        ↓
attempted execution
        ↓
NX / execute permission
        ↓
blocked
```

Historically, attackers could sometimes rely on executable stacks or writable code regions.

Modern systems generally make this much harder.

## 2.4 NX / DEP

**NX (No-eXecute)** marks pages as non-executable.

On x86, this is associated with the page-table **execute-disable** mechanism.

Conceptually:

```text
Heap page
┌──────────────────────┐
│ attacker-controlled  │
│ bytes                 │
└──────────────────────┘
       RW
       NX
```

An instruction fetch from that page should fail.

Windows commonly describes the corresponding protection concept as **DEP (Data Execution Prevention)**.

The important security principle is:

> **Data memory should not automatically become executable code.**

## 2.5 User vs supervisor privilege

R/W/X is only part of the protection model.

Pages also have privilege restrictions.

Conceptually:

```text
User process
     │
     ├── User page → allowed
     │
     └── Kernel page → denied
                         ↓
                     page fault
```

The CPU distinguishes between privileged and unprivileged execution.

A normal user process therefore cannot simply dereference a kernel virtual address and obtain kernel memory.

## 2.6 Why privilege separation matters to attackers

Consider:

```text
User space
────────────────────────
Application
Libraries
Heap
Stack
────────────────────────
Kernel space
────────────────────────
Kernel code
Kernel data
Page tables
Device interfaces
────────────────────────
```

A vulnerability that remains inside user space may compromise one application.

A vulnerability that crosses into kernel privilege can potentially compromise the entire operating system.

Therefore attackers care intensely about whether a vulnerability provides:

```text
User → User
```

or:

```text
User → Kernel
```

The second represents a privilege-boundary violation.

## 2.7 Realistic attacker workflow

1. Identify a vulnerable process or memory-corruption primitive.
2. Enumerate its memory mappings and permissions.
3. Determine which regions are writable.
4. Determine which regions are executable.
5. Check whether stack and heap execution are blocked.
6. Determine whether the process has access to privileged mappings.
7. Analyze whether the vulnerability can modify code pointers, data pointers, or control-flow state.
8. Determine which protection prevents the straightforward attack.
9. Identify whether the vulnerability provides another primitive, such as information disclosure or control-flow reuse.
10. Feed the resulting primitive into exploitation analysis.

The important mindset is:

```text
Vulnerability
    ↓
Primitive
    ↓
Memory permissions
    ↓
Privilege boundary
    ↓
Mitigations
    ↓
Actual exploitability
```

## 2.8 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
The process has writable heap memory.
```

That's normal.

Every ordinary dynamically allocating application needs writable memory.

**High-value finding:**

```text
A process obtains an unexpected writable + executable
mapping, and attacker-controlled data can reach it.
```

That deserves investigation because the normal separation between:

```text
write data
```

and:

```text
execute code
```

has been weakened.

Another high-value finding is:

```text
An unprivileged process can modify a privileged memory mapping.
```

That attacks the privilege boundary itself.

## 2.9 Where results feed next

```text
Memory permissions
       ↓
Understand what memory can do
       ↓
Analyze vulnerability primitive
       ↓
Determine mitigation
 ┌─────┼─────────────┐
 ↓     ↓             ↓
 NX   ASLR       Privilege isolation
       ↓
Control-flow / privilege analysis
       ↓
Advanced exploitation
```

# Section 3 — Core concepts and terminology

| Term                   | Meaning                                                                       |
| ---------------------- | ----------------------------------------------------------------------------- |
| Memory protection      | Rules controlling permitted memory accesses                                   |
| R                      | Read permission                                                               |
| W                      | Write permission                                                              |
| X                      | Execute permission                                                            |
| Page permission        | Access properties attached to a memory page                                   |
| NX                     | No-eXecute protection preventing instruction execution from a page            |
| DEP                    | Windows mechanism preventing execution from protected data regions            |
| User mode              | Unprivileged CPU execution level used by applications                         |
| Kernel mode            | Privileged execution level used by the OS kernel                              |
| Supervisor             | Hardware privilege designation for privileged memory access                   |
| User page              | Page permitted for unprivileged access                                        |
| Supervisor page        | Page restricted to privileged execution                                       |
| Page fault             | CPU exception generated when a memory access cannot be satisfied              |
| Protection fault       | Fault caused by violating memory-access permissions                           |
| W^X                    | Policy that prevents memory from being writable and executable simultaneously |
| Guard page             | Deliberately protected/unmapped page used to detect boundary crossings        |
| Memory mapping         | Virtual address range associated with memory or an object                     |
| ASLR                   | Randomizes memory locations                                                   |
| Control-flow integrity | Protection that restricts execution to legitimate control-flow targets        |
| Privilege boundary     | Security boundary separating different execution privileges                   |

### Common permission combinations

| Permission | Read | Write | Execute | Typical use                           |
| ---------- | ---: | ----: | ------: | ------------------------------------- |
| `---`      |    ❌ |     ❌ |       ❌ | Guard/unusable region                 |
| `r--`      |    ✅ |     ❌ |       ❌ | Read-only data                        |
| `rw-`      |    ✅ |     ✅ |       ❌ | Heap/data/stack                       |
| `r-x`      |    ✅ |     ❌ |       ✅ | Program/library code                  |
| `rwx`      |    ✅ |     ✅ |       ✅ | Generally undesirable for normal data |

### Two independent protection questions

```text
Question 1:
"What operation is being attempted?"
       ↓
Read / Write / Execute

Question 2:
"Who is attempting it?"
       ↓
User / Supervisor
```

Both must pass.

For example:

```text
User process
   ↓
Write
   ↓
Supervisor-only read-only page
   ↓
Denied
```

Even though the process requested only one memory operation, multiple protection conditions can fail.

# Section 4 — Tools and commands

| Tool       | Command                                          | What it finds/shows                | When to use it               |
| ---------- | ------------------------------------------------ | ---------------------------------- | ---------------------------- |
| `pmap`     | `pmap -x PID`                                    | Process mappings and permissions   | Quick mapping inspection     |
| `/proc`    | `cat /proc/PID/maps`                             | Exact virtual mappings             | Detailed permission analysis |
| `gdb`      | `info proc mappings`                             | Debugger-visible mappings          | Controlled binary analysis   |
| `gdb`      | `vmmap`                                          | Memory mappings where supported    | Process memory inspection    |
| `readelf`  | `readelf -l ./program`                           | ELF segment permissions            | Analyze executable layout    |
| `checksec` | `checksec --file=./program`                      | NX/PIE/RELRO/canary status         | Mitigation assessment        |
| `strace`   | `strace -e trace=mmap,mprotect,munmap ./program` | Memory mapping/protection syscalls | Observe permission changes   |

### `/proc/PID/maps`

```bash
cat /proc/$(pgrep -n bash)/maps
```

Example:

```text
555555554000-555555575000 r-xp ... /usr/bin/bash
555555575000-555555581000 rw-p ... /usr/bin/bash
7ffff7dd0000-7ffff7fa0000 r-xp ... libc.so.6
7ffffffde000-7ffffffff000 rw-p ... [stack]
```

Interpretation:

```text
r-xp → readable + executable
rw-p  → readable + writable
```

This is one of the most useful first checks when analyzing a Linux process.

### `pmap`

```bash
pmap -x PID
```

Example:

```text
Address           Kbytes   RSS   Dirty Mode  Mapping
555555554000        132    80       0 r-x-- program
555555575000         16    16       8 rw--- program
7ffff7dd0000       1800   900       0 r-x-- libc.so.6
7ffffffde000        132    32      16 rw--- [ stack ]
```

The important information is the mapping mode and object.

### `readelf`

```bash
readelf -l ./program
```

Look for `LOAD` segments and their flags:

```text
Segment Sections...
LOAD    ...  R E
LOAD    ...  RW
```

Conceptually:

```text
R E → executable segment
RW  → writable data segment
```

The ELF loader uses this information when establishing process mappings.

### `checksec`

```bash
checksec --file=./program
```

Example:

```text
RELRO           STACK CANARY      NX
Full RELRO      Canary found      NX enabled
PIE             RPATH
PIE enabled     No RPATH
```

For this topic, the most immediately relevant field is:

```text
NX → enabled
```

That indicates the binary is using non-executable memory protection.

### `strace`

```bash
strace -e trace=mmap,mprotect,munmap ./program
```

Example:

```text
mmap(NULL, 4096, PROT_READ|PROT_WRITE, ...) = 0x7f...
mprotect(0x7f..., 4096, PROT_READ) = 0
```

The important observation is that applications can request mappings and protection changes through OS interfaces.

`mprotect()` can change the permissions associated with an existing mapping, subject to OS policy and mapping constraints.

### GDB mappings

```bash
gdb -q ./program
```

Then:

```text
(gdb) info proc mappings
```

Example:

```text
Start Addr   End Addr     Perms
0x555555...  0x555555...  r-xp
0x555555...  0x555555...  rw-p
```

This lets you correlate runtime addresses with permissions.

# Section 5 — Defender detection

* Unexpected `mprotect()`/mapping activity, especially transitions involving executable permissions, can be valuable behavioral telemetry.
* EDR products can detect suspicious executable-memory creation, permission changes, and execution from unusual anonymous mappings.
* Linux audit/eBPF-based telemetry can provide additional visibility into processes making sensitive memory-management operations.
* Kernel/user separation violations generally manifest through faults, kernel security mechanisms, crashes, or exploit-specific telemetry rather than ordinary application logs.
* Defenders commonly miss that a process can have many legitimate mappings; the suspicious element is usually the **combination of permissions, source, timing, and behavior**.
* Skilled operators avoid unnecessary permission transitions and noisy allocation patterns because these can produce useful behavioral indicators.
* Strong defenses include NX/DEP, ASLR, PIE, W^X policies, kernel/user isolation, control-flow protections, and least-privilege execution.

# Section 6 — Lab task

**Platform:** Kali Linux VM targeting a local C program.

**Objective:** Demonstrate that different process mappings have different R/W/X permissions and observe a runtime permission change.

**Steps:**

1. Create a C program that allocates one memory page using `mmap()`.
2. Request it initially as readable and writable.
3. Write a known value into the mapping.
4. Change the mapping to read-only using `mprotect()`.
5. Run the program while observing its memory syscalls with `strace`.
6. Inspect the process mappings while the program is running.
7. Confirm that the mapping changes from `rw-` to `r--`.
8. Use `checksec` on the binary to determine whether NX is enabled.
9. Save the syscall and mapping evidence.
10. Explain why a write after the `mprotect()` transition should fail.

Example:

```c
#include <stdio.h>
#include <sys/mman.h>
#include <unistd.h>

int main(void)
{
    size_t size = 4096;

    char *mem = mmap(NULL, size,
                     PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS,
                     -1, 0);

    if (mem == MAP_FAILED)
        return 1;

    mem[0] = 'A';

    printf("Mapping: %p\n", (void *)mem);
    printf("PID: %d\n", getpid());

    mprotect(mem, size, PROT_READ);

    sleep(300);

    munmap(mem, size);
    return 0;
}
```

Compile:

```bash
gcc -g -O0 memory_protection.c -o memory_protection
```

Observe:

```bash
strace -e trace=mmap,mprotect,munmap ./memory_protection
```

**Expected output:**

You should see behavior resembling:

```text
mmap(..., PROT_READ|PROT_WRITE, ...) = 0x7f...
mprotect(0x7f..., 4096, PROT_READ) = 0
```

The process mapping should transition conceptually from:

```text
rw-
 ↓
r--
```

**Git artifact:**

```text
memory-protection-lab/
├── README.md
├── src/
│   └── memory_protection.c
└── evidence/
    ├── syscall-trace.txt
    ├── mappings.txt
    └── observations.md
```

Commit:

```bash
git add memory-protection-lab/
git commit -m "Add memory protection permissions lab"
```

# Section 7 — Common mistakes

1. **Mistake → Treating `RWX` as normal for every memory region.**
   **Why it matters →** Writable + executable memory defeats an important separation between data and code.
   **Instead →** Investigate why a region needs both permissions.

2. **Mistake → Thinking NX prevents writing to memory.**
   **Why it matters →** NX controls execution, not writing.
   **Instead →** Separate `W` and `X` conceptually.

3. **Mistake → Thinking user/kernel separation is just a software convention.**
   **Why it matters →** Hardware privilege checks participate directly in enforcing the boundary.
   **Instead →** Understand page-table privilege bits and CPU execution levels.

4. **Mistake → Assuming every permission change is malicious.**
   **Why it matters →** Legitimate applications and runtimes change mappings.
   **Instead →** Correlate permission changes with process identity, timing, memory source, and subsequent execution.

5. **Mistake → Assuming NX alone stops memory exploitation.**
   **Why it matters →** Attackers can potentially reuse existing executable code rather than executing new code from data pages.
   **Instead →** Study NX together with ASLR, PIE, canaries, and control-flow protections.

6. **Mistake → Assuming a writable page can always be modified by user space.**
   **Why it matters →** Write permission and privilege permission are separate checks.
   **Instead →** Evaluate both R/W/X and User/Supervisor properties.

7. **Mistake → Confusing a page fault with proof of exploitation.**
   **Why it matters →** Invalid accesses are common and can result from ordinary programming errors.
   **Instead →** Determine exactly which permission or mapping condition caused the fault.

# Section 8 — Move-on gate

1. **Inspect permissions:** run `pmap` or `/proc/PID/maps` against a live process and correctly identify its executable, heap, stack, and read-only regions without looking at your notes.

2. **Observe a protection transition:** run the lab program under `strace`, identify the `mprotect()` call, and correctly explain the transition from `RW` to `R`.

3. **Analyze binary defenses:** run `checksec` against a compiled binary and correctly identify whether NX is enabled, then explain what specific class of memory execution it prevents.
