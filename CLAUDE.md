# Unomove AI coding guide

This file is intentionally small because Claude Code reads it on every
session. It is the always-loaded spine only. The full rule book stays in
this repo and is loaded on demand through `skills/`.

## Always-on rules

- Work in four phases: understand, plan, implement, verify.
- State assumptions. If the request has multiple plausible meanings, ask.
- Reproduce or diagnose before changing non-trivial behavior.
- Make surgical edits. Touch only files needed for the request.
- Keep code simple. No speculative flags, abstractions, or extra features.
- Do not add an environment-variable switch for behavior, config, code path,
  or device selection. Derive from hardware/role or checked-in config; ask if
  neither fits.
- Default new reusable code to Rust and put shared logic in `unolib`. Python is
  for Python-bound UI, ML/tensor glue, and true one-off scripts.
- Verify with the strongest practical signal and say what you ran. If the real
  proof needs hardware, fleet access, or a GPU you do not have, say so.
- No status claim without fresh evidence from a command you just ran. "I could
  not check" is a valid answer; an unverified all-clear is not.
- Find the cause before changing anything. A symptom that disappears without a
  named mechanism is not fixed.
- Watch every new test fail before you trust it, and never weaken a test to
  reach green.
- Commit only the paths you changed. Never discard work you did not write.
- Run the review checklist before claiming done.

## On-demand rule loading

Use the smallest relevant skill instead of reading the whole rule book:

| Task shape | Load |
|---|---|
| Interfaces, schemas, units, safety-critical paths, runtime/process behavior, config/defaults, shared logic | `skills/unomove-engineering/SKILL.md` |
| Any UI/frontend/operator surface work | `skills/unomove-ui/SKILL.md` |
| Build, deploy, diagnostics, tests, repo hygiene, network, external integrations, security/PII/money | `skills/unomove-delivery/SKILL.md` |
| Before final answer / "done" | `skills/unomove-review/SKILL.md` |
| A bug that resembles a previous shipped failure mode | `skills/unomove-anti-patterns/SKILL.md` |
| A task that resembles past work here — reuse what was learned | `skills/unomove-learned/SKILL.md` |
| About to claim done / fixed / passing / deployed / healthy | `skills/unomove-verification/SKILL.md` |
| A bug, failure, hang, or surprise — before any fix | `skills/unomove-debugging/SKILL.md` |
| Writing behavior, or adding any test | `skills/unomove-tdd/SKILL.md` |
| Anything bigger than one edit, or an ambiguous request | `skills/unomove-planning/SKILL.md` |
| Splitting work across agents, or several unrelated failures | `skills/unomove-subagents/SKILL.md` |
| Shared checkout, long task, or a broad git/file sweep | `skills/unomove-isolation/SKILL.md` |
| Building a BINARY Office file (PowerPoint .pptx / Word .docx / Excel .xlsx) — NOT HTML, not PDF via soffice | `skills/deck-doc/SKILL.md` |
| A tracker/checklist/dashboard the user will USE (one interactive .html + unoStore persist) | `skills/deliverable-html/SKILL.md` |
| Cloning/pushing/creating a repo (SSH-key origin + `~/unocoding/setup.sh`, never `gh`) | `skills/git/SKILL.md` |
| Reaching a company machine (rig/server, e.g. ECAR70) via `unoai-jump` (read-only) | `skills/machine-access/SKILL.md` |
| Importing/reusing shared code, or adding a dependency (build on `unolib`) | `skills/unolib/SKILL.md` |

For a fast index without loading full detail, scan `quickref.md`.

## Project-local notes

These files are codebase-agnostic. Project-specific commands, paths, ports,
and exceptions belong in that project's own `CLAUDE.local.md`, never here.
