---
name: unomove-learned
description: Reuse evidence-backed lessons from prior Unomove work on replay, sensor fusion, shared-memory lifecycle, asynchronous state, external sync, releases, and operator controls. Use when a task resembles one of these failure shapes or when distilling recent repository changes.
---

# Learned engineering knowledge

Use this as an on-demand index, not an always-loaded rule dump. Read only the
reference matching the task. These are scoped lessons from inspected changes,
not proof that a current checkout or deployed system is healthy.

## Select the relevant knowledge

| Task shape | Read |
| --- | --- |
| Delayed measurements, replay, sensor calibration, fusion, planning verdicts, shared-memory recovery | [Robotics and messaging](references/robotics-messaging.md) |
| Async cache invalidation, uncertain writes, capability catalogs, scheduling exclusion, inherited locks, stream settlement, capacity | [Platform and lifecycle](references/platform-lifecycle.md) |
| Authentication recovery, external ledger writes, approval invalidation, extraction, calendar snapshots, monitor isolation, controls, demand rendering | [Applications and integrations](references/applications-integrations.md) |
| Provenance, surveyed repositories, date window, coverage limits, or adding another lesson | [September 2026 evidence ledger](references/2026-09-evidence.md) |

## Apply a lesson

1. Match the failure mechanism, not just a project or symptom name.
2. Read the current implementation and its regression guard; historical commits
   explain the lesson but are not today's runtime state.
3. Reuse the established owner and contract. Keep hardware constants, time
   budgets, deployment paths, and protocol versions project-local.
4. Use the reference's suggested check when relevant and authorized. A suggested
   check is not a claim that it ran; hardware or production checks need their
   own authority and fresh evidence.
5. Distinguish absent, invalid, stale, rejected, and successful evidence. Do not
   erase uncertainty or weaken an existing safety gate to make a symptom vanish.

## Retained lessons from the August intake

Repeated entries are consolidated here; their full history remains in git.

- After interruption, re-read the actual files, branch, pending jobs, and partial
  artifacts before continuing. Recover usable work rather than blindly restarting.
- Completion needs fresh proof of the requested result, not a completed substep:
  use [verification](../unomove-verification/SKILL.md).
- Regression guards must fail on the real fault; use
  [test-first guidance](../unomove-tdd/SKILL.md).
- Preserve others' work; stage only owned paths and use an isolated, reproducible
  build input for deployment: [isolation](../unomove-isolation/SKILL.md).
- Keep one actuator authority and verify applied telemetry:
  [engineering rules](../../engineering-rules.md#safety-critical).
- Ground deliverables in inspected source material and existing artifacts. Load
  the format-appropriate skill before drafting; clarify material unknowns, not
  every cosmetic choice. After layout edits, check the full rendered artifact
  for overflow, orphan pages/lines, dimensions, and missing assets.

## Adding knowledge

When publishing or vendoring this skill, include its `references/` directory;
a selective copy of only `SKILL.md` is incomplete. Publication and downstream
synchronization require their own scope and verification.

Record **trigger → mechanism → action → falsifying check**, with a repository,
commit and source/test path. Separate observed implementation, historical test
claims, and fresh runtime proof. Deduplicate against the curated rules first.
One incident is a scoped lesson, not automatically a universal requirement;
promote recurring patterns under the
[rule-book contribution contract](../../engineering-rules.md#how-to-add).
Never import raw sessions, secrets, personal records, or stale completion claims.
