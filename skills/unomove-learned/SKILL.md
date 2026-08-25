---
name: unomove-learned
description: Agent-contributed rules and lessons unoai promoted from real sessions. Load when a task resembles past work or a previously hit failure mode, to reuse what was already learned.
---

# Learned rules (agent-contributed)

Candidate rules unoai promoted from real sessions — reviewable; fold the durable
ones into the curated rule files, then prune. This is a skill (on-demand), NOT
the CLAUDE.md spine — keep it out of the always-loaded path.

- (2026-08-10 07:40) - For PDFs with non-ASCII (e.g., CJK) text, verify with `pdffonts` that fonts are fully embedded (emb=yes, uni=yes) to guarantee no missing glyphs.
- After any infrastructure/server restart, re-verify deliverables intact rather than assuming no data loss.
- For paginated output, verify page-by-page layout integrity (e.g., no orphan lines) rather than relying on total page count alone.

- (2026-08-10 07:40) • When embedding custom fonts (especially CJK) in a PDF, always verify with `pdffonts` that embedding (`emb=yes`) and Unicode mapping (`uni=yes`) succeeded — silent missing glyphs will render as blanks or boxes.

• After any interruption (server restart, crash), re-verify every deliverable's integrity (page count, font embedding, visual layout) before declaring done — don't trust pre-interruption state.

- (2026-08-10 07:40) - When embedding fonts for CJK text, always verify with a font inspection tool (e.g., pdffonts) that fonts are fully embedded with unicode mappings — un-embedded CJK fonts silently drop or substitute characters.

- Layout verification is never "done" by one method: combine page-count (pdfinfo), font-embedding (pdffonts), and rasterized visual (pdftoppm) checks to catch different failure modes.

- A server restart or environment reload invalidates prior verification — never trust earlier "verified" state; re-run end-to-end checks after any restart before declaring completion.

- To eliminate orphans (a trailing item pushed to the next page), compress section spacing on the current page rather than padding the next page — packing earlier prevents the break.

- Treat deliverables as a unit of source + rendered artifact (e.g., HTML + PDF): keep the editable source alongside the final rendered output so future edits don't require reverse-engineering.

NONE

- (2026-08-10 07:40) - When eliminating layout orphans (e.g., a single line/section pushed to the next page), adjust section/timeline spacing to pull content together rather than accepting the orphan, and verify page-by-page integrity afterward.
- For PDF deliverables in non-Latin scripts, verify font embedding (emb=yes, uni=yes) AND render visually (e.g., pdftoppm) to catch missing-glyph boxes that metadata alone won't reveal.
- When a server restart or environment disruption interrupts a task, re-read all workspace artifacts and re-verify end-to-end before declaring completion—never emit a done marker on stale prior validation.

- (2026-08-10 07:40) • When embedding CJK fonts in a generated PDF, always verify with `pdffonts` that embedding (emb) and unicode (uni) flags are both "yes" — missing glyphs often render as blank boxes and only a font-embedding check catches it.

• To eliminate layout orphans (e.g., a section's last lines spilling to the next page), proactively adjust section/line spacing to force-fit the content block on the target page rather than accepting the break.

• After any environment restart or interruption, re-read the workspace and re-verify deliverables end-to-end before emitting a "done" marker — never assume prior-verified outputs are still intact.

NONE of the other session-specific items (company name, color accent, page split) rise to a generalizable rule.

- (2026-08-10 07:40) - When embedding fonts in a PDF, verify with `pdffonts` that all fonts are fully embedded (emb=yes, uni=yes) to avoid rendering issues across systems.
- For paginated documents, eliminate orphans by constraining content blocks (e.g., spacing, section grouping) rather than letting them flow naturally.
- Verify page-by-page layout integrity after any environment change (e.g., server restart), since artifacts can silently regress.

- (2026-08-10 07:40) - When embedding CJK fonts in a PDF, verify with `pdffonts` that every font reports `emb=yes,uni=yes` — non-embedded subset fonts cause silent missing-character (tofu) rendering.

- Lock exact deliverable dimensions (e.g., page count, page size) and verify with tooling (`pdfinfo`, `pdftoppm`) after each regeneration; "looks right" is not a substitute for measured page-by-page layout integrity.

- (2026-08-10 07:40) - Prevent layout orphans by forcing related content blocks onto the same page, adjusting spacing rather than splitting them.
- For PDFs with non-Latin text, require full font embedding and verify with `pdffonts` (emb=yes, uni=yes) to avoid missing-glyph rendering.
- After any server/process restart, re-verify deliverables end-to-end before claiming completion—prior "verified" state does not survive restart.
- Verify page-by-page layout integrity, not just overall page count, since orphan/widow and overflow issues hide at the page boundary.
- Never invent content beyond the source; formatting work is transformation, not generation.

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
