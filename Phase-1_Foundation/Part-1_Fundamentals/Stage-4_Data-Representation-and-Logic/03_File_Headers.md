# File Headers

**Roadmap:** Part 1 — Fundamentals → Stage 4 — Data Representation & Logic → File Headers

# Section 1 — What it is and where it sits

A **file header** contains metadata and structural information that helps software identify and interpret a file. One of the most useful pieces is the **magic bytes** (magic number): a characteristic byte sequence near the beginning of a file that identifies its format.

The filename extension is only a label. An attacker can rename `malware.exe` to `photo.jpg`; the underlying bytes do not change. Security tools therefore inspect file content, especially headers and structure, rather than trusting `.exe`, `.jpg`, `.pdf`, etc.

```text
Data Representation & Logic
└── Stage 4
    ├── Number Systems
    ├── Boolean Logic
    └── File Headers
         ├── Magic bytes
         ├── File signatures
         ├── EXE / PE
         ├── ELF
         └── JPG
              ↓
        File identification
              ↓
        Malware analysis
              ↓
        Forensics / detection
```

```text
File
 ↓
Read raw bytes
 ↓
Check signature / structure
 ↓
Identify format
 ↓
Parse format-specific metadata
 ↓
Analyze contents
```

If you skip this, you can be fooled by renamed files, misleading extensions, malformed uploads, polyglot files, and suspicious attachments. In security work, **"what does the filename claim to be?"** and **"what do the bytes prove it is?"** are different questions.

This follows binary/hexadecimal fundamentals and leads directly into file forensics, malware analysis, reverse engineering, and content-based detection.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers and defenders examine:

* First bytes of a file
* Magic numbers/signatures
* File format identifiers
* Header fields
* Architecture information
* Entry points
* Section/table offsets
* Embedded objects
* Trailing data
* Mismatches between extension and actual format

Typical signatures include:

```text
Windows PE / EXE → 4D 5A ("MZ")
ELF              → 7F 45 4C 46
JPEG             → FF D8 FF
PNG              → 89 50 4E 47 0D 0A 1A 0A
PDF              → 25 50 44 46 (" %PDF")
ZIP              → 50 4B 03 04
```

The important point is that a magic signature is **evidence**, not necessarily absolute proof. A malicious or deliberately malformed file can contain a recognizable signature without being a valid complete file of that format.

## 2.2 Extension vs actual content

Suppose:

```text
invoice.jpg
```

The extension says:

```text
JPG
```

But the first bytes are:

```text
4D 5A 90 00 ...
```

`4D 5A` corresponds to:

```text
M Z
```

That strongly suggests a Windows PE file.

The attacker may be attempting to exploit a system that trusts:

```text
filename = invoice.jpg
```

instead of validating:

```text
actual file content = PE executable
```

## 2.3 Identifying an ELF binary

A Linux executable commonly begins:

```text
7F 45 4C 46
```

In ASCII:

```text
7F  E  L  F
```

The first byte is the ELF magic value `0x7F`, followed by `ELF`.

After the signature, the header contains information such as:

* 32-bit vs 64-bit class
* Endianness
* ELF version
* Target architecture
* Entry point
* Program-header location
* Section-header location

This lets an analyst quickly determine whether a file is plausibly an executable and what architecture it targets.

## 2.4 Identifying a JPEG

JPEG files commonly begin with:

```text
FF D8 FF
```

The initial marker:

```text
FF D8
```

is the JPEG **Start of Image (SOI)** marker.

A common JPEG ends with:

```text
FF D9
```

Therefore an analyst can inspect both the beginning and structural markers rather than trusting `.jpg`.

## 2.5 Realistic attacker workflow

1. Obtain a suspicious file.
2. Ignore the extension initially.
3. Inspect the first bytes.
4. Recognize a known magic signature.
5. Determine the claimed format versus actual format.
6. Parse the appropriate header.
7. Check whether the rest of the file is structurally consistent.
8. Look for embedded or appended data.
9. Determine whether the file contains executable content or suspicious objects.
10. Feed the result into static analysis, sandboxing, or reverse engineering.

