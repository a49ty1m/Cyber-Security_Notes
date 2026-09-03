# The Stack

**Roadmap:** Part 1 — Fundamentals → Stage 3 — Memory Management → The Stack

# Section 1 — What it is and where it sits

The **stack** is a memory region used by a running program to manage function calls and temporary execution state. It follows **LIFO (Last-In, First-Out)**: the most recently created stack data is normally removed first.

For offensive security, the critical part is understanding how a function creates its **stack frame**, saves state, stores local variables, calls another function, and eventually restores execution state. In common architectures, the function's **return address** is placed on the stack as part of the call mechanism.

```text
Memory Management
└── The Stack
    ├── LIFO
    ├── Stack frames
    ├── Function arguments
    ├── Local variables
    ├── Saved registers
    ├── Return address
    └── Prologue / Epilogue
```

```text
Program execution
      ↓
Function call
      ↓
Return address + stack frame
      ↓
Function executes
      ↓
Epilogue restores state
      ↓
Return to caller
```

If you skip this, stack-based buffer overflows, stack canaries, ROP, debugging crashes, and control-flow hijacking will look like unrelated tricks instead of consequences of a predictable calling mechanism.

It builds directly on virtual memory and leads into memory corruption and control-flow exploitation.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

An attacker examining stack behavior wants to determine:

* Where the stack is mapped
* How the stack grows
* Where local variables are located
* Where function arguments are stored
* Where saved registers are located
* Where the return address is stored
* How much data separates a vulnerable buffer from control-flow data
* Whether a stack canary exists
* Whether the stack is executable
* Whether ASLR changes stack addresses
* Which calling convention the binary uses

The fundamental question is:

> **Can attacker-controlled data reach something that determines where execution continues?**

## 2.2 A realistic attacker workflow

1. Identify a function that processes attacker-controlled input.
2. Determine whether the function creates a stack frame.
3. Identify local stack objects such as character buffers.
4. Determine how compiler-generated stack protection changes the frame.
5. Observe the function's prologue and epilogue in a debugger.
6. Determine where the saved control-flow state resides relative to local data.
7. Test how excessive input affects the program's behavior in a controlled lab.
8. If memory corruption occurs, determine whether a security control such as a canary detects it.
9. Analyze whether ASLR, NX/DEP, PIE, or other mitigations affect exploitation.
10. Use the resulting knowledge to understand the appropriate exploitation primitive rather than blindly guessing offsets.

## 2.3 The function call

Consider:

```c
int add(int a, int b)
{
    int result = a + b;
    return result;
}
```

Conceptually:

```text
Caller
  │
  │ call add()
  ↓
add()
  ├── establish stack frame
  ├── execute instructions
  ├── calculate result
  └── restore stack state
  │
  ↓
Caller continues
```

The processor must know where execution should continue after `add()` finishes.

That continuation information is the **return address**.

## 2.4 Return addresses

On x86-64, a `call` instruction normally saves the address of the next instruction and transfers execution to the called function.

Conceptually:

```text
Before call:

Caller:
    instruction A
    call function
    instruction B  ← execution eventually returns here

After call:

Stack contains return information
        ↓
function executes
        ↓
return
        ↓
instruction B
```

The exact physical representation depends on architecture and calling convention, but the security concept is universal:

> **Function calls create state that must eventually be restored so execution can return to the caller.**

## 2.5 Function prologue

A traditional x86-64 function may begin with instructions resembling:

```asm
push rbp
mov  rbp, rsp
sub  rsp, 0x20
```

Conceptually:

```text
Caller stack
    ↓
return address
saved RBP
local variables
    ↓
RSP
```

The prologue establishes storage for the function's execution state.

Modern compilers may optimize this heavily. A function may omit the frame pointer entirely, so do not assume every binary contains:

```asm
push rbp
mov rbp, rsp
```

## 2.6 Function epilogue

A traditional epilogue may resemble:

```asm
leave
ret
```

Conceptually:

```text
local variables removed
        ↓
saved frame state restored
        ↓
return address consumed
        ↓
execution resumes in caller
```

`ret` obtains the return target from the stack and transfers execution there.

This is why corruption of stack control-flow data can become a control-flow security problem.

## 2.7 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
The program has a stack.
```

Every normal process has stack memory, so this tells an attacker almost nothing.

**High-value finding:**

```text
A function copies attacker-controlled data
into a fixed-size stack buffer without enforcing
the destination's size.
```

Now the attacker has identified a potential **stack-based buffer overflow**.

The security significance comes from the interaction between:

```text
attacker-controlled input
        +
stack-resident object
        +
unsafe memory operation
        +
control-flow data
```

—not simply from the existence of a stack.

## 2.8 Stack canaries change the attack

A protected function may conceptually look like:

```text
local buffer
canary
saved state
return address
```

Before returning, the program verifies the canary.

If an overwrite changes it:

```text
Canary mismatch
      ↓
Abort / detection
      ↓
