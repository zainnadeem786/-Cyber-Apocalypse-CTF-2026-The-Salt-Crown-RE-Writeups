# The Cinder Engine — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | The Cinder Engine |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{c1nd3r_3ng1n3_unw0und}` |


## 1. Challenge Overview

`The Cinder Engine` was a reverse engineering challenge based on a stripped ARM64 ELF binary.

The story described an old machine with a lost opcode map. Only one scribe understood how the engine interpreted its instructions, and after his death nobody knew how to read the machine anymore. This was the main hint that the challenge involved a custom virtual machine.

The provided files were:

```text
.DS_Store
._.DS_Store
cinder
````

The `.DS_Store` and `._.DS_Store` files were macOS metadata files and were not relevant to the challenge.

The real challenge file was:

```text
cinder
```

Initial inspection showed that `cinder` was:

```text
ELF 64-bit ARM aarch64, stripped
```

So this was not a normal x86 Linux binary. It was an ARM64 executable containing a custom VM and an embedded firmware/cipher routine.

The goal was to understand the VM, recover the opcode map, reverse the cipher implemented inside the firmware, and provide the correct 32-byte input to reveal the flag.


## 2. Initial File Inspection

I started with basic file inspection:

```bash
file cinder
ls -lh cinder
strings -a cinder | head
readelf -h cinder
checksec --file=cinder
```

The important result was:

```text
ELF 64-bit LSB executable, ARM aarch64, stripped
```

Because the binary was stripped, there were no helpful function names.

This meant the analysis had to focus on:

```text
program structure
constant tables
VM dispatch logic
input handling
comparison routine
output generation
```

The `.DS_Store` files were ignored because they were only filesystem metadata.


## 3. Running the Binary

Since the binary was ARM64, it may not run directly on an x86_64 Kali system.

If running on an x86 machine, `qemu-user` can be used:

```bash
sudo apt install qemu-user qemu-user-static
qemu-aarch64 ./cinder
```

If the binary requires ARM64 libraries, it can be run with an ARM64 sysroot:

```bash
qemu-aarch64 -L /usr/aarch64-linux-gnu ./cinder
```

The binary expected input from stdin.

A normal printable input did not reveal the flag, which suggested that the expected input was not a normal password.

Later analysis confirmed that the correct input was 32 raw bytes.


## 4. Identifying the Custom VM

The challenge story strongly hinted at a custom instruction set:

```text
opcode map
engine
firmware
machine still runs
work out how it moves
```

During reverse engineering, the binary showed a VM-like structure:

```text
instruction pointer
register/state array
firmware bytecode
opcode dispatch
memory load/store
comparison flag
conditional jumps
```

The VM did not use normal CPU instructions for the actual challenge logic. Instead, the ARM64 binary acted as an interpreter for its own custom bytecode.

The high-level structure was:

```c
while (running) {
    opcode = firmware[ip++];

    switch (opcode) {
        case ...:
            execute_vm_instruction();
            break;
    }
}
```

Because the binary was stripped, the opcode map had to be recovered manually by observing how each opcode affected VM state.


## 5. Recovered Opcode Map

After analyzing the dispatch logic and VM state changes, the important opcode map was reconstructed as:

```text
0x3a = load immediate
0xe0 = read input byte
0xe1 = output byte

0xc4 = memory load
0xc5 = memory store
0xc6 = load byte from firmware table

0x52 = xor
0x7c = add
0x7d = sub
0x80 = add immediate

0x90 = compare
0xa0 = jump
0xa1 = jump if zero / equal flag
0xa2 = jump if not zero / not equal flag
```

This confirmed that the embedded firmware was implementing real logic through the VM.

The most important VM instructions were:

```text
0xe0 -> read user input
0xc6 -> read S-box / table data
0x52 -> XOR operation
0x90 -> compare against target
0xe1 -> output result
```

These were the key to understanding the cipher and recovering the flag.


## 6. Input Length

The VM read exactly 32 bytes of input.

This was important because the expected input was not a normal printable string.

The correct input was:

```text
83 ef aa ac 9b 9a 46 fb
fe 0c b6 65 e0 82 12 70
80 78 23 20 d5 1c 1d 13
4f 5b cd 15 7a ba f5 0f
```

As one hex string:

```text
83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f
```

Because this input contains non-printable bytes, it cannot be typed normally into the terminal.

It must be piped as raw bytes.


## 7. Understanding the Firmware Cipher

After the opcode map was recovered, the VM firmware could be understood at a higher level.

The firmware implemented a custom 32-byte transformation:

```text
input[32]
    ↓
XOR with initial key
    ↓
S-box substitution
    ↓
8 rounds of transformation
    ↓
linear XOR mixing
    ↓
round key addition
    ↓
S-box substitution
    ↓
