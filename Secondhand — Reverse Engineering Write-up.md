# Secondhand — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | Secondhand |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{WHY_S00_345Y_H0N}` |


## 1. Challenge Overview

`Secondhand` was a reverse engineering challenge based on a Linux executable.

The challenge story described a mechanism that looked clean and perfectly built, but something had been hidden inside it before inspection. Technically, this hinted that the provided binary looked normal when executed, but contained a hidden validation routine that had to be reversed.

Unlike `Cinderbound`, which was a MicroPython `.mpy` bytecode challenge, this challenge was a native Linux ELF binary.

The goal was to inspect the executable, understand how it validates input, reverse the transformation, and recover the correct input required to reveal the flag.


## 2. Initial File Inspection

I started by checking the file type and basic binary metadata:

```bash
file secondhand
chmod +x secondhand
./secondhand
````

The binary was a 64-bit Linux ELF executable.

Basic static checks were also useful:

```bash
strings -a secondhand
checksec --file=secondhand
```

The program did not reveal the flag with normal printable input. This suggested that the correct input was either transformed, encoded, or non-printable.


## 3. Understanding the Input Requirement

At first, it looked like the program expected a normal string.

However, after reversing the validation logic, it became clear that the program reads input and treats the first 8 bytes as a 64-bit little-endian integer.

That means the correct input is not a normal readable password.

The required input bytes were:

```text
0f 68 80 17 8d 2f 38 28
```

As a hex string:

```text
0f6880178d2f3828
```

This is important because trying to type this input manually in a terminal will not work properly due to non-printable bytes.


## 4. Reversing the Validation Logic

After analyzing the binary, the core validation logic was reconstructed as a 64-bit transformation.

The program takes the first 8 bytes of user input as a little-endian integer:

```c
x = input_as_u64_little_endian;
```

Then it applies the following operations:

```c
x ^= 0xa5a5a5a5deadbeef;
x = rol64(x, 17);
x *= 0x100000001b3;
x = ~x;
```

Finally, the transformed value is compared against a constant target:

```c
target = 0xb8491337c0debabe;
```

So the validation can be represented as:

```c
if (~((rol64(input ^ 0xa5a5a5a5deadbeef, 17) * 0x100000001b3)) == 0xb8491337c0debabe) {
    print_flag();
}
```

The important part was that all operations are reversible under 64-bit arithmetic:

```text
XOR        -> reversible with the same key
ROL        -> reversible using ROR
Multiplication by odd constant -> reversible modulo 2^64
Bitwise NOT -> reversible using NOT again
```


## 5. Reversing the Transform

The forward check was:

```python
x ^= 0xa5a5a5a5deadbeef
x = rol64(x, 17)
x *= 0x100000001b3
x = ~x
```

To recover the input, I reversed the operations in the opposite order.

Forward:

```text
input
  ↓ XOR key
  ↓ ROL 17
  ↓ multiply by 0x100000001b3
  ↓ bitwise NOT
  ↓ compare with target
```

Reverse:

```text
target
  ↓ bitwise NOT
  ↓ multiply by modular inverse
  ↓ ROR 17
  ↓ XOR key
  ↓ original input
```

The target was:

```text
0xb8491337c0debabe
```

The XOR key was:

```text
0xa5a5a5a5deadbeef
```

The multiplication constant was:

```text
0x100000001b3
```

Since the multiplication happens in 64-bit space, all values must be masked with:

```text
0xffffffffffffffff
```


## 6. Solver Script

I wrote a Python solver to reverse the transformation and recover the required input bytes.

```python
MASK = 0xffffffffffffffff

target = 0xb8491337c0debabe
xor_key = 0xa5a5a5a5deadbeef
mul_const = 0x100000001b3

def ror64(x, r):
    return ((x >> r) | (x << (64 - r))) & MASK

# Reverse final bitwise NOT
x = (~target) & MASK

# Reverse multiplication modulo 2^64
inv_mul = pow(mul_const, -1, 1 << 64)
x = (x * inv_mul) & MASK

# Reverse ROL 17 using ROR 17
x = ror64(x, 17)

# Reverse XOR
x ^= xor_key

