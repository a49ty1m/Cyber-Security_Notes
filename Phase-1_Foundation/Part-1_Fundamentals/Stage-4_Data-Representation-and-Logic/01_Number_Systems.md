# Number Systems

**Roadmap:** Part 1 — Fundamentals → Stage 4 — Data Representation & Logic → Number Systems

# Section 1 — What it is and where it sits

Number systems are the different ways computers and humans represent numeric values. For cybersecurity, the three most important are **binary (base 2)**, **decimal (base 10)**, and **hexadecimal (base 16)**. Binary is the underlying representation used by digital hardware; hexadecimal is a compact human-readable representation of binary.

You need to convert between them quickly because security tooling constantly exposes values in different forms: IP addresses, packet fields, file offsets, memory addresses, permissions, hashes, machine instructions, and bitmasks.

```text
Data Representation & Logic
└── Stage 4
    └── Number Systems
         ├── Binary
         ├── Decimal
         └── Hexadecimal
              ↓
        Bits / bytes / addresses
              ↓
        Networking / OS / reversing
              ↓
        Exploitation & forensics
```

If you cannot mentally recognize values such as `0xFF = 255 = 11111111`, low-level security work becomes unnecessarily slow. You will constantly stop to calculate basic representations instead of reasoning about the actual system.

This follows your OS/memory fundamentals and becomes the numerical foundation for binary analysis, networking, bitwise operations, memory addresses, and cryptographic data.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers constantly encounter numeric values represented as:

```text
Binary      → 11010110
Decimal     → 214
Hexadecimal → 0xD6
```

The representation changes, but the underlying value does not.

Important examples include:

* Memory addresses
* Register values
* Opcode bytes
* File offsets
* Network addresses
* Port numbers
* Subnet masks
* Permission bitmasks
* Protocol fields
* Character encodings
* Magic numbers
* Exploit offsets

## 2.2 Reading a memory address

A debugger might show:

```text
0x7fffffffe120
```

You need to recognize immediately:

```text
0x → hexadecimal
```

Each hexadecimal digit represents exactly **4 bits**.

Therefore:

```text
0x7
= 0111

0xF
= 1111
```

You can mentally expand:

```text
0x7F
 ↓
0111 1111
```

This is much faster than converting the entire address to decimal.

## 2.3 Reading byte values

Suppose a packet or binary contains:

```text
48 65 6c 6c 6f
```

These are hexadecimal bytes.

Convert them mentally:

```text
48 → 72
65 → 101
6c → 108
6c → 108
6f → 111
```

Those decimal values correspond to ASCII:

```text
72  = H
101 = e
108 = l
108 = l
111 = o
```

Therefore:

```text
48 65 6c 6c 6f
       ↓
     Hello
```

This type of recognition is routine in packet analysis, reverse engineering, and forensic work.

## 2.4 Binary bitmasks

Attackers and defenders frequently encounter flags packed into individual bits.

For example:

```text
00000101
```

means:

```text
128 64 32 16 8 4 2 1
 0   0  0  0 0 1 0 1
```

Therefore:

```text
4 + 1 = 5
```

So:

```text
00000101₂ = 5₁₀ = 0x05
```

This becomes important when analyzing permission flags, protocol fields, CPU flags, and security configuration values.

## 2.5 Hexadecimal makes binary manageable

Consider:

```text
1010111111000011
```

Reading 16 bits directly is annoying.

Group into four:

```text
1010 1111 1100 0011
```

Convert each group:

```text
1010 → A
1111 → F
1100 → C
0011 → 3
```

Therefore:

```text
1010111111000011₂
=
0xAFC3
```

The attacker can now reason about the same 16-bit value much more compactly.

## 2.6 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
0x41
```

By itself, this is just a hexadecimal byte.

**High-value finding:**

```text
0x41 0x42 0x43
```

Recognizing immediately:

```text
41 → A
42 → B
43 → C
```

can reveal that a supposedly binary region actually contains:

```text
ABC
```

The security value comes from correctly interpreting the representation in context.

## 2.7 Realistic attacker workflow

1. Encounter a value in a tool's output.
2. Identify its base from syntax or context.
3. Determine whether it represents a byte, word, integer, address, offset, or bitmask.
4. Convert it mentally when necessary.
5. Break hexadecimal values into 4-bit groups when binary interpretation is required.
6. Interpret individual bits when analyzing flags.
7. Convert between decimal and hexadecimal when comparing with documentation or tool output.
8. Use the interpreted value to understand the protocol, binary, memory layout, or security control.
9. Continue analysis without repeatedly reaching for a calculator.

## 2.8 Where results feed next

```text
Binary / Decimal / Hex
          ↓
Bits and bytes
          ↓
Memory addresses
          ↓
Machine code
          ↓
Registers / flags
          ↓
Reverse engineering
          ↓
