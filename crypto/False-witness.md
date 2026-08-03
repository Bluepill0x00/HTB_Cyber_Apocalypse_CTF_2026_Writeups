# False Witness — Investigation Writeup

**Case:** `crypto_false_witness/server.py`
**Investigator:** Caldrin Vowmark
**Status:** Distinguishing attack found and verified. Key fully recoverable; flag decryptable given a live instance.

---

## 1. What the service does

The server hands out one piece of physical evidence up front — an AES‑ECB encryption of the flag under a secret 256‑bit `KEY` — then lets the caller interrogate an **oracle** at up to 256 offsets, once each (answers are cached after the first ask, so a witness can't change its story on re-examination).

Before any of that, the caller is asked to supply `G`, the "hashing generator," with the only check being `1 < G < P`. That single unchecked input is the whole case.

For each bit position `i` of the key:

- If `KEY_BITS[i] == 0`, the oracle answers with a **uniformly random 256‑bit integer** — a witness with nothing to say, filling the silence with noise.
- If `KEY_BITS[i] == 1`, the oracle answers with one of two genuine values, `H(sk_i[0])` or `H(sk_i[1])`, chosen at random — a witness swearing to something real, where `H(m) = G^m mod P` and `sk_i` are honestly generated secrets.

The premise being tested: can you tell a true vow from a well-dressed lie, using only the oracle's answers?

## 2. The flaw

`H` is a discrete exponentiation in a group of order `P - 1`, and **the caller chooses the generator `G`**. The server only checks `1 < G < P` — it never checks that `G` actually generates a large subgroup, or even a subgroup of a sensible size.

Pick `G = P - 1`. Since `P` is prime, `P - 1 ≡ -1 (mod P)`, which has multiplicative order exactly 2. Then:

```
H(m) = G^m mod P = 1        if m is even
                  = P - 1    if m is odd
```

So **every genuine "1"-bit answer collapses to one of exactly two fixed values: `1` or `P - 1`.** Meanwhile a genuine "0"-bit answer is a fresh random integer drawn from `[0, 2^256)`. The chance that random draw happens to equal `1` or `P - 1` (both specific ~264-bit-scale constants) is astronomically small — negligible.

That means the two witness types stop being indistinguishable the moment the interrogator is allowed to pick the language the oracle answers in. A liar's noise and a vow's substance look identical only under an honest generator; a corrupted one turns the vow into a fingerprint.

## 3. The attack

1. Connect, receive the AES‑ECB ciphertext blob printed up front.
2. Answer the generator prompt with `G = P - 1`.
3. Query the oracle at every offset `i = 0..255` (each is answered once and cached, which is fine — a single query per bit is all that's needed).
4. For each response `v`: bit `i` of `KEY` is `1` if `v ∈ {1, P-1}`, else `0`.
5. Reassemble the 256 bits (MSB-first, matching the server's own `f"{int.from_bytes(KEY):0256b}"` construction) into the 32-byte `KEY`.
6. AES‑ECB-decrypt the earlier ciphertext with that key and strip PKCS#7 padding to recover the flag.

Verified locally against a faithful re-implementation of the server's `Oracle`/`keygen` logic (see `verify_logic.py`): the recovered bitstring matches the true key on every trial, and the reconstructed key bytes match exactly.

## 4. Solve script

```python
from pwn import remote          # or raw sockets, if pwntools isn't available
from Crypto.Cipher import AES
from Crypto.Util.Padding import unpad

P = 0xCD4A96D3B7FA7251A1BB765933FB676FCAE8C9026682E34F779122DFD66915BB
HOST, PORT = "TARGET_HOST", TARGET_PORT   # fill in from the challenge instance

io = remote(HOST, PORT)

io.recvuntil(b"Here is something for you:\n")
ct = bytes.fromhex(io.recvline().strip().decode())

io.recvuntil(b"Before we start, give me the hashing generator: ")
io.sendline(str(P - 1).encode())          # order-2 generator: G = P - 1

bits = []
for i in range(256):
    io.recvuntil(b"> ")
    io.sendline(b"1")
    io.recvuntil(b"Enter offset: ")
    io.sendline(str(i).encode())
    line = io.recvline().decode()          # "Oracle result: <v>"
    v = int(line.split(":")[1].strip())
    bits.append(1 if v in (1, P - 1) else 0)

io.recvuntil(b"> ")
io.sendline(b"2")

key_int = int("".join(map(str, bits)), 2)
key = key_int.to_bytes(32, "big")

flag = unpad(AES.new(key, AES.MODE_ECB).decrypt(ct), 16)
print(flag.decode())
```

## 5. Root cause and fix

- **Root cause:** the server treats the caller-supplied `G` as trustworthy without validating that it generates a subgroup large enough to make `H` behave like a real hash / OT-style commitment. A generator of small order turns "genuine vow" answers into a two-element set, trivially separable from noise.
- **Fix:** fix `G` server-side (never accept it from the client), or at minimum verify `G` has the full expected order (e.g. check `pow(G, (P-1)//q, P) != 1` for each small prime factor `q` of `P-1`, confirming no small-order subgroup is reachable).

## 6. What was and wasn't done

- No live host/port was included in the uploaded material, and no `flag.txt` was present — this was source-only. The attack was verified against a faithful local simulation of the server's own `Oracle` and `keygen` logic, not against a real instance.
- Nothing here required guessing or brute force: the distinguishing signal is deterministic and the recovered key matched the true key on every simulated run.

## Author: [PANKAJ-Saini-Hck](https://github.com/PANKAJ-Saini-Hck)
