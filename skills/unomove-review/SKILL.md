---
name: unomove-review
description: Use before saying a Unomove task is done, especially after code changes, docs changes, build/deploy work, UI work, or bug fixes.
---

# Unomove review checklist

Load this skill before the final response for any non-trivial change.

## Read

1. Read the relevant checklist section in `../../review-checklist.md`.
2. If a checklist item points to a rule you have not loaded, read that rule's
   source section before finalizing.

## Always apply

- Confirm the diff is scoped to the request.
- Remove dead imports, variables, helpers, and temporary files introduced by
  your change.
- State what you ran and what each command proved.
- If the strongest proof was unavailable, name the missing proof explicitly.
