# CPU Operations — Fetch–Decode–Execute Cycle

## 1. The Core Mechanism (Under the Hood)

A CPU executes instructions by repeatedly performing a **fetch → decode → execute** cycle. The CPU uses the **Program Counter (PC)** to know where the next instruction is located in memory. During **fetch**, the address in the PC is sent through the memory system, the instruction bytes are retrieved into a CPU register/instruction queue, and the PC advances to the next instruction (unless the instruction changes control flow).

During **decode**, the CPU interprets those instruction bytes according to the processor's **Instruction Set Architecture (ISA)** — for example x86-64 or ARM64. The instruction tells the CPU what operation is required, which registers or memory locations are involved, and sometimes how operands should be interpreted. The CPU then performs the operation during **execute**: arithmetic may happen in an ALU, data may move between registers/memory, or a branch may change the PC.

A simplified example:

```text
Memory
0x401000: 48 83 C0 01    ; add rax, 1

             │
             ▼
        FETCH instruction
             │
             ▼
        DECODE opcode
             │
             ▼
        EXECUTE: RAX = RAX + 1
             │
             ▼
        FETCH next instruction
```

Modern CPUs complicate this heavily with **pipelines, caches, branch prediction, out-of-order execution, and speculative execution**. You don't need to mentally simulate all of that as a junior pentester. The important model is: **software ultimately becomes machine instructions, and those instructions manipulate CPU registers, memory, and control flow.**

---

## 2. Attacker's Angle vs. Defense Footprint

### Offense

This matters because many offensive techniques ultimately manipulate **execution flow**.

For example, in binary exploitation, an attacker may corrupt memory so that a function returns to an attacker-controlled address. The CPU doesn't know "this is a hacker." It simply fetches instructions from the address placed into the instruction pointer and attempts to execute them.

This is the foundation behind concepts such as:

* Buffer overflows
* Return-oriented programming (ROP)
* Shellcode execution
* Code injection
* Control-flow hijacking
* Exploitation of memory-corruption vulnerabilities

### Defense

The basic fetch/decode/execute cycle itself doesn't generate a useful security log.

The **consequences** can.

For example, Windows EDR/Sysmon may detect suspicious process behavior such as:

```text
Office application
      ↓
cmd.exe / powershell.exe
      ↓
network connection
      ↓
unusual memory execution
```

At the hardware level, specialized telemetry such as CPU performance counters can exist, but that's specialist territory. For normal pentesting, you're much more likely to investigate **process creation, memory behavior, crashes, parent-child relationships, and execution anomalies** than raw CPU-cycle telemetry.

---

## 3. Manual Check Before the Tool

For this topic, the best "manual" exercise is **debugging an actual binary instruction-by-instruction**.

Compile a tiny C program with debugging symbols:

```c
#include <stdio.h>

int main() {
    int x = 10;
    x = x + 5;
    printf("%d\n", x);
    return 0;
}
```

Then use a debugger and stop at the instruction responsible for:

```c
x = x + 5;
```

Your job is to observe:

```text
source code
    ↓
assembly instruction
    ↓
CPU register/memory state
    ↓
instruction pointer moves
    ↓
result changes
```

Specifically inspect:

* **RIP** — current instruction address
* **RAX/RBX/etc.** — general-purpose registers
* The instruction bytes
* The disassembled instruction
* Memory containing `x`
* How RIP changes after stepping

**Industry tools:**

* **GDB** — when you want precise instruction/register/memory control on Linux.
* **x64dbg** — when analyzing Windows PE binaries interactively.

Don't start with a debugger plugin or automated reversing tool. First understand what one instruction actually does.

---

## 4. Hands-On Target Task — Local VM

### Target

Use your **Kali Linux VM** and create the small C program above.

Compile it with debugging information:

```bash
gcc -g -O0 cpu.c -o cpu
```

`-O0` is intentional: optimization would make the relationship between your C statements and generated instructions less obvious.

Start the debugger:

```bash
gdb ./cpu
```

Inside GDB, find `main`, place a breakpoint there, and execute the program.

Your task is **not merely to run it**.

