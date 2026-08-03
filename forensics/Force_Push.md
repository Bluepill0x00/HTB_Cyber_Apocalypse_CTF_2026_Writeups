# Writeup — Force Push (Git Forensics)

## Challenge Overview
We're given a leaked backup of the `crownspire-deploy` repository. At some point, a production secret (the "warden's key") was accidentally committed and then removed via a history rewrite / force-push, leaving the visible history looking clean. The goal is to recover the deleted secret.

## Step 1: Check the Visible History

```bash
git log --all --oneline
```

The commit history appears completely clean — commits about adding a `.gitignore` entry for `.env`/`*.creds`, loading credentials from environment variables, and so on. There's no visible trace of any leaked file in the current log.

## Step 2: Check the Reflog

```bash
git reflog show --all
```

Empty. This backup copy didn't carry over any reflog entries, so the usual "undo history" trick (recovering via `HEAD@{n}` references) isn't available here.

## Step 3: Scan for Dangling Objects

This is the key step. A force-push or history rewrite only detaches a commit from its branch pointer — it doesn't actually delete the underlying objects from `.git/objects` unless someone explicitly runs `git gc --prune`. So we can search for objects that still physically exist but are no longer reachable from any branch or tag:

```bash
git fsck --full --no-reflogs --unreachable --lost-found
```

This turns up:

```
unreachable commit 3c8803d...
unreachable blob 12b1497...
unreachable tree 1fe5a4a...
```

These are exactly the leftovers a force-push produces: data still sitting in the object store, orphaned from any ref.

## Step 4: Inspect the Dangling Commit

```bash
git show 3c8803d...
```

This reveals the original commit, with the message:

```
temp: add reliquary.creds to debug 403 on manifest push (REVERT ME)
```

The full diff shows the file `reliquary.creds` being added, containing:

```
RELIQUARY_ENDPOINT=https://reliquary.crownspire.valyssar:9000
RELIQUARY_BUCKET=crownspire-reliquary-prod
AWS_ACCESS_KEY_ID=AKIACROWNSPIRE7WARD3N
AWS_SECRET_ACCESS_KEY=HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}
WARDEN_SIGNING_KEY=astrael-relic-sigil-2f9c
```

## Result

The flag is embedded directly in the leaked secret file.

**Flag:** `HTB{th3_r3l1qu4ry_n3v3r_f0rg3ts}`

## Takeaway
Removing a file in a later commit — or even rewriting history with a force-push — does **not** erase the data from a Git repository's object database. Without an explicit `git gc --prune=now` (and often a full clone/repack afterward), dangling commits and blobs remain recoverable via `git fsck --unreachable`. Rotating any credential that ever touched version control is the only reliable fix.

## Author: [ashirali84](https://github.com/ashirali84)
