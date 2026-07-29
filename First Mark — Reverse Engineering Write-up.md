# First Mark — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | First Mark |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{cut_f0r_th3_P1NT}` |


## 1. Challenge Overview

`First Mark` was a reverse engineering challenge based on a stripped RISC-V ELF binary.

The challenge story described a stone that could judge an oath and answer with only “true” or nothing at all. It also mentioned that four of its instructions were undocumented on purpose. This was the main hint that the binary was using custom instructions or a custom execution model.

The provided file was:

```text
first-mark.elf
```

Initial inspection showed that the binary was not a normal x86 executable. It was:

```text
ELF 32-bit RISC-V, stripped
```

The important challenge hint was:

```text
The third rune is not a plain XOR. Each round it computes out = a0 ^ state ^ carry,
then updates the hidden carry as carry = old_a0 & state. The carry starts at 0,
and the state starts at 0xA5 and becomes the rune's output each round.
```

This hint was critical because one of the custom operations looked like XOR, but actually maintained hidden state and carry between rounds.

The goal was to recover the behavior of the four undocumented runes, reverse the validation routine, and recover the input that causes the binary to attest successfully.


## 2. Initial File Inspection

I started with basic file inspection:

```bash
file first-mark.elf
ls -lh first-mark.elf
strings -a first-mark.elf
readelf -h first-mark.elf
checksec --file=first-mark.elf
```

The key result was:

```text
ELF 32-bit LSB executable, UCB RISC-V, stripped
```

Because the file was stripped, there were no helpful function names.

This meant the challenge required static analysis of the program flow and instruction behavior.

The scenario text strongly hinted that the binary contained four undocumented instructions, referred to as “runes”.

Technically, these mapped to custom RISC-V instructions or custom operations that normal tooling would not fully explain.


## 3. Understanding the Challenge Hint

The challenge gave one direct clue about the third rune:

```text
The third rune is not a plain XOR.
Each round it computes:

out = a0 ^ state ^ carry

Then updates the hidden carry as:

carry = old_a0 & state

The carry starts at 0.
The state starts at 0xA5.
The state becomes the rune's output each round.
```

This was important because without the hint, the third rune could easily be mistaken for a normal XOR operation.

The real behavior was stateful:

```python
state = 0xA5
carry = 0

for a0 in values:
    out = a0 ^ state ^ carry
    carry = a0 & state
    state = out
```

So the output of each round affects the next round.

That means this was not a simple byte-by-byte independent check. The validation had chained state.


## 4. Recovered Rune Behavior

After reversing the binary, the four runes were reconstructed as follows.

### Rune 1 — Rotate Right

The first rune rotates the byte right by a table-controlled amount:

```python
rune1 = ROR8(a0, table1[i] & 7)
```

Recovered rotation table:

```python
rot = [3, 7, 1, 5, 2, 6, 4, 0, 3, 7, 1, 5, 2, 6, 4, 0]
```

### Rune 2 — GF(256) Multiplication

The second rune performs multiplication in GF(256), using the AES polynomial:

```text
0x11b
```

In implementation form, reduction uses:

```text
0x1b
```

Recovered multiplier table:

```python
mul = [3, 2, 3, 2, 5, 7, 2, 3, 5, 7, 2, 3, 5, 7, 2, 3]
```

### Rune 3 — Stateful XOR With Carry

The third rune was explained by the scenario hint.

It computes:

```python
out = a0 ^ state ^ carry
carry = old_a0 & state
state = out
```

Initial values:

```python
state = 0xA5
carry = 0
```

This rune is the most important part of the challenge because it creates dependency between bytes.

### Rune 4 — Attestation / Compare

The fourth rune compares the transformed bytes against a target table.

Recovered target bytes:

```text
11 7a 35 90 7e 88 b0 59 79 7f 56 6a 3a 10 e9 05
```

As Python bytes:

```python
target = bytes.fromhex("117a35907e88b059797f566a3a10e905")
```

If the transformed input matches this target, the program accepts the input and reveals the flag.


## 5. High-Level Validation Logic

After reconstructing the runes, the validation logic looked like this:

```python
input_bytes = user_input

for i in range(16):
    x = input_bytes[i]

    x = ror8(x, rot[i])
    x = gf_mul(x, mul[i])
    x = rune3_stateful_xor(x)

    if x != target[i]:
        fail()

success()
```

The target length was 16 bytes, so the expected input was also 16 characters.

The final recovered input was:

```text
cut_f0r_th3_P1NT
```

Length check:

```text
c  u  t  _  f  0  r  _  t  h  3  _  P  1  N  T
1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16
```


## 6. Reversing the Third Rune

Since rune 3 is stateful, I reversed it first.

Forward operation:

```python
out = a0 ^ state ^ carry
carry = old_a0 & state
state = out
```

To reverse it, if we know `out`, `state`, and `carry`, we can recover `a0`:

```python
a0 = out ^ state ^ carry
```

Then update the state and carry exactly as the forward routine would:

```python
carry = a0 & state
state = out
```

Reverse implementation:

```python
state = 0xA5
carry = 0
before_rune3 = []

for out in target:
    a0 = out ^ state ^ carry
    before_rune3.append(a0)

    carry = a0 & state
    state = out
```

This recovered the bytes that must have existed before rune 3.


## 7. Reversing Rune 2 and Rune 1

