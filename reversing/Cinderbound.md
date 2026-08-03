# Writeup — Cinderbound (.mpy Reversing)

## Challenge Overview
We're given a compiled MicroPython file, `cinderbound.mpy`. The task is to decompile it and recover the hidden logic that determines the correct input string.

## Step 1: Disassemble the Bytecode

MicroPython's own tooling can disassemble compiled `.mpy` files. Clone the MicroPython source and use `mpy-tool.py`:

```bash
git clone https://github.com/micropython/micropython
python3 tools/mpy-tool.py -d cinderbound.mpy
```

This produces a raw bytecode disassembly. Inside it is a function `judge(syllable)`, along with a 16-byte constant tuple embedded in the code:

```
(57, 129, 154, 31, 199, 192, 73, 243, 43, 176, 255, 173, 54, 203, 67, 15)
```

## Step 2: Understanding the Logic

Tracing through the bytecode reveals that `judge()` implements a simple stream-cipher-style check on the input string `s`:

- `prev` starts at `90`
- For each character index `i`:
  ```
  c[i] = (ord(s[i]) ^ prev) ^ ((i * 13) & 255)
  prev = (prev + ord(s[i])) & 255
  ```
- If the resulting list `c` matches the embedded 16-byte constant, `judge()` returns `True`.

In short: the correct input, once run through this transformation, must exactly reproduce the 16-byte key.

## Step 3: Reversing the Cipher

Since the transformation is invertible, we can solve for each original character:

```
ord(s[i]) = prev ^ c[i] ^ ((i * 13) & 255)
```

`prev` is then updated the same way the original function does, using the *recovered* character — so this has to be done iteratively, left to right.

```python
key = (57, 129, 154, 31, 199, 192, 73, 243, 43, 176, 255, 173, 54, 203, 67, 15)
prev = 90
out = []
for i, c in enumerate(key):
    mask = (i * 13) & 255
    ch = prev ^ c ^ mask
    out.append(ch)
    prev = (prev + ch) & 255

print(bytes(out).decode())
```

## Result

Running the script recovers the input string:

```
c1nd3rbound_v0w5
```

This is the exact "syllable" that makes `judge()` return `True` — the core content of the flag.

**Flag:** `HTB{c1nd3rbound_v0w5}`

## Takeaway
Compiled MicroPython bytecode is not a security boundary — it disassembles cleanly with public tooling. Custom "obfuscated" validation logic built on reversible bitwise operations (XOR combined with a running state) can always be inverted once the transformation is understood, regardless of how the constant is embedded.

## Author: [ashirali84](https://github.com/ashirali84)
