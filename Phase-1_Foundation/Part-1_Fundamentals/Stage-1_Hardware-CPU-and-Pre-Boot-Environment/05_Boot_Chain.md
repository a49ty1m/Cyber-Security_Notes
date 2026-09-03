# The Boot Chain — UEFI, Secure Boot, Bootloader, Kernel & systemd

**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → The Boot Chain

# Section 1 — What it is and where it sits

The **boot chain** is the sequence that takes a powered-off computer from firmware execution to a running operating system. On a modern Linux machine, the simplified path is **UEFI → Secure Boot verification → bootloader → Linux kernel → initramfs → systemd/PID 1 → services and user space**. Each stage establishes enough state and trust for the next stage to continue.

For offensive security, this chain matters because compromise below the operating-system level can survive normal OS security controls. An attacker who modifies a boot component can potentially execute code before Linux's normal security mechanisms and monitoring are active.

```text
Power On
   ↓
UEFI Firmware
   ↓
Secure Boot verification
   ↓
Bootloader
   ↓
Linux Kernel + initramfs
   ↓
systemd (PID 1)
   ↓
Services
   ↓
Login / User Space
```

Attack-chain placement:

```text
Hardware / CPU
      ↓
UEFI / Pre-Boot ← THIS TOPIC
      ↓
Bootloader
      ↓
Kernel
      ↓
Process / Memory
      ↓
Operating System
      ↓
Security Controls
```

If you underestimate the boot chain, you can miss persistence mechanisms that operate **before the operating system starts**. You also risk misunderstanding why Secure Boot, kernel signatures, bootloader configuration, and firmware integrity matter.

This connects the CPU/pre-boot fundamentals to the next stage: understanding **how the operating system establishes processes, memory, privileges, and services after boot**.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

A boot-chain attacker is interested in components that execute before or during operating-system initialization:

* UEFI firmware
* UEFI variables
* EFI System Partition (ESP)
* bootloader binaries
* bootloader configuration
* kernel images
* initramfs
* kernel command-line parameters
* Secure Boot state
* signing keys and trust databases
* early-boot services
* recovery environments

The key question is:

> **Can I modify something that executes before the normal security stack is fully operational?**

A simplified trust chain looks like:

```text
Firmware
   ↓ verifies
Bootloader
   ↓ loads/verifies
Kernel
   ↓ initializes
systemd
   ↓ starts
Services
```

## 2.2 UEFI executes first

When the machine starts, the CPU begins executing firmware stored on the motherboard's non-volatile flash.

On modern systems this is normally **UEFI firmware**.

UEFI initializes enough hardware to locate a boot target and provides the firmware environment from which an operating system loader can execute.

Conceptually:

```text
CPU reset
   ↓
UEFI firmware
   ↓
hardware initialization
   ↓
UEFI boot manager
   ↓
EFI executable
```

An attacker interested in pre-OS persistence therefore examines whether firmware settings, UEFI variables, or boot files have been modified.

## 2.3 Secure Boot establishes a trust decision

With Secure Boot enabled, UEFI can verify the cryptographic signature of an EFI executable before executing it.

Conceptually:

```text
UEFI
  ↓
EFI bootloader
  ↓
signature verification
  ↓
valid?
 ┌───────┴───────┐
YES             NO
 ↓                ↓
execute         reject
```

The important point is that Secure Boot is a **chain of trust**, not a magical "no malware" switch.

An attacker does not necessarily need to defeat cryptography directly.

Potential attack surfaces include:

* compromised signing keys
* incorrectly trusted keys
* vulnerable but legitimately signed boot components
* firmware vulnerabilities
* configuration weaknesses
* attacks against later stages of the boot chain

## 2.4 The EFI System Partition

The **EFI System Partition (ESP)** is a filesystem partition containing EFI boot files.

A typical Linux installation may contain something resembling:

