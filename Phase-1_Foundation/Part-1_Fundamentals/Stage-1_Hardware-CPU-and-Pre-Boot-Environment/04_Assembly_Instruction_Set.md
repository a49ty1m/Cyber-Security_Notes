# Assembly Instruction Set — MOV, PUSH, POP, CALL, JMP

**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → Instruction Sets

# Section 1 — What it is and where it sits

**Assembly language** is a human-readable representation of machine instructions executed by a CPU. Instead of working with raw bytes, assembly lets you reason about operations such as moving values between registers and memory, manipulating the stack, calling functions, and changing control flow.

For offensive security, you need to read these instructions fluently enough to follow **data flow and control flow** inside binaries. `MOV` tells you where data goes, `PUSH`/`POP` reveal stack manipulation, `CALL` reveals function transitions, and `JMP` reveals control-flow changes.

```text
C / C++ source
     ↓
Compiler
     ↓
Assembly
     ↓
Machine-code bytes
     ↓
CPU
     ↓
MOV / PUSH / POP / CALL / JMP
```

Attack-chain placement:

```text
CPU Fundamentals
      ↓
Registers
      ↓
x86 vs x64
      ↓
Assembly Instructions ← THIS TOPIC
      ↓
Binary / Memory Analysis
      ↓
Reverse Engineering
      ↓
Exploitation
```

If you skip assembly, tools such as GDB, `objdump`, and Ghidra become largely black boxes. You may recognize a vulnerability but struggle to understand exactly how input reaches memory or control flow.

This builds directly on registers and architecture: you now learn how instructions manipulate **RAX/RSP/RIP** and their x86 equivalents.

# Section 2 — How attackers actually use this

## 2.1 What attackers look for

When reversing a binary, attackers primarily follow two things:

```text
Data flow:
Where does this value come from?
Where does it go?

Control flow:
Which instruction executes next?
Why?
```

The five instructions in this topic map naturally to those questions:

| Instruction | Main question                         |
| ----------- | ------------------------------------- |
| `MOV`       | Where is data being transferred?      |
| `PUSH`      | What is being placed onto the stack?  |
| `POP`       | What is being removed from the stack? |
| `CALL`      | Which function is execution entering? |
| `JMP`       | Where is execution being redirected?  |

## 2.2 MOV — follow data

`MOV` copies data from a source to a destination.

Conceptually:

```text
MOV destination, source
```

Example:

```text
MOV RAX, RBX
```

means:

```text
RAX ← RBX
```

It does **not** mean that the original `RBX` value disappears.

An attacker might see:

```text
MOV RAX, [RBP-0x20]
```

and infer:

```text
stack memory
     ↓
[RBP-0x20]
     ↓
RAX
```

This immediately identifies a data-flow relationship.

Memory can also be written:

```text
MOV [RBP-0x20], RAX
```

Now the direction is:

```text
RAX
 ↓
stack memory
```

## 2.3 PUSH and POP — follow the stack

`PUSH` places a value onto the stack.

`POP` removes a value from the stack.

Conceptually:

```text
PUSH RAX
    ↓
RSP changes
    ↓
RAX value stored on stack
```

and:

```text
POP RBX
    ↓
value taken from stack
    ↓
RSP changes
    ↓
RBX receives value
```

This is particularly important when analyzing function prologues and epilogues.

A common function beginning is:

```text
PUSH RBP
MOV  RBP, RSP
```

Conceptually:

```text
old RBP
   ↓
saved on stack

RBP
   ↓
updated to current RSP
   ↓
new stack frame
```

An attacker examining stack corruption therefore needs to understand exactly how instructions manipulate `RSP`.

## 2.4 CALL — follow function transitions

`CALL` transfers execution to another location while preserving a return location.

Conceptually:

```text
CALL function
     ↓
save return location
     ↓
RIP → function
     ↓
execute function
```

Then:

```text
RET
 ↓
return to saved location
```

For reverse engineering, `CALL` tells you:

> "The current function is invoking another piece of code."

For example:

```text
main
 ↓
CALL authenticate
 ↓
authenticate
```

An attacker can then investigate:

* what arguments were supplied
* what registers contain those arguments
* what memory the function accesses
* what value it returns
* what conditions control its behavior

## 2.5 JMP — follow control flow

`JMP` directly changes execution to another location.

Conceptually:

```text
JMP target
    ↓
RIP → target
```

Unlike `CALL`, a normal `JMP` does not establish a normal function-return relationship.

Conditional jumps are especially important:

```text
CMP ...
JE  target
```

Conceptually:

```text
comparison
    ↓
CPU flags
    ↓
condition satisfied?
   / \
 yes  no
 ↓     ↓
JE    continue
 ↓
target
```

During reverse engineering, conditional jumps frequently reveal:

* authentication decisions
* bounds checks
* error handling
* input validation
* loops
* privilege checks

## 2.6 Realistic attacker workflow

A realistic binary-analysis workflow is:

