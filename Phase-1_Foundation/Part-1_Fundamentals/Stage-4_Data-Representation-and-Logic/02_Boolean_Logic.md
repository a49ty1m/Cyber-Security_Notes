# Boolean Logic

**Roadmap:** Part 1 — Fundamentals → Stage 4 — Data Representation & Logic → Boolean Logic

# Section 1 — What it is and where it sits

**Boolean logic** operates on two possible values: **TRUE/1** and **FALSE/0**. The four operations you need to master for low-level security work are **AND, OR, NOT, and XOR**. They are implemented directly or indirectly through CPU instructions and are fundamental to bit manipulation, permissions, protocol flags, checksums, cryptographic constructions, and obfuscation.

For cybersecurity, the important distinction is that Boolean logic can operate on individual logical values **and on every bit of multi-bit values**.

```text
Data Representation & Logic
└── Stage 4
    ├── Number Systems
    └── Boolean Logic
         ├── AND
         ├── OR
         ├── NOT
         └── XOR
              ↓
        Bitwise operations
              ↓
      Flags / masks / crypto
              ↓
      Reverse engineering
              ↓
       Exploitation
```

If you skip this, bitmasks, permission flags, packet fields, XOR-based transformations, and low-level code will constantly look more complicated than they actually are.

Number systems taught you how to read the bits; Boolean logic teaches you how operations transform those bits.

# Section 2 — How attackers actually use this

## 2.1 What attackers are looking for

Attackers encounter Boolean operations when analyzing:

* CPU instructions
* Binary file formats
* Protocol flags
* Permission bitmasks
* Malware configuration
* Obfuscated strings
* Encryption routines
* Checksums
* Feature flags
* Conditional branches
* Cryptographic code
* Exploit payloads

The key question is:

> **Which bits are being tested, cleared, set, inverted, or combined?**

genui{"learning_viz":{"type_id":"BOOLEAN_LOGIC"}}

## 2.2 AND — isolate or test bits

AND produces `1` only when **both corresponding inputs are `1`**.

```text
A B | A AND B
----+--------
0 0 |    0
0 1 |    0
1 0 |    0
1 1 |    1
```

The major security use is **masking**.

Suppose:

```text
value = 10110110
mask  = 00001111
```

AND them:

```text
10110110
AND 00001111
-----------
00000110
```

The upper four bits were removed.

This is useful when a protocol stores several fields inside one byte:

```text
┌──────┬──────┐
│ flags│ value│
└──────┴──────┘
```

An attacker or analyst can isolate the field they care about using an AND mask.

## 2.3 OR — set selected bits

OR produces `1` when **at least one corresponding input is `1`**.

```text
A B | A OR B
----+-------
0 0 |   0
0 1 |   1
1 0 |   1
1 1 |   1
```

Example:

```text
value = 10000001
mask  = 00000100
```

OR:

```text
10000001
OR  00000100
-----------
10000101
```

The selected bit was forced to `1`.

This is commonly used for:

* Enabling flags
* Setting permission bits
* Constructing protocol fields
* Combining feature flags

## 2.4 NOT — invert bits

NOT reverses each bit:

```text
0 → 1
1 → 0
```

Example:

```text
NOT 10110010
    --------
    01001101
```

For an 8-bit value:

```text
~10110010 = 01001101
```

In programming, the result depends on the operand's width and integer representation, so always consider the data type.

## 2.5 XOR — different bits become 1

XOR produces `1` when the two corresponding bits are **different**.

```text
A B | A XOR B
----+---------
0 0 |    0
0 1 |    1
1 0 |    1
1 1 |    0
```

Example:

```text
10110110
XOR 01010101
-----------
11100011
```

XOR has an extremely important property:

```text
A XOR B XOR B = A
```

For example:

```text
10110110
XOR 01010101
-----------
11100011

11100011
XOR 01010101
-----------
10110110
```

This is why XOR appears frequently in reversible transformations.

## 2.6 XOR and encryption — important correction

It is correct that XOR is foundational to **many cryptographic constructions**, but saying:

> "XOR is encryption"

would be wrong.

Simple XOR with a repeating or predictable key is usually **obfuscation, not secure encryption**.

For example:

```text
plaintext XOR key = ciphertext
ciphertext XOR key = plaintext
```

If the same key is reused improperly, attackers can often exploit statistical relationships.

