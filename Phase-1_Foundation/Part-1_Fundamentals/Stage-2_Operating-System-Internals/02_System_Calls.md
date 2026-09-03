# System Calls

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → System Calls

# Section 1 — What it is and where it sits

A **system call (syscall)** is the controlled mechanism that lets a user-space program request a service from the operating-system kernel. Applications cannot directly perform privileged operations such as creating kernel-managed processes, accessing protected files, or manipulating hardware. Instead, they invoke an API that eventually reaches a syscall interface, causing the CPU to transition from user mode to kernel mode.

The exact path differs between Linux and Windows. On modern x86-64 Linux, `execve()` reaches the kernel through the `syscall` instruction. On modern Windows, an API such as `CreateFileW()` eventually reaches the native `NtCreateFile` syscall, typically through a system-call transition rather than the old-style software interrupt mechanism.

```text
User application
      │
      ▼
User-space API / library
      │
      ▼
System-call interface
      │
      │  syscall instruction
      │
      ▼
CPU privilege transition
      │
      ▼
Kernel syscall entry
      │
      ▼
Kernel handler
      │
      ▼
Kernel subsystem
      │
      ▼
Return value
      │
      ▼
User space
```

If you skip this, kernel exploitation, EDR behavior, syscall tracing, API hooking, process creation, file operations, and user/kernel boundary analysis will remain mostly black boxes.

This follows privilege levels and leads directly into kernel internals, process creation, memory management, and local privilege escalation.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers care about the syscall boundary because it is one of the main places where attacker-controlled user-space data enters privileged kernel code.

They specifically look for:

* Which user-space API eventually invokes a syscall
* Which syscall is actually executed
* What arguments reach the kernel
* Which kernel subsystem handles the request
* Whether attacker-controlled data crosses the boundary
* Whether input validation occurs before privileged operations
* Whether security controls observe the API, syscall, or kernel behavior
* Whether a vulnerable syscall or kernel component can be abused

The conceptual chain is:

```text
Attacker-controlled application
          ↓
User-space API
          ↓
Native API / libc wrapper
          ↓
Syscall mechanism
          ↓
Kernel entry
          ↓
Kernel subsystem
          ↓
Security-sensitive operation
```

## 2.2 Linux example — `execve`

Suppose a shell launches a program:

```text
Shell
  ↓
libc
  ↓
execve()
  ↓
syscall instruction
  ↓
x86-64 kernel syscall entry
  ↓
sys_execve / execve implementation
  ↓
process image replacement
  ↓
new program begins execution
```

The important point is that `execve()` does **not** create a second process.

It replaces the current process's program image with another executable.

For an attacker, this matters because process execution is a critical transition point.

For example:

```text
Compromised process
      ↓
execve()
      ↓
/bin/sh
      ↓
Shell execution
```

Security monitoring can therefore treat suspicious process-execution behavior as an important signal.

## 2.3 Linux syscall transition

On x86-64 Linux, the modern path is approximately:

```text
Ring 3
  │
  │ syscall
  ▼
CPU hardware
  │
  ▼
Kernel syscall entry
  │
  ▼
Syscall number lookup
  │
  ▼
Kernel syscall implementation
```

The syscall number identifies which kernel service was requested.

Conceptually:

```text
RAX = syscall number
RDI = argument 1
RSI = argument 2
RDX = argument 3
...
```

The CPU executes:

```text
syscall
```

The processor performs the controlled transition into kernel execution.

The kernel then uses the syscall number to select the appropriate handler.

## 2.4 Windows example — `NtCreateFile`

Windows provides a different abstraction.

An application may call:

```text
CreateFileW()
```

The Windows API layer eventually reaches the native API:

```text
NtCreateFile()
```

The transition then enters the Windows kernel:

```text
Application
    ↓
CreateFileW()
    ↓
Kernel32 / KernelBase
    ↓
ntdll!NtCreateFile
    ↓
syscall
    ↓
Windows kernel
    ↓
NtCreateFile kernel implementation
    ↓
I/O Manager
    ↓
Filesystem driver
    ↓
File operation
```

This is a critical distinction:

```text
CreateFileW
```

is a high-level Windows API.

```text
NtCreateFile
```

is a native NT interface associated with the actual kernel system-call mechanism.

## 2.5 Why attackers trace the entire chain

Suppose malware opens a sensitive file.

At the application level you might see:

```text
CreateFileW("C:\\sensitive.txt")
```

But tracing deeper reveals:

```text
CreateFileW
      ↓
NtCreateFile
      ↓
syscall
      ↓
ntoskrnl
      ↓
I/O Manager
      ↓
filesystem driver
      ↓
disk
```

That tells an analyst where the operation actually occurs.

It also tells an attacker where security instrumentation might exist.

For example:

```text
User API hook
      ↓
Can potentially observe CreateFileW

Syscall-level observation
      ↓
Can observe native system-call activity

Kernel telemetry
      ↓
Can observe deeper filesystem behavior
```

This distinction becomes important when studying EDRs and malware that attempts to bypass user-mode monitoring.

## 2.6 Dead-end vs high-value finding

**Dead-end finding:**

```text
The program calls CreateFileW().
```

That only tells you that the application requested a file operation.

**High-value finding:**

```text
CreateFileW()
    ↓
NtCreateFile
    ↓
syscall
    ↓
kernel I/O path
    ↓
unexpected access to a protected location
```

Now you know:

* Which user-space API was involved
* Which native syscall was reached
* That execution crossed the privilege boundary
* What kernel subsystem ultimately processed the operation
* What security-sensitive resource was accessed

That is much more useful for reverse engineering and detection analysis.

## 2.7 Pivots and next-stage use

Syscall tracing can feed several later activities:

```text
Syscall tracing
      │
      ├── Malware analysis
      ├── EDR analysis
      ├── Process injection research
      ├── Privilege escalation research
      ├── Exploit development
      ├── Rootkit analysis
      └── Detection engineering
```

For example, if suspicious software repeatedly performs:

```text
open/read
mmap
ptrace
execve
```

the sequence can reveal far more than examining a single API call.

The important skill is learning to interpret **syscall sequences as behavior**, not merely memorizing syscall names.

# Section 3 — Core concepts and terminology

| Term                      | Meaning                                                                                                         |
| ------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **System call**           | Controlled request from user space to kernel space.                                                             |
| **Syscall interface**     | Mechanism through which user programs enter kernel functionality.                                               |
| **API**                   | Higher-level programming interface exposed to applications.                                                     |
| **Native API**            | Windows NT-level API exposed primarily through `ntdll.dll`.                                                     |
| **libc**                  | Common Linux user-space library that provides wrappers around many syscalls.                                    |
| **Syscall number**        | Numeric identifier used to select a Linux kernel syscall.                                                       |
| **Syscall handler**       | Kernel code responsible for servicing a particular syscall.                                                     |
| **System-call table**     | Kernel mapping between syscall identifiers and implementations.                                                 |
| **`execve`**              | Linux syscall that replaces a process image with another executable.                                            |
| **`NtCreateFile`**        | Windows native interface for creating/opening file objects.                                                     |
| **`CreateFileW`**         | High-level Windows API used to create/open files and related objects.                                           |
| **`syscall` instruction** | x86-64 instruction used to enter the kernel through the configured syscall mechanism.                           |
| **Interrupt**             | Hardware/software event that transfers control to a handler; older syscall mechanisms used software interrupts. |
| **Kernel entry**          | Architecture-specific code reached when execution transitions into the kernel.                                  |
| **Return value**          | Result supplied by the kernel back to user space.                                                               |
| **User mode**             | Restricted execution mode for ordinary applications.                                                            |
| **Kernel mode**           | Privileged execution mode used by the kernel.                                                                   |
| **I/O Manager**           | Windows kernel subsystem coordinating I/O requests.                                                             |
| **System-call boundary**  | The security boundary separating ordinary application execution from kernel execution.                          |

### Important syscall paths

