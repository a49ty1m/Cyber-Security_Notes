# Process & Thread Mechanics

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → Process & Thread Mechanics

# Section 1 — What it is and where it sits

A **process** is a running program together with its execution state and resources; a **thread** is an execution path inside a process. The operating system creates processes, creates and schedules their threads, gives them CPU time, manages their memory, and saves/restores execution state during **context switches**.

For offensive security, this matters because malware, exploit payloads, EDRs, debuggers, process injection, privilege escalation, and post-exploitation all depend on understanding **who is executing, where they execute, what resources they own, and when the CPU switches between them**.

```text
Operating System Internals
        │
        ├── Privilege Levels
        ├── System Calls
        │
        └── Process & Thread Mechanics
                │
                ├── Process lifecycle
                ├── Thread lifecycle
                ├── Scheduling
                └── Context switching
                         ↓
                 Process Injection
                         ↓
                 Privilege Escalation
                         ↓
                 Malware / EDR Analysis
```

If you underestimate this topic, commands like `ps`, `top`, task managers, debuggers, process injection techniques, and EDR process trees become collections of output rather than things you can reason about.

This builds directly on privilege levels and system calls: syscalls create/control processes and threads, while scheduling determines which runnable thread gets CPU execution next.

# Section 2 — How attackers actually use this

## 2.1 What attackers care about

An attacker rarely cares about a process merely because it exists.

They care about:

* **PID** — which process is involved
* **PPID** — which process created it
* **UID / security token** — what identity it runs under
* **Threads** — how many execution contexts it contains
* **Parent-child relationships** — how execution began
* **Executable path** — what binary is actually running
* **Command line** — how it was launched
* **Open files and sockets** — what resources it controls
* **Memory mappings** — what code/data exists in memory
* **CPU activity** — whether something is actively executing
* **Thread start addresses** — where individual threads began execution
* **Scheduling state** — whether a thread is running, sleeping, waiting, or stopped

The attacker is building a picture like:

```text
PID 4200
  │
  ├── PPID: 1800
  ├── User: alice
  ├── Binary: /opt/app/server
  ├── Threads: 8
  │
  ├── TID 4201 → network worker
  ├── TID 4202 → request worker
  ├── TID 4203 → timer
  └── ...
```

That information can reveal where execution is happening and where a useful process might be targeted.

## 2.2 Process lifecycle

A simplified Linux process lifecycle looks like:

```text
             fork()/clone()
                  │
                  ▼
             New process
                  │
                  ▼
               READY
                  │
              scheduler
                  │
                  ▼
              RUNNING
              /      \
             /        \
            ▼          ▼
       BLOCKED       EXITING
            │           │
     I/O/event          ▼
            │        ZOMBIE
            ▼           │
          READY          │
                        ▼
                    Reaped
```

A process does not continuously execute.

The scheduler repeatedly selects runnable threads and gives them CPU execution.

For example:

```text
Process A → RUNNING
Process B → READY
Process C → SLEEPING
```

The scheduler may later stop A and run B.

## 2.3 Thread lifecycle

Threads have a similar lifecycle:

```text
CREATED
   ↓
READY
   ↓
RUNNING
   ├──→ BLOCKED / WAITING
   │          ↓
   │        READY
   │
   └──→ TERMINATED
```

A thread may stop running because:

* Its time slice expires
* A higher-priority runnable thread needs CPU time
* It performs blocking I/O
* It waits for a lock
* It sleeps
* It voluntarily yields
* It is interrupted or preempted

The process remains alive while other threads continue.

## 2.4 Process vs thread from an attacker's perspective

Consider:

```text
chrome
├── Thread 1
├── Thread 2
├── Thread 3
├── Thread 4
└── ...
```

All threads belong to the same process.

They normally share:

* Virtual address space
* Code
* Heap
* Global data
* Open file descriptors
* Many process-level resources

But each thread has its own:

* Instruction pointer
* CPU register state
* Stack
* Scheduling state
* Thread identifier

That distinction becomes extremely important when analyzing process injection.

An attacker may not need to create an entirely new process.

They may attempt to make an existing process execute additional code.

