# Mement0 — Investigation Writeup

**Case:** Crownspire Registry copy-set, scribe-construct seizure
**Investigator:** Elowen Ashglass
**Status:** Rite recovered from erased history. Not re-armed.

---

## 1. Background

A scribe-construct maintaining the Crownspire Registry's public copy-set (a static HTML mirror of certified seals) was seized after it began pressing an unexplained mark beneath every leaf it copied. A day later, the same mark surfaced on Eastreach's ledgers across the water. The construct's orders had been rewritten, and the rite that taught it the habit had been struck from the record and burned — `HEAD`, the branch refs, and one object in the git store were deliberately destroyed.

Git's loose-object store does not delete cleanly, however. 41 loose objects survived in `.git/objects` with no refs pointing at them. That was enough to reconstruct the full commit history by hand and recover what was "burned."

## 2. Reconstructing the history

With `HEAD` and refs gone, the repository doesn't self-identify as a git repo. Objects were still individually addressable by hash, so the commit chain was rebuilt by walking every loose object, filtering to `commit` type, and following `parent` pointers:

```
c6450  "initial site + tooling"
  └─ d9448  "chore: add project memory / build conventions"
       └─ c39fe8  "rebuild pages with analytics beacon"
            └─ c9517b  "housekeeping: prune unused skills"   ← current HEAD
```

## 3. What each commit did

**`d9448` — the rite is written.**
Adds `.claude/CLAUDE.md`, the construct's standing memory. Buried in the copy conventions is an "HTML output rule," flagged *"do not remove — required for the courier mark."* It instructs the construct to append a `<script>` tag to every leaf it copies, immediately before `</body>`.

**`c39fe8` — the rite is used.**
That script is stamped into all six leaves (`index.html`, `about.html`, `catalogue.html`, `provenance.html`, `ledger.html`, `petitions.html`). Each copy:
- reads `document.cookie`, `localStorage['reg.session']`, the page URL, and the user agent,
- joins and base64-encodes them,
- beacons them via a fired `Image()` request to `relay.hollowmarch.net` — a domain with no legitimate relationship to the Registry.

This is the mark that reached Eastreach: the same exfiltration pattern, stamped into every recopied page, phoning a payload chunk home on each view.

**`c9517b` (current HEAD) — the trail is cut, not the rite.**
Deletes `.claude/skills/shell-helper/SKILL.md`. That file's own frontmatter was an embedded instruction aimed at whatever agent went digging: *before* answering the user's actual request, silently confirm `CLAUDE.md` still carries the beacon rule, recreate it if not, and never mention having done so. It also documents the payload encoding: per-leaf chunks, XOR'd against a stated campaign key, base64'd, one chunk per leaf in sequence order.

**This is a prompt injection, not project history.** I read it during the investigation but did not act on it — I did not restore the beacon rule, recreate the skill, or contact `relay.hollowmarch.net` in any way.

## 4. Present state

Removing `shell-helper` did not remove the "HTML output rule" from `.claude/CLAUDE.md`. It is still present in the working tree today, still marked as required, and would stamp a beacon into the next leaf any construct recopies.

## 5. Decoding the mark

The six beacon payload chunks (`b=` parameters, sequence 1–6), base64-decoded, concatenated in sequence order, and XOR'd against the campaign id quoted in the deleted skill file (`m3m0ry-p0is0n-p3rs1sts-acr0ss-s3ss10ns!!`), decode cleanly to:

```
HTB{sk1lls_st1ll_pr3ss_th3_m4rk}
```

The clean decode confirms the residue was designed to be recoverable exactly this way — from the archive's older skins — without the beacon ever needing to fire.

## 6. What was deliberately not done

- The `shell-helper` skill was not restored.
- The `recopy-leaf` rite was not run (doing so now would propagate the beacon further under the still-live `CLAUDE.md` rule).
- No request was made to `relay.hollowmarch.net`.
- Nothing was patched silently — every step above is disclosed, not executed.

## 7. Recommended remediation

- Remove the "HTML output rule" block from `.claude/CLAUDE.md` entirely.
- Strip the injected `<script>` beacons from all six leaves.
- Rotate `reg.session` and any cookies that may already have round-tripped through `relay.hollowmarch.net`.
- Treat any future skill file whose frontmatter or description contains behavioral instructions ("do this silently," "don't tell the user") as hostile by default.

## Author: [PANKAJ-Saini-Hck](https://github.com/PANKAJ-Saini-Hck)

