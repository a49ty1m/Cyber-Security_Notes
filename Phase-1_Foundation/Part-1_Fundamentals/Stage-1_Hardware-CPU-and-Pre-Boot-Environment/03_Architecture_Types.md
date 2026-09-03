# x86 vs x64 Architecture

**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → Architecture Types

# Section 1 — What it is and where it sits

**x86** generally refers to the 32-bit Intel/AMD architecture, while **x86-64/x64** refers to its 64-bit extension. The most important practical difference is the width of general-purpose registers and pointers: x86 commonly uses 32-bit registers and addresses, while x64 uses 64-bit registers and addresses. This changes how instructions access memory, how functions receive arguments, how stacks are organized, and how binaries are exploited.

For offensive security, architecture identification must happen early. A 32-bit binary and a 64-bit binary can use different registers, calling conventions, instruction encodings, stack layouts, and exploitation techniques.

```text
Target binary
     ↓
Identify architecture
     ├── x86 / 32-bit
     │     ├── EAX / ESP / EIP
     │     ├── 32-bit pointers
     │     └── 32-bit calling convention
     │
     └── x86-64 / x64
           ├── RAX / RSP / RIP
           ├── 64-bit registers
           ├── 64-bit pointers
           └── x64 calling convention
```

Attack-chain placement:

```text
CPU Fundamentals
      ↓
Registers
      ↓
Architecture Types ← THIS TOPIC
      ↓
Memory Layout
      ↓
Processes / Stack / Heap
      ↓
Assembly & Debugging
      ↓
Binary Exploitation
```

If you mistake x86 for x64, your register names, pointer sizes, debugger commands, calling-convention assumptions, and exploit reasoning can all be wrong.

This follows the register topic: you learned **EAX/RAX, ESP/RSP, EIP/RIP**; now you need to understand why those register families differ and how that affects memory operations.

# Section 2 — How attackers actually use this

## 2.1 First question: what architecture is the target?

Before reversing or exploiting a binary, an attacker establishes:

```text
Is this:
32-bit x86?
64-bit x86-64?
ARM?
ARM64?
```

For this topic:

```text
x86   → 32-bit
x64   → 64-bit
```

The architecture determines what the attacker should expect:

| Property                            |         x86 |      x86-64 |
| ----------------------------------- | ----------: | ----------: |
| General register width              |      32-bit |      64-bit |
| Typical pointer width               |      32-bit |      64-bit |
| Instruction pointer                 |         EIP |         RIP |
| Stack pointer                       |         ESP |         RSP |
| Common accumulator                  |         EAX |         RAX |
| Address space capability            |     Smaller | Much larger |
| Common modern desktop/server target | Less common | Very common |

The attacker does not assume the architecture from the operating system alone.

A 64-bit Linux system can execute a 32-bit binary.

```text
64-bit Linux
   ├── 64-bit executable
   └── 32-bit executable
```

Therefore:

> **Identify the binary, not merely the operating system.**

## 2.2 Why pointer size matters

Consider a pointer:

```text
0x7fffffffe120
```

On x64, this can be represented naturally as a 64-bit value.

A pointer occupies **8 bytes** in the normal x64 data model.

On 32-bit x86, pointers are normally **4 bytes**.

Conceptually:

```text
x86:
pointer = 4 bytes = 32 bits

x64:
pointer = 8 bytes = 64 bits
```

This changes structures, stack layouts, memory calculations, and binary interfaces.

For example:

```c
char *ptr;
```

The C declaration is identical.

But its size normally differs:

```text
x86:
sizeof(ptr) = 4

x64:
sizeof(ptr) = 8
```

That difference can alter the layout of a structure:

```text
struct Example {
    char flag;
    char *ptr;
};
```

Padding and alignment may cause the complete structure to have a different layout between architectures.

An attacker analyzing memory corruption therefore cannot blindly reuse offsets from a different architecture.

## 2.3 Memory instructions are width-sensitive