Conceptually:

```text
Existing trusted process
        │
        ├── Existing threads
        │
        └── New / modified execution
                 ↓
          Attacker-controlled code
```

This is one reason process and thread internals matter to offensive security.

## 2.5 Scheduling

The scheduler decides which runnable thread should execute.

Imagine one CPU core with three runnable threads:

```text
T1 ─┐
T2 ─┼──> Scheduler ──> CPU
T3 ─┘
```

The CPU can execute only one thread at a time on that core.

A simplified sequence might be:

```text
Time
 │
 ├── T1 running
 │
 ├── context switch
 │
 ├── T2 running
 │
 ├── context switch
 │
 ├── T3 running
 │
 └── context switch
     T1 running again
```

On a multicore machine, several threads can genuinely execute simultaneously:

```text
CPU Core 0 → T1
CPU Core 1 → T2
CPU Core 2 → T3
CPU Core 3 → T4
```

But the scheduler still determines which runnable threads are assigned to which CPUs.

## 2.6 Context switching

A **context switch** occurs when the CPU stops executing one thread and begins executing another.

The operating system must preserve enough state for the first thread to resume later.

Conceptually:

```text
Thread A
   │
   │ CPU registers
   │ instruction pointer
   │ stack state
   ▼
Save A's execution context
   │
   ▼
Scheduler selects Thread B
   │
   ▼
Restore B's context
   │
   ▼
CPU executes B
```

The context includes processor state such as:

* Instruction pointer
* Stack pointer
* General-purpose registers
* Relevant control state

The exact implementation is architecture- and OS-dependent.

## 2.7 Why context switching matters to attackers

Context switching creates an important analytical distinction:

```text
Process exists
       ≠
Process is currently executing
```

A suspicious process might spend most of its time sleeping:

```text
RUNNING
   ↓
SLEEP
   ↓
RUNNING briefly
   ↓
SLEEP
```

This behavior can be useful for malware that periodically wakes up, performs an operation, and sleeps again.

Conversely, a CPU-intensive process may produce a very different execution profile.

Therefore, a good analyst looks at:

```text
Process identity
+
Thread activity
+
CPU behavior
+
Parent/child relationship
+
Resource usage
```

rather than relying on the process name alone.

## 2.8 High-value process-tree finding

Suppose you see:

```text
systemd
  └── web-server
        └── shell
              └── suspicious-tool
```

The process tree is valuable because it reveals **execution lineage**.

The suspicious executable might be legitimate by itself.

The unusual parent-child relationship is what makes the observation interesting.

For example:

```text
Web server
    ↓
unexpected shell
    ↓
command execution
```

can indicate that an externally reachable application has been abused.

## 2.9 Dead-end vs high-value finding

**Dead-end finding:**

```text
PID 2410
CPU: 0.1%
Process: python3
```

This tells you very little by itself.

**High-value finding:**

```text
Network-facing service
       ↓
unexpected child process
       ↓
shell
       ↓
new process executing attacker-controlled command
```

The second finding provides execution lineage.

It tells the attacker and defender where the execution originated and what process currently carries it.

## 2.10 Where the results feed next

Process and thread analysis feeds directly into:

```text
Process enumeration
       ↓
Identify interesting process
       ↓
Inspect identity/resources
       ↓
Inspect threads
       ↓
Memory analysis
       ↓
Injection / exploitation
       ↓
Persistence / lateral movement
```

For defenders:

```text
Process tree
    ↓
Parent-child anomaly
    ↓
Thread behavior
    ↓
Memory investigation
    ↓
Malware detection
```

# Section 3 — Core concepts and terminology

