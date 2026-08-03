# Writeup — The Ledger Beneath the Hull (OSINT)

## Challenge Description
Lord Damas Marrowcairn doesn't command fleets directly — he owns them through a maze of paper. A single cargo vessel, the *Ashen Mercy*, sits at the center of a web spanning five companies: a nominee-owned shell holding the hull, a management firm running the machinery, a coordination house directing the voyages, and a commodities trader filling the holds. Each layer looks clean on its own, registered under a different name — but every thread, traced far enough, leads back to the same hand. The Outer Isles P&I Club has opened its registry for inspection: registry files, charter fixtures, company ledgers, and P&I entries are all available to those who know where to look.

**Goal:** Reconstruct the full ownership chain and prove that the hand beneath the ink belongs to Marrowcairn.

**Flag format:** `HTB{ITEM_WAS_SHARED_AMONG_QUANTITY_TARGETS}`
**Flag example:** `HTB{THE_WATER_WAS_RATIONED_AMONG_SIX_VILLAGES}`

## Solution

The challenge provides a portal at:

```
http://154.57.164.69:31864/
```

The portal presents an "Oath Submission" form with five questions, each answerable by cross-referencing the registry files, charter fixtures, and company ledgers made available on the portal.

| # | Question | Answer | Source |
|---|----------|--------|--------|
| 1 | Who is the vessel's registered owner? | Thirteenth Tide Shipping Ltd | Company Register |
| 2 | Who is the vessel's ISM manager? | Morrow Fleet Management SA | Registry / fixture documents |
| 3 | Which company is listed as the commercial operator? | Eastreach Maritime Coordination PLC | Charter fixtures |
| 4 | Which company is the time charterer? | Gilded Knife Commodities Ltd | Charter fixtures |
| 5 | Which parent company ultimately controls both the operator and charterer? | Marrowcairn Strategic Holdings PLC | Cross-referenced company ledgers |

Submitting all five correct answers reveals the final confirmation message:

> "The hull belongs to a paper company. The machinery answers to a manager. The voyages answer to Eastreach. Beneath every layer sits the same hand, clean because the ink has been divided among five names."

This message maps directly onto the flag format — "the ink has been divided among five names" corresponds to `ITEM_WAS_SHARED_AMONG_QUANTITY_TARGETS`.

## Result

**Flag:** `HTB{THE_INK_HAS_BEEN_DIVIDED_AMONG_FIVE_NAMES}`

## Takeaway
This challenge is a classic corporate-structure OSINT exercise: no single document reveals the full picture, but cross-referencing ownership records, management contracts, and charter fixtures across five nominally separate entities exposes a single beneficial owner behind all of them — a common pattern in real-world shell-company investigations (e.g. shipping, offshore trusts, and sanctions evasion cases).

## Author: [ashirali84](https://github.com/ashirali84)