1. Identify the architecture.
2. Disassemble the target function.
3. Locate `MOV` instructions involving user-controlled data.
4. Track whether that data moves into registers or memory.
5. Identify `PUSH`/`POP` operations affecting the stack.
6. Locate `CALL` instructions to understand function transitions.
7. Follow `JMP` and conditional branches to reconstruct control flow.
8. Debug the function and verify assumptions against actual register state.
9. Determine whether attacker-controlled data reaches a security-sensitive operation.
10. Pivot into memory-corruption, authentication, or control-flow analysis if appropriate.

The attacker is essentially building a graph:

```text
Input
  ↓
MOV
  ↓
Register
  ↓
MOV
  ↓
Memory
  ↓
CALL
  ↓
Validation
  ↓
JMP
 ├── success
 └── failure
```

## 2.7 Dead-end finding vs high-value finding

### Dead-end

```text
MOV EAX, 5
MOV EBX, 10
ADD EAX, EBX
```

This demonstrates normal arithmetic and contains no obvious attacker-controlled input or interesting control flow.

### High-value

```text
MOV RAX, [user-controlled-location]
MOV [RBP-0x20], RAX
CALL sensitive_function
```

This deserves investigation because attacker-controlled data appears to flow into memory and subsequently into a security-sensitive function.

The important point is not the presence of `MOV` or `CALL alone.

It is the **relationship between instructions**.

## 2.8 Pivots

These instructions provide direct pivots into deeper analysis:

```text
MOV
 ↓
Data-flow analysis
 ↓
Input → register → memory
```

```text
PUSH / POP
 ↓
Stack analysis
 ↓
Stack frame / saved state
```

```text
CALL
 ↓
Function analysis
 ↓
Arguments / return values
```

```text
JMP
 ↓
Control-flow analysis
 ↓
Branches / loops / validation
```

Together they let you read a binary as an execution story rather than a list of hexadecimal addresses.

# Section 3 — Core concepts and terminology

| Term                | Meaning                                                            |
| ------------------- | ------------------------------------------------------------------ |
| Assembly            | Human-readable representation of machine instructions.             |
| Instruction         | CPU operation encoded as machine code.                             |
| Operand             | Value, register, or memory location used by an instruction.        |
| Source Operand      | Where an instruction obtains its input.                            |
| Destination Operand | Where an instruction places its result.                            |
| `MOV`               | Copies data between registers, memory, and/or immediate values.    |
| `PUSH`              | Places a value onto the stack.                                     |
| `POP`               | Removes a value from the stack into a destination.                 |
| `CALL`              | Transfers execution to a function and preserves a return location. |
| `JMP`               | Transfers execution directly to another instruction.               |
| Conditional Jump    | Branch executed only when a condition is satisfied.                |
| Stack               | Memory region used for function-related temporary state.           |
| Control Flow        | Sequence of instructions executed by a program.                    |
| Data Flow           | Movement of values between registers, memory, and instructions.    |
| Immediate           | Constant encoded directly in an instruction.                       |
| Dereference         | Accessing memory using an address/pointer.                         |
| Label               | Human-readable name representing an instruction location.          |
| Disassembly         | Converting machine-code bytes into assembly representation.        |

### Core instruction map

| Instruction | Example         | Conceptual effect                         |
| ----------- | --------------- | ----------------------------------------- |
| `MOV`       | `mov rax, rbx`  | `RAX ← RBX`                               |
| `PUSH`      | `push rax`      | Put RAX on stack                          |
| `POP`       | `pop rbx`       | Remove stack value into RBX               |
| `CALL`      | `call function` | Enter function and save return location   |
| `JMP`       | `jmp target`    | Redirect execution                        |
| `JE`        | `je target`     | Jump if equality condition is satisfied   |
| `JNE`       | `jne target`    | Jump if inequality condition is satisfied |

# Section 4 — Tools and commands

| Tool      | Command                                 | What it finds/shows     | When to use it               |
| --------- | --------------------------------------- | ----------------------- | ---------------------------- |
| `objdump` | `objdump -d -M intel ./program`         | Full disassembly        | Static assembly analysis     |
| `objdump` | `objdump -d -M intel ./program \| less` | Navigable disassembly   | Large binaries               |
| `gdb`     | `gdb ./program`                         | Interactive debugging   | Dynamic analysis             |
| GDB       | `disassemble main`                      | Function assembly       | Analyze a function           |
| GDB       | `x/i $rip`                              | Current instruction     | See what executes next       |
| GDB       | `si`                                    | Execute one instruction | Observe instruction behavior |
| GDB       | `info registers`                        | Register state          | Track operands/results       |
| GDB       | `x/16gx $rsp`                           | Stack contents          | Analyze `PUSH`/`POP` effects |

Example:

```bash
$ objdump -d -M intel ./program

0000000000001139 <main>:
    1139: 55              push   rbp
    113a: 48 89 e5        mov    rbp,rsp
    113d: b8 05 00 00 00  mov    eax,0x5
    1142: 5d              pop    rbp
    1143: c3              ret
