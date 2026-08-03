# The Ashen Field — Writeup

**Category:** Crypto
**Scheme:** HFE (Hidden Field Equations) — Multivariate Public Key Cryptosystem

## Challenge Overview

We were given `source.sage` and its corresponding `output.txt`. The source implements an
**HFE (Hidden Field Equations)** based public-key encryption scheme:

- `q = 2` (base field GF(2))
- `n = 137` variables (`x1` through `x137`)
- Central map: `F(X) = X^(2q) + X^q + 1`
- Two secret affine transforms `S = (S_A, S_B)` and `T = (T_A, T_B)` mask the central map
- The final Public Key `PK` is a vector of 137 multivariate polynomials in these variables
- A random 137-bit `KEY` is encrypted under `PK`, and the flag is AES-ECB encrypted with the
  SHA-256 hash of that same `KEY`

`output.txt` contained three things:
1. `PK` — the public key polynomials (vector of 137 expressions)
2. `encrypted_key` — `KEY` encrypted under `PK` (137-bit output)
3. `enc_flag` (hex) — the AES-ECB encrypted flag

---

## The Vulnerability — a "Quadratic" HFE That Was Actually Linear

The security of standard HFE relies on the central polynomial `F(X)` having GF(q)-degree
exactly two (quadratic) — this is what turns the public key into a system of non-linear
(degree-2) equations, whose solving (the MQ problem) is believed to be NP-hard.

Here, the central map was:

```
F(X) = X^(2q) + X^q + 1
```

Since `q = 2`, the exponents work out to:

- `2q = 4 = 2^2`
- `q  = 2 = 2^1`

In other words, **every exponent is a Frobenius power of 2** (`X -> X^(2^k)`).

A key property of characteristic-2 fields is that **the Frobenius map is always linear**
over GF(2), because `(a + b)^2 = a^2 + b^2` (the cross term `2ab` vanishes mod 2).

Therefore:

```
X^4 = (X^2)^2   -> linear map
X^2             -> linear map
```

So `F(X) = X^4 + X^2 + 1` is actually just a **linear map plus a constant (an affine map)**,
not a quadratic one!

On top of that, `S` and `T` are themselves just affine transforms (matrix * x + constant
vector). Composing affine maps with affine maps always gives another affine map. This means:

> **The entire public key `PK` reduces to a single affine (linear + constant) function over
> GF(2)^137 — there is no genuine "multivariate quadratic" hardness left at all.**

We confirmed this directly by inspecting `output.txt`: across the full ~155KB PK expression,
there wasn't a single multiplication (`*`) term. Every component was just an XOR-sum of `xi^4`
and `xi^2` terms (which, on Boolean values `{0,1}`, are simply equal to `xi` itself).

This is the intended "aha" moment of the challenge — the flag itself hints at it
(`e1th3r_gr0bn3r_0r_v4r13ty` — whether you attack it with Gröbner basis techniques or reason
about it from field theory, you land on the same conclusion: the scheme collapses to
something linear).

---

## Exploitation Steps

### 1. Parsed the Public Key into a Linear System

Each component of the `PK` vector is a Boolean polynomial like:

```
x1^4 + x2^4 + x4^4 + ... + x1^2 + x2^2 + ...
```

Since over GF(2), `xi^k = xi` for any `k >= 1` when `xi ∈ {0,1}`, each term just refers to a
variable index. If a variable appears an **odd** number of times within a component (even
across different exponents), its net contribution is `1` (XOR parity); if it appears an
**even** number of times, the contributions cancel out to `0`.

From this we built a 137×137 matrix `A` over GF(2) and a constant vector `b`, such that:

```
PK(x) = A · x + b
```

### 2. Solved the Linear System (GF(2) Gaussian Elimination)

```
encrypted_key = A · x + b   (mod 2)
=>  x = A^{-1} · (encrypted_key XOR b)
```

Running Gaussian elimination revealed:

```
rank(A) = 135   (out of 137)
```