Exploitation & forensics
```

# Section 3 — Core concepts and terminology

| Term        | Meaning                                       |
| ----------- | --------------------------------------------- |
| Bit         | Single binary digit: `0` or `1`               |
| Byte        | 8 bits                                        |
| Nibble      | 4 bits                                        |
| Binary      | Base-2 number system                          |
| Decimal     | Base-10 number system                         |
| Hexadecimal | Base-16 number system                         |
| Base        | Number of symbols used by a positional system |
| Digit       | One symbol representing a value               |
| Place value | Value contributed by a digit's position       |
| MSB         | Most Significant Bit                          |
| LSB         | Least Significant Bit                         |
| Prefix      | Notation identifying a number's base          |
| Bitmask     | Value whose bits represent independent flags  |
| Integer     | Whole-number value                            |
| Word        | Architecture-dependent data unit              |

### The three systems

| System      | Base | Digits     | Common prefix |
| ----------- | ---: | ---------- | ------------- |
| Binary      |    2 | `0–1`      | `0b`          |
| Decimal     |   10 | `0–9`      | None          |
| Hexadecimal |   16 | `0–9, A–F` | `0x`          |

Hexadecimal digits:

```text
Decimal → Hex

0 → 0
1 → 1
2 → 2
3 → 3
4 → 4
5 → 5
6 → 6
7 → 7
8 → 8
9 → 9
10 → A
11 → B
12 → C
13 → D
14 → E
15 → F
```

### Binary ↔ hexadecimal mapping

Memorize this table.

| Binary | Hex | Decimal |
| ------ | --: | ------: |
| `0000` | `0` |       0 |
| `0001` | `1` |       1 |
| `0010` | `2` |       2 |
| `0011` | `3` |       3 |
| `0100` | `4` |       4 |
| `0101` | `5` |       5 |
| `0110` | `6` |       6 |
| `0111` | `7` |       7 |
| `1000` | `8` |       8 |
| `1001` | `9` |       9 |
| `1010` | `A` |      10 |
| `1011` | `B` |      11 |
| `1100` | `C` |      12 |
| `1101` | `D` |      13 |
| `1110` | `E` |      14 |
| `1111` | `F` |      15 |

### Powers of two

Memorize these:

```text
2⁰  = 1
2¹  = 2
2²  = 4
2³  = 8
2⁴  = 16
2⁵  = 32
2⁶  = 64
2⁷  = 128
2⁸  = 256
2¹⁶ = 65536
```

This makes binary → decimal conversion much faster.

### Binary → decimal

Example:

```text
101101₂
```

Write the powers:

```text
32 16 8 4 2 1
 1  0 1 1 0 1
```

Therefore:

```text
32 + 8 + 4 + 1
= 45
```

So:

```text
101101₂ = 45₁₀
```

### Decimal → binary

Example:

```text
45
```

Largest powers of two:

```text
32 → remaining 13
8  → remaining 5
4  → remaining 1
1  → remaining 0
```

Therefore:

```text
45 = 32 + 8 + 4 + 1
```

So:

```text
45₁₀ = 101101₂
```

### Hexadecimal → decimal

Example:

```text
0x2D
```

Hex positions represent powers of 16:

```text
2 × 16¹ + D × 16⁰
```

Since:

```text
D = 13
```

Then:

```text
2 × 16 + 13
= 32 + 13
= 45
```

Therefore:

```text
0x2D = 45
```

### Decimal → hexadecimal

Example:

```text
45
```

Divide by 16:

```text
45 ÷ 16 = 2 remainder 13
```

`13 = D`.

Therefore:

```text
45₁₀ = 0x2D
```

### The mental shortcut

The most important relationship is:

```text
1 hex digit = 4 bits
2 hex digits = 1 byte
```

Examples:

```text
F    → 1111
A    → 1010
7    → 0111