```text
EFI System Partition
└── EFI
    ├── Boot
    │   └── bootx64.efi
    └── linux-distribution
        ├── bootloader.efi
        └── configuration
```

The exact structure depends on the distribution and bootloader.

An attacker who gains sufficient privileges may investigate the ESP because modifying a bootloader or related configuration can affect what executes during the next boot.

The security significance is therefore:

```text
OS compromise
     ↓
sufficient privileges
     ↓
boot-related modification
     ↓
reboot
     ↓
modified pre-OS component executes
```

That is a different persistence layer from a normal user-space startup service.

## 2.5 Bootloader execution

After firmware selects the boot target, the bootloader takes control.

Common Linux bootloaders include:

* GRUB
* systemd-boot

The bootloader's job includes locating and loading the kernel and, commonly, the initramfs.

A simplified GRUB flow:

```text
UEFI
 ↓
GRUB EFI executable
 ↓
GRUB configuration
 ↓
kernel selection
 ↓
initramfs selection
 ↓
kernel loaded
```

The bootloader may also pass parameters to the kernel.

Conceptually:

```text
Bootloader
    ↓
Linux kernel
    +
kernel command line
    +
initramfs
```

Those parameters can significantly affect kernel behavior.

## 2.6 Kernel load

The bootloader loads the Linux kernel into memory and transfers execution to it.

The kernel then takes responsibility for operating-system initialization.

Conceptually:

```text
Bootloader
    ↓
kernel image loaded
    ↓
kernel execution
    ↓
CPU / memory / drivers initialized
    ↓
early user space
```

The kernel establishes:

* virtual memory
* CPU scheduling
* hardware drivers
* interrupt handling
* process management
* security mechanisms
* filesystem support

The attacker cares about the transition because code executing at kernel privilege has dramatically greater authority than ordinary user-space code.

## 2.7 initramfs — the often-forgotten stage

There is an important stage between kernel startup and the normal root filesystem becoming fully available: **initramfs**.

The initramfs is a temporary early userspace environment loaded into memory.

It can contain:

* storage drivers
* filesystem drivers
* encryption tooling
* scripts
* utilities
* modules required to locate/mount the real root filesystem

Conceptually:

```text
Bootloader
    ↓
Kernel
    ↓
initramfs
    ↓
find / unlock / mount root filesystem
    ↓
switch to real root filesystem
    ↓
systemd
```

For security analysis, this matters because the initramfs itself contains executable code that runs extremely early.

## 2.8 systemd becomes PID 1

Once the kernel has initialized sufficiently and the real root filesystem is available, Linux starts the first userspace process.

On a typical modern Linux distribution:

```text
PID 1 = systemd
```

Conceptually:

```text
Kernel
  ↓
/sbin/init
  ↓
systemd
  ↓
PID 1
```

systemd then starts and manages services.

The resulting process tree may resemble:

```text
systemd (PID 1)
├── system services
├── networking
├── logging
├── authentication
├── desktop/session services
└── user processes
```

This is where the system transitions from **boot-time initialization** into normal operating-system operation.

## 2.9 Realistic attacker workflow

A boot-chain investigation after obtaining privileged access might proceed conceptually like this:

1. Determine whether the system boots using UEFI or legacy BIOS.
2. Determine whether Secure Boot is enabled.
3. Identify the EFI System Partition.
4. Identify the configured bootloader.
5. Determine which kernel image is being loaded.
6. Examine bootloader configuration and kernel parameters.
7. Determine how the initramfs is generated.
8. Identify the process that becomes PID 1.
9. Examine early-boot services.
10. Look for unexpected modifications to boot-related components.
11. Compare important boot artifacts against trusted versions or expected state.
12. Determine whether the persistence mechanism survives normal OS remediation.

The goal is not merely:

```text
"What starts Linux?"
```

It is:

```text
"What code executes before normal security controls,
who authorized it, and can an attacker alter that chain?"
```

