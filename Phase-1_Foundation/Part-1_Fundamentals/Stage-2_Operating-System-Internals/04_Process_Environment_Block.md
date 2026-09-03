# Process Environment Block (PEB)

**Roadmap:** Part 1 — Fundamentals → Stage 2 — Operating System Internals → Process Environment Block (PEB)

# Section 1 — What it is and where it sits

The **Process Environment Block (PEB)** is a Windows user-mode data structure maintained for each process. It contains process-environment information such as module information, process startup data, heap-related information, and loader state. Malware analysts and reverse engineers commonly inspect the PEB because it provides a low-level view of a process without relying entirely on high-level Windows APIs.

A particularly useful field is the loader data referenced through the PEB. It can be used to enumerate modules already loaded into a process, while fields such as `BeingDebugged` and `NtGlobalFlag` can be relevant to anti-debugging analysis.

```text
Windows Process
      │
      ├── User-mode address space
      │
      ├── PEB
      │    ├── Process metadata
      │    ├── Loader data
      │    ├── Heap/environment data
      │    └── Debug-related fields
      │
      └── Loaded modules
           ├── EXE
           ├── ntdll.dll
           ├── kernel32.dll
           └── other DLLs
```

```text
Process creation
      ↓
PEB initialized
      ↓
Loader initializes module information
      ↓
DLLs loaded/unloaded
      ↓
PEB reflects process state
```

If you skip the PEB, Windows malware analysis becomes heavily dependent on API-level abstractions and you will miss an important bridge between process internals, the loader, DLL enumeration, and anti-debugging.

This follows process/thread mechanics and leads into Windows loader internals, DLL analysis, process injection, and malware reverse engineering.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers and malware authors commonly inspect the PEB for:

* Loaded-module information
* Process parameters
* `BeingDebugged`
* `NtGlobalFlag`
* Loader state
* Image base information
* Heap-related state
* Information useful for resolving modules without ordinary API calls

The practical goal is often:

```text
Current process
      ↓
Locate PEB
      ↓
Read useful fields
      ↓
Discover process state
      ↓
Resolve loaded modules
      ↓
Adapt execution
```

## 2.2 Locating the PEB

On Windows x64, user-mode code can obtain the PEB through the **TEB** using the segment register:

```text
GS
 ↓
TEB
 ↓
PEB
```

On 32-bit Windows, the corresponding path uses:

```text
FS
 ↓
TEB
 ↓
PEB
```

The exact offsets are architecture-dependent, so analysts normally inspect symbols or debugger-provided structures rather than blindly assuming offsets.

## 2.3 Enumerating loaded DLLs

The PEB points to loader information.

Conceptually:

```text
PEB
 ↓
PEB_LDR_DATA
 ↓
Module linked lists
 ↓
LDR_DATA_TABLE_ENTRY
 ↓
DLL name + base address
```

A simplified example:

```text
PEB
 │
 └── Ldr
      │
      └── InMemoryOrderModuleList
           ├── program.exe
           ├── ntdll.dll
           ├── kernel32.dll
           └── suspicious.dll
```

This is valuable because the loader maintains information about modules already mapped into the process.

An analyst can compare this information with:

```text
Expected modules
        vs.
Observed modules
```

An unexpected DLL can become a high-value lead for further investigation.

## 2.4 Why malware may avoid high-level APIs

Malware does not necessarily need to call a convenient API such as:

```text
GetModuleHandle()
GetProcAddress()
```

It can inspect loader structures directly.

Conceptually:

```text
Normal approach
API
 ↓
Windows loader
 ↓
Module information

Low-level approach
PEB
 ↓
Loader structures
 ↓
Module information
```

The second approach can reduce dependence on easily identifiable API calls.

This technique is important in shellcode and malware analysis because shellcode often needs to locate essential DLLs without assuming that ordinary import resolution is available.

## 2.5 API resolution through the PEB

