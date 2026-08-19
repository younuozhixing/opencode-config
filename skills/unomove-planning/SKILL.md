---
name: unomove-planning
description: Use before implementing anything bigger than a single edit, and whenever a request has more than one plausible meaning.
---

# Plan before you type

Load this when the work is more than one obvious change.

## The rule

```
AGREE ON THE SHAPE BEFORE WRITING THE CODE
```

The expensive mistake is not a bad line; it is a correct implementation of the
wrong thing, discovered after it ships.

## First, understand

- Say the request back in one sentence, including what it does NOT cover.
- If it has two plausible readings, **ask** — one question, with the options.
- Find the authority that already owns this decision. Most requests are "change
  the one place", not "add a place". If you are about to edit several copies of
  the same fact, stop: the duplication IS the bug.

## Then, plan

A plan is good when a competent stranger with no context could execute it:

- **Ordered steps**, each one landable and verifiable on its own.
- **The proof for each step** — the command or observation that shows it worked.
- **What you will NOT do** — the scope fence. Say what you are deliberately
  leaving alone, especially shared/production surfaces.
- **The risk** — what breaks if you are wrong, and how it is undone.

Keep it short enough to read. A plan nobody reads is a diary.

## While executing

- One idea per commit; a commit message says WHY and names the failure it
  prevents. Path-limited (see `skills/unomove-isolation/SKILL.md`).
- Verify each step as you land it, not all at the end
  (`skills/unomove-verification/SKILL.md`).
- If reality contradicts the plan, say so and re-plan out loud. Do not quietly
  do something else.
- Stop and ask when the next step would touch shared production infrastructure
  that the request did not clearly authorize.

## Why this exists (real failures)

- A model-id change was applied to six copies before anyone asked why six copies
  existed. The operator's actual request was "one place to configure all models",
  and the right change collapsed the duplicates instead of editing each.
- A "publish the app" task nearly grew a downloads path on a shared nginx that
  fronts three other products. The scope fence — say it, do not do it — is what
  kept a build script from changing shared production infrastructure.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