## 2.10 Dead-end finding vs high-value finding

### Dead-end finding

An attacker discovers:

```text
Secure Boot: enabled
Bootloader: GRUB
Kernel: expected distribution kernel
```

This is useful reconnaissance, but it does not establish compromise.

### High-value finding

Suppose privileged investigation reveals:

```text
Secure Boot: disabled
       +
unexpected EFI executable on ESP
       +
boot configuration references it
```

That combination is much more significant.

The reasoning becomes:

```text
Unexpected pre-OS executable
        ↓
configured for boot execution
        ↓
runs before normal user-space controls
        ↓
potential persistence / early execution
```

This still requires validation. An unfamiliar EFI file is not automatically malicious.

## 2.11 Pivots

Boot-chain findings feed directly into deeper investigation:

```text
UEFI
 ↓
Secure Boot state
 ↓
ESP
 ↓
Bootloader
 ↓
Kernel parameters
 ↓
initramfs
 ↓
Kernel
 ↓
systemd
 ↓
Services
```

Different findings produce different pivots.

```text
Unexpected EFI file
        ↓
Analyze EFI binary
```

```text
Unexpected boot parameter
        ↓
Investigate kernel behavior
```

```text
Modified initramfs
        ↓
Inspect embedded files/scripts/modules
```

```text
Unexpected systemd unit
        ↓
Investigate user-space persistence
```

# Section 3 — Core concepts and terminology

| Term                | Meaning                                                                                     |
| ------------------- | ------------------------------------------------------------------------------------------- |
| UEFI                | Modern firmware interface responsible for initializing hardware and launching EFI programs. |
| Secure Boot         | UEFI mechanism that verifies trusted signatures before executing boot components.           |
| EFI                 | Executable/application format and firmware environment used by UEFI.                        |
| ESP                 | EFI System Partition containing boot-related EFI files.                                     |
| UEFI Variable       | Firmware-managed persistent key/value data used for boot and platform configuration.        |
| Boot Manager        | UEFI component that selects a boot entry to execute.                                        |
| Bootloader          | Program that loads and starts the operating-system kernel.                                  |
| GRUB                | Common Linux bootloader.                                                                    |
| systemd-boot        | Lightweight UEFI boot manager commonly used with Linux systems.                             |
| Kernel              | Privileged core of the operating system.                                                    |
| Kernel Image        | File containing the Linux kernel that the bootloader loads.                                 |
| initramfs           | Temporary early userspace filesystem loaded alongside the kernel.                           |
| Kernel Command Line | Parameters passed from the bootloader to the Linux kernel.                                  |
| Root Filesystem     | Filesystem containing the normal Linux userspace hierarchy.                                 |
| PID 1               | First userspace process; normally systemd on modern Linux distributions.                    |
| systemd             | Linux system and service manager commonly running as PID 1.                                 |
| Boot Entry          | UEFI configuration describing a boot target and its parameters.                             |
| Trust Chain         | Sequence in which one trusted component authorizes or validates the next component.         |
| Firmware            | Low-level software stored on hardware that executes before the OS.                          |

### Boot stages

| Stage       | Main responsibility                                     |
| ----------- | ------------------------------------------------------- |
| UEFI        | Hardware/firmware initialization and boot selection     |
| Secure Boot | Verify trusted EFI executable                           |
| Bootloader  | Select/load kernel and initramfs                        |
| Kernel      | Initialize operating-system core                        |
| initramfs   | Provide early userspace needed to reach root filesystem |
| systemd     | Start and manage normal userspace services              |

### BIOS vs UEFI

| Feature                      | Legacy BIOS | UEFI   |
| ---------------------------- | ----------- | ------ |
| Modern firmware standard     | No          | Yes    |
| EFI executable support       | No          | Yes    |
| ESP                          | No          | Yes    |
| Secure Boot                  | No          | Yes    |
| Typical modern Linux systems | Less common | Common |