A common shellcode pattern is:

```text
PEB
 ↓
Loader data
 ↓
Find kernel32.dll / another module
 ↓
Find module export table
 ↓
Find desired function
 ↓
Call function
```

The attacker is effectively reconstructing information normally provided by the Windows loader/API environment.

This is especially useful in position-independent shellcode, where hard-coded addresses are unreliable because of address-space layout randomization.

## 2.6 Anti-debugging with `BeingDebugged`

The PEB contains:

```text
BeingDebugged
```

Conceptually:

```text
PEB
 ↓
BeingDebugged
 ↓
Debugger-related process state
```

A malware sample may check this value and alter behavior when it believes a debugger is attached.

Possible behavior:

```text
Debugger detected
      ↓
Exit
or
Sleep
or
Change execution
or
Corrupt analysis results
```

The important point is that this is a **signal**, not absolute proof that a human debugger is present.

## 2.7 `NtGlobalFlag`

Another field analysts commonly encounter is:

```text
NtGlobalFlag
```

Certain debugging configurations can affect related process behavior and heap settings.

Malware may therefore inspect the field as part of a larger anti-analysis strategy.

A skilled analyst does not treat one field as definitive.

Instead:

```text
BeingDebugged
+
NtGlobalFlag
+
Heap behavior
+
Debugger artifacts
+
Timing checks
```

can provide stronger evidence.

## 2.8 Dead-end vs high-value finding

**Dead-end finding:**

```text
PEB exists.
```

Every normal Windows process has one.

That observation alone provides almost no security value.

**High-value finding:**

```text
PEB
 ↓
Loader list
 ↓
Unexpected manually loaded module
```

or:

```text
PEB
 ↓
BeingDebugged checked
 ↓
Execution changes when analysis begins
```

The second findings directly reveal either suspicious module activity or anti-analysis logic.

## 2.9 Where results feed next

PEB findings can unlock:

```text
PEB inspection
     ↓
DLL enumeration
     ↓
Module base discovery
     ↓
Export resolution
     ↓
Reverse engineering
     ↓
Shellcode analysis
```

Or:

```text
PEB inspection
     ↓
Anti-debugging discovered
     ↓
Identify detection logic
     ↓
Understand malware execution branches
     ↓
Continue dynamic analysis
```

# Section 3 — Core concepts and terminology

| Term                     | Meaning                                                                                         |
| ------------------------ | ----------------------------------------------------------------------------------------------- |
| **PEB**                  | Per-process Windows user-mode structure containing environment and loader-related information.  |
| **TEB**                  | Per-thread structure containing thread-specific information and a pointer to the process's PEB. |
| **PEB_LDR_DATA**         | Loader-related structure referenced by the PEB.                                                 |
| **LDR_DATA_TABLE_ENTRY** | Structure describing a loaded module.                                                           |
| **Loader**               | Windows component responsible for loading executable images and DLLs.                           |
| **Module**               | Executable image loaded into a process, such as an EXE or DLL.                                  |
| **Image base**           | Virtual address where a module is mapped.                                                       |
| **BeingDebugged**        | PEB field indicating debugger-related process state.                                            |
| **NtGlobalFlag**         | PEB field associated with global process/debugging-related behavior.                            |
| **TEB**                  | Thread Environment Block associated with one thread.                                            |
| **WOW64**                | Windows subsystem allowing 32-bit applications to run on 64-bit Windows.                        |
| **ASLR**                 | Randomizes image locations to make fixed addresses unreliable.                                  |
| **Export table**         | PE structure describing functions/data exported by a module.                                    |
| **Manual mapping**       | Loading a PE image without using the normal loader path.                                        |
| **Anti-debugging**       | Techniques designed to detect or interfere with debugging.                                      |
| **Shellcode**            | Small position-independent code commonly used as an exploitation payload.                       |
| **PE format**            | Windows executable file format used by EXEs and DLLs.                                           |
| **Module enumeration**   | Identifying loaded executable images inside a process.                                          |

