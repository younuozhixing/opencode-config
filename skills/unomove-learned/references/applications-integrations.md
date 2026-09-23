# Applications and integration lessons

Distilled 2026-09-23. These lessons describe inspected source changes, not live
service certification. Suggested checks are validation criteria, not claims
that this survey executed them. See the [evidence ledger](2026-09-evidence.md).

## A1. Authentication recovery needs an explicit stopping state

When an identity provider accepts a session that an application rejects,
redirecting back to the provider can return the same unusable cookie forever.
Reject ambiguous duplicate cookie names, expire the known host/domain/path
variants, and prevent automatic reuse of the rejected session. Bound the recovery
attempt and finish with an explicit authentication failure if it still fails.
Recovery responses should not be cached; successful recovery clears only its
retry marker, not the newly valid session.

Check missing identity, revoked identity, invalid signature and duplicate cookies
through both page and API routes. A rejected session must not loop indefinitely
or become authorized merely because recovery was attempted.

Evidence: `feishu_reimbursement@174276332d63`, `auth.py`, `server.py`,
`tests/test_session_recovery.py`.

## A2. Durable intent and verified completion are different records

When changing a consequential external ledger, persist a bounded, access-controlled,
masked intent before dispatch. If preparation fails, do not write. After dispatch,
record only the state confirmed by readback; lost acknowledgments and failed
completion persistence need an explicit reconciliation state, not blind replay
or a success claim. Match batch results by stable identity and expected fields,
not response position alone.

Check preparation-storage failure (zero external writes), lost acknowledgment
(readback without replay), partial/mismatched results, failed readback, and failed
completion-storage after the write. Preserve the intent for reconciliation.
A reimbursement-status snapshot is not evidence that a bank transfer occurred.

Evidence: `feishu_reimbursement@96b69624b5f4`, `server.py`
(`_store_payout_snapshot`, `_batch_update_status`, settlement and batch-create
handlers), `tests/test_enterprise_robustness.py`.
Extends [idempotent external writes](../../../delivery-rules.md#idempotent-writes).

## A3. Editing evidence invalidates the old approval first

When amount, date or attachment content changes, a verification verdict on the
old inputs no longer authorizes the new record. Invalidate that verdict in the
serialized update before slow re-verification. Recheck attachment identity
before applying extracted output; a completed extraction must not overwrite a
replacement attachment's record. A verifier outage should leave the record
pending, never silently inherit the old pass.

Check an approved record edited while verification is unavailable, attachment
replacement during extraction, and mismatching document identity. Settlement
must not accept the stale verdict. Revalidation success is a separate transition.

Evidence: `feishu_reimbursement@96b69624b5f4`, `server.py`
(`handle_correct`, `handle_reocr`), `tests/test_enterprise_robustness.py`.

## A4. Extract numeric meaning from context, not a value blacklist

When OCR places dates, quantities, promotions and payable amounts on one line,
ranking all numeric tokens can choose a year or discount threshold. Remove
recognized non-money spans before ranking and prefer explicit total/currency
context. Do not globally reject values that resemble years: the same number can
be a legitimate currency-marked amount. Ambiguous text should remain unresolved
rather than being converted to a confident amount.

Check dates with mixed separators, short dates, promotional threshold/discount
phrases and a nearby paid total. Pair every false-positive guard with a legitimate
year-sized monetary amount so a blunt exclusion cannot pass the suite.

Evidence: `feishu_reimbursement@29f9fa0f254d`, `parser.py`,
`tests/test_receipt_safety.py`, `tests/test_batch_receipt_formats.py`;
follow-up `b1f45fe` covers two-digit date forms. Use structured source data first
when available under [structured extraction guidance](../../../delivery-rules.md#structured-over-ocr).

## A5. A usable snapshot certifies age AND query coverage

When a cached availability snapshot answers a different date range, a recent
timestamp does not establish that the requested range was read. Track covered
intervals and freshness independently. Age slow reads from their start, not
completion, and reapply recent local bookings after selecting a snapshot so an
older in-flight refresh cannot reopen a newly booked slot. Keep fast browsing
separate from live, fail-closed conflict checks at booking time.

Check a fresh but out-of-range snapshot, stale complete coverage, a slow read,
a new booking racing refresh, and a transient source failure. Unknown coverage
must not become "free." Bound UI waits and discard failed extra-date results.

Evidence: `feishu-icloud-sync@85b77c1a91b1`, `booking_server.py`,
`tests/test_booking_cache.py`.
This complements the publication-generation race in
[platform lesson P1](platform-lifecycle.md), rather than replacing write-time checks.

## A6. Observability tooling must not replace the system it observes

When deploying a monitor beside a robot/runtime stack, consuming shared code
does not require sharing mutable release files. Package the monitor's pinned
transport library and schema/registry inputs in its own versioned release;
do not overwrite or symlink into the monitored deployment's replaceable bundle.
Record the files' hashes and verify the loader/decoder against that layout.

Check by removing the monitored bundle in an isolated fixture, starting the
monitor without library-path environment assistance, and validating its manifest.
This is release isolation, not permission to fork shared-library source or relax
wire compatibility requirements.

Evidence: `unoflow@191d52f02f4a`, `scripts/prepare-rig-release.sh`,
`tests/rig-isolation.sh`.

## A7. A shared command receiver must arbitrate session authority

When several connected operator clients publish to one command topic, an idle
client's zeros can overwrite the active operator at every tick even with perfect
network delivery. Enforce session ownership at the common receiver. Distinguish
the owner's release-zero from another session's idle-zero, release ownership on
defined timeout/close transitions, and preserve the existing authorized emergency
stop path. Sender-side silence is only defense in depth.

Check interleaved active and idle clients at realistic rates, competing active
clients, owner release-to-zero, timeout, disconnect and emergency braking. The
historical brake-preemption policy applies to connected control sessions; it is
not permission to admit unauthenticated commands or bypass other safety gates.

Evidence: `unogui@8a57e3ce3e05`, `scripts/control_owner.py`,
`scripts/webrtcd.py`, `tests/test_control_owner.py`, `unogui/app/joystick.py`.
Refines the existing [single publisher](../../../engineering-rules.md#sole-publisher)
and actuator-authority rules with a multi-session fan-in failure.

## A8. Demand rendering needs complete invalidation and time-based motion

When reducing continuous rendering, enumerate every visual change: telemetry,
map/image arrival, gesture, resize, theme, selection and time-based overlay
expiry. Draw when dirty or animation is settling, within a measured frame budget.
Convert per-frame smoothing to elapsed-time smoothing so reduced frame rate
does not change camera/control feel. Cache theme lookups and batch drawing where
measurements show repeated work.

Check idle versus live paint counts, every invalidation source, overlay expiry
while telemetry is idle, and animation behavior at varied frame rates. This
implementation still schedules animation callbacks; fewer paints is not a claim
of zero idle callbacks. No new regression tests were added in the cited commit;
its historical performance numbers were not reproduced by this survey.

Evidence: `unomobile@810e669e1073`, `app/js/map.js`.