The system was consistent, but had a **2-dimensional kernel** — meaning 4 possible solutions
(`2^2` combinations of the free variables).

### 3. Picking the Correct Candidate

For each candidate solution, we reconstructed a `KEY` integer (checking `KEY.nbits() == 137`,
which the challenge's own key-generation loop enforces), computed its SHA-256 hash as the AES
key, and attempted to decrypt `enc_flag`. Only **one** candidate produced valid PKCS#7
padding — that was the correct `KEY`.

### 4. Decrypted the Flag

```python
AES_KEY = hashlib.sha256(str(KEY).encode()).digest()
cipher  = AES.new(AES_KEY, AES.MODE_ECB)
flag    = unpad(cipher.decrypt(enc_flag), 16)
```

---

## Solve Script (Summary)

```python
import re, pickle, hashlib
from itertools import product
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

N = 137

# --- 1. Parse output.txt ---
lines = open("output.txt").read().splitlines()
pk_line   = lines[0].strip()[1:-1]          # strip outer ( )
components = pk_line.split(", ")            # 137 polynomial expressions
enc_key   = eval(lines[1])
enc_flag  = bytes.fromhex(lines[2].strip())

# --- 2. Build linear system A x + b over GF(2) ---
A, b = [], []
for comp in components:
    row, const = [0]*N, 0
    for term in comp.split(" + "):
        term = term.strip()
        if term == "1":
            const ^= 1
            continue
        idx = int(re.match(r"^x(\d+)", term).group(1)) - 1
        row[idx] ^= 1
    A.append(row); b.append(const)

target = [enc_key[i] ^ b[i] for i in range(N)]

# --- 3. Gaussian elimination mod 2 (bit-packed rows) ---
rows = [ (sum(A[i][j] << j for j in range(N)) | (target[i] << N)) for i in range(N) ]
pivot_cols, r = [], 0
for col in range(N):
    piv = next((i for i in range(r, N) if (rows[i] >> col) & 1), None)
    if piv is None: continue
    rows[r], rows[piv] = rows[piv], rows[r]
    for i in range(N):
        if i != r and (rows[i] >> col) & 1:
            rows[i] ^= rows[r]
    pivot_cols.append(col); r += 1

free_cols = [c for c in range(N) if c not in pivot_cols]   # kernel dimension

# --- 4. Try all free-variable combinations, find valid AES padding ---
def build(free_vals):
    x = [0]*N
    for fc, v in zip(free_cols, free_vals): x[fc] = v
    for idx, col in enumerate(pivot_cols):
        val = (rows[idx] >> N) & 1
        for fc, v in zip(free_cols, free_vals):
            if v and (rows[idx] >> fc) & 1: val ^= 1
        x[col] = val
    return x

for free_vals in product([0, 1], repeat=len(free_cols)):
    x = build(free_vals)
    KEY = sum(x[i] << i for i in range(N))          # x1 = LSB ... x137 = MSB
    if KEY.bit_length() != N:
        continue
    aes_key = hashlib.sha256(str(KEY).encode()).digest()
    pt = AES.new(aes_key, AES.MODE_ECB).decrypt(enc_flag)
    try:
        print(unpad(pt, 16))
    except ValueError:
        pass
```

---

## Flag

```
HTB{e1th3r_gr0bn3r_0r_v4r13ty___1t_st1ll_w0rks!th4nks_f4l4y_f0r_y0ur_4tt4ck_0n_HFE}
```

---

## Takeaway

The security of multivariate cryptosystems (HFE, UOV, Rainbow, etc.) fundamentally depends on
the central map `F` being **strictly quadratic** over the base field `GF(q)`. If the central
map is accidentally chosen so that all of its exponents are themselves powers of `q` (as here,
with `2q` and `q`), then the additivity property of characteristic-`q` fields (Frobenius
linearity) causes the entire system to **degenerate into something linear** — at which point
simple Gaussian elimination alone is enough to recover the plaintext, without ever needing the
private key.

## Author: [PANKAJ-Saini-Hck](https://github.com/PANKAJ-Saini-Hck)