For example:

```text
Filename: image.jpg
        ↓
Extension says JPEG
        ↓
First bytes: 4D 5A
        ↓
Likely PE
        ↓
Inspect PE structure
        ↓
Determine whether executable content is present
```

## 2.6 Magic bytes are not enough

A common beginner mistake is:

```text
4D 5A = definitely malicious EXE
```

Wrong.

`MZ` identifies the traditional beginning of a DOS-compatible executable header used by PE files, but it does **not** mean the file is malicious.

Likewise:

```text
FF D8 FF = definitely safe JPEG
```

is also wrong.

An attacker can deliberately construct malformed or deceptive files.

The correct workflow is:

```text
Signature
   ↓
Format identification
   ↓
Structural validation
   ↓
Content analysis
   ↓
Security assessment
```

## 2.7 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
report.jpg has JPEG-like magic bytes.
```

That is expected for a legitimate JPEG and tells you little about maliciousness.

**High-value finding:**

```text
report.jpg has a PE signature and PE structure
despite being presented as an image.
```

This is immediately worth investigating because the file's declared type and actual structure disagree.

Another high-value finding:

```text
A supposedly ordinary JPEG contains substantial
data appended after its normal image structure.
```

That does not automatically mean malware, but it warrants analysis.

## 2.8 Where results feed next

```text
Magic bytes
    ↓
File identification
    ↓
Format-specific parser
    ↓
Header / sections / metadata
    ↓
Static analysis
    ↓
Malware / forensic analysis
```

# Section 3 — Core concepts and terminology

| Term           | Meaning                                                            |
| -------------- | ------------------------------------------------------------------ |
| File header    | Metadata/structural information near the beginning of a file       |
| Magic bytes    | Characteristic byte sequence identifying a file format             |
| Magic number   | Common term for a format-identifying value/signature               |
| File signature | Byte pattern used to identify a file type                          |
| File extension | Filename suffix such as `.exe` or `.jpg`                           |
| PE             | Portable Executable format used by Windows                         |
| ELF            | Executable and Linkable Format used by Linux and Unix-like systems |
| JPEG           | Image format commonly identified by `FF D8 FF` at the beginning    |
| MZ             | DOS executable header signature used at the start of PE files      |
| SOI            | JPEG Start of Image marker                                         |
| EOF marker     | Marker indicating a format's logical end where applicable          |
| Header offset  | Location of a structure within a file                              |
| Endianness     | Byte ordering used to represent multi-byte values                  |
| Parser         | Software that interprets a file according to its format            |
| MIME type      | Content-type label used by applications/protocols                  |
| Polyglot file  | File intentionally valid or interpretable as multiple formats      |
| Embedded data  | Data contained inside another file                                 |
| Trailing data  | Bytes appended after the expected file structure                   |

### Important signatures

| Format | Hex signature             | ASCII/meaning  |
| ------ | ------------------------- | -------------- |
| PE/EXE | `4D 5A`                   | `MZ`           |
| ELF    | `7F 45 4C 46`             | `ELF`          |
| JPEG   | `FF D8 FF`                | JPEG marker    |
| PNG    | `89 50 4E 47 0D 0A 1A 0A` | PNG signature  |
| PDF    | `25 50 44 46`             | `%PDF`         |
| ZIP    | `50 4B 03 04`             | `PK..`         |
| GIF    | `47 49 46 38`             | `GIF8`         |
| GZIP   | `1F 8B`                   | GZIP signature |

### PE structure

A simplified PE file looks like:

```text
┌──────────────────────┐
│ DOS Header           │
│ "MZ"                 │
├──────────────────────┤
│ DOS Stub             │
├──────────────────────┤
│ PE Signature         │
│ "PE\0\0"             │
├──────────────────────┤
│ COFF Header          │
├──────────────────────┤
│ Optional Header      │
├──────────────────────┤
│ Section Headers      │
├──────────────────────┤
│ .text                │
│ .rdata               │
│ .data                │
│ ...                  │
└──────────────────────┘
```

The `MZ` signature therefore identifies the beginning of the DOS header; the complete PE format has additional structures.

### ELF structure

```text
┌──────────────────────┐
│ ELF Header           │
│ 7F ELF               │
├──────────────────────┤
│ Program Headers      │
├──────────────────────┤
│ Sections             │
├──────────────────────┤
│ .text                │
│ .rodata              │
│ .data                │
│ ...                  │
└──────────────────────┘
```

### JPEG structure

Simplified:

```text
FF D8
  ↓