| Term                    | Meaning                                                                                       |
| ----------------------- | --------------------------------------------------------------------------------------------- |
| **Process**             | Running program plus its OS-managed execution context and resources.                          |
| **PID**                 | Process identifier assigned by the operating system.                                          |
| **PPID**                | Parent process identifier.                                                                    |
| **Thread**              | Independently schedulable execution context inside a process.                                 |
| **TID**                 | Thread identifier.                                                                            |
| **Process image**       | Program code and associated memory/resources loaded for execution.                            |
| **Process state**       | Current lifecycle/execution condition of a process or thread.                                 |
| **Runnable**            | Ready to execute when a CPU becomes available.                                                |
| **Running**             | Currently executing on a CPU.                                                                 |
| **Sleeping**            | Temporarily not executing, often waiting for time or an event.                                |
| **Blocked**             | Unable to continue until a required resource/event becomes available.                         |
| **Terminated**          | Execution has finished.                                                                       |
| **Zombie**              | Linux process that has terminated but whose parent has not yet collected its exit status.     |
| **Scheduler**           | Kernel component responsible for deciding which runnable thread executes.                     |
| **Preemption**          | Kernel interruption of currently running execution so another thread can run.                 |
| **Context switch**      | Changing CPU execution from one thread to another while preserving/restoring execution state. |
| **Time slice**          | Amount of CPU execution time allocated before scheduling may switch tasks.                    |
| **CPU affinity**        | Restriction or preference for which CPU cores a thread/process can execute on.                |
| **Concurrency**         | Multiple tasks making progress during overlapping periods.                                    |
| **Parallelism**         | Multiple tasks executing simultaneously on separate CPU resources.                            |
| **Thread stack**        | Memory used for a thread's function calls and local execution state.                          |
| **Instruction pointer** | CPU register identifying the next instruction to execute.                                     |
| **Stack pointer**       | CPU register identifying the current stack position.                                          |
| **Process tree**        | Parent-child hierarchy showing process creation relationships.                                |
| **Thread group**        | Linux concept grouping threads belonging to the same process.                                 |
| **Scheduler state**     | Information describing whether a task is runnable, sleeping, stopped, etc.                    |
| **Process injection**   | Technique in which code execution is introduced into another process.                         |

### Process vs thread

| Property       | Process                                        | Thread                                       |
| -------------- | ---------------------------------------------- | -------------------------------------------- |
| Address space  | Own virtual address space                      | Shares process address space                 |
| Scheduling     | Threads are normally scheduled                 | Directly schedulable execution unit          |
| Stack          | Process has resources for its threads          | Own stack                                    |
| Registers      | Execution is represented by its threads        | Own CPU execution state                      |
| PID            | Yes                                            | Process has PID                              |
| TID            | Threads have TIDs                              | Yes                                          |
| Memory sharing | Separate between processes                     | Shared within process                        |
| Failure impact | Process termination can remove all its threads | One thread failure can affect entire process |

The key mental model is:

```text
PROCESS
├── Virtual address space
├── Files/resources
├── Security identity
├── Environment
│
├── THREAD A
│   ├── registers
│   ├── instruction pointer
│   └── stack
│
├── THREAD B
│   ├── registers
│   ├── instruction pointer
│   └── stack
│
└── THREAD C
    ├── registers
    ├── instruction pointer
    └── stack
```

# Section 4 — Tools and commands

| Tool      | Command                  | What it finds/shows                | When to use it                            |
| --------- | ------------------------ | ---------------------------------- | ----------------------------------------- |
| `ps`      | `ps -ef`                 | Processes and parent relationships | Process enumeration                       |
| `ps`      | `ps -eLf`                | Processes plus individual threads  | Thread enumeration                        |
| `pstree`  | `pstree -p`              | Process hierarchy with PIDs        | Analyze execution lineage                 |
| `top`     | `top`                    | Live process/CPU information       | Observe scheduling/resource behavior      |
| `htop`    | `htop`                   | Interactive process/thread view    | Live investigation                        |
| `pidstat` | `pidstat -t 1`           | Per-thread CPU statistics          | Thread activity                           |
| `/proc`   | `cat /proc/<PID>/status` | Process state, IDs, threads        | Detailed process inspection               |
| `/proc`   | `cat /proc/<PID>/sched`  | Scheduler statistics               | Scheduling analysis                       |
| `taskset` | `taskset -pc <PID>`      | CPU affinity                       | Examine CPU assignment                    |
| `strace`  | `strace -f -p <PID>`     | Syscalls from process/threads      | Observe execution behavior                |
| `perf`    | `perf stat -p <PID>`     | CPU performance counters           | Analyze execution/context-switch behavior |