A CPU instruction does not simply say:

```text
"read memory"
```

It specifies an operation and operand size.

For example, conceptually:

```text
32-bit memory access:
load 4 bytes

64-bit memory access:
load 8 bytes
```

A simplified x64 example:

```text
mov eax, [address]
```

loads **4 bytes** into `EAX`.

Whereas:

```text
mov rax, [address]
```

loads **8 bytes** into `RAX`.

That distinction is extremely important.

```text
mov eax, [memory]
       ↓
      4 bytes

mov rax, [memory]
       ↓
      8 bytes
```

The architecture is x64 in both examples, but the **operand size** differs.

This is an important correction to a common beginner misconception:

> **64-bit CPU does not mean every memory operation automatically reads 8 bytes.**

Instruction operands determine how much data is accessed.

## 2.4 Registers influence memory addressing

Suppose:

```text
RAX = 0x1000
```

An instruction can use the register as a memory address:

```text
mov rbx, [rax]
```

Conceptually:

```text
RAX
 ↓
0x1000
 ↓
memory at 0x1000
 ↓
8-byte value → RBX
```

In x86-64, addressing expressions can also combine registers and offsets.

Conceptually:

```text
mov rax, [rbp-0x20]
```

means:

```text
effective address =
RBP - 0x20
```

This is fundamental to understanding stack variables.

## 2.5 x86 stack addressing

A typical 32-bit function may use:

```text
EBP
 ↓
stack frame
 ↓
local variables / arguments
```

Conceptually:

```text
higher addresses
┌──────────────────────┐
│ function arguments   │
├──────────────────────┤
│ return information   │
├──────────────────────┤
│ saved EBP            │ ← EBP
├──────────────────────┤
│ local variables      │
└──────────────────────┘
        ↓
       ESP
```

A local variable might conceptually be accessed using:

```text
[ebp-0x10]
```

An argument might be accessed using:

```text
[ebp+0x08]
```

The exact layout depends on the compiler and calling convention.

## 2.6 x64 stack addressing

The same conceptual function on x64 uses:

```text
RBP
 ↓
stack frame
```

and:

```text
RSP
 ↓
current stack position
```

A local variable could appear as:

```text
[rbp-0x10]
```

But x64 calling conventions can differ substantially from classic 32-bit conventions.

For example, under the common System V AMD64 ABI, early integer/pointer arguments are passed through registers rather than being placed entirely on the stack.

That means an attacker reversing a function may encounter:

```text
RDI
RSI
RDX
RCX
R8
R9
```

before looking at stack-based arguments.

This is a major practical difference.

## 2.7 Calling conventions change exploitation reasoning

A simplified comparison:

```text
32-bit x86
arguments
   ↓
often stack-based
   ↓
function
```

Common x86 conventions include:

```text
cdecl
stdcall
fastcall
```

On Linux x64 using System V AMD64:

```text
1st integer/pointer → RDI
2nd                 → RSI
3rd                 → RDX
4th                 → RCX
5th                 → R8
6th                 → R9
return value        → RAX
```

Therefore, when reversing:

```text
function(a, b, c)
```

you might observe:

```text
RDI = a
RSI = b
RDX = c
```

rather than seeing all three values pushed onto the stack.

This matters enormously when tracing attacker-controlled input.

## 2.8 Realistic attacker workflow

A practical binary-analysis workflow looks like this:

```text
1. Obtain binary
       ↓
2. Identify architecture
       ↓
3. Identify executable format
       ↓
4. Disassemble using correct architecture
       ↓
5. Identify calling convention
       ↓
6. Identify register usage
       ↓
7. Trace memory accesses
       ↓
8. Analyze stack / heap layout
       ↓
9. Investigate vulnerability
```

For example, suppose an attacker discovers:

```text
ELF 64-bit x86-64
```

They immediately know to investigate:

```text
RIP
RSP
RBP
RAX
RDI
RSI
RDX
RCX
R8
R9
```