### Architecture map

| Architecture | TEB access                  | PEB relationship               |
| ------------ | --------------------------- | ------------------------------ |
| x86          | `FS`                        | TEB → PEB                      |
| x64          | `GS`                        | TEB → PEB                      |
| WOW64        | Mixed 32/64-bit environment | Architecture-specific handling |

### Relevant PEB path

```text
TEB
 ↓
PEB
 ↓
Ldr
 ↓
PEB_LDR_DATA
 ↓
Module lists
 ↓
LDR_DATA_TABLE_ENTRY
```

# Section 4 — Tools and commands

| Tool       | Command                                       | What it finds/shows                   | When to use it                  |
| ---------- | --------------------------------------------- | ------------------------------------- | ------------------------------- |
| WinDbg     | `dt ntdll!_PEB @$peb`                         | PEB fields                            | Inspect PEB structure           |
| WinDbg     | `!peb`                                        | Current process PEB information       | Fast PEB inspection             |
| WinDbg     | `dt ntdll!_PEB_LDR_DATA poi(@$peb+0x18)`      | Loader structure                      | Inspect loader internals        |
| WinDbg     | `lm`                                          | Loaded modules                        | Compare module state            |
| WinDbg     | `r @$peb`                                     | Current PEB address                   | Locate PEB                      |
| x64dbg     | `TEB` / PEB navigation through debugger views | Thread/process environment structures | Interactive reverse engineering |
| x64dbg     | `info` / module views                         | Loaded module information             | Compare loader/module state     |
| PowerShell | `Get-Process`                                 | Process information                   | Establish target process        |
| PowerShell | `Get-Process -Id <PID> \| Select-Object *`    | Detailed process properties           | Build baseline before debugging |

### WinDbg — `!peb`

```text
0:000> !peb
PEB at 000001A2`........
    ImageBaseAddress: ...
    Ldr: ...
    ProcessParameters: ...
```

Interpretation:

```text
PEB address
     ↓
Image base
     ↓
Loader data
     ↓
Process parameters
```

The exact address values vary for every process.

### WinDbg — `dt ntdll!_PEB @$peb`

```text
0:000> dt ntdll!_PEB @$peb
ntdll!_PEB
   +0x002 BeingDebugged : UChar
   +0x018 Ldr           : Ptr64 ...
   ...
```

This lets you inspect named structure fields rather than guessing memory offsets.

### WinDbg — `lm`

```text
0:000> lm
start             end                 module name
00007ff...        ...                 ntdll
00007ff...        ...                 kernel32
00007ff...        ...                 kernelbase
```

This provides the debugger's module view.

Compare this against the loader information when investigating unusual module state.

### WinDbg — `r @$peb`

```text
0:000> r @$peb
@$peb = 0x000001a2`........
```

The value is the address of the current process's PEB.

### x64dbg

In a debugging session, inspect the current thread/process environment structures and follow:

```text
TEB
 ↓
PEB
 ↓
Ldr
 ↓
module list
```

A suspicious entry can then be compared against the normal module list.

### PowerShell — `Get-Process`

```text
PS> Get-Process
Handles  NPM(K)  PM(K)  WS(K)  CPU(s) Id ProcessName
...
```

Use this to identify a process before attaching a debugger.

### PowerShell — detailed process information

```text
PS> Get-Process -Id 4200 | Select-Object *
```

This exposes properties useful for selecting and baselining the process before low-level inspection.

# Section 5 — Defender detection

