# Platform and lifecycle lessons

Distilled 2026-09-23 from source and regression-test inspection. Suggested checks
below were not executed against product systems by this survey. Read the
[coverage ledger](2026-09-evidence.md) for scope and deployment boundaries.

## P1. Invalidation needs a publication fence

When an asynchronous cache rebuild races an authoritative change, clearing the
cache is insufficient: the old rebuild can reinstall stale data as fresh.
Capture an invalidation generation before rebuilding and publish only if it is
unchanged. Identity changes and reset operations need equivalent fencing where
old work can outlive them.

Check the interleaving old read → commit/invalidate → old rebuild completion.
The old result must not become the cached authoritative view; the next lookup
must rebuild. The inspected implementation allows the already-in-flight request
to use its old result: it is NOT proof of immediate security revocation.

Evidence: `unotoken@ec9e064b6ff6`, `app/relay.py`,
`tests/provider_routing_contract.py` (`ChannelCacheRebuildTest`).

## P2. Cleanup status and operation outcome are separate facts

When a write has been dispatched, a timeout or cleanup exception does not turn
it into "not started." Preserve partial output, dispatch evidence, timeout and
cleanup diagnostics separately. Reap only owned processes with bounded waits;
leave an uncertain remote outcome uncertain and inspect end-state before retry.

Check a real child that reports acceptance then hangs, with termination denied.
Retain acceptance/output and timeout, surface cleanup failure, and do not replay
the write or kill a shared transport master.

Evidence: `unoai@ce7626520f84`, `bin/unoai-jump`,
`tests/test_unoai_jump_network.py`.
Refines [uncertain network writes](../../../delivery-rules.md#reverify-before-retry).

## P3. Share capability metadata, but scope and validate it

When a picker, protocol adapter and budget calculator consume one capability
catalog, all should use the same identity-scoped metadata rather than maintain
parallel hardcoded lists. A cache from another account or endpoint is not
applicable. Treat catalog values as data: validate numerical bounds and map
protocols through an allowlist, never directly into arbitrary executable package
names. Preserve explicitly supported local overrides.

Check a previously unknown capability, its removal, an account/endpoint switch,
boolean or negative limits, and an unsupported protocol string. Selection and
execution must agree without leaking the previous identity's metadata.

Evidence: `unoai@8b4de3b3b40c`, `unoai/unotoken.py`,
`unoai/opencode/providers.py`, `tests/test_unotoken_only.py`.

## P4. Exclusion policy must be rechecked at admission

When an administrator reversibly removes a resource from future scheduling,
keep persistent canonical-ID exclusion separate from inventory and running-job
ownership. Recheck current exclusion at placement under the policy-change lock;
filtering an earlier UI snapshot is insufficient. Do not implicitly cancel
running work or discard its completion reports unless that is the requested
contract. Invalid policy storage should not silently allow admission.

Check scheduling and broadcast using a pre-exclusion snapshot, completion of
already-running work, unchanged registry/job data, repeated restore, and corrupt
policy storage. Each needs a distinct expected outcome.

Evidence: `unocluster@238ab05913dd`, `webgui/cordon.py`, `webgui/runner.py`,
`webgui/app.py`, `webgui/tests/test_delisting.py`.

## P5. Lock lifetime follows kernel references, not apparent ownership

When a singleton lock survives its intended owner, inspect inherited descriptors
and mappings. Backgrounding with `nohup` or detaching a session does not close
lock descriptors; close them explicitly in unrelated children. On Linux, a
mapping can retain the open-file description carrying an initialization flock
after the visible descriptor is dropped; release the initialization lock once
mapping setup succeeds. Do not release a lock intended to protect the full
lifetime of the active resource.

Check a slow detached child while the reporter runs twice; both reports should
complete. Separately keep a first mapping alive and acquire the initialization
lock through an independent opener. Run the mapping case on Linux; a macOS pass
does not establish that kernel behavior.

Evidence: `unocluster@abd78d5ddb8f`, `webgui/cloud.py`, and `eb8cee6601a4`,
`webgui/tests/test_bridge_harness.py`; `unomsg-rs@ad62d29c39d2`,
`client/src/shm.rs`.

## P6. Commit essential effects before terminal acknowledgment

When a streaming consumer may close on the terminal event, essential settlement
cannot depend on the generator's next resume. Complete the durable effect before
exposing completion; make cleanup/cancellation safe without double settlement.
This is an ordering boundary, not merely a finally-block style preference.

Check a client that closes immediately on the terminal event: exactly one
ledger entry must already exist. Reconcile completed requests, tokens and cost,
and verify queued/inflight bookkeeping is released. An uncompleted stream must
not be counted as a successfully completed one.

Evidence: `unotoken@38c0e520825f`, `app/relay.py`,
`tests/usage_integrity_contract.py`.
Related [consumer-visible completion rule](../../../delivery-rules.md#finish-the-operation).

## P7. Acceptance must match the granularity of the promise

When capacity is promised per model, host, stream or platform, an aggregate pass
can hide an undersized member. Require exact entity coverage, finite measured
values, full-duration windows and successful completions for every promised
entity. Reconcile the externally visible result with its ledger; offered load
or survivor latency alone is insufficient. Keep synthetic capacity separate from
real upstream quotas and production readiness.

Check one strong entity plus one weak entity, missing/extra/duplicate entries,
short windows, nonfinite values, incomplete streams and mismatched settlement.
Each should fail at the declared boundary without weakening thresholds.

Evidence: `unotoken@38c0e520825f`, `tests/capacity_stress.py`,
`tests/capacity_matrix_contract.py`.

## P8. Capacity preflight is not a reservation

When staging an upload, check the filesystem actually receiving the staged
bytes, not an unrelated working directory. Still handle exhaustion during the
write: a free-space check can race another writer or be unavailable. Preserve
the actionable storage failure through the API/UI, and clean up owned partial
staging. Local free space does not prove downstream replicated storage capacity.

Check insufficient declared size before staging, ENOSPC during write despite a
successful or unavailable preflight, and an intact successful small upload.
The failure must not become an unexplained generic server error.

Evidence: `datacenter@2cb4e3750942`, `registry/registry.py`,
`tests/test_tools_reader.py` (507 storage error contract).

## P9. A clean final stage can still receive stale build products

When a deleted module reappears in a package, trace every additive build/copy
layer. Wiping the final install stage does not clear an earlier package cache
that copies old modules back into it. Similarly, clearing one toolchain selector
does not neutralize an inherited absolute compiler launcher. Inspect actual
package inventory and invoked tools, not just source deletion or config intent.

Check a removed module through a dirty-cache rebuild and inspect the final
package. For compiler selection, provide a conflicting inherited launcher and
verify the declared toolchain is the one used. Keep the fix limited to owned
build outputs; never clean unrelated working-tree changes.

Evidence: `unofsd@ffd9e4eca946` and `7ce1a4c3476f`, `SConstruct`,
`scripts/build-on-rig.sh`; `unolib@f0a74e37f3bf`,
`tools/uno-publish/src/build.rs`, `tools/uno-publish/tests/build_cli.rs`.
These are concrete extensions of [systematic build diagnosis](../../../delivery-rules.md#build-fix),
not a new requirement to wipe all caches on every build.