If instead the binary is:

```text
ELF 32-bit Intel 80386
```

they expect:

```text
EIP
ESP
EBP
EAX
EBX
ECX
EDX
```

and different calling-convention behavior.

## 2.9 Dead-end finding vs high-value finding

### Dead-end finding

An attacker discovers:

```text
64-bit executable
```

That information is useful, but by itself it does not reveal a vulnerability.

It simply establishes the analysis environment.

### High-value finding

The attacker discovers that:

```text
attacker-controlled input
        ↓
register
        ↓
memory address calculation
        ↓
memory access
```

For example, a value derived from input eventually participates in an effective address:

```text
RAX = attacker-influenced value

mov rdx, [rax]
```

Now the attacker has a concrete data-flow relationship to investigate.

The architecture determines how that value is represented and how the CPU performs the access.

## 2.10 Pivots

Architecture identification feeds directly into:

```text
x86/x64
   ↓
Register set
   ↓
Calling convention
   ↓
Stack layout
   ↓
Pointer size
   ↓
Memory access width
   ↓
Binary exploitation technique
```

For example:

```text
x86 binary
 ↓
ESP/EIP
 ↓
32-bit pointers
 ↓
stack-based arguments commonly encountered
```

versus:

```text
x64 binary
 ↓
RSP/RIP
 ↓
64-bit pointers
 ↓
register-based argument passing
 ↓
different stack/control-flow assumptions
```

# Section 3 — Core concepts and terminology

| Term               | Meaning                                                                                                   |
| ------------------ | --------------------------------------------------------------------------------------------------------- |
| x86                | 32-bit Intel-compatible architecture in this roadmap context.                                             |
| x64                | Common name for the 64-bit extension of x86; also called x86-64 or AMD64.                                 |
| Word Size          | Natural data width associated with an architecture; x86 commonly uses 32 bits, x64 commonly uses 64 bits. |
| Pointer            | Value representing a memory address.                                                                      |
| Operand Size       | Number of bits/bytes an instruction operates on.                                                          |
| Effective Address  | Final memory address calculated from registers, offsets, and addressing components.                       |
| Address Space      | Set of memory addresses available to a process/architecture.                                              |
| EAX                | 32-bit x86 general-purpose register.                                                                      |
| RAX                | 64-bit x86-64 general-purpose register.                                                                   |
| ESP                | 32-bit x86 stack pointer.                                                                                 |
| RSP                | 64-bit x86-64 stack pointer.                                                                              |
| EIP                | 32-bit x86 instruction pointer.                                                                           |
| RIP                | 64-bit x86-64 instruction pointer.                                                                        |
| EBP                | 32-bit x86 base/frame pointer.                                                                            |
| RBP                | 64-bit x86-64 base/frame pointer.                                                                         |
| Calling Convention | Rules defining how functions receive arguments and return values.                                         |
| ABI                | Application Binary Interface defining binary-level conventions between software components.               |
| Sign Extension     | Extending a smaller signed value while preserving its sign.                                               |
| Zero Extension     | Extending a value by filling higher bits with zeros.                                                      |
| Alignment          | Requirement/preference that data begins at particular address boundaries.                                 |

### Architecture comparison

| Feature                             | x86             | x86-64                                                                              |
| ----------------------------------- | --------------- | ----------------------------------------------------------------------------------- |
| Common register width               | 32 bits         | 64 bits                                                                             |
| Instruction pointer                 | EIP             | RIP                                                                                 |
| Stack pointer                       | ESP             | RSP                                                                                 |
| Accumulator                         | EAX             | RAX                                                                                 |
| Pointer size                        | Usually 4 bytes | Usually 8 bytes                                                                     |
| General-purpose registers           | Fewer           | More                                                                                |
| Common Linux calling convention     | Stack-oriented  | Register-oriented                                                                   |
| Maximum architectural address width | 32-bit          | 64-bit architecture, though implementations use fewer physical/virtual address bits |
| Typical modern desktop/server use   | Legacy          | Dominant                                                                            |

