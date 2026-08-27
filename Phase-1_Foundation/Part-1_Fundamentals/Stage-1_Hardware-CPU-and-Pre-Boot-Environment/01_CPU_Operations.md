# CPU Operations: Fetch-Decode-Execute

**Roadmap:** Part 1 → Fundamentals → Stage 1: Hardware, CPU & Pre-Boot Environment

# Section 1 — What it is and where it sits

The **Fetch-Decode-Execute cycle** is the fundamental process a CPU uses to execute machine instructions. The CPU repeatedly retrieves an instruction from memory, determines what that instruction means, performs the operation, and advances to the next instruction. This is the lowest-level bridge between compiled code and actual processor execution.

For offensive security, this matters because malware, shellcode, debuggers, exploit payloads, and reverse-engineered binaries eventually reduce to **CPU instructions manipulating registers and memory**.

```text
Hardware
   ↓
CPU registers + memory
   ↓
Fetch
   ↓
Decode
   ↓
Execute
   ↓
Update registers / memory / instruction pointer
   ↓
Fetch next instruction
```

Roadmap placement:

```text
Hardware
  ↓
CPU architecture
  ↓
Fetch–Decode–Execute  ← This topic
  ↓
Memory + registers
  ↓
Operating system
  ↓
Processes
  ↓
Machine code / assembly
  ↓
Programs and exploits
```

If you skip this, assembly and debugger output become memorization exercises. You may recognize instructions without understanding what the processor is actually doing.

This connects hardware fundamentals to the next stages: **memory, operating systems, processes, and eventually binary exploitation and reverse engineering**.

# Section 2 — How attackers actually use this

## 2.1 Turning code into CPU operations

An attacker ultimately needs the target CPU to perform operations such as:

```text
load data
compare values
calculate addresses
modify memory
change control flow
call functions
return from functions
```

A high-level statement such as:

```c
x = x + 1;
```

eventually becomes machine instructions.

Conceptually:

```text
C source
   ↓
Compiler
   ↓
Assembly
   ↓
Machine code
   ↓
CPU
   ↓
Fetch → Decode → Execute
```

The CPU does not understand C, Python, Bash, or PowerShell. It executes instructions encoded for its architecture.

---

## 2.2 What an attacker cares about

When analyzing or constructing low-level payloads, important CPU state includes:

* **Instruction Pointer (`RIP`)** — address of the next instruction.
* **Stack Pointer (`RSP`)** — location of the current stack.
* **Base Pointer (`RBP`)** — commonly used to reference stack-frame data.
* **General-purpose registers** — hold values, addresses, arguments, and temporary data.
* **Flags** — record results of operations such as comparisons.
* **Memory** — contains instructions and data.

A simplified execution might look like:

```text
RIP = 0x401126

Memory[0x401126]
       ↓
instruction: mov eax, 5
       ↓
CPU decodes instruction
       ↓
EAX becomes 5
       ↓
RIP advances
       ↓
next instruction executes
```

The attacker doesn't necessarily need to manipulate the CPU directly. Exploitation often works by causing the processor to execute instructions in an unintended sequence.

---

## 2.3 Control-flow manipulation

One of the most important offensive applications is understanding **control flow**.

Normally:

```text
Instruction A
    ↓
Instruction B
    ↓
Instruction C
    ↓
Instruction D
```

A vulnerability may allow an attacker to influence where execution continues:

```text
Instruction A
    ↓
unexpected control-flow change
    ↓
attacker-controlled location
    ↓
different instructions execute
```

This is why understanding `RIP`, stack contents, `call`, `ret`, `jmp`, and conditional branches becomes essential later for exploit development.

---

## 2.4 Dead-end vs high-value finding

### Dead-end finding

You inspect a binary and find:

```text
mov eax, 0
```

By itself, this tells you very little.

It is simply moving the value `0` into a register.

### High-value finding

You discover that a vulnerable function's execution reaches:

```text
ret
```

while the stack contains an unexpected address that controls where execution continues.

That is much more interesting because it may indicate **control-flow manipulation**.

The important distinction is:

> Don't treat individual instructions as inherently malicious. Understand how they affect program state and control flow.

---

## 2.5 Where this feeds next

Understanding CPU execution unlocks:

```text
CPU instructions
    ↓
Assembly analysis
    ↓
Registers + stack
    ↓
Function calling conventions
    ↓
Memory corruption
    ↓
Control-flow hijacking
    ↓
ROP / shellcode / exploit development
```

For malware analysis, the same knowledge lets you understand what a suspicious binary actually does instead of trusting its high-level appearance.

# Section 3 — Core concepts and terminology

| Term         | Meaning                                                              |
| ------------ | -------------------------------------------------------------------- |
| Instruction  | A machine-level operation executed by the CPU.                       |
| Fetch        | Retrieving the next instruction from memory.                         |
| Decode       | Determining what the instruction means and what operands it uses.    |
| Execute      | Performing the operation requested by the instruction.               |
| `RIP`        | x86-64 instruction pointer containing the current execution address. |
| `RSP`        | x86-64 stack pointer.                                                |
| `RBP`        | Commonly used as a stack-frame base pointer.                         |
| Register     | Small, extremely fast storage location inside the CPU.               |
| Operand      | Data or location an instruction operates on.                         |
| Opcode       | Encoded portion identifying the CPU operation.                       |
| Memory       | Addressable storage containing instructions and data.                |
| Flag         | CPU state bit representing conditions such as zero, carry, or sign.  |
| Control flow | The order in which instructions execute.                             |
| Branch       | Instruction that changes normal sequential execution.                |
| `call`       | Transfers execution to a function while saving a return address.     |
| `ret`        | Returns execution using a stored return address.                     |

### Common instruction categories

| Category           | Examples                          | Purpose                   |
| ------------------ | --------------------------------- | ------------------------- |
| Data movement      | `mov`, `lea`, `push`, `pop`       | Move or calculate data    |
| Arithmetic         | `add`, `sub`, `inc`, `dec`        | Perform calculations      |
| Logic              | `and`, `or`, `xor`, `not`         | Bitwise operations        |
| Comparison         | `cmp`, `test`                     | Set flags based on values |
| Control flow       | `jmp`, `je`, `jne`, `call`, `ret` | Change execution          |
| System interaction | `syscall`                         | Request kernel services   |

# Section 4 — Tools and commands

| Tool      | Command                | What it finds/shows                        | When to use it                           |
| --------- | ---------------------- | ------------------------------------------ | ---------------------------------------- |
| `objdump` | `objdump -d ./program` | Disassembled machine code                  | Static instruction analysis              |
| `readelf` | `readelf -h ./program` | ELF architecture/header information        | Identify binary architecture             |
| `file`    | `file ./program`       | Binary type and architecture               | First inspection                         |
| GDB       | `gdb ./program`        | Registers, instructions, memory, execution | Dynamic instruction analysis             |
| GDB       | `disassemble main`     | Assembly for a function                    | Understand function control flow         |
| GDB       | `info registers`       | Current CPU register state                 | Inspect execution state                  |
| GDB       | `x/i $rip`             | Instruction at current RIP                 | Observe the next CPU instruction         |
| GDB       | `si`                   | Execute one machine instruction            | Observe Fetch/Decode/Execute progression |

### `file`

```bash
file ./program
```

Example:

```text
ELF 64-bit LSB pie executable, x86-64
```

This tells you the binary is an ELF executable targeting x86-64.

### `objdump`

```bash
objdump -d ./program
```

Example:

```text
401126:  b8 05 00 00 00    mov    $0x5,%eax
```

The bytes are the machine-code representation; the right side is the disassembled instruction.

### GDB

```bash
gdb ./program
```

Then:

```gdb
disassemble main
```

