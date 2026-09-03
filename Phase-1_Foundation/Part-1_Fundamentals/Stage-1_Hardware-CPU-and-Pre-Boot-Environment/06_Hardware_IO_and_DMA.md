# Hardware I/O & DMA
**Roadmap:** Part 1 — Fundamentals → Stage 1 — Hardware, CPU & Pre-Boot Environment → Hardware I/O & DMA

# Section 1 — What it is and where it sits

**Direct Memory Access (DMA)** allows a hardware device to transfer data directly between the device and system memory without the CPU copying every byte itself. The CPU configures the transfer, the DMA-capable device or controller performs the transfer, and the CPU is notified when the operation completes.

For offensive security, DMA matters because some peripherals can access memory independently of normal CPU instruction execution. That creates a fundamentally different attack surface from ordinary software: instead of asking the CPU to read sensitive memory, a DMA-capable device may be able to access memory directly, subject to platform protections such as an **IOMMU**.

```text
Without DMA:

Peripheral
    ↓
CPU
    ↓
RAM


With DMA:

CPU ──configures──→ DMA-capable device
                      ↓
                 System RAM
                      ↓
                   CPU notified
```

Attack-chain placement:

```text
CPU / Registers
      ↓
Hardware I/O ← THIS TOPIC
      ↓
DMA
      ↓
Physical Memory
      ↓
IOMMU / Device Isolation
      ↓
Kernel / Drivers
      ↓
Hardware Attack Surface
```

If you skip DMA, you miss an important distinction between **CPU-controlled memory access** and **device-initiated memory access**. That distinction becomes important when studying PCIe, Thunderbolt, IOMMUs, kernel security, virtualization, and DMA-based attacks.

This builds on the CPU and memory fundamentals and leads naturally into **memory protection, device isolation, and hardware-assisted security**.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

A hardware-focused attacker is interested in whether a peripheral can access sensitive system memory independently of ordinary CPU execution.

Relevant targets include:

- PCIe devices
- Thunderbolt-connected peripherals
- FireWire/IEEE 1394 devices
- Network adapters
- Storage controllers
- GPUs
- USB controllers and devices with relevant bus access
- DMA-capable FPGA/custom hardware
- Virtual devices in hypervisors

The core question is:

> **Can this device initiate memory transactions, and what prevents it from accessing memory it should not be allowed to access?**

A simplified model:

```text
Device
  ↓
DMA request
  ↓
Memory subsystem
  ↓
RAM
```

On a modern protected system, the path may instead look like:

```text
Device
  ↓
DMA request
  ↓
IOMMU
  ↓
Allowed?
 ┌──────┴──────┐
YES           NO
 ↓             ↓
RAM          Block
```

## 2.2 Why DMA exists

Imagine a high-speed network card receiving a large packet stream.

Without DMA:

```text
Network card
    ↓
CPU receives data
    ↓
CPU copies data
    ↓
RAM
```

The CPU would spend significant time moving data instead of executing application code.

With DMA:

```text
Network card
    ↓
DMA transfer
    ↓
RAM buffer
    ↓
interrupt
    ↓
CPU processes completed buffer
```

The CPU still participates.

It normally:

1. allocates/prepares a buffer
2. configures the device
3. tells the device where/how much data to transfer
4. allows the device to operate
5. handles completion/error notification

The CPU is therefore **not completely uninvolved**.

## 2.3 The attacker cares about the address

A DMA transaction fundamentally needs information about where data should go or come from.

Conceptually:

```text
Device
   ↓
DMA address
   ↓
memory region
   ↓
read/write
```

For a legitimate network transfer:

```text
NIC
 ↓
DMA write
 ↓
network receive buffer
 ↓
kernel processes packet
```

For a malicious device:

```text
Malicious peripheral
 ↓
DMA request
 ↓
attempted sensitive-memory access
```

The security boundary therefore becomes:

```text
Device → memory
```

rather than simply:

```text
Process → kernel
```

## 2.4 A realistic attack workflow

A hardware attacker with physical access might conceptually proceed like this:

1. Identify buses and peripherals available on the target.
2. Determine whether a relevant external interface exposes DMA-capable hardware.
3. Determine whether the operating system/platform enables DMA protections.
4. Determine whether an IOMMU is present and active.
5. Determine which devices are assigned which memory-access permissions.
6. Identify whether the target system allows external devices to access memory before the OS fully establishes protections.
7. Evaluate whether the device can communicate with sensitive memory regions.
8. Validate the protection boundary in an isolated laboratory.
9. Determine whether the platform blocks unauthorized device memory transactions.
10. If protection is absent or incorrectly configured, assess the security impact.

The important distinction is:

```text
CPU exploitation:
attacker controls software execution
       ↓
CPU performs memory access


DMA attack:
attacker controls a device
       ↓
device initiates memory access
```

## 2.5 What could be valuable in memory?

An attacker interested in DMA exposure might theoretically target memory containing:

- kernel data
- process memory
- credentials
- cryptographic material
- security-sensitive configuration
- authentication state
- hypervisor/VM-related memory
- sensitive application data

But the exact accessibility depends heavily on the platform's DMA isolation.

A modern system should not simply allow an arbitrary peripheral to access every physical RAM location.

## 2.6 IOMMU changes the attack model

An **IOMMU** is effectively a memory-management layer for devices.

The CPU has an MMU for CPU-side virtual memory.

Devices can be restricted using an IOMMU:

```text
CPU:
Virtual address
      ↓
MMU
      ↓
RAM


Device:
DMA address
      ↓
IOMMU
      ↓
Allowed RAM
```

This creates a critical security boundary.

Without effective isolation:

```text
Device
  ↓
DMA
  ↓
potentially broad physical memory access
```

With isolation:

```text
Device
  ↓
DMA
  ↓
IOMMU
  ↓
restricted mapping
  ↓
authorized memory only
```

## 2.7 Dead-end finding vs high-value finding

### Dead-end finding

An attacker identifies:

```text
PCIe network controller
DMA-capable
IOMMU enabled
device restricted to expected memory mappings
```

DMA capability alone does not establish a vulnerability.

The protection boundary may be working exactly as designed.

### High-value finding

A much more significant finding would be:

```text
DMA-capable external device
        ↓
platform lacks effective DMA isolation
        ↓
device can access memory outside its intended region
```

That changes the threat model substantially.

The attacker may now have a hardware-level path to memory that bypasses normal process permissions.

The important finding is therefore not:

> "This device supports DMA."

It is:

> **"This device has unauthorized memory access despite the security boundary that should restrict it."**

## 2.8 Pre-boot DMA is especially interesting

The boot stage matters because security protections may not be identical throughout the system's lifecycle.

Conceptually:

```text
Power on
   ↓
Firmware
   ↓
Device initialization
   ↓
OS starts
   ↓
IOMMU / DMA protections established
   ↓
Normal operation
```

A security assessment therefore asks:

```text
When does DMA protection become active?
Who configures it?
What devices are trusted before that point?
```

This connects directly to the previous boot-chain topic.

## 2.9 Pivots

A DMA finding can lead into several deeper areas:

```text
DMA
 ↓
PCIe
 ↓
IOMMU
 ↓
Kernel device drivers
 ↓
Physical memory
```

or:

```text
DMA
 ↓
Pre-boot environment
 ↓
UEFI
 ↓
Boot security
```

or:

```text
DMA
 ↓
Virtualization
 ↓
Device passthrough
 ↓
Guest/host isolation
```

# Section 3 — Core concepts and terminology