# Section 4 — Tools and commands

| Tool           | Command                                     | What it finds/shows                           | When to use it                                     |
| -------------- | ------------------------------------------- | --------------------------------------------- | -------------------------------------------------- |
| `bootctl`      | `bootctl status`                            | Firmware, Secure Boot, bootloader information | Initial boot-chain assessment                      |
| `efibootmgr`   | `efibootmgr -v`                             | UEFI boot entries and paths                   | Examine boot configuration                         |
| `lsblk`        | `lsblk -f`                                  | Partitions/filesystems                        | Locate ESP                                         |
| `findmnt`      | `findmnt /boot/efi`                         | ESP mount information                         | Identify mounted EFI partition                     |
| `mokutil`      | `mokutil --sb-state`                        | Secure Boot state                             | Verify Secure Boot                                 |
| `ls`           | `ls -lah /boot/efi/EFI/`                    | EFI files/directories                         | Inspect ESP                                        |
| `grub-editenv` | `grub-editenv list`                         | GRUB environment variables                    | Examine GRUB state                                 |
| `cat`          | `cat /proc/cmdline`                         | Active kernel command line                    | Identify kernel parameters                         |
| `uname`        | `uname -a`                                  | Running kernel information                    | Identify loaded kernel                             |
| `ps`           | `ps -p 1 -f`                                | PID 1 process                                 | Identify init system                               |
| `systemctl`    | `systemctl list-unit-files --state=enabled` | Enabled services                              | Examine startup persistence                        |
| `journalctl`   | `journalctl -b`                             | Current boot logs                             | Investigate boot sequence                          |
| `strings`      | `strings /path/to/file`                     | Printable strings                             | Initial static inspection of suspicious boot files |
| `file`         | `file /path/to/file`                        | File format/architecture                      | Identify EFI binaries                              |

Example:

```bash
$ mokutil --sb-state
SecureBoot enabled
```

Interpretation:

```text
Secure Boot is active.
```

If it reports:

```text
SecureBoot disabled
```

the system does not currently enforce Secure Boot verification.

Example:

```bash
$ efibootmgr -v

BootCurrent: 0001
BootOrder: 0001,0000
Boot0000* Linux HD(...)/File(\EFI\...)
Boot0001* UEFI OS HD(...)/File(\EFI\...)
```

Interpretation:

```text
BootOrder → order in which UEFI considers boot entries
File(...)  → EFI executable associated with an entry
```

An unexpected entry deserves investigation rather than automatic classification as malicious.

Example:

```bash
$ lsblk -f
NAME        FSTYPE FSVER LABEL UUID                                 MOUNTPOINTS
nvme0n1
├─nvme0n1p1 vfat   FAT32       XXXX-XXXX                            /boot/efi
└─nvme0n1p2 ext4               XXXX-XXXX                            /
```

The `vfat` partition mounted at `/boot/efi` is commonly the ESP.

Example:

```bash
$ cat /proc/cmdline
BOOT_IMAGE=/vmlinuz root=/dev/nvme0n1p2 ro quiet
```

This shows parameters actually supplied to the running kernel.

Example:

```bash
$ ps -p 1 -f
UID   PID  PPID  C STIME TTY          TIME CMD
root    1     0  0 ...   ?        ... /sbin/init
```

On many distributions:

```text
/sbin/init → systemd
```

You can verify it with:

```bash
$ readlink -f /sbin/init
/usr/lib/systemd/systemd
```

Example:

```bash
$ journalctl -b | head
Sep 03 ... kernel: Linux version ...
Sep 03 ... systemd[1]: Starting ...
Sep 03 ... systemd[1]: Reached target ...
```

This gives a chronological view of the current boot.

# Section 5 — Defender detection