This shows the instructions making up `main`.

### Registers

```gdb
info registers
```

Example:

```text
rax    0x5
rbx    0x0
rip    0x401126
rsp    0x7fffffffe120
```

The most important value here is `rip`: it tells you where execution currently is.

### Current instruction

```gdb
x/i $rip
```

Example:

```text
=> 0x401126 <main+10>: mov eax,0x5
```

### Single instruction execution

```gdb
si
```

After executing it, inspect:

```gdb
info registers
x/i $rip
```

You can directly observe the CPU state changing from one instruction to the next.

# Section 5 — Defender detection

* **EDR telemetry** can detect suspicious process execution, memory permissions, injected code, and unusual control-flow behavior.
* **Debugger/reverse-engineering analysis** can reveal unexpected branches, calls, returns, and memory modifications.
* **Memory forensics** can identify executable memory regions that do not correspond to legitimate loaded modules.
* **Behavioral detection** is generally more useful than flagging individual CPU instructions because instructions such as `xor`, `mov`, and `jmp` are normal everywhere.
* **Control-flow anomalies** become important when execution moves into unexpected memory regions or attacker-controlled content.
* Defenders commonly miss the distinction between **legitimate low-level instructions and suspicious execution context**.
* Skilled operators reduce visibility by using legitimate processes, existing executable memory, indirect execution, and techniques designed to blend into normal process behavior.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM with a small x86-64 test binary.

**Objective:** Use GDB to observe the CPU's instruction pointer and register state changing as individual machine instructions execute.

### Steps

1. Create or use a simple x86-64 ELF test program containing basic arithmetic.
2. Compile it with debugging symbols so GDB can associate instructions with the program.
3. Confirm the resulting binary architecture with `file`.
4. Open the binary with GDB.
5. Disassemble `main` and identify its instructions.
6. Set a breakpoint at `main`.
7. Start execution and inspect `RIP` with `info registers`.
8. Use `x/i $rip` followed by `si` repeatedly.
9. After each instruction, observe which register or memory value changed.
10. Explain the execution sequence in terms of **fetch → decode → execute → state update**.

**Expected output:**

You should be able to point at an instruction such as:

```text
mov eax,0x5
```

and demonstrate that after executing it:

```text
EAX = 5
```

You should also be able to show that `RIP` moves to the next instruction.

**Git artifact:**

```text
cpu-operations/
├── README.md
├── lab/
│   ├── source.c
│   └── notes.md
└── screenshots/
    ├── disassembly.png
    └── registers.png
```

```bash
git add cpu-operations/
git commit -m "lab: trace fetch decode execute cycle"
```

# Section 7 — Common mistakes

1. **Memorizing instructions without tracking state** → You recognize `mov` but don't know what changed. → Track `RIP`, registers, and memory after each instruction.

2. **Thinking the CPU executes source code** → CPUs execute machine instructions, not C or Python. → Trace the compilation path from source to machine code.

3. **Treating every `jmp` or `ret` as malicious** → These are normal instructions used constantly by legitimate programs. → Analyze where control flow goes and why.

4. **Ignoring `RIP`** → You lose track of where execution actually is. → Check `$rip` whenever stepping through instructions.

5. **Ignoring architecture** → x86, x86-64, ARM64, and other architectures use different instruction sets and registers. → Identify the binary architecture first.

6. **Using static disassembly alone** → You may misunderstand how instructions behave with actual runtime values. → Combine `objdump` with GDB when learning execution.

# Section 8 — Move-on gate

1. **Run a test ELF through GDB and single-step at least 10 instructions, identifying the value of `RIP` and the instruction being executed at every step without looking at notes.**

2. **Disassemble a function with `objdump` or GDB and correctly identify at least five data-movement, arithmetic, comparison, or control-flow instructions and explain their effect on CPU state.**

3. **Set a breakpoint, inspect the registers, execute one instruction with `si`, and correctly identify which register or memory value changed and why.**
