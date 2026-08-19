---
name: unomove-tdd
description: Use when writing or changing any behavior, and whenever you add a test — the test must be watched failing before it is trusted.
---

# Watch it fail first

Load this before writing implementation code, and before believing any green test.

## The rule

```
A TEST YOU NEVER SAW FAIL PROVES NOTHING
```

Green on the first run means one of two things: the behavior already worked, or
the test does not test it. Until you have seen the red, you do not know which.

## The loop

1. **RED** — write the test for the behavior you want. Run it. Watch it fail, and
   read the failure: it must fail for the RIGHT reason (the assertion), not an
   import error or a typo'd fixture.
2. **GREEN** — the smallest change that passes.
3. **PROVE** — for a bug fix or a guard, revert the fix and confirm the test goes
   red again, then restore. State that you did this. This is our mutation check,
   and it is the difference between a test and a comment.
4. **CLEAN** — remove scaffolding; keep the test's failure message useful enough
   that a stranger could act on it.

## Never

- **Never weaken, skip, delete, or narrow a test to get green.** If a test is
  wrong, fix it deliberately and say why, in the commit, with the reason it was
  wrong — never quietly.
- **Never assert what the code currently returns** because it is convenient. Assert
  what the behavior should be, then make it so.
- **Never test through a mock you also wrote to match the implementation.** That
  passes forever and catches nothing.
- **Never leave a test that cannot fail.** A guard with no teeth trains everyone to
  ignore the suite.

## Test the failure, not just the feature

The highest-value tests in this codebase encode a specific past incident: the
empty path that meant "everything", the column shift that hid a failed unit, the
manifest that swore a stale artifact was fresh. When you fix something, ask "what
input would bring this back?" and pin THAT.

## Why this exists (real failures)

- A reaper that stops processes by path prefix was handed `""`. `realpath("")` is
  the current directory, so it matched **eleven live processes** — an unset
  variable in any caller would have killed the whole checkout. The test that
  passed a bad path caught it before it shipped.
- A CSS rule to hide an element was defeated by a class that set `display`. The
  fix was global; the test was verified by REMOVING the fix and watching five
  checks go red, which is the only reason it is known to work.
- A guard was written for "unit is unhealthy". Mutating it three ways (drop the
  crash-loop state, report unknown as ok, flag any past restart) each made a
  specific test fail — so the guard is known to have teeth.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
