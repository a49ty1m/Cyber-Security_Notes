# Privilege Levels

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → Privilege Levels

# Section 1 — What it is and where it sits

Privilege levels are hardware-enforced execution boundaries that determine what CPU instructions and resources a piece of code is allowed to access. On x86 systems, the classic model uses **Ring 0** for the kernel and **Ring 3** for ordinary user applications. Ring 0 has the highest privilege; Ring 3 has the least.

The security objective is isolation: a compromised application should not automatically be able to execute privileged CPU operations, directly manipulate kernel memory, or control hardware. When user-space code needs a privileged operation, it crosses into the kernel through a controlled mechanism such as a system call.

```text
Operating System Internals
        │
        ├── User Space
        │     └── Ring 3
        │          ├── Browser
        │          ├── Shell
        │          └── Applications
        │
        └── Kernel Space
              └── Ring 0
                   ├── Kernel
                   ├── Drivers
                   ├── Memory manager
                   └── Scheduler

Ring 3 ── system call / controlled transition ──> Ring 0
Ring 0 ── return ──> Ring 3
```

If you skip this, you will struggle to understand why a normal process cannot simply read kernel memory, why exploits targeting kernel vulnerabilities are more serious than ordinary application bugs, and why privilege escalation works.

This follows process and memory fundamentals and leads directly into system calls, memory protection, kernel exploitation, and privilege escalation.

# Section 2 — How attackers actually use this

## 2.1 What attackers are trying to cross

An attacker normally starts with code executing at a relatively restricted privilege level:

```text
Initial compromise
       ↓
User-controlled process
       ↓
Ring 3
       ↓
Find privileged interface or vulnerability
       ↓
Ring 0
       ↓
Kernel-level control
```

The important distinction is not simply "user vs administrator."

An administrator process can still execute in **Ring 3**.

A root process on Linux normally executes in **Ring 3** too.

The kernel executes in **Ring 0**.

Therefore:

```text
root ≠ Ring 0
administrator ≠ Ring 0
```

This distinction is critical during vulnerability analysis.

## 2.2 The normal transition

Applications constantly require kernel functionality.

For example, a program wants to read a file.

Conceptually:

```text
Application
   │
   │ request file read
   ↓
System-call interface
   │
   │ controlled CPU transition
   ↓
Kernel
   │
   ├── validates arguments
   ├── checks permissions
   ├── accesses filesystem
   └── copies result
   ↓
Application
```

The application does **not** receive unrestricted kernel access.

The CPU changes execution privilege as part of the controlled transition.

## 2.3 What attackers inspect

An attacker interested in privilege boundaries looks for:

* Kernel vulnerabilities
* Vulnerable device drivers
* Unsafe system calls
* Kernel memory corruption
* Improper validation of user-controlled data
* Race conditions
* Privileged processes exposing dangerous interfaces
* Misconfigured capabilities
* Kernel modules
* Interfaces that allow user-space input to reach privileged code

The key question becomes:

> "Can attacker-controlled data cross from Ring 3 into privileged kernel code in a way the kernel fails to validate safely?"

## 2.4 A realistic attack workflow

Consider a compromised Linux application running as an ordinary user.

The attacker first determines:

```text
Current execution context
        ↓
Process privileges
        ↓
Kernel version/configuration
        ↓
Available privileged interfaces
        ↓
Installed drivers/modules
        ↓
Potential vulnerability
        ↓
Kernel exploitation
        ↓
Ring 0 execution
        ↓
OS-level control
```

Suppose the attacker discovers a vulnerable kernel driver.

The driver may expose an interface where a user process supplies specially crafted input.

Normally:

```text
Ring 3 input
     ↓
Kernel validates input
     ↓
Safe operation
```

A vulnerable implementation might effectively do:

```text
Ring 3 attacker data
        ↓
Driver
        ↓
Insufficient validation
        ↓
Memory corruption
        ↓
Kernel control-flow corruption
        ↓
Potential Ring 0 execution
```

That is fundamentally different from merely obtaining a root shell through a weak password.

## 2.5 Why Ring 0 matters

A successful kernel-level compromise can potentially affect:

* Every process
* Process credentials
* Virtual memory
* Filesystems
* Network traffic
* Security controls
* Kernel modules
* Device access
* Logging mechanisms

A normal compromised process is constrained by operating-system protections.

Kernel compromise attacks the mechanism enforcing those protections.

## 2.6 Dead-end vs high-value finding

**Dead-end finding:**

```text
Browser runs in Ring 3.
```

This is expected behavior.

It tells the attacker almost nothing about privilege escalation.

**High-value finding:**

```text
Unprivileged process
        ↓
Can interact with vulnerable kernel driver
        ↓
Driver contains unsafe memory operation
```

This matters because the attacker may have found a path from restricted execution toward kernel-level control.

The difference is that the second finding identifies a **privilege boundary crossing opportunity**, not merely an observation about the operating system.