* **Module telemetry:** EDR products can compare loaded modules against expected images and detect unusual DLL loading or memory-backed modules.
* **Process-image inspection:** Suspicious discrepancies between normal loader state and observed memory mappings can indicate manual mapping or injection.
* **Anti-debugging indicators:** Repeated debugger checks, environment checks, or branches based on `BeingDebugged` can be identified during malware analysis.
* **Memory analysis:** Analysts can inspect executable memory regions that do not correspond cleanly to legitimate loaded modules.
* **What defenders miss:** Looking only at normal DLL-load events can miss code that was manually mapped or otherwise made executable without following the conventional loader path.
* **Behavioral correlation:** Suspicious module state becomes much stronger evidence when combined with remote thread creation, executable private memory, unusual parent processes, or unsigned code.
* **Operator footprint:** Skilled malware may avoid obvious API-based module enumeration or manipulate analysis conditions, so defenders should corroborate PEB observations with memory and kernel-backed telemetry rather than trusting one structure.

# Section 6 — Lab task

**Platform:** Windows 10/11 analysis VM with WinDbg.

**Objective:** Inspect a running process's PEB, identify its loader information, and compare PEB-derived module state with the debugger's module list.

**Steps:**

1. Start a harmless Windows process in the analysis VM.
2. Attach WinDbg to that process.
3. Locate the process PEB.
4. Display the PEB structure and identify `BeingDebugged` and `Ldr`.
5. Follow the loader pointer into `PEB_LDR_DATA`.
6. Display the debugger's loaded-module list.
7. Compare several modules from the loader information with the debugger's module output.
8. Record the PEB address, image base, loader pointer, and observed module names.
9. Repeat after starting the process normally without the debugger attached.
10. Record whether the observed debugger-related PEB state differs.

**Expected output:**

```text
PEB
 ├── ImageBaseAddress → target.exe
 ├── BeingDebugged    → debugger-dependent state
 └── Ldr
      ↓
   loader data
      ↓
   ntdll.dll
   kernel32.dll
   kernelbase.dll
   target DLLs
```

Success means you can navigate:

```text
TEB → PEB → Ldr → module information
```

and explain what each transition represents.

**Git artifact:**

```text
peb-analysis/
├── README.md
├── evidence/
│   ├── peb.txt
│   ├── modules.txt
│   └── comparison.txt
└── notes.md
```

```bash
git add peb-analysis/
git commit -m "Document Windows PEB and loader analysis"
```

# Section 7 — Common mistakes

1. **Mistake:** Thinking the PEB is kernel memory.
   **Why it matters:** The PEB is primarily a user-mode process structure.
   **Do instead:** Treat it as user-mode process metadata.

2. **Mistake:** Assuming PEB offsets are universal.
   **Why it matters:** Structure layouts vary between architectures and Windows versions.
   **Do instead:** Use symbols/debugger structures.

3. **Mistake:** Treating `BeingDebugged` as definitive proof of a debugger.
   **Why it matters:** It is only one signal and can be modified or circumvented.
   **Do instead:** Correlate multiple anti-debugging indicators.

4. **Mistake:** Confusing the PEB with the TEB.
   **Why it matters:** The TEB is thread-specific; the PEB is process-specific.
   **Do instead:** Remember `TEB → PEB`.

5. **Mistake:** Assuming every loaded DLL must appear in the normal loader lists.
   **Why it matters:** Manual mapping and other techniques can produce module-like executable memory without conventional loader bookkeeping.
   **Do instead:** Compare loader data with actual memory mappings.

6. **Mistake:** Studying only `BeingDebugged`.
   **Why it matters:** The PEB's loader structures are equally important for malware and shellcode analysis.
   **Do instead:** Learn both process-state fields and loader relationships.

# Section 8 — Move-on gate

1. **Attach WinDbg to a Windows process, locate its PEB, and identify `BeingDebugged`, `ImageBaseAddress`, and `Ldr` without looking at your notes.**

2. **Follow `PEB → Ldr → loader data` and identify at least three loaded modules, then correctly match them against the debugger's module list.**

3. **Analyze a small Windows program that checks `BeingDebugged`, identify the PEB-based anti-debugging logic in the debugger, and explain exactly which branch changes when debugging is detected.**