### Memory-instruction comparison

```text
x86:

mov eax, [address]
       ↓
read 4 bytes
       ↓
EAX


x64:

mov eax, [address]
       ↓
read 4 bytes
       ↓
EAX

mov rax, [address]
       ↓
read 8 bytes
       ↓
RAX
```

Notice that **x64 still supports 32-bit operations**.

This is why you may see both:

```text
EAX
```

and:

```text
RAX
```

inside the same x64 binary.

# Section 4 — Tools and commands

| Tool      | Command                         | What it finds/shows             | When to use it                 |
| --------- | ------------------------------- | ------------------------------- | ------------------------------ |
| `file`    | `file ./program`                | Binary architecture             | First identification           |
| `readelf` | `readelf -h ./program`          | ELF class and machine type      | Confirm Linux ELF architecture |
| `objdump` | `objdump -d -M intel ./program` | x86/x64 disassembly             | Static analysis                |
| `objdump` | `objdump -f ./program`          | Architecture/file format        | Quick architecture check       |
| `gdb`     | `gdb ./program`                 | Architecture-aware debugging    | Dynamic analysis               |
| GDB       | `show architecture`             | Debugger architecture           | Confirm GDB interpretation     |
| GDB       | `info registers`                | Architecture-specific registers | Inspect CPU state              |
| GDB       | `x/8gx $rsp`                    | 64-bit stack memory             | x64 stack inspection           |
| GDB       | `x/16wx $esp`                   | 32-bit stack memory             | x86 stack inspection           |

Example:

```bash
$ file ./program32
program32: ELF 32-bit LSB executable, Intel 80386

$ file ./program64
program64: ELF 64-bit LSB pie executable, x86-64
```

The first binary should be analyzed using the x86 register model.

The second uses the x86-64 register model.

Example:

```bash
$ readelf -h ./program64 | grep -E 'Class|Machine'
Class:                             ELF64
Machine:                           Advanced Micro Devices X86-64
```

Interpretation:

```text
ELF64
  ↓
64-bit ELF

X86-64
  ↓
x86-64 instruction architecture
```

Example:

```bash
$ objdump -f ./program32
architecture: i386
```

This identifies the executable as 32-bit x86.

Example:

```bash
$ objdump -d -M intel ./program64

0000000000001129 <main>:
    1129: 55                    push   rbp
    112a: 48 89 e5              mov    rbp,rsp
    112d: 48 83 ec 10           sub    rsp,0x10
```

Notice:

```text
rbp
rsp
```

These are x64 registers.

Example x86-style output:

```text
08049156 <main>:
 8049156: 55                    push   ebp
 8049157: 89 e5                 mov    ebp,esp
 8049159: 83 ec 10              sub    esp,0x10
```

Here:

```text
ebp
esp
```

indicate 32-bit x86 register usage.

Example:

```gdb
(gdb) show architecture
The target architecture is set to "auto".
The target is assumed to be i386:x86-64.
```

GDB has identified the target as x86-64.

Example x64 register inspection:

```gdb
(gdb) info registers rax rsp rip
rax            0x2a
rsp            0x7fffffffe000
rip            0x55555555515a
```

Example 32-bit inspection:

```gdb
(gdb) info registers eax esp eip
eax            0x2a
esp            0xffffcf20
eip            0x804915a
```

The register names immediately reveal which execution model you are dealing with.

# Section 5 — Defender detection