After reversing rune 3, the remaining task was to recover the original input byte before rune 1 and rune 2.

Forward:

```python
x = ror8(input_byte, rot[i])
x = gf_mul(x, mul[i])
```

Since the input length is only 16 bytes and each byte has 256 possibilities, a simple brute-force per byte is clean and reliable.

For each position:

```python
for c in range(256):
    x = ror8(c, rot[i])
    x = gf_mul(x, mul[i])

    if x == wanted:
        answer.append(c)
        break
```

This avoids mistakes with manually implementing the inverse of GF multiplication.


## 8. Final Solver

The full solver:

```python
target = bytes.fromhex("117a35907e88b059797f566a3a10e905")

rot = [3, 7, 1, 5, 2, 6, 4, 0, 3, 7, 1, 5, 2, 6, 4, 0]
mul = [3, 2, 3, 2, 5, 7, 2, 3, 5, 7, 2, 3, 5, 7, 2, 3]

def ror8(x, n):
    n &= 7
    return ((x >> n) | (x << (8 - n))) & 0xff if n else x

def gf_mul(a, b):
    res = 0

    for _ in range(8):
        if b & 1:
            res ^= a

        hi = a & 0x80
        a = (a << 1) & 0xff

        if hi:
            a ^= 0x1b

        b >>= 1

    return res

# Reverse rune 3 first
state = 0xA5
carry = 0
before_rune3 = []

for out in target:
    a0 = out ^ state ^ carry
    before_rune3.append(a0)

    carry = a0 & state
    state = out

# Reverse rune 2 and rune 1 by brute-forcing one byte at a time
answer = []

for i, wanted in enumerate(before_rune3):
    for c in range(256):
        x = ror8(c, rot[i])
        x = gf_mul(x, mul[i])

        if x == wanted:
            answer.append(c)
            break

token = bytes(answer).decode()

print("[+] recovered token:", token)
print("[+] final flag: HTB{%s}" % token)
```

Output:

```text
[+] recovered token: cut_f0r_th3_P1NT
[+] final flag: HTB{cut_f0r_th3_P1NT}
```


## 9. Running the Binary

After recovering the token, it can be supplied to the binary.

If running directly on a compatible RISC-V environment:

```bash
echo 'cut_f0r_th3_P1NT' | ./first-mark.elf
```

If running on another architecture, QEMU user emulation can be used:

```bash
echo 'cut_f0r_th3_P1NT' | qemu-riscv32 ./first-mark.elf
```

Expected result:

```text
HTB{cut_f0r_th3_P1NT}
```


## 10. Why This Was Different From Normal ELF Reversing

This challenge was different because normal decompilation alone was not enough.

The binary used custom or undocumented operations represented in the story as “runes”.

The correct approach was not only to read the binary, but to recover the behavior of the custom operations from context and hints.

The third rune was especially important because it carried state across rounds:

```text
state starts at 0xA5
carry starts at 0
out becomes the next state
carry depends on old input and old state
```

So treating it as a plain XOR would produce the wrong answer.


## 11. Key Observations

The most important observations were:

```text
1. The binary was a stripped 32-bit RISC-V ELF.
2. The challenge story hinted at four undocumented instructions.
3. The third rune was explicitly described in the prompt.
4. Rune 1 performed an 8-bit rotate-right.
5. Rune 2 performed GF(256) multiplication using AES-style reduction.
6. Rune 3 was a stateful XOR-with-carry operation.
7. Rune 4 compared the transformed bytes with a target table.
8. Reversing rune 3 first simplified the solve.
9. Rune 1 and rune 2 could be reversed by brute-forcing each byte.
10. The recovered token was `cut_f0r_th3_P1NT`.
```


## 12. Solve Chain

The full solve chain was:

```text
Inspect first-mark.elf
        ↓
Identify stripped 32-bit RISC-V ELF
        ↓
Notice story hints about undocumented runes
        ↓
Recover rune behavior from binary and prompt hint
        ↓
Extract rotation table
        ↓
Extract GF multiplier table
        ↓
Extract target table
        ↓
Reverse stateful rune 3
        ↓
Reverse rune 2 and rune 1 per byte
        ↓
Recover token
        ↓
Wrap token in HTB flag format
```

Recovered token:

```text
cut_f0r_th3_P1NT
```

Final flag:

```text
HTB{cut_f0r_th3_P1NT}
```


## 13. Lessons Learned

`First Mark` was a strong reverse engineering challenge because it combined architecture-level reversing with custom instruction reasoning.

The main lesson was that when a binary uses undocumented or custom operations, the analyst must recover their behavior from surrounding code, constants, and challenge hints.

The third rune showed why careful reading matters. The prompt explicitly warned that it was not a plain XOR. Missing that detail would make the solver fail.

Another important lesson was that not every operation needs a complex symbolic inverse. Since each byte had only 256 possible values, brute-forcing rune 1 and rune 2 per position was simple, fast, and reliable.


## 14. Final Summary

`First Mark` was a stripped 32-bit RISC-V reverse engineering challenge involving four undocumented custom operations called runes.

By recovering the rune behavior, reversing the stateful third rune, and brute-forcing the remaining byte-wise transformations, I recovered the valid token:

```text
cut_f0r_th3_P1NT
```

Final flag:

```text
HTB{cut_f0r_th3_P1NT}
```