Normal return prevented
```

Therefore, finding a stack overflow does not automatically mean finding an exploitable return-address overwrite.

## 2.9 Where results feed next

Stack knowledge feeds directly into:

```text
Stack frames
      ↓
Memory corruption
      ↓
Stack buffer overflow
      ↓
Mitigation analysis
 ┌────┼─────────┐
 ↓    ↓         ↓
Canary ASLR    NX/DEP
      ↓
Control-flow exploitation
      ↓
ROP / related techniques
```

# Section 3 — Core concepts and terminology

| Term                   | Meaning                                                                      |
| ---------------------- | ---------------------------------------------------------------------------- |
| Stack                  | Memory region used for function-call state and temporary data                |
| LIFO                   | Last-in, first-out ordering                                                  |
| Stack frame            | Memory/state associated with one function invocation                         |
| `RSP`                  | x86-64 stack pointer register                                                |
| `RBP`                  | x86-64 base/frame pointer register when used                                 |
| `ESP`                  | 32-bit x86 stack pointer                                                     |
| `EBP`                  | 32-bit x86 frame pointer when used                                           |
| Return address         | Address where execution resumes after a function returns                     |
| Prologue               | Instructions establishing a function's execution frame                       |
| Epilogue               | Instructions restoring the caller's execution state                          |
| `call`                 | x86 instruction that transfers control to a function and saves return state  |
| `ret`                  | x86 instruction that returns using the saved return target                   |
| Stack pointer          | Register identifying the current top of the stack                            |
| Frame pointer          | Optional register used to provide a stable frame reference                   |
| Local variable         | Function-local data commonly stored in the stack frame                       |
| Stack overflow         | Writing beyond intended stack storage                                        |
| Stack canary           | Value used to detect certain stack overwrites                                |
| NX                     | No-eXecute protection preventing execution from marked non-executable memory |
| ASLR                   | Randomization of memory locations                                            |
| Calling convention     | Rules governing arguments, registers, stack usage, and return values         |
| Frame-pointer omission | Compiler optimization that removes the traditional frame pointer             |

### Traditional x86-64 stack-frame layout

A simplified conceptual layout is:

```text
Higher addresses
┌──────────────────────────┐
│ Function arguments       │
├──────────────────────────┤
│ Return address           │
├──────────────────────────┤
│ Saved frame pointer      │
├──────────────────────────┤
│ Stack canary (if used)   │
├──────────────────────────┤
│ Local variables          │
├──────────────────────────┤
│ Temporary compiler data  │
└──────────────────────────┘
Lower addresses
             ↑
            RSP
```

This is **conceptual**, not a universal fixed layout. Compiler optimization, architecture, calling convention, and function complexity can change it.

### Stack operation model

```text
push X

Before:
RSP → top

After:
RSP decreases
X stored at new top
```

```text
pop X

Before:
RSP → X

