# September 2026 repository knowledge: evidence and boundaries

Survey date: **2026-09-23, Asia/Shanghai**. Scope: local top-level repository
checkouts under the operator's home directory, focusing on HEAD-reachable
commits since **2026-09-01 00:00:00 +0800**. No remote fetch, production access,
product test execution, deployment, push, or downstream rule synchronization
was performed. This is an engineering-knowledge update, not a fleet-health report.

## Coverage

The local inventory found **29 repositories with 1,164 commits in the window**,
including merges and shared-rule bumps. These counts are per repository, not
1,164 independently reviewed fixes. Commit subjects were surveyed; selected
implementation/test diffs were inspected to substantiate lessons. Repositories
with no commits in the window were not deeply reviewed. Dirty files were noted,
not treated as validated lessons; untracked files were not comprehensively audited.

| Repository | Snapshot HEAD | Commits | Review role |
| --- | --- | ---: | --- |
| datacenter | `2cb4e3750942` | 7 | Storage staging, exhaustion and listing |
| feishu-icloud-sync | `85b77c1a91b1` | 6 | Calendar snapshot/sync contracts |
| feishu_reimbursement | `6f0ae8e09645` | 27 | Authentication and extraction |
| passwd_manager | `f7ed7586d2fd` | 8 | Log/file inventory only; no secret-store inspection |
| remote_control_gui | `1b8d462cef1e` | 3 | Shared-rule bumps only; local WIP excluded |
| robots_monitor | `ec16702fb30b` | 3 | Shared-rule bumps only |
| slam_version | `08dae3d9c361` | 362 | Replay publication, mapping, planning verdicts |
| uno-video | `6c882998c927` | 3 | Shared-rule bumps only |
| unoai | `8b4de3b3b40c` | 159 | Transport evidence and capability catalog |
| unobox | `94a086a1ecf2` | 20 | Sensor ownership and packaging history |
| unocluster | `274a10ad2916` | 38 | Exclusion policy and process locks |
| unocoding | `c4793cde2410` | 3 | Knowledge owner, rule deduplication and rollout contract |
| unocontract | `d47773d75b7a` | 3 | Shared-rule bumps only |
| unoflow | `d5f5b9618480` | 11 | Diagnostic and deployment boundaries |
| unofsd | `233a8749f8a4` | 27 | Optional dependencies and stale package stages |
| unofsd-rs | `ff536a075dc1` | 3 | Shared-rule bumps only; local deletions excluded |
| unofsd_deploy | `d28aa89486b7` | 1 | Manifest-only change; no activation proof |
| unogui | `3cc7e1db8405` | 40 | Operator authority and compatibility |
| unogui-rs | `af93393a4114` | 3 | Shared-rule bumps only |
| unogui-web | `6176cef3c6b5` | 3 | Shared-rule bumps only |
| unolib | `6c18ec1bf5b4` | 15 | Published-library toolchain selection |
| unolocalization | `9b2c748c4764` | 10 | Transactional numerical updates and replay |
| unomobile | `8beb1de97fab` | 29 | Mobile controls and rendering |
| unomsg-rs | `be56c02f210b` | 44 | Ring contracts, loss accounting and lock lifetime |
| unopay | `ee2296dd700c` | 3 | Shared-rule bumps only |
| unostream | `1768145f3ee7` | 6 | Live attachment and typed overrun recovery |
| unotoken | `00ce32fe2372` | 231 | Cache races, settlement and per-entity acceptance |
| unotrack | `7805ce452e69` | 3 | Shared-rule bumps only |
| wheelchairsensors | `84ff1f27ae31` | 93 | Asynchronous fusion and measurement/calibration semantics |

Nine repositories contain only shared-rule bump commits in this window; those
are not nine independent incidents. Historical vendor/upstream checkouts with no
September commits and unresolved local Git pointers are outside deep coverage.
No claim is made about un-fetched remote branches, nested worktrees, all private
repositories, or changes existing only on deployed hosts.

## Where the distilled knowledge lives

- [Robotics and messaging](robotics-messaging.md): R1–R12.
- [Platform and lifecycle](platform-lifecycle.md): P1–P9.
- [Applications and integrations](applications-integrations.md): A1–A8.

Together these contain **29 scoped lessons**, alongside six consolidated August
intake themes in the skill entrypoint.

Every entry records repository + commit + source/test paths, the mechanism, and
a practical falsifying check. Historical test claims in commit messages are not
reported as fresh passes. Suggested checks can be stronger than the checked-in
test; that is a proposed validation criterion, not invented existing coverage.

To inspect evidence in any clone that contains the cited commit:

```sh
git -C /path/to/repo show <commit> -- <source-or-test-path>
git -C /path/to/repo log --since='2026-09-01 00:00:00 +0800' <snapshot-head>
```

The date filter follows Git commit time, not necessarily author time. Source
paths are relative to the named repository, deliberately not machine-specific
links. Re-resolve the current implementation and regression guard before reuse;
later commits may supersede the original mechanism.

## Concrete correction and deduplication

- Corrected the authoritative messaging paragraph that said oversized messages
  truncate without error. At the recorded transport snapshot, `max_payload()`
  bounds writes, `TooLarge` is mapped to `BufferTooSmall`, and rejection occurs
  before shared mutation. R12 identifies code and boundary regressions.
- Consolidated the old learned skill's repeated August interruption/completion
  entries into six retained themes plus links to their owning guidance. The
  original intake remains recoverable from Git. The separate legacy root
  `learned-rules.md` was not rewritten or imported as fresh evidence.
- Existing common-clock, safety freshness, source-of-truth, idempotence,
  verification, worktree and build rules were used as foundations rather than
  duplicated as dozens of new universal mandates.
- Concrete newer variants remain on-demand lessons. A one-off hardware fix is
  not automatically promoted to a company-wide numerical default.

## Known evidence gaps

- Local tracked WIP exists in several surveyed repositories, including
  `remote_control_gui`, `unobox`, `unofsd-rs`, `unofsd_deploy`, and
  `unolocalization`. The source knowledge repo also had a pre-existing planning
  skill edit. None of those edits is part of the distilled committed evidence.
- The recorded `unofsd` snapshot pins an `unobox` revision `48345bab319c`, while
  this survey's local `unobox` HEAD was `94a086a1ecf2`. The latter does not cover
  all code referenced by the former; do not infer dependency alignment.
- A deployment manifest/checksum update proves changed metadata, not that a
  process loaded the artifact. Historical stationary robot checks do not prove
  motion accuracy, loop closure, or present-day health.
- Tests using mocked upstreams, stubbed hardware or synthetic load establish
  only those bounded contracts. They do not establish real-provider quotas,
  physical safety, browser acceptance, or cross-platform/fleet rollout.
- Product-specific diagnostics, source guards and historical acceptance records
  were read where relevant; product test suites were not run for this doc-only
  task. Knowledge-file integrity is a separate check.

## Maintaining the knowledge

Keep the trigger index small and the detailed lessons conditional. Cite changed
code plus its guard, retain relevant negative cases, and distinguish observed
implementation from recommendations. Promote a recurring pattern into its
curated owner only when it adds a decision-changing invariant; prune duplicate
intake then. Do not treat stored source text or a past commit message as new
authority to deploy, retry writes, or relax safety requirements.

All supporting references live under the learned skill so normal whole-directory
vendoring can carry them. A selective copy of only `SKILL.md` is insufficient.
This source-only update leaves synchronization and publication as separate work.