JPEG image segments
  ↓
Image data
  ↓
FF D9
```

Real JPEG structure contains multiple markers and segments; `FF D8` and `FF D9` are the important start/end markers for basic identification.

# Section 4 — Tools and commands

| Tool       | Command                     | What it finds/shows          | When to use it             |
| ---------- | --------------------------- | ---------------------------- | -------------------------- |
| `file`     | `file sample`               | Identifies file from content | First-pass identification  |
| `xxd`      | `xxd -l 32 sample`          | First bytes in hex/ASCII     | Inspect magic bytes        |
| `hexdump`  | `hexdump -C sample \| head` | Raw bytes                    | Manual signature analysis  |
| `xxd`      | `xxd -l 8 sample`           | Exact opening signature      | Fast header check          |
| `readelf`  | `readelf -h sample`         | ELF header                   | Analyze ELF binaries       |
| `objdump`  | `objdump -f sample`         | Binary architecture/format   | Executable analysis        |
| `strings`  | `strings -n 6 sample`       | Printable embedded strings   | Follow-up content analysis |
| `exiftool` | `exiftool sample.jpg`       | File metadata                | Image/document forensics   |

### `file`

```bash
file sample
```

Example:

```text
sample: ELF 64-bit LSB pie executable, x86-64, ...
```

`file` does not simply trust the extension. It examines content and recognizes known signatures/structures.

### `xxd`

```bash
xxd -l 16 sample
```

For an ELF binary:

```text
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000
```

The first four bytes are:

```text
7F 45 4C 46
```

Therefore the file begins with the ELF signature.

### PE/EXE header

```bash
xxd -l 16 program.exe
```

Example:

```text
00000000: 4d5a 9000 0300 0000 0400 0000 ffff 0000
```

The first two bytes are:

```text
4D 5A
```

which represent:

```text
MZ
```

This indicates a DOS-compatible executable header and is characteristic of PE files.

### JPEG

```bash
xxd -l 16 image.jpg
```

Example:

```text
00000000: ffd8 ffe0 0010 4a46 4946 0001 0100 ...
```

The opening:

```text
FF D8 FF
```

is consistent with JPEG.

### `readelf`

```bash
readelf -h program
```

Example:

```text
ELF Header:
  Class:                             ELF64
  Data:                              2's complement, little endian
  Type:                              DYN
  Machine:                           Advanced Micro Devices X86-64