## 2.7 Where the result feeds next

A discovered privilege-boundary weakness can unlock:

```text
Ring 3 foothold
      ↓
Kernel vulnerability
      ↓
Ring 0 control
      ↓
Credential / memory / filesystem access
      ↓
Persistence
      ↓
Defense evasion
      ↓
Further compromise
```

This is why privilege levels are foundational for understanding local privilege escalation and kernel exploitation.

# Section 3 — Core concepts and terminology

| Term                     | Meaning                                                                                                                   |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------- |
| **Privilege level**      | CPU-enforced level determining what code can access or execute.                                                           |
| **Ring 0**               | Highest traditional x86 privilege level; normally used by the kernel.                                                     |
| **Ring 3**               | Lowest traditional x86 privilege level; normally used by applications.                                                    |
| **Kernel**               | Privileged operating-system core responsible for managing hardware and system resources.                                  |
| **User space**           | Restricted execution environment containing normal applications.                                                          |
| **Kernel space**         | Protected memory/execution region used by the operating-system kernel.                                                    |
| **System call**          | Controlled interface through which user-space programs request kernel services.                                           |
| **Privilege transition** | CPU-controlled change between execution privilege levels.                                                                 |
| **Ring 1 / Ring 2**      | Intermediate x86 privilege levels that modern general-purpose OSes generally do not use for normal application execution. |
| **Supervisor mode**      | CPU execution mode allowing privileged operations.                                                                        |
| **User mode**            | Restricted CPU execution mode used by ordinary applications.                                                              |
| **Kernel-mode code**     | Code executing with kernel privileges.                                                                                    |
| **User-mode code**       | Code executing with application-level restrictions.                                                                       |
| **Kernel vulnerability** | A security flaw in privileged kernel code that may allow unintended control or access.                                    |
| **Driver**               | Kernel-level software that communicates with hardware or exposes device functionality.                                    |
| **Privilege escalation** | Obtaining greater authority than the attacker initially possesses.                                                        |

### x86 privilege model

| Ring   | Typical use      | Privilege |
| ------ | ---------------- | --------- |
| Ring 0 | Kernel           | Highest   |
| Ring 1 | Rare/OS-specific | High      |
| Ring 2 | Rare/OS-specific | High      |
| Ring 3 | Applications     | Lowest    |

The practical modern model is therefore:

```text
Ring 0 → Kernel
Ring 3 → User applications
```

The CPU and memory-management hardware enforce important parts of this separation.

A Ring 3 process cannot arbitrarily execute privileged instructions simply because the process requests them.

Instead, it must use mechanisms provided by the operating system.

# Section 4 — Tools and commands

| Tool      | Command                 | What it finds/shows                 | When to use it                       |
| --------- | ----------------------- | ----------------------------------- | ------------------------------------ |
| `id`      | `id`                    | Current UID/GID and groups          | Establish process/user identity      |
| `uname`   | `uname -a`              | Kernel/system information           | Identify kernel environment          |
| `lsmod`   | `lsmod`                 | Loaded kernel modules               | Inspect privileged kernel components |
| `modinfo` | `modinfo <module>`      | Kernel-module metadata              | Investigate a specific module        |
| `dmesg`   | `dmesg`                 | Kernel messages                     | Examine kernel/device activity       |
| `strace`  | `strace -f <program>`   | System calls and signals            | Observe Ring 3 → kernel interaction  |
| `ps`      | `ps aux`                | Running processes                   | Identify privileged processes        |
| `cat`     | `cat /proc/self/status` | Process security/status information | Inspect current process context      |

### `id`

```text
$ id
uid=1000(user) gid=1000(user) groups=1000(user),27(sudo)
```

The process is associated with UID 1000.

Being a member of a privileged group does **not** mean the process itself is executing in Ring 0.

### `uname`

```text
$ uname -a
Linux kali 6.x.x-amd64 #1 SMP ... x86_64 GNU/Linux
```

The important information includes architecture and kernel version.

Kernel version can become relevant when investigating kernel vulnerabilities.

### `lsmod`

```text
$ lsmod
Module                  Size  Used by
...
```

This lists currently loaded kernel modules.

A security assessment may investigate whether third-party or vulnerable modules are present.

### `modinfo`

```text
$ modinfo example_module
filename: ...
license: ...
description: ...
version: ...
```

This provides metadata about a kernel module.

### `dmesg`

```text
$ dmesg | tail
[  ... ] device ...
[  ... ] driver ...
```

Kernel messages can reveal driver initialization, hardware events, crashes, and other kernel activity.

### `strace`

```text
$ strace -f ./program
execve("./program", ["./program"], ...) = 0
openat(..., "file.txt", ...)          = 3
read(3, ..., ...)                     = ...
close(3)                              = 0
```

This is particularly useful for visualizing the boundary conceptually.