| Platform             | Typical path                                                                      |
| -------------------- | --------------------------------------------------------------------------------- |
| Linux x86-64         | Application → libc → syscall wrapper → `syscall` → kernel entry → syscall handler |
| Windows x64          | Application → Win32 API → `ntdll` native API → `syscall` → Windows kernel         |
| Older x86 mechanisms | User code → software interrupt such as `int 0x80` → kernel entry                  |

Do not confuse:

```text
API call
```

with:

```text
system call
```

An API can perform additional work before eventually making a syscall.

For example:

```text
CreateFileW()
    ↓
Windows user-mode libraries
    ↓
NtCreateFile()
    ↓
syscall
    ↓
kernel
```

# Section 4 — Tools and commands

| Tool             | Command                            | What it finds/shows                            | When to use it                           |
| ---------------- | ---------------------------------- | ---------------------------------------------- | ---------------------------------------- |
| `strace`         | `strace -f ./program`              | Linux syscalls                                 | Trace Linux programs                     |
| `strace`         | `strace -e trace=execve ./program` | Only `execve` activity                         | Study process execution                  |
| `strace`         | `strace -e trace=file ./program`   | File-related syscalls                          | Study filesystem operations              |
| `ltrace`         | `ltrace ./program`                 | User-space library calls                       | Compare API/library calls with syscalls  |
| `gdb`            | `gdb ./program`                    | Program execution and registers                | Inspect syscall arguments/registers      |
| `objdump`        | `objdump -d ./program`             | Disassembled instructions                      | Locate syscall instructions              |
| `strings`        | `strings ./program`                | Embedded strings                               | Identify paths/commands used by binaries |
| `x64dbg`         | `x64dbg`                           | Windows user-mode execution                    | Trace Windows API/native API calls       |
| WinDbg           | `windbg`                           | Windows debugging                              | Inspect native/kernel execution          |
| Process Monitor  | `procmon`                          | Windows process/file/registry/network activity | Observe high-level Windows behavior      |
| Process Explorer | `procexp`                          | Process and module information                 | Inspect Windows processes                |

### `strace -e trace=execve`

```text
$ strace -e trace=execve /bin/echo hello
execve("/bin/echo", ["/bin/echo", "hello"], ...) = 0
hello
+++ exited with 0 +++
```

Interpretation:

```text
execve(...)
    ↓
syscall successfully executed
    ↓
return value = 0
```

The `0` indicates successful execution.

### `strace -e trace=file`

```text
$ strace -e trace=file cat /etc/hostname
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
...
```

The process requested access to `/etc/hostname`.

The kernel returned file descriptor `3`.

### `ltrace`

```text
$ ltrace ./program
printf("Hello\n") = 6
puts("Hello") = 6
```

`ltrace` primarily shows library-level activity rather than directly showing the kernel syscall boundary.

This makes it useful for comparing:

```text
Library/API behavior
        vs.
Kernel syscall behavior
```

### `gdb`

```text
$ gdb ./program
(gdb) disassemble main
...
(gdb) info registers
```

On x86-64 Linux, syscall arguments and the syscall number can be inspected through registers.

For example:

```text
RAX → syscall number
RDI → argument 1
RSI → argument 2
RDX → argument 3
```

### `objdump`

```text
$ objdump -d ./program | grep syscall
```

Possible output:

```text
...:
    syscall
```

This identifies the actual `syscall` instruction in executable code.

### Windows — Process Monitor

A suspicious application might produce events resembling:

```text
Process: example.exe
Operation: CreateFile
Path: C:\Users\User\test.txt
Result: SUCCESS
```

This is valuable behavioral telemetry, although Process Monitor does not simply display the raw CPU transition itself.

### Windows — x64dbg

In a Windows lab, you can trace:

```text
CreateFileW
    ↓
ntdll!NtCreateFile
    ↓
syscall
```

The debugger allows you to inspect registers, stack state, modules, and execution flow.

# Section 5 — Defender detection

