# 4. Words from the Past

**Category:** pwn (solved independently — flag captured, not analyzed in this write-up series)

Flag naming ("five bytes of precision") suggests a partial/last-few-bytes overwrite technique — commonly used to defeat PIE without a full leak, since the low bits of a page address are unaffected by ASLR.

## Flag

```
HTB{f1v3_byt3s_0f_pr3c1s10n_t0_rul3_th3m_4ll_7b1381df71238b4c4ba496ca74c2e02d}
```

---

## Summary

| # | Challenge | Bug Class | Status |
| :-: | :-- | :-- | :-: |
| **1** | RING the Bell | Stack overflow → ret2win | ✅ Solved |
| **2** | The Hinge Whisper | Stack leak → shellcode-on-stack | ✅ Solved |
| **3** | The Corroded Crown | Heap UAF → tcache poisoning → `__free_hook` | ✅ Solved |
| **4** | Words from the Past | Partial overwrite (not analyzed here) | ✅ Solved |

## Author: [Anurag886354](https://github.com/Anurag886354)