A **one-time pad** is a special case where XOR can provide information-theoretic security when its strict requirements are satisfied.

Modern encryption algorithms also use XOR extensively, but security comes from the complete construction, not XOR alone.

## 2.7 Obfuscation example

Suppose a malware sample stores:

```text
Plaintext:
HELLO
```

It could XOR each byte with a constant:

```text
plaintext byte XOR key
```

The resulting bytes may no longer look like readable ASCII.

An analyst seeing:

```text
1A 0F 12 12 15
```

might suspect an encoded/obfuscated string.

If the key is discovered, XOR can be reversed because:

```text
ciphertext XOR key = plaintext
```

This is why XOR patterns are common during reverse engineering.

## 2.8 Bit flags

Suppose a program stores four security states in one byte:

```text
0000 0000
|||| ||||
|||| |||└─ DEBUG
|||| ||└── LOGGING
|||| |└─── NETWORK
|||| └──── ADMIN
```

An analyst wants to determine whether `ADMIN` is enabled.

Use a mask:

```text
flags = 00001011
mask  = 00001000
```

Then:

```text
00001011
AND 00001000
-----------
00001000
```

Non-zero result:

```text
ADMIN = enabled
```

This pattern appears everywhere in systems programming.

## 2.9 Dead-end finding vs high-value finding

**Dead-end finding:**

```text
The binary contains XOR instructions.
```

XOR is extremely common. This tells you almost nothing by itself.

**High-value finding:**

```text
A loop XORs every byte of a suspicious string
with the same constant key, and XORing the bytes
again with that key produces readable configuration data.
```

That is valuable because you have identified a reversible transformation and recovered meaningful data.

## 2.10 Realistic attacker workflow

1. Identify a suspicious Boolean/bitwise operation.
2. Determine the operand width.
3. Identify whether the instruction operates on individual bits or entire machine words.
4. Trace the input values.
5. Determine whether the operation is AND, OR, NOT, or XOR.
6. For AND/OR, determine whether a bitmask is being used.
7. For XOR, check whether a repeated key or transformation pattern exists.
8. Reverse the transformation where appropriate.
9. Interpret the resulting bytes in context.
10. Use the recovered information to understand the program's next behavior.

## 2.11 Where results feed next

```text
Boolean operations
       ↓
Bit manipulation
       ↓
Flags / masks
       ↓
Binary analysis
       ↓
Obfuscation analysis
       ↓
Reverse engineering
       ↓
Exploit / malware analysis
```

# Section 3 — Core concepts and terminology

| Term              | Meaning                                                                                                 |
| ----------------- | ------------------------------------------------------------------------------------------------------- |
| Boolean           | Value represented as TRUE/FALSE or 1/0                                                                  |
| AND               | Produces 1 only when both inputs are 1                                                                  |
| OR                | Produces 1 when at least one input is 1                                                                 |
| NOT               | Inverts each input bit                                                                                  |
| XOR               | Produces 1 when inputs differ                                                                           |
| Bitwise operation | Boolean operation performed independently on corresponding bits                                         |
| Logical operation | Boolean operation treating values as logical conditions                                                 |
| Bitmask           | Value used to select or modify specific bits                                                            |
| Flag              | Individual bit representing a state or option                                                           |
| Set bit           | Change a bit to `1`                                                                                     |
| Clear bit         | Change a bit to `0`                                                                                     |
| Toggle bit        | Invert a bit                                                                                            |
| Bitwise AND       | AND applied independently to each bit                                                                   |
| Bitwise OR        | OR applied independently to each bit                                                                    |
| Bitwise NOT       | Invert every bit in the operand                                                                         |
| Bitwise XOR       | XOR applied independently to each bit                                                                   |
| XOR key           | Value used in an XOR transformation                                                                     |
| Obfuscation       | Making data/code harder to interpret without necessarily providing cryptographic security               |
| One-time pad      | XOR-based cryptographic construction using a truly random, secret, non-reused key of appropriate length |

### Truth tables

|  A |  B | AND | OR | XOR |
| -: | -: | --: | -: | --: |
|  0 |  0 |   0 |  0 |   0 |
|  0 |  1 |   0 |  1 |   1 |
|  1 |  0 |   0 |  1 |   1 |
|  1 |  1 |   1 |  1 |   0 |

NOT:

|  A | NOT A |
| -: | ----: |
|  0 |     1 |
|  1 |     0 |

### The four operations in one view

```text
AND → keep/test bits
OR  → set bits
NOT → invert bits
XOR → toggle/combine/difference
```

### Essential XOR properties

```text
A XOR 0 = A
A XOR A = 0
A XOR B = B XOR A
(A XOR B) XOR B = A
```

The last property is especially important when reversing XOR transformations.

### Masking patterns

**Test a bit:**

```text
value & mask
```

**Set a bit:**

```text
value | mask
```

**Toggle a bit:**

```text
value ^ mask
```

**Clear selected bits:**

```text
value & ~mask
```

Example:

```text
value = 10110110
mask  = 00000100
```

Toggle:

```text
10110110
XOR 00000100
-----------
10110010
```

The selected bit changed from `1` to `0`.

# Section 4 — Tools and commands

| Tool      | Command                  | What it finds/shows             | When to use it           |
| --------- | ------------------------ | ------------------------------- | ------------------------ |
| Bash      | `echo $((0xB6 & 0x0F))`  | Bitwise AND                     | Quick mask calculation   |
| Bash      | `echo $((0x80 \| 0x04))` | Bitwise OR                      | Quick flag calculation   |
| Bash      | `echo $((0xB6 ^ 0x55))`  | Bitwise XOR                     | XOR analysis             |
| Bash      | `echo $((~0xB2 & 0xFF))` | 8-bit NOT                       | Fixed-width inversion    |
| `gdb`     | `p/x $rax`               | Register value in hex           | Reverse engineering      |
| `gdb`     | `x/16bx ADDRESS`         | Bytes at memory address         | Inspect transformed data |
| `objdump` | `objdump -d ./program`   | Bitwise assembly instructions   | Static analysis          |
| `xxd`     | `xxd file.bin`           | Hexadecimal byte representation | Inspect encoded data     |

### Bash AND

```bash
echo $((0xB6 & 0x0F))
```

Output:

```text
6
```

Because:

```text
B6 = 10110110
0F = 00001111

AND
   00000110
```

So:

```text
0xB6 & 0x0F = 0x06
```

### Bash OR

```bash
echo $((0x80 | 0x04))
```

Output:

```text
132
```

Because:

```text
80 = 10000000
04 = 00000100
     --------
     10000100
```

`10000100₂ = 132`.

### Bash XOR

```bash
echo $((0xB6 ^ 0x55))
```

Output:

```text
227
```

Because:

```text
B6 = 10110110
55 = 01010101
     --------
     11100011
```

`11100011₂ = 0xE3 = 227`.

### Bash NOT

Use a fixed width when reasoning about bytes:

```bash
echo $((~0xB2 & 0xFF))
```

Output:

```text
77
```

Because:

```text
B2 = 10110010
NOT  = 01001101
```

Therefore:

```text
0x4D = 77
```

The `& 0xFF` matters because Bash integers are wider than 8 bits.

### GDB register inspection

```bash
gdb -q ./program
```

Then:

```text
(gdb) p/x $rax
$1 = 0xb6
```

You can immediately interpret:

```text
0xB6 = 10110110
```

This is useful when tracing bitwise instructions.

### GDB memory inspection

```text
(gdb) x/16bx ADDRESS
```

Example:

```text
0x555555559000:
0x48 0x65 0x6c 0x6c 0x6f 0x00
```

If you are investigating an XOR transformation, this lets you compare the raw bytes before and after the operation.

### `objdump`

```bash
objdump -d ./program
```

Look for instructions such as:

```asm
and    $0xf,%eax
or     $0x4,%eax
xor    $0x55,%eax
not    %eax
```

Interpretation:

```text
and → mask/test
or  → set bits
xor → combine/toggle
not → invert
```

### `xxd`

```bash
xxd suspicious.bin
```

Example:

```text
00000000: 1a 0f 12 12 15 55  ... 
```

If repeated XOR with a candidate key transforms these bytes into readable ASCII, you may have identified a simple obfuscation layer.

# Section 5 — Defender detection

