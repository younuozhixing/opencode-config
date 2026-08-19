---
name: unomove-subagents
description: Use when a task splits into independent pieces, when several unrelated failures need investigating, or when a long plan would otherwise fill one agent's context.
---

# Dispatch subagents deliberately

Load this before spawning helpers, and before deciding to do everything yourself.

## The rule

```
A SUBAGENT INHERITS NOTHING. YOU CONSTRUCT EXACTLY WHAT IT NEEDS.
```

A subagent that is handed your whole history wanders. One handed a crisp contract
finishes. Their isolation is the point: it keeps your own context for coordination.

## When it pays

- **2+ independent problems** — different subsystems, unrelated failures. One
  agent per problem domain, concurrently.
- **A long plan of mostly-independent steps** — a fresh implementer per step, so
  no step inherits the last one's confusion.
- **A wide read** — surveying many files to answer one question.

## When it does NOT

- Steps that share state or must happen in order.
- Anything touching money, credentials, deletion, or a production surface — do
  that yourself, where the judgement is.
- Work whose context is the conversation itself (a design argument with the
  operator).

## The contract for each subagent

1. **The goal**, in one sentence, with the definition of done.
2. **The exact files/paths** it may read and change, and the ones it must not.
3. **The proof** it must produce — the command to run and what output means pass.
4. **The house rules that apply** — name the skill to load, do not paste the book.
5. **What to do when blocked**: report, do not improvise around the fence.

## After each subagent

- **Review before accepting.** Read the diff yourself against the contract: right
  scope, no widened blast radius, no weakened test, evidence attached.
- **Re-run the proof in your own session.** A subagent's green is a claim like any
  other (`skills/unomove-verification/SKILL.md`).
- Keep a one-line ledger per task. The ledger is the record; do not narrate.

## Why this exists (real failures)

- Two agents worked the same checkout at once. One swept the tree with a broad
  git command and destroyed the other's uncommitted work. Isolation is not
  optional when more than one agent is live — see
  `skills/unomove-isolation/SKILL.md`.
- A subagent's "engine suite green" was taken at face value while the tree had a
  contradiction in it: two commits asserting opposite defaults. Re-running the
  full suite in the coordinating session is what surfaced it.

---
Adapted from obra/superpowers (MIT, © 2025 Jesse Vincent), reshaped to Unomove's
rules and grounded in our own incidents.