After:
X retrieved
RSP increases
```

On x86, the stack conventionally grows toward **lower virtual addresses**.

# Section 4 — Tools and commands

| Tool       | Command                     | What it finds/shows             | When to use it                     |
| ---------- | --------------------------- | ------------------------------- | ---------------------------------- |
| `gdb`      | `gdb -q ./program`          | Interactive execution/debugging | Analyze calls and frames           |
| `gdb`      | `info registers rsp rbp`    | Stack/frame registers           | Examine current stack state        |
| `gdb`      | `x/16gx $rsp`               | Raw stack memory                | Inspect stack contents             |
| `gdb`      | `disassemble /m main`       | Source/assembly relationship    | Study prologue/epilogue            |
| `gdb`      | `bt`                        | Call stack/backtrace            | See active function frames         |
| `objdump`  | `objdump -d ./program`      | Disassembled binary             | Static assembly analysis           |
| `readelf`  | `readelf -h ./program`      | ELF properties                  | Check architecture/PIE information |
| `checksec` | `checksec --file=./program` | Common binary mitigations       | Assess stack exploitation defenses |

### `gdb`

Start:

```bash
gdb -q ./program
```

Set a breakpoint:

```text
(gdb) break main
(gdb) run
```

Example:

```text
Breakpoint 1, main () at demo.c:5
5       int value = 42;
```

Inspect stack registers:

```text
(gdb) info registers rsp rbp
rsp            0x7fffffffe120
rbp            0x7fffffffe150
```

The important point is that `RSP` identifies the current stack position while `RBP`, when used as a frame pointer, provides a reference for the current frame.

### Examine stack memory

```text
(gdb) x/16gx $rsp
```

Example:

```text
0x7fffffffe120: 0x0000000000000000  0x00007fffffffe150
0x7fffffffe130: 0x0000555555555169  0x0000000000000001
```

You should not blindly label every value a return address. Determine its meaning from the surrounding instructions and call stack.

### Disassemble a function

```text
(gdb) disassemble /m main
```

Typical assembly:

```asm
push   rbp
mov    rbp,rsp
sub    rsp,0x10
...
leave
ret
```

This exposes the classic prologue and epilogue.

### `bt`

```text
(gdb) bt
```

Example:

```text
#0  helper ()
#1  main ()
#2  0x00007ffff... in __libc_start_main ()
```

This gives the current call chain.

### `objdump`

```bash
objdump -d ./program
```

Example:

```text
0000000000001149 <helper>:
1149: 55        push   %rbp
114a: 48 89 e5  mov    %rsp,%rbp
114d: 48 83 ec 10 sub $0x10,%rsp
```

This is useful when you want to inspect assembly without executing the binary.

### `readelf`

```bash
readelf -h ./program
```

Look for information such as:

```text
Class:   ELF64
Machine: Advanced Micro Devices X86-64
Type:    DYN
```

`DYN` commonly indicates a PIE executable when examining a normal Linux executable.

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

This quickly shows whether common exploitation mitigations are present.

# Section 5 — Defender detection

* Stack corruption can manifest as crashes, abnormal control flow, corrupted stack canaries, or process termination.
* Linux crash information, core dumps, application logs, and EDR telemetry can help identify suspicious memory corruption.
* Stack-canary failures are particularly useful indicators because they show that protected stack data was altered before a function returned.
* Defenders commonly miss vulnerabilities that do not immediately crash; memory corruption can sometimes produce controlled but apparently normal behavior.
* EDR/runtime protections can detect suspicious executable-memory behavior and abnormal control-flow patterns, but prevention should begin with memory-safe programming and compiler hardening.
* Skilled operators may avoid repeated crashing because crashes create noisy artifacts and can alert defenders.
* Important mitigations include stack canaries, ASLR, PIE, NX/DEP, compiler hardening, and safer memory-handling practices.

# Section 6 — Lab task

**Platform:** Kali Linux VM with a locally compiled C program.

**Objective:** Use GDB to observe a function call, identify the stack frame, and distinguish the stack pointer, frame pointer, and return-control state.

**Steps:**

1. Create a small C program containing `main()` and a separate function.
2. Compile it with debugging information.
3. Open the binary in GDB.
4. Set a breakpoint at the called function.
5. Run the program until the breakpoint is reached.
6. Inspect `RSP` and `RBP`.
7. Disassemble the function and identify its prologue and epilogue.
8. Display stack memory around `RSP`.
9. Use `bt` to compare the actual call chain with your understanding of the stack.
10. Save the assembly, register values, and observations.

Example test program:

```c
#include <stdio.h>

void helper(int value)
{
    int local = value + 10;
    printf("local = %d\n", local);
}

int main(void)
{
    helper(32);
    return 0;
}
```

Compile:

```bash
gcc -g -O0 -fno-omit-frame-pointer stack_demo.c -o stack_demo
```

**Expected output:**

You should be able to identify something resembling:

```asm
push   rbp
mov    rbp,rsp
sub    rsp,...
```

and later:

```asm
leave
ret
```

Your debugger output should also show:

```text
RSP → current stack location
RBP → current frame reference
bt  → helper() → main()
```

**Git artifact:**

```text
the-stack-lab/
├── README.md
├── src/
│   └── stack_demo.c
└── evidence/
    ├── disassembly.txt
    ├── registers.txt
    └── observations.md
```

Commit:

```bash
git add the-stack-lab/
git commit -m "Add stack frame analysis lab"
```

# Section 7 — Common mistakes

1. **Mistake → Assuming every function has `push rbp; mov rbp,rsp`.**
   **Why it matters →** Optimizing compilers can omit frame pointers.
   **Instead →** Learn to reason from actual instructions.

2. **Mistake → Assuming the return address is always at a fixed offset from a local variable.**
   **Why it matters →** Compiler optimization and stack layout vary.
   **Instead →** Inspect the actual generated binary.

3. **Mistake → Thinking the stack only stores local variables.**
   **Why it matters →** Function-call state, saved registers, temporary data, and other objects can occupy the frame.
   **Instead →** Treat the stack as execution-state storage.

4. **Mistake → Confusing `RSP` and `RBP`.**
   **Why it matters →** They serve different purposes and `RBP` may not even be used as a frame pointer.
   **Instead →** Track how instructions modify each register.

5. **Mistake → Assuming a stack overflow automatically means control-flow hijacking.**
   **Why it matters →** Canaries, layout, bounds, and other mitigations may prevent it.
   **Instead →** Analyze the complete memory-corruption primitive and defenses.

6. **Mistake → Ignoring calling conventions.**
   **Why it matters →** Arguments and return values may primarily use registers rather than the stack.
   **Instead →** Identify the target architecture and ABI before analyzing a frame.

# Section 8 — Move-on gate

1. **Debug a function call:** run a compiled C program in GDB, stop inside a called function, and identify `RSP`, `RBP`, the current frame, and caller without looking at your notes.

2. **Identify the call mechanism:** disassemble a function and correctly point out its prologue, function body, epilogue, and return instruction.

3. **Analyze a real binary:** run `checksec` and GDB against your lab binary, identify whether stack canaries, NX, and PIE are enabled, and explain how each changes the difficulty of stack-based exploitation.