| Term | Meaning |
|---|---|
| DMA | Hardware mechanism allowing devices to transfer data to/from system memory with limited CPU involvement. |
| DMA-capable Device | Peripheral capable of initiating DMA transactions. |
| DMA Transfer | Movement of data between a device and memory without CPU copying every byte. |
| DMA Address | Address supplied for a device memory transaction. |
| IOMMU | Hardware mechanism that translates/restricts device memory accesses. |
| MMU | CPU-side memory-management hardware that translates virtual memory addresses. |
| Physical Address | Address referring to physical system memory/resources. |
| Virtual Address | Address used by software that is translated to physical memory. |
| PCIe | High-speed peripheral interconnect commonly used by DMA-capable devices. |
| Bus Mastering | Device capability to initiate transactions on a bus rather than waiting for CPU mediation. |
| Scatter-Gather | DMA technique allowing transfers across multiple non-contiguous memory buffers. |
| DMA Buffer | Memory region prepared for device data transfer. |
| Interrupt | Hardware notification used to inform the CPU that an event occurred. |
| I/O | Input/output communication between the system and hardware. |
| Device Driver | Kernel software controlling a hardware device. |
| Memory Mapping | Association between device-visible addresses and memory resources. |
| DMA Remapping | IOMMU-based translation/restriction of device memory addresses. |
| Thunderbolt | High-speed external interface capable of exposing PCIe connectivity and therefore relevant DMA security concerns. |

### Memory-access comparison

| Mechanism | Initiator | Typical protection |
|---|---|---|
| CPU memory access | CPU instruction | MMU, page permissions |
| Process memory access | User process | Virtual memory isolation |
| Kernel memory access | Kernel | Kernel privileges |
| DMA | Hardware device | IOMMU/device isolation |
| Virtual-device DMA | VM/device | Hypervisor + IOMMU mechanisms |

# Section 4 — Tools and commands

| Tool | Command | What it finds/shows | When to use it |
|---|---|---|---|
| `lspci` | `lspci -nn` | PCI/PCIe devices | Identify DMA-capable hardware |
| `lspci` | `lspci -vv` | Detailed PCI device capabilities | Inspect device/bus configuration |
| `dmesg` | `dmesg \| grep -Ei 'iommu\|dma'` | Kernel DMA/IOMMU messages | Check kernel initialization |
| `journalctl` | `journalctl -k \| grep -Ei 'iommu\|dma'` | Kernel boot messages | Review DMA protection events |
| `find` | `find /sys/kernel/iommu_groups -maxdepth 2 -type l` | IOMMU device groups | Determine device isolation |
| `ls` | `ls -lah /sys/kernel/iommu_groups/` | IOMMU groups | Inspect isolation topology |
| `lspci` | `lspci -t` | PCI topology | Understand device relationships |
| `dmesg` | `dmesg \| grep -Ei 'DMAR\|IOMMU\|AMD-Vi'` | Intel/AMD IOMMU initialization | Verify hardware virtualization/IOMMU messages |

Example:

```bash
$ lspci -nn
00:00.0 Host bridge ...
00:02.0 VGA compatible controller ...
00:14.0 USB controller ...
00:1f.3 Audio device ...
01:00.0 Ethernet controller ...
```

This establishes which PCI/PCIe devices exist.

Do not conclude that every listed device is independently DMA-accessible merely because it appears in `lspci`.

Example:

```bash
$ dmesg | grep -Ei 'iommu|dma'
[    0.000000] DMAR: IOMMU enabled
[    0.012345] iommu: Default domain type: Translated
```

This indicates that the kernel detected and enabled an IOMMU.

Exact messages vary by kernel and hardware.

Example:

```bash
$ ls /sys/kernel/iommu_groups/
0  1  2  3  4  5
```

This shows IOMMU groups exposed by the kernel.

You can inspect a group:

```bash
$ find /sys/kernel/iommu_groups/0/devices -maxdepth 1 -type l
.../0000:00:00.0
```

Interpretation:

```text
IOMMU group
   ↓
PCI device
   ↓
device isolation relationship
```

Example:

```bash
$ lspci -t
-[0000:00]-+-00.0
           +-02.0
           +-14.0
           \-1f.3
```

This shows the PCI topology rather than proving a particular security property.

Example:

```bash
$ journalctl -k | grep -Ei 'iommu|dma'
Sep 03 ... kernel: DMAR: IOMMU enabled
Sep 03 ... kernel: iommu: Default domain type: Translated
```

Use kernel logs to correlate the active protection state with boot initialization.

# Section 5 — Defender detection