compare with target table
```

If the transformed input matched the target table, the VM then produced output bytes.

The output was not stored directly as the flag. Instead, the VM used the original input and an output mask:

```text
flag = original_input XOR output_mask
```

This design meant that the flag was not visible through `strings`.

The binary contained the logic and tables needed to verify the input, but the correct flag was only produced after the right 32-byte value was supplied.


## 8. High-Level Cipher Flow

The reconstructed high-level logic was:

```python
def check(inp):
    state = list(inp)

    # Initial whitening
    for i in range(32):
        state[i] ^= key0[i]

    # First substitution layer
    for i in range(32):
        state[i] = sbox[state[i]]

    # Round function
    for round_no in range(8):
        state = linear_xor_mix(state)

        for i in range(32):
            state[i] ^= round_keys[round_no][i]

        for i in range(32):
            state[i] = sbox[state[i]]

    return state == target
```

If the check passed, the VM produced the flag:

```python
flag = bytes(input[i] ^ output_mask[i] for i in range(32))
```

So the solving strategy was:

```text
1. Recover VM opcode map
2. Lift VM firmware into Python-like logic
3. Extract S-box, round keys, target table, and output mask
4. Reverse or solve the cipher
5. Recover the 32-byte input
6. Feed raw input into the binary
7. Read the flag
```


## 9. Why Strings Did Not Work

A direct strings check was not enough:

```bash
strings -a cinder | grep HTB
```

The flag was not stored directly inside the binary.

This was intentional.

The VM only generated the flag after the correct input passed the firmware cipher.

So the challenge was not about finding a hidden string. It was about understanding the custom VM and the cipher implemented by its firmware.


## 10. Recovering the Correct Input

After reversing the firmware cipher, the required 32-byte input was recovered:

```text
83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f
```

This value is not the flag itself. It is the raw input needed to satisfy the Cinder Engine.

The flag is produced only when the VM accepts this input and applies the output mask.


## 11. Running the Final Payload

The correct input contains non-printable bytes, so I used Python to pipe it into the binary:

```bash
python3 - <<'PY' | ./cinder
import sys

sys.stdout.buffer.write(bytes.fromhex(
    "83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f"
))
PY
```

Expected output:

```text
HTB{c1nd3r_3ng1n3_unw0und}
```

If running through QEMU on x86_64:

```bash
python3 - <<'PY' | qemu-aarch64 ./cinder
import sys

sys.stdout.buffer.write(bytes.fromhex(
    "83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f"
))
PY
```


## 12. Final Solver

A minimal final solver for validation:

```python
import subprocess

payload = bytes.fromhex(
    "83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f"
)

print("[+] payload length:", len(payload))
print("[+] payload hex:", payload.hex())

p = subprocess.run(
    ["./cinder"],
    input=payload,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE
)

print(p.stdout.decode(errors="replace"))
```

Expected output:

```text
[+] payload length: 32
[+] payload hex: 83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f
HTB{c1nd3r_3ng1n3_unw0und}
```


## 13. Key Observations

The important observations were:

```text
1. The real challenge file was `cinder`.
2. `.DS_Store` and `._.DS_Store` were irrelevant macOS metadata files.
3. `cinder` was a stripped ARM64 ELF binary.
4. The binary implemented a custom virtual machine.
5. The VM executed embedded firmware bytecode.
6. The opcode map had to be reconstructed manually.
7. The firmware implemented a 32-byte cipher.
8. The input was not printable.
9. The correct input caused the VM to generate the flag.
10. The flag was not visible through strings.
```


## 14. Solve Chain

The full solve chain was:

```text
Inspect provided folder
        ↓
Ignore .DS_Store metadata files
        ↓
Identify `cinder` as stripped ARM64 ELF
        ↓
Run static analysis
        ↓
Find VM dispatch loop
        ↓
Recover opcode map
        ↓
Lift firmware bytecode into readable logic
        ↓
Extract S-box, keys, target, and output mask
        ↓
Reverse the 32-byte cipher
        ↓
Recover raw input payload
        ↓
Pipe payload into binary
        ↓
VM validates input
        ↓
VM generates final flag
```

Recovered input:

```text
83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f
```

Final flag:

```text
HTB{c1nd3r_3ng1n3_unw0und}
```


## 15. Lessons Learned

`The Cinder Engine` was a strong reverse engineering challenge because it was not solved by simple string extraction or direct decompilation.

The binary was only the outer layer. The actual challenge logic lived inside a custom VM firmware.

The important lesson was:

```text
When a binary contains a custom VM, focus on recovering the instruction set before trying to understand the protected logic.
```

Another important point was that the flag was generated dynamically:

```text
correct input XOR output mask = flag
```

So the flag could only be recovered after the VM firmware was understood and the correct input was found.


## 16. Final Summary

`The Cinder Engine` was a stripped ARM64 ELF reverse engineering challenge built around a custom virtual machine.

By analyzing the VM dispatch loop, recovering the opcode map, lifting the firmware logic, and reversing the embedded 32-byte cipher, I recovered the required raw input:

```text
83efaaac9b9a46fbfe0cb665e082127080782320d51c1d134f5bcd157abaf50f
```

Feeding that input into the binary produced the final flag:

```text
HTB{c1nd3r_3ng1n3_unw0und}
```