* **Linux audit telemetry:** Linux auditing can record selected syscall activity and associate it with users, processes, executable paths, and results.
* **EDR syscall telemetry:** Endpoint products can observe suspicious process behavior at API, syscall, and kernel levels depending on their architecture.
* **Process execution:** Unexpected `execve` activity, especially from unusual parent/child relationships, can be a strong behavioral indicator.
* **Windows process/file telemetry:** Windows security and endpoint telemetry can associate processes with file and process operations even when the raw syscall transition is not directly exposed.
* **Kernel-level monitoring:** Kernel security components can observe operations after the user/kernel transition and may detect behavior invisible to simple user-mode API monitoring.
* **What defenders miss:** Monitoring only high-level APIs can miss alternative execution paths, direct/native API usage, or activity that bypasses a particular user-mode hook.
* **Operator footprint:** Sophisticated malware may reduce obvious API-level indicators and alter its syscall/API usage, but this does not make the underlying kernel operation invisible; defenders can correlate the resulting behavior instead of relying on one API name.

# Section 6 — Lab task

**Platform:** Kali Linux VM.

**Objective:** Trace `/bin/echo` and a small process-execution workflow from the user-space program into the Linux syscall interface and correctly identify `execve`.

**Steps:**

1. Open a terminal and confirm that `/bin/echo` exists.
2. Run the program normally so you have a baseline.
3. Trace only process-execution syscalls with `strace`.
4. Repeat the trace while following child processes where applicable.
5. Save the syscall output to an evidence file.
6. Trace file-related activity from a simple program such as `cat /etc/hostname`.
7. Compare the library-level behavior using `ltrace` with the syscall-level behavior from `strace`.
8. Use `objdump` on a suitable binary or library to identify a `syscall` instruction.
9. Record the Linux x86-64 register convention for syscall number and arguments.
10. Write a short diagram showing user space → syscall instruction → kernel entry → handler.

**Expected output:**

```text
execve("/bin/echo", ["/bin/echo", "hello"], ...) = 0
```

You should be able to explain:

```text
/bin/echo
   ↓
user-space execution
   ↓
execve request
   ↓
syscall mechanism
   ↓
kernel syscall entry
   ↓
execve implementation
   ↓
new process image
```

**Git artifact:**

```text
system-calls/
├── README.md
├── evidence/
│   ├── execve-strace.txt
│   ├── file-strace.txt
│   └── library-trace.txt
└── notes.md
```

```bash
git add system-calls/
git commit -m "Trace Linux syscall execution from user space to kernel"
```

# Section 7 — Common mistakes

1. **Mistake:** Calling every API call a syscall.
   **Why it matters:** High-level APIs often perform significant user-space work before reaching the kernel.
   **Do instead:** Trace the API down to the actual syscall boundary.

2. **Mistake:** Assuming `execve()` creates a new process.
   **Why it matters:** `execve()` replaces the current process image.
   **Do instead:** Separate `fork()`/`clone()` process creation from `execve()` program replacement.

3. **Mistake:** Calling the modern Linux `syscall` transition an interrupt.
   **Why it matters:** `syscall` is a dedicated x86 instruction, whereas mechanisms such as `int 0x80` are software interrupts.
   **Do instead:** Learn both, but use the architecture-appropriate terminology.

4. **Mistake:** Assuming the syscall itself contains all the work.
   **Why it matters:** A syscall commonly enters a larger kernel subsystem.
   **Do instead:** Follow the request beyond the syscall handler into the relevant subsystem.

5. **Mistake:** Using only `ltrace` to study kernel interaction.
   **Why it matters:** `ltrace` primarily exposes user-space library calls.
   **Do instead:** Use `strace` when your objective is syscall tracing.

6. **Mistake:** Assuming syscall tracing automatically means kernel debugging.
   **Why it matters:** `strace` observes syscall activity but does not automatically show every internal kernel function executed afterward.
   **Do instead:** Use kernel tracing/debugging tools when you need internal kernel execution paths.

# Section 8 — Move-on gate

1. **Run `strace -e trace=execve` against a program and correctly identify the `execve` request, its arguments, and its return value without looking at your notes.**

2. **Trace a file operation and draw the actual path from the user-space program through the syscall boundary to the kernel filesystem subsystem.**

3. **Take a Linux x86-64 binary, locate a `syscall` instruction, inspect the relevant registers in a debugger, and correctly identify which register contains the syscall number and which contain its first three arguments.**
