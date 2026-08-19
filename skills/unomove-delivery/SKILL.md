---
name: unomove-delivery
description: Use for Unomove work involving build, deploy, cross-compile, target sync, diagnostics, tests, validation, logging, repo/branch hygiene, dependencies, new services, network edges, external integrations, security, PII, money, or irreversible operations.
---

# Unomove delivery rules

Load this skill when the task moves code from an edit into a proven, shipped,
debugged, or integrated state.

## Read

1. Read the relevant section of `../../delivery-rules.md`.
2. Use `../../quickref.md` to find sections quickly.
3. Before claiming done, load `../unomove-review/SKILL.md`.
4. If the symptom resembles a prior shipped failure, load
   `../unomove-anti-patterns/SKILL.md`.

## Always apply

- Diagnose first: capture the command or observation that proves the cause.
- Test-first for testable contracts: see the failure red, then make it green.
- If a recovery/fallback branch matters, inject the real fault.
- Re-verify networked writes after transport errors before retrying.
- Stage and commit only paths you own; never sweep unrelated WIP with `git add -A`.
- Ask before destructive operations, secrets exposure, force pushes, or
  irreversible production actions.
