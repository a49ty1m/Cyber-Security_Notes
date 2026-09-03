# The Heap

**Roadmap:** Part 1 — Fundamentals → Stage 3 — Memory Management → The Heap

# Section 1 — What it is and where it sits

The **heap** is a process memory region used for **dynamic memory allocation**: memory requested while the program is running rather than having its size fixed entirely at compile time. In C, functions such as `malloc()`, `calloc()`, `realloc()`, and `free()` provide the application-level interface to the allocator.

The heap is not simply "unused RAM." A user-space allocator manages a larger virtual-memory region and divides it into allocations requested by the program. Underneath that allocator, the OS provides virtual memory through mechanisms such as `mmap()` and `brk()`/`sbrk()`. The exact allocator behavior depends on the platform and runtime.

```text
Operating Systems
└── Memory Management
    └── Stage 3
        ├── Virtual Memory
        ├── Stack
        └── Heap
             ↓
        Dynamic allocation
             ↓
        Memory corruption
             ↓
        Heap exploitation
```

```text
Application
    ↓
malloc(size)
    ↓
C runtime allocator
    ↓
Existing heap space?
 ┌──┴─────────────┐
Yes              No
 ↓                ↓
Return chunk   Request more virtual memory
                  ↓
                OS
                  ↓
             mmap / brk
```

If you skip this, vulnerabilities such as **use-after-free, double-free, heap overflow, and allocator metadata corruption** become difficult to understand. It also becomes much harder to distinguish an application-level allocator problem from an OS-level virtual-memory operation.

The stack taught you function-local execution state; the heap now teaches you runtime-managed objects whose lifetime can extend beyond a single function call.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

An attacker analyzing heap behavior looks for:

* Dynamically allocated objects
* Object sizes and layouts
* Allocation and free operations
* User-controlled heap data
* Adjacent allocations
* Lifetime mistakes
* Use-after-free conditions
* Double-free conditions
* Heap out-of-bounds accesses
* Allocator metadata corruption
* Function pointers or sensitive objects stored in heap memory
* Heap addresses and ASLR behavior

The important question is:

> **Can attacker-controlled operations corrupt, reuse, or disclose a heap object in a way that changes program behavior?**

## 2.2 Application requests memory

Consider:

```c
char *name = malloc(64);
```

The application is effectively saying:

```text
"I need at least 64 bytes of dynamically usable storage."
```

The allocator may already have suitable free memory.

If not, it can obtain additional memory from the operating system and manage it internally.

The application normally does **not** receive a raw physical RAM address.

It receives a virtual address:

```text
malloc(64)
    ↓
allocator
    ↓
virtual address
    ↓
process accesses memory
```

## 2.3 A realistic attacker workflow

1. Identify application functionality that creates dynamic objects.
2. Determine which input controls their contents or size.
3. Observe allocation and deallocation behavior.
4. Determine whether attacker-controlled data can outlive the object that owns it.
5. Look for operations that access freed memory.
6. Look for writes beyond an allocation's boundaries.
7. Determine whether corrupted data influences pointers, object fields, lengths, or control-flow-related values.
8. Check which allocator and hardening mechanisms are present.
9. Determine whether the primitive provides information disclosure, arbitrary memory corruption, or control-flow influence.
10. Feed the result into the appropriate exploitation analysis.

## 2.4 Heap overflow

Suppose an application allocates:

```text
Object A
┌──────────────────────┐
│ 32-byte buffer       │
└──────────────────────┘

Object B
┌──────────────────────┐
│ sensitive data       │
└──────────────────────┘
```

If the program incorrectly writes 80 bytes into A:

```text
Object A
┌─────────────────────────────────────────┐
│ attacker-controlled data                │
└─────────────────────────────────────────┘
                       ↓
Object B
┌──────────────────────┐
│ potentially corrupted│
└──────────────────────┘
```

The security impact depends on what sits nearby and what the corrupted data controls.

A heap overflow therefore does **not** automatically mean code execution.

## 2.5 Use-after-free

Consider:

```c
char *p = malloc(32);

free(p);

/* Later */
printf("%s\n", p);
```

After `free(p)`, the program no longer owns that allocation.

Conceptually:

```text
Allocated
   ↓
free()
   ↓
Memory becomes available to allocator
   ↓
Old pointer still exists
   ↓
Program uses old pointer
   ↓
Use-after-free
```

An attacker may try to influence what occupies that memory afterward.

The important primitive is not simply "freed memory exists."

It is:

```text
old pointer
    +
attacker-controlled replacement object
    +
later dereference
```

## 2.6 Double-free

A double-free occurs when the same allocation is released more than once:

```c
char *p = malloc(64);

free(p);
free(p);
```

Modern allocators contain consistency checks and hardening mechanisms that detect many straightforward double-free patterns.

Therefore:

```text
double-free ≠ automatic exploitation
```

The attacker must determine whether the allocator accepts the sequence and whether the resulting state can produce a useful memory-corruption primitive.

## 2.7 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
The application uses malloc().
```

Almost every non-trivial C application does.

That tells you little.

**High-value finding:**

```text
An attacker-controlled object is freed,
then attacker-controlled data can occupy the
same region before the stale pointer is used.
```

This creates a potentially meaningful use-after-free primitive.

The difference is **control over memory lifetime and subsequent interpretation**, not simply the presence of heap allocation.

## 2.8 Allocator behavior matters

The application sees:

```c
malloc(100);
```

But internally the allocator may manage:

```text
Heap arena
├── allocator metadata
├── allocated chunk
├── free chunk
├── allocated chunk
└── free chunk
```

The exact representation varies by allocator.

On modern Linux systems using glibc, mechanisms such as **tcache**, **fastbins**, and other allocator structures influence how freed chunks are managed.

You should learn the allocator actually used by your target rather than memorizing one historical heap layout.

## 2.9 Where results feed next

Heap analysis feeds directly into:

```text
Dynamic allocation
      ↓
Object lifetime
      ↓
Heap corruption
 ┌────┼──────────────┐
 ↓    ↓              ↓
Overflow UAF      Double-free
      ↓
Memory corruption primitive
      ↓
Information disclosure / control
      ↓
Advanced exploitation
```

# Section 3 — Core concepts and terminology

| Term               | Meaning                                                            |
| ------------------ | ------------------------------------------------------------------ |
| Heap               | Process memory managed for dynamic allocations                     |
| Dynamic allocation | Requesting memory while the program executes                       |
| Allocator          | Runtime component managing allocations and frees                   |
| `malloc()`         | Allocates uninitialized dynamic memory                             |
| `calloc()`         | Allocates and zero-initializes memory                              |
| `realloc()`        | Changes the size of an existing allocation                         |
| `free()`           | Releases an allocation back to the allocator                       |
| Allocation         | Memory region currently given to the application                   |
| Chunk              | Allocator-managed unit of heap memory                              |
| Arena              | Allocator-managed heap management area                             |
| Metadata           | Allocator information associated with managed memory               |
| Fragmentation      | Inefficient gaps created by allocation patterns                    |
| Heap overflow      | Writing beyond an allocated heap object                            |
| Use-after-free     | Accessing an object after it has been freed                        |
| Double-free        | Freeing the same allocation more than once                         |
| Memory leak        | Allocated memory that is no longer reachable but not freed         |
| Dangling pointer   | Pointer referring to an object whose lifetime ended                |
| `brk()`            | System interface historically used to expand the process data area |
| `mmap()`           | System call capable of creating memory mappings                    |
| tcache             | glibc per-thread cache for recently freed allocations              |
| ASLR               | Randomizes memory locations to make addresses less predictable     |

### Allocation lifecycle

```text
Request
   ↓
malloc()
   ↓
Allocator finds/provisions memory
   ↓
Application uses allocation
   ↓
free()
   ↓
