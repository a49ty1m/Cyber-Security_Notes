# Registers

**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → CPU Operations → Registers

# Section 1 — What it is and where it sits

CPU **registers** are tiny, extremely fast storage locations inside the processor. They hold values the CPU needs immediately while executing instructions: data, addresses, stack locations, and the address of the next instruction. For offensive security, registers are critical because they expose the CPU's current execution state.

The three register families in this topic are especially important: **RAX/EAX** commonly holds arithmetic results and function return values, **RSP/ESP** tracks the stack, and **RIP/EIP** identifies the current execution location.

```text
CPU
├── RAX/EAX → data / arithmetic / return values
├── RSP/ESP → stack location
└── RIP/EIP → instruction location
       ↓
Fetch → Decode → Execute
```

Attack-chain placement:

```text
CPU Operations
      ↓
Registers ← THIS TOPIC
      ↓
Process Memory
      ↓
Stack / Heap
      ↓
Assembly & Debugging
      ↓
Memory Corruption
      ↓
Control-Flow Hijacking
```

If you underestimate registers, debugging becomes guesswork. You may see a crash but not know **what instruction was executing, where the stack was, or which value was being manipulated**.

This topic directly builds on the Fetch-Decode-Execute cycle and leads into process memory, calling conventions, assembly analysis, and eventually binary exploitation.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

During binary analysis, an attacker wants to answer questions such as:

* What instruction is executing?
* Where is the stack?
* What value is currently being processed?
* Which register contains a function argument?
* Where will the function return?
* Has attacker-controlled data reached a register?
* Did a memory corruption overwrite control-flow data?
* What register values are required by a useful instruction sequence?

The key mental model is:

```text
RIP = Where execution is
RSP = Where the stack currently is
RAX = A commonly used working/result register
```

These are not the only registers that matter, but they are excellent starting points.

## 2.2 RAX/EAX — following data through execution

`RAX` is a 64-bit x86-64 general-purpose register.

`EAX` is its lower 32 bits.

```text
RAX
┌──────────────────────────────────────────────┐
│                  64 bits                     │
└──────────────────────┬───────────────────────┘
                       │
                 lower 32 bits
                       ↓
                     EAX
```

Suppose a program calculates:

```text
10 + 20
```

Conceptually:

```text
RAX = 10
RAX = RAX + 20
RAX = 30
```

An attacker reversing a binary tracks this movement of data.

A register can contain:

```text
integer
pointer
address
function result
temporary calculation
part of a structure
```

The meaning depends on how the surrounding instructions use it.

### Why this matters offensively

Suppose you encounter:

```text
mov rax, [address]
add rax, 8
```

The attacker immediately asks:

> What is stored at `address`, and why is 8 being added?

If the value is a pointer, `RAX` may now point to another field in a structure.

If the value is attacker-controlled, the resulting pointer may influence a later memory access.

The register itself is not "dangerous." Its **data flow** is what matters.

## 2.3 ESP/RSP — understanding the stack

`RSP` is the 64-bit x86-64 **stack pointer**.

`ESP` is the lower 32-bit register traditionally associated with the stack pointer in 32-bit x86.

Conceptually:

```text
Higher addresses
┌─────────────────────┐
│ older stack data    │
├─────────────────────┤
│ return information  │
├─────────────────────┤
│ local variables     │
├─────────────────────┤ ← RSP
│ current stack top   │
└─────────────────────┘
Lower addresses
```

The exact layout depends on the function, compiler, ABI, and optimization.

`RSP` tells the CPU where the current stack top is.

Instructions such as stack pushes and pops modify it.

Conceptually:

```text
PUSH value
    ↓
RSP changes
    ↓
value placed on stack
```

and:

```text
POP register
    ↓
value read from stack
    ↓
RSP changes
```

### Why attackers care

The stack commonly contains:

* local variables
* saved register state
* function-related data
* temporary values
* return-control information

During vulnerability research, an attacker may therefore inspect:

```text
buffer
 ↓
stack layout
 ↓
saved control data
 ↓
RIP
```

The important distinction is that **RSP points to the stack**, while **RIP determines execution**.

## 2.4 EIP/RIP — controlling execution