The application executes in user space, while operations such as `openat()` and `read()` request kernel services through the system-call interface.

### `ps`

```text
$ ps aux
root       512  ... /usr/sbin/...
user      1824  ... /usr/bin/...
```

This helps distinguish processes running under different OS identities.

Again:

```text
root process
    ≠
Ring 0 process
```

A root-owned application normally remains user-mode code.

### `/proc/self/status`

```text
$ cat /proc/self/status
Name:   cat
State:  R
Pid:    ...
Uid:    1000  1000  1000  1000
Gid:    ...
```

This exposes process-level security and execution information through Linux's `/proc` interface.

# Section 5 — Defender detection

* **Kernel logs:** Kernel crashes, driver errors, module events, and abnormal kernel behavior can appear through kernel logging mechanisms such as `dmesg` and centralized logging.
* **EDR telemetry:** Modern endpoint sensors can detect suspicious process behavior, unusual driver loading, kernel exploitation indicators, and abnormal privilege transitions.
* **Driver/module monitoring:** Unexpected kernel modules or unsigned/unapproved drivers are high-value signals.
* **Crash analysis:** Repeated kernel crashes or faults associated with a particular process/driver can indicate attempted exploitation.
* **Behavioral detection:** A normal Ring 3 application suddenly interacting with unusual privileged interfaces can be suspicious.
* **What defenders miss:** They may focus exclusively on UID changes such as `user → root` and miss kernel exploitation where the attacker manipulates privileged execution without an obvious login or `sudo` event.
* **Reducing attacker footprint:** Skilled attackers may avoid noisy crashes, minimize repeated interaction with vulnerable interfaces, and avoid unnecessary kernel modifications because unstable kernel activity can immediately expose the intrusion.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM.

**Objective:** Prove that ordinary applications execute in restricted user space and observe their interaction with the kernel through system calls.

**Steps:**

1. Open a terminal in your Kali VM.
2. Record the current user and group context.
3. Inspect the running kernel and architecture.
4. Start a simple process such as `sleep`.
5. Identify the process using the process list.
6. Trace the process and observe system calls such as process creation, file access, or signal handling.
7. Compare the user-space process with kernel messages available to the system.
8. Record the distinction between **OS identity** (`UID`) and **CPU privilege level** (Ring 3 vs Ring 0).
9. Save the observations as Markdown evidence.
10. Commit the finished artifact to Git.

**Expected output:**

```text
User process
   ↓
UID 1000
   ↓
Ring 3 execution
   ↓
system call
   ↓
Kernel
   ↓
Ring 0
```

Your evidence should show that a normal user process can request kernel services without becoming kernel-mode code.

**Git artifact:**

```text
privilege-levels/
├── README.md
├── evidence/
│   ├── id.txt
│   ├── uname.txt
│   ├── ps.txt
│   └── strace.txt
└── notes.md
```

```bash
git add privilege-levels/
git commit -m "Document x86 privilege levels and syscall boundary"
```

# Section 7 — Common mistakes

1. **Mistake:** Thinking `root` means Ring 0.
   **Why it matters:** Root is an OS identity; Ring 0 is a CPU execution privilege.
   **Do instead:** Keep UID privilege and CPU privilege conceptually separate.

2. **Mistake:** Assuming Ring 1 and Ring 2 are commonly used by Linux applications.
   **Why it matters:** Modern general-purpose systems primarily use Ring 0 and Ring 3.
   **Do instead:** Learn the four-ring architecture, but focus operationally on Rings 0 and 3.

3. **Mistake:** Thinking a system call gives an application unrestricted kernel access.
   **Why it matters:** System calls are controlled entry points.
   **Do instead:** Think of them as a validated interface into privileged functionality.

4. **Mistake:** Treating kernel space as simply "another folder" or process area.
   **Why it matters:** Kernel memory and execution are protected by hardware and OS mechanisms.
   **Do instead:** Think in terms of execution privilege and memory-access permissions.

5. **Mistake:** Assuming every privilege escalation produces a root shell.
   **Why it matters:** Kernel exploitation and other escalation techniques can produce different forms of privileged control.
   **Do instead:** Identify the actual privilege boundary crossed.

6. **Mistake:** Ignoring drivers.
   **Why it matters:** Drivers execute with kernel privileges and can expose attack surfaces to unprivileged processes.
   **Do instead:** Include kernel modules and device interfaces in local attack-surface analysis.

# Section 8 — Move-on gate

1. **Run a system-call trace against a normal Linux application and correctly identify at least three operations that require kernel services without looking at your notes.**

2. **Run the process and kernel inspection commands in your Kali VM and correctly distinguish UID/GID privilege from Ring 3/Ring 0 execution.**

3. **Given a hypothetical unprivileged process interacting with a vulnerable kernel driver, draw the complete path from Ring 3 → vulnerable interface → kernel compromise → Ring 0 without referring to your notes.**