* Architecture differences themselves are not malicious and normally generate no security alert.
* EDR/process telemetry can identify whether a 32-bit process is running under a 64-bit operating system and correlate that with process behavior.
* Defenders may investigate unexpected 32-bit processes, unusual compatibility layers, or execution of binaries inconsistent with the expected software environment.
* Exploitation telemetry is more valuable than architecture alone: crashes, suspicious memory permissions, abnormal control flow, and unexpected process behavior can expose attacks.
* Defenders commonly miss architecture-specific indicators when analyzing malware with the wrong disassembler/debugger architecture.
* Skilled operators may deliberately use architecture-compatible binaries and legitimate system components to avoid obvious anomalies.
* During incident response, analysts should establish architecture before interpreting registers, stack layouts, disassembly, or memory structures.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM with GCC and GDB. Build one 32-bit executable and one x86-64 executable from equivalent C source.

**Objective:** Prove the architectural differences by comparing register usage, pointer size, stack representation, and memory-instruction operand width.

**Steps:**

1. Create one simple C program that declares an integer, obtains a pointer to it, and performs a calculation.
2. Compile an x86-64 version and a 32-bit x86 version using the appropriate GCC multilib support.
3. Use `file` to verify that one binary is ELF 32-bit and the other is ELF 64-bit.
4. Use `readelf` to confirm their ELF classes and machine architectures.
5. Disassemble both programs with Intel syntax and locate the function prologue.
6. Identify `EAX/ESP/EBP/EIP` in the 32-bit binary and `RAX/RSP/RBP/RIP` in the 64-bit binary.
7. Locate a memory-load instruction and determine whether it accesses 4 or 8 bytes from its operand.
8. Open each binary in GDB and compare the register state at the same logical point in execution.
9. Record at least three concrete differences between the two execution environments.

**Expected output:**

```text
32-bit:
ELF32
i386
EAX / ESP / EIP
4-byte pointers

64-bit:
ELF64
x86-64
RAX / RSP / RIP
8-byte pointers
```

Your disassembly should also demonstrate that x64 code can still use 32-bit operands:

```text
mov eax, [address]   → 4-byte access
mov rax, [address]   → 8-byte access
```

**Git artifact:**

```text
x86-vs-x64/
├── README.md
├── src/
│   └── architecture_demo.c
├── binaries/
│   └── .gitkeep
├── disassembly/
│   ├── x86.asm
│   └── x64.asm
└── notes/
    └── architecture-comparison.md
```

```bash
git add x86-vs-x64/
git commit -m "Add x86 and x64 architecture comparison lab"
```

# Section 7 — Common mistakes

1. **Assuming x64 means every instruction operates on 64 bits** → x64 supports 8-, 4-, 2-, and 1-byte operands → Always inspect the actual instruction operands.

2. **Assuming a 64-bit OS means every program is 64-bit** → 64-bit operating systems can execute 32-bit applications → Identify the binary with `file` or `readelf`.

3. **Treating x86 and x64 as completely unrelated architectures** → x64 extends the x86 instruction-set family and retains substantial compatibility → Learn the register extensions and architectural differences.

4. **Assuming pointers are always 8 bytes** → 32-bit programs normally use 4-byte pointers → Determine pointer size from the target architecture/data model.

5. **Using x64 calling-convention assumptions on x86 binaries** → Function arguments can be located somewhere completely different → Identify the ABI/calling convention before tracing function arguments.

6. **Reading `[rax]` as automatically meaning "read 8 bytes"** → The operand determines the access width → Compare the destination register/instruction encoding.

7. **Reusing exploitation offsets between x86 and x64** → Stack layouts, pointer widths, calling conventions, and generated code differ → Recalculate and verify architecture-specific offsets in the actual target.

# Section 8 — Move-on gate

1. **Take one 32-bit and one 64-bit ELF binary, identify their architectures with `file`/`readelf`, and correctly choose the corresponding EAX/ESP/EIP or RAX/RSP/RIP register set without looking at your notes.**

2. **Disassemble both binaries and identify one memory instruction in each, then correctly state how many bytes it reads or writes from the instruction's operand size.**

3. **Debug both binaries in GDB and trace one function call, correctly identifying where its arguments, return value, stack pointer, and instruction pointer reside in each architecture without looking at your notes.**