### `ps -ef`

```text
$ ps -ef
UID       PID   PPID  C STIME TTY      TIME CMD
root        1      0  0 ...   ?        ... /sbin/init
user     2401   1800  0 ...   pts/0    ... bash
user     2510   2401  2 ...   pts/0    ... ./server
```

The important fields are:

```text
PID  → process identity
PPID → parent
CMD  → executable/command
```

This lets you reconstruct execution lineage.

### `ps -eLf`

```text
$ ps -eLf
UID   PID   PPID   LWP   NLWP  CMD
user  2510  2401   2510     3  ./server
user  2510  2401   2511     3  ./server
user  2510  2401   2512     3  ./server
```

Here:

```text
PID 2510
```

represents the process.

```text
LWP 2511
LWP 2512
```

represent individual Linux threads.

`NLWP` indicates the number of lightweight processes/threads associated with the process.

### `pstree`

```text
$ pstree -p
systemd(1)
 ├─sshd(700)
 │   └─bash(1800)
 │       └─server(2510)
 │           ├─{server}(2511)
 │           └─{server}(2512)
```

This immediately shows the parent-child process relationship and the server's threads.

### `top`

```text
$ top
PID   USER   PR  NI  VIRT   RES   %CPU  COMMAND
2510  user   20   0  ...    ...   72.5   server
```

A high CPU value means the process is consuming substantial CPU time.

It does **not** by itself prove malicious behavior.

### `htop`

`htop` provides an interactive process view.

You can inspect:

```text
PID
PPID
CPU
Memory
Threads
Command
```

Thread display can be enabled depending on the `htop` configuration/version.

### `pidstat -t`

```text
$ pidstat -t 1
PID     TGID    TID   %usr  %system  %CPU
2510    2510   2510    1.0      0.2    1.2
2510    2510   2511   35.0      2.0   37.0
2510    2510   2512    0.1      0.1    0.2
```

This shows that one thread may be doing most of the CPU work while others remain mostly idle.

### `/proc/<PID>/status`

```text
$ cat /proc/2510/status
Name:   server
State:  S (sleeping)
Pid:    2510
PPid:   2401
Threads:        3
Uid:    1000    1000    1000    1000
```

This gives a compact view of identity, state, parent, and thread count.

### `/proc/<PID>/sched`

```text
$ cat /proc/2510/sched
server (2510, #threads: 3)
se.exec_start              : ...
se.sum_exec_runtime       : ...
nr_switches               : ...
nr_voluntary_switches     : ...
nr_involuntary_switches   : ...
```

The scheduling information can reveal how frequently execution has been switched and how much CPU time the task has accumulated.

### `taskset`

```text
$ taskset -pc 2510
pid 2510's current affinity list: 0-3
```

This indicates the process can execute on CPUs 0 through 3.

### `strace -f`

```text
$ strace -f -p 2510
[pid 2511] futex(..., FUTEX_WAIT, ...) = 0
[pid 2512] epoll_wait(..., ...) = 1
```

The `-f` option makes tracing follow relevant child execution.

The output demonstrates that different threads can be blocked on completely different kernel operations.

### `perf stat`

```text
$ sudo perf stat -p 2510 sleep 5

      ... context-switches ...
      ... cpu-migrations ...
      ... cycles ...
      ... instructions ...
```

Context-switch statistics can help quantify scheduling behavior rather than merely observing it qualitatively.

# Section 5 — Defender detection

* **Process creation telemetry:** Linux audit/EDR telemetry and Windows process-creation telemetry can establish parent-child execution chains.
* **Thread telemetry:** Security products may monitor unusual thread creation, start addresses, or execution inside another process.
* **Context-switch behavior:** Excessive or unusual scheduling activity can be investigated, but context switches alone are not a reliable malware indicator.
* **Process-tree analytics:** A trusted network service spawning an unexpected shell or scripting engine is often more meaningful than the executable name alone.
* **CPU behavior:** Long-running high-CPU threads, periodic execution bursts, or unusual thread behavior can provide supporting evidence.
* **What defenders miss:** Looking only at processes can hide behavior occurring at the thread level, particularly when suspicious execution is introduced into an otherwise legitimate process.
* **Skilled operators:** Attackers may try to blend execution into legitimate processes or avoid obvious process creation, making thread-level and memory-level telemetry important for detecting process injection.