Allocator reclaims memory
```

### Common allocation APIs

| API               | Purpose            | Initial contents                                   |
| ----------------- | ------------------ | -------------------------------------------------- |
| `malloc(n)`       | Allocate `n` bytes | Uninitialized                                      |
| `calloc(n, size)` | Allocate array     | Zero-initialized                                   |
| `realloc(p, n)`   | Resize allocation  | Preserves existing contents up to applicable limit |
| `free(p)`         | Release allocation | Allocation becomes invalid                         |

### Important distinction

```text
Application layer
    malloc()
        ↓
Allocator
        ↓
OS virtual-memory interface
        ↓
MMU / page tables
        ↓
Physical memory
```

`malloc()` is therefore **not itself a system call**.

# Section 4 — Tools and commands

| Tool       | Command                              | What it finds/shows                        | When to use it                |
| ---------- | ------------------------------------ | ------------------------------------------ | ----------------------------- |
| `gdb`      | `gdb -q ./program`                   | Heap addresses and execution state         | Interactive analysis          |
| `gdb`      | `break malloc`                       | Stops on allocation calls                  | Trace allocation behavior     |
| `gdb`      | `break free`                         | Stops on releases                          | Analyze object lifetime       |
| `gdb`      | `heap chunks`                        | Allocator chunk information when supported | Heap-layout investigation     |
| `valgrind` | `valgrind --tool=memcheck ./program` | Invalid reads/writes and leaks             | Find memory bugs              |
| `ltrace`   | `ltrace ./program`                   | Library calls such as malloc/free          | Observe allocation API usage  |
| `strace`   | `strace -e trace=memory ./program`   | Relevant memory-related syscalls           | See OS-level memory requests  |
| `pmap`     | `pmap -x PID`                        | Process mappings                           | Locate heap mapping           |
| `checksec` | `checksec --file=./program`          | Binary hardening                           | Establish mitigation baseline |

### `gdb`

Start the program:

```bash
gdb -q ./heap_demo
```

Set an allocation breakpoint:

```text
(gdb) break malloc
(gdb) run
```

Typical result:

```text
Breakpoint 1, malloc (...)
```

Continue execution and inspect the returned pointer when the call finishes.

The returned value is a **virtual address inside the process**, not a physical RAM address.

### `valgrind`

```bash
valgrind --tool=memcheck ./heap_demo
```

Example:

```text
==4217== Invalid read of size 1
==4217==    at 0x1091A2: main (heap_demo.c:18)
==4217== Address 0x4a45040 is 0 bytes inside a block of size 32 free'd
```

This is highly valuable during development because it identifies memory accesses that violate object lifetime or bounds.

### `ltrace`

```bash
ltrace ./heap_demo
```

Example:

```text
malloc(64) = 0x55...
free(0x55...) = <void>
+++ exited (status 0) +++
```

This exposes library-level allocation activity.

### `strace`

```bash
strace -e trace=memory ./heap_demo
```

Depending on the program and allocator behavior, you may observe calls such as:

```text
brk(NULL) = 0x55...
mmap(NULL, 135168, PROT_READ|PROT_WRITE, ...) = 0x7f...
```

Interpretation:

```text
malloc()
  ↓
allocator
  ↓
