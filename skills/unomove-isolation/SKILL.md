---
name: unomove-isolation
description: Use when more than one agent or person may touch a repo at once, before any broad git operation (add -A, reset, checkout, clean, stash), and when starting a task that will run long.
---

# Do not share a working tree

Load this before any git command that acts on files you did not edit.

## The rule

```
COMMIT ONLY YOUR OWN PATHS. NEVER DISCARD WORK YOU DID NOT WRITE.
```

A working tree is shared state. Another agent, another session, or a person may
have hours of uncommitted work in it right now, and it is invisible in your
context.

## Before a broad git operation

| Instead of | Do |
|---|---|
| `git add -A` / `git commit -a` | `git add <the files you changed>` — name them |
| `git reset --hard` | `git stash push -- <your paths>`, or nothing at all |
| `git checkout .` | restore only the path you broke |
| `git clean -fd` | list first (`-n`), delete only what you created |

If a tool of yours must sweep the tree (a ship script, a formatter), it must
either be path-limited or refuse when the tree holds changes it did not make.

## Isolation for long work

- Prefer a **separate worktree per agent** so two long tasks cannot collide in one
  checkout. Detect first: if the git dir and the common git dir differ, you are
  already isolated — do not nest another.
- **One per SESSION, reused — not one per step.** Ask for it, do not invent it:

      cd "$(bin/unoai-worktrees ensure <session-id> <repo>)"   # isolate
      ...work, commit...
      bin/unoai-worktrees land <session-id> <repo> --apply      # bring it back

  Idempotent: the same session gets the same path, created once and reused after.
  That is the whole fix. This rule used to stop at "make one", so every step
  invented its own name (`-deploy`, `-verify`, `-final`, `-clean`, per commit sha)
  because none could find the previous one. Measured on one machine: 99 worktrees,
  73 GB, on a disk that was 100% full — which then failed builds and tests with
  ENOSPC and pinned a core on the OS storage scanner.
- **The build directory is already shared for you.** A turn running in a linked
  worktree gets `CARGO_TARGET_DIR=<main-repo>/.build-shared` automatically, so N
  worktrees cost ONE build dir instead of N — the 3.7 GB in each of those was
  `target/`, not source. The main checkout is untouched and your own setting always
  wins. Cargo locks that directory, so parallel builds serialise rather than
  corrupt; that wait is the price of not filling the disk. Do the same for any
  other builder with a cache.
- **Never `--detach`.** A session worktree is created on branch `unoai/<session>`,
  which is what makes the work findable (`git branch --list 'unoai/*'`) and
  landable (`git merge unoai/<session>`). 58 of the 99 worktrees found on one
  machine were on detached HEADs: their commits belonged to no branch, so nothing
  listed them and removing the worktree left them reachable only through the
  reflog. That is what "hard to merge and track" means, and it is a one-word fix
  at creation time.
- **Everything we generate lives in `<repo>/.unoai/`** — worktrees and the shared
  build output — and that directory ignores itself, so it never appears in your
  `git status` and can never be swept into a commit by `git add -A`.
- **Remove it when the task lands** — `land --apply` does it, or
  `git worktree remove <path>` (it refuses a dirty tree, which is the check you
  want). Leftovers are collected by
  `bin/unoai-worktrees reap --apply`, which only touches worktrees we created that
  are clean, fully merged, untouched for hours and with no process inside;
  `bin/unoai-worktrees list` shows what exists and why each one is kept.
- Keep the task's branch and the task's files together; land with a normal merge.
- Long-running builds must run from an **immutable copy** of their own script:
  bash reads a script incrementally by byte offset, so editing a script while it
  runs shifts every later offset and the shell resumes mid-word.

## Landing safely

- Commit early and often. Uncommitted work is the only work that can be destroyed.
- Say what you staged. If your commit contains a file you did not intend, split it.
- If you find someone else's in-flight edits in the tree, leave them alone and say
  so — do not fix, format, or revert them to make your run green.

## Why this exists (real failures)

- One agent's deploy script ran `git add -A`, committed, and then `git reset
  --hard`. It destroyed roughly an hour of another agent's uncommitted work in the
  same checkout — the reflog shows the reset, and the only reason the work was
  recoverable is that a build had embedded a copy of one file inside an artifact.
- A 15-minute deploy died 12 minutes in with `syntax error near unexpected token`
  on a line that was valid: the script had been edited while running, so the
  shell resumed at a shifted byte offset, half-way through a word. The fleet was
  left half shipped.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