Find the instructions corresponding to:

```c
x = x + 5;
```

Then record:

```text
1. Address of the instruction
2. Raw instruction bytes
3. Assembly mnemonic
4. Register containing the relevant value
5. RIP before execution
6. RIP after execution
7. Value before x + 5
8. Value after x + 5
```

### Artifact you should produce

Create a small diagram:

```text
C statement
    ↓
assembly instruction
    ↓
instruction bytes
    ↓
register/memory input
    ↓
CPU operation
    ↓
register/memory output
    ↓
next RIP
```

If you cannot explain that chain without saying **"the CPU just executes it"**, you haven't finished this exercise.

---

## 5. Gotchas / Common Misconceptions

### 1. "One C statement = one CPU instruction"

Wrong.

```c
x = x + 5;
```

may become several machine instructions depending on architecture, compiler, optimization level, and surrounding code.

---

### 2. The CPU executes source code

No.

The CPU executes **machine instructions**.

The rough chain is:

```text
C source
 ↓
compiler
 ↓
assembly
 ↓
machine-code bytes
 ↓
CPU
```

---

### 3. The PC always simply increments

Not necessarily.

A normal sequential instruction advances execution, but:

```text
jmp
call
ret
conditional branch
exception
interrupt
```

can change the next instruction address.

This is extremely important for understanding exploitation.

---

### 4. "Memory is directly executed like a script"

Not exactly.

The CPU fetches instruction bytes through the memory hierarchy, and modern operating systems enforce memory permissions such as **read/write/execute**. A page containing data isn't normally supposed to become executable merely because bytes exist there.

That distinction becomes critical in shellcode and memory-exploitation scenarios.

---

### 5. Modern CPUs aren't literally doing one complete instruction at a time

The fetch/decode/execute model is a **learning abstraction**.

Real CPUs can:

```text
fetch multiple instructions
       ↓
decode multiple instructions
       ↓
execute them out of order
       ↓
retire results in architectural order
```

Don't throw away the basic model because modern CPUs are more complicated. Use the simple model first, then understand where the abstraction breaks.

---

## 6. Terms Worth Defining

* **ISA (Instruction Set Architecture):** The specification defining instructions a CPU understands, such as x86-64 or ARM64.
* **Instruction Pointer (RIP):** On x86-64, the register containing the address of the next instruction to execute.
* **Opcode:** The part of a machine instruction that identifies the operation being performed.
* **Register:** Very small, extremely fast storage locations inside the CPU used while executing instructions.
* **ALU:** Arithmetic Logic Unit; performs operations such as addition, subtraction, AND, OR, and comparisons.
* **Control flow:** The sequence of instruction addresses the CPU follows. Branches, calls, and returns can alter it.
* **Retirement:** The stage where the CPU commits the result of an executed instruction to the architecturally visible CPU state.

---

# 7. Grill Me — Active Recall Gate

**Don't look this up. Explain it from your own understanding.**

### Question 1 — Execution mechanics

Suppose RIP currently points to an instruction that adds `5` to a value held in a register.

Walk me through **exactly what happens from the CPU fetching that instruction until the next instruction begins executing**.

I want the roles of:

* RIP
* memory/cache
* instruction bytes
* decoding
* register/ALU
* updated CPU state
* next RIP

Don't just say *"fetch, decode, execute."* Explain what each stage actually does.

### Question 2 — Offensive failure mode

A vulnerable program contains attacker-controlled data in a memory region that is **read/write but not executable**.

The attacker somehow manages to redirect the instruction pointer to that region.

**Why does redirecting execution there not automatically mean the attacker's code will execute successfully?**

Explain what the CPU and OS/hardware are likely to do, and connect your answer to **memory permissions and control-flow hijacking**.

**Answer both. I'll grade them strictly Pass / Partial / Fail.**

---

# Lab Execution Log — My Actual Session

> **Date:** 2026-09-10 · **Environment:** Fedora Linux · GDB 17.2

## Setup

```bash
gcc -g -O0 cpu.c -o cpu
gdb ./cpu
```

## Disassembly of `main`

