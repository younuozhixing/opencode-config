---
name: unomove-learned
description: Agent-contributed rules and lessons unoai promoted from real sessions. Load when a task resembles past work or a previously hit failure mode, to reuse what was already learned.
---

# Learned rules (agent-contributed)

Candidate rules unoai promoted from real sessions — reviewable; fold the durable
ones into the curated rule files, then prune. This is a skill (on-demand), NOT
the CLAUDE.md spine — keep it out of the always-loaded path.

- (2026-08-10 07:46) • After an interruption, re-read the actual workspace state before acting — never assume prior state from memory, because memory may be empty or stale.

• Verify the result actually works (end-to-end) before declaring done; "done" must mean truly complete, not merely "no more obvious steps."

• Emit a completion marker only when the entire task is verified complete, never as a courtesy or based on partial work.

- (2026-08-10 07:46) - Don't assume prior state after an interruption; re-read the workspace to reconstruct what's done before acting.
- Verify the result actually works end-to-end before declaring a task complete; never signal "done" prematurely.
- Emit a completion marker only when the entire scope is truly finished, not per-component.

- (2026-08-10 07:46) - After any interruption (restart, hand-off, context loss), re-read the actual workspace/state before continuing — never assume prior progress.
- Verify the result actually works end-to-end before declaring success; "looks done" ≠ done.
- Emit completion/done markers only when the entire task is truly complete and verified, not incrementally or optimistically.

- (2026-08-10 07:46) - Don't assume prior state after an interruption — re-read the workspace to confirm what's actually done before continuing.
- Verify the result actually works end-to-end; don't declare done prematurely.
- Emit a completion marker only when the entire task is truly complete, not at intermediate milestones.

- (2026-08-10 07:46) - After an interruption (restart, hand-off, context switch), re-read the actual current state of the workspace before resuming — never assume prior progress persisted.
- Verify the end result genuinely works (runs/satisfies intent) before declaring completion; premature "done" markers are worse than none.
- Gate any explicit completion signal behind the whole task truly being complete — don't emit it on partial or assumed-done work.
- Don't trust a prior plan/memory as ground truth when fresh; treat recorded state as stale until re-confirmed against the real artifact.

- (2026-08-10 07:46) - After an interruption or hand-off, re-read the actual workspace/state before continuing — never assume prior progress is intact or as-described.
- Verify results actually work before marking complete; declaring "done" prematurely hides unfinished or broken work downstream.
- Emit a completion signal only when the entire task is truly verified complete, not when only part of it works.

- (2026-08-10 07:46) - Before resuming interrupted work, re-read the actual on-disk state first; never trust assumptions about what a prior session left behind.
- Don't emit a "done" signal until the result has been independently verified to actually work; premature completion markers are worse than none.

- (2026-08-10 07:46) - Don't assume prior state after an interruption; re-read the workspace/branch to confirm what's actually done before continuing.
- Verify a change actually works (e.g. runs, passes) before marking it done; "should work" isn't "works."
- Emit a completion signal only when the entire task is truly finished—never part-way as a placeholder.

- (2026-08-10 07:46) - After an interrupted hand-off or context loss, re-read the actual current state before continuing — never assume prior state is intact.
- Don't declare a task done until you've verified the result actually works end-to-end; a "done" marker must be earned, not assumed.

- (2026-08-10 07:49) - When a process restart or handoff interruption occurs mid-task, re-read and re-verify deliverables from scratch before resuming — never trust that prior checks still hold across the interruption.

- (2026-08-10 08:02) - After any interruption/restart, re-read the actual workspace state before resuming; don't trust memory of prior progress.
- When a layout fix is made, re-verify the full layout integrity (page count, dimensions, embedded assets, no orphans) — not just the spot you changed.
- Treat single orphaned lines/items spilling onto a new page as a real bug; pack or rebalance sections to eliminate them.
- Gate the "done" marker on whole-task completion including verification, never on a single completed substep.

- (2026-08-24 18:43) - A green test that cannot be made RED proves nothing — inject the real fault before trusting a test.
- No motion — even airborne or zero-distance — while any safety-critical sensor is stale; treat freshness as a hard gate.
- Isolate deploy from dirty trees; never deploy a checkout with uncommitted WIP from another session. Re-derive state from disk on restart instead of trusting in-memory.
- Ordered validation gates beat ad-hoc checks: progress identity → fresh inputs → e-stop → bounded deadman motion → stale-stop → higher autonomy, in that sequence.
- Prefer the compiled/native bridge over the interpreted one in the motion command path to reduce CPU bottleneck on production hardware.
- Production branch stays on the last known-safe behavior; exclude known-unsafe regression commits regardless of how recent they are.