# Convert recovered integer into little-endian bytes
payload = x.to_bytes(8, "little")

print("[+] recovered integer:", hex(x))
print("[+] payload hex:", payload.hex())
print("[+] payload bytes:", payload)
```

The solver recovered:

```text
payload hex: 0f6880178d2f3828
```

So the correct input bytes were:

```text
0f 68 80 17 8d 2f 38 28
```


## 7. Running the Binary with Non-Printable Input

Because the input contains non-printable bytes, I used Python to pipe the exact bytes into the program.

```bash
python3 - <<'PY' | ./secondhand
import sys
sys.stdout.buffer.write(bytes.fromhex("0f6880178d2f3828") + b"\n")
PY
```

Program output:

```text
H4D5_UP_Y0U_M4D3_IT
HTB{WHY_S00_345Y_H0N}
```

This confirmed that the reversed input was correct.


## 8. Why Normal Input Did Not Work

A normal attempt such as:

```bash
./secondhand
```

and typing printable strings would fail because the program does not compare the input as a normal string.

Instead, it interprets the first 8 bytes as a 64-bit little-endian value.

That means the correct solution is byte-level input, not a readable password.

This is why the final input had to be sent as raw bytes:

```text
0f6880178d2f3828
```


## 9. Key Observations

The main observations were:

```text
1. The challenge file was a Linux ELF executable.
2. The flag was not visible through strings.
3. The program used an 8-byte input as a 64-bit integer.
4. The validation logic applied XOR, rotate-left, multiplication, and bitwise NOT.
5. Each operation was reversible.
6. The correct input contained non-printable bytes.
7. Python was needed to pipe the exact raw bytes into the binary.
```


## 10. Solve Chain

The full solve chain was:

```text
Inspect binary
        ↓
Identify ELF executable
        ↓
Analyze validation routine
        ↓
Recover 64-bit arithmetic transform
        ↓
Reverse NOT operation
        ↓
Reverse multiplication using modular inverse
        ↓
Reverse ROL using ROR
        ↓
Reverse XOR
        ↓
Recover raw 8-byte input
        ↓
Pipe payload into binary
        ↓
Receive final flag
```

Recovered input:

```text
0f6880178d2f3828
```

Program output:

```text
H4D5_UP_Y0U_M4D3_IT
HTB{WHY_S00_345Y_H0N}
```


## 11. Final Solver

```python
MASK = 0xffffffffffffffff

target = 0xb8491337c0debabe
xor_key = 0xa5a5a5a5deadbeef
mul_const = 0x100000001b3

def ror64(x, r):
    return ((x >> r) | (x << (64 - r))) & MASK

x = (~target) & MASK

inv_mul = pow(mul_const, -1, 1 << 64)
x = (x * inv_mul) & MASK

x = ror64(x, 17)

x ^= xor_key

payload = x.to_bytes(8, "little")

print("[+] payload hex:", payload.hex())
print("[+] command:")
print(f'python3 - <<\\'PY\\' | ./secondhand\\nimport sys\\nsys.stdout.buffer.write(bytes.fromhex("{payload.hex()}") + b"\\\\n")\\nPY')
```

Expected output:

```text
[+] payload hex: 0f6880178d2f3828
```


## 12. Lessons Learned

`Secondhand` was a compact but nice reversing challenge because the correct input was not a visible string or a normal password.

The important lesson was to recognize that the program was validating a transformed 64-bit integer, not a human-readable input.

The operations looked intimidating at first, but the full transformation was reversible:

```text
XOR        -> reverse with XOR
ROL        -> reverse with ROR
multiply   -> reverse with modular inverse
NOT        -> reverse with NOT
```

The most important detail was handling everything as 64-bit arithmetic and preserving little-endian byte order when building the final payload.


## 13. Final Summary

`Secondhand` was a native ELF reverse engineering challenge where the program validated an 8-byte input using a reversible 64-bit transformation.

By reconstructing the validation routine and reversing each operation, I recovered the required raw input:

```text
0f6880178d2f3828
```

Running the binary with that payload revealed the final flag:

```text
HTB{WHY_S00_345Y_H0N}
```