`RIP` is the 64-bit x86-64 instruction pointer.

`EIP` is the 32-bit x86 instruction pointer.

It represents the location from which the CPU obtains the next instruction in the normal architectural model.

For example:

```text
RIP = 0x401156
```

means execution is currently associated with code around:

```text
0x401156
```

An analyst can therefore ask:

```text
What instruction is at RIP?
Why did execution arrive here?
Where will execution go next?
```

This makes `RIP` one of the most important registers in debugging.

### Why attackers care

Control-flow vulnerabilities become particularly interesting when attacker-controlled data influences where execution goes.

Conceptually:

```text
Normal:

RIP
 ↓
legitimate instruction
 ↓
legitimate instruction
 ↓
legitimate function


Corrupted control flow:

attacker-controlled corruption
          ↓
      control data
          ↓
         RIP
          ↓
unexpected execution
```

This is the foundation behind many forms of control-flow exploitation.

## 2.5 Tracking the three together

Consider:

```text
RSP = 0x7fffffffe000
RIP = 0x401156
RAX = 0x2a
```

You can immediately form a basic CPU-state picture:

```text
RIP
 ↓
program is executing around 0x401156

RSP
 ↓
current stack top is around 0x7fffffffe000

RAX
 ↓
currently contains 42
```

During debugging, repeatedly observing these values lets you reconstruct what the CPU is doing.

## 2.6 A realistic attacker workflow

An attacker analyzing a suspicious or vulnerable executable might work through this sequence:

1. Identify whether the binary is 32-bit or 64-bit.
2. Determine the architecture-specific registers.
3. Disassemble the interesting function.
4. Identify instructions manipulating `RAX`.
5. Identify stack manipulation involving `RSP`.
6. Locate branches and function calls involving `RIP`.
7. Execute the program under a debugger.
8. Stop at interesting instructions.
9. Record register values.
10. Trace attacker-controlled input into registers or memory.
11. Determine whether that input influences data or control flow.
12. Use the resulting understanding to investigate the next exploitation primitive.

The goal is not to memorize register names.

The goal is to answer:

> **"What value is here, why is it here, and where will the CPU use it next?"**

## 2.7 Dead-end finding vs high-value finding

### Dead-end finding

```text
RAX = 0x7
RAX = 0x8
RAX = 0x9
```

You observe normal arithmetic.

Nothing indicates attacker influence or interesting control flow.

It is useful for learning, but probably not an exploitation primitive.

### High-value finding

Suppose debugging reveals:

```text
RAX = 0x4141414141414141
```

and the value came directly from attacker-controlled input.

That is much more interesting.

The analyst now asks:

```text
Input
 ↓
memory
 ↓
RAX
 ↓
pointer / comparison / indirect operation?
```

If that value subsequently becomes part of an instruction target or memory address, the finding may become highly significant.

The value itself does not automatically mean "exploitable." **Context determines its security significance.**

## 2.8 Pivots

Registers provide pivots into other areas of binary analysis.

For example:

```text
RIP analysis
    ↓
Control flow
    ↓
Branches / calls / returns
```

```text
RSP analysis
    ↓
Stack
    ↓
Local variables / saved state
    ↓
Stack corruption
```

```text
RAX analysis
    ↓
Data flow
    ↓
Pointers / calculations / return values
```

Together:

```text
RAX → What data?
RSP → Where is the stack?
RIP → Where is execution?
```

That three-question model is extremely useful when first opening a binary in GDB.

# Section 3 — Core concepts and terminology