* **UEFI/Secure Boot:** inspect firmware state, UEFI variables, boot entries, and Secure Boot measurements where platform tooling supports them.
* **ESP modifications:** file-integrity monitoring can detect unexpected changes to EFI executables or bootloader files.
* **Boot configuration:** unexpected UEFI boot entries or changed boot paths are strong investigation triggers.
* **Kernel/initramfs:** verify kernel and initramfs integrity and investigate unexpected modifications or modules.
* **systemd:** audit enabled units, service binaries, timestamps, ownership, and unexpected early-start services.
* **Boot logs:** `journalctl -b` and kernel logs can reveal unusual boot parameters, failed components, or unexpected startup behavior.
* **Common miss:** defenders often monitor user-space persistence thoroughly while ignoring the ESP, firmware configuration, bootloader, and initramfs.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM using UEFI firmware. Do this inside a disposable VM so you can inspect boot configuration without risking your primary system.

**Objective:** Map the complete Linux boot chain from UEFI/Secure Boot through bootloader, kernel, initramfs, and systemd.

**Steps:**

1. Confirm that the VM boots using UEFI rather than legacy BIOS.
2. Check the current Secure Boot state.
3. Identify the EFI System Partition and inspect its directory structure.
4. Enumerate UEFI boot entries and record the EFI executable associated with the active entry.
5. Identify the running kernel and record the active kernel command line.
6. Identify PID 1 and verify whether it is systemd.
7. Examine the current boot journal and identify messages from the kernel and systemd.
8. Build a diagram connecting the observed firmware, EFI executable, kernel, initramfs, root filesystem, and PID 1.
9. Record which artifacts you would verify during an investigation for pre-OS persistence.

**Expected output:**

```text
UEFI
 ↓
Secure Boot: [enabled/disabled]
 ↓
EFI System Partition
 ↓
Bootloader: [identified]
 ↓
Kernel: [identified]
 ↓
initramfs: [identified]
 ↓
Root filesystem
 ↓
systemd / PID 1
 ↓
Services
```

You should be able to explain exactly which artifact corresponds to every stage.

**Git artifact:**

```text
boot-chain/
├── README.md
├── inventory/
│   ├── uefi.txt
│   ├── boot-entries.txt
│   └── kernel.txt
└── notes/
    └── boot-chain-analysis.md
```

```bash
git add boot-chain/
git commit -m "Add Linux boot chain analysis lab"
```

# Section 7 — Common mistakes

1. **Treating UEFI and Secure Boot as the same thing** → UEFI is the firmware environment; Secure Boot is a verification mechanism within it → Always record both separately.

2. **Assuming Secure Boot means the entire boot chain is automatically trustworthy** → Secure Boot depends on keys, signatures, firmware, and trusted components → Investigate the complete chain of trust.

3. **Forgetting the ESP** → Boot-related EFI executables commonly reside there → Include the ESP in any serious boot-persistence investigation.

4. **Jumping from bootloader directly to systemd** → The kernel and often initramfs execute in between → Understand the complete transition.

5. **Assuming `/sbin/init` means a specific implementation** → It can be a symlink to different init systems → Resolve it and identify the actual PID 1 process.

6. **Treating an unfamiliar EFI file as automatically malicious** → Boot directories can contain legitimate vendor and recovery components → Validate its boot entry, signature, origin, hash, and expected installation state.

7. **Ignoring kernel command-line parameters** → Boot parameters can significantly alter kernel behavior → Always inspect the command line when analyzing unusual boot behavior.

# Section 8 — Move-on gate

1. **Boot a UEFI Linux VM and independently identify its Secure Boot state, ESP, active UEFI boot entry, and bootloader without looking at your notes.**

2. **Trace one real boot from the kernel command line through initramfs to PID 1, identifying the kernel, initramfs, root filesystem transition, and systemd startup from local evidence.**

3. **Given a Linux VM containing an unexpected EFI boot entry, determine which EFI executable it launches, trace where that file resides on the ESP, and identify the next artifact you would investigate without modifying the system.**