- Monitor **PCI/Thunderbolt device insertion and authorization**, especially for systems where external PCIe-capable peripherals are possible.
- Monitor kernel logs for IOMMU initialization failures, DMA faults, or unexpected device-assignment changes.
- IOMMU fault telemetry can reveal a device attempting to access a memory region outside its permitted mapping.
- Maintain an inventory of PCIe devices and investigate unexpected hardware changes.
- Defenders commonly miss DMA because traditional endpoint monitoring focuses on processes and files rather than device-initiated memory transactions.
- Skilled operators using a physical DMA attack may generate little conventional process telemetry because the device, rather than a malicious process, initiates the memory access.
- Strong mitigation is **DMA isolation through IOMMU plus appropriate external-device authorization**, rather than relying only on application-level controls.

# Section 6 — Lab task

**Platform:** Local Kali Linux VM with hardware virtualization enabled. Use the VM to study the host/guest-visible PCI and IOMMU state; do not perform unauthorized DMA memory access.

**Objective:** Identify the system's DMA-capable PCI devices and determine whether the Linux environment exposes IOMMU-based isolation.

**Steps:**

1. Enumerate the PCI/PCIe devices visible to the Kali VM.
2. Identify devices that represent network, storage, graphics, or other high-speed hardware.
3. Inspect detailed PCI capabilities for several devices.
4. Search kernel messages for IOMMU and DMA initialization.
5. Inspect `/sys/kernel/iommu_groups/` if the environment exposes IOMMU groups.
6. Map at least two PCI devices to their IOMMU groups.
7. Compare the observed configuration with the expected DMA security model: device → IOMMU → permitted memory.
8. Document whether the VM exposes a real IOMMU and which parts of the physical hardware are hidden by virtualization.

**Expected output:**

```text
PCI devices
    ↓
DMA-capable hardware identified
    ↓
IOMMU status identified
    ↓
IOMMU groups inspected
    ↓
Device isolation documented
```

A valid result is **not** "I found DMA, therefore I hacked memory."

Success means you can demonstrate where DMA exists and where the platform places the isolation boundary.

**Git artifact:**

```text
dma-analysis/
├── README.md
├── inventory/
│   └── pci-devices.txt
├── evidence/
│   ├── iommu.txt
│   └── iommu-groups.txt
└── notes/
    └── dma-security-analysis.md
```

```bash
git add dma-analysis/
git commit -m "Add DMA and IOMMU security analysis lab"
```

# Section 7 — Common mistakes

1. **Thinking DMA means the CPU is completely bypassed** → The CPU normally configures the device and handles completion → Think "CPU does not copy every byte," not "CPU has nothing to do."

2. **Assuming every DMA-capable device can read all RAM** → IOMMUs can restrict device memory access → Determine the actual device-to-memory mapping.

3. **Confusing MMU and IOMMU** → MMU primarily controls CPU memory translation; IOMMU controls device memory access → Keep the initiator in mind: CPU versus device.

4. **Treating `lspci` output as proof of a vulnerability** → Seeing a PCI device only establishes hardware presence → Investigate DMA capability and isolation separately.

5. **Ignoring pre-boot DMA** → DMA protections may have different states during firmware/early boot → Include the boot phase when analyzing hardware attack surfaces.

6. **Assuming a VM exposes the host's complete DMA environment** → Hypervisors virtualize and restrict hardware visibility → Distinguish guest-visible hardware from the physical host's actual DMA topology.

7. **Testing DMA against a real system without understanding the consequences** → Unrestricted memory access can crash or corrupt the system → Perform DMA research only in controlled hardware/lab environments and begin with passive enumeration.

# Section 8 — Move-on gate

1. **Enumerate PCI/PCIe devices on your Kali lab and identify which devices are plausible DMA initiators, explaining the evidence for each without looking at your notes.**

2. **Determine whether IOMMU protection is active, map at least two devices to their IOMMU groups, and explain what the grouping tells you about device isolation.**

3. **Given a hypothetical PCIe device attempting to access unauthorized RAM, trace the expected path from device → DMA request → IOMMU → allowed/blocked memory access and identify exactly where the security boundary should stop the request.**