| Term                     | Meaning                                                                                          |
| ------------------------ | ------------------------------------------------------------------------------------------------ |
| Register                 | Very fast CPU storage used during instruction execution.                                         |
| General-purpose register | Register usable for many types of operations rather than one specialized purpose.                |
| RAX                      | 64-bit x86-64 general-purpose register commonly used for calculations and return values.         |
| EAX                      | Lower 32 bits of RAX; also the accumulator register name in 32-bit x86.                          |
| RSP                      | 64-bit x86-64 stack pointer.                                                                     |
| ESP                      | 32-bit x86 stack pointer.                                                                        |
| RIP                      | 64-bit x86-64 instruction pointer.                                                               |
| EIP                      | 32-bit x86 instruction pointer.                                                                  |
| Stack Pointer            | Register identifying the current top of the stack.                                               |
| Instruction Pointer      | Register identifying the current execution location.                                             |
| Accumulator              | Traditional term for a register used heavily for arithmetic/results; associated with AX/EAX/RAX. |
| Register State           | Values currently contained in CPU registers.                                                     |
| Data Flow                | Movement and transformation of values through instructions.                                      |
| Control Flow             | Movement of execution from one instruction to another.                                           |
| Pointer                  | A value representing a memory address.                                                           |
| 64-bit                   | Architecture/register width commonly used by modern x86-64 systems.                              |
| 32-bit                   | Older x86 execution model using 32-bit registers and addresses.                                  |
| Zero Extension           | Filling upper bits with zeros when a smaller unsigned value is written into a larger register.   |

### Register mapping

| 32-bit x86 | 64-bit x86-64 | Primary relevance               |
| ---------- | ------------- | ------------------------------- |
| EAX        | RAX           | Data, arithmetic, return values |
| ESP        | RSP           | Stack location                  |
| EIP        | RIP           | Instruction location            |

A useful relationship is:

```text
RAX
└── EAX
    └── AX
        ├── AH
        └── AL
```

The same general idea exists for many x86 registers.

One important x86-64 behavior:

```text
mov eax, 0x12345678
```

writes the value into `EAX` and clears the upper 32 bits of `RAX`.

Therefore:

```text
RAX = 0x0000000012345678
```

This matters when analyzing 64-bit assembly because operations on subregisters can affect the full register state.

# Section 4 — Tools and commands

| Tool   | Command               | What it finds/shows           | When to use it                          |
| ------ | --------------------- | ----------------------------- | --------------------------------------- |
| `file` | `file ./program`      | 32/64-bit architecture        | Determine register model                |
| `gdb`  | `gdb ./program`       | Interactive debugging         | Inspect execution                       |
| GDB    | `info registers`      | All important register values | Inspect CPU state                       |
| GDB    | `p/x $rax`            | RAX in hexadecimal            | Analyze data/pointers                   |
| GDB    | `p/x $rsp`            | RSP in hexadecimal            | Locate stack                            |
| GDB    | `p/x $rip`            | RIP in hexadecimal            | Locate execution                        |
| GDB    | `x/16gx $rsp`         | Stack memory around RSP       | Inspect stack contents                  |
| GDB    | `x/i $rip`            | Instruction at RIP            | Identify current instruction            |
| GDB    | `si`                  | Execute one instruction       | Track register changes                  |
| GDB    | `ni`                  | Step over function calls      | Follow execution without entering calls |
| GDB    | `disassemble /m main` | Source + assembly             | Correlate C and assembly                |

Example:

```bash
$ file ./program
program: ELF 64-bit LSB pie executable, x86-64
```

Interpretation:

```text
64-bit x86-64 → use RAX/RSP/RIP
```

Example:

```gdb
(gdb) info registers
rax  0x2a
rsp  0x7fffffffe000
rip  0x55555555515a
```

Interpretation:

```text
RAX = 42
RSP = current stack location
RIP = current instruction location
```

Example:

```gdb
(gdb) p/x $rax
$1 = 0x2a
```

`RAX` contains hexadecimal `0x2a`, which is decimal `42`.

Example:

```gdb
(gdb) x/i $rip
=> 0x55555555515a <main+17>: add eax,0x1
```

The CPU is at an instruction that adds `1` to `EAX`.

Example:

```gdb
(gdb) x/8gx $rsp
0x7fffffffe000: 0x0000000000000000  0x00007fffffffe120
0x7fffffffe010: 0x0000555555555160  0x000000000000002a
```

This displays memory around the stack pointer.

Do not assume every value is a specific stack object merely from its appearance. Use the surrounding assembly and debugger information to determine what each value represents.

Example:

```gdb
(gdb) si
(gdb) info registers rax rip rsp
rax            0x2b
rsp            0x7fffffffe000
rip            0x55555555515d
```

After one instruction:

```text
RAX: 0x2a → 0x2b
RIP: moved to the next instruction
RSP: unchanged
```

That gives you an immediate execution-state transition.

# Section 5 — Defender detection

* Register changes themselves generally do **not** appear in ordinary operating-system logs; defenders detect the process behavior that produces suspicious execution.
* Debuggers, EDR sensors, crash telemetry, and endpoint instrumentation can expose unusual process execution and memory behavior.
* Exploitation attempts may produce abnormal instruction-pointer behavior, crashes, access violations, or execution from unexpected memory regions.
* Modern defenses can monitor suspicious memory permissions and control-flow behavior rather than attempting to log every CPU register transition.
* Defenders commonly miss the difference between a **strange register value** and an actual security event; a register can legitimately contain arbitrary-looking data.
* Skilled operators reduce unnecessary telemetry by minimizing noisy debugging and tooling during operations, but legitimate security monitoring can still detect the resulting process behavior.
* For defenders, the valuable question is usually not "Was RAX unusual?" but "Why did this process execute this code path with this memory state?"

# Section 6 — Lab task

**Platform:** Local Kali Linux VM targeting a locally compiled x86-64 C program.

**Objective:** Use GDB to observe `RAX`, `RSP`, and `RIP` changing during real instruction execution.

**Steps:**

1. Create a small C program containing a function that performs an integer calculation and returns the result.
2. Compile it with debugging symbols and optimization disabled so the generated instructions are easy to follow.
3. Confirm that the executable is x86-64.
4. Open the executable in GDB and break at the target function.
5. Record the initial `RAX`, `RSP`, and `RIP` values.
6. Display the instruction referenced by `RIP`.
7. Single-step through the function while recording changes to `RAX` and `RIP`.
8. Inspect memory around `RSP` and identify the stack region.
9. Compare the observed register transitions against the function's disassembly.

**Expected output:**

You should be able to produce an observation similar to:

```text
Before instruction:
RAX = 0x2a
RSP = 0x7fffffffe000
RIP = 0x55555555515a

After instruction:
RAX = 0x2b
RSP = 0x7fffffffe000
RIP = 0x55555555515d
```

You should correctly explain:

```text
RAX changed → instruction modified the value
RSP unchanged → stack pointer did not move
RIP changed → execution advanced
```

**Git artifact:**

```text
cpu-registers/
├── README.md
├── src/
│   └── register_demo.c
├── disassembly/
│   └── register_demo.asm
└── notes/
    └── gdb-register-trace.md
```

```bash
git add cpu-registers/
git commit -m "Add CPU register tracing lab"
```

# Section 7 — Common mistakes

1. **Thinking RAX always means "accumulator" and nothing else** → Modern compilers use it for many purposes → Determine its role from the current instruction and calling convention.

2. **Confusing RSP with RIP** → One identifies stack state while the other identifies execution state → When debugging, explicitly ask "stack or instruction?"

3. **Assuming every value in a register is an integer** → Registers frequently contain pointers and addresses → Interpret values according to how subsequent instructions use them.

4. **Ignoring hexadecimal representation** → Addresses and binary-analysis values are normally shown in hexadecimal → Become comfortable converting between hexadecimal and decimal when needed.

5. **Assuming a strange RAX value proves exploitation** → Arbitrary-looking values are completely normal in CPU registers → Trace where the value came from and what it influences.

6. **Using only `info registers` without examining the instruction** → A register value has little meaning without execution context → Always pair register inspection with the instruction at `RIP`.

7. **Learning only 64-bit registers and forgetting 32-bit terminology** → Security material still frequently uses EAX/ESP/EIP → Know the relationship between the 32-bit and 64-bit names.

# Section 8 — Move-on gate

1. **Run a compiled x86-64 program in GDB, stop at a function, and identify `RAX`, `RSP`, and `RIP` correctly without looking at your notes.**

2. **Single-step through at least 10 instructions and record which instructions modify `RAX` or `RSP`, explaining the change from the actual assembly.**

3. **Given a debugger crash state, identify the instruction referenced by `RIP`, inspect the memory around `RSP`, and determine whether a suspicious register value came from program data or control-flow state without looking at your notes.**
