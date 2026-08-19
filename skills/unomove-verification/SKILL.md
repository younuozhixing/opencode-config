---
name: unomove-verification
description: Use before claiming anything is done, fixed, passing, deployed, published, or healthy — every status claim needs fresh evidence from a command you just ran.
---

# Evidence before claims

Load this the moment you are about to say done, fixed, green, shipped, or healthy.

## The rule

```
NO STATUS CLAIM WITHOUT FRESH EVIDENCE FROM THIS SESSION
```

If you have not run the command in this message, you cannot report its result.
A remembered pass, an earlier run, or "it should now work" is not evidence.

## The gate

Before any claim:

1. **Name the command that would prove it.** If you cannot name one, you cannot
   make the claim — say what is unverified instead.
2. **Run it fresh and whole.** Not a subset, not `-x`, not a cached log.
3. **Read the output and the exit code.** Count the failures.
4. **Compare to the claim.** If they disagree, report what actually happened.
5. **State the claim WITH the evidence**, or state the gap.

## What each claim actually requires

| Claim | Evidence | NOT sufficient |
|---|---|---|
| "tests pass" | full run, exit code, failure count | the file you touched passing |
| "the bug is fixed" | the failing case now passing, and the test failing without the fix | the code looks right |
| "deployed" | the target serving the new build id / bytes | the deploy script exited 0 |
| "published" | the artifact fetched from where clients read it | the upload step ran |
| "the box is healthy" | every unit on it, not the one you touched | your own service is up |
| "it is faster" | a measurement, both sides | reasoning about the change |

## Unknown is not OK

If you could not look — offline, no permission, a tool missing — say **UNKNOWN**.
Reporting an all-clear about something you never read is the failure this skill
exists to prevent, and it is worse than silence because it stops the next person
from checking.

## Why this exists (real failures)

- A fleet ship printed `datacenter: ok` for hours while another unit on that same
  box was crash-looping: 444 restarts, no listener. The deploy verified the unit it
  installed and nothing else, so it was truthful about its service and wrong about
  the machine.
- The same summary reported `apps: ok` for a build that was never published — no
  token on the box — so the installers reached nobody while the line read green.
- A staleness check printed "every published platform is up to date" after the
  manifest fetch had failed. It was an all-clear about a file that was never read.

## Then

Run `skills/unomove-review/SKILL.md` before the final answer.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