# Section 6 — Lab task

**Platform:** Kali Linux VM.

**Objective:** Create a multithreaded process, identify its process/thread structure, observe individual thread CPU behavior, and inspect scheduling information.

**Steps:**

1. Create a small C program that launches multiple POSIX threads; make one thread perform CPU-intensive work while another sleeps.
2. Compile the program with GCC and execute it.
3. Identify the program's PID using the process list.
4. Enumerate its individual threads and record their TIDs.
5. Use the interactive process monitor to observe CPU consumption.
6. Use per-thread statistics to determine which thread consumes the CPU.
7. Inspect `/proc/<PID>/status` and record the process state and thread count.
8. Inspect `/proc/<PID>/sched` and identify context-switch counters.
9. Attach syscall tracing to observe the threads entering kernel operations such as waits or scheduling-related calls.
10. Save the observations and explain the relationship between process, thread, scheduler, and context switch.

**Expected output:**

```text
Process:  4200
Threads:  3

PID   TID   CPU
4200  4200   low
4200  4201   high
4200  4202   low
```

Your evidence should demonstrate that:

```text
One process
   ↓
Multiple threads
   ↓
Different execution states
   ↓
Scheduler selects runnable threads
   ↓
CPU switches execution between them
```

**Git artifact:**

```text
process-thread-mechanics/
├── README.md
├── src/
│   └── thread_lab.c
├── evidence/
│   ├── ps-threads.txt
│   ├── pstree.txt
│   ├── proc-status.txt
│   ├── proc-sched.txt
│   └── pidstat.txt
└── notes.md
```

```bash
git add process-thread-mechanics/
git commit -m "Document process lifecycle threading and context switching"
```

# Section 7 — Common mistakes

1. **Mistake:** Treating a process and thread as the same thing.
   **Why it matters:** Threads are the actual schedulable execution contexts inside a process.
   **Do instead:** Think "process owns resources; threads execute."

2. **Mistake:** Assuming only one thread can execute per process.
   **Why it matters:** Modern applications routinely contain dozens or thousands of threads.
   **Do instead:** Always inspect thread count when analyzing a suspicious process.

3. **Mistake:** Thinking a context switch means switching processes.
   **Why it matters:** The scheduler switches between execution contexts, commonly threads.
   **Do instead:** Use "thread/task context switch" unless the evidence specifically establishes a process transition.

4. **Mistake:** Treating high CPU usage as proof of malware.
   **Why it matters:** Compilation, browsers, compression, games, and legitimate servers can legitimately consume CPU.
   **Do instead:** Correlate CPU activity with process lineage, executable path, network behavior, and thread behavior.

5. **Mistake:** Ignoring the process tree.
   **Why it matters:** Parent-child relationships frequently reveal how suspicious execution originated.
   **Do instead:** Start with PID + PPID + command line before drawing conclusions.

6. **Mistake:** Assuming sleeping means harmless.
   **Why it matters:** A thread can deliberately sleep between short execution bursts.
   **Do instead:** Observe behavior over time rather than taking one instantaneous snapshot.

7. **Mistake:** Confusing concurrency with parallelism.
   **Why it matters:** Multiple threads can make progress through scheduling on one CPU without literally executing simultaneously.
   **Do instead:** Use concurrency for overlapping progress and parallelism for simultaneous execution on separate CPU resources.

# Section 8 — Move-on gate

1. **Run a multithreaded process, enumerate its PID/TIDs, and correctly identify which TID is consuming CPU without looking at your notes.**

2. **Inspect `/proc/<PID>/status` and `/proc/<PID>/sched`, then correctly identify the process state, thread count, and context-switch counters without referring to the note.**

3. **Observe a running process for several seconds and reconstruct one actual execution sequence — RUNNING → WAITING/SLEEPING → READY → RUNNING — using process/thread telemetry rather than simply describing the lifecycle from memory.**
