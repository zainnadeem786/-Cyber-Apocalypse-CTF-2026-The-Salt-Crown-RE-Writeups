# Ringtrue — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | Ringtrue |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}` |


## 1. Challenge Overview

`Ringtrue` was a reverse engineering challenge based on a small ELF binary that validated a sequence of eight tones.

The challenge story described a shard that was trained to listen for “one voice”. The player had to find the correct eight tones that would make the shard “ring true” and reveal the sealed vow.

The provided file was:

```text
ringtrue
```

Initial inspection showed that the binary was:

```text
ELF 64-bit x86-64 PIE, not stripped
```

Unlike some reversing challenges where symbols are stripped or heavily obfuscated, this binary still contained useful symbol names. Those symbols revealed that the binary was using a small neural-network style checker.

The goal was to recover the correct eight integer tones, pass them to the binary, and reveal the final flag.


## 2. Initial File Inspection

I started with basic static inspection:

```bash
file ringtrue
chmod +x ringtrue
strings -a ringtrue
nm -C ringtrue
objdump -t ringtrue
```

The important result was that the binary was not stripped, so several useful symbols were visible.

Interesting symbols included:

```text
L0_W
L1_W
L2_W

L0_B
L1_B
L2_B

ECHO_S
VOW_CIPHER
```

These names were highly meaningful.

They suggested that the binary contained:

```text
L0_W, L1_W, L2_W  -> neural network layer weights
L0_B, L1_B, L2_B  -> neural network layer biases
ECHO_S            -> target output signature
VOW_CIPHER        -> encrypted flag data
```

This immediately changed the approach. The challenge was not a normal password comparison. It was checking the input through a small neural-network style model.


## 3. Understanding the Input

The story said:

```text
Play the right eight tones
```

This matched the binary behavior. The program expected eight integer values as input.

A normal run looked like:

```bash
./ringtrue
```

The input format was eight numbers separated by spaces.

The final recovered tones were:

```text
83 97 108 116 67 114 119 110
```

These values are interesting because they are valid ASCII codes.

Converting them to characters:

```text
83  -> S
97  -> a
108 -> l
116 -> t
67  -> C
114 -> r
119 -> w
110 -> n
```

So the correct voice was:

```text
SaltCrwn
```

This matched the event theme: `The Salt Crown`.


## 4. Neural Network Checker

The binary implemented a small multi-layer perceptron style checker.

The visible symbols made the structure clear:

```text
L0_W / L0_B -> first layer weights and biases
L1_W / L1_B -> second layer weights and biases
L2_W / L2_B -> third layer weights and biases
```

The input was an array of eight tones:

```python
tones = [83, 97, 108, 116, 67, 114, 119, 110]
```

The model processed those tones through three layers and compared the final output against a target signature:

```text
ECHO_S
```

At a high level, the checker worked like this:

```python
x = tones

x = layer0(x, L0_W, L0_B)
x = activation(x)

x = layer1(x, L1_W, L1_B)
x = activation(x)

x = layer2(x, L2_W, L2_B)

if x == ECHO_S:
    decrypt_flag()
else:
    fail()
```

The exact implementation used small integer operations, which made the checker deterministic and reversible enough to analyze.


## 5. Important Binary Symbols

The symbol names were the main clue.

### Weight Tables

```text
L0_W
L1_W
L2_W
```

These represented the model weights.

### Bias Tables

```text
L0_B
L1_B
L2_B
```

These represented the layer biases.

### Target Signature

```text
ECHO_S
```

This was the expected output of the network for the correct eight tones.

### Encrypted Flag

```text
VOW_CIPHER
```

This contained the encrypted flag. The binary only decrypted it after the correct tones passed the checker.

The presence of `VOW_CIPHER` confirmed that the flag was not meant to be recovered by simple string extraction. It had to be decrypted through the correct input path.


## 6. Recovering the Correct Tones

Since the program expected exactly eight tones, each tone could be treated as an integer input.

The final recovered tone sequence was:

```text
83 97 108 116 67 114 119 110
```

As bytes:

```python
bytes([83, 97, 108, 116, 67, 114, 119, 110])
```

Output:

```text
SaltCrwn
```

This value was the correct voice that caused the neural-network checker to match `ECHO_S`.

The challenge wording also supported this result:

```text
the shard still holds a bit of the dragon Astrael's power
Eastreach built a small device around it and taught it to listen for one voice
Play the right eight tones
```

The “one voice” was the eight-byte phrase:

```text
SaltCrwn
```


## 7. Flag Decryption Logic

After the network accepted the tones, the binary used the low bytes of the tone values to derive a stream/key material.

That derived material was then used to decrypt `VOW_CIPHER`.

High-level flow:

```text
input tones
    ↓
3-layer integer MLP checker
    ↓
output compared with ECHO_S
    ↓
if matched, derive decrypt stream from tones
    ↓
decrypt VOW_CIPHER
    ↓
print flag
```

This meant the flag was protected behind two layers:

```text
1. Neural-network style input validation
2. Encrypted flag buffer
```

The binary did not contain the flag directly as plain text.

A direct strings check would not reveal the flag:

```bash
strings -a ringtrue | grep HTB
```

The correct input was required to decrypt it.


## 8. Running the Binary

After recovering the tones, I supplied them to the binary:

```bash
chmod +x ringtrue
printf '83 97 108 116 67 114 119 110\n' | ./ringtrue
```

The binary accepted the tones and revealed the flag:

```text
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```

The tone values can also be shown as ASCII:

```bash
python3 - <<'PY'
tones = [83, 97, 108, 116, 67, 114, 119, 110]
print(bytes(tones).decode())
PY
```

Output:

```text
SaltCrwn
```


## 9. Solver Script

A simple final solver/runner:

```python
import subprocess

tones = [83, 97, 108, 116, 67, 114, 119, 110]

print("[+] recovered tones:", tones)
print("[+] ASCII voice:", bytes(tones).decode())

payload = " ".join(map(str, tones)) + "\n"

p = subprocess.run(
    ["./ringtrue"],
    input=payload.encode(),
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

print(p.stdout.decode(errors="replace"))
```

Expected output:

```text
[+] recovered tones: [83, 97, 108, 116, 67, 114, 119, 110]
[+] ASCII voice: SaltCrwn
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```


## 10. Why This Was Different

`Ringtrue` was different from normal binary reversing because the check was not a simple string comparison.

Instead of:

```c
strcmp(input, "password")
```

the binary used a tiny model-like checker:

```text
8 tones
  ↓
layer 0 weights/biases
  ↓
layer 1 weights/biases
  ↓
layer 2 weights/biases
  ↓
target signature comparison
```

This made the challenge feel like a mix of reverse engineering and model extraction.

The useful symbol names made the intended path clearer:

```text
L0_W / L1_W / L2_W -> weights
L0_B / L1_B / L2_B -> biases
ECHO_S             -> expected signature
VOW_CIPHER         -> encrypted flag
```


## 11. Key Observations

The important observations were:

```text
1. The file was an ELF 64-bit x86-64 PIE binary.
2. The binary was not stripped.
3. Useful symbols exposed the model structure.
4. L0_W, L1_W, L2_W were layer weights.
5. L0_B, L1_B, L2_B were layer biases.
6. ECHO_S was the target model output.
7. VOW_CIPHER contained the encrypted flag.
8. The program expected eight integer tones.
9. The correct tones were ASCII values for SaltCrwn.
10. Passing the correct tones decrypted and printed the flag.
```


## 12. Solve Chain

The full solve chain was:

```text
Inspect ringtrue
        ↓
Identify ELF x86-64 PIE binary
        ↓
Notice binary is not stripped
        ↓
Extract symbols with nm/objdump
        ↓
Find L0_W, L1_W, L2_W and biases
        ↓
Identify tiny MLP checker
        ↓
Recover correct eight tones
        ↓
Convert tones to ASCII
        ↓
Get SaltCrwn
        ↓
Run binary with tones
        ↓
Decrypt VOW_CIPHER
        ↓
Recover final flag
```

Recovered tones:

```text
83 97 108 116 67 114 119 110
```

Recovered voice:

```text
SaltCrwn
```

Final flag:

```text
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```


## 13. Lessons Learned

`Ringtrue` was a good reminder that reverse engineering challenges do not always use classic password checks.

A binary can hide its validation behind unusual logic such as:

```text
integer transforms
lookup tables
custom ciphers
virtual machines
small neural networks
```

In this challenge, the symbol names were extremely valuable. Since the binary was not stripped, names such as `L0_W`, `L1_W`, `L2_W`, `ECHO_S`, and `VOW_CIPHER` exposed the structure of the checker.

The main lesson was:

```text
When a binary contains weight and bias arrays, treat the validation path like a small model and recover the input that produces the target signature.
```


## 14. Final Summary

`Ringtrue` was a reverse engineering challenge built around a tiny neural-network style checker.

The binary accepted eight integer tones, passed them through a three-layer MLP-like validation routine, compared the result against `ECHO_S`, and decrypted `VOW_CIPHER` only when the correct tones were provided.

The correct tones were:

```text
83 97 108 116 67 114 119 110
```

Which decode to:

```text
SaltCrwn
```

Running the binary with those tones revealed the final flag:

```text
HTB{h3y_s1gn3t_1_4m_y0ur_k1ng}
```