```text
(gdb) break main
Breakpoint 1 at 0x40046e: file cpu.c, line 4.

(gdb) run
Breakpoint 1, main () at cpu.c:4
4           int x = 10;

(gdb) disassemble main
   0x0000000000400466 <+0>:     push   %rbp
   0x0000000000400467 <+1>:     mov    %rsp,%rbp
   0x000000000040046a <+4>:     sub    $0x10,%rsp
=> 0x000000000040046e <+8>:     movl   $0xa,-0x4(%rbp)       ; x = 10  (stored on stack)
   0x0000000000400475 <+15>:    addl   $0x5,-0x4(%rbp)       ; x = x + 5  (add to stack memory directly)
   0x0000000000400479 <+19>:    mov    -0x4(%rbp),%eax       ; load x into eax for printf
   ...
```

> **Key observation:** The compiler stored `x` on the **stack** (`-0x4(%rbp)`), not in a register.
> The `addl` instruction operated directly on that stack memory address.
> `eax` was never involved in the `x + 5` calculation.

## Stepping to the `addl` instruction

```text
(gdb) stepi        ; steps past movl $0xa,-0x4(%rbp)
5           x = x + 5;

(gdb) info registers rip
rip    0x400475    0x400475 <main+15>    ; RIP now points at addl
```

## Examining the instruction bytes at the correct address

```bash
(gdb) x/4bx 0x400475
```

> ⚠️ **Mistake made during session:** I used the example address `0x555555555166` from the guide
> instead of my actual address `0x400475`. That caused `Cannot access memory`.
> Always use addresses from your own disassemble output, not from examples.

The correct bytes for `addl $0x5,-0x4(%rbp)`:

```text
0x400475:  0x83  0x45  0xfc  0x05
           │     │     │     └── immediate value: 5
           │     │     └──────── offset: -0x4 (two's complement of 4)
           │     └────────────── ModRM byte: RBP-relative memory operand
           └──────────────────── opcode: 0x83 = ADD r/m32, imm8
```

## Stack value before and after `stepi`

```text
; Before — check stack memory at rbp-4
(gdb) x/dw $rbp-4
0x7fffffffe...:    10    ; x = 10 (0xa)

(gdb) stepi        ; execute addl $0x5,-0x4(%rbp)
6           printf("%d\n", x);

; After — check stack memory at rbp-4
(gdb) x/dw $rbp-4
0x7fffffffe...:    15    ; x = 15 (0xf)
```

## RIP movement

```text
RIP before addl:   0x400475
RIP after  addl:   0x400479
Difference:        4 bytes  (instruction size of addl $0x5,-0x4(%rbp))
```

## What I actually got wrong

| Mistake | What happened | Correct action |
|---------|--------------|----------------|
| Used example address `0x555555555166` | `Cannot access memory` | Use the address from your own `disassemble` output |
| Checked `info registers eax` for x | eax showed junk `-134652440` | x lived on the stack; check with `x/dw $rbp-4` |
| Expected eax to hold x during the add | It didn't — compiler used stack memory directly | `eax` only received x's value *after* the add, for the `printf` call |

## The Real Chain (corrected diagram)

```text
C statement
    ↓
x = x + 5;
    ↓
Assembly instruction
    ↓
addl   $0x5, -0x4(%rbp)
    ↓
Instruction bytes
    ↓
83 45 FC 05
    ↓
Memory input (stack, not register)
    ↓
[rbp - 4] = 0x0000000A  (value: 10)
    ↓
CPU Operation
    ↓
ALU: 0x0A + 0x05 = 0x0F  — result written back to same stack address
    ↓
Memory output
    ↓
[rbp - 4] = 0x0000000F  (value: 15)
    ↓
Next RIP
    ↓
0x400479  (RIP + 4 bytes)
```

> **Bottom line:** The compiler decided `x` didn't need a register — it lived and died on the stack for its entire lifetime in this function.
> The add happened entirely in memory, not in a CPU register.
> If you assumed "variables = registers," your mental model of what the CPU was doing was wrong.
> This is exactly why you do the exercise instead of reading about it.