brk/mmap when additional virtual memory is needed
```

Do not assume every `malloc()` corresponds to one `mmap()` call.

### `pmap`

```bash
pmap -x PID | grep '\[heap\]'
```

Example:

```text
0000555566a00000    132      16      16 rw---   [ heap ]
```

This identifies the process's conventional heap mapping.

### `checksec`

```bash
checksec --file=./heap_demo
```

Example:

```text
RELRO           STACK CANARY      NX
Partial RELRO   No canary found   NX enabled
PIE             RPATH
No PIE          No RPATH
```

This establishes the binary's general hardening baseline before deeper analysis.

# Section 5 — Defender detection

* **AddressSanitizer (ASan)** detects many heap out-of-bounds accesses and use-after-free conditions during testing.
* Valgrind Memcheck can expose invalid heap reads/writes, leaks, and lifetime violations, though it is generally much slower than native execution.
* EDR/runtime telemetry can identify suspicious memory allocation, protection changes, crashes, and abnormal process behavior.
* Defenders commonly miss heap vulnerabilities because the process may continue running after corruption instead of immediately crashing.
* Repeated allocator errors, abnormal process termination, and allocator consistency failures can provide useful indicators.
* Skilled attackers may avoid repeated crashes and minimize unusual allocation patterns, making behavioral context more important than a single event.
* Production defenses include hardened allocators, ASLR, NX, compiler sanitizers during testing, memory-safe languages where appropriate, and rigorous ownership/lifetime management.

# Section 6 — Lab task

**Platform:** Kali Linux VM with a local C program containing controlled allocation and deallocation mistakes.

**Objective:** Observe dynamic allocation and use Valgrind to identify a heap lifetime violation.

**Steps:**

1. Create a C program that allocates a small heap buffer.
2. Write known data into the buffer.
3. Print the allocation address.
4. Free the buffer.
5. Intentionally access the stale pointer afterward.
6. Compile the program with debugging information.
7. Run it normally and record its behavior.
8. Run it under Valgrind Memcheck.
9. Identify the invalid access and the original allocation.
10. Save the allocation/lifetime evidence in your repository.

Example:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void)
{
    char *data = malloc(32);

    strcpy(data, "heap laboratory");

    printf("Allocation: %p\n", (void *)data);
    printf("Before free: %s\n", data);

    free(data);

    printf("After free: %s\n", data);

    return 0;
}
```

Compile:

```bash
gcc -g -O0 heap_demo.c -o heap_demo
```

Run:

```bash
./heap_demo
```

Then:

```bash
valgrind --tool=memcheck ./heap_demo
```

**Expected output:**

Valgrind should report an invalid access associated with the freed allocation, conceptually:

```text
Invalid read of size 1
Address ... is ... inside a block of size 32 free'd
```

The exact output and address will vary.

**Git artifact:**

```text
the-heap-lab/
├── README.md
├── src/
│   └── heap_demo.c
└── evidence/
    ├── normal-run.txt
    ├── valgrind.txt
    └── observations.md
```

Commit:

```bash
git add the-heap-lab/
git commit -m "Add dynamic heap allocation lab"
```

# Section 7 — Common mistakes

1. **Mistake → Thinking `malloc()` directly allocates physical RAM.**
   **Why it matters →** The application receives virtual memory managed by the allocator and OS.
   **Instead →** Separate `malloc()` → allocator → OS → virtual memory → physical memory.

2. **Mistake → Treating the heap as one simple contiguous block.**
   **Why it matters →** Allocators manage multiple regions, chunks, caches, and metadata.
   **Instead →** Analyze the actual allocator and runtime.

3. **Mistake → Using memory after `free()`.**
   **Why it matters →** The object lifetime has ended and the allocator may reuse the memory.
   **Instead →** Set pointers to `NULL` where appropriate and enforce clear ownership.

4. **Mistake → Assuming every heap overflow is exploitable.**
   **Why it matters →** The corrupted data may be irrelevant or protected by allocator/application defenses.
   **Instead →** Identify exactly what gets corrupted and whether it influences security-sensitive behavior.

5. **Mistake → Ignoring allocation size.**
   **Why it matters →** Integer mistakes and incorrect size calculations can create undersized allocations.
   **Instead →** Trace size calculations from attacker input through allocation and subsequent writes.

6. **Mistake → Memorizing old glibc exploitation techniques without understanding allocator behavior.**
   **Why it matters →** Modern allocator versions contain substantial hardening and behavior differs by version/configuration.
   **Instead →** First understand the allocator's current behavior, then study historical techniques.

# Section 8 — Move-on gate

1. **Trace an allocation:** run your lab program under `ltrace` and identify the `malloc()` and `free()` calls, including the returned allocation address.

2. **Find a lifetime violation:** run the lab under Valgrind and correctly identify the freed allocation and the later invalid access without consulting your notes.

3. **Trace the OS boundary:** inspect a dynamically allocating program with `strace` and explain which observed `brk()`/`mmap()` activity represents the allocator obtaining additional virtual memory.