FF   → 11111111
80   → 10000000
C0   → 11000000
```

# Section 4 — Tools and commands

| Tool             | Command                     | What it finds/shows   | When to use it           |
| ---------------- | --------------------------- | --------------------- | ------------------------ |
| `printf`         | `printf '%d\n' $((16#2D))`  | Hex → decimal         | Verify conversions       |
| `printf`         | `printf '%X\n' 45`          | Decimal → hex         | Verify conversions       |
| Bash arithmetic  | `echo $((2#101101))`        | Binary → decimal      | Verify mental conversion |
| `bc`             | `echo "obase=16; 45" \| bc` | Decimal → hex         | Larger conversions       |
| `bc`             | `echo "ibase=16; 2D" \| bc` | Hex → decimal         | Hex verification         |
| `xxd`            | `xxd file.bin`              | Hexadecimal byte view | Binary/file analysis     |
| `hexdump`        | `hexdump -C file.bin`       | Hex + ASCII view      | Forensics/reversing      |
| Python-free `od` | `od -An -tx1 file.bin`      | Hexadecimal bytes     | Raw byte inspection      |

### Bash: binary → decimal

```bash
echo $((2#101101))
```

Output:

```text
45
```

Interpretation:

```text
101101₂ = 45₁₀
```

### Bash: hexadecimal → decimal

```bash
printf '%d\n' $((16#2D))
```

Output:

```text
45
```

### Bash: decimal → hexadecimal

```bash
printf '%X\n' 45
```

Output:

```text
2D
```

### `bc`

```bash
echo "obase=16; 45" | bc
```

Output:

```text
2D
```

Reverse:

```bash
echo "ibase=16; 2D" | bc
```

Output:

```text
45
```

### `xxd`

```bash
xxd file.bin
```

Example:

```text
00000000: 4865 6c6c 6f20 776f 726c 64              Hello world
```

The left side is the offset.

The middle is hexadecimal.

The right side is the printable interpretation.

### `hexdump`

```bash
hexdump -C file.bin
```

Example:

```text
00000000  48 65 6c 6c 6f 20 77 6f  72 6c 64 0a  |Hello world.|
```

This is particularly useful when inspecting unknown binary data.

# Section 5 — Defender detection

This topic is fundamentally **representational rather than an attack technique**, so there is no unique server-side event or detection signature for converting numbers between bases.

* Defenders use hexadecimal and binary representations heavily when interpreting packet captures, memory dumps, logs, and malware artifacts.
* Suspicious byte patterns, magic numbers, encoded configuration values, and unusual bit flags can become indicators during analysis.
* Analysts commonly miss meaningful data because they inspect bytes only as decimal or only as printable text.
* Skilled operators may represent the same value differently to make analysis less obvious, but changing the numerical representation does not change the underlying bytes.
* Detection therefore depends on interpreting the data correctly rather than detecting "hexadecimal usage."

# Section 6 — Lab task

**Platform:** Kali Linux VM.

**Objective:** Prove that you can convert common cybersecurity-relevant values between binary, decimal, and hexadecimal and verify them using Linux tools.

**Steps:**

1. Create a list of ten values without calculators.
2. Convert each decimal value to binary and hexadecimal.
3. Convert each hexadecimal value back to decimal.
4. Convert the binary values back to decimal.
5. Create a small binary file containing known bytes.
6. Inspect the file with `xxd`.
7. Pick five bytes from the output.
8. Convert those bytes mentally to decimal and binary.
9. Verify your answers with Bash arithmetic.
10. Record your accuracy and conversion time.

Use these test values:

```text
Decimal:
15
31
45
64
127
128
170
192
255
512
```

You should be able to recognize:

```text
15  → 0x0F → 00001111
45  → 0x2D → 00101101
64  → 0x40 → 01000000
127 → 0x7F → 01111111
128 → 0x80 → 10000000
170 → 0xAA → 10101010
192 → 0xC0 → 11000000
255 → 0xFF → 11111111
```

**Expected output:**

Your final table should resemble:

```text
Decimal    Hex     Binary
15         0F      00001111
31         1F      00011111
45         2D      00101101
64         40      01000000
127        7F      01111111
128        80      10000000
170        AA      10101010
192        C0      11000000
255        FF      11111111
512        200     1000000000
```

The important success criterion is **mental conversion first, tool verification second**.

**Git artifact:**

```text
number-systems-lab/
├── README.md
├── conversions.md
└── evidence/
    └── hexdump.txt
```

Commit:

```bash
git add number-systems-lab/
git commit -m "Add binary decimal hexadecimal conversion lab"
```

# Section 7 — Common mistakes

1. **Mistake → Forgetting that hexadecimal is base 16.**
   **Why it matters →** `0x10` is not decimal 10; it is decimal 16.
   **Instead →** Always think in powers of 16.

2. **Mistake → Treating each hex digit as a byte.**
   **Why it matters →** One hex digit is only 4 bits.
   **Instead →** Remember `2 hex digits = 1 byte`.

3. **Mistake → Dropping leading zeroes in binary.**
   **Why it matters →** `0x0F` and `0xF` represent the same value, but byte-level analysis often requires `00001111`.
   **Instead →** Represent bytes using exactly 8 bits.

4. **Mistake → Converting binary digit-by-digit instead of using powers of two.**
   **Why it matters →** It is slower and increases errors.
   **Instead →** Memorize powers of two through at least `2⁸`.

5. **Mistake → Converting everything to decimal first.**
   **Why it matters →** This defeats the purpose of hexadecimal's compact binary representation.
   **Instead →** Convert hex ↔ binary directly through 4-bit groups.

6. **Mistake → Confusing a value's representation with its meaning.**
   **Why it matters →** `0x41` could represent decimal 65, ASCII `A`, a protocol field, or part of an address depending on context.
   **Instead →** Identify the data type and context before interpreting the number.

# Section 8 — Move-on gate

1. **Mental conversion test:** convert `0x3A`, `0x7F`, `0xC0`, `0xE1`, and `0xFF` to binary and decimal without using a calculator or conversion tool.

2. **Reverse conversion test:** convert `37`, `85`, `127`, `170`, and `255` from decimal to hexadecimal and binary mentally, then verify your answers with Bash.

3. **Security-data test:** run `xxd` against a binary file, select five hexadecimal bytes, and correctly convert each into its 8-bit binary representation and decimal value without looking at notes.
