---
name: unomove-anti-patterns
description: Use when debugging a Unomove failure that resembles a previous shipped bug shape, a regression, a stale/default/config issue, a false health signal, a platform artifact mismatch, a repeated failed fix, or a UI/control-state mismatch.
---

# Unomove anti-patterns

Load this skill when the shape of the bug matters more than the specific file.

## Read

1. Scan `../../anti-patterns.md#recurring-shapes`.
2. Read any matching anti-pattern entries in `../../anti-patterns.md`.
3. Follow cross-references into `../../engineering-rules.md`,
   `../../ui-rules.md`, or `../../delivery-rules.md` as needed.

## Always apply

- Identify the bug shape before stacking fixes.
- Prefer the nearest working sibling as the reference.
- Write one hypothesis, make the smallest change that tests it, and revert
  failed fixes instead of accumulating them.