```

Interpretation:

```text
push rbp → save RBP
mov rbp,rsp → establish frame
mov eax,5 → place 5 into EAX
pop rbp → restore RBP
ret → return
```

Example:

```gdb
(gdb) disassemble main
...
0x... <+0>:  push   rbp
0x... <+1>:  mov    rbp,rsp
0x... <+4>:  mov    eax,0x5
```

This gives the function's instruction sequence.

Example:

```gdb
(gdb) x/i $rip
=> 0x555555555139 <main+0>: push rbp
```

`RIP` currently points at `push rbp`.

Example:

```gdb
(gdb) si
(gdb) x/i $rip
=> 0x55555555513a <main+1>: mov rbp,rsp
```

One machine instruction has executed and `RIP` advanced.

Example:

```gdb
(gdb) info registers rbp rsp
rbp  0x7fffffffe100
rsp  0x7fffffffe0f0
```

After a `PUSH`, `RSP` typically changes because the stack grows.

Example:

```gdb
(gdb) x/8gx $rsp
0x7fffffffe0f0: 0x00007fffffffe100 ...
```

This allows you to inspect the stack value affected by stack operations.

# Section 5 — Defender detection

* Normal assembly execution produces no individual security event; defenders observe the process behavior resulting from the instructions.
* EDR telemetry can identify suspicious process creation, unusual memory execution, crashes, and abnormal control-flow behavior.
* Exploitation involving corrupted control flow may produce unexpected instruction-pointer destinations or execution from unusual memory regions.
* Debugging and reverse-engineering activity against a live target can generate process, file, or endpoint telemetry depending on the environment.
* Defenders commonly miss malicious behavior when they examine instructions individually instead of following the complete data/control-flow chain.
* Skilled operators reduce unnecessary tooling and process activity, but suspicious memory and control-flow behavior can still expose exploitation.
* Static analysis is especially valuable because it can reveal dangerous control-flow paths without executing the sample.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM with GCC and GDB.

**Objective:** Disassemble and dynamically trace a program containing `MOV`, `PUSH`, `POP`, `CALL`, and `JMP` instructions.

**Steps:**

1. Create a C program containing two functions, an integer calculation, and a conditional branch.
2. Compile it with debugging symbols and optimization disabled.
3. Disassemble the binary using Intel syntax.
4. Locate the function prologue and identify `PUSH` and `MOV`.
5. Locate the function call and identify the `CALL` instruction.
6. Locate the conditional branch and identify the relevant jump instruction.
7. Open the binary in GDB and break before the target function executes.
8. Single-step through the instructions while recording `RIP`, `RSP`, and `RAX`.
9. Compare the dynamic execution sequence against the static disassembly.

**Expected output:**

You should be able to identify a sequence resembling:

```text
PUSH RBP
MOV  RBP,RSP
MOV  EAX,...
CALL function
CMP  ...
JE   target
POP  RBP
RET
```

and explain what each instruction does without relying on the compiler source.

**Git artifact:**

```text
assembly-instructions/
├── README.md
├── src/
│   └── instruction_demo.c
├── disassembly/
│   └── instruction_demo.asm
└── notes/
    └── instruction-trace.md
```

```bash
git add assembly-instructions/
git commit -m "Add x86 assembly instruction tracing lab"
```

# Section 7 — Common mistakes

1. **Reading `MOV` as a generic "move" without checking operands** → Source/destination direction determines the data flow → Always read `MOV destination, source`.

2. **Assuming `PUSH` and `POP` only manipulate data** → They also modify the stack pointer → Track `RSP/ESP` whenever analyzing stack instructions.

3. **Treating `CALL` as simply a jump** → `CALL` preserves return information and participates in function control flow → Analyze it together with the eventual return.

4. **Assuming every `JMP` is malicious or suspicious** → Compilers use jumps constantly for legitimate branches and loops → Determine why execution is being redirected.

5. **Ignoring operand size** → `eax`, `rax`, and memory operands can access different widths → Always inspect the operand size.

6. **Reading assembly instruction-by-instruction without following data** → Individual instructions are less useful than the relationships between them → Track important values from input through registers and memory.

7. **Memorizing instructions instead of debugging them** → Recognition without execution knowledge breaks down during real reverse engineering → Single-step the instructions and observe register/stack changes.

# Section 8 — Move-on gate

1. **Disassemble an unfamiliar x86-64 function and identify every `MOV`, `PUSH`, `POP`, `CALL`, and `JMP` instruction without looking at your notes.**

2. **Single-step through a function in GDB and correctly explain the effect of each of those five instruction types on `RIP`, `RSP`, registers, memory, or control flow.**

3. **Take a 20–30 instruction disassembly and trace one attacker-controlled value from its input location through `MOV` instructions into a register/memory location, then identify the `CALL` or `JMP` that determines what happens next without looking at your notes.**