* AND, OR, NOT, and XOR are normal CPU operations, so their presence is not itself suspicious.
* Repeated XOR loops over strings or configuration data can be a useful behavioral clue during malware reverse engineering.
* Static analysis can identify constant XOR keys, repeated transformations, and bitmask-heavy control logic.
* Defenders commonly miss encoded strings because they inspect the binary only as printable text.
* EDR generally detects the **behavior produced by** a malicious transformation rather than "XOR instructions" themselves.
* Skilled operators may use simple transformations to hide strings or configuration, but weak constant/repeating-key XOR is usually reversible through analysis.
* Analysts should compare raw bytes, operation sequence, key reuse, and resulting plaintext rather than assuming every XOR operation represents encryption.

# Section 6 — Lab task

**Platform:** Kali Linux VM with a local C program containing a simple XOR-obfuscated string.

**Objective:** Recover an XOR-obfuscated string by identifying the operation and reversing it.

**Steps:**

1. Create a C program containing a string encoded with a single-byte XOR key.
2. Store the encoded bytes in an array.
3. Compile the program with debugging information.
4. Run it and inspect the encoded output.
5. Disassemble the program and identify the XOR instruction.
6. Determine the XOR key from the program's logic.
7. Apply XOR with the same key to the encoded bytes.
8. Verify that the original plaintext is recovered.
9. Inspect the bytes with `xxd` or GDB.
10. Document the transformation.

Example:

```c
#include <stdio.h>

int main(void)
{
    unsigned char data[] = {
        0x1D, 0x10, 0x19, 0x19, 0x1A
    };

    unsigned char key = 0x55;

    for (int i = 0; i < 5; i++)
        printf("%c", data[i] ^ key);

    printf("\n");

    return 0;
}
```

Compile:

```bash
gcc -g -O0 xor_demo.c -o xor_demo
```

Run:

```bash
./xor_demo
```

**Expected output:**

```text
HELLO
```

Because:

```text
1D XOR 55 = 48 → H
10 XOR 55 = 45 → E
19 XOR 55 = 4C → L
19 XOR 55 = 4C → L
1A XOR 55 = 4F → O
```

The important skill is not memorizing this particular key. It is recognizing:

```text
encoded byte XOR key = plaintext byte
```

**Git artifact:**

```text
boolean-logic-lab/
├── README.md
├── src/
│   └── xor_demo.c
└── evidence/
    ├── disassembly.txt
    └── recovered-string.md
```

Commit:

```bash
git add boolean-logic-lab/
git commit -m "Add Boolean logic and XOR analysis lab"
```

# Section 7 — Common mistakes

1. **Mistake → Confusing XOR with secure encryption.**
   **Why it matters →** Simple XOR with a reused or predictable key is easily attacked.
   **Instead →** Treat simple XOR as an encoding/obfuscation technique unless it is part of a proper cryptographic construction.

2. **Mistake → Forgetting that bitwise operations work independently on corresponding bits.**
   **Why it matters →** `1010 XOR 1100` is calculated bit-by-bit.
   **Instead →** Align the operands vertically.

3. **Mistake → Using NOT without considering integer width.**
   **Why it matters →** `~0xB2` on a normal machine integer does not produce only an 8-bit result.
   **Instead →** Mask with `0xFF` when intentionally working with one byte.

4. **Mistake → Confusing OR with XOR.**
   **Why it matters →** OR keeps a bit set when either input is `1`; XOR clears `1 XOR 1`.
   **Instead →** Memorize `1 OR 1 = 1`, but `1 XOR 1 = 0`.

5. **Mistake → Assuming XOR appearing in a binary means obfuscation.**
   **Why it matters →** XOR is everywhere in normal software and cryptographic algorithms.
   **Instead →** Look for repeated keys, loops, data transformations, and resulting plaintext.

6. **Mistake → Using a decimal mask when the binary structure is what matters.**
   **Why it matters →** Decimal hides which individual bits are being selected.
   **Instead →** Express masks in hexadecimal or binary during analysis.

# Section 8 — Move-on gate

1. **Bitmask test:** take `0xB7` and determine the result of `AND 0x0F`, `OR 0x08`, and `XOR 0x08` mentally, then verify each result in Bash.

2. **Flag-analysis test:** given `flags = 0b10101100` and `mask = 0b00000100`, use a bitwise operation to determine whether that flag is set, without looking at your notes.

3. **XOR-reversal test:** take the bytes `0x1D 0x10 0x19 0x19 0x1A`, identify the repeated XOR key `0x55`, recover the plaintext, and explain why applying the same XOR operation reverses the transformation.