```

This goes beyond magic-byte identification and validates important ELF header information.

### `objdump`

```bash
objdump -f program
```

Example:

```text
program:     file format elf64-x86-64
architecture: i386:x86-64
```

This confirms the executable format and architecture.

### `strings`

```bash
strings -n 6 sample
```

Example:

```text
GET /login
User-Agent
configuration
```

This is a follow-up technique. It does not identify the file format itself; it helps determine what content exists inside the identified file.

### `exiftool`

```bash
exiftool image.jpg
```

Example:

```text
File Type                       : JPEG
File Type Extension             : jpg
MIME Type                       : image/jpeg
Image Width                     : 1920
Image Height                    : 1080
```

Metadata can provide additional forensic context after the file type has been identified.

# Section 5 — Defender detection

* File-upload systems should validate **content**, not merely trust the filename extension or client-provided MIME type.
* SIEM/EDR pipelines can flag mismatches such as `.jpg` files whose content identifies as PE/ELF.
* YARA rules commonly inspect magic bytes and deeper structural characteristics to classify suspicious files.
* Defenders commonly miss malicious files because they validate only extensions such as `.jpg`, `.pdf`, or `.docx`.
* A signature match alone is insufficient; malformed headers, impossible offsets, suspicious sections, and appended data can provide stronger evidence.
* Skilled attackers may deliberately manipulate headers, use polyglots, or append data to make simplistic file-type checks unreliable.
* Strong validation combines magic bytes, structural parsing, MIME/content checks, file-size limits, metadata analysis, and sandbox/static analysis where appropriate.

# Section 6 — Lab task

**Platform:** Kali Linux VM using benign local files available on the system.

**Objective:** Identify files by their actual content signatures and demonstrate that extensions are not sufficient for determining file type.

**Steps:**

1. Create a working directory for the investigation.
2. Copy one ELF executable into it.
3. Copy one JPEG image into it.
4. Copy one other known file format such as PNG or PDF.
5. Rename the ELF file with a `.jpg` extension.
6. Run `file` against all files.
7. Inspect the first 16 bytes of each file with `xxd`.
8. Match the observed signatures against known magic bytes.
9. Compare the extension with the detected content type.
10. Record the results and explain the mismatch.

Example:

```bash
mkdir file-header-lab
cd file-header-lab
```

For the renamed executable:

```bash
file fake.jpg
xxd -l 16 fake.jpg
```

**Expected output:**

Even though:

```text
fake.jpg
```

has a `.jpg` extension, `file` should identify the underlying executable format if the copied file remains intact.

The hex output should begin with something resembling:

```text
00000000: 7f45 4c46 ...
```

for an ELF binary, or:

```text
00000000: 4d5a ...
```

for a PE file.

**Git artifact:**

```text
file-header-lab/
├── README.md
├── analysis.md
└── evidence/
    ├── file-output.txt
    ├── elf-header.txt
    └── renamed-file.txt
```

Commit:

```bash
git add file-header-lab/
git commit -m "Add file header and magic byte analysis lab"
```

# Section 7 — Common mistakes

1. **Mistake → Trusting the file extension.**
   **Why it matters →** Extensions are user-controlled labels and can be changed trivially.
   **Instead →** Inspect magic bytes and validate the actual file structure.

2. **Mistake → Assuming a magic-byte match proves the entire file is valid.**
   **Why it matters →** An attacker can place a recognizable signature into malformed data.
   **Instead →** Validate the complete format structure.

3. **Mistake → Thinking `MZ` means "malware."**
   **Why it matters →** `MZ` is a file-format signature, not a maliciousness indicator.
   **Instead →** Separate format identification from security assessment.

4. **Mistake → Memorizing signatures without understanding hexadecimal.**
   **Why it matters →** Reverse engineering constantly requires interpreting surrounding bytes.
   **Instead →** Use your binary/hex skills to inspect the complete header.

5. **Mistake → Checking only the first four bytes.**
   **Why it matters →** Many formats contain additional structural fields that must be internally consistent.
   **Instead →** Move from signature → header → complete structure.

6. **Mistake → Assuming every file has a unique magic signature.**
   **Why it matters →** Some formats have weak or ambiguous signatures, and some lack reliable magic bytes.
   **Instead →** Combine signatures with structural and metadata analysis.

# Section 8 — Move-on gate

1. **Signature test:** inspect an unknown file with `xxd -l 16` and correctly identify whether it begins like an ELF, PE, JPEG, PNG, PDF, or ZIP file without using the filename.

2. **Extension-bypass test:** rename a known ELF or PE file to `.jpg`, run `file` and `xxd`, and correctly identify the actual format from its content.

3. **Forensic test:** take three different files, record their first 16 bytes, identify their signatures, and explain which bytes establish the format and which subsequent bytes belong to the format-specific header.
