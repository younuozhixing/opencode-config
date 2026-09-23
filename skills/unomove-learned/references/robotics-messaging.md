# Robotics and messaging lessons

Distilled 2026-09-23. These are conditional engineering lessons, not permission
to change motion authority. Evidence below is inspected source and checked-in
tests, not tests executed by this survey. See the [coverage ledger](2026-09-evidence.md).
Keep physical constants and acceptance budgets in each project's measured contract.

## R1. Delayed observations require transactional replay

When late data affects an estimator, moving its clock backwards does not undo
intervening updates. Store raw events and pre-event checkpoints; insert by
timestamp plus deterministic tie order, replay later events into independent
staging state, and commit state/history/time together only on success. Bound
both retained history and insertion staging. Epoch and eviction barriers define
when replay is no longer possible. Replay callbacks must not mutate external
state that the transaction cannot roll back.

Check: ordered and delayed permutations converge within the declared tolerance;
a failure in the inserted event OR a later replayed event leaves the entire
committed snapshot unchanged; old-epoch and evicted events are rejected.

Evidence: `unolocalization@0a95a025dfef`,
`crates/unoloc-ekf/src/history.rs` (implementation and inline tests).
Extends [common-clock guidance](../../../engineering-rules.md#units).

## R2. Invalid data and statistical rejection have different consequences

When a filter adapts its acceptance gate, malformed numerical input is not an
outlier. Separate invalid dimensions, nonfinite values, invalid covariance and
singular innovation from a well-formed observation rejected by a statistical
gate. Invalid input must not relax that gate, advance time, or partially commit
a posterior. A pseudoinverse fallback is not proof that a singular innovation
is a valid measurement.

Check: inject each invalid category, assert unchanged state/covariance/time and
adaptation counters, then accept a valid observation. Exercise posterior failure
as well as input validation.

Evidence: `unolocalization@203321bf7c58`, `crates/unoloc-ekf/src/lib.rs` and
`crates/unoloc-estimator/src/lib.rs`.

## R3. A measurement identity must keep its geometry immutable

When consumers key observations by epoch and source timestamp, a replay-corrected
pose under the same key is a conflict, not fresh data. Hold already-published
geometry/twist until the next genuine measurement or an explicitly new epoch.
Allow safety degradation immediately, but do not recover confidence or authority
on geometry whose correction has not been published. Never fabricate a later
timestamp merely to pass freshness checks.

Check: repeated same-key geometry is idempotent; changed same-key geometry is
rejected by the real history consumer; LOST propagates immediately; recovery
waits for a valid new key.

Evidence: `slam_version@27346a3bd2fd`, `bin/locd/src/main.rs` (`PosePublication`
and its tests), `bin/navd/src/measurements.rs`.

## R4. A sampled failure verdict must survive until superseded

When a slow planner publishes into a faster control loop and another process
samples its state, a one-tick failure pulse can vanish between samples. Latch
the verdict for the goal, apply it after competing state branches, and clear it
only on defined transitions such as success, cancellation or a genuinely new
goal. An explicit recovery state is distinct from pretending normal following
has resumed.

Check: sweep producer/consumer phase; every sample before a permitted reset
sees failure or explicit recovery. Later branches cannot overwrite failure or
restore motion inadvertently.

Evidence: `slam_version@c72f43f71385`, `bin/navd/src/main.rs`
(`state_after_plan_failure`, latch lifecycle and regression test).

## R5. Rejected estimates must govern downstream accumulation

When localization holds a meaningful contradictory pose proposal, continuing
to paint scans at the disputed pose turns uncertainty into permanent map
evidence. Carry the verdict into map integration. Distinguish accepted geometry,
a meaningful dispute, ordinary noise, and deliberately uninformative bootstrap
observations; blanket suppression of every weak match can prevent startup.

Check: meaningful translation/heading disputes suppress map writes, while
accepted observations and intentional sparse-map bootstrap still integrate.
Inspect accumulated geometry, not only estimator counters.

Evidence: `slam_version@972101343839`, `bin/locd/src/mapping.rs`
(`integrate_with_verdict`) and `bin/locd/src/main.rs`.

## R6. Preserve attach and loss semantics across transport upgrades

When upgrading a live stream, compile compatibility does not preserve where a
subscriber starts or what overrun means. Declare live-only versus retained
history attachment explicitly. Recover only the typed error whose semantics
permit dropping backlog; keep unrelated corruption/ownership errors visible.
Benchmark latency together with loss: fast surviving frames do not establish
correct delivery. Check every subscriber, not just a primary reader.

Check: wrap the real ring before late attachment; pause a reader until lapped,
then require an explicit overrun and fresh-frame recovery. Inject an unexplained
sequence gap into each subscriber and require acceptance to fail.

Evidence: `unostream@798fdb7dbcf4`, `src/publish.rs`, `tests/roundtrip.rs`;
`unomsg-rs@25e05c2802e4`, `crates/unomsg-shm/examples/matrix_bench.rs`.

## R7. Optional absence is not a failed required sensor

When a supported configuration declares a subsystem absent, exclude its data
AND its faults/validity contribution consistently. Removing tracks while still
letting that absent source invalidate the plan is a half-implemented contract.
This applies only to a deliberately supported optional dependency; it does not
authorize reclassifying a stale required sensor as optional.

Check both configurations: hostile tracks/faults from an absent source have no
contribution, remaining required sources still gate health, and the same faults
remain effective when the subsystem is configured present.

Evidence: `unofsd@abee83cc499e`, `selfdrive/controls/radard.py`,
`selfdrive/controls/tests/test_radard_unavailable.py`.
The [required-sensor freshness gate](../../../engineering-rules.md#stale-input-no-motion)
still applies.

## R8. Synchronization must bound age without phase-starving output

When independent periodic sensors are not phase locked, a narrow timestamp
coincidence gate can create long publication gaps even though both are healthy.
Where the geometry supports it, align measurements using measured pose history
at both source times, with unchanged age limits and no unvalidated extrapolation.
Do not substitute the latest pose or relax freshness merely to obtain output.

Check across a full beat cycle of unequal rates and phase offsets. Assert a
bounded maximum output gap, bounded source age, no reused source geometry, and
correct rotation alignment. Missing interpolation support must wait then expire.

Evidence: `wheelchairsensors@c26651053e01` and `a6e0017e3b02`,
`src/lidar_fusion.rs`, `src/bin/sensors.rs`.

## R9. Calibration must respect the sensor's quantization behavior

For a sensor demonstrated to zero-clamp at rest, subtracting a small calibration
bias can manufacture a constant rate and integrated drift. Establish raw output
semantics and resolution before applying a correction. The inspected sensor's
fix preserves its dead-zone zero and ignores its sub-LSB bias; that policy is
not universal for noisy or dithered sensors whose mean can resolve sub-LSB bias.

Check stationary raw/corrected traces and a known rotation. Cover clamped zero,
sub-LSB bias, measurable bias, and moving input separately; do not copy another
device's LSB or deadband.

Evidence: `wheelchairsensors@9796650257c0`, `src/corrections.rs`, including
`gyro_bias_is_not_subtracted_from_a_dead_zone_zero_or_when_below_one_lsb`.

## R10. A position counter may reset instead of wrap

When integrating an encoder, verify whether its position is absolute, modular,
or per-motion-segment. A stop reset can look like a valid modular delta and add
phantom travel. Cross-check candidate increments against measured velocity and
elapsed time with a quantization-aware bound, skip discontinuities, and avoid
bridging stale gaps. Calibration tools should consume the same validated
integration rather than maintain a competing decoder.

Check a real wrap, stop/reset, stationary frames, stale gap, and new segment.
The reset adds no distance; valid post-reset movement still integrates.

Evidence: `wheelchairsensors@a8b950bdd9b3`, `src/motor.rs`
(`encoder_ignores_the_stop_reset_and_keeps_velocity_consistent_steps`),
`src/bin/wheel_calib.rs`.

## R11. A safety intervention is not evidence of actuator deadband

When online calibration interprets held command plus stationary wheels as
under-drive, an obstacle guard reducing the command can cause the learner to
increase effort into the obstruction. Qualify calibration data by the applied
command and intervention state. Suspend the probe branch during guard reduction;
evaluate other adaptation branches against their own valid-data contract.

Check identical stationary feedback with guard clear versus guard active. The
active guard must not cause effort-increasing probe adaptation. Explicitly test
missing/stale guard feedback too: the historical implementation permits a
missing guard, so this evidence does NOT establish fail-closed guard freshness.

Evidence: `wheelchairsensors@722e4e20e0ee`, `src/can.rs` (`probe_allowed`,
`PlantGain`, `TurnGain`, inline test), `src/bin/can_driver.rs`.

## R12. Ring capacity is not a payload contract

When sizing a shared-memory record, account for alignment, framing and wrap
geometry. Use the pinned implementation's maximum payload and typed rejection
contract. Rejection must occur before shared cursor, lease or payload mutation;
silently truncating serialized data is not a valid fallback.

Check max and max-plus-one payloads at wrap boundaries and small unusual ring
capacities. A rejected send leaves reader-visible state unchanged and the next
valid send works. Do not transplant a historical size formula into another
protocol version.

Evidence: `unomsg-rs@c03775468056`, `client/src/shm.rs` (wrapped rejection
regression); snapshot `unomsg-rs@be56c02f210b`,
`crates/unomsg-shm-core/src/lib.rs`, `crates/unomsg-shm-core/tests/contract.rs`,
`client/src/shm.rs` (typed error mapping).
This corrects the old blanket truncation claim in
[messaging rules](../../../engineering-rules.md#messaging-hot-path).