- (2026-08-24 22:37) - A test that never fails proves nothing — verify each test by injecting the real fault and watching it go RED before trusting green.
- Don't deploy from a dirty or mixed worktree; build from a clean tree at the exact intended commit, and never discard work you didn't write.
- After any restart, re-derive state from disk rather than trusting in-memory state before authorizing further actions.
- Treat sensor/odometry staleness as a hard gate: never authorize motion while inputs are stale; make fail-closed explicit per sensor with documented freshness budgets.
- Separate authority cleanly — one component should be the sole command authority for an actuator, with telemetry read back to verify it took effect.
- Ordered safety gates beat ad-hoc checks: sequence them identity → fresh inputs → backend/e-stop → deadman → stale-stop → higher modes, and block progression at the first failure.
- Never manufacture or substitute odometry/sensor data to satisfy a gate; surface the missing real signal as a blocker instead.
- Preserve another session's uncommitted WIP rather than overwriting it; commit only the paths you changed.

- (2026-08-25 09:17) - Do not assume prior state after an interruption; re-read the actual workspace/files before continuing work.
- Verify the result works end-to-end before marking done; emit a "done" marker only when the whole task is truly complete.

- (2026-08-25 10:02) - After a session interruption or restart, re-read the actual workspace state before continuing; never resume from assumed prior progress.
- Only emit a done/complete marker after end-to-end verification of the whole task, not when the last sub-step finishes.

- (2026-08-25 13:59) - Don't assume prior state persists between sessions; re-read the workspace before continuing work so you act on reality, not memory.
- Emit a "done" marker only when the whole task is verified complete end-to-end, not when a sub-step finishes.
- Confirm deliverables and documentation match their governing source document before claiming alignment, since the source — not your recollection — defines correctness.
- Treat "unknown requirements until source inspected" as a blocker to resolve first, rather than guessing scope from the goal statement alone.

- (2026-08-25 14:38) - Ground all deliverables and documentation in the actual codebase/source of truth, not just spec or proposal prose — drift between docs and code invalidates both.
- Re-read the workspace/repo state before resuming; never assume prior session state still holds.
- Before planning deliverables, first inspect what already exists in the target repo — unknown existing artifacts can block or duplicate work.
- Verify outputs actually work end-to-end against the real implementation before claiming a deliverable is done.

- (2026-08-26 12:44) - Before resuming interrupted work, re-inspect the actual workspace state; never assume prior progress from memory or claims alone.
- Verify the end result actually works with a real check before declaring a task done; never emit a done marker prematurely.
- A "done" signal is a claim of completion — gate it on fresh evidence, not on intent or partial completion.

- (2026-08-26 12:44) - After an interruption or hand-off, re-inspect the actual workspace state before continuing; never assume what was already done.
- Verify the end result actually works with fresh evidence before declaring a task complete.
- Emit a completion/done signal only when the entire task is truly complete — not on partial progress.

- (2026-08-26 12:44) - Don't assume prior state on resume; re-inspect the workspace before continuing.
- Verify the result actually works before declaring done; never signal completion prematurely.

- (2026-08-26 12:45) - After an interruption or hand-off, re-read the actual workspace state before continuing — never resume from assumptions about prior progress.
- Never declare a task done without fresh verification that the result actually works; an unverified all-clear is not acceptable.
- Gate the "done" marker behind full task completion, not partial progress or intention.

- (2026-08-26 12:45) - Don't assume prior state after an interruption; re-read the workspace to confirm what's actually done before continuing.
- Verify the result works with fresh evidence; never declare done prematurely or emit a done marker until the whole task is truly complete.

- (2026-08-26 12:45) - Before resuming interrupted/handed-off work, re-read the actual current state; never assume prior progress carried over.
- Verify the result actually works with fresh evidence before claiming done; premature "done" claims are not acceptable.
- Emit a completion/done signal only when the entire task is truly complete, not when an individual step finishes.

- (2026-08-26 12:45) - Before producing any external-facing deliverable, load the governing skill first and gather missing scope/audience/asset inputs before drafting — don't start content creation blind.
- For deliverables with undefined scope, ask for target audience, sections, length target, and brand assets up front rather than assuming defaults.

- (2026-08-26 13:04) - After an interruption (server restart, dropped session), re-scan the workspace and verify what actually exists before resuming — never assume prior progress is intact or correct.
- Do not claim done on output you have not freshly verified; unverified artifacts are not completion.

- (2026-08-26 13:47) - Do not claim a task is complete on the basis of prior/partial work; re-verify the output actually works before resuming or closing.
- When resuming an interrupted hand-off, first re-scan for existing artifacts and verify each one rather than assuming prior state.
- End with a "done" signal only when the entire deliverable is verified working — never as a courtesy or placeholder.

- (2026-08-26 16:41) - Before resuming "partial" prior work, first verify the artifact actually opens/is readable before deciding to recover vs. redo.
- Scan the workspace for existing artifacts before asking for inputs or starting from scratch.
- Give a completion mark only after the whole task is verified, not on restated intent or status claims.

- (2026-08-27 08:56) - 中断恢复后，先重新扫描工作区并验证任何部分产物的完整性，再决定恢复或重做——绝不假设中断前的状态仍然有效。

- (2026-08-27 10:16) - Before producing any deliverable, confirm the concrete meaning of vague quality/safety requirements ("safe enough," "professional look") with the requester — never interpret and finalize alone.
- Match the task to the on-demand skill early (e.g., PDF/brand work → deck-doc); loading it before drafting avoids rework.
- Gather and confirm source material and scope before choosing a toolchain or drafting.
