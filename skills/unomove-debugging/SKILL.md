---
name: unomove-debugging
description: Use at the first sign of a bug, test failure, hang, or surprising behavior — before proposing or applying any fix.
---

# Root cause before fix

Load this when something is wrong and you feel the urge to try something.

## The rule

```
NO FIX BEFORE THE CAUSE IS UNDERSTOOD AND STATED
```

A change that makes the symptom disappear without a named cause is not a fix; it
is a disguise. Say the mechanism out loud — "X happens because Y" — before editing.

## The four phases

1. **Reproduce.** Get the failure to happen on demand, smallest case possible. If
   it is intermittent, say so and capture the conditions.
2. **Locate.** Follow the evidence to the exact line/decision. Read the code path,
   the logs, and the state — do not guess from the symptom's shape.
3. **Explain.** State the mechanism, and say what it predicts. A real cause
   predicts other things (which cases also break, which are safe).
4. **Fix, then prove.** Make the change, then show the failure gone AND the test
   failing without the fix (see `skills/unomove-tdd/SKILL.md`).

## Rules of engagement

- **Fix the class, not the instance.** If a trap caught you once, it will catch the
  next person: fix it where the whole class lives, once.
- **Two plausible causes means you have not finished phase 2.** Distinguish them
  with an observation before you touch code.
- **Your first suspicion is a hypothesis, not a diagnosis.** Say when you were
  wrong; a corrected diagnosis is worth more than a confident guess.
- **Do not widen the blast radius to make a symptom go away** (killing processes,
  clearing state, disabling a check) until you know what you are removing.

## Why this exists (real failures)

- A tool "ran silent for ten minutes". The cause was not the network or the model:
  the reader asked for a fixed-size chunk that never arrived, so it blocked until
  a watchdog killed it. The fix was one call — but only after the mechanism was
  named. Every earlier guess had been about the wrong layer.
- A service crash-looped with `port collision: gui and staging both resolve to
  8122`. It looked like a squatter holding a port, and "kill the port holder" was
  the obvious move — it would have fixed nothing. It died at *import*, before any
  bind, because a unit's env set the other role's knob. No port was ever held.
- An account "resurrection" was blamed twice on the merge rules before the actual
  cause was found: the test suite was pushing fixture accounts to the live fleet.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
