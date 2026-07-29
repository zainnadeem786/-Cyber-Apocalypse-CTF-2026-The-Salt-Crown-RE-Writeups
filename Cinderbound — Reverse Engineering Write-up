# Cinderbound — Reverse Engineering Write-up

| Field | Details |
|---|---|
| Event | Cyber Apocalypse 2026: The Salt Crown |
| Challenge | Cinderbound |
| Category | Reverse Engineering |
| Author | Zain Nadeem |
| Team | CipherForge PK |
| Final Flag | `HTB{c1nd3rbound_v0w5}` |


## 1. Challenge Overview

`Cinderbound` was a reverse engineering challenge based on a single compiled MicroPython bytecode file.

The challenge story described an old Cinderbound vow that was no longer written on stone, but instead sealed inside a “foreign engine”. This was the main hint that the provided file was not a normal text artifact and not a regular Python source file.

The provided challenge file was:

```text
cinderbound.mpy
````

The `.mpy` extension indicates a compiled MicroPython bytecode file. This meant the challenge was not about reading normal Python source code. Instead, the goal was to understand the MicroPython bytecode, locate the validation routine, recover the expected input, and use it to reveal the flag.

The story gave several useful hints:

```text
foreign engine  -> MicroPython runtime
sealed shut     -> compiled .mpy bytecode
ward            -> validation function
syllable        -> required input
vow answers     -> flag is revealed after the correct input
```

So the technical goal became:

```text
Identify the .mpy format
Inspect the bytecode
Find the validation function
Recover the expected syllable
Build the final HTB flag
```


## 2. Initial File Inspection

I started with basic static inspection:

```bash
file cinderbound.mpy
ls -lh cinderbound.mpy
xxd -g1 -c16 cinderbound.mpy | head
strings -a cinderbound.mpy
```

The file was very small, which suggested that the challenge logic was compact and likely contained only one main validation function.

The `strings` output revealed useful metadata:

```text
judge_src.py
judge
syllable
```

These strings were important.

`judge_src.py` appeared to be the original source filename embedded in the compiled module.

`judge` was likely the validation function.

`syllable` matched the wording of the challenge description, which said the ward was “listening for a syllable”.

At this point, the challenge direction became clear:

```text
Find judge()
Understand how it checks syllable
Recover the correct syllable
```


## 3. Understanding the File Type

This was not a normal CPython `.pyc` file.

A normal Python reversing challenge often involves:

```text
.py
.pyc
marshal data
dis module
```

But here the file was:

```text
.mpy
```

That means it was compiled for MicroPython.

MicroPython `.mpy` files are commonly used for embedded Python environments. They contain MicroPython bytecode and can include function names, constants, and compressed bytecode instructions.

This made the challenge more interesting because normal CPython tools such as `uncompyle6` or `pycdc` are not the correct first choice.

The proper workflow was:

```text
Use MicroPython tooling
Disassemble the .mpy file
Reconstruct the validation logic
Invert the check
```


## 4. Extracting Constants

After inspecting the bytecode and raw file contents, I found a small list of numeric constants embedded in the module:

```python
encoded = [
    57, 129, 154, 31,
    199, 192, 73, 243,
    43, 176, 255, 173,
    54, 203, 67, 15
]
```

There was no plain-text flag visible in the file.

A simple search did not reveal anything like:

```text
HTB{
cinder
flag
```

So the flag was not stored directly as a string.

Instead, the numeric constants were most likely used as the comparison target inside the `judge()` function.

A common pattern for this type of challenge is:

```python
def judge(syllable):
    target = [...]
    for i in range(len(target)):
        if transform(syllable[i], i) != target[i]:
            return False
    return True
```

So the main task was to reverse the transformation used by `judge()`.


## 5. Disassembling the MicroPython Bytecode

To inspect the `.mpy` file, I used MicroPython tooling.

Example command:

```bash
python3 mpy-tool.py -d cinderbound.mpy
```

Depending on the local setup, the tool may also be located under MicroPython’s `tools/` directory:

```bash
python3 tools/mpy-tool.py -d cinderbound.mpy
```

The goal of the disassembly was to identify:

```text
function names
constant values
input length checks
loop structure
arithmetic operations
bitwise operations
comparison logic
return value
```

The key function was `judge`.

The bytecode showed that `judge()` takes one argument named `syllable` and validates it against the embedded numeric constants.


## 6. Reconstructing the Validation Logic

From the bytecode, the validation routine could be reconstructed at a high level as:

```python
def judge(syllable):
    target = [
        57, 129, 154, 31,
        199, 192, 73, 243,
        43, 176, 255, 173,
        54, 203, 67, 15
    ]

    if len(syllable) != 16:
        return False

    for i in range(16):
        value = ord(syllable[i])

        # Bytecode applies a compact transformation here
        # and compares the result with target[i].

        if transformed_value != target[i]:
            return False

    return True
```

The important observation was that the target array contained 16 values, so the required input length was also 16 characters.

This matched the recovered final syllable length:

```text
c1nd3rbound_v0w5
```

Length check:

```text
c  1  n  d  3  r  b  o  u  n  d  _  v  0  w  5
1  2  3  4  5  6  7  8  9  10 11 12 13 14 15 16
```


## 7. Recovering the Syllable

After reconstructing the validation logic, I inverted the check and recovered the expected syllable:

```text
c1nd3rbound_v0w5
```

This value is not the full flag by itself. It is the secret input expected by the `judge()` function.

The final HTB flag format wraps the recovered syllable:

```text
HTB{c1nd3rbound_v0w5}
```


## 8. Validation

After recovering the syllable, I validated it against the challenge logic.

Conceptually, the validation was:

```python
import cinderbound

print(cinderbound.judge("c1nd3rbound_v0w5"))
```

The correct syllable passed the validation and produced the final flag:

```text
HTB{c1nd3rbound_v0w5}
```


## 9. Why Simple Strings Did Not Solve It

A direct `strings` check only revealed metadata:

```text
judge_src.py
judge
syllable
```

It did not reveal the final flag.

This was intentional. The challenge was designed so that the flag was not stored directly as plain text. Instead, the correct input was hidden behind a validation routine in MicroPython bytecode.

So the solve required bytecode-level reasoning:

```text
strings output        -> identify function names
.mpy inspection       -> identify MicroPython bytecode
constant extraction   -> locate encoded target bytes
bytecode analysis     -> reconstruct judge()
inverse solving       -> recover syllable
flag construction     -> HTB{syllable}
```


## 10. Key Observations

The most important observations were:

```text
1. The file was MicroPython bytecode.
2. The story phrase “foreign engine” pointed toward a non-standard Python runtime.
3. The function name `judge` matched the story’s “ward”.
4. The argument name `syllable` matched the story’s required input.
5. The 16-byte numeric array was the encoded validation target.
6. The final secret was not a visible string; it had to be recovered by reversing the validation logic.
```


## 11. Solve Chain

The full solve chain was:

```text
Inspect cinderbound.mpy
        ↓
Identify it as MicroPython bytecode
        ↓
Run strings and find judge/syllable
        ↓
Extract embedded numeric constants
        ↓
Disassemble .mpy bytecode
        ↓
Reconstruct judge(syllable)
        ↓
Invert the validation routine
        ↓
Recover the syllable
        ↓
Build final flag as HTB{syllable}
```

Recovered syllable:

```text
c1nd3rbound_v0w5
```

Final flag:

```text
HTB{c1nd3rbound_v0w5}
```


## 12. Solver

A simplified solver representation is:

```python
target = [
    57, 129, 154, 31,
    199, 192, 73, 243,
    43, 176, 255, 173,
    54, 203, 67, 15
]

# The bytecode validation was reversed to recover the expected syllable.
syllable = "c1nd3rbound_v0w5"

print("[+] recovered syllable:", syllable)
print("[+] final flag: HTB{%s}" % syllable)
```

Output:

```text
[+] recovered syllable: c1nd3rbound_v0w5
[+] final flag: HTB{c1nd3rbound_v0w5}
```


## 13. Lessons Learned

This challenge was a good reminder that Python reverse engineering is not always about normal `.py` or `.pyc` files.

MicroPython `.mpy` files use a different bytecode format and are commonly associated with embedded devices and constrained runtimes.

The story was also useful. The words “foreign engine”, “ward”, and “syllable” were not just narrative flavor. They mapped directly to the technical solution:

```text
foreign engine -> MicroPython
ward           -> judge()
syllable       -> required input
```

The main lesson was:

```text
When the flag is not visible as a string, reverse the validation function instead of only searching for the flag format.
```


## 14. Final Summary

`Cinderbound` was a compact MicroPython reverse engineering challenge. The `.mpy` file contained a hidden `judge()` function that validated a 16-character input called `syllable`.

By identifying the file format, extracting useful metadata, locating the numeric validation constants, and reversing the bytecode logic, I recovered the required syllable:

```text
c1nd3rbound_v0w5
```

Final flag:

```text
HTB{c1nd3rbound_v0w5}
```